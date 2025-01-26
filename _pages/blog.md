---
layout: grid
title: Blog
permalink: /blog
no_groups: true
---

## Tags

<div>
    {% for tag in site.tags %}
    <a href="/tags/{{ tag[0] | slugify }}" class="post-tag"><strong>{{ tag[0] | capitalize }}</strong></a>
    {% endfor %}
</div>

## Posts