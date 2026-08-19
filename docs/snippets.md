---
layout: page
title: Snippets
---

## All Snippets

<ul>
  {% for snippet in site.snippets %}
    <li>
      <a href="{{ snippet.url }}">{{ snippet.title | default: snippet.name }}</a>
    </li>
  {% endfor %}
</ul>
