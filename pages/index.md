---
layout: start
title: Benjamin Bisping's Research
permalink: /
weight: 1
---

{% assign width = 1140 %}
{% assign height = 4000 %}
{% assign cap_at_year = 2016 %}

{% assign products = site.data.products | sort: "date" | reverse %}

{% assign product_years = products | group_by_exp: "prod", "prod.date | truncate: 4, ''" %}

{% for ar in site.data.areas %}
<div class="area-info" id="area-{{ar.id}}" data-id="{{ar.id}}">
  <h2 class="area-name" style="border-bottom: 3px solid {{ar.color}}">{{ar.name}}</h2>
  <p class="area-description">{{ar.description | markdownify}}</p>
</div>
{% endfor %}

<div id="product-canvas" class="row justify-content-center" style="width: {{width}}px; position: relative;">

<svg width="{{width}}" height="0" id="research-history">
  <defs>
  </defs>
</svg>

{% for y in product_years %}
<div class="year-mark" data-year="{{y.name}}">{{y.name}}</div>
{% endfor %}


{% for y in product_years %}
  {% assign products_by_area = y.items | group_by: "area" %}
  {% for a in site.data.areas %}
    {% assign prods = products_by_area | find: 'name', a.id %}
    {% assign num_in_group = 0 %}
    {% for p in prods.items %}
      {% include elements/product.html product=p num=num_in_group%}
      {% assign num_in_group = num_in_group | plus: 1 %}
    {% endfor %}
  {% endfor %}
{% endfor %}

<button type="button" id="show-times" class="btn btn-light">Show undergrad times</button>

</div>


<script type="module">

import * as d3 from "https://cdn.skypack.dev/d3@7";

const width={{width}};
const height={{height}};
let capAtYear={{cap_at_year}};

const areas = {{site.data.areas | jsonify }};
const areaNums = new Map(d3.map(d3.range(0, areas.length), (i => [areas[i].id, i])));

const areaInfos = d3.selectAll(".area-info")
  .datum(function() { return this.dataset; });

const products = d3.selectAll("div.product")
  .on("click", expandProduct);
// fetch data from dom
products.datum(function() { return {
  date: new Date(this.dataset.date),
  year: parseInt(this.dataset.year),
  area: this.dataset.area,
  num: parseInt(this.dataset.num)
}; });
const productsData = products.data();

const bounds = d3.extent(productsData, d => d.year);
const years = d3.range(bounds[0], bounds[1] + 2);

const productsByYear = d3.group(productsData, d => d.year);

const areaNames = d3.map(areas, a => a.id);

const areaGraph = d3.map(
  years,
  y => {
    const row = d3.rollup(productsByYear.get(y) || [], v => v.length, p => p.area);
    row['year'] = y
    return row;
  });

console.log(areaGraph);

let focus = '';
let expandedProduct = null;

const stacking = d3.stack()
  .keys(areaNames)
  .value((d, k) => {
    const val = d.get(k) || 0.2;
    return val * (k == focus ? 1.5 : 1.0);
  })
  .offset(d3.stackOffsetSilhouette);

const x = d3.scaleLinear([-8,8], [0, width]);
const y = d3.scaleLinear([bounds[0], bounds[1] + .5], [height,0]);
const yD = d3.scaleTime([new Date(bounds[0],5,1), new Date(bounds[1],11,31)], [height,0]);

const graphShapes = d3.area()
  .x0(d => x(d[0]))
  .x1(d => x(d[1]))
  .y(d => y(d.data.year))
  .curve(d3.curveCardinal);

const svg = d3.select("#research-history");

// add backgrounds for areas
svg.selectAll("def > pattern")
  .data(areas)
  .enter()
  .append("pattern")
  .attr("id", d => "pattern-"+d.id)
  .attr("x", 0)
  .attr("y", 0)
  .attr("width", 200)
  .attr("height", 200)
  .attr("patternUnits", "userSpaceOnUse")
  .html((d,i) =>
    `<rect x="0" y="0" width="200" height="200" fill="${d.color}"></rect>
      <text fill="white" class="pattern-watermark" x="50" y="${40 + (i * 20) % 50}" font-size="4rem">${d.symbols}</text>
      <text fill="white" class="pattern-watermark" x="150" y="${140 + (i * 20) % 50}" font-size="4rem">${d.symbols}</text>`
  )

let currentStack = stacking(areaGraph);

const path = svg.selectAll("path")
  .data(currentStack)
  .join("path")
    .attr("d", graphShapes)
    .classed("area", true)
    .attr("id", (d,i) => "area-" + areas[i].id)
    .attr("fill", (d,i) => `url(#pattern-${areas[i].id})`)
    .on("click", (e,d) => focusArea(d.key));

const yearMarks = d3.selectAll(".year-mark")
  .datum(function() { return {year: parseInt(this.dataset.year)}; });

d3.select("#show-times")
  .on("click", (b) => {
    if (capAtYear === 0) {
      setCap({{cap_at_year}});
    } else {
      setCap(0);
    }
    focusArea("");
  });


setCap({{cap_at_year}});
focusArea("");

function focusArea(areaName) {
  focus = areaName;
  areaInfos.classed("area-highlighted", a => a.id == areaName);

  currentStack = stacking(areaGraph);
  path.data(currentStack)
    .transition()
    .duration(400)
    .attr("d", graphShapes);
  yearMarks
    .classed("capped", d => d.year < capAtYear)
    .transition()
    .duration(400)
    .style("top", d => yD(new Date(d.year,0,1)) + "px")
    .style("left", d => x(currentStack[0][Math.max(0, d.year - bounds[0] - 1)][0] * .5 + currentStack[0][d.year - bounds[0]][0] *.5) + "px");
  products
    .classed("capped", d => d.year < capAtYear)
    .transition()
    .duration(400)
    .style("top", d => yD(d.date) + "px")
    .style("left", d => {
      const pos = (d.num % 4 + 1.0) / 5.0;
      const yearStart = yD(new Date(d.year,0,1));
      const yePr = Math.abs((yD(d.date) - yearStart) / (yD(new Date(d.year,11,31)) - yearStart));
      const space0 = currentStack[areaNums.get(d.area) || 0][d.year - bounds[0]];
      const space1 = yePr > .5
        ? currentStack[areaNums.get(d.area) || 0][Math.min(years.length - 1, d.year - bounds[0] + 1)]
        : currentStack[areaNums.get(d.area) || 0][Math.max(0, d.year - bounds[0] - 1)];
      const yMix = Math.abs(yePr - .5);
      return x(
        (1.0 - pos) * ((1-yMix) * space0[0] + yMix * space1[0])
        + pos * ((1-yMix) * space0[1] + yMix * space1[1])) + "px";
    });
  if (areaName) expandProduct(null, null);
}

function expandProduct(event, product) {
  d3.select(expandedProduct).classed("expanded", false);
  d3.select(this).classed("expanded", true);
  expandedProduct = this;
  if (product) focusArea("");
}

function setCap(cap) {
  capAtYear = cap;
  svg.attr("height", Math.min({{height}}, yD(new Date(capAtYear,0,1))+30));
}

</script>