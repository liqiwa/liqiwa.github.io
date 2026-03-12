---
title: "OpenClaw 实战专栏"
layout: archive
permalink: /openclaw/
author_profile: true
---

> 记录 OpenClaw 日常使用中的排障、升级、配置优化过程，附上实际使用的提示词和命令。信息均已脱敏处理，希望对同样在折腾 AI 助手的朋友有参考价值。

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'OpenClaw'" %}
{% for post in posts %}
  {% include archive-single.html %}
{% endfor %}
