# Managed Identity & Resource Firewall Dashboard

**Version 1.0** | March 2026

Azure Monitor Workbook providing cross-subscription visibility into managed identity permissions and resource-level firewall/network access posture for PaaS resources.

---

## Features

### Tab 1 — Managed Identities
- **User-Assigned MI Inventory**: Lists all user-assigned managed identities with principal ID, client ID, subscription, and resource group.
- **System-Assigned MI Inventory**: Lists all resources with system-assigned identities enabled, with friendly resource type names.

### Tab 2 — High-Privilege Assignments
- Identifies service principals with **Owner**, **Contributor**, or **User Access Administrator** roles.
- Risk classification: Critical / High / Medium based on role + scope combination.
- Conditional formatting with severity icons.

### Tab 3 — Cross-Subscription
- **User-Assigned MIs**: Finds identities with role assignments in a different subscription than where the identity lives.
- **System-Assigned MIs**: Same analysis for system-assigned identities.

### Tab 4 — Firewall Overview
- Unified view of **14 PaaS resource types** with access classification:
  - Secured by Perimeter, Fully Private, Private (No PE), Semi-Public (Whitelisted), Fully Public, Unknown
- Shows firewall default action, IP/VNet rule counts, private endpoint count, trusted services bypass.
- Sorted by risk (Fully Public first).

### Tab 5 — IP Whitelist
- Drill-down into actual **IP addresses / CIDR ranges** whitelisted per resource.
- One row per IP per resource using `mv-expand`.
- Classifies each entry as Single IP or CIDR Range.

### Tab 6 — VNet Whitelist
- Drill-down into **VNet/Subnet service endpoint rules** per resource.
- Flags cross-subscription VNets for security review.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| **Subscriptions** | Subscription picker | Multi-select, defaults to all |
| **ResourceGroups** | Dropdown | Multi-select, populated from Resource Graph, defaults to all |

---

## Deployment

### Import via Azure Portal

1. Navigate to **Azure Monitor** > **Workbooks**
2. Click **+ New**
3. Click the **Advanced Editor** button (`</>`)
4. Copy the contents of `ManagedIdentityFirewallDashboard.workbook`
5. Paste into the editor and click **Apply**
6. Click **Save** and choose a name/location

### Supported Resource Types

| Tab | Resource Types |
|-----|---------------|
| Managed Identities | `microsoft.managedidentity/userassignedidentities`, all resources with system identity |
| High-Privilege | `microsoft.authorization/roleassignments` (ServicePrincipal type) |
| Firewall Overview | Storage, Key Vault, SQL Server, PostgreSQL/MySQL Flex, Cosmos DB, ACR, Redis, Event Hub, Service Bus, App Service, Cognitive Services, APIM, AI Search |
| IP Whitelist | Storage, Key Vault, SQL Server, Cosmos DB, ACR, Cognitive Services, AI Search |
| VNet Whitelist | Storage, Key Vault, Cosmos DB, ACR |

---

## Validation

Validate JSON syntax before importing:

```powershell
Get-Content "ManagedIdentityFirewallDashboard.workbook" | ConvertFrom-Json | Out-Null
```

---

## Related Files

- [ResourceGraphQueries-ManagedIdentity-Firewall.md](../ResourceGraphQueries-ManagedIdentity-Firewall.md) — Standalone KQL queries used in this workbook

---

*Last Updated: March 2026*
