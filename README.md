# Azure Monitor Workbooks Collection

A collection of Azure Monitor Workbooks for governance, security, and compliance — all powered by Azure Resource Graph (KQL) queries.

---

## Workbooks

### [DORA Compliance Dashboard](DORAComplianceDashboard/)

Comprehensive assessment of your Azure environment's compliance with the **Digital Operational Resilience Act (DORA) - EU 2022/2554**. Covers 150+ policy controls mapped to DORA articles 9.3a–9.3d, 10.1–10.2 with executive summary tiles, configurable thresholds, and resource-level drill-down.

- **Key features**: Executive summary, article-specific tabs, compliance rate scoring, CSV export
- **Resource types**: VMs, Storage, SQL, Cosmos DB, AKS, ACR, Key Vault, Redis, Event Hub, Service Bus, App Service, APIM, Cognitive Services, AI Search

### [Private Endpoints Inventory](PrivateEndpointsInventory/)

Complete visibility into Private Endpoint configurations across subscriptions and tenants. Designed for security teams and compliance officers in enterprise/financial services environments.

- **Key features**: Cross-subscription and cross-tenant analysis, connection state tracking, location distribution, detailed inventory
- **Resource types**: All resources with private endpoint connections

### [Availability Zones Inventory](AvailabilityZonesInventory/)

Dashboard to identify resources without Availability Zones enabled. Covers 18 Azure service types across Compute, Containers, Networking, Storage, Databases, and Integration categories.

- **Key features**: Overall zone compliance rate, per-category breakdown, executive summary tiles
- **Smart exclusions**: Automatically excludes Key Vault (zone redundant by default)

### [Managed Identity & Firewall Dashboard](ManagedIdentityFirewallDashboard/)

Cross-subscription visibility into managed identity permissions and resource-level firewall/network access posture for PaaS resources.

- **Key features**: MI inventory, high-privilege role audit, cross-subscription assignments, unified firewall overview, IP/VNet whitelist drill-down
- **Resource types**: 14 PaaS types (Storage, Key Vault, SQL, Cosmos DB, ACR, Redis, Event Hub, Service Bus, App Service, Cognitive Services, APIM, AI Search, PostgreSQL/MySQL Flex)

---

## Standalone References

| File | Description |
|------|-------------|
| [ResourceGraphQueries-ManagedIdentity-Firewall.md](ResourceGraphQueries-ManagedIdentity-Firewall.md) | 8 standalone KQL queries for managed identity permissions and resource firewall visibility |
| [DORAComplianceDashboard-KQLQueries.md](DORAComplianceDashboard/DORAComplianceDashboard-KQLQueries.md) | DORA compliance queries with scoring |
| [DORAComplianceDashboard-KQLQueries-Simplified.md](DORAComplianceDashboard/DORAComplianceDashboard-KQLQueries-Simplified.md) | DORA raw data queries without scoring |
| [PrivateEndpointsQueries.md](PrivateEndpointsInventory/PrivateEndpointsQueries.md) | Private Endpoints KQL queries |
| [WORKBOOK_DEVELOPMENT_GUIDE.md](WORKBOOK_DEVELOPMENT_GUIDE.md) | Comprehensive guide for creating Azure Workbooks — structure, parameters, tabs, ARG queries, grid config, common pitfalls, and templates |

---

## Deployment

All workbooks follow the same deployment pattern:

1. Navigate to **Azure Monitor** > **Workbooks** in the Azure Portal
2. Click **+ New**
3. Click the **Advanced Editor** button (`</>`)
4. Paste the contents of the `.workbook` file
5. Click **Apply**, then **Save**

### Validate JSON before importing

```powershell
Get-Content "path/to/workbook.workbook" | ConvertFrom-Json | Out-Null
```

---

## Repository Structure

```
Workbooks/
  AvailabilityZonesInventory/        AZ compliance
  DORAComplianceDashboard/           DORA EU 2022/2554 compliance
  ManagedIdentityFirewallDashboard/  MI permissions & firewall posture
  PrivateEndpointsInventory/         Private endpoint inventory
  ResourceGraphQueries-ManagedIdentity-Firewall.md
  WORKBOOK_DEVELOPMENT_GUIDE.md
```

Each workbook folder contains:
- `.workbook` — the Azure Monitor Workbook JSON (importable)
- `README.md` — features, deployment, and customization docs
- Optional KQL query reference files (`.md`)
