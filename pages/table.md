---
layout: default
title: Benjamin Bisping
permalink: /cv
weight: 2
---

<div id="header" style="min-height: 8rem">
  <div style="position: absolute">
    <img src="{{ site.author.image }}" style="width: 6rem" alt="{{ site.title }}" class="circle-image">
  </div>
  <div class="header-right" style="margin-left: 7rem">
    <a href="{{ site.baseurl }}/"><h1 class="mb-0 mt-0"><b>{{ site.title }}</b></h1></a>
    <p class="text-muted mb-0 mt-1">{{ site.description }}</p>
    <p class="text-muted mb-0 mt-0"><a href="mailto:{{ site.author.email }}">{{ site.author.email }}</a></p>
  </div>
</div>

{% assign products = site.data.products | sort: "date" | reverse %}

{% assign article_kinds = "article|formalization" | split: "|" %}
{% assign journal_products = products | where_exp: "p", "article_kinds contains p.kind"%}
{% assign conference_products = products | where: "kind", "paper" %}
{% assign talk_products = products | where: "kind", "talk" %}
{% assign software_kinds = "tool|website" | split: "|" %}
{% assign software_products = products | where_exp: "p", "software_kinds contains p.kind" %}
{% assign course_products = products | where: "kind", "course" %}
{% assign thesis_products = products | where: "kind", "advising" %}

{% include elements/product-table.html title="Journal articles and formalizations" kind="article" products=journal_products %}

{% include elements/product-table.html title="Conference papers" kind="paper" products=conference_products %}

{% include elements/product-table.html title="Talks" kind="talk" products=talk_products %}

{% include elements/product-table.html title="Software and artifacts" kind="tool" products=software_products %}

{% include elements/product-table.html title="Courses taught" kind="course" products=course_products hide_venue=true %}

{% include elements/product-table.html title="Theses supervised" kind="advising" products=thesis_products hide_venue=true %}
