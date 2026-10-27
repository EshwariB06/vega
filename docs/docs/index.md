---
layout: docs
title: Documentation
permalink: /docs/index.html
---

The **Vega** visualization grammar provides basic building blocks for a wide variety of visualization designs. This page provides documentation of [Vega JSON specifications](#specification) and [JavaScript API](#api).

To start learning Vega, we recommend first working through the introductory **[Let's Make A Bar Chart tutorial](../tutorials/bar-chart)** and exploring the [example gallery](../examples), then digging into the documentation. These pages document _Vega version 3.0 and later_; if you are familiar with Vega 2.x, you might begin with the [**Vega 2 porting guide**](porting-guide).


## <a name="specification"></a>Specification Reference

A **Vega specification** is a [JSON](http://en.wikipedia.org/wiki/JSON) object that describes an interactive visualization design. A [specification](specification) consists of basic properties (such as the width and height of the view) and definitions for the [data](data) to visualize, [scales](scales) that map data values to visual values, [axes](axes) and [legends](legends) that visualize these scales, cartographic [projections](projections) for drawing maps, graphical [marks](marks) such as rectangles, lines, and symbols to visually represent data, and [signals](signals) to process user input and modify the visualization in response.

| :----------------------------- | :----------- |
| [Specification](specification) | Overview of a full Vega specification, including sizing and metadata.|
| [Config](config)               |{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "description": "Supply-chain flow: stages as columns (headers on top), one node per country per stage, colour per country",
  "autosize": { "type": "fit", "contains": "padding" },
  "padding": { "left": 10, "right": 10, "top": 10, "bottom": 10 },

  "signals": [
    { "name": "nodeW", "value": 16 },
    { "name": "gap", "value": 14 },
    { "name": "headerH", "value": 34 },
    { "name": "labelSpace", "value": 90 },
    { "name": "colW", "update": "(width - nodeW - labelSpace) / max(length(data('stages')) - 1, 1)" },
    { "name": "k", "update": "(height - headerH - gap * data('meta')[0].maxN) / data('meta')[0].maxTot" }
  ],

  "data": [
    { "name": "dataset" },

    {
      "name": "stages",
      "values": [
        { "stage": "Extraction", "idx": 0 },
        { "stage": "Mineral Processing", "idx": 1 },
        { "stage": "Intermediate 1", "idx": 2 },
        { "stage": "Intermediate 2", "idx": 3 },
        { "stage": "Battery Grade Material", "idx": 4 },
        { "stage": "pCAM", "idx": 5 },
        { "stage": "CAM", "idx": 6 }
      ]
    },

    {
      "name": "countryColors",
      "values": [
        { "country": "Australia", "color": "#1f77b4" },
        { "country": "China", "color": "#2ca02c" },
        { "country": "Indonesia", "color": "#ff7f0e" },
        { "country": "Chile", "color": "#d62728" },
        { "country": "Japan", "color": "#9467bd" },
        { "country": "South Korea", "color": "#8c564b" },
        { "country": "Germany", "color": "#e377c2" },
        { "country": "United States", "color": "#17becf" }
      ]
    },

    {
      "name": "links",
      "source": "dataset",
      "transform": [
        { "type": "filter", "expr": "isValid(datum.Source_Conc) && isValid(datum.Target_Conc)" },
        { "type": "formula", "as": "sStage", "expr": "trim(split(datum.Source_Conc, '|')[0])" },
        { "type": "formula", "as": "sCountry", "expr": "trim(split(datum.Source_Conc, '|')[1])" },
        { "type": "formula", "as": "tStage", "expr": "trim(split(datum.Target_Conc, '|')[0])" },
        { "type": "formula", "as": "tCountry", "expr": "trim(split(datum.Target_Conc, '|')[1])" },
        { "type": "formula", "as": "sKey", "expr": "datum.sStage + '|' + datum.sCountry" },
        { "type": "formula", "as": "tKey", "expr": "datum.tStage + '|' + datum.tCountry" },
        { "type": "formula", "as": "val", "expr": "+datum['Flow Value']" },
        {
          "type": "aggregate",
          "groupby": ["sKey", "tKey", "sStage", "sCountry", "tStage", "tCountry"],
          "ops": ["sum"], "fields": ["val"], "as": ["val"]
        },
        { "type": "filter", "expr": "datum.val > 0" },
        { "type": "lookup", "from": "stages", "key": "stage", "fields": ["sStage"], "values": ["idx"], "as": ["sIdx"] },
        { "type": "lookup", "from": "stages", "key": "stage", "fields": ["tStage"], "values": ["idx"], "as": ["tIdx"] },
        { "type": "stack", "groupby": ["sKey"], "field": "val", "sort": { "field": "tKey" }, "as": ["s0", "s1"] },
        { "type": "stack", "groupby": ["tKey"], "field": "val", "sort": { "field": "sKey" }, "as": ["t0", "t1"] },
        { "type": "lookup", "from": "countryColors", "key": "country", "fields": ["sCountry"], "values": ["color"], "as": ["sColor"], "default": "#999999" }
      ]
    },

    {
      "name": "nodesS",
      "source": "links",
      "transform": [
        { "type": "aggregate", "groupby": ["sKey", "sStage", "sCountry", "sIdx"], "ops": ["sum"], "fields": ["val"], "as": ["val"] },
        { "type": "project", "fields": ["sKey", "sStage", "sCountry", "sIdx", "val"], "as": ["key", "stage", "country", "idx", "val"] }
      ]
    },
    {
      "name": "nodesT",
      "source": "links",
      "transform": [
        { "type": "aggregate", "groupby": ["tKey", "tStage", "tCountry", "tIdx"], "ops": ["sum"], "fields": ["val"], "as": ["val"] },
        { "type": "project", "fields": ["tKey", "tStage", "tCountry", "tIdx", "val"], "as": ["key", "stage", "country", "idx", "val"] }
      ]
    },

    {
      "name": "nodesBase",
      "source": ["nodesS", "nodesT"],
      "transform": [
        { "type": "aggregate", "groupby": ["key", "stage", "country", "idx"], "ops": ["max"], "fields": ["val"], "as": ["val"] },
        {
          "type": "window",
          "groupby": ["stage"],
          "sort": { "field": "country" },
          "ops": ["row_number", "sum"],
          "fields": [null, "val"],
          "as": ["rank", "cum"],
          "frame": [null, 0]
        }
      ]
    },
    {
      "name": "stageStats",
      "source": "nodesBase",
      "transform": [
        { "type": "aggregate", "groupby": ["stage"], "ops": ["sum", "count"], "fields": ["val", "val"], "as": ["tot", "n"] }
      ]
    },
    {
      "name": "meta",
      "source": "stageStats",
      "transform": [
        { "type": "aggregate", "ops": ["max", "max"], "fields": ["tot", "n"], "as": ["maxTot", "maxN"] }
      ]
    },

    {
      "name": "nodes",
      "source": "nodesBase",
      "transform": [
        { "type": "formula", "as": "nx", "expr": "datum.idx * colW" },
        { "type": "formula", "as": "ny", "expr": "headerH + (datum.cum - datum.val) * k + (datum.rank - 1) * gap" },
        { "type": "formula", "as": "nh", "expr": "max(datum.val * k, 1)" },
        { "type": "lookup", "from": "countryColors", "key": "country", "fields": ["country"], "values": ["color"], "as": ["color"], "default": "#999999" }
      ]
    },

    {
      "name": "linkPaths",
      "source": "links",
      "transform": [
        { "type": "lookup", "from": "nodes", "key": "key", "fields": ["sKey"], "values": ["ny"], "as": ["sNy"] },
        { "type": "lookup", "from": "nodes", "key": "key", "fields": ["tKey"], "values": ["ny"], "as": ["tNy"] },
        { "type": "formula", "as": "x0", "expr": "datum.sIdx * colW + nodeW" },
        { "type": "formula", "as": "x1", "expr": "datum.tIdx * colW" },
        { "type": "formula", "as": "xm", "expr": "(datum.x0 + datum.x1) / 2" },
        { "type": "formula", "as": "yA", "expr": "datum.sNy + datum.s0 * k" },
        { "type": "formula", "as": "yB", "expr": "datum.yA + datum.val * k" },
        { "type": "formula", "as": "yC", "expr": "datum.tNy + datum.t0 * k" },
        { "type": "formula", "as": "yD", "expr": "datum.yC + datum.val * k" },
        {
          "type": "formula",
          "as": "path",
          "expr": "'M' + datum.x0 + ',' + datum.yA + ' C' + datum.xm + ',' + datum.yA + ' ' + datum.xm + ',' + datum.yC + ' ' + datum.x1 + ',' + datum.yC + ' L' + datum.x1 + ',' + datum.yD + ' C' + datum.xm + ',' + datum.yD + ' ' + datum.xm + ',' + datum.yB + ' ' + datum.x0 + ',' + datum.yB + ' Z'"
        }
      ]
    }
  ],

  "marks": [
    {
      "type": "path",
      "from": { "data": "linkPaths" },
      "encode": {
        "enter": {
          "tooltip": {
            "signal": "{'From': datum.sStage + ' | ' + datum.sCountry, 'To': datum.tStage + ' | ' + datum.tCountry, 'Flow': format(datum.val, ',.1f')}"
          }
        },
        "update": {
          "path": { "field": "path" },
          "fill": { "field": "sColor" },
          "fillOpacity": { "value": 0.45 }
        },
        "hover": { "fillOpacity": { "value": 0.8 } }
      }
    },
    {
      "type": "rect",
      "from": { "data": "nodes" },
      "encode": {
        "enter": {
          "tooltip": {
            "signal": "{'Stage': datum.stage, 'Country': datum.country, 'Total': format(datum.val, ',.1f')}"
          }
        },
        "update": {
          "x": { "field": "nx" },
          "y": { "field": "ny" },
          "width": { "signal": "nodeW" },
          "height": { "field": "nh" },
          "fill": { "field": "color" }
        }
      }
    },
    {
      "type": "text",
      "from": { "data": "nodes" },
      "encode": {
        "update": {
          "x": { "signal": "datum.nx + nodeW + 4" },
          "y": { "signal": "datum.ny + datum.nh / 2" },
          "text": { "field": "country" },
          "baseline": { "value": "middle" },
          "fontSize": { "value": 10 },
          "fill": { "value": "#333333" }
        }
      }
    },
    {
      "type": "text",
      "from": { "data": "stages" },
      "encode": {
        "update": {
          "x": { "signal": "datum.idx * colW + nodeW / 2" },
          "y": { "value": 12 },
          "text": { "field": "stage" },
          "align": { "value": "center" },
          "fontSize": { "value": 12 },
          "fontWeight": { "value": "bold" },
          "fill": { "value": "#333333" }
        }
      }
    }
  ]
}
 Configure defaults for visual encoding choices.|
| [Data](data)                   | Define, load, and parse data to visualize.|
| [Transforms](transforms)       | Apply data transforms (filter, sort, aggregate, layout) prior to visualization.|
| [Triggers](triggers)           | Modify data sets or mark properties in response to signal values.|
| [Projections](projections)     | Cartographic projections to map (longitude, latitude) data.|
| [Scales](scales)               | Map data values (numbers, strings) to visual properties (coordinates, colors, sizes).|
| [Schemes](schemes)             | Color schemes that can be used as scale ranges.|
| [Axes](axes)                   | Visualize scale mappings for spatial encodings using coordinate axes.|
| [Legends](legends)             | Visualize scale mappings for color, shape and size encodings.|
| [Title](title)                 | Specify a chart title for a visualization.|
| [Marks](marks)                 | Visually encode data with graphical marks such as rectangles, lines, and symbols.|
| [Signals](signals)             | Dynamic variables that can drive interactive updates.|
| [Event Streams](event-streams) | Define input event streams to specify interactions.|
| [Expressions](expressions)     | Express custom calculations over data and signals.|
| [Layout](layout)               | Perform grid layout for a collection of group marks.|
| [Types](types)                 | Documentation of recurring parameter types.|[supply_chain_sankey_deneb.json](https://github.com/user-attachments/files/34011142/supply_chain_sankey_deneb.json)



## <a name="api"></a>Vega API Reference

Vega also provides a **JavaScript runtime API**, responsible for parsing JSON specifications to produce interactive views backed by a reactive dataflow graph of data processing operators. For more on deploying Vega, see the [usage instructions](../usage).

| :----------------------------- | :------------------ |
| [Parser](api/parser)           | Parse Vega JSON specifications to dataflow descriptions. |
| [View](api/view)               | Create interactive views from dataflow descriptions. |
| [Locale](api/locale)           | Use number and date formats for a specific locale. |
| [Extensibility](api/extensibility) | Extend Vega with new projections, scales, color schemes, or data transforms. |
| [Statistics](api/statistics)   | Statistical methods used by Vega. |
| [Time](api/time)               | Utility methods for date-time values. |
| [Util](api/util)               | General JavaScript utilities. |
| [Debugging](api/debugging)     | A guide to debugging Vega visualizations at runtime. |
