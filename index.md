---
layout: page
title: 有熊出没请躺下
tagline: di 技术博客 -- 看看书 & 读读码 & 爱妻/子 & 思人生
---
{% include JB/setup %}

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date:"%Y-%m-%d" }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

