---
title: "What PostgreSQL 19 changes for logical replication, tested on a two-node pair"
date: 2026-10-13
description: >-
  Sequence sync, update_deleted, subscriptions over a foreign server, per-subscription
  timeouts, EXCEPT and logical decoding without a restart: what each one is for,
  how to try it on a pair of nodes, and the gotchas I hit doing that.
---

Like almost every PostgreSQL release since version 10, PostgreSQL 19 includes a number of logical replication enhancements. Thank you to all the PostgreSQL hackers and reviewers who made 19 another great release for logical replication. At the time of this writing, packages exist for beta4. I went through each logical replication item in the release notes and ran it on a pair of PostgreSQL 19 beta4 containers.  Here are my notes, the practical implications, and what surprised me.

Two caveats. It is a beta, so a feature can still be pulled before release. An unusual number of features were reverted late in the cycle. Everything below was run on 19beta4. And some of these features already have very good write-ups. In those cases, I kept it short and linked to them. 

## The test setup

I used [cider-press](https://github.com/theadamwright/cider-press), the lab I [wrote about last month]({% post_url 2026-09-22-Disposable-Postgres-Clusters-On-Apple-Container %}). Using `cider logical` gives you two PostgreSQL nodes from PGDG, each in its own lightweight VM with its own address, so the pair talks over a real network rather than two clusters sharing localhost.

```bash
CIDER_LOGICAL_PG_MAJOR=19 ./cider logical build
CIDER_LOGICAL_PG_MAJOR=19 ./cider logical up
```

`up` prints the SQL that wires the two nodes in both directions. Paste it into `./cider logical psql 1` and `./cider logical psql 2`, and `./cider logical status` shows both subscriptions streaming, plus any conflicts by type. The nodes run with `track_commit_timestamp = on`, because several of the features below need it.

Each feature below has the commits, the pgsql-hackers thread, a short way to try it, and what it means in practice.

## Sequence synchronisation

Commits: [96b3784](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96b37849734673e7c82fb86c4f0a46a28f500ac8) (`ALL SEQUENCES`), [f0b3573](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0b3573c3aac6c0ea4cbc278f98178516579d370) (`REFRESH SEQUENCES`), [5509055](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5509055d6956745532e65ab218e15b99d87d66ce) (the sync worker). Thread: [pgsql-hackers](https://postgr.es/m/CAA4eK1LC+KJiAkSrpE_NwvNdidw9F2os7GERUeSxSKv71gXysQ@mail.gmail.com).

Rows have always replicated and sequences never have. So you move a table with ids 1 to 1000 to a subscriber, cut over, and the first insert there gets id 1 and fails. PostgreSQL 19 lets a publication include every sequence, and copies their values to the subscriber:

```sql
-- publisher
CREATE PUBLICATION pub_sequences FOR ALL SEQUENCES;

-- subscriber, just before cutover
ALTER SUBSCRIPTION orders_from_dolores_1 REFRESH SEQUENCES;
SELECT last_value FROM order_id;
```

On 19, `./cider logical endpoints` prints a complete one-way script for this, from creating the sequence to the first insert after cutover.

The values are copied when the subscription is created (with the default `copy_data = true`), when `REFRESH PUBLICATION` finds a new sequence, and when you run `REFRESH SEQUENCES`. At no other time. In between, rows keep replicating and the sequences stay where they were. That makes this a feature for cutovers and upgrades, and the practical points all follow from it:

- **`REFRESH SEQUENCES` returns before the values change.** A background worker applies them, in under half a second in my tests, but an insert sent straight after the command still got the old value. Check `last_value` before you move writes. [depesz's walkthrough](https://www.depesz.com/2025/11/11/waiting-for-postgresql-19-sequence-synchronization-in-logical-replication/) shows the same thing with timings on 10,000 sequences.
- **It needs a free worker slot, and says so only in the log.** On a node already running as many subscriptions as `max_logical_replication_workers` allows (4 by default), `REFRESH SEQUENCES` returned success and nothing changed. The sequence stayed in state `i` in `pg_subscription_rel`, and the only clue was `WARNING: out of logical replication worker slots` in the server log. Freeing a slot let it finish.
- **A refresh can move a sequence backwards.** The worker sets the publisher's value on the subscriber without comparing the two. I set the subscriber's sequence to 700 against the publisher's 100, refreshed, and got 100 back. The next `nextval()` was 101, below ids the subscriber had already handed out. So don't write on the subscriber before cutover, and don't point sequence sync both ways between two nodes that both take writes.
- **`FOR ALL SEQUENCES` can't share a publication with a table list.** `FOR TABLE orders, ALL SEQUENCES` is a syntax error. Use a second publication and subscribe to both, or `FOR ALL SEQUENCES, ALL TABLES`.
- **Monitoring queries may need a change.** `pg_stat_subscription_stats.sync_error_count` is renamed `sync_table_error_count` ([3edaf29](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3edaf29fa5e0d4fd6419c7eb73fd6df5b0406789)), next to a new `sync_seq_error_count` ([f6a4c49](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f6a4c498dcf6f05b4ef79051e95de12cc48bdeee)).

## Telling a deleted row from a missing one: `update_deleted`

Commits: [228c370](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=228c3708685542d34e6f02c74240656327a5c622) (retain the information), [fd5a1a0](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fd5a1a0c3e566f7fc860838084466a1c25944281) (detect `update_deleted`), [a850be2](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a850be2fe653b3b529969946c1cefe0fd9e34a8d) (`max_retention_duration`), [0d48d39](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0d48d393d465b6f1abe18b86bd5ac2de0636a40e) (resume retention automatically). Thread: [pgsql-hackers](https://postgr.es/m/OS0PR01MB5716BE80DAEB0EE2A6A5D1F5949D2@OS0PR01MB5716.jpnprd01.prod.outlook.com).

PostgreSQL 18 started detecting and logging apply conflicts. One case it could not get right: node A deletes a row while node B updates it. When B's update reaches A, the row is gone, and 18 reports `update_missing`, the same as an update to a row that never existed. Those are different problems. One is a concurrent delete, the other usually means the two nodes were never in sync.

PostgreSQL 19 can tell them apart if the subscription keeps deleted rows around long enough to look at them. That is the subscription option `retain_dead_tuples`. It creates a physical replication slot on the subscriber, `pg_conflict_detection`, whose `xmin` holds VACUUM back. Detection also needs `track_commit_timestamp`.

To see it, turn the option on for the subscription on node 1, then make the two changes meet:

```sql
-- node 1: the option can only be changed while the subscription is disabled
ALTER SUBSCRIPTION from_dolores_2 DISABLE;
ALTER SUBSCRIPTION from_dolores_2 SET (retain_dead_tuples = true);
ALTER SUBSCRIPTION from_dolores_2 ENABLE;

INSERT INTO pingpong VALUES (200, 'original');   -- let it replicate

-- pause BOTH subscriptions, then:
DELETE FROM pingpong WHERE id = 200;                                   -- node 1
UPDATE pingpong SET msg = 'updated on dolores-2' WHERE id = 200;       -- node 2
-- resume node 2's subscription, then node 1's
```

The [cider-press README](https://github.com/theadamwright/cider-press#postgresql-19-a-row-deleted-on-one-node-and-updated-on-the-other) has the full step-by-step version. Node 1 logs:

```
LOG:  conflict detected on relation "public.pingpong": conflict=update_deleted
DETAIL:  Could not find the row to be updated: remote row (200, updated on dolores-2), replica identity (id)=(200).
         The row to be updated was deleted locally in transaction 691 at 2026-10-12 16:57:20.061431+00.
```

and node 2 logs `delete_origin_differs`. Without `retain_dead_tuples`, the same steps log `update_missing` on node 1.

What it means in practice:

- **It is detection, not resolution.** Node 2 applies the incoming delete and node 1 skips the incoming update. Both nodes end without the row, so the update is lost on both, and the only record is the log line and a counter in `pg_stat_subscription_stats.confl_update_deleted`.
- **Pause both subscriptions to reproduce it.** If only one is paused, the delete reaches the other node before the update is made, and the `UPDATE` finds nothing to change.
- **`SET` can fail straight after `DISABLE`.** I got `cannot alter option "retain_dead_tuples" when logical replication worker is still running`. Waiting a moment and running it again worked.
- **The cost is dead rows.** While the subscription runs, the slot's `xmin` keeps moving forward by itself. While it is disabled, the `xmin` stays put and VACUUM can't remove the dead rows. Postgres warns about this at `DISABLE`, and the docs suggest turning the option off before a long pause. Turning it off drops the slot.
- **`max_retention_duration` caps the retention.** According to the docs, if the apply worker can't move the slot forward within that time, retention stops (`pg_subscription.subretentionactive` goes false), `update_deleted` detection pauses, and it resumes when the worker catches up. The slot is shared, so retention stops only once every subscription using it has passed its limit. In my tests an open transaction on the publisher did not hold the slot back; the cap is aimed at a subscriber that falls behind in applying changes.

## Subscribing through a foreign server

Commit: [8185bb5](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8185bb53476378443240d57f7d844347d5fae1bf). Thread: [pgsql-hackers](https://postgr.es/m/61831790a0a937038f78ce09f8dd4cef7de7456a.camel@j-davis.com).

A subscription can now take its connection from a foreign server instead of a connection string. `postgres_fdw` supports it:

```sql
-- on the subscriber
CREATE EXTENSION postgres_fdw;
CREATE SERVER node1 FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'dolores-1.cider', dbname 'demo', tcp_user_timeout '5000');
CREATE USER MAPPING FOR postgres SERVER node1
  OPTIONS (user 'postgres', password 'secret');
CREATE SUBSCRIPTION notes_from_1 SERVER node1 PUBLICATION p_notes;
```

That worked first time on the pair. The subscription's `subconninfo` is empty: the host lives in the server and the password in the user mapping. Practical consequences:

- **Change it in one place.** The host lives in the server and the password in the user mapping, and the subscription stores neither. `ALTER SERVER` restarts the subscription’s worker so it reconnects with the new settings, so moving a publisher or changing its options doesn’t mean touching each subscription. A changed password in the user mapping is used the next time the worker connects.
- **The usual rules for foreign servers apply.** The subscription owner needs `USAGE` on the server and a user mapping for itself, and a non-superuser still needs `pg_create_subscription` to create the subscription.
- **Keep `tcp_user_timeout` in the server's options.** It is a libpq option like any other here, and the next section shows why you want it.

## A timeout per subscription

Commits: [fb80f38](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb80f388f4a140934f51f68578ba321152332ce7) (per subscription), [8a6af3a](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8a6af3ad08799755e575275d0df5feb7b102ca35) (user-settable). Thread: [pgsql-hackers](https://postgr.es/m/a1414b64-bf58-43a6-8494-9704975a41e9@oss.nttdata.com).

`wal_receiver_timeout` decides how long an apply worker waits on a silent connection before giving up on it. It used to be one value for the whole server. Now each subscription can override it:

```sql
CREATE SUBSCRIPTION from_far_away
  CONNECTION 'host=dolores-1.cider dbname=demo tcp_user_timeout=5000'
  PUBLICATION pp WITH (wal_receiver_timeout = '5s');

SELECT subname, subwalrcvtimeout FROM pg_subscription;
```

The default, `-1`, keeps the server's setting. That helps when one node subscribes to peers over very different links: a short timeout for a nearby peer, so a dead one is noticed quickly, without making a slow cross-region link flap.

One thing it doesn't cover: connection attempts. I hard-killed the publisher node. A subscription with this 5-second timeout but no `tcp_user_timeout` in its connection string then sat in a single connection attempt for about 84 seconds before it logged anything. The subscriptions with `tcp_user_timeout=5000` retried every 5 seconds. Set both.

## Excluding tables from a publication

Commits: [fd36606](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fd366065e06ae953c4f2d973d5c5f0474f3b87b6), [493f8c6](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=493f8c6439cf64d75883c650b5dd573d8fe0664b) (`ALTER PUBLICATION`), [5984ea8](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5984ea868eeeb202bddedfbdaca81adfe7ea6779) (the final syntax). Thread: [pgsql-hackers](https://postgr.es/m/CALDaNm3=JrucjhiiwsYQw5-PGtBHFONa6F7hhWCXMsGvh=tamA@mail.gmail.com).

```sql
CREATE PUBLICATION everything_but_audit FOR ALL TABLES EXCEPT (TABLE audit_log);
```

`\dRp+` lists the exclusions under `Except tables`. The syntax changed during development, which is why older posts and patches show other forms; the parenthesised form above is the one in 19. [Peter Smith's post](https://www.postgresql.fastware.com/pzone/excluding-tables-from-publications-in-postgresql-19) goes through how partitions and inheritance are handled: only a partitioned table's root can be excluded, which excludes its whole tree, and `ONLY` limits an exclusion to the parent of an inheritance tree. [depesz](https://www.depesz.com/2026/03/26/waiting-for-postgresql-19-add-support-for-except-table-in-alter-publication/) shows that `ALTER PUBLICATION ... SET ALL TABLES EXCEPT` replaces the exclusion list rather than adding to it. His post predates the syntax change, so it uses the older `EXCEPT TABLE (...)` form, which 19 rejects.

Two things matter day to day, and neither is new in 19: they apply to any `FOR ALL TABLES` publication.

**Create a new table on the subscriber first.** A table created later joins the publication automatically. If a row is written to it before the table exists on the subscriber, the apply worker stops with `logical replication target relation "public.…" does not exist`, and nothing else in that subscription gets through until the table is created there. Once it exists, the subscriber ignores the table until `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`, which copies the rows written so far because `copy_data` is on by default.

**In both directions, copy on one side only.** Refresh one node with the default and the other `WITH (copy_data = false)`. With the default on both, the second node copies back rows that came from the first, the copy fails on a duplicate key and retries, and that table stops replicating in that direction. Postgres warns about it at the refresh: `requested copy_data with origin = NONE but might copy data that had a different origin`.

In a bidirectional setup, `EXCEPT` is also a simple way to keep node-local tables, an audit log say, out of the stream: exclude them in each node's publication.

## Logical decoding without a restart

Commit: [67c2097](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=67c20979ce72b8c236622e5603f9775968ff501c). Thread: [pgsql-hackers](https://postgr.es/m/CAD21AoCVLeLYq09pQPaWs+Jwdni5FuJ8v2jgq-u9_uFbcp6UbA@mail.gmail.com).

Turning on logical replication used to mean `wal_level = logical` and a restart. In 19, a server at `wal_level = replica` switches to logical WAL by itself when the first logical slot is created, and a new read-only setting, `effective_wal_level`, shows which level is really in effect. I checked it on 19beta4: `wal_level` and `effective_wal_level` both said `replica`, then I created a logical slot, `effective_wal_level` said `logical`, and the slot decoded changes, with no restart.

The practical upshot is one less restart to schedule before you can start replicating. Going back down when the last slot is dropped, and how standbys are affected, are covered well by [Shveta Malik](https://www.postgresql.fastware.com/blog/online-wal-level-change-for-logical-decoding-in-postgresql-19) and [Christophe Pettus](https://thebuild.com/blog/the-wallevel-you-set-is-not-the-wallevel-you-get/). cider-press starts its nodes at `wal_level = logical`, so `SHOW effective_wal_level` there just says `logical`.

## Smaller changes

- **`pg_replication_origin_session_setup()` takes a pid** ([5b14870](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5b148706c5c8ffffe5662fe569a0f0bcef2351d9), [thread](https://postgr.es/m/CAMPB6wfe4zLjJL8jiZV5kjjpwBM2=rTRme0UCL7Ra4L8MTVdOg@mail.gmail.com)), so several processes can share one replication origin, the way core's parallel apply workers already do. That lets replication tools outside core apply large transactions in parallel.
- **pg_createsubscriber** can reuse publications that already exist ([85ddcc2](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85ddcc2f4cdef490276d151c80459e287bceb782), [thread](https://postgr.es/m/CAHv8Rj%2BsxWutv10WiDEAPZnygaCbuY2RqiLMj2aRMH-H3iZwyA%40mail.gmail.com)), writes its recovery settings to a separate `pg_createsubscriber.conf` instead of `postgresql.auto.conf` ([639352d](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=639352d904c8a3d233989253f569cccbea0be123), [thread](https://postgr.es/m/CAGWv16K6L6Pzm99i1KiXLjFWx2bUS3DVsR6yV87-YR9QO7xb3A@mail.gmail.com)), and gains `--logdir` ([6b5b7ea](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6b5b7eae3ae65572e2abda20ef2b2c908527c28a), [thread](https://postgr.es/m/CAEqnbaUthOQARV1dscGvB_EsqC-YfxiM6rWkVDHc+G+f4oSUHw@mail.gmail.com)).

## If you run it in both directions

Most of this release is aimed at one-way replication: upgrades, migrations, cutovers. Running a pair in both directions, the useful parts and the limits line up like this:

- `origin = none` (since 16) still does the essential job of stopping changes bouncing back and forth.
- `update_deleted` and the conflict types added in 18 tell you when the two nodes have diverged. Nothing in core decides what should have happened; the outcome per conflict type is fixed, and some of those outcomes leave the nodes different.
- Sequence sync is one way and overwrites, so it does not help two writers share an id space. Give each node its own range, or use UUIDs (`uuidv7()` arrived in 18).
- `EXCEPT` helps keep node-local tables out.
- DDL still is not replicated, in 19 as before.

## Coming after 19

Already on master for PostgreSQL 20: conflicts can be written to a table as well as the log ([a5918fd](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a5918fddf10d297c70f7ec9067e9177e0be6d520), [thread](https://postgr.es/m/CAFiTN-u5D5o_AGNbHRZHaOqAMWkxLf%2BhSk_r9X3gv6HbLOB5%2Bg%40mail.gmail.com)), with a subscription option, `conflict_log_destination`, of `log`, `table` or `all`. A table you can query is a much better place than the server log to find out which rows two nodes disagree on. It is a year away and can still change.