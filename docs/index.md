---
layout: default
title: Docs
permalink: /docs/
description: Browse all StackNotes documentation categories.
---

<h1 class="section-title">Documentation</h1>
<p class="section-lede">Every category, generated from the site navigation.</p>

{% for group in site.data.navigation.sidebar %}
<h2 class="docs-group">{{ group.group }}</h2>
<div class="topic-grid">
  {% for item in group.items %}
    {% if item.collection %}{% assign target = '/' | append: item.collection | append: '/' %}{% else %}{% assign target = item.url %}{% endif %}
    {% if item.external %}
      <a href="{{ item.url }}" class="topic-card glow-card" target="_blank" rel="noopener">
        <span class="topic-emoji" aria-hidden="true">{{ item.icon }}</span>
        <span class="topic-name">{{ item.title }}</span>
      </a>
    {% else %}
      <a href="{{ target | relative_url }}" class="topic-card glow-card">
        <span class="topic-emoji" aria-hidden="true">{{ item.icon }}</span>
        <span class="topic-name">{{ item.title }}</span>
        {% if item.status == 'soon' %}<span class="topic-soon">soon</span>{% endif %}
      </a>
    {% endif %}
  {% endfor %}
</div>
{% endfor %}
