# Filters (Global vs Page-Level)

> **CRITICAL**: Filter widgets use DIFFERENT widget types than charts!
> - Valid types: `filter-multi-select`, `filter-single-select`, `filter-date-range-picker`
> - **DO NOT** use `widgetType: "filter"` - this does not exist and will cause errors
> - Filters use `spec.version: 2`
> - **ALWAYS include `frame` with `showTitle: true`** for filter widgets

**Filter widget types:**
- `filter-date-range-picker`: for DATE/TIMESTAMP fields (date range selection)
- `filter-single-select`: categorical with single selection
- `filter-multi-select`: categorical with multiple selections (preferred for drill-down)

> **Performance note**: Global filters automatically apply `WHERE` clauses to dataset queries at runtime. You don't need to pre-filter data in your SQL - the dashboard engine handles this efficiently.

---

## Global Filters vs Page-Level Filters

| Type | Placement | Scope | Use Case |
|------|-----------|-------|----------|
| **Global Filter** | Dedicated page with `"pageType": "PAGE_TYPE_GLOBAL_FILTERS"` | Affects ALL pages that have datasets with the filter field | Cross-dashboard filtering (e.g., date range, campaign) |
| **Page-Level Filter** | Regular page with `"pageType": "PAGE_TYPE_CANVAS"` | Affects ONLY widgets on that same page | Page-specific filtering (e.g., platform filter on breakdown page only) |

**Key Insight**: A filter only affects datasets that contain the filter field. To have a filter affect only specific pages:
1. Include the filter dimension in datasets for pages that should be filtered
2. Exclude the filter dimension from datasets for pages that should NOT be filtered

---

## Filter Widget Structure

> **CRITICAL**: Do NOT use `associative_filter_predicate_group` - it causes SQL errors!
> Use a simple field expression instead.

```json
{
  "widget": {
    "name": "filter_region",
    "queries": [{
      "name": "ds_data_region",
      "query": {
        "datasetName": "ds_data",
        "fields": [
          {"name": "region", "expression": "`region`"}
        ],
        "disaggregated": false
      }
    }],
    "spec": {
      "version": 2,
      "widgetType": "filter-multi-select",
      "encodings": {
        "fields": [{
          "fieldName": "region",
          "displayName": "Region",
          "queryName": "ds_data_region"
        }]
      },
      "frame": {"showTitle": true, "title": "Region"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 2}
}
```

---

## Global Filter Example

Place on a dedicated filter page:

```json
{
  "name": "filters",
  "displayName": "Filters",
  "pageType": "PAGE_TYPE_GLOBAL_FILTERS",
  "layout": [
    {
      "widget": {
        "name": "filter_campaign",
        "queries": [{
          "name": "ds_campaign",
          "query": {
            "datasetName": "overview",
            "fields": [{"name": "campaign_name", "expression": "`campaign_name`"}],
            "disaggregated": false
          }
        }],
        "spec": {
          "version": 2,
          "widgetType": "filter-multi-select",
          "encodings": {
            "fields": [{
              "fieldName": "campaign_name",
              "displayName": "Campaign",
              "queryName": "ds_campaign"
            }]
          },
          "frame": {"showTitle": true, "title": "Campaign"}
        }
      },
      "position": {"x": 0, "y": 0, "width": 2, "height": 2}
    }
  ]
}
```

---

## Page-Level Filter Example

Place directly on a canvas page (affects only that page):

```json
{
  "name": "platform_breakdown",
  "displayName": "Platform Breakdown",
  "pageType": "PAGE_TYPE_CANVAS",
  "layout": [
    {
      "widget": {
        "name": "page-title",
        "multilineTextboxSpec": {"lines": ["## Platform Breakdown"]}
      },
      "position": {"x": 0, "y": 0, "width": 4, "height": 1}
    },
    {
      "widget": {
        "name": "filter_platform",
        "queries": [{
          "name": "ds_platform",
          "query": {
            "datasetName": "platform_data",
            "fields": [{"name": "platform", "expression": "`platform`"}],
            "disaggregated": false
          }
        }],
        "spec": {
          "version": 2,
          "widgetType": "filter-multi-select",
          "encodings": {
            "fields": [{
              "fieldName": "platform",
              "displayName": "Platform",
              "queryName": "ds_platform"
            }]
          },
          "frame": {"showTitle": true, "title": "Platform"}
        }
      },
      "position": {"x": 4, "y": 0, "width": 2, "height": 2}
    }
    // ... other widgets on this page
  ]
}
```

---

## Date Range Filtering (IMPORTANT)

> **Best Practice**: Most dashboards should include a date range filter on datasets with time-based data.
> This allows users to focus on relevant time periods. However, be thoughtful about which datasets
> should be filtered - metrics like "All-Time Total" or "MRR" should NOT be date-filtered.

There are **two approaches** to date range filtering:

### Approach 1: Field-Based Filtering (Automatic)

When your dataset has a date column, the filter automatically applies `IN_RANGE()` to that field.
This is the simplest approach when the date field is directly in the SELECT.

**Dataset** (date field in output):
```json
{
  "name": "weekly_trend",
  "displayName": "Weekly Trend",
  "queryLines": [
    "SELECT week_start, revenue_usd, returns_usd ",
    "FROM catalog.schema.weekly_summary ",
    "ORDER BY week_start"
  ]
}
```

**Filter widget** (binds to field):
```json
{
  "widget": {
    "name": "date_range_filter",
    "queries": [{
      "name": "ds_weekly_trend_date",
      "query": {
        "datasetName": "weekly_trend",
        "fields": [{"name": "week_start", "expression": "`week_start`"}],
        "disaggregated": false
      }
    }],
    "spec": {
      "version": 2,
      "widgetType": "filter-date-range-picker",
      "encodings": {
        "fields": [{
          "fieldName": "week_start",
          "queryName": "ds_weekly_trend_date"
        }]
      },
      "frame": {"showTitle": true, "title": "Date Range"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 2}
}
```

### Approach 2: Parameter-Based Filtering (Explicit Control)

When you need the date range in a WHERE clause (e.g., filtering before aggregation),
use SQL parameters with `:param_name.min` and `:param_name.max` syntax.

**Dataset** (with parameter in WHERE clause):
```json
{
  "name": "revenue_by_category",
  "displayName": "Revenue by Category",
  "queryLines": [
    "SELECT category, SUM(revenue_usd) as revenue ",
    "FROM catalog.schema.daily_orders ",
    "WHERE order_date BETWEEN :date_range.min AND :date_range.max ",
    "GROUP BY category ORDER BY revenue DESC"
  ],
  "parameters": [{
    "displayName": "date_range",
    "keyword": "date_range",
    "dataType": "DATE",
    "complexType": "RANGE",
    "defaultSelection": {
      "range": {
        "dataType": "DATE",
        "min": {"value": "now-12M/M"},
        "max": {"value": "now/M"}
      }
    }
  }]
}
```

**Filter widget** (binds to parameter):
```json
{
  "widget": {
    "name": "date_range_filter",
    "queries": [{
      "name": "ds_revenue_date_param",
      "query": {
        "datasetName": "revenue_by_category",
        "parameters": [{"name": "date_range", "keyword": "date_range"}],
        "disaggregated": false
      }
    }],
    "spec": {
      "version": 2,
      "widgetType": "filter-date-range-picker",
      "encodings": {
        "fields": [{
          "parameterName": "date_range",
          "queryName": "ds_revenue_date_param"
        }]
      },
      "frame": {"showTitle": true, "title": "Date Range"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 2}
}
```

### Combining Both Approaches

A single date range filter can bind to multiple datasets using different approaches:

```json
{
  "widget": {
    "name": "date_range_filter",
    "queries": [
      {
        "name": "ds_trend_field",
        "query": {
          "datasetName": "weekly_trend",
          "fields": [{"name": "week_start", "expression": "`week_start`"}],
          "disaggregated": false
        }
      },
      {
        "name": "ds_category_param",
        "query": {
          "datasetName": "revenue_by_category",
          "parameters": [{"name": "date_range", "keyword": "date_range"}],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-date-range-picker",
      "encodings": {
        "fields": [
          {"fieldName": "week_start", "queryName": "ds_trend_field"},
          {"parameterName": "date_range", "queryName": "ds_category_param"}
        ]
      },
      "frame": {"showTitle": true, "title": "Date Range"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 2}
}
```

### When NOT to Apply Date Filtering

Some metrics should NOT be filtered by date:
- **MRR/ARR**: Monthly/Annual recurring revenue is a point-in-time metric
- **All-Time Totals**: Cumulative metrics since inception
- **YTD Comparisons**: When comparing year-to-date against prior year
- **Fixed Snapshots**: "As of" metrics for a specific date

For these, either:
1. Don't bind them to the date filter (omit from filter queries)
2. Use a separate dataset not connected to the date range filter

---

## Multi-Dataset Filters

When a filter should affect multiple datasets (e.g., "Region" filter for both sales and customers data), add multiple queries - one per dataset:

```json
{
  "widget": {
    "name": "filter_region",
    "queries": [
      {
        "name": "sales_region",
        "query": {
          "datasetName": "sales",
          "fields": [{"name": "region", "expression": "`region`"}],
          "disaggregated": false
        }
      },
      {
        "name": "customers_region",
        "query": {
          "datasetName": "customers",
          "fields": [{"name": "region", "expression": "`region`"}],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-multi-select",
      "encodings": {
        "fields": [
          {"fieldName": "region", "displayName": "Region (Sales)", "queryName": "sales_region"},
          {"fieldName": "region", "displayName": "Region (Customers)", "queryName": "customers_region"}
        ]
      },
      "frame": {"showTitle": true, "title": "Region"}
    }
  },
  "position": {"x": 0, "y": 0, "width": 2, "height": 2}
}
```

Each `queryName` in `encodings.fields` binds the filter to that specific dataset. Datasets not bound will not be filtered.

---

## Filter Layout Guidelines

- Global filters: Position on dedicated filter page, stack vertically at `x=0`
- Page-level filters: Position in header area of page (e.g., top-right corner)
- Typical sizing: `width: 2, height: 2`
