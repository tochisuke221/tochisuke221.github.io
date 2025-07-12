---
layout: default
---

<ul class="post-list">
{% for post in site.posts %}
  <li class="post-item">
    <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
    <a href="{{ post.url | relative_url }}" class="post-title">{{ post.title }}</a>
    {% if post.category %}<span class="category">{{ post.category }}</span>{% endif %}
  </li>
{% endfor %}
</ul>