---
layout: page
title: Others
permalink: /others/
---

Miscellaneous posts and content.

{% for post in site.categories.others %}
  <article class="post-preview">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
  </article>
{% endfor %}

