---
layout: page
title: 算法笔记
permalink: /algorithm/
---

# 算法学习笔记

<div style="text-align: center; margin: 20px 0; padding: 15px; background-color: #f5f5f5; border-radius: 5px;">
  <a href="#by-category" style="margin: 0 10px; font-weight: bold;">📁 按分类</a> | 
  <a href="#by-tag" style="margin: 0 10px; font-weight: bold;">🏷️ 按标签</a> | 
  <a href="#by-date" style="margin: 0 10px; font-weight: bold;">📅 按日期</a>
</div>

---

<h2 id="by-category">📁 按分类浏览</h2>

{% assign grouped = site.algorithm | group_by_exp: "item", "item.path | split: '/' | slice: 1, 1 | first" %}

{% for group in grouped %}
### {{ group.name }}

{% for doc in group.items %}
- [{{ doc.title }}]({{ doc.url | relative_url }})
{% endfor %}

{% endfor %}

---

<h2 id="by-tag">🏷️ 按标签浏览</h2>

{% assign all_tags = site.algorithm | map: "tags" | join: ',' | split: ',' | uniq | sort %}

{% for tag in all_tags %}
{% if tag != "" %}
### {{ tag }}

{% assign tagged_docs = site.algorithm | where_exp: "item", "item.tags contains tag" | sort: "date" | reverse %}
{% for doc in tagged_docs %}
- [{{ doc.title }}]({{ doc.url | relative_url }}) <small style="color: #666;">{{ doc.date | date: "%Y-%m-%d" }}</small>
{% endfor %}

{% endif %}
{% endfor %}

---

<h2 id="by-date">📅 按日期浏览</h2>

{% assign sorted_by_date = site.algorithm | sort: "date" | reverse %}

{% for doc in sorted_by_date %}
- **{{ doc.date | date: "%Y-%m-%d" }}** - [{{ doc.title }}]({{ doc.url | relative_url }})
  {% if doc.tags %}
  <br><small style="color: #666;">🏷️ {{ doc.tags | join: ", " }}</small>
  {% endif %}
{% endfor %}