---
layout: page
title: Projects
permalink: /projects/
---

# Projects

Showcase of my technical projects and contributions.

{% for post in site.categories.projects %}
  <article class="post-preview">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
    <p>{{ post.excerpt }}</p>
  </article>
{% endfor %}
