---
layout: default
---

# theadamwright.github.io
### Databases, Security, and K8s 

Welcome to my site. I document my work with distributed databases, security best practices, and Kubernetes orchestration.

---

## Blog Posts

{% for post in site.posts %}
* {{ post.date | date: "%B %d, %Y" }} &raquo; [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}

---
