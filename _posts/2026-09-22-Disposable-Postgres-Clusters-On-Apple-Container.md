---
title: "Disposable Postgres clusters on Apple's container runtime"
date: 2026-09-22
description: >-
  What I learned putting a multi-node Postgres lab on apple/container instead
  of Docker Desktop: name resolution, address families, signal handling, and
  the numbers.
---

Most of my week is spinning something up to check it. Does this feature do what the docs say it does. Can I reproduce what the customer hit. What actually happens to client connections when that node goes away. The answer is almost always a small throwaway cluster, and which cluster matters less than how fast I can have one and how little I care about destroying it.

Docker Desktop does this well, and for plenty of people it is the right answer. My usage just does not match its shape. I will lean on a local cluster hard for a week while I am chasing something down, then not open it again for a month. Docker Desktop keeps a Linux VM and a daemon resident across all of that, and at most employers it also wants a paid subscription for commercial use. Carrying a standing cost for something I use in bursts is what sent me looking, not anything wrong with the tool.

So I spent a while on [`apple/container`](https://github.com/apple/container), Apple's own container runtime for Apple silicon, working out whether a multi-node Postgres lab fits on it. It does. The first thing I built is a three-node EDB Postgres Distributed cluster, [cider-press](https://github.com/theadamwright/cider-press), because PGD is what I work on. Almost nothing I learned along the way was specific to it.

## What the runtime gives you

No daemon. There is no always-on Linux VM holding memory in your menu bar. Containers start on demand and are gone when stopped, so you pay for three Postgres nodes only while three Postgres nodes are running.

Each container is a lightweight VM on macOS's own virtualization framework, with real isolation rather than shared-kernel namespaces.

And the one that changed how I built everything: every container gets a real IP on your Mac's network and a DNS name to go with it. For clustered software that is not cosmetic. Nodes can address each other the way they already expect to, rather than through a proxy, a compose alias, or a published port. `psql -h host-1.cider` works from macOS with no port forward.

Installation is a signed package and there is no licence to think about.

## What it takes away

There is no `compose`. No declarative file, no dependency ordering, no `up`.

For a single container that costs nothing. For anything where node 1 must exist and be healthy before node 2 can join it, you are writing that orchestration yourself. That is most of the reason cider-press exists as a tool rather than a README full of commands.

## Three things that will bite you

These would hit anyone running clustered software on this runtime, not just Postgres.

**Names are fully qualified or they do not work.** The runtime resolves containers through an embedded DNS service and registers them as `<name>.<domain>`. Looking one up by bare hostname is explicitly unsupported ([apple/container#1809](https://github.com/apple/container/issues/1809)). That matters for any system that writes a peer address into its own configuration or catalog, which is most of them. PGD records, per node, the address every other node should dial it on. Put a name in there that only resolves sometimes and you get a cluster that works today and half-fails after a restart, and you will not think to blame the name.

**Every container gets both an IPv4 and an IPv6 address, and the records land moments apart.** This produced two separate failures for me. First, host-based auth: the `pg_hba.conf` generated for me covered `0.0.0.0/0` only, so a peer arriving over IPv6 was rejected outright. Second, and nastier, Postgres resolves `listen_addresses` once at startup. A node that starts early can bind IPv4 only while its peers bind both, at which point the peers dial it over IPv6, get nothing, and it sits `Unreachable` with consensus failing. Listening on every address removes the race. Whatever you are running, do not assume your peers reach you on the address family you tested.

**Signals.** This one cost me real time, and I would not have found it without measuring for this post, so it gets its own section below.

## The one I got wrong

Bringing an existing cluster back up was taking 4 minutes 43 seconds, against 25 seconds to press a completely fresh one. That is backwards, and nothing in normal use pointed at why.

`container stop` sends SIGTERM and kills the container five seconds later. Both defaults are wrong for a database. Postgres reads SIGTERM as a *smart* shutdown: stop accepting new connections, then wait for existing client sessions to end on their own.

Replication connections are exempt from that wait. The postmaster reclassifies walsenders and shuts them down later, after the checkpoint. Ordinary client backends are not exempt, and a PGD node always has some: Connection Manager holds pooled backends, the monitor worker has its own, and consensus traffic between nodes arrives as ordinary connections too. None of them are going to disconnect because you asked the container to stop, so the wait does not end. The five seconds expire, the runtime SIGKILLs the server, and the node lands on disk unclean:

```
LOG:  database system was not properly shut down; automatic recovery in progress
```

Every stop was a crash. Nothing complained at the time, because the container did stop. You only pay for it on the next startup, which is slow for no visible reason.

The fix is SIGINT, which is Postgres's fast shutdown, with a grace period that is not five seconds.

None of this is new. The official Postgres Docker image has set `STOPSIGNAL SIGINT` since September 2020, for exactly this reason. What bit me is that `apple/container` has no equivalent default and I had not thought to set one, so I got the OCI default of SIGTERM and a five second timer. Worth checking on any runtime where you built the image yourself.

To be clear about the damage: an unclean shutdown costs you WAL replay on the next start, not your data. Postgres is built for this. It was slow, not dangerous.

```
down                    16.9s  ->   1.1s
up on same volumes      4m43s  ->  18.6s
container stop host-1   5.376s ->  0.199s
```

## The numbers

One run on an M4 Pro with 48 GB, macOS 26.6.2, `container` 1.4.1, three nodes at the 2 GB default. Single sample, not a benchmark.

Pressing a cluster from nothing takes 34 seconds: three volumes created, node 1 seeded, nodes 2 and 3 joined in sequence, connection pooling set. Bringing an existing one back takes 18.6 seconds.

Memory came in lower than I expected, because allocation is lazy and a node holds what it is using rather than what you gave it:

```
host-1   360.05 MiB / 2.00 GiB
host-2   308.98 MiB / 2.00 GiB
host-3   302.69 MiB / 2.00 GiB
```

Under a gigabyte for three nodes. Counting the host side, where each container is a real VM with its own overhead, the three came to roughly 2 GB. It all comes back on teardown. An idle cluster costs about 1.5 percent CPU per node.

Fifty sequential connects through the cluster's connection router, each opening a connection, running a query and exiting, took 2.6 seconds. About 52 ms each including psql startup. Stop the write leader and the first successful connection on another node answered 8.3 seconds later.

## What is still off

One case survives the signal fix. Stop a node while it is still catching up rather than settled, and fast shutdown does not finish inside the grace period either, so that node crash-recovers exactly as before. I hit it by tearing down 20 seconds after restarting a node. That cycle took 1 minute 5 seconds to stop and 3 minutes 42 seconds to come back. The same down and up on a settled cluster takes 1.1 seconds and 18.6 seconds.

Raising the grace period further is not the answer. I need to check the node is in a state where it can shut down cleanly before asking it to, and I have not built that yet.

Two smaller ones. One node occasionally takes a minute to rejoin when the other two come back immediately, with nothing in its log to explain it. And now that startup finishes in 18 seconds, a readiness probe that used to have minutes of cover sometimes does not finish in time.

Still happy with it. I get a real cluster in under a minute, which was the whole point, and the number that was embarrassing last week is fine now.

## What is next

The commands are grouped as `cider <product> <verb>` and the verbs are deliberately product-agnostic, so a second stack is a new group rather than a rewrite.

[EDB Failover Manager](https://www.enterprisedb.com/docs/efm/latest/) is the one I have checked as far as feasibility. `edb-efm54` is published for Debian 12 arm64, and EFM's virtual IP works on this runtime: an address added under `--cap-add NET_ADMIN` is reachable from both macOS and a peer container, because vmnet routes addresses it did not itself assign. [Patroni](https://patroni.readthedocs.io/) with etcd is the same failover problem solved in the open-source world, and is the other obvious candidate.

Neither is built and neither is promised. If you are running something else clustered on this runtime, I would like to know what broke for you, because I suspect the three problems above are most of the list.

## What this is not

It is a lab. The password is `secret` and it is written in the repository. Ports bind to loopback only, there is no TLS between nodes, and every node is a container on one Mac, so it demonstrates leader routing and failover convincingly while surviving none of the things high availability exists for, starting with that Mac going away.

cider-press is also not an EDB project. It is not endorsed or supported by EnterpriseDB and is not covered by any support agreement, so please open an issue on the repo rather than a support ticket. PGD itself is commercial software and you need your own subscription. For anything real, use the [supported deployment paths](https://www.enterprisedb.com/docs/pgd/latest/deploying/).

One last detail, and it is the one most likely to cost you an afternoon on a managed laptop. Beware enterprise IT software. On some hosts `host-1.cider` will not resolve or connect from macOS at all, because a VPN client, an endpoint-security agent or a per-application proxy intercepts the traffic before it ever reaches the container network. Netskope, Zscaler, Cisco Secure Client and similar all do this, and it can affect one process and not another, so "it works in my other terminal" rules nothing out.

The error you get is `No route to host`, which reads like a broken cluster and is not one. Nothing in the cluster depends on that path. Nodes talk to each other inside the container network where your host's policies do not apply, and every `127.0.0.1` port works normally. Connecting by name is a convenience, not a requirement.

<https://github.com/theadamwright/cider-press>
