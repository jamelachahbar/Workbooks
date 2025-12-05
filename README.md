# Azure Private Endpoints Inventory Workbook

A comprehensive Azure Workbook for monitoring and managing Private Endpoints across subscriptions and tenants, designed for enterprise environments including Financial Services institutions.

## Overview

This workbook provides complete visibility into Private Endpoint configurations across your Azure environment, enabling security teams, network administrators, and compliance officers to track and audit private connectivity.

### Screenshots

![Overview Statistics](image1.png)
*Dashboard overview showing total counts and distribution*

![Location and Connection State](pelocationstate.png)
*Private Endpoints distribution by location and connection state*

![Detailed Inventory](detailedpeinv.png)
*Detailed Private Endpoints inventory with all configuration details*

![Cross-Subscription Analysis](crosssub.png)
*Cross-subscription and cross-tenant connection analysis*

![Subscription Summary](imagesubsummary.png)
*Tenant and subscription summary with aggregated metrics*

## Features

### Multi-Tenant & Multi-Subscription Support
- **Cross-subscription visibility**: View Private Endpoints across all subscriptions in your tenant
- **Cross-tenant analysis**: Identify Private Endpoints connecting to resources in different tenants
- **Dynamic filtering**: Filter by subscription and resource group

### Comprehensive Inventory Views
1. **Overview Statistics**: Total counts, subscription distribution, resource group summary
2. **Location Analysis**: Geographic distribution of Private Endpoints
3. **Connection State Monitoring**: Track approval status (Approved, Pending, Rejected)
4. **Detailed Inventory**: Complete Private Endpoint details including:
   - Target resource information
   - Network configuration (VNet, Subnet, NIC)
   - Connection state and group IDs
   - Tenant and subscription metadata

### Advanced Analytics
- **Target Resource Type Analysis**: Breakdown by Azure service type
- **Cross-Subscription Connections**: Identify Private Endpoints connecting across subscription boundaries
- **IP Configuration Details**: Network interface and IP allocation information
- **Tenant & Subscription Summary**: Aggregated metrics per tenant/subscription

## Installation

### Prerequisites
- Azure subscription with Reader or higher permissions
- Access to Azure Monitor / Workbooks
- Permissions to query Azure Resource Graph across target subscriptions

### Deployment Steps

1. **Download the workbook**:
   ```powershell
   # Clone or download PrivateEndpointsInventory.workbook
   ```

2. **Import to Azure Portal**:
   - Navigate to **Azure Monitor** → **Workbooks**
   - Click **+ New**
   - Click the **Advanced Editor** button (</> icon)
   - Paste the contents of `PrivateEndpointsInventory.workbook`
   - Click **Apply**
   - Click **Done Editing**
   - Click **Save** and provide:
     - Title: Private Endpoints Inventory
     - Subscription, Resource Group, Location
     - Save it in a shared gallery for organizational access

3. **Configure Permissions**:
   - Ensure users have **Reader** access to subscriptions they need to monitor
   - For cross-subscription views, grant **Reader** on all relevant subscriptions

## Usage

### Basic Usage

1. Open the workbook from Azure Monitor → Workbooks
2. Select **Subscriptions** to monitor (default: All)
3. Optionally filter by **Resource Groups**
4. Review the dashboard sections

### Query Testing

Individual queries are available in `PrivateEndpointsQueries.md` for testing in Azure Resource Graph Explorer.

### Common Use Cases

**Security Audit**:
- Review "Cross-Subscription Private Endpoints" section
- Check "Connection State" for Pending or Rejected entries
- Verify Private Endpoints connect only to approved resources

**Network Planning**:
- Use "Private Endpoints by Location" for capacity planning
- Review "Target Resource Type" distribution
- Analyze IP configuration details for subnet planning

**Compliance Reporting**:
- Export "Detailed Private Endpoints Inventory" to Excel
- Use "Subscription Summary" for management reporting
- Filter by Resource Group for application-specific reports

## Architecture

### Data Source
- **Azure Resource Graph**: Queries `resources` table for real-time data
- **Resource Types Queried**:
  - `microsoft.network/privateendpoints`
  - `microsoft.network/networkinterfaces`

### Query Optimization
- Uses parameterized queries for dynamic filtering
- Efficient joins between Private Endpoints and Network Interfaces
- Optimized for large-scale environments (1000+ Private Endpoints)

## Troubleshooting

### Common Issues

**Error: "Query is invalid"**
- Ensure you have Reader permissions on the selected subscriptions
- Verify Azure Resource Graph provider is registered
- Check that parameter syntax uses `'*'` for "All" selections

**No data displayed**
- Confirm Private Endpoints exist in selected subscriptions
- Check Resource Group filter isn't excluding all resources

## Version History

### v1.0 (December 2025)
- Initial release
- Support for multi-subscription and cross-tenant scenarios
- Complete Private Endpoint inventory with IP configuration
- DORA compliance considerations documented

## License

This workbook is provided as-is for use in Azure environments. Customize as needed for your organization's requirements.

## Disclaimer

This workbook is a monitoring and visibility tool. It does not automatically enforce security policies or compliance requirements. Organizations are responsible for:
- Implementing appropriate Azure policies
- Configuring alerting and response procedures
- Maintaining compliance documentation
- Regular security audits and reviews

## References

- [Azure Private Link Documentation](https://docs.microsoft.com/azure/private-link/)
- [Azure Resource Graph Documentation](https://docs.microsoft.com/azure/governance/resource-graph/)

- [Azure Workbooks Documentation](https://docs.microsoft.com/azure/azure-monitor/visualize/workbooks-overview)
