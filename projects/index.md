---
title: Projects
permalink: /projects/
---

Open-source projects and selected shipped work.

<div class="grid">
  {% for p in site.data.projects %}
    <div class="card">
      <h2>{{ p.name }}</h2>
      <p class="muted">{{ p.description }}</p>
      {% if p.repo %}<p><a href="{{ p.repo }}">Repo &rarr;</a></p>{% endif %}
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
