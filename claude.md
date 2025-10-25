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

## List Endpoints and the Verbose Parameter

### Problem
List endpoints (dashboards, charts, datasets, etc.) can return extremely large responses when dealing with hundreds or thousands of items. This causes:
- Token limit issues for LLMs
- Slow response times
- Unnecessary data transfer

### Solution: Verbose Parameter
All list endpoints now support a `verbose` parameter (default: `False`):

```python
# Compact response (default) - only essential fields
superset_dashboard_list(search="", verbose=False)  # or omit verbose parameter

# Full response - all fields from API
superset_dashboard_list(search="", verbose=True)
```

### ⚠️ IMPORTANT: Token Consumption Warning

**ALWAYS use `verbose=False` (the default) unless absolutely necessary.**

When `verbose=True` is used, the full API response with ALL fields is returned, which can be extremely large:
- Dashboard list: Includes `json_metadata`, `position_json`, tags, roles, certification details
- Chart list: Includes full `params` configuration, query context, form data
- Query list: Includes complete SQL queries (no truncation)

**Before using `verbose=True`, warn the user:**
> "⚠️ Warning: Using verbose=True will return the full response with all fields, which consumes significantly more tokens (up to 20x more). This is only recommended if you need complete metadata. For most use cases, the default compact response is sufficient. Do you want to proceed with verbose=True?"

**Best practice:** Always start with `verbose=False`, then use `get_by_id()` functions to fetch complete details for specific items.

### Supported Endpoints

| Endpoint | Default Fields (verbose=False) | Impact |
|----------|-------------------------------|--------|
| `superset_dashboard_list()` | id, dashboard_title, url, slug, owners, charts | ~95% smaller |
| `superset_chart_list()` | id, slice_name, viz_type, datasource_id, datasource_name, datasource_type | ~90% smaller |
| `superset_dataset_list()` | id, table_name, schema, database, owners | ~85% smaller |
| `superset_database_list()` | id, database_name, backend, allow_run_async, expose_in_sqllab | ~80% smaller |
| `superset_query_list()` | id, sql (truncated 200 chars), status, start_time, end_time, database | ~85% smaller |
| `superset_sqllab_get_saved_queries()` | id, label, description, sql (truncated 200 chars), database | ~85% smaller |

### Recommended Workflow

**Discovery Pattern (Two-Tier Lookup):**
1. **Search/List** (verbose=False): Get minimal data to find items
   ```python
   dashboards = superset_dashboard_list(search="sales", verbose=False)
   # Returns: [{id: 123, dashboard_title: "Sales Dashboard", url: "...", owners: [...], charts: [...]}]
   ```

2. **Get Details** (by ID): Fetch full information for specific items
   ```python
   dashboard_full = superset_dashboard_get_by_id(dashboard_id=123)
   # Returns: Complete dashboard with all metadata, json_metadata, position_json, etc.
   ```

**Or use verbose=True when you need everything:**
```python
dashboards_full = superset_dashboard_list(search="sales", verbose=True)
# Returns: All fields including json_metadata, position_json, tags, roles, etc.
```

### Examples

**Finding and inspecting a dashboard:**
```python
# 1. Search for dashboard (compact)
results = superset_dashboard_list(search="Test dashboard", verbose=False)
# Returns: {count: 1, ids: [8117], result: [{id: 8117, dashboard_title: "Test dashboard by 1237281", ...}]}

# 2. Get specific dashboard details (full data)
dashboard = superset_dashboard_get_by_id(dashboard_id=8117)
# Returns: Complete dashboard with all fields

# 3. Get chart details from the dashboard
for chart in dashboard['result']['charts']:
    chart_details = superset_chart_get_by_id(chart_id=chart['id'])
```

**Browsing all charts (with large datasets):**
```python
# Don't do this with verbose=True on 84K+ charts!
charts = superset_chart_list(verbose=False)
# Returns: Compact list of 84,117 charts with only essential fields

# Then get details for specific charts
chart_full = superset_chart_get_by_id(chart_id=85888)
```

### Technical Implementation
- **Method**: Post-processing (Python filtering after API response)
- **SQL Truncation**: Query SQL is truncated to 200 characters with "..." in non-verbose mode
- **Nested Fields**: Supports nested field extraction (e.g., "database.database_name", "owners.username")
- **Backward Compatible**: Default verbose=False provides optimal performance; set verbose=True for legacy behavior

---

*Reference compiled on 2025-10-19*
*Updated on 2025-10-25 with verbose parameter documentation*