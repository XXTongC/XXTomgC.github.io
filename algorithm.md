---
layout: page
title: 算法笔记
permalink: /algorithm/
---

# 算法学习笔记

{% assign grouped = site.algorithm | group_by_exp: "item", "item.path | split: '/' | slice: 1, 1 | first" %}

{% for group in grouped %}
## {{ group.name }}

{% for doc in group.items %}
- [{{ doc.title }}]({{ doc.url | relative_url }})
{% endfor %}

{% endfor %}