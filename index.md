---
layout: default
title: Home
---

# ⚡ TeckMagix Docs

Study guides, lab solutions and notes 

## 🗒️ Recent Notes

{% assign notes = site.pages | where_exp: "p", "p.path contains 'os_notes/'" | sort: 'date' | reverse %}
{% if notes.size > 0 %}
{% for note in notes %}
- [{{ note.title | default: note.name }}]({{ note.url }}) {% if note.date %} — {{ note.date | date: "%b %d, %Y" }}{% endif %}
{% endfor %}
{% else %}
Lagging
{% endif %}