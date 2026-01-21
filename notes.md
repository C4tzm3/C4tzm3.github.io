---
layout: page
title: Notes
permalink: /notes/
---

# Technical Notes

My collection of technical notes, tutorials, and documentation.

{% for post in site.categories.notes %}
  <article class="post-preview">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
    <p>{{ post.excerpt }}</p>
  </article>
{% endfor %}
