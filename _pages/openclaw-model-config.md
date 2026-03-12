---
title: "OpenClaw · 模型配置"
layout: archive
permalink: /openclaw/model-config/
author_profile: true
---

> 收录 OpenClaw 的模型接入、参数调优、API 配置相关经验，包括不同模型的行为差异和最佳实践。

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'OpenClaw' and post.categories contains '模型配置'" %}
{% if posts.size == 0 %}
<p>暂无文章，敬请期待。</p>
{% else %}
{% for post in posts %}
  {% include archive-single.html %}
{% endfor %}
{% endif %}
