---
title: Blog
permalink: /blog/
---

Write-ups on projects and what I learned building them.

{% if site.posts and site.posts.size > 0 %}
  <div class="grid">
    {% for post in site.posts %}
      <div class="card">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <p class="muted">
          <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%b %d, %Y" }}</time>
        </p>
        {% if post.description %}<p class="muted">{{ post.description }}</p>{% endif %}
      </div>
    {% endfor %}
  </div>
{% else %}
  <p class="muted">No posts yet.</p>
{% endif %}
