---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
title: Tags
---

<h1>All Tags</h1>

<ul class="tag-list">
{% for tag in site.tags %}
  <li class="tag-list-item">
  <a href="/tags/{{tag.tag-name}}">{{ tag.tag-name }}</a>
  </li>
{% endfor %}
</ul>
