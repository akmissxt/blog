---
layout: page
permalink: /posts/
title: sundries
description: 杂七杂八的一些东西
---

<ul class="post-list">
{% for poem in site.posts reversed %}
    <li>
        <h2><a class="poem-title" href="{{ poem.url | prepend: site.baseurl }}">{{ poem.title }}</a></h2>
        <p class="post-meta">{{ poem.date | date: '%B %-d, %Y — %H:%M' }}</p>
      </li>
{% endfor %}
</ul>