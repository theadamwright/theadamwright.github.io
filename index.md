---
layout: default
title: Adam Wright — Distributed Postgres, Security, and Kubernetes
---

<section class="hero">
  <h1>Hi, I'm Adam. 👋</h1>
  <p>
    I'm the Product Manager for <a href="https://www.enterprisedb.com/products/edb-postgres-distributed">EDB Postgres Distributed (PGD)</a>.
    Here I document my work with distributed databases, security best practices, and
    Kubernetes orchestration — the practical side of running serious workloads on PostgreSQL.
  </p>
</section>

<section class="posts">
  <h2>Writing</h2>
  {% for post in site.posts %}
  <article class="post-card">
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    <p class="post-meta"><time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time></p>
    {% if post.description %}<p>{{ post.description }}</p>{% endif %}
  </article>
  {% endfor %}
</section>
