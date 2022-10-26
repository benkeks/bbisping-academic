---
layout: default
title: Benjamin Bisping's Research
permalink: /
weight: 1
---


{% assign products = site.data.products | sort: "date" | reverse %}

{% assign product_years = products | group_by_exp: "prod", "prod.date | truncate: 4, ''" %}

{% for y in product_years %}
  <div style="text-align: center; width: 100%">{{y.name}}</div>
  <div class=row>
  {% assign products_by_area = y.items | group_by: "area" %}
  {% for a in site.data.areas %}
    {% assign prods = products_by_area | find: 'name', a.id %}
    <div class="col-lg-{{prods.items | size | at_most: 3}}">
    {% for p in prods.items %}
      {% include elements/product.html product=p %}
    {% endfor %}
    </div>
  {% endfor %}
  </div>
{% endfor %}

<script>

</script>