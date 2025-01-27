---
layout: grid
title: Blog
permalink: /blog
no_groups: true
---

<div>
    {% for tag in site.tags %}
    <a href="/tags/{{ tag[0] | slugify }}" class="post-tag"><strong>{{ tag[0] | capitalize }}</strong></a>{% unless forloop.last %} | {% endunless %}
    {% endfor %}
</div>

---
## Recent Posts