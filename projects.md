---
layout: page
title: Projects
permalink: /projects.html
---

<ul class="project-list">
{% for project in site.projects %}
  <li>
    <h3><a href="{{ project.url | relative_url }}">{{ project.title }}</a></h3>
    <p>{{ project.summary }}</p>
    {% if project.repo %}<p><a href="{{ project.repo }}">View code on GitHub →</a></p>{% endif %}
  </li>
{% endfor %}
</ul>
