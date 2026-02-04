# Azure Workbook Development Guide

A comprehensive guide for creating Azure Workbooks based on lessons learned and best practices.

---

## Table of Contents

1. [Workbook Structure](#workbook-structure)
2. [Item Types](#item-types)
3. [Parameters](#parameters)
4. [Tab Navigation](#tab-navigation)
5. [Conditional Visibility](#conditional-visibility)
6. [Azure Resource Graph Queries](#azure-resource-graph-queries)
7. [Grid Configuration](#grid-configuration)
8. [Tiles and Visualizations](#tiles-and-visualizations)
9. [Common Pitfalls](#common-pitfalls)
10. [Validation](#validation)
11. [Templates](#templates)

---

## Workbook Structure

Azure Workbooks are JSON files with the following base structure:

```json
{
  "version": "Notebook/1.0",
  "items": [
    // Array of workbook items (text, queries, parameters, etc.)
  ],
  "fallbackResourceIds": [
    // Optional: Default resource scope
  ],
  "$schema": "https://github.com/Microsoft/Application-Insights-Workbooks/blob/master/schema/workbook.json"
}
```

---

## Item Types

| Type | Description | Use Case |
|------|-------------|----------|
| `1` | Text/Markdown | Headers, descriptions, help text |
| `3` | Query | KQL queries against Azure Resource Graph or Log Analytics |
| `9` | Parameters | Filters, dropdowns, subscriptions selector |
| `11` | Links/Tabs | Navigation tabs, external links |
| `12` | Group | Container for grouping items together |

### Basic Item Structure

```json
{
  "type": 1,
  "content": {
    "json": "## Your Markdown Content Here"
  },
  "name": "text - section header",
  "conditionalVisibility": {
    "parameterName": "SelectedTab",
    "comparison": "isEqualTo",
    "value": "TabValue"
  }
}
```

---

## Parameters

### Subscription Selector

```json
{
  "id": "unique-guid-here",
  "version": "KqlParameterItem/1.0",
  "name": "Subscriptions",
  "type": 6,
  "isRequired": true,
  "multiSelect": true,
  "quote": "'",
  "delimiter": ",",
  "typeSettings": {
    "additionalResourceOptions": ["value::all"],
    "includeAll": true,
    "showDefault": false
  },
  "defaultValue": "value::all",
  "value": ["value::all"]
}
```

### Resource Groups (Dynamic Query)

```json
{
  "id": "unique-guid-here",
  "version": "KqlParameterItem/1.0",
  "name": "ResourceGroups",
  "type": 2,
  "isRequired": false,
  "multiSelect": true,
  "quote": "'",
  "delimiter": ",",
  "query": "resources\n| summarize by resourceGroup\n| order by resourceGroup asc\n| project value = resourceGroup, label = resourceGroup, selected = false",
  "crossComponentResources": ["{Subscriptions}"],
  "typeSettings": {
    "additionalResourceOptions": ["value::all"],
    "showDefault": false
  },
  "defaultValue": "value::all",
  "queryType": 1,
  "resourceType": "microsoft.resourcegraph/resources"
}
```

### Static Dropdown (JSON Data)

```json
{
  "id": "unique-guid-here",
  "version": "KqlParameterItem/1.0",
  "name": "ShowHelp",
  "label": "Show Help",
  "type": 2,
  "isRequired": true,
  "typeSettings": {
    "additionalResourceOptions": [],
    "showDefault": false
  },
  "jsonData": "[{\"value\": \"Yes\", \"label\": \"📖 Show\"}, {\"value\": \"No\", \"label\": \"🔒 Hide\"}]",
  "value": "No"
}
```

### Hidden Parameters

To hide a parameter from the UI (useful for tab navigation):

```json
{
  "name": "SelectedTab",
  "type": 2,
  "isRequired": true,
  "isHiddenWhenLocked": true,
  "jsonData": "[{\"value\": \"Tab1\", \"label\": \"Tab 1\"}]",
  "value": "Tab1"
}
```

---

## Tab Navigation

### Creating Tabs (Type 11)

```json
{
  "type": 11,
  "content": {
    "version": "LinkItem/1.0",
    "style": "tabs",
    "links": [
      {
        "id": "tab-summary",
        "cellValue": "SelectedTab",
        "linkTarget": "parameter",
        "linkLabel": "📊 Summary",
        "subTarget": "Summary",
        "style": "link"
      },
      {
        "id": "tab-details",
        "cellValue": "SelectedTab",
        "linkTarget": "parameter",
        "linkLabel": "📋 Details",
        "subTarget": "Details",
        "style": "link"
      }
    ]
  },
  "name": "tabs - navigation"
}
```

### How Tabs Work

1. **Hidden Parameter**: Create a hidden parameter (e.g., `SelectedTab`) with possible values
2. **Tab Links**: Each tab sets `SelectedTab` to its `subTarget` value when clicked
3. **Conditional Visibility**: Items show/hide based on `SelectedTab` value

---

## Conditional Visibility

### ⚠️ CRITICAL: Order Matters!

**The parameter MUST be defined BEFORE any item that references it in `conditionalVisibility`.**

Azure Workbooks evaluates items in order. If you reference a parameter before it's defined, the condition won't work.

### Correct Order

```json
{
  "items": [
    { "type": 9, "name": "parameters" },      // 1. Parameters defined first
    { "type": 11, "name": "tabs" },            // 2. Tabs (optional)
    { 
      "type": 1, 
      "conditionalVisibility": { ... },        // 3. Items with conditionalVisibility
      "name": "content"
    }
  ]
}
```

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `isEqualTo` | Exact match |
| `isNotEqualTo` | Not equal |
| `contains` | String contains |
| `startsWith` | String starts with |
| `endsWith` | String ends with |
| `isNull` | Value is null/empty |
| `isNotNull` | Value is not null |

### Example

```json
{
  "conditionalVisibility": {
    "parameterName": "ShowHelp",
    "comparison": "isEqualTo",
    "value": "Yes"
  }
}
```

---

## Azure Resource Graph Queries

### Basic Query Structure

```json
{
  "type": 3,
  "content": {
    "version": "KqlItem/1.0",
    "query": "resources\n| where type =~ 'microsoft.compute/virtualmachines'\n| project name, resourceGroup, location, id",
    "size": 0,
    "title": "Virtual Machines",
    "queryType": 1,
    "resourceType": "microsoft.resourcegraph/resources",
    "crossComponentResources": ["{Subscriptions}"]
  }
}
```

### Filtering by Parameters

```kusto
resources
| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
| where type =~ 'microsoft.compute/virtualmachines'
```

### ⚠️ Case-Sensitive Joins

Azure Resource Graph joins are **case-sensitive**. Always use `tolower()` on both sides:

```kusto
// WRONG - may miss matches
| join kind=leftouter (otherTable) on resourceId

// CORRECT - case-insensitive matching
| join kind=leftouter (
    otherTable
    | extend resourceId = tolower(resourceId)
) on $left.resourceId == $right.resourceId
```

### ⚠️ The `securityresources` Table

The `securityresources` table has limitations:

- ❌ Does NOT support `tags` property
- ❌ Does NOT support `resourceGroup` filtering
- ✅ Only supports `subscriptionId` filtering

```kusto
// WRONG - will fail
securityresources
| where resourceGroup in ({ResourceGroups})

// CORRECT - only filter by subscription
securityresources
| where type =~ 'microsoft.security/pricings'
| extend pricingTier = tostring(properties.pricingTier)
```

### Dynamic Counts (Avoid Hardcoding)

```kusto
// WRONG - hardcoded values
| where resourceName in ('Plan1', 'Plan2', 'Plan3')
| summarize Total = count()
| project Value = strcat(EnabledCount, '/3 Enabled')

// CORRECT - dynamic calculation
| summarize 
    TotalPlans = dcount(planName),
    EnabledPlans = dcountif(planName, status == 'Enabled')
| project Value = strcat(EnabledPlans, '/', TotalPlans, ' Enabled')
```

### Division by Zero Protection

```kusto
// WRONG - will error if Total is 0
| extend Percentage = round(todouble(Count) / todouble(Total) * 100, 2)

// CORRECT - protected
| extend Percentage = iff(Total == 0, 0.0, round(todouble(Count) / todouble(Total) * 100, 2))
```

### Use `dcount()` vs `count()`

```kusto
// count() - counts all rows (including duplicates across subscriptions)
| summarize Total = count()

// dcount() - counts unique values
| summarize TotalPlans = dcount(planName)
```

---

## Grid Configuration

### Making Resources Clickable

To make resource names clickable (navigate to Azure Portal):

1. **Include Resource ID in query** (can be hidden):
```kusto
| project 
    Name = name,
    ['Resource Group'] = resourceGroup,
    ['Resource ID'] = id  // Required for linking
```

2. **Configure formatter with linkColumn**:
```json
{
  "columnMatch": "Name",
  "formatter": 1,
  "formatOptions": {
    "linkTarget": "Resource",
    "linkColumn": "Resource ID"
  }
}
```

3. **Hide the Resource ID column**:
```json
{
  "columnMatch": "Resource ID",
  "formatter": 5
}
```

### Common Formatters

| Formatter | Description |
|-----------|-------------|
| `1` | Text |
| `5` | Hidden |
| `7` | Link |
| `12` | Big number |
| `15` | Subscription (with icon) |
| `18` | Thresholds (icons/colors) |

### Threshold Formatting (Icons/Colors)

```json
{
  "columnMatch": "Status",
  "formatter": 18,
  "formatOptions": {
    "thresholdsOptions": "icons",
    "thresholdsGrid": [
      {
        "operator": "==",
        "thresholdValue": "Compliant",
        "representation": "success",
        "text": "✅ Compliant"
      },
      {
        "operator": "==",
        "thresholdValue": "Non-Compliant",
        "representation": "failed",
        "text": "❌ Non-Compliant"
      },
      {
        "operator": "Default",
        "thresholdValue": null,
        "representation": "unknown",
        "text": "{0}"
      }
    ]
  }
}
```

### Grid Options

```json
{
  "gridSettings": {
    "formatters": [...],
    "filter": true,
    "sortBy": [
      {
        "itemKey": "Name",
        "sortOrder": 1
      }
    ],
    "rowLimit": 10000
  },
  "showExportToExcel": true
}
```

---

## Tiles and Visualizations

### Tile Visualization

```json
{
  "visualization": "tiles",
  "tileSettings": {
    "titleContent": {
      "columnMatch": "Metric",
      "formatter": 1
    },
    "leftContent": {
      "columnMatch": "Value",
      "formatter": 12,
      "formatOptions": {
        "palette": "auto"
      }
    },
    "secondaryContent": {
      "columnMatch": "Status",
      "formatter": 1
    },
    "showBorder": true
  }
}
```

### ⚠️ Chart Issues

For bar charts, **only project the columns needed**:

```kusto
// WRONG - extra columns cause overlapping series
| project ResourceType, Total, Compliant, NonCompliant, Percentage

// CORRECT - only what's needed for X and Y axis
| project ['Resource Type'] = ResourceType, ['Compliance %'] = Percentage
```

### Width Control

```json
{
  "customWidth": "25",  // Percentage width (25%, 33%, 50%, etc.)
  "name": "query - tile"
}
```

---

## Common Pitfalls

### 1. conditionalVisibility Not Working

- ✅ Check parameter is defined BEFORE the item
- ✅ Check parameter name spelling exactly matches
- ✅ Check value matches exactly (case-sensitive)

### 2. Queries Return No Results

- ✅ Check `crossComponentResources` includes `{Subscriptions}`
- ✅ Check ResourceGroups filter uses `'*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})`
- ✅ For `securityresources`, don't filter by resourceGroup or tags

### 3. Links Not Working

- ✅ Ensure `linkColumn` points to column with full resource ID
- ✅ Query must include `id` or resource ID column
- ✅ Column can be hidden with formatter 5

### 4. JSON Validation Errors

- ✅ Check for trailing commas
- ✅ Check for proper escaping in query strings (`\n` for newlines)
- ✅ Validate with: `Get-Content file.workbook | ConvertFrom-Json`

### 5. Emoji Encoding Issues

- Use actual emoji characters in JSON, not escape sequences
- If copy-pasting causes issues, use simple text alternatives

---

## Validation

Always validate JSON after making changes:

### PowerShell

```powershell
Get-Content 'path\to\workbook.workbook' | ConvertFrom-Json | Out-Null
Write-Host "JSON is valid"
```

### Python

```python
import json
with open('workbook.workbook', 'r', encoding='utf-8') as f:
    json.load(f)
print("JSON is valid")
```

---

## Templates

### Minimal Workbook Template

```json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 1,
      "content": {
        "json": "# 📊 My Workbook Title"
      },
      "name": "text - title"
    },
    {
      "type": 9,
      "content": {
        "version": "KqlParameterItem/1.0",
        "parameters": [
          {
            "id": "sub-param",
            "version": "KqlParameterItem/1.0",
            "name": "Subscriptions",
            "type": 6,
            "isRequired": true,
            "multiSelect": true,
            "quote": "'",
            "delimiter": ",",
            "typeSettings": {
              "additionalResourceOptions": ["value::all"],
              "includeAll": true
            },
            "defaultValue": "value::all"
          }
        ],
        "style": "pills",
        "queryType": 0,
        "resourceType": "microsoft.operationalinsights/workspaces"
      },
      "name": "parameters"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "resources\n| summarize Count = count() by type\n| order by Count desc\n| take 10",
        "size": 0,
        "title": "Top 10 Resource Types",
        "queryType": 1,
        "resourceType": "microsoft.resourcegraph/resources",
        "crossComponentResources": ["{Subscriptions}"]
      },
      "name": "query - resources"
    }
  ],
  "$schema": "https://github.com/Microsoft/Application-Insights-Workbooks/blob/master/schema/workbook.json"
}
```

### Workbook with Tabs Template

```json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 1,
      "content": {
        "json": "# 📊 Tabbed Workbook"
      },
      "name": "text - title"
    },
    {
      "type": 9,
      "content": {
        "version": "KqlParameterItem/1.0",
        "parameters": [
          {
            "id": "sub-param",
            "version": "KqlParameterItem/1.0",
            "name": "Subscriptions",
            "type": 6,
            "isRequired": true,
            "multiSelect": true,
            "quote": "'",
            "delimiter": ",",
            "typeSettings": {
              "additionalResourceOptions": ["value::all"],
              "includeAll": true
            },
            "defaultValue": "value::all"
          },
          {
            "id": "tab-param",
            "version": "KqlParameterItem/1.0",
            "name": "SelectedTab",
            "type": 2,
            "isRequired": true,
            "isHiddenWhenLocked": true,
            "jsonData": "[{\"value\": \"Overview\", \"label\": \"Overview\"}, {\"value\": \"Details\", \"label\": \"Details\"}]",
            "value": "Overview"
          }
        ],
        "style": "pills",
        "queryType": 0,
        "resourceType": "microsoft.operationalinsights/workspaces"
      },
      "name": "parameters"
    },
    {
      "type": 11,
      "content": {
        "version": "LinkItem/1.0",
        "style": "tabs",
        "links": [
          {
            "id": "tab-overview",
            "cellValue": "SelectedTab",
            "linkTarget": "parameter",
            "linkLabel": "📊 Overview",
            "subTarget": "Overview",
            "style": "link"
          },
          {
            "id": "tab-details",
            "cellValue": "SelectedTab",
            "linkTarget": "parameter",
            "linkLabel": "📋 Details",
            "subTarget": "Details",
            "style": "link"
          }
        ]
      },
      "name": "tabs"
    },
    {
      "type": 1,
      "content": {
        "json": "## Overview Section Content"
      },
      "conditionalVisibility": {
        "parameterName": "SelectedTab",
        "comparison": "isEqualTo",
        "value": "Overview"
      },
      "name": "text - overview"
    },
    {
      "type": 1,
      "content": {
        "json": "## Details Section Content"
      },
      "conditionalVisibility": {
        "parameterName": "SelectedTab",
        "comparison": "isEqualTo",
        "value": "Details"
      },
      "name": "text - details"
    }
  ],
  "$schema": "https://github.com/Microsoft/Application-Insights-Workbooks/blob/master/schema/workbook.json"
}
```

---

## Useful KQL Patterns

### Resource Compliance Check

```kusto
resources
| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
| where type =~ 'microsoft.storage/storageaccounts'
| extend isCompliant = tobool(properties.supportsHttpsTrafficOnly)
| summarize 
    Total = count(),
    Compliant = countif(isCompliant == true),
    NonCompliant = countif(isCompliant != true)
| extend ComplianceRate = iff(Total == 0, 0.0, round(todouble(Compliant) / todouble(Total) * 100, 2))
| extend Status = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 70, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Improvement',
    '🔴 Critical'
)
```

### Cross-Tenant Resource Lookup

```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| mv-expand connection = properties.privateLinkServiceConnections
| extend targetResourceId = tolower(tostring(connection.properties.privateLinkServiceId))
| join kind=leftouter (
    resources
    | extend targetResourceId = tolower(id)
    | project targetResourceId, TargetTenant = tenantId
) on targetResourceId
| extend TargetTenant = iff(isempty(TargetTenant), 'Cross-Tenant', TargetTenant)
```

### Defender Plans Status

```kusto
securityresources
| where type =~ 'microsoft.security/pricings'
| extend pricingTier = tostring(properties.pricingTier)
| extend planName = name
| summarize 
    TotalPlans = dcount(planName),
    EnabledPlans = dcountif(planName, pricingTier == 'Standard')
| extend Status = iff(EnabledPlans == TotalPlans, '🟢 All Enabled', '🟠 Partial')
| project Value = strcat(EnabledPlans, '/', TotalPlans, ' Plans'), Status
```

---

## File Organization

```
WorkbookProject/
├── MyWorkbook.workbook          # Main workbook file
├── README.md                    # Documentation
├── queries/                     # Optional: Store complex queries separately
│   ├── compliance-check.kql
│   └── resource-summary.kql
└── scripts/                     # Helper scripts
    └── validate.ps1
```

---

## Quick Reference Card

| Task | Solution |
|------|----------|
| Hide parameter | `"isHiddenWhenLocked": true` |
| Make resource clickable | `"linkTarget": "Resource", "linkColumn": "Resource ID"` |
| Hide column | `"formatter": 5` |
| Show based on param | `"conditionalVisibility": { "parameterName": "X", "comparison": "isEqualTo", "value": "Y" }` |
| Case-insensitive join | Use `tolower()` on both sides |
| Avoid div by zero | `iff(Total == 0, 0.0, calculation)` |
| Count unique | `dcount(column)` instead of `count()` |
| Filter all resource groups | `'*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})` |

---

*Last updated: January 2026*
