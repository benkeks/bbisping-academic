---
layout: start
title: Benjamin Bisping's Research
permalink: /
weight: 1
---

{% assign width = 1140 %}
{% assign height = 4000 %}
{% assign cap_at_year = 2019 %}

{% assign products = site.data.products | sort: "date" | reverse %}
{% assign positions = site.data.positions | sort: "start" %}

{% assign product_years = products | group_by_exp: "prod", "prod.date | truncate: 4, ''" %}

{% for ar in site.data.areas %}
<div class="area-info" id="area-{{ar.id}}" data-id="{{ar.id}}">
  <h2 class="area-name" style="border-bottom: 3px solid {{ar.color}}">{{ar.name}}</h2>
  <p class="area-description">{{ar.description | markdownify}}</p>
</div>
{% endfor %}

<div id="product-canvas" class="justify-content-center" style="width: {{width}}px; position: relative;">

{% for position in positions %}
<div
  id="position-{{ position.id }}"
  class="cv-position"
  data-id="{{ position.id }}"
  data-start="{{ position.start }}"
  data-end="{{ position.end }}"
  data-index="{{ forloop.index0 }}"
  tabindex="0"
  role="button"
  aria-expanded="false">
  <div class="cv-position-handle">{{ position.short_title | default: position.organization | default: position.title }}</div>
  <div class="cv-position-card">
    {% if position.kind %}
    <div class="cv-position-kind">{{ position.kind }}</div>
    {% endif %}
    <div class="cv-position-title-row">
      {% if position.url %}
      <a href="{{ position.url }}" class="cv-position-title">{{ position.title }}</a>
      {% else %}
      <span class="cv-position-title">{{ position.title }}</span>
      {% endif %}
    </div>
    {% if position.organization %}
    <div class="cv-position-organization">
      {% if position.organization_url %}
      <a href="{{ position.organization_url }}">{{ position.organization }}</a>
      {% else %}
      {{ position.organization }}
      {% endif %}
    </div>
    {% endif %}
    <div class="cv-position-date-range">
      {{ position.start | date: "%Y-%m" }} – {% if position.end %}{{ position.end | date: "%Y-%m" }}{% else %}present{% endif %}
    </div>
    {% if position.location %}
    <div class="cv-position-location">{{ position.location }}</div>
    {% endif %}
    {% if position.description %}
    <div class="cv-position-description">{{ position.description | markdownify }}</div>
    {% endif %}
  </div>
</div>
{% endfor %}

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

<button type="button" id="show-times" class="btn btn-light">Show student times</button>
</div>

<div style="text-align: center;">
  <a href="/cv" title="If you look for a printable (boring) version of my curriculum vitae">Tabular CV</a>
</div>

<script type="module">

const width={{width}};
const height={{height}};
let capAtYear={{cap_at_year}};

function parseYearMonth(value, endOfPeriod = false) {
  if (!value) return null;
  const [yearString, monthString] = value.split('-');
  const year = parseInt(yearString, 10);
  const month = monthString ? parseInt(monthString, 10) : (endOfPeriod ? 12 : 1);
  return endOfPeriod ? new Date(year, month, 0) : new Date(year, month - 1, 1);
}

const areas = {{site.data.areas | jsonify }};
const areaNums = new Map(d3.map(d3.range(0, areas.length), (i => [areas[i].id, i])));

const areaInfos = d3.selectAll(".area-info")
  .datum(function() { return this.dataset; });

const positions = d3.selectAll(".cv-position")
  .datum(function() { return {
    id: this.dataset.id,
    start: parseYearMonth(this.dataset.start),
    end: parseYearMonth(this.dataset.end, true),
    index: parseInt(this.dataset.index, 10)
  }; })
  .on("click", function (e) {
    if (e.target.closest('a')) return;
    navigateFocus(this.id, "click");
  })
  .on("focus", function (e) {
    if (e.target.closest('a')) return;
    navigateFocus(this.id, "focus");
  })
  .on("keydown", function (e) {
    if (e.key === "Enter" || e.key === " ") {
      e.preventDefault();
      navigateFocus(this.id, "keyboard");
    }
  });
const positionsData = positions.data();
const positionHandleSpread = positionsData.length > 0 ? Math.min(280, 84 + positionsData.length * 34) : 0;

const products = d3.selectAll("div.product")
  .on("click", function (e, d) { navigateFocus(this.id, "click"); })
  .on("focus", function (e, d) { navigateFocus(this.id, "focus"); })
  .on("keydown", function (e, d) {
    if (e.key === "Enter" || e.key === " ") { e.preventDefault(); navigateFocus(this.id, "keyboard"); }
  });

const innerElements = products.selectAll("*")
  .on("focus", function (e, d) { navigateFocus(this.closest('.product').id, "focus"); });

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
    row.forEach((v, k) => row.set(k, .2 + v ** .8));
    row['year'] = y
    return row;
  });

let focus = '';
let expandedProduct = null;
let expandedPosition = '';

const stacking = d3.stack()
  .keys(areaNames)
  .value((d, k) => {
    const val = d.get(k) || 0.2;
    return val * (k == focus ? 1.5 : 1.0);
  })
  .offset(d3.stackOffsetWiggle);

const x = d3.scaleLinear([-8.5,8.5], [0, width]);
const y = d3.scaleLinear([bounds[0], bounds[1] + .5], [height,0]);
const yD = d3.scaleTime([new Date(bounds[0],5,1), new Date(bounds[1],11,31)], [height,0]);
const graphEndDate = new Date(bounds[1],11,31);

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
    .on("click", (e,d) => navigateFocus("area-" + d.key, "click"))
    .on("focus", (e,d) => navigateFocus("area-" + d.key, "focus"))
    .on("keydown", (e,d) => { if (e.key === "Enter" || e.key === " ") { e.preventDefault(); navigateFocus("area-" + d.key, "keyboard"); } });

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
focusPosition("");
focusByUrl();

window.addEventListener('hashchange', focusByUrl);

let lastFocusNavigation = { key: "", at: 0 };

function navigateFocus(key, source = "click") {
  const now = Date.now();
  if (source === "click" && lastFocusNavigation.key === key && now - lastFocusNavigation.at < 350) {
    return;
  }
  if (source === "focus") {
    lastFocusNavigation = { key, at: now };
  }

  if (window.location.hash === '#' + key) {
    if (source !== "focus") {
      window.history.pushState({}, '', window.location.pathname + window.location.search);
    }
  } else {
    window.history.pushState({}, '', '#' + key);
  }
  focusByUrl();
}

function focusByUrl() {
  if (window.location.hash.startsWith('#area-')) {
    const target = window.location.hash.substring(6);
    focusPosition("");
    focusArea(target);
  } else if (window.location.hash.startsWith('#position-')) {
    const target = window.location.hash.substring(10);
    expandProduct(null, null);
    focusArea("");
    focusPosition(target);
  } else if (window.location.hash.startsWith('#product-')) {
    const target = window.location.hash;
    d3.select(target).each(expandProduct);
    focusPosition("");
    focusArea("");
  } else {
    focusPosition("");
    focusArea("");
  }
}

function layoutPositions() {
  const capStart = capAtYear === 0 ? new Date(bounds[0], 0, 1) : new Date(capAtYear, 0, 1);

  positions
    .classed("capped", d => (d.end || graphEndDate) < capStart)
    .style("top", "0px")
    .style("height", "0px")
    .each(function(d) {
      const visibleStart = d.start < capStart ? capStart : d.start;
      const startY = yD(visibleStart);
      const handlePitch = positionHandleSpread / positionsData.length;
      const startX = 4 + d.index * handlePitch;
      const svgHeight = parseFloat(svg.attr("height"));
      const clampedY = Math.min(startY, svgHeight - 12);
      const cardEl = this.querySelector(".cv-position-card");
      const cardHeight = cardEl ? cardEl.offsetHeight : 220;
      const nearBottom = (clampedY + 8 + cardHeight) > (svgHeight - 4);
      const handleWidth = Math.max(150, positionHandleSpread - startX + 56);
      const cardLeft = Math.max(10, startX + handleWidth + 8);
      const cardTop = nearBottom
        ? Math.max(8, clampedY - cardHeight - 8)
        : clampedY + 8;

      d3.select(this)
        .style("left", "0px")
        .style("--cv-position-handle-left", `${startX}px`)
        .style("--cv-position-handle-top", `${Math.max(0, clampedY)}px`)
        .style("--cv-position-handle-width", `${handleWidth}px`)
        .style("--cv-position-card-left", `${cardLeft}px`)
        .style("--cv-position-card-top", `${cardTop}px`);
    });
}

function focusPosition(positionId) {
  expandedPosition = positionId;
  positions
    .classed("expanded", d => d.id === positionId)
    .attr("aria-expanded", d => d.id === positionId ? "true" : "false");
}

function focusArea(areaName) {
  focus = areaName;
  areaInfos.classed("area-highlighted", a => a.id == areaName);

  currentStack = stacking(areaGraph);
  path.data(currentStack)
    .classed("area-focussed", a => {return a.key == areaName})
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
  layoutPositions();
  if (areaName) expandProduct(null, null);
}

function expandProduct(event, product) {
  d3.select(expandedProduct).classed("expanded", false);
  d3.select(this).classed("expanded", true);
  expandedProduct = this;
  if (product) {
    focusPosition("");
    focusArea("");
  }
}

function setCap(cap) {
  capAtYear = cap;
  svg.attr("height", Math.min({{height}}, yD(new Date(capAtYear,0,1))+30));
  layoutPositions();
}

</script>