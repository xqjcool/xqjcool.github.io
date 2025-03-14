---
layout: default
title: "我的博客"
author: Jack Xing
description: Analysis of Linux related issues.
---

# 欢迎来到我的博客

这里是我的技术博客，最新文章如下：

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> - <small>{{ post.date | date: "%Y-%m-%d" }}</small>
    </li>
  {% endfor %}
</ul>
