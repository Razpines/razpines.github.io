---
title: Projects
permalink: /projects/
---

Open-source projects, fun sidequests, and write-ups.

<div class="grid">
  {% for p in site.data.projects %}
    <div class="card">
      <h2>{{ p.name }}</h2>
      <p class="muted">{{ p.description }}</p>
      <p>
        {% if p.repo %}<a href="{{ p.repo }}">Repo &rarr;</a>{% endif %}
        {% if p.post %}{% if p.repo %}<span class="muted">&nbsp;&middot;&nbsp;</span>{% endif %}<a href="{{ p.post | relative_url }}">Write-up &rarr;</a>{% endif %}
        {% if p.url %}{% if p.repo or p.post %}<span class="muted">&nbsp;&middot;&nbsp;</span>{% endif %}<a href="{{ p.url }}">arXiv &rarr;</a>{% endif %}
        {% if p.url2 %}{% if p.repo or p.post or p.url %}<span class="muted">&nbsp;&middot;&nbsp;</span>{% endif %}<a href="{{ p.url2 }}">IEEE Xplore &rarr;</a>{% endif %}
      </p>
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
