---
title: Projects
permalink: /projects/
---

Here are a few things I’ve built and shipped.

<div class="grid">
  {% for p in site.data.projects %}
    <div class="card">
      <h2>{{ p.name }}</h2>
      <p class="muted">{{ p.description }}</p>
      {% if p.repo %}<p><a href="{{ p.repo }}">Repo →</a></p>{% endif %}
      {% if p.highlights %}
        <ul class="list">
          {% for h in p.highlights %}
            <li>{{ h }}</li>
          {% endfor %}
        </ul>
      {% endif %}
    </div>
  {% endfor %}
</div>
