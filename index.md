---
title: Home
description: Portfolio, projects, and notes.
---

<section class="hero">
  <h1>Hi, I'm Raz.</h1>
  <p>
    This site is for open-source projects, fun sidequests, and a personal blog where I
    write up what I build and learn along the way.
  </p>
  <div class="cta-row">
    <a class="button primary" href="{{ '/projects/' | relative_url }}">View projects</a>
    <a class="button" href="{{ '/blog/' | relative_url }}">Blog</a>
    {% if site.linkedin_url %}<a class="button" href="{{ site.linkedin_url }}">LinkedIn</a>{% endif %}
    {% if site.email %}<a class="button" href="mailto:{{ site.email }}">Email</a>{% endif %}
  </div>
</section>

<section class="grid">
  <div class="card half">
    <h2>Featured project</h2>
    {% assign p = site.data.projects[0] %}
    <h3>{{ p.name }}</h3>
    <p class="muted">{{ p.description }}</p>
    <p>
      {% if p.repo %}<a href="{{ p.repo }}">Repo &rarr;</a>{% endif %}
      {% if p.post %}<span class="muted">&nbsp;&middot;&nbsp;</span><a href="{{ p.post | relative_url }}">Write-up &rarr;</a>{% endif %}
    </p>
    {% if p.highlights %}
      <ul class="list">
        {% for h in p.highlights limit: 3 %}
          <li>{{ h }}</li>
        {% endfor %}
      </ul>
    {% endif %}
  </div>

  <div class="card half">
    <h2>Latest write-up</h2>
    {% assign post = site.posts.first %}
    {% if post %}
      <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
      {% if post.description %}<p class="muted">{{ post.description }}</p>{% endif %}
      <p class="muted">
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%b %d, %Y" }}</time>
      </p>
      <p><a href="{{ '/blog/' | relative_url }}">All posts &rarr;</a></p>
    {% else %}
      <p class="muted">No posts yet.</p>
    {% endif %}
  </div>
</section>
