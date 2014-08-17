---
layout: page
title: 主页
description: "This a blog shares about my thinking of the technology in field of computer science and software engineering. 这里是我的博客，博客的内容主要为技术相关的分享和思考。"
---
{% include JB/setup %}

<h3>近期发表的博客</h3>
<hr />
<ul class="posts">
  {% for post in site.posts %}
    <li><span class="date-time">{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>