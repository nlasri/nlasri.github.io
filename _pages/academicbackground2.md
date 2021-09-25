---
layout: archive
title: "Academic background"
permalink: /academicbackground/
author_profile: true
---

Here is my academic background:

{% include base_path %}

{% for post in site.academicbackground reversed %}
  {% include archive-single.html %}
{% endfor %}
