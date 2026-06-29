---
permalink: /database/
title: "数据库与技术学习"
layout: archive
author_profile: false
---

这里保留 Oracle、SQL、数据库支持和技术学习记录。它不是为了包装成“资深生产 DBA”，而是呈现真实做过的实验、理解和排错过程。

{% assign database_count = 0 %}
{% for post in site.posts %}
  {% if post.categories contains "Oracle" %}
    {% assign database_count = database_count | plus: 1 %}
    {% include archive-single.html %}
  {% elsif post.categories contains "数据库" %}
    {% assign database_count = database_count | plus: 1 %}
    {% include archive-single.html %}
  {% elsif post.categories contains "技术" %}
    {% assign database_count = database_count | plus: 1 %}
    {% include archive-single.html %}
  {% endif %}
{% endfor %}

{% if database_count == 0 %}
数据库内容正在整理中。
{% endif %}
