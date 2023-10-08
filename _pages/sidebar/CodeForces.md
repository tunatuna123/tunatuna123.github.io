---
title: "CodeForces"
layout: archive
permalink: sidebar/CodeForces
author_profile: true
types: posts
---

{% assign posts = site.categories['CodeForces']%}
{% for post in posts %}
  {% include archive-single.html type=page.entries_layout %}
{% endfor %}