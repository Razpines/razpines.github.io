---
title: Home
description: Portfolio, CV, projects, and notes.
---

<section class="hero">
  <h1>Hi, I’m Raz.</h1>
  <p>
    I build pragmatic software: automation pipelines, tools, and systems that ship.
    This site is my portfolio/CV and a place to write up projects.
  </p>
  <div class="cta-row">
    <a class="button primary" href="{{ '/projects/' | relative_url }}">View projects</a>
    <a class="button" href="{{ '/cv/' | relative_url }}">Read CV</a>
    <a class="button" href="{{ '/blog/' | relative_url }}">Blog</a>
    <a class="button" href="{{ '/contact/' | relative_url }}">Contact</a>
  </div>
</section>

<section class="grid">
  <div class="card half">
    <h2>Featured project</h2>
    {% assign p = site.data.projects[0] %}
    <h3>{{ p.name }}</h3>
    <p class="muted">{{ p.description }}</p>
    <p><a href="{{ p.repo }}">Repo →</a></p>
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
      <p><a href="{{ '/blog/' | relative_url }}">All posts →</a></p>
    {% else %}
      <p class="muted">No posts yet.</p>
    {% endif %}
  </div>

  <div class="card">
    <h2>What I’m optimizing for</h2>
    <ul class="list">
      <li>Shipping reliable systems end-to-end</li>
      <li>Automation that handles ugly real-world inputs</li>
      <li>Clean, inspectable pipelines over brittle monoliths</li>
    </ul>
  </div>
</section>
