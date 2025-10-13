---
layout: home
title: "Projects Home"
---

<div class="home-page-content">
  <h1>Projects</h1>
  <p>Welcome to my project blog! Each post below includes a thumbnail, tags, and short description.</p>

  <ul style="list-style:none; padding:0;">
    {% for post in site.posts %}
    <li style="margin-bottom:2rem; display:flex; align-items:flex-start;">
      {% if post.thumbnail %}
        <a href="{{ post.url }}">
          <img src="{{ post.thumbnail }}" alt="{{ post.title }} thumbnail"
               style="width:150px; height:auto; border-radius:10px; margin-right:1rem; box-shadow:0 2px 8px rgba(0,0,0,0.1);" />
        </a>
      {% endif %}
      <div>
        <h2 style="margin:0;"><a href="{{ post.url }}">{{ post.title }}</a></h2>
        <p style="margin:0.25rem 0; font-size:0.9rem; color:gray;">
          {{ post.date | date: "%B %d, %Y" }}
        </p>
        <p>{{ post.excerpt }}</p>
        
        <!-- Tags section -->
        {% if post.tags %}
        <div style="margin-top:0.5rem;">
          {% for tag in post.tags %}
          <a href="{{ '/tags/#' | relative_url }}#{{ tag | slugify }}" class="tag">{{ tag }}</a>
          {% endfor %}
        </div>
        {% endif %}
      </div>
    </li>
    {% endfor %}
  </ul>
</div>
