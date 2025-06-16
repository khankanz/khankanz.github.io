---
layout: default
title: Home
---

# Posts

<ul>
  {% for post in site.posts %}
    <li style="margin-bottom: 1rem;">
      <a href="{{ post.url }}"><strong>{{ post.title }}</strong></a> — {{ post.date | date: "%Y-%m-%d" }}  
      <br />
      {{ post.excerpt | strip_html | truncate: 160 }}
    </li>
  {% endfor %}
</ul>
