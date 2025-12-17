---
title: Home
description: Portfolio, projects, and notes.
---

<section class="hero">
  <h1>Hi, I'm Raz Pines.</h1>
  <p>
    I'm an Applied Scientist focused on ML personalization and time-series modeling.
    I like shipping end-to-end systems: messy data → models → production.
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
    <p><a href="{{ p.repo }}">Repo &rarr;</a></p>
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

  <div class="card">
    <h2>What I'm optimizing for</h2>
    <ul class="list">
      <li>Personalization and ranking at scale</li>
      <li>Strong evaluation: offline metrics + online experiments</li>
      <li>Production ML: pipelines, deployment, and monitoring</li>
    </ul>
  </div>
</section>
