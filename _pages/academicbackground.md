---
layout: archive
title: "Academic background"
permalink: /academicbacground/
author_profile: true
---

Here is my academic background:

{% include base_path %}

{% for post in site.publications reversed %}
  {% include archive-single.html %}
{% endfor %}
