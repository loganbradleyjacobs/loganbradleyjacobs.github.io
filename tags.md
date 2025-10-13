---
layout: default
title: "Projects by Tag"
permalink: /tags/
---

<h1>Projects by Tag</h1>

{% for tag in site.tags %}
<h2>{{ tag[0] }}</h2>
<ul>
  {% for post in tag[1] %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
{% endfor %}