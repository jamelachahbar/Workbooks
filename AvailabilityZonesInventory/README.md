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
   - Service tier and SKU information

9. **🐳 Container and Kubernetes Services**
   - AKS Clusters without zone-redundant node pools
   - Container Apps without zone redundancy
   - Tier and environment configuration

10. **🚀 App Services and API Management**
    - App Services without zone redundancy
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

## Best Practices

1. **Virtual Machines**: Deploy critical VMs across availability zones (zones 1, 2, 3)
2. **Load Balancers**: Use Standard SKU with zone-redundant frontend IPs
3. **Storage Accounts**: Use ZRS or GZRS for production workloads requiring high availability
4. **Databases**: Enable zone redundancy for business-critical database services
5. **Public IPs**: Configure Standard SKU public IPs with zone redundancy

## Limitations

- Only shows resources in regions that support Availability Zones
- Some resource types may have zone configuration in properties not detected by this workbook
- Database zone redundancy detection depends on specific property availability
- **Azure Key Vault is excluded** as it provides zone redundancy by default in supported regions
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
