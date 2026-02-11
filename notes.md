---
layout: page
title: Notes
permalink: /notes/
---

My collection of technical notes, tutorials, and documentation organized by category.

## 📁 Categories

<div class="category-list" style="margin: 2em 0;">
  <div style="padding: 1em; border-left: 4px solid #0366d6; background: #f6f8fa; margin-bottom: 1em;">
    <h3 style="margin-top: 0;">
      <a href="/notes/splunk/" style="text-decoration: none; color: #0366d6;">
        📊 Splunk Administration
      </a>
    </h3>
    <p style="margin-bottom: 0;">SIEM deployment, configuration, forwarders, and management guides</p>
    <small>{{ site.categories.splunk | size }} posts</small>
  </div>

  <!-- Add more categories here as you create them -->
</div>

---

## All Notes

{% for post in site.categories.notes %}
  <article class="post-preview">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
  </article>
{% endfor %}
