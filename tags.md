---
layout: page
title: "Projects by Tag"
---

{% for tag in site.tags %}
## {{ tag[0] }}
<ul>
  {% for post in tag[1] %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
{% endfor %}
