---
layout: default
title: Benjamin Bisping's Research
permalink: /
weight: 1
---


{% assign products = site.data.products | sort: "date" | reverse %}

{% assign product_years = products | group_by_exp: "prod", "prod.date | truncate: 4, ''" %}

<div id="product-canvas" style="width: 1140px; height: 3000px; position: relative;">

<svg width="1140" height="3000" id="research-history"></svg>

{% for y in product_years %}
  {% assign products_by_area = y.items | group_by: "area" %}
  {% for a in site.data.areas %}
    {% assign prods = products_by_area | find: 'name', a.id %}
    {% assign num_in_group = 0 %}
    {% for p in prods.items %}
      {% assign num_in_group = num_in_group | plus: 1 %}
      {% include elements/product.html product=p num=num_in_group%}
    {% endfor %}
  {% endfor %}
{% endfor %}

</div>


<script type="module">

import * as d3 from "https://cdn.skypack.dev/d3@7";

const width=1140;
const height=3000;

const areas = {{site.data.areas | jsonify }};
const areaNums = new Map(d3.map(d3.range(0, areas.length), (i => [areas[i].id, i])));

console.log(areaNums);

const products = d3.selectAll("div.product");
// fetch data from dom
products.datum(function() { return {
  date: new Date(this.dataset.date),
  year: parseInt(this.dataset.year),
  area: this.dataset.area
}; });
const productsData = products.data();


const bounds = d3.extent(productsData, d => d.year);
const years = d3.range(bounds[0], bounds[1] + 1);

const productsByYear = d3.group(productsData, d => d.year);

const areaNames = d3.map(areas, a => a.id);
console.log(areaNames);

console.log(productsByYear.get(2020));
const areaGraph = d3.map(
  years,
  y => {
    const row = d3.rollup(productsByYear.get(y) || [], v => v.length, p => p.area);
    row['year'] = y
    return row;
  });

console.log(areaGraph);

let focus = '';

const stacking = d3.stack()
  .keys(areaNames)
  .value((d, k) => {
    const val = d.get(k) || 0.2;
    return val * (k == focus ? 1.5 : 1.0);
  })
  .offset(d3.stackOffsetSilhouette);

console.log(bounds)

const x = d3.scaleLinear([-10,10], [0, width]);
const y = d3.scaleLinear([bounds[0]-1, bounds[1]+1], [height,0]);
const yD = d3.scaleTime([new Date(bounds[0]-1,1,1), new Date(bounds[1]+1,12,31)], [height,0]);

const graphShapes = d3.area()
  .x0(d => x(d[0]))
  .x1(d => x(d[1]))
  .y(d => y(d.data.year))
  .curve(d3.curveNatural);

const svg = d3.select("#research-history");

let currentStack = stacking(areaGraph);

console.log(currentStack);
const path = svg.selectAll("path")
  .data(currentStack)
  .join("path")
    .attr("d", graphShapes)
    .attr("fill", (d,i) => d3.schemeCategory10[i])
    .on("click", (e,d) => focusArea(d.key));

const yearMarks = svg.selectAll(".graph-year")
  .data(years)
  .enter()
    .append("text")
    .attr("text-anchor", "end")
    .text(d => d)

focusArea("");

function focusArea(areaName) {
  focus = areaName;
  currentStack = stacking(areaGraph);
  path.data(currentStack)
    .transition()
    .duration(400)
    .attr("d", graphShapes);
  yearMarks
    .transition()
    .duration(400)
    .attr("y", d => yD(new Date(d,1,1)))
    .attr("x", d => x(currentStack[0][Math.max(0, d - bounds[0] - 1)][0] * .5 + currentStack[0][d - bounds[0]][0] *.5) - 10);
  products
    .transition()
    .duration(400)
    .style("top", d => yD(d.date) + "px")
    .style("left", d => {
      const space0 = currentStack[areaNums.get(d.area) || 0][d.year - bounds[0]];
      const space1 = currentStack[areaNums.get(d.area) || 0][Math.max(0, d.year - bounds[0] - 1)];
      return x(d3.mean(space0.concat(space1))) + "px";
    });
}

</script>