# DORA Compliance Dashboard for Azure

## 📋 Table of Contents

- [Overview](#overview)
- [DORA Regulation Background](#dora-regulation-background)
- [Dashboard Structure](#dashboard-structure)
- [Deployment Guide](#deployment-guide)
- [Customization Guide](#customization-guide)
- [Query Development](#query-development)
- [Troubleshooting](#troubleshooting)
- [Maintenance and Updates](#maintenance-and-updates)

---

## Overview

### What is This Workbook?

The **DORA Compliance Dashboard** is an Azure Monitor Workbook that provides comprehensive assessment of your Azure environment's compliance with the **Digital Operational Resilience Act (DORA) - Regulation (EU) 2022/2554** for the European Union financial sector.

### Key Features

- ✅ **150+ Policy Controls** - Evaluates compliance across 150+ Azure Policy controls
- 📊 **Interactive Dashboard** - Real-time compliance monitoring with drill-down capabilities
- 📈 **Executive Summary** - High-level compliance metrics with percentage-based risk assessment
- 🔍 **Detailed Analysis** - Resource-level compliance details with exportable data
- 🎯 **DORA Article Mapping** - Direct mapping to DORA regulation articles (9.3a-9.3d, 10.1-10.2)
- 📋 **CSV Export** - All data tables support CSV export for reporting and analysis
- 🏷️ **Tag Filtering** - Filter resources by custom Azure tags

### Compliance Areas Covered

| DORA Article | Focus Area | Key Controls |
|-------------|------------|--------------|
| **9.3a** | ICT Systems Availability | SSL/TLS enforcement, private endpoints, secure transfer, availability zones |
| **9.3b** | Access Control & Authorization | RBAC, VM backup, geo-redundancy, CMK encryption |
| **9.3c** | Data Protection & Encryption | Key rotation, system updates, CMK, soft delete |
| **9.3d** | Identity & Authentication | AAD-only authentication, multiple subscription owners |
| **10.1** | Detection & Monitoring | Microsoft Defender, logging, activity logs, resource logs |
| **10.2** | Incident Response & Logging | NSG flow logs, PostgreSQL logging, traffic analytics |

---

## DORA Regulation Background

### Regulation (EU) 2022/2554

- **Official Name**: Digital Operational Resilience Act
- **Effective Date**: January 17, 2025
- **Applies To**: Financial entities operating in the European Union
- **Scope**: ICT risk management, incident reporting, digital operational resilience testing, third-party ICT risk management

### Why This Matters

Financial institutions in the EU must demonstrate robust ICT risk management frameworks. This workbook helps Azure customers:
- Assess current compliance posture
- Identify gaps and non-compliant resources
- Prioritize remediation efforts
- Generate compliance reports for auditors
- Track compliance improvements over time

---

## Dashboard Structure

### File Structure

```
DORAComplianceDashboard/
├── DORAComplianceDashboard.workbook                 # Main Azure Monitor Workbook (JSON)
├── DORAComplianceDashboard-KQLQueries.md            # KQL queries WITH compliance scoring/thresholds
├── DORAComplianceDashboard-KQLQueries-Simplified.md # KQL queries WITHOUT compliance scoring (raw data)
└── README.md                                         # This documentation file
```

### KQL Query Documentation

The dashboard includes **two companion markdown files** with all KQL queries extracted for standalone use:

#### 📊 DORAComplianceDashboard-KQLQueries.md

**Purpose**: Complete query reference with DORA compliance scoring and threshold-based assessments

**Features**:
- ✅ **40+ Production-Ready Queries** - All queries from the workbook
- ✅ **Compliance Scoring** - Includes status indicators (✅ Compliant, ⚠️ Partial, ❌ Non-Compliant)
- ✅ **Threshold-Based Logic** - Pre-configured compliance thresholds (e.g., 90% = Excellent, 75% = Good)
- ✅ **Ready for Resource Graph Explorer** - No parameter substitution needed
- ✅ **Threshold Warnings** - Queries with configurable thresholds clearly marked with ⚠️ Note
- ✅ **Guidance on Removing Compliance Scoring** - Complete section with examples

**Use Cases**:
- Run queries directly in Azure Resource Graph Explorer
- Test queries before adding to custom workbooks
- Export compliance-assessed data to CSV/JSON
- Reference for understanding workbook logic
- Create custom dashboards with pre-built compliance logic

#### 📋 DORAComplianceDashboard-KQLQueries-Simplified.md

**Purpose**: Raw resource data queries without any compliance assessments or thresholds

**Features**:
- ✅ **30+ Simplified Queries** - All compliance scoring removed
- ✅ **Raw Data Only** - Boolean/numeric values without status indicators
- ✅ **No Thresholds** - No "Excellent/Good/Critical" classifications
- ✅ **Export-Friendly** - Clean CSV exports without emoji characters
- ✅ **Custom Analysis Ready** - Apply your own logic in Excel/Power BI

**Use Cases**:
- Export data to Excel or Power BI for custom analysis
- Apply your own compliance thresholds and business rules
- Integrate with third-party tools and APIs
- Generate simple resource inventories
- Performance-optimized queries for large environments

**Example Difference**:

*With Compliance Scoring (Full version)*:
```kql
| extend DORACompliance = case(
    httpsOnly == true and minTlsVersion == 'TLS1_2', '✅ Compliant',
    httpsOnly == true, '⚠️ Partial - Check TLS',
    '❌ Non-Compliant'
)
| project ['Storage Account'] = name, ['DORA Status'] = DORACompliance, ...
```

*Without Compliance Scoring (Simplified version)*:
```kql
| project 
    ['Storage Account'] = name,
    ['HTTPS Only'] = httpsOnly,
    ['Min TLS'] = minTlsVersion,
    ...
```

**Which File to Use?**:
- Use **DORAComplianceDashboard-KQLQueries.md** if you want DORA-aligned compliance assessments
- Use **DORAComplianceDashboard-KQLQueries-Simplified.md** if you need raw data for custom analysis

### Workbook Architecture

The workbook is organized into **2 main category tabs** with multiple sub-sections:

#### Category 1: **📖 Overview Tab**
- **Purpose**: Documentation, changelog, and usage instructions
- **Content**:
  - Introduction to DORA and workbook purpose
  - **📋 Changelog & Version History** (expandable, collapsed by default)
    - Version 1.0 features (Network Security Perimeter, 6-state compliance logic, comprehensive DORA coverage)
    - Roadmap for future releases
  - **🎯 Features Supported by Section** (expandable, collapsed by default)
    - Comprehensive breakdown of all assessments by DORA article
    - Resource types covered (14 PaaS types for private endpoints)
    - Threshold configuration guide
    - Current limitations
  - **📚 Usage Instructions** (expandable, collapsed by default)
    - Step-by-step usage guide
    - Compliance status interpretation
    - Best practices and troubleshooting
    - Links to DORA regulation and Azure documentation
- **Use Case**: Onboarding, understanding features, accessing documentation

#### Category 2: **📊 Dashboard Tab** (Default View)
Contains all compliance assessment views organized into 8 sub-tabs:

##### **Executive Summary Tab**
- **Purpose**: High-level compliance metrics for leadership reporting
- **Content**:
  - 8 executive summary tiles showing compliance rates
  - Business Continuity Risk Assessment (percentage-based thresholds: 0%, ≤10%, ≤25%, >25%)
  - Key metrics: Zone Redundancy, Private Endpoints, Secure Transfer, Key Vault RBAC, AAD Auth, Defender Coverage, NSG Flow Logs
  - PaaS Network Isolation shows % of compliant resources (not endpoint count)
- **Use Case**: Quick compliance status check, board/executive reporting

##### **Article 9.3a - ICT Systems Availability Tab**
- **Purpose**: Network isolation and secure communications
- **Key Queries**:
  - Availability Zones compliance (18+ resource types)
  - Private Endpoints inventory and coverage (14 PaaS types)
  - SSL/TLS enforcement for Redis, databases, storage
  - PaaS resources with 6-state compliance logic:
    - 🔒 Network Security Perimeter (highest security)
    - ✅ Fully Isolated (Private Endpoint + public disabled)
    - ⚠️ Has PE - Public Access Enabled (needs remediation)
    - 🔐 Public Access Disabled - No PE (good but improvable)
    - 🔍 Review Required - Unknown Status (needs investigation)
    - ❌ No Private Endpoint (non-compliant)
  - SecuredByPerimeter support (Azure Network Security Perimeter)
- **Use Case**: Network security assessment, private connectivity validation

##### **Article 9.3b - Access Control & Authorization Tab**
- **Purpose**: Access control and data protection
- **Key Queries**:
  - RBAC and access control validation
  - Backup configuration assessment
- **Use Case**: Access management review

##### **Article 9.3c - Data Protection Tab**
- **Purpose**: Encryption and secure transfer validation
- **Key Queries**:
  - Key Vault RBAC and security features
  - Storage account security (encryption, HTTPS, CMK)
  - Virtual Machine backup compliance
  - Geo-redundancy assessment
- **Use Case**: Access control audit, backup validation

#### 4. **Article 9.3c - Data Protection & Encryption**
- **Purpose**: Data encryption and protection mechanisms
- **Key Queries**:
  - Customer-Managed Keys (CMK) usage
  - Encryption at rest validation
  - Soft delete and purge protection
  - Key rotation policies
- **Use Case**: Data protection audit, encryption validation

#### 5. **Article 9.3d - Identity & Authentication**
- **Purpose**: Identity management and authentication controls
- **Key Queries**:
  - Azure Active Directory (AAD) only authentication
  - SQL Server authentication methods
  - MySQL/PostgreSQL authentication
  - Subscription owner validation
- **Use Case**: Identity audit, authentication review

#### 6. **Article 10.1 & 10.2 - Detection, Monitoring & Incident Response**
- **Purpose**: Security monitoring and logging
- **Key Queries**:
  - Microsoft Defender for Cloud coverage
  - NSG Flow Logs configuration
  - Diagnostic logs validation
  - Traffic Analytics enablement
- **Use Case**: Security monitoring review, logging compliance

### Parameters (Filters)

The workbook includes dynamic parameters that filter all queries:

| Parameter | Type | Purpose | Default |
|-----------|------|---------|---------|
| **Subscriptions** | Multi-select | Select Azure subscriptions to analyze | All accessible subscriptions |
| **ResourceGroups** | Multi-select | Filter by resource groups | All (`*`) |
| **TimeRange** | Time picker | Not actively used in current version | Last 24 hours |
| **ShowHelp** | Dropdown | Show/hide help text sections | Yes |
| **TagName** | Dropdown | Select tag key for filtering | None |
| **TagValue** | Dropdown | Select tag value for filtering | None |

**How Filtering Works**:
- All queries include: `| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})`
- This allows "All" (`*`) selection or specific resource groups
- Tag filtering is available but optional

---

## Deployment Guide

### Prerequisites

- **Azure Subscription** with resources to monitor
- **Permissions**:
  - `Reader` role on subscriptions/resource groups to monitor
  - `Microsoft.Resources/subscriptions/resourceGroups/read`
  - `Microsoft.Resources/subscriptions/resources/read`
  - `Microsoft.Insights/workbooks/read` and `write` (to deploy workbook)

### Deployment Steps

#### Option 1: Azure Portal (Import JSON)

1. **Navigate to Azure Monitor Workbooks**:
   ```
   Azure Portal → Monitor → Workbooks → + New
   ```

2. **Open Advanced Editor**:
   - Click `</>` (Advanced Editor) button in toolbar
   - Select "Gallery Template" dropdown → Choose "Empty"

3. **Import Workbook JSON**:
   - Delete all existing JSON
   - Copy entire contents of `DORAComplianceDashboard.workbook`
   - Paste into editor
   - Click "Apply"

4. **Save Workbook**:
   - Click "Save" icon (disk)
   - **Title**: `DORA Compliance Dashboard`
   - **Subscription**: Select subscription
   - **Resource Group**: Select resource group (or create new)
   - **Location**: Select Azure region
   - Click "Apply"

5. **Set Permissions** (if needed):
   - Click "Share" button
   - Grant access to other users/groups

#### Option 2: Azure CLI

```bash
# Set variables
SUBSCRIPTION_ID="<your-subscription-id>"
RESOURCE_GROUP="<your-resource-group>"
LOCATION="<azure-region>"
WORKBOOK_NAME="DORA-Compliance-Dashboard"

# Login to Azure
az login

# Set subscription context
az account set --subscription $SUBSCRIPTION_ID

# Create resource group (if needed)
az group create --name $RESOURCE_GROUP --location $LOCATION

# Deploy workbook
az monitor workbook create \
  --name $WORKBOOK_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --display-name "DORA Compliance Dashboard" \
  --serialized-data @DORAComplianceDashboard.workbook \
  --category "workbook"
```

#### Option 3: Azure PowerShell

```powershell
# Set variables
$SubscriptionId = "<your-subscription-id>"
$ResourceGroup = "<your-resource-group>"
$Location = "<azure-region>"
$WorkbookName = "DORA-Compliance-Dashboard"
$WorkbookPath = ".\DORAComplianceDashboard.workbook"

# Login and set context
Connect-AzAccount
Set-AzContext -SubscriptionId $SubscriptionId

# Read workbook JSON
$workbookJson = Get-Content $WorkbookPath -Raw

# Deploy (using ARM template or REST API)
# Note: Direct PowerShell cmdlet for workbooks may vary by Az module version
```

### Post-Deployment

1. **Select Subscriptions**: Choose which subscriptions to monitor
2. **Select Resource Groups**: Choose `*` (all) or specific resource groups
3. **Validate Data**: Ensure queries return data (may take 30-60 seconds on first load)
4. **Bookmark**: Save workbook URL for quick access

---

## Customization Guide

### Common Customizations

#### 1. Adjust Compliance Thresholds

Many queries use configurable thresholds for compliance ratings. These are marked with `⚠️ Note` in the KQL queries file.

**Example: Storage Account HTTPS Compliance**

Location: Article 9.3a tab, "Secure Transfer Enforcement" tile

Original thresholds:
```kql
| extend ComplianceStatus = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 75, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Improvement',
    '🔴 Critical'
)
```

To adjust (e.g., make stricter):
```kql
| extend ComplianceStatus = case(
    ComplianceRate >= 95, '🟢 Excellent',    // Changed from 90
    ComplianceRate >= 85, '🟡 Good',         // Changed from 75
    ComplianceRate >= 70, '🟠 Needs Improvement',  // Changed from 50
    '🔴 Critical'
)
```

**Queries with Configurable Thresholds**:
1. Zone Redundancy Compliance (Executive Summary)
2. Secure Transfer Enforcement
3. Key Vault RBAC Compliance
4. AAD-Only Authentication
5. Microsoft Defender Coverage
6. NSG Flow Logs
7. VM Backup Compliance
8. Business Continuity Risk (percentage-based)

#### 2. Add New Resource Types

**Example: Add Azure API Management to Zone Redundancy Check**

1. Find the Zone Redundancy query (line ~290 in workbook)
2. Add new resource type to `where type in~()` clause:

```kql
| where type in~ (
    'microsoft.compute/virtualmachines',
    'microsoft.network/loadbalancers',
    ...
    'microsoft.apimanagement/service'  // Add this line
)
```

3. Add human-readable type to case statement (if needed):

```kql
| extend resourceType = case(
    type =~ 'microsoft.compute/virtualmachines', 'Virtual Machine',
    ...
    type =~ 'microsoft.apimanagement/service', 'API Management',  // Add this
    type
)
```

#### 3. Modify PaaS Private Endpoint Coverage

The PaaS Private Endpoint Coverage query (Article 9.3a) now supports:
- **SecuredByPerimeter**: Resources protected by Network Security Perimeter
- **Private Endpoints**: Resources with private endpoint connections
- **Public Access Status**: Whether public network access is disabled

**To add more PaaS resource types**:

```kql
| where type in~ (
    'microsoft.storage/storageaccounts',
    'microsoft.keyvault/vaults',
    ...
    'microsoft.synapse/workspaces',        // Add Synapse Analytics
    'microsoft.datafactory/factories'      // Add Data Factory
)
```

**To customize compliance classifications**:

```kql
| extend DORACompliance = case(
    publicNetworkAccess =~ 'SecuredByPerimeter', '🔒 Network Security Perimeter',
    hasPrivateEndpoint and publicNetworkAccess =~ 'Disabled', '✅ Fully Isolated',
    hasPrivateEndpoint, '⚠️ Has PE - Public Access Enabled',
    publicNetworkAccess =~ 'Disabled', '🔐 Public Access Disabled',
    '❌ No Private Endpoint'
)
```

You can add more conditions or change the status labels.

#### 4. Add Custom Tags for Filtering

To add additional tag-based filtering:

1. Use the existing `TagName` and `TagValue` parameters (already configured)
2. Modify queries to include tag filtering:

```kql
| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
| where isempty('{TagName}') or tostring(tags['{TagName}']) == '{TagValue}'
```

#### 5. Change Tile Sizes

All executive summary tiles use `"size": "auto"`. To customize:

```json
"tileSettings": {
  "showBorder": true,
  "size": "auto"  // Options: "auto", "small", "medium", "large"
}
```

#### 6. Customize Color Schemes

Modify the `palette` in formatters:

```json
"formatters": [
  {
    "columnMatch": "DORA Status",
    "formatter": 18,
    "formatOptions": {
      "thresholdsOptions": "colors",
      "thresholdsGrid": [
        { "text": "✅", "color": "green" },
        { "text": "⚠️", "color": "orange" },
        { "text": "❌", "color": "red" }
      ]
    }
  }
]
```

### Advanced Customizations

#### Add New Queries/Tabs

1. **Create query in KQL Queries file first** (`DORAComplianceDashboard-KQLQueries.md`)
2. Test in Azure Resource Graph Explorer
3. Add to workbook using Advanced Editor:

```json
{
  "type": 3,
  "content": {
    "version": "KqlItem/1.0",
    "query": "resources\n| where type =~ 'your.resource.type'\n...",
    "size": 0,
    "title": "Your Query Title",
    "queryType": 1,
    "resourceType": "microsoft.resourcegraph/resources",
    "crossComponentResources": ["{Subscriptions}"]
  },
  "name": "query - your-query-name"
}
```

4. Add grid formatting, exportability, etc.

#### Create New Tabs

To add a new tab:

1. Add tab link to navigation parameter (line ~250):

```json
{
  "id": "tab-your-custom-tab",
  "cellValue": "SelectedTab",
  "linkTarget": "parameter",
  "linkLabel": "🔧 Your Custom Tab",
  "subTarget": "CustomTab",
  "style": "link"
}
```

2. Add conditional visibility to all elements in the tab:

```json
"conditionalVisibility": {
  "parameterName": "SelectedTab",
  "comparison": "isEqualTo",
  "value": "CustomTab"
}
```

---

## Query Development

### Using the KQL Query Files

The workbook comes with **two comprehensive KQL query reference files** that make query development easier:

#### 📊 DORAComplianceDashboard-KQLQueries.md
- **40+ queries with compliance scoring and thresholds**
- All queries ready to run in Azure Resource Graph Explorer
- Perfect for understanding workbook logic and creating similar assessments
- See [KQL Query Documentation](#kql-query-documentation) section for details

#### 📋 DORAComplianceDashboard-KQLQueries-Simplified.md
- **30+ simplified queries with raw data only**
- No compliance scoring - ideal for custom analysis
- Better performance for large environments
- Export-friendly for Excel/Power BI integration

**Workflow**: Test queries from markdown files → Customize if needed → Add to workbook

### Best Practices

1. **Always Test in Resource Graph Explorer First**
   - Navigate to: Azure Portal → Resource Graph Explorer
   - Copy queries from the markdown files
   - Test without workbook parameters first
   - Validate results and performance

2. **Start with Simplified Queries for Development**
   - Begin with queries from `DORAComplianceDashboard-KQLQueries-Simplified.md`
   - Add your own compliance logic if needed
   - Test thoroughly before adding thresholds

3. **Use Proper Newline Encoding**
   - In JSON: Use `\n` (single backslash)
   - NOT `\\n` (double backslash) - causes parsing errors

3. **Avoid Unsupported Functions**
   - ❌ `make_list()` - NOT supported in Resource Graph
   - ❌ `make_set()` - NOT supported
   - ✅ `count()`, `dcount()`, `summarize` - Supported
   - ✅ `array_length()`, `isnotempty()` - Supported

4. **Include Resource Group Filtering**
   ```kql
   | where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
   ```

5. **Use Consistent Naming**
   - Column names in brackets: `['Resource Name']`
   - Clear, descriptive names
   - Include DORA article reference where applicable

6. **Handle Null Values**
   ```kql
   | extend value = tostring(properties.someProperty)
   | extend value = iif(isempty(value), 'Unknown', value)
   ```

### Testing Workflow

1. **Write Query in Markdown File**
   - Edit `DORAComplianceDashboard-KQLQueries.md`
   - Include context comments

2. **Test in Resource Graph Explorer**
   - Copy query (without workbook parameters)
   - Run against your subscription
   - Verify results and performance

3. **Add to Workbook**
   - Open workbook in Advanced Editor
   - Find appropriate section
   - Add query with proper JSON formatting
   - Remember: `\n` for newlines, NOT `\\n`

4. **Add Grid Formatting**
   - Configure column formatters
   - Set up conditional formatting
   - Enable CSV export

5. **Test in Workbook**
   - Save workbook
   - Test with different parameters
   - Verify filtering works
   - Check export functionality

6. **Validate JSON**
   ```powershell
   Get-Content "DORAComplianceDashboard.workbook" | ConvertFrom-Json | Out-Null
   ```

### Common Query Patterns

#### Pattern 1: Compliance Rate Calculation

```kql
resources
| where type =~ 'resource.type'
| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
| extend isCompliant = <condition>
| summarize 
    Total = count(),
    Compliant = countif(isCompliant),
    NonCompliant = countif(not(isCompliant))
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 2)
| extend Status = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 75, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Improvement',
    '🔴 Critical'
)
```

#### Pattern 2: Resource-Level Details

```kql
resources
| where type =~ 'resource.type'
| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
| extend compliance = case(
    <condition1>, '✅ Compliant',
    <condition2>, '⚠️ Partial',
    '❌ Non-Compliant'
)
| project 
    ['Resource Name'] = name,
    ['Compliance Status'] = compliance,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['Compliance Status'] asc, ['Resource Name'] asc
```

#### Pattern 3: Private Endpoint Check

```kql
resources
| where type =~ 'paas.resource.type'
| extend privateEndpointConnections = properties.privateEndpointConnections
| extend hasPrivateEndpoint = isnotempty(privateEndpointConnections)
| extend peCount = array_length(privateEndpointConnections)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend DORACompliance = case(
    publicNetworkAccess =~ 'SecuredByPerimeter', '🔒 Network Security Perimeter',
    hasPrivateEndpoint and publicNetworkAccess =~ 'Disabled', '✅ Fully Isolated',
    hasPrivateEndpoint, '⚠️ Has PE - Public Access Enabled',
    '❌ No Private Endpoint'
)
```

---

## Troubleshooting

### Common Issues

#### Issue 1: Query Shows `resources\n| where type...` (Literal Text)

**Symptom**: Query displays with `\n` characters instead of formatting properly

**Cause**: Double backslash `\\n` in JSON instead of single `\n`

**Solution**:
1. Open Advanced Editor
2. Find the query
3. Replace all `\\n` with `\n`
4. Save and reload

#### Issue 2: ParserFailure Errors

**Symptom**: Query fails with repeated "ParserFailure" messages

**Possible Causes**:
- Using unsupported functions (`make_list()`, `make_set()`)
- Emoji encoding issues (less common after fix)
- Complex join syntax not supported by Resource Graph
- Missing or mismatched parentheses

**Solution**:
1. Test query in Resource Graph Explorer
2. Remove unsupported functions
3. Simplify query logic
4. Use `count()` instead of `make_list()` for aggregations

#### Issue 3: Empty Results

**Symptom**: Queries return no data

**Possible Causes**:
- No resources of that type in selected subscriptions/resource groups
- Incorrect parameter selection (`{ResourceGroups}` not set)
- Missing permissions on subscription

**Solution**:
1. Verify resource exists: Run simple query in Resource Graph Explorer
2. Check parameter selection (top of workbook)
3. Verify permissions: `Reader` role required
4. Check subscription context

#### Issue 4: Slow Performance

**Symptom**: Queries take a long time to load

**Possible Causes**:
- Large number of resources (1000s)
- Complex joins or aggregations
- Multiple subscriptions selected

**Solution**:
1. Filter by specific resource groups instead of `*` (all)
2. Reduce number of subscriptions
3. Optimize queries (remove unnecessary `extend` statements)
4. Use Resource Graph throttling limits awareness

#### Issue 5: JSON Validation Errors

**Symptom**: Cannot save workbook, JSON errors

**Possible Causes**:
- Missing commas between JSON objects
- Unclosed brackets or quotes
- Invalid escape sequences

**Solution**:
1. Validate JSON:
   ```powershell
   Get-Content "DORAComplianceDashboard.workbook" | ConvertFrom-Json
   ```
2. Use JSON formatter/validator (e.g., jsonlint.com)
3. Compare with backup version
4. Check for common issues:
   - Missing closing `}`
   - Extra commas at end of arrays
   - Unescaped quotes in strings

#### Issue 6: CSV Export Not Working

**Symptom**: Export button missing or not working

**Possible Causes**:
- Grid settings not configured for export
- Query returns no data

**Solution**:
1. Check grid settings include:
   ```json
   "gridSettings": {
     "formatters": [...],
     "exportFieldName": "Resource ID",
     "exportParameterName": "ExportedData"
   }
   ```
2. Verify query returns data first
3. Try different browser (some browsers block downloads)

### Debug Mode

To enable debug mode for troubleshooting:

1. Add debug output to queries:
```kql
resources
| where type =~ 'microsoft.storage/storageaccounts'
| project name, type, location, properties  // See all properties
| take 10  // Limit results for testing
```

2. Check actual parameter values:
```kql
print ResourceGroups = "{ResourceGroups}", Subscriptions = "{Subscriptions}"
```

---

## Maintenance and Updates

### Regular Maintenance Tasks

#### Monthly
- Review new Azure resource types and add to relevant queries
- Update compliance thresholds based on organizational requirements
- Check for deprecated resource types or properties
- Review DORA regulation updates from EU authorities

#### Quarterly
- Update based on Azure Policy Initiative updates
- Add new Azure services to monitoring
- Review and optimize slow queries
- Update documentation with new features

#### When Azure Releases New Features
- Network Security Perimeter support (included in v1.0)
- New private endpoint-enabled services
- New Azure Policy definitions for DORA

### Version Control

Recommended approach:

1. **Store in Git Repository**
   ```bash
   git init
   git add DORAComplianceDashboard.workbook
   git add DORAComplianceDashboard-KQLQueries.md
   git add README.md
   git commit -m "Initial DORA dashboard version"
   ```

2. **Tag Releases**
   ```bash
   git tag -a v1.0 -m "Version 1.0 - Initial release"
   git tag -a v1.1 -m "Version 1.1 - Future enhancements"
   ```

3. **Branch for Major Changes**
   ```bash
   git checkout -b feature/add-new-queries
   # Make changes
   git commit -am "Add new compliance queries"
   git checkout main
   git merge feature/add-new-queries
   ```

### Backup Strategy

1. **Export Workbook JSON Regularly**
   - Download from Azure Portal → Monitor → Workbooks → Select workbook → Advanced Editor → Copy all

2. **Store in Blob Storage**
   ```powershell
   $date = Get-Date -Format "yyyy-MM-dd"
   Copy-Item "DORAComplianceDashboard.workbook" "backups/DORACompliance_$date.json"
   ```

3. **Automate Backups**
   - Use Azure DevOps pipeline
   - Or GitHub Actions
   - Schedule weekly exports

### Update Process

When making changes:

1. **Create Backup**
   ```powershell
   Copy-Item "DORAComplianceDashboard.workbook" "DORAComplianceDashboard.workbook.backup"
   ```

2. **Test in Non-Production**
   - Create copy of workbook
   - Name it "DORA Compliance Dashboard - TEST"
   - Make changes in test version

3. **Validate Changes**
   - Test all tabs and queries
   - Verify parameter filtering works
   - Check CSV export functionality
   - Validate JSON: `Get-Content workbook.json | ConvertFrom-Json`

4. **Document Changes**
   - Update CHANGELOG.md
   - Update README.md if needed
   - Add comments in KQL queries file

5. **Deploy to Production**
   - Open production workbook
   - Advanced Editor → Replace JSON
   - Save and verify

6. **Communicate Changes**
   - Notify stakeholders
   - Update training materials
   - Document new features

### Known Limitations

1. **Resource Graph Limits**
   - Maximum 1000 results per query (use pagination if needed)
   - Throttling limits: 15 requests/10 seconds per tenant
   - Some resource properties may not be available immediately

2. **Workbook Limitations**
   - Cannot schedule automatic reports (use Azure Automation)
   - No built-in alerting (use Azure Monitor alerts separately)
   - Limited to read-only operations (cannot modify resources)

3. **Query Limitations**
   - `make_list()` and `make_set()` not supported in Resource Graph
   - Complex joins may have performance impact
   - Some resource types don't expose all properties

### Future Enhancements (Roadmap)

Potential improvements to consider:

- [ ] Add automated PDF report generation
- [ ] Integrate with Azure Security Center recommendations
- [ ] Add trend analysis (compliance over time)
- [ ] Include cost optimization recommendations
- [ ] Add Azure Landing Zone compliance checks
- [ ] Multi-tenant support (Lighthouse integration)
- [ ] Custom DORA article mappings per organization
- [ ] Integration with ServiceNow/JIRA for ticket creation
- [ ] API for programmatic access
- [ ] Mobile-friendly responsive design

---

## Support and Resources

### Official Documentation

- **DORA Regulation**: [EUR-Lex - Regulation (EU) 2022/2554](https://eur-lex.europa.eu/eli/reg/2022/2554/oj)
- **Azure Policy DORA Initiative**: Search Azure Policy for "DORA" in Azure Portal
- **Azure Resource Graph**: [Microsoft Learn - Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/)
- **Azure Workbooks**: [Microsoft Learn - Workbooks](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview)
- **Network Security Perimeter**: [Microsoft Learn - NSP Concepts](https://learn.microsoft.com/en-us/azure/private-link/network-security-perimeter-concepts)

### Community Resources

- **Azure Governance Toolkit**: Various compliance tools and scripts
- **Azure Compliance Initiative**: Community-driven policy initiatives
- **GitHub**: Search for "Azure DORA compliance" for related projects

### Getting Help

1. **Query Issues**: Test in Azure Resource Graph Explorer first
2. **JSON Issues**: Validate with JSON linter
3. **Workbook Issues**: Check Azure Monitor service health
4. **Compliance Questions**: Consult with compliance/legal team

### Contributing

If you make improvements to this workbook:
1. Document changes clearly
2. Test thoroughly
3. Share with community (if appropriate)
4. Update this README

---

## Appendix

### Azure Resource Types Covered

**Compute**:
- Virtual Machines
- Virtual Machine Scale Sets
- App Services
- Container Instances
- AKS Clusters

**Networking**:
- Load Balancers
- Public IP Addresses
- Application Gateways
- Azure Firewall
- Private Endpoints
- Network Security Groups
- Virtual Networks

**Storage**:
- Storage Accounts
- Managed Disks

**Databases**:
- SQL Servers & Databases
- PostgreSQL Flexible Servers
- MySQL Flexible Servers
- MariaDB Servers
- Cosmos DB Accounts

**Data Services**:
- Event Hubs
- Service Bus
- Redis Cache
- Container Registries

**Security**:
- Key Vaults
- Microsoft Defender for Cloud

**Monitoring**:
- Log Analytics Workspaces
- Application Insights

**AI/ML**:
- Cognitive Services
- Azure OpenAI
- API Management
- AI Search

### Glossary

- **DORA**: Digital Operational Resilience Act - EU regulation for financial sector ICT risk management
- **CMK**: Customer-Managed Keys - Encryption keys managed by customer, not Microsoft
- **NSP**: Network Security Perimeter - Logical network boundary for PaaS resources
- **PE**: Private Endpoint - Private network connection to Azure PaaS services
- **AAD**: Azure Active Directory (now Microsoft Entra ID)
- **RBAC**: Role-Based Access Control
- **NSG**: Network Security Group
- **KQL**: Kusto Query Language - Query language for Azure Resource Graph
- **HTTPS**: Hypertext Transfer Protocol Secure
- **TLS**: Transport Layer Security
- **SSL**: Secure Sockets Layer (deprecated, use TLS)

### Compliance Calculation Methods

**Percentage-Based** (used in most queries):
```
Compliance Rate = (Compliant Resources / Total Resources) × 100
```

**Risk-Based** (used in Business Continuity):
```
Risk Percentage = (Non-Compliant Resources / Total Resources) × 100
```

**Thresholds** (customizable):
- 🟢 Excellent: ≥90%
- 🟡 Good: 75-89%
- 🟠 Needs Improvement: 50-74%
- 🔴 Critical: <50%

---

## Changelog

### Version 1.2 (January 2026)
- ✅ Added PaaS Private Endpoint Coverage query
- ✅ Added Network Security Perimeter (SecuredByPerimeter) support
- ✅ Changed Business Continuity Risk to percentage-based assessment
- ✅ Fixed newline encoding issues in queries (`\n` vs `\\n`)
- ✅ Added comprehensive README documentation
- ✅ Updated KQL queries file with all standalone queries

### Version 1.1
- ✅ Added executive summary tiles with auto-sizing
- ✅ Added tag-based filtering capability
- ✅ Improved grid formatting and CSV export
- ✅ Added threshold documentation in KQL queries

### Version 1.0
- ✅ Initial release based on Azure Policy DORA Initiative v1.4.0
- ✅ 6 main tabs covering all DORA articles
- ✅ 150+ policy control assessments
- ✅ Interactive filtering and drill-down

---

## License and Disclaimer

### Disclaimer

This workbook is provided as-is without warranty. It is intended to help assess Azure resource compliance with DORA requirements but should not be considered as legal or compliance advice. 

**Important Notes**:
- This tool assesses technical controls only
- DORA compliance requires broader organizational measures
- Consult with legal/compliance professionals for full DORA compliance
- Azure Policy definitions and requirements may change
- Validate all results before using for official compliance reporting

### Usage Rights

Feel free to modify, distribute, and use this workbook for your organization's needs. Attribution appreciated but not required.

---

**Document Version**: 1.2  
**Last Updated**: January 8, 2026  
**Maintained By**: Your Organization Name  
**Contact**: compliance-team@yourorg.com
