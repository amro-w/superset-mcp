# Superset MCP Reference Guide

## Working with Snowflake Data

### Database Configuration
- **Snowflake Database ID**: Always use `database_id: 2` when creating datasets or charts from Snowflake
- **Database Name**: "Snowflake"

### Using sf-reader MCP for Data Exploration

Before creating charts, use the `sf-reader` MCP tools to:
- **Inspect table schemas**: `DESCRIBE VIEW schema_name.table_name`
- **Query data**: Execute any read-only SQL query to understand the data structure
- **Validate data**: Check column names, data types, and sample values

Example workflow:
1. Check table structure using sf-reader MCP: `DESCRIBE VIEW ACHRISK.ACH_RISK`
2. Query sample data: `SELECT column_name, COUNT(*) FROM schema.table GROUP BY column_name LIMIT 10`
3. Create Superset dataset: `superset_dataset_create(database_id=2, schema="ACHRISK", table_name="ACH_RISK")`
4. Create chart from the dataset: `superset_chart_create(datasource_id=dataset_id, datasource_type="table", ...)`

---

## Supported Chart Types (viz_type values)

When creating charts via the Superset API, use these exact `viz_type` values:

### Time Series Charts
- `echarts_timeseries` - Standard time series
- `echarts_timeseries_line` - Line chart
- `echarts_timeseries_bar` - Bar chart
- `echarts_timeseries_scatter` - Scatter plot
- `echarts_timeseries_smooth` - Smooth line
- `echarts_timeseries_step` - Step chart
- `mixed_timeseries` - Mixed time series

### Area and Distribution
- `echarts_area` - Area chart
- `histogram_v2` - Histogram
- `box_plot` - Box plot
- `violin` - Violin plot (if available)

### Geographic Visualizations
- `country_map` - Country map
- `world_map` - World map
- `mapbox` - MapBox visualization

### Big Numbers and KPIs
- `big_number` - Big number
- `big_number_total` - Big number with total
- `pop_kpi` - Period over period KPI

### Tables and Grids
- `table` - Standard table
- `ag-grid-table` - AG Grid table
- `pivot_table_v2` - Pivot table
- `time_table` - Time table
- `time_pivot` - Time pivot

### Statistical and Advanced
- `bubble_v2` - Bubble chart (modern)
- `bubble` - Legacy bubble chart
- `paired_ttest` - Paired t-test
- `compare` - Comparison chart

### Hierarchical and Flow
- `treemap_v2` - Treemap
- `sunburst_v2` - Sunburst
- `sankey_v2` - Sankey diagram
- `tree_chart` - Tree chart
- `partition` - Partition chart

### Specialized Visualizations
- `pie` - Pie chart
- `rose` - Rose chart
- `gauge_chart` - Gauge
- `bullet` - Bullet chart
- `funnel` - Funnel chart
- `waterfall` - Waterfall chart
- `radar` - Radar chart
- `chord` - Chord diagram
- `graph_chart` - Graph/network chart
- `gantt_chart` - Gantt chart
- `cal_heatmap` - Calendar heatmap
- `heatmap_v2` - Standard heatmap
- `horizon` - Horizon chart
- `para` - Parallel coordinates
- `cartodiagram` - Carto diagram
- `word_cloud` - Word cloud
- `handlebars` - Handlebars template

---

## Dashboard Chart Association Fix

### Problem
When creating dashboards programmatically via Superset's REST API, charts may appear as empty placeholders with the error:
```
"There is no chart definition associated with this component, could it have been deleted?"
```

### Root Cause
**Charts exist but are not properly associated with the dashboard.**

Superset requires two separate configurations:
1. **Layout Definition** (`position_json`) - WHERE charts should appear ✅
2. **Chart Association** - WHICH charts belong to the dashboard ❌

The REST API `PUT /api/v1/dashboard/{id}` only updates the layout but doesn't establish the relationship between dashboards and charts.

### Solution: Update Chart with Dashboard Association

**The proper way to associate a chart with a dashboard is to update the chart's `dashboards` field:**

Using the Superset MCP:
```python
superset_chart_update(
    chart_id=85888,
    data={"dashboards": [8117]}
)
```

Using the REST API directly:
```python
requests.put(
    f"{BASE_URL}/api/v1/chart/{chart_id}",
    json={"dashboards": [dashboard_id]}
)
```

This establishes the relationship between the chart and dashboard.

### Best Practice for Creating Dashboards with Charts

1. **Create charts first** using `POST /api/v1/chart/` or `superset_chart_create()`
2. **Create dashboard** using `POST /api/v1/dashboard/` or `superset_dashboard_create()`
3. **Associate charts with dashboard** by updating each chart:
   ```python
   superset_chart_update(chart_id=chart_id, data={"dashboards": [dashboard_id]})
   ```
4. **Set layout** using `PUT /api/v1/dashboard/{id}` or `superset_dashboard_update()` with `position_json`

### Verification
After applying the fix, verify through the API:
```bash
curl -X GET "http://localhost:8088/api/v1/dashboard/12" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

Should return charts array populated instead of empty: `"charts": ["Chart Name 1", "Chart Name 2", ...]`

---

*Reference compiled on 2025-10-19*