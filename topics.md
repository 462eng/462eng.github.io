---
layout: page
title: All topics
permalink: /topics/
---

Every article of every day on one page, newest day first. Use your
browser's search (Ctrl+F) to find a term.

{% assign articles = site.posts | where_exp: "p", "p.day" %}
{% assign days = articles | group_by: "day" %}
<ul class="topic-list">
{%- for g in days %}
  {%- assign daypost = nil -%}
  {%- for p in site.categories.daily -%}
    {%- assign pd = p.date | date: "%Y-%m-%d" -%}
    {%- if pd == g.name -%}{%- assign daypost = p -%}{%- endif -%}
  {%- endfor -%}
  {%- assign items = g.items | sort_natural: "title" -%}
  {%- for a in items %}
  <li>
    <a href="{{ a.url | relative_url }}"><strong>{{ a.title | escape }}</strong></a><br>
    <span class="post-meta">{% if daypost %}<a href="{{ daypost.url | relative_url }}">{{ g.name }}</a>{% else %}{{ g.name }}{% endif %}{% if a.tags.size > 0 %} · {{ a.tags | join: ", " }}{% endif %}</span><br>
    {{ a.description | default: a.excerpt | strip_html | escape }}
  </li>
  {%- endfor %}
{%- endfor %}
</ul>
