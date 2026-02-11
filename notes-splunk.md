---
layout: page
title: "Splunk Notes"
permalink: /notes/splunk/
---

# Splunk Administration & Configuration

Complete guides and notes for Splunk deployment, configuration, and management.

## All Splunk Posts

<ul class="post-list">
{% for post in site.categories.splunk %}
  <li>
    <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
    <h3>
      <a class="post-link" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
      </a>
    </h3>
    {% if post.excerpt %}
      <p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
    {% endif %}
  </li>
{% endfor %}
</ul>

---

[← Back to all Notes](/notes/)
