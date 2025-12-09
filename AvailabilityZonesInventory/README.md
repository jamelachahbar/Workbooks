# 🛡️ Azure Availability Zones Compliance Dashboard

## Overview

This enhanced Azure Workbook provides a comprehensive, visually appealing dashboard to identify resources without Availability Zones enabled. It helps you improve resilience, meet high availability requirements, and ensure business continuity across your Azure environment.

**Note**: Azure Key Vault is excluded from this dashboard as it provides zone redundancy automatically by default in supported regions and does not require explicit configuration.

## Purpose

The workbook provides comprehensive visibility into:
- 🔍 **Identify Risk**: Discover resources without zone redundancy
- 📊 **Improve Resilience**: Enhance application availability and fault tolerance
- ✅ **Meet SLAs**: Ensure compliance with high availability requirements
- 🎯 **Optimize Architecture**: Make informed decisions about zone-redundant deployments
- 📈 **Visual Analytics**: Beautiful charts and metrics for executive reporting
- 🌐 **Wide Coverage**: Analyzes 18 Azure service types across multiple categories
- 🔑 **Smart Exclusions**: Automatically excludes Key Vault (zone redundant by default)

## Features

### Dashboard Sections

#### 📈 Executive Summary
1. **Overall Zone Compliance Rate**
   - Percentage of resources with Availability Zones enabled
   - Total resources analyzed across all types
   - Visual metric tile with blue color scheme

2. **Resources Without Availability Zones**
   - Total count of non-compliant resources
   - Critical metric for risk assessment
   - Visual metric tile with red color scheme

3. **Affected Resource Types**
   - Number of different resource types requiring attention
   - Helps prioritize remediation efforts
   - Visual metric tile with orange color scheme

#### 📋 Detailed Inventories

4. **💻 Virtual Machines Without Availability Zones**
   - VMs not deployed across availability zones
   - VM size, OS type, and power state details
   - Identifies potential single points of failure

5. **⚖️ Load Balancers Without Availability Zones**
   - Standard SKU load balancers without zone redundancy
   - Frontend and backend pool configuration
   - SKU tier visualization

6. **🌐 Public IP Addresses Without Availability Zones**
   - Public IPs not configured for zone redundancy
   - Allocation method and associated resources
   - Standard vs Basic SKU identification

7. **💾 Storage Accounts Without Zone-Redundant Replication**
   - Storage accounts using LRS, GRS, or RA-GRS
   - Excludes ZRS, GZRS, and RA-GZRS storage types
   - Storage kind, access tier, and location details

8. **🗄️ Database Services Without Availability Zones**
   - SQL Databases, PostgreSQL, MySQL, and Redis Cache
   - Zone redundancy configuration status
   - **Note**: These require explicit zone redundancy configuration - not enabled by default
   - Service tier and SKU information

9. **🐳 Container and Kubernetes Services**
   - AKS Clusters without zone-redundant node pools
   - Container Apps without zone redundancy
   - Tier and environment configuration

10. **🚀 App Services and API Management**
    - App Services without zone redundancy
    - **Note**: App Service zone redundancy requires manual configuration (Premium v2/v3/v4 or Isolated v2 SKUs)
    - API Management instances without zones
    - SKU and pricing tier details

11. **🔒 Network Security and Gateway Services**
    - Application Gateways without zone redundancy
    - Azure Firewalls without zone configuration
    - Security tier and SKU information

12. **📨 Messaging and Integration Services**
    - Event Hubs namespaces without zones
    - Service Bus namespaces without zones
    - **Note**: Key Vault excluded (zone redundant by default)

#### 📊 Analytics & Summary

13. **Distribution by Resource Type**
    - Interactive bar chart showing non-compliant resources by type
    - Friendly resource type names
    - Count-based visualization

14. **Distribution by Location**
    - Interactive pie chart showing regional distribution
    - Identifies regions with most non-compliant resources
    - Helps prioritize regional improvements

15. **Compliance by Subscription**
    - Detailed grid with per-subscription metrics
    - Compliance rate percentage with color coding
    - Total resources, compliant vs non-compliant counts
    - Resource type diversity per subscription

## Resource Types Covered (18 Types)

**Note**: Azure Key Vault is intentionally excluded from this analysis as it provides zone redundancy automatically by default in supported regions.

### Compute Services
- ☁️ Virtual Machines
- 📦 Virtual Machine Scale Sets
- 🐳 AKS (Azure Kubernetes Service) Clusters
- 📱 Container Apps

### Network Services
- ⚖️ Load Balancers (Standard SKU)
- 🌐 Public IP Addresses
- 🔒 Application Gateways
- 🛡️ Azure Firewalls

### Storage Services
- 💾 Storage Accounts (all tiers)

### Database Services
- 🗄️ SQL Databases
- 🐘 PostgreSQL Flexible Servers
- 🐬 MySQL Flexible Servers
- 🔴 Redis Cache

### Application Services
- 🚀 App Services (Web Apps, Function Apps, API Apps)
- 🔌 API Management Services

### Messaging & Integration
- 📨 Event Hubs Namespaces
- 🚌 Service Bus Namespaces

## How to Use

### Installation

1. Open Azure Portal
2. Navigate to **Azure Workbooks**
3. Click **+ New** or **Empty**
4. Click **</>** (Advanced Editor) in the toolbar
5. Paste the contents of `AvailabilityZonesInventory.workbook`
6. Click **Apply**
7. Save the workbook to your subscription

### Filtering Options

The workbook provides two parameter filters:

- **Subscriptions**: Select one or more subscriptions (default: all)
- **Resource Groups**: Filter by specific resource groups (default: all)

### Interpreting Results

**Visual Enhancements:**
- 📈 Executive Summary tiles provide at-a-glance metrics
- 🎨 Color-coded formatters for quick status identification
- 📊 Interactive charts for data exploration
- 🔍 Clickable resource names link directly to Azure Portal
- ➖ Visual separators between major sections for clarity

**Color Coding:**
- 🔴 Red: Critical - Locally redundant only (LRS) or high-risk states
- 🟡 Yellow/Orange: Warning - Geo-redundant but not zone-redundant (GRS, RA-GRS)
- 🟢 Green: Success - Compliant resources or healthy states
- 🔵 Blue: Information - Compliance rate and summary metrics
- ⚪ Gray: Unknown/Other states

**Compliance Rate:**
- Percentage of resources with Availability Zones enabled
- Higher percentages indicate better resilience posture
- Target: 80%+ for production environments
- Critical workloads should aim for 95%+ compliance

## Storage Account SKU Types

### Non-Zone-Redundant (Shown in Report)
- **Standard_LRS**: Locally Redundant Storage
- **Standard_GRS**: Geo-Redundant Storage
- **Standard_RAGRS**: Read-Access Geo-Redundant Storage
- **Premium_LRS**: Premium Locally Redundant Storage

### Zone-Redundant (Excluded from Report)
- **Standard_ZRS**: Zone-Redundant Storage
- **Standard_GZRS**: Geo-Zone-Redundant Storage
- **Premium_ZRS**: Premium Zone-Redundant Storage
- **Standard_RAGZRS**: Read-Access Geo-Zone-Redundant Storage

## Azure Key Vault and Zone Redundancy

Azure Key Vault is **intentionally excluded** from this workbook's compliance checks because it handles zone redundancy differently than other Azure services:

### Why Key Vault Is Excluded

- **Zone Redundancy by Default**: Azure Key Vault automatically provides zone redundancy in all supported regions where availability zones are available. This is built into the service and does not require explicit configuration.
- **No Configuration Needed**: Unlike VMs, Load Balancers, or other infrastructure resources, you don't need to specify zones or enable zone redundancy when creating a Key Vault.
- **Transparent Replication**: Key Vault contents are automatically replicated across availability zones within the region and to paired regions for disaster recovery.
- **Always Compliant**: Since zone redundancy is automatic, Key Vaults are inherently compliant with availability zone best practices in supported regions.

### Microsoft Documentation

According to [Microsoft Learn documentation on Key Vault reliability](https://learn.microsoft.com/en-us/azure/reliability/reliability-key-vault):
> "By default, Key Vault achieves redundancy by replicating your key vault and its contents within the region and to a paired region."

This automatic redundancy means Key Vault should not be flagged as non-compliant in availability zone assessments.

## Azure App Service and Zone Redundancy

Azure App Service requires **explicit configuration** to enable zone redundancy, and it is **not enabled by default**:

### App Service Zone Redundancy Behavior

- **Manual Configuration Required**: Zone redundancy must be explicitly enabled during App Service Plan creation or upgrade. It is NOT automatic.
- **SKU Requirements**: Only Premium v2, Premium v3, Premium v4, and Isolated v2 SKUs support zone redundancy. Basic and Standard tiers do not support this feature.
- **Instance Requirements**: Minimum of 2 instances required. With 2+ instances, you qualify for the 99.99% SLA.
- **Region Support**: Only available in regions that support availability zones and have zone-redundant scale units.
- **Configuration Timing**: Zone redundancy must be set during plan creation for new apps, or can be enabled for existing plans if they meet the requirements.

### How Zone Redundancy is Detected

This workbook checks for App Services without zone redundancy by examining the `zones` property, which is only populated when zone redundancy is explicitly enabled.

### Microsoft Documentation

According to [Microsoft Learn documentation on App Service zone redundancy](https://learn.microsoft.com/en-us/azure/app-service/configure-zone-redundancy):
> Zone redundancy must be enabled when creating a new App Service plan. Once enabled, instances of your app are automatically distributed across available zones.

**Important**: App Services flagged by this workbook require manual action to enable zone redundancy if high availability is needed.

## Azure Database Services and Zone Redundancy

Database services have different zone redundancy behaviors depending on the specific service:

### Azure SQL Database

- **NOT enabled by default**: Zone redundancy must be explicitly configured during database creation or via database settings.
- **Local redundancy is default**: By default, SQL Database uses local redundancy (multiple copies within same datacenter).
- **Tier requirements**: Available for Premium, Business Critical, General Purpose (select regions), and Hyperscale tiers.
- **DTU tiers**: Basic and Standard DTU tiers do NOT support zone redundancy.
- **Property checked**: This workbook detects zone redundancy via the `properties.zoneRedundant` property.

### Azure Database for PostgreSQL Flexible Server

- **Zone redundancy available with HA**: When enabling High Availability, zone-redundant HA is offered as the preferred option if the region supports availability zones.
- **HA must be enabled**: High Availability itself must be explicitly enabled - it is not automatic.
- **Standby in different zone**: Zone-redundant HA places the primary and standby servers in different availability zones.
- **Tier requirements**: Available for General Purpose and Business Critical tiers, NOT for Burstable tier.
- **Must enable at creation**: High availability (including zone redundancy) must be selected during server creation.
- **Property checked**: This workbook detects via `properties.highAvailability.mode == 'ZoneRedundant'`.

### Azure Database for MySQL Flexible Server

- **Zone redundancy available with HA**: Zone-redundant HA is an available option when creating servers in supported regions.
- **HA must be enabled**: High Availability must be explicitly enabled during server creation.
- **Must enable at creation**: Zone-redundant HA must be selected during server creation and cannot be changed afterwards (only disabling HA is supported).
- **Tier requirements**: Available for General Purpose and Business Critical tiers only.
- **Property checked**: This workbook detects via `properties.highAvailability.mode == 'ZoneRedundant'`.

### Redis Cache

- **Zone redundancy available**: Premium and Enterprise tiers support zone redundancy in supported regions.
- **NOT enabled by default**: Must be explicitly configured during cache creation.
- **Property checked**: Detected via the `zones` property when explicitly configured.

### Microsoft Documentation

- [Azure SQL Database zone redundancy](https://learn.microsoft.com/en-us/azure/azure-sql/database/high-availability-sla-local-zone-redundancy)
- [PostgreSQL Flexible Server HA](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-high-availability)
- [MySQL Flexible Server HA](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/concepts-high-availability)

**Summary**: All database services in this workbook require explicit zone redundancy configuration and are correctly flagged when not configured.

## Best Practices

1. **Virtual Machines**: Deploy critical VMs across availability zones (zones 1, 2, 3)
2. **Load Balancers**: Use Standard SKU with zone-redundant frontend IPs
3. **Storage Accounts**: Use ZRS or GZRS for production workloads requiring high availability
4. **Databases**: 
   - **SQL Database**: Enable zone redundancy in Premium, Business Critical, or General Purpose tiers
   - **PostgreSQL/MySQL Flexible Server**: Enable zone-redundant high availability during server creation (General Purpose or Business Critical tiers)
   - **Redis Cache**: Configure zone redundancy in Premium or Enterprise tiers
5. **App Services**: Enable zone redundancy for Premium v2/v3/v4 or Isolated v2 plans with minimum 2 instances
6. **Public IPs**: Configure Standard SKU public IPs with zone redundancy
7. **Key Vault**: No action needed - zone redundancy is automatic in supported regions

## Limitations

- Only shows resources in regions that support Availability Zones
- Some resource types may have zone configuration in properties not detected by this workbook
- Database zone redundancy detection depends on specific property availability (uses `zoneRedundant` and `highAvailability.mode` properties)
- **Azure Key Vault is excluded** as it provides zone redundancy by default in supported regions
- **App Services** require explicit zone redundancy configuration (Premium v2/v3/v4 or Isolated v2 SKUs with 2+ instances)
- **Database services** (SQL, PostgreSQL, MySQL) require explicit zone redundancy or HA configuration - not enabled by default
- Requires appropriate RBAC permissions to query resources

## Required Permissions

- **Reader** role at subscription or resource group level
- Access to Azure Resource Graph

## Export Capabilities

Each grid in the workbook supports:
- Export to CSV
- Export to Excel
- Up to 10,000 rows per export
- Filtering and sorting before export

## Troubleshooting

### No Data Displayed
- Verify subscription and resource group filters are correctly set
- Ensure you have Reader permissions on the selected scope
- Check that resources exist in the selected subscriptions

### Query Errors
- Ensure you're using the latest workbook version
- Verify Azure Resource Graph API is accessible
- Check for any Azure service outages

### Storage Account Query Issues
The storage account section uses KQL operators that filter out zone-redundant SKUs. If you encounter parsing errors:
- Verify the query uses separate `where` clauses for `!contains 'ZRS'` and `!contains 'GZRS'`
- The `kind` field is aliased to `storageKind` to avoid KQL keyword conflicts

## Contributing

To improve this workbook:
1. Test changes in Azure Resource Graph Explorer first
2. Validate KQL syntax compatibility with Azure Resource Graph
3. Document any new resource types or properties added

## Version History

- **v2.3** (2025-12-09): Enhanced Documentation for App Services and Databases
  - 📚 Added comprehensive documentation sections for App Service zone redundancy behavior
  - 📚 Added detailed database services zone redundancy documentation (SQL, PostgreSQL, MySQL, Redis)
  - 📝 Clarified that App Services require explicit zone redundancy configuration (Premium SKUs)
  - 📝 Documented database zone redundancy requirements and detection methods
  - ✅ Updated Best Practices section with specific guidance for each service type
  - 🔍 No query changes - existing detection logic already correct

- **v2.2** (2025-12-09): Key Vault Exclusion Fix
  - 🔧 Removed Key Vault from zone redundancy compliance checks
  - 📝 Key Vault provides zone redundancy by default in supported regions
  - ✅ Prevents false non-compliance reports for Key Vault resources
  - 📊 Updated coverage from 19 to 18 service types
  - 📖 Added comprehensive documentation explaining Key Vault's automatic zone redundancy

- **v2.0** (2025-12-05): Enhanced UI and Expanded Coverage
  - 🎨 Beautiful dashboard with emoji-enhanced sections
  - 📈 Reorganized layout with Executive Summary first
  - 🔢 Expanded from 10 to 19 resource types (90% increase)
  - 📊 Added 13 comprehensive sections (3 summary + 9 inventories + 3 analytics)
  - ➕ New services: AKS, Container Apps, App Services, API Management, Application Gateway, Azure Firewall, Event Hubs, Service Bus, Key Vault
  - ✨ Professional formatting with visual separators
  - 🎯 Enhanced analytics with detailed subscription compliance grid

- **v1.0** (2025-12-05): Initial release
  - 10 dashboard sections
  - 10 resource types covered
  - Export support on all grids
  - Subscription and resource group filtering

## Support

For issues or questions:
- Review the Azure Resource Graph documentation
- Check Azure Workbooks documentation
- Verify KQL syntax in Resource Graph Explorer

## Related Resources

- [Azure Availability Zones Documentation](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)
- [Azure Resource Graph Query Language](https://learn.microsoft.com/en-us/azure/governance/resource-graph/concepts/query-language)
- [Azure Workbooks Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview)
- [Storage Redundancy Options](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
