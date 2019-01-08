---
layout: page
title: 分类
# The name of the tag, used in a post's front matter (e.g. tags: [<slug>]).
# slug: example
menu: true
order: 1
---

{% for category in site.categories %}<a class="button" href="#{{ category | first }}">{{ category | first }}</a> {% endfor %}

{% for category in site.categories %}
<h2><a id="{{ category | first }}">{{ category | first }}</a></h2>

<ul class="title-list">
{% for post in category.last %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}<span>{{ post.date | date:"%Y-%m-%d" }}</span></a></li>
{% endfor %}
</ul>

{% endfor %}