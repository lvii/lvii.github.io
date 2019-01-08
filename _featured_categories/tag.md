---
layout: page
title: 标签
# The name of the tag, used in a post's front matter (e.g. tags: [<slug>]).
# slug: example
menu: true
order: 2
---
{% for tag in site.tags %}<a href="#{{ tag | first }}">{{ tag | first }}<sup>{{ tag[1].size }}</sup></a> {% endfor %}
{% for tag in site.tags %}
<h2>
  <a id="{{ tag | first }}"># {{ tag | first }}
  </a>
</h2>
<ul class="title-list">
{% for post in tag.last %}
  <li><a href="{{ post.url | relative_url }}">{{ post.title }}<span>{{ post.date | date:"%Y-%m-%d" }}</span></a></li>
{% endfor %}
</ul>
{% endfor %}