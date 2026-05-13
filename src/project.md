---
title: Project Sketches
toc: true
---

# Project Sketches

This page collects the workshop exercises: layout, inputs, zip attachments, loaders, and a small D3 chart.

## Zip contents (`gapminder.zip`)

```js
const gapminderZip = await FileAttachment("./data/gapminder.zip").zip();
```

The archive exposes file paths via **`filenames`** (handy when you are unsure of exact paths):

```js
gapminderZip.filenames
```

## Continents from the zip

```js
const continents = await gapminderZip.file("gapminder/continents.csv").csv({typed: true});
const countryNames = [...new Set(continents.map((d) => d.Entity))].sort((a, b) => a.localeCompare(b));
```

```js
Inputs.table(continents)
```

### Country names (`Entity`)

```js
import {html} from "npm:htl";

display(
  html`<ul style="column-count:2;column-gap:2rem;max-height:32rem;overflow:auto;padding-left:1.1rem">${countryNames.map(
    (name) => html`<li>${name}</li>`
  )}</ul>`
);
```

## 2010 life expectancy & GDP (loaders)

Loaders `life-2010.csv.js` and `gdp-2010.csv.js` fetch Gapminder CSVs, keep **2010** rows, and emit CSV to stdout at build time (see `launches.csv.js` for the same pattern).

```js
const life2010 = await FileAttachment("./data/life-2010.csv").csv({typed: true});
const gdp2010 = await FileAttachment("./data/gdp-2010.csv").csv({typed: true});
```

```js
Inputs.table(life2010.slice(0, 12))
```

```js
Inputs.table(gdp2010.slice(0, 12))
```

## Scatter: GDP (log) vs life expectancy

```js
const lifeByKey = new Map(
  life2010.filter((d) => d.Code).map((d) => [`${d.Entity}|${d.Code}`, d])
);

const mergedRaw = gdp2010
  .filter((d) => d.Code)
  .map((g) => {
    const key = `${g.Entity}|${g.Code}`;
    const l = lifeByKey.get(key);
    if (!l) return null;
    return {
      entity: g.Entity,
      code: g.Code,
      gdp: g["GDP per capita"],
      life: l["Life expectancy"]
    };
  })
  .filter(Boolean);

const merged = mergedRaw.filter(
  (d) => d.gdp > 0 && d.life != null && Number.isFinite(d.gdp) && Number.isFinite(d.life)
);
```

```js
const colorInput = Inputs.radio(["#31688e", "#e98c6a", "#35b779"], {label: "Dot color"});
const dotColor = Generators.input(colorInput);
```

```js
import * as d3 from "npm:d3";

function gapminderScatter(data, {width, fill}) {
  const height = 420;
  const margin = {top: 28, right: 18, bottom: 48, left: 56};
  const innerW = width - margin.left - margin.right;
  const innerH = height - margin.top - margin.bottom;
  const xDomain = d3.extent(data, (d) => d.gdp);
  const yDomain = d3.extent(data, (d) => d.life);
  const x = d3
    .scaleLog()
    .domain([Math.max(xDomain[0], 50), xDomain[1]])
    .range([0, innerW])
    .nice();
  const y = d3.scaleLinear().domain(yDomain).nice().range([innerH, 0]);
  const svg = d3
    .create("svg")
    .attr("viewBox", `0 0 ${width} ${height}`)
    .attr("width", width)
    .attr("height", height)
    .style("max-width", "100%")
    .style("height", "auto");
  const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);
  g.append("g")
    .attr("transform", `translate(0,${innerH})`)
    .call(d3.axisBottom(x).ticks(6, "~s"))
    .call((axis) => axis.selectAll("text").attr("fill", "currentColor"))
    .call((axis) => axis.selectAll("path,line").attr("stroke", "currentColor"));
  g.append("g")
    .call(d3.axisLeft(y))
    .call((axis) => axis.selectAll("text").attr("fill", "currentColor"))
    .call((axis) => axis.selectAll("path,line").attr("stroke", "currentColor"));
  g.selectAll("circle")
    .data(data)
    .join("circle")
    .attr("cx", (d) => x(d.gdp))
    .attr("cy", (d) => y(d.life))
    .attr("r", 4.5)
    .attr("fill", fill)
    .attr("fill-opacity", 0.88)
    .append("title")
    .text((d) => `${d.entity}\nGDP: ${d.gdp}\nLife: ${d.life}`);
  g.append("text")
    .attr("x", innerW / 2)
    .attr("y", innerH + 40)
    .attr("text-anchor", "middle")
    .attr("fill", "currentColor")
    .attr("font-size", 12)
    .text("GDP per capita (log scale)");
  g.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -innerH / 2)
    .attr("y", -44)
    .attr("text-anchor", "middle")
    .attr("fill", "currentColor")
    .attr("font-size", 12)
    .text("Life expectancy (years)");
  return svg.node();
}
```

${resize((width) => gapminderScatter(merged, {width, fill: dotColor}))}

## Image inside `<details>`

```js
import {html} from "npm:htl";

const sketchUrl = await FileAttachment("./data/sketch.svg").url();
display(
  html`<details>
    <summary>Show bundled SVG sketch</summary>
    <img src="${sketchUrl}" width="240" height="120" alt="Simple placeholder sketch" />
  </details>`
);
```

## 3 × 3 grid (middle row spans three columns)

<div class="grid grid-cols-3">
  <div class="card"><span class="red">A</span></div>
  <div class="card"><span class="yellow">B</span></div>
  <div class="card"><span class="green">C</span></div>
  <div class="card grid-colspan-3"><span class="blue">D</span></div>
  <div class="card"><span class="muted">E</span></div>
  <div class="card"><span class="red">F</span></div>
  <div class="card"><span class="yellow">G</span></div>
</div>

## First *n* words (slider + `FileAttachment` text)

```js
const prose = await FileAttachment("./data/lorem.txt").text();
const words = prose.trim().split(/\s+/).filter(Boolean);
```

```js
const wordSlider = Inputs.range([0, words.length], {step: 1, label: "Words to show", value: 24});
const nWords = Generators.input(wordSlider);
```

${wordSlider}

${words.slice(0, nWords).join(" ")}
