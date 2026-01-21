---
layout: page
title: CTF Writeups
permalink: /ctf/
---

# CTF Writeups

Collection of my Capture The Flag challenge writeups and solutions.

{% for post in site.categories.ctf %}
  <article class="post-preview">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
    <p>{{ post.excerpt }}</p>
  </article>
{% endfor %}
