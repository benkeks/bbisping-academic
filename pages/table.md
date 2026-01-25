---
layout: default
title: Benjamin Bisping
permalink: /cv
weight: 2
---

{% assign products = site.data.products | sort: "date" | reverse %}

{% assign journal_products = products | where: "kind", "article" %}
{% assign conference_products = products | where: "kind", "paper" %}
{% assign talk_products = products | where: "kind", "talk" %}
{% assign software_kinds = "tool|artifact|website" | split: "|" %}
{% assign software_products = products | where_exp: "p", "software_kinds contains p.kind" %}
{% assign course_products = products | where: "kind", "course" %}
{% assign thesis_products = products | where: "kind", "advising" %}
{% assign handled_kinds = "article|paper|talk|tool|artifact|website|course|advising" | split: "|" %}

{% include elements/product-table.html title="Journal articles" kind="article" products=journal_products %}

{% include elements/product-table.html title="Conference papers" kind="paper" products=conference_products %}

{% include elements/product-table.html title="Talks" kind="talk" products=talk_products %}

{% include elements/product-table.html title="Software and artifacts" kind="tool" products=software_products %}

{% include elements/product-table.html title="Courses taught" kind="course" products=course_products %}

{% include elements/product-table.html title="Theses supervised" kind="advising" products=thesis_products %}
