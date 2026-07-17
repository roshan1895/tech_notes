---
layout: default
title: Certifications
permalink: /certifications/
description: Certification tracks mapped to learning roadmaps.
---

<nav class="breadcrumb" aria-label="Breadcrumb">
  <a href="{{ '/docs/' | relative_url }}">Docs</a>
  <span class="breadcrumb-sep" aria-hidden="true">/</span>
  <span class="breadcrumb-current" aria-current="page">Certifications</span>
</nav>

<h1 class="section-title">Certifications</h1>
<p class="section-lede">Certification tracks, each mapped to a learning roadmap.</p>

<div class="cert-grid">
  {% for c in site.data.certifications.certifications %}
    <a href="{% if c.roadmap %}{{ '/roadmaps/' | append: c.roadmap | append: '/' | relative_url }}{% else %}#{% endif %}" class="cert-card glow-card">
      <div class="cert-top">
        <span class="cert-emoji" aria-hidden="true">{{ c.icon }}</span>
        <span class="cert-status status-{{ c.status }}">{{ c.status }}</span>
      </div>
      <h3>{{ c.short | default: c.name }}</h3>
      <p class="cert-provider">{{ c.provider }}</p>
      <p class="cert-full">{{ c.name }}</p>
    </a>
  {% endfor %}
</div>
