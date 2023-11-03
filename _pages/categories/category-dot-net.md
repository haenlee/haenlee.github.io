---
title: ".NET"
layout: archive
permalink: /categories/dotnet
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories['.NET'] %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} 
{% endfor %}