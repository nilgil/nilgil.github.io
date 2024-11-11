---
title: Home
nav_order: 1
description: "홈페이지 설명"
---

# __Daily Update Wiki__

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
      {% assign top_category = path_parts[1] | capitalize %}
    {% endif %}
    <div class="doc-entry">
      <div class="doc-title">
        <a href="{{ doc.url | relative_url }}" class="doc-link">
          {{ doc.title | default: doc.path | split: "/" | last | split: "." | first }}
        </a>
      </div>
      <div class="doc-meta">
        <span class="label label-blue">수정일: {{ doc.last_modified_at | date: "%Y-%m-%d" }}</span>
        <span class="label label-green">{{ top_category }}</span>
      </div>
    </div>
    {% endunless %}
  {% endfor %}
</div>
