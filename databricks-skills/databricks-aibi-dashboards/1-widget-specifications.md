# Widget Specifications

Detailed JSON patterns for each AI/BI dashboard widget type.

## Widget Naming Convention (CRITICAL)

- `widget.name`: alphanumeric + hyphens + underscores ONLY (no spaces, parentheses, colons)
  - **Maximum 60 characters** - longer names cause validation errors
- `frame.title`: human-readable name (any characters allowed)
- `widget.queries[0].name`: always use `"main_query"`

## Version Requirements

| Widget Type | Version |
|-------------|---------|
| counter | 2 |
| table | 2 |
| filter-multi-select | 2 |
| filter-single-select | 2 |
| filter-date-range-picker | 2 |
| bar | 3 |
| line | 3 |
| pie | 3 |
| combo | 1 |
| text | N/A (no spec block) |

---

## Text (Headers/Descriptions)

- **CRITICAL: Text widgets do NOT use a spec block!**
- Use `multilineTextboxSpec` directly on the widget
- Supports markdown: `#`, `##`, `###`, `**bold**`, `*italic*`
- **CRITICAL: Multiple items in the `lines` array are concatenated on a single line, NOT displayed as separate lines!**
- For title + subtitle, use **separate text widgets** at different y positions

```json
// CORRECT: Separate widgets for title and subtitle
{
  "widget": {
    "name": "title",
    "multilineTextboxSpec": {
      "lines": ["## Dashboard Title"]
    }
  },
  "position": {"x": 0, "y": 0, "width": 6, "height": 1}
},
{
  "widget": {
    "name": "subtitle",
    "multilineTextboxSpec": {
      "lines": ["Description text here"]
    }
  },
  "position": {"x": 0, "y": 1, "width": 6, "height": 1}
}

// WRONG: Multiple lines concatenate into one line!
{
  "widget": {
    "name": "title-widget",
    "multilineTextboxSpec": {
      "lines": ["## Dashboard Title", "Description text here"]  // Becomes "## Dashboard TitleDescription text here"
    }
  },
  "position": {"x": 0, "y": 0, "width": 6, "height": 2}
}
```

---

## Counter (KPI)

- `version`: **2** (NOT 3!)
- `widgetType`: "counter"
- **Percent values must be 0-1** in the data (not 0-100)

### Number Formatting

Use the `format` property in `encodings.value` to control display:

```json
// Currency - displays "$1.2M" instead of "1234567"
"encodings": {
  "value": {
    "fieldName": "revenue",
    "displayName": "Total Revenue",
    "format": {
      "type": "number-currency",
      "currencyCode": "USD",
      "abbreviation": "compact",
      "decimalPlaces": {"type": "max", "places": 2}
    }
  }
}

// Percent - displays "45.2%" (data must be 0-1)
"encodings": {
  "value": {
    "fieldName": "conversion_rate",
    "displayName": "Conversion Rate",
    "format": {
      "type": "number-percent",
      "decimalPlaces": {"type": "max", "places": 1}
    }
  }
}

// Plain number with formatting
"encodings": {
  "value": {
    "fieldName": "order_count",
    "displayName": "Orders",
    "format": {
      "type": "number",
      "decimalPlaces": {"type": "max", "places": 0}
    }
  }
}
```

**Two patterns for counters:**

**Pattern 1: Pre-aggregated dataset (1 row, no filters)**
- Dataset returns exactly 1 row
- Use `"disaggregated": true` and simple field reference
- Field `name` matches dataset column directly

```json
{
  "widget": {
    "name": "total-revenue",
    "queries": [{
      "name": "main_query",
      "query": {
        "datasetName": "summary_ds",
        "fields": [{"name": "revenue", "expression": "`revenue`"}],
        "disaggregated": true
      }
    }],
    "spec": {
      "version": 2,
      "widgetType": "counter",
      "encodings": {
        "value": {"fieldName": "revenue", "displayName": "Total Revenue"}
      },
      "frame": {"showTitle": true, "title": "Total Revenue"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 3}
}
```

**Pattern 2: Aggregating widget (multi-row dataset, supports filters)**
- Dataset returns multiple rows (e.g., grouped by a filter dimension)
- Use `"disaggregated": false` and aggregation expression
- **CRITICAL**: Field `name` MUST match `fieldName` exactly (e.g., `"sum(spend)"`)

```json
{
  "widget": {
    "name": "total-spend",
    "queries": [{
      "name": "main_query",
      "query": {
        "datasetName": "by_category",
        "fields": [{"name": "sum(spend)", "expression": "SUM(`spend`)"}],
        "disaggregated": false
      }
    }],
    "spec": {
      "version": 2,
      "widgetType": "counter",
      "encodings": {
        "value": {"fieldName": "sum(spend)", "displayName": "Total Spend"}
      },
      "frame": {"showTitle": true, "title": "Total Spend"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 3}
}
```

---

## Table

- `version`: **2** (NOT 1 or 3!)
- `widgetType`: "table"
- **Columns only need `fieldName` and `displayName`** - no other properties!
- Use `"disaggregated": true` for raw rows

```json
{
  "widget": {
    "name": "details-table",
    "queries": [{
      "name": "main_query",
      "query": {
        "datasetName": "details_ds",
        "fields": [
          {"name": "name", "expression": "`name`"},
          {"name": "value", "expression": "`value`"}
        ],
        "disaggregated": true
      }
    }],
    "spec": {
      "version": 2,
      "widgetType": "table",
      "encodings": {
        "columns": [
          {"fieldName": "name", "displayName": "Name"},
          {"fieldName": "value", "displayName": "Value"}
        ]
      },
      "frame": {"showTitle": true, "title": "Details"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 6, "height": 6}
}
```

---

## Line / Bar Charts

- `version`: **3**
- `widgetType`: "line" or "bar"
- Use `x`, `y`, optional `color` encodings
- `scale.type`: `"temporal"` (dates), `"quantitative"` (numbers), `"categorical"` (strings)
- Use `"disaggregated": true` with pre-aggregated dataset data

**Multiple Lines - Two Approaches:**

1. **Multi-Y Fields** (different metrics on same chart):
```json
"y": {
  "scale": {"type": "quantitative"},
  "fields": [
    {"fieldName": "sum(orders)", "displayName": "Orders"},
    {"fieldName": "sum(returns)", "displayName": "Returns"}
  ]
}
```

2. **Color Grouping** (same metric split by dimension):
```json
"y": {"fieldName": "sum(revenue)", "scale": {"type": "quantitative"}},
"color": {"fieldName": "region", "scale": {"type": "categorical"}, "displayName": "Region"}
```

### Color Encoding

**Two types of color scales:**

1. **Categorical** (discrete colors for groups):
```json
"color": {"fieldName": "priority", "scale": {"type": "categorical"}, "displayName": "Priority"}
```

2. **Quantitative** (gradient based on numeric value - for heatmap-style effects):
```json
"color": {"fieldName": "sum(revenue)", "scale": {"type": "quantitative"}, "displayName": "Revenue"}
```

> **CRITICAL**: Color scale for bar/line/area/scatter/pie ONLY supports these properties:
> - `type`: required ("categorical", "quantitative", or "temporal")
> - `sort`: optional
>
> **DO NOT** add `scheme`, `colorRamp`, or `mappings` - these only work for choropleth-map widgets and will cause errors on other chart types.

### Bar Chart Modes

Choose based on your visualization goal:

| Mode | When to Use | Configuration |
|------|-------------|---------------|
| **Stacked** (default) | Show total + composition breakdown | No `mark` field |
| **Grouped** | Compare values across categories side-by-side | Add `"mark": {"layout": "group"}` |

**Stacked mode** (default - bars stack on top of each other):
```json
"spec": {
  "version": 3,
  "widgetType": "bar",
  "encodings": {
    "x": {"fieldName": "daily(date)", "scale": {"type": "temporal"}},
    "y": {"fieldName": "sum(revenue)", "scale": {"type": "quantitative"}},
    "color": {"fieldName": "region", "scale": {"type": "categorical"}}
  }
  // No "mark" field = stacked
}
```

**Grouped mode** (bars side-by-side for comparison):
```json
"spec": {
  "version": 3,
  "widgetType": "bar",
  "encodings": {
    "x": {"fieldName": "category", "scale": {"type": "categorical"}},
    "y": {"fieldName": "sum(revenue)", "scale": {"type": "quantitative"}},
    "color": {"fieldName": "region", "scale": {"type": "categorical"}}
  },
  "mark": {"layout": "group"}
}
```

> **Tip**: For grouped bars with a time series X-axis, use weekly or monthly aggregation (`DATE_TRUNC("WEEK", date)`) for readability instead of daily.

## Pie Chart

- `version`: **3**
- `widgetType`: "pie"
- `angle`: quantitative aggregate
- `color`: categorical dimension
- Limit to 3-8 categories for readability

---

## Combo Chart (Bar + Line)

Combo charts display two visualization types on the same widget - bars for one metric and a line for another. Useful for showing related metrics with different representations (e.g., revenue as bars + growth rate as a line).

- `version`: **1**
- `widgetType`: "combo"
- `y.primary`: bar chart fields
- `y.secondary`: line chart fields
- **Important**: Both primary and secondary should have similar scales since they share the Y-axis

```json
{
  "widget": {
    "name": "revenue-and-growth",
    "queries": [{
      "name": "main_query",
      "query": {
        "datasetName": "metrics_ds",
        "fields": [
          {"name": "daily(date)", "expression": "DATE_TRUNC(\"DAY\", `date`)"},
          {"name": "sum(revenue)", "expression": "SUM(`revenue`)"},
          {"name": "avg(growth_rate)", "expression": "AVG(`growth_rate`)"}
        ],
        "disaggregated": false
      }
    }],
    "spec": {
      "version": 1,
      "widgetType": "combo",
      "encodings": {
        "x": {
          "fieldName": "daily(date)",
          "scale": {"type": "temporal"}
        },
        "y": {
          "scale": {"type": "quantitative"},
          "primary": {
            "fields": [
              {"fieldName": "sum(revenue)", "displayName": "Revenue ($)"}
            ]
          },
          "secondary": {
            "fields": [
              {"fieldName": "avg(growth_rate)", "displayName": "Growth Rate"}
            ]
          }
        },
        "label": {"show": false}
      },
      "frame": {"title": "Revenue & Growth Rate", "showTitle": true}
    }
  },
  "position": {"x": 0, "y": 0, "width": 6, "height": 5}
}
```
