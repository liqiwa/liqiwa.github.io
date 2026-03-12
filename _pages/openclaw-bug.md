---
title: "OpenClaw · Bug 排障"
layout: archive
permalink: /openclaw/bug/
author_profile: true
---

> 收录 OpenClaw 使用中遇到的 Bug、异常行为及修复过程，包括工具调用失败、性能问题、兼容性回退等。

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'OpenClaw' and post.categories contains '排障'" %}
{% if posts.size == 0 %}
<p>暂无文章，敬请期待。</p>
{% else %}
{% for post in posts %}
  {% include archive-single.html %}
{% endfor %}
{% endif %}
