---
title: Home
nav_order: 1
---

## 최근 업데이트

{% assign docs = site.pages |
where_exp: "item", "item.path contains 'docs/'" |
where_exp: "item", "item.last_modified_at" |
sort: "last_modified_at" | reverse %}

<div class="recent-docs">
  {% for doc in docs %}
    {% unless doc.path contains 'assets' or doc.path contains '.scss' %}
    {% assign path_parts = doc.path | split: '/' %}
    {% if path_parts.size >= 3 %}
      {% assign top_categories = "" | split: ',' %}
      {% assign sub_categories = "" | split: ',' %}
      {% assign last_index = path_parts.size | minus: 2 %}
      {% for i in (1..last_index) %}
        {% assign category = path_parts[i] | capitalize %}
        {% if i == 1 %}
            {% assign top_categories = top_categories | push: category %}
        {% endif %}
        {% if i == 2 %}
            {% assign sub_categories = sub_categories | push: category %}
        {% endif %}
      {% endfor %}
    {% endif %}
    <div class="doc-entry">
      <div class="doc-title">
        <a href="{{ doc.url | relative_url }}" class="doc-link">
          {{ doc.title | default: doc.path | split: "/" | last | split: "." | first }}
        </a>
      </div>
      <div class="doc-meta">
        <span class="label label-grey">수정일: {{ doc.last_modified_at | date: "%Y-%m-%d" }}</span>
        {% for category in top_categories %}
          <span class="label label-custom-sky">{{ category }}</span>
        {% endfor %}
        {% for category in sub_categories %}
          <span class="label label-custom-green">{{ category }}</span>
        {% endfor %}
      </div>
    </div>
    {% endunless %}
  {% endfor %}
</div>
