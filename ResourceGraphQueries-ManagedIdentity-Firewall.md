# Azure Resource Graph Queries - Managed Identity Permissions & Resource Firewall Visibility

**Version 1.0** | March 2026

---

## ✅ Ready for Resource Graph Explorer

All queries work directly in Azure Resource Graph Explorer:

1. Open [Azure Portal](https://portal.azure.com) → **Resource Graph Explorer**
2. Copy any query from this document
3. Paste and click **Run query**
4. Export results as CSV/JSON if needed

---

## Table of Contents

### Part 1 — Managed Identity Cross-Subscription Permissions
- [Azure Resource Graph Queries - Managed Identity Permissions \& Resource Firewall Visibility](#azure-resource-graph-queries---managed-identity-permissions--resource-firewall-visibility)
  - [✅ Ready for Resource Graph Explorer](#-ready-for-resource-graph-explorer)
  - [Table of Contents](#table-of-contents)
    - [Part 1 — Managed Identity Cross-Subscription Permissions](#part-1--managed-identity-cross-subscription-permissions)
    - [Part 2 — Resource-Level Firewall \& Network Access Visualization](#part-2--resource-level-firewall--network-access-visualization)
- [Part 1 — Managed Identity Cross-Subscription Permissions](#part-1--managed-identity-cross-subscription-permissions-1)
  - [1. User-Assigned Managed Identity Inventory](#1-user-assigned-managed-identity-inventory)
  - [2. System-Assigned Managed Identity Inventory](#2-system-assigned-managed-identity-inventory)
  - [3. All Role Assignments for Managed Identities](#3-all-role-assignments-for-managed-identities)
  - [4. Cross-Subscription Role Assignments (User-Assigned)](#4-cross-subscription-role-assignments-user-assigned)
  - [5. Cross-Subscription Role Assignments (System-Assigned)](#5-cross-subscription-role-assignments-system-assigned)
  - [6. High-Privilege Managed Identity Assignments](#6-high-privilege-managed-identity-assignments)
- [Part 2 — Resource-Level Firewall \& Network Access Visualization](#part-2--resource-level-firewall--network-access-visualization-1)
  - [7. Unified Resource Network Access Overview](#7-unified-resource-network-access-overview)
  - [8. Firewall IP Whitelist Detail (Drill-Down)](#8-firewall-ip-whitelist-detail-drill-down)
    - [8a. IP Address Whitelist per Resource](#8a-ip-address-whitelist-per-resource)
    - [8b. VNet Service Endpoint Whitelist per Resource](#8b-vnet-service-endpoint-whitelist-per-resource)
  - [Azure Resource Graph Limitations](#azure-resource-graph-limitations)

### Part 2 — Resource-Level Firewall & Network Access Visualization
7. [Unified Resource Network Access Overview](#7-unified-resource-network-access-overview)
8. [Firewall IP Whitelist Detail (Drill-Down)](#8-firewall-ip-whitelist-detail-drill-down)

---

# Part 1 — Managed Identity Cross-Subscription Permissions

## 1. User-Assigned Managed Identity Inventory

Lists all user-assigned managed identities across your subscriptions with their principal IDs.

```kql
resources
| where type =~ 'microsoft.managedidentity/userassignedidentities'
| extend principalId = tostring(properties.principalId)
| extend clientId = tostring(properties.clientId)
| extend tenantId = tostring(properties.tenantId)
| project
    ['Identity Name'] = name,
    ['Principal ID'] = principalId,
    ['Client ID'] = clientId,
    ['Subscription'] = subscriptionId,
    ['Resource Group'] = resourceGroup,
    ['Location'] = location,
    ['Tenant ID'] = tenantId,
    ['Resource ID'] = id
| order by ['Identity Name'] asc
```

---

## 2. System-Assigned Managed Identity Inventory

Lists all resources that have a system-assigned managed identity enabled.

```kql
resources
| where isnotempty(identity)
| where tostring(identity.type) has 'SystemAssigned'
| extend principalId = tostring(identity.principalId)
| extend tenantId = tostring(identity.tenantId)
| extend identityType = tostring(identity.type)
| extend friendlyType = case(
    type =~ 'microsoft.compute/virtualmachines', 'Virtual Machine',
    type =~ 'microsoft.web/sites', 'App Service',
    type =~ 'microsoft.containerservice/managedclusters', 'AKS Cluster',
    type =~ 'microsoft.app/containerapps', 'Container App',
    type =~ 'microsoft.sql/servers', 'SQL Server',
    type =~ 'microsoft.databricks/workspaces', 'Databricks',
    type =~ 'microsoft.logic/workflows', 'Logic App',
    type =~ 'microsoft.datafactory/factories', 'Data Factory',
    type =~ 'microsoft.cognitiveservices/accounts', 'Cognitive Services',
    type =~ 'microsoft.apimanagement/service', 'API Management',
    type =~ 'microsoft.keyvault/vaults', 'Key Vault',
    type =~ 'microsoft.containerregistry/registries', 'Container Registry',
    type
)
| project
    ['Resource Name'] = name,
    ['Resource Type'] = friendlyType,
    ['Identity Type'] = identityType,
    ['Principal ID'] = principalId,
    ['Subscription'] = subscriptionId,
    ['Resource Group'] = resourceGroup,
    ['Location'] = location,
    ['Resource ID'] = id
| order by ['Resource Type'] asc, ['Resource Name'] asc
```

---

## 3. All Role Assignments for Managed Identities

Shows all role assignments where the principal type is `ServicePrincipal` (which includes managed identities, app registrations, etc.). Includes a case-mapped role name for the most common built-in roles.

```kql
authorizationresources
| where type =~ 'microsoft.authorization/roleassignments'
| extend principalType = tostring(properties.principalType)
| where principalType =~ 'ServicePrincipal'
| extend principalId = tostring(properties.principalId)
| extend roleDefinitionId = tostring(properties.roleDefinitionId)
| extend roleGuid = tostring(split(roleDefinitionId, '/')[-1])
| extend scope = tostring(properties.scope)
| extend createdOn = tostring(properties.createdOn)
| extend roleName = case(
    roleGuid =~ '8e3af657-a8ff-443c-a75c-2fe8c4bcb635', 'Owner',
    roleGuid =~ 'b24988ac-6180-42a0-ab88-20f7382dd24c', 'Contributor',
    roleGuid =~ 'acdd72a7-3385-48ef-bd42-f606fba81ae7', 'Reader',
    roleGuid =~ '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9', 'User Access Administrator',
    roleGuid =~ 'ba92f5b4-2d11-453d-a403-e96b0029c9fe', 'Storage Blob Data Contributor',
    roleGuid =~ '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1', 'Storage Blob Data Reader',
    roleGuid =~ '00482a5a-887f-4fb3-b363-3b7fe8e74483', 'Key Vault Administrator',
    roleGuid =~ '4633458b-17de-408a-b874-0445c86b69e6', 'Key Vault Secrets User',
    roleGuid =~ 'a4417e6f-fecd-4de8-b567-7b0420556985', 'Key Vault Crypto User',
    roleGuid =~ '21090545-7ca7-4776-b22c-e363652d74d2', 'Key Vault Reader',
    roleGuid =~ '090c5cfd-751d-490a-894a-3ce6f1109419', 'Azure Service Bus Data Owner',
    roleGuid =~ 'f526a384-b230-433a-b45c-95f59c4a2dec', 'Azure Event Hubs Data Owner',
    roleGuid =~ '9980e02c-c2be-4d73-94e8-173b1dc7cf3c', 'Virtual Machine Contributor',
    roleGuid =~ 'de139f84-1756-47ae-9be6-808fbbe84772', 'Website Contributor',
    roleGuid =~ 'e147488a-f6f5-4113-8e2d-b22465e65bf6', 'Key Vault Crypto Service Encryption User',
    strcat('Custom/Other (', roleGuid, ')')
)
| extend scopeLevel = case(
    scope matches regex '^/subscriptions/[^/]+$', 'Subscription',
    scope matches regex '^/subscriptions/[^/]+/resourceGroups/[^/]+$', 'Resource Group',
    scope matches regex '^/$', 'Root',
    scope matches regex '^/providers/Microsoft.Management/managementGroups/', 'Management Group',
    'Resource'
)
| project
    ['Principal ID'] = principalId,
    ['Role'] = roleName,
    ['Scope Level'] = scopeLevel,
    ['Scope'] = scope,
    ['Assignment Subscription'] = subscriptionId,
    ['Created On'] = createdOn,
    ['Role Definition ID'] = roleDefinitionId,
    ['Assignment ID'] = id
| order by ['Role'] asc, ['Scope Level'] asc
```

---

## 4. Cross-Subscription Role Assignments (User-Assigned)

**Key query**: Finds user-assigned managed identities that have role assignments in a **different subscription** than where the identity itself lives. This is critical for understanding cross-subscription trust relationships.

```kql
resources
| where type =~ 'microsoft.managedidentity/userassignedidentities'
| extend mi_principalId = tostring(properties.principalId)
| project mi_principalId, mi_name = name, mi_subscription = subscriptionId, mi_resourceGroup = resourceGroup, mi_id = id
| join kind=inner (
    authorizationresources
    | where type =~ 'microsoft.authorization/roleassignments'
    | where tostring(properties.principalType) =~ 'ServicePrincipal'
    | extend ra_principalId = tostring(properties.principalId)
    | extend ra_roleDefinitionId = tostring(properties.roleDefinitionId)
    | extend ra_roleGuid = tostring(split(ra_roleDefinitionId, '/')[-1])
    | extend ra_scope = tostring(properties.scope)
    | extend ra_subscription = subscriptionId
    | extend ra_roleName = case(
        ra_roleGuid =~ '8e3af657-a8ff-443c-a75c-2fe8c4bcb635', 'Owner',
        ra_roleGuid =~ 'b24988ac-6180-42a0-ab88-20f7382dd24c', 'Contributor',
        ra_roleGuid =~ 'acdd72a7-3385-48ef-bd42-f606fba81ae7', 'Reader',
        ra_roleGuid =~ '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9', 'User Access Administrator',
        ra_roleGuid =~ 'ba92f5b4-2d11-453d-a403-e96b0029c9fe', 'Storage Blob Data Contributor',
        ra_roleGuid =~ '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1', 'Storage Blob Data Reader',
        ra_roleGuid =~ '00482a5a-887f-4fb3-b363-3b7fe8e74483', 'Key Vault Administrator',
        ra_roleGuid =~ '4633458b-17de-408a-b874-0445c86b69e6', 'Key Vault Secrets User',
        strcat('Other (', ra_roleGuid, ')')
    )
    | project ra_principalId, ra_roleName, ra_scope, ra_subscription, ra_roleGuid
) on $left.mi_principalId == $right.ra_principalId
| where mi_subscription != ra_subscription
| project
    ['Identity Name'] = mi_name,
    ['Identity Subscription'] = mi_subscription,
    ['Identity Resource Group'] = mi_resourceGroup,
    ['Role Assigned'] = ra_roleName,
    ['Assignment Subscription'] = ra_subscription,
    ['Assignment Scope'] = ra_scope,
    ['Cross-Sub'] = 'Yes',
    ['Identity Resource ID'] = mi_id
| order by ['Identity Name'] asc, ['Role Assigned'] asc
```

---

## 5. Cross-Subscription Role Assignments (System-Assigned)

Same concept but for **system-assigned** managed identities. Finds resources whose system-assigned identity has been granted roles in other subscriptions.

```kql
resources
| where isnotempty(identity)
| where tostring(identity.type) has 'SystemAssigned'
| extend mi_principalId = tostring(identity.principalId)
| extend mi_resourceType = case(
    type =~ 'microsoft.compute/virtualmachines', 'VM',
    type =~ 'microsoft.web/sites', 'App Service',
    type =~ 'microsoft.containerservice/managedclusters', 'AKS',
    type =~ 'microsoft.app/containerapps', 'Container App',
    type =~ 'microsoft.sql/servers', 'SQL Server',
    type =~ 'microsoft.datafactory/factories', 'Data Factory',
    type =~ 'microsoft.logic/workflows', 'Logic App',
    type =~ 'microsoft.cognitiveservices/accounts', 'Cognitive Services',
    type =~ 'microsoft.apimanagement/service', 'APIM',
    type
)
| project mi_principalId, mi_name = name, mi_resourceType, mi_subscription = subscriptionId, mi_resourceGroup = resourceGroup, mi_id = id
| join kind=inner (
    authorizationresources
    | where type =~ 'microsoft.authorization/roleassignments'
    | where tostring(properties.principalType) =~ 'ServicePrincipal'
    | extend ra_principalId = tostring(properties.principalId)
    | extend ra_roleDefinitionId = tostring(properties.roleDefinitionId)
    | extend ra_roleGuid = tostring(split(ra_roleDefinitionId, '/')[-1])
    | extend ra_scope = tostring(properties.scope)
    | extend ra_subscription = subscriptionId
    | extend ra_roleName = case(
        ra_roleGuid =~ '8e3af657-a8ff-443c-a75c-2fe8c4bcb635', 'Owner',
        ra_roleGuid =~ 'b24988ac-6180-42a0-ab88-20f7382dd24c', 'Contributor',
        ra_roleGuid =~ 'acdd72a7-3385-48ef-bd42-f606fba81ae7', 'Reader',
        ra_roleGuid =~ '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9', 'User Access Administrator',
        ra_roleGuid =~ 'ba92f5b4-2d11-453d-a403-e96b0029c9fe', 'Storage Blob Data Contributor',
        ra_roleGuid =~ '00482a5a-887f-4fb3-b363-3b7fe8e74483', 'Key Vault Administrator',
        strcat('Other (', ra_roleGuid, ')')
    )
    | project ra_principalId, ra_roleName, ra_scope, ra_subscription, ra_roleGuid
) on $left.mi_principalId == $right.ra_principalId
| where mi_subscription != ra_subscription
| project
    ['Resource Name'] = mi_name,
    ['Resource Type'] = mi_resourceType,
    ['Home Subscription'] = mi_subscription,
    ['Home Resource Group'] = mi_resourceGroup,
    ['Role Assigned'] = ra_roleName,
    ['Target Subscription'] = ra_subscription,
    ['Target Scope'] = ra_scope,
    ['Cross-Sub'] = 'Yes',
    ['Resource ID'] = mi_id
| order by ['Resource Type'] asc, ['Resource Name'] asc
```

---

## 6. High-Privilege Managed Identity Assignments

Identifies managed identities (service principals) with dangerous high-privilege roles: **Owner**, **Contributor**, or **User Access Administrator**. These should be reviewed as part of least-privilege audits.

```kql
authorizationresources
| where type =~ 'microsoft.authorization/roleassignments'
| extend principalType = tostring(properties.principalType)
| where principalType =~ 'ServicePrincipal'
| extend principalId = tostring(properties.principalId)
| extend roleDefinitionId = tostring(properties.roleDefinitionId)
| extend roleGuid = tostring(split(roleDefinitionId, '/')[-1])
| where roleGuid in~ (
    '8e3af657-a8ff-443c-a75c-2fe8c4bcb635',
    'b24988ac-6180-42a0-ab88-20f7382dd24c',
    '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9'
)
| extend roleName = case(
    roleGuid =~ '8e3af657-a8ff-443c-a75c-2fe8c4bcb635', 'Owner',
    roleGuid =~ 'b24988ac-6180-42a0-ab88-20f7382dd24c', 'Contributor',
    roleGuid =~ '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9', 'User Access Administrator',
    'Unknown'
)
| extend scope = tostring(properties.scope)
| extend scopeLevel = case(
    scope matches regex '^/subscriptions/[^/]+$', 'Subscription',
    scope matches regex '^/subscriptions/[^/]+/resourceGroups/[^/]+$', 'Resource Group',
    scope matches regex '^/$', 'Root',
    scope matches regex '^/providers/Microsoft.Management/managementGroups/', 'Management Group',
    'Resource'
)
| extend riskLevel = case(
    roleGuid in~ ('8e3af657-a8ff-443c-a75c-2fe8c4bcb635', '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9') 
        and scope matches regex '^(/|/providers/Microsoft.Management/managementGroups/|/subscriptions/[^/]+)$', 'Critical',
    roleGuid in~ ('8e3af657-a8ff-443c-a75c-2fe8c4bcb635', '18d7d88d-d35e-4fb5-a5c3-7773c20a72d9'), 'High',
    roleGuid =~ 'b24988ac-6180-42a0-ab88-20f7382dd24c'
        and scope matches regex '^/subscriptions/[^/]+$', 'High',
    'Medium'
)
| project
    ['Principal ID'] = principalId,
    ['Risk Level'] = riskLevel,
    ['Role'] = roleName,
    ['Scope Level'] = scopeLevel,
    ['Scope'] = scope,
    ['Subscription'] = subscriptionId,
    ['Assignment ID'] = id
| order by ['Risk Level'] asc, ['Role'] asc
```

---

# Part 2 — Resource-Level Firewall & Network Access Visualization

## 7. Unified Resource Network Access Overview

Single comprehensive query that shows **every PaaS resource** with its complete network access posture: access classification, firewall settings, IP/VNet whitelisting details, private endpoint count, and trusted services bypass — all in one unified Resource Graph query.

> **Access Categories (sorted by risk):**
>
> | Category | Meaning |
> |----------|---------|
> | **Secured by Perimeter** | Azure Network Security Perimeter — highest protection |
> | **Fully Private** | Public access disabled + private endpoint(s) present, or firewall deny-all with PE |
> | **Private (No PE)** | Public access disabled or firewall deny-all, but no private endpoints configured |
> | **Semi-Public (Whitelisted)** | Firewall default action = Deny with IP and/or VNet rules — reachable from whitelisted sources only |
> | **Fully Public** | No firewall restrictions, public network access enabled or not configured |
> | **Unknown** | Cannot determine status from available Resource Graph properties |

```kql
resources
| where type in~ (
    'microsoft.storage/storageaccounts',
    'microsoft.keyvault/vaults',
    'microsoft.sql/servers',
    'microsoft.dbforpostgresql/flexibleservers',
    'microsoft.dbformysql/flexibleservers',
    'microsoft.documentdb/databaseaccounts',
    'microsoft.containerregistry/registries',
    'microsoft.cache/redis',
    'microsoft.eventhub/namespaces',
    'microsoft.servicebus/namespaces',
    'microsoft.web/sites',
    'microsoft.cognitiveservices/accounts',
    'microsoft.apimanagement/service',
    'microsoft.search/searchservices'
)
//
// --- Friendly type name ---
| extend friendlyType = case(
    type =~ 'microsoft.storage/storageaccounts', 'Storage Account',
    type =~ 'microsoft.keyvault/vaults', 'Key Vault',
    type =~ 'microsoft.sql/servers', 'SQL Server',
    type =~ 'microsoft.dbforpostgresql/flexibleservers', 'PostgreSQL Flex',
    type =~ 'microsoft.dbformysql/flexibleservers', 'MySQL Flex',
    type =~ 'microsoft.documentdb/databaseaccounts', 'Cosmos DB',
    type =~ 'microsoft.containerregistry/registries', 'Container Registry',
    type =~ 'microsoft.cache/redis', 'Redis Cache',
    type =~ 'microsoft.eventhub/namespaces', 'Event Hub',
    type =~ 'microsoft.servicebus/namespaces', 'Service Bus',
    type =~ 'microsoft.web/sites', 'App Service',
    type =~ 'microsoft.cognitiveservices/accounts', 'Cognitive Services',
    type =~ 'microsoft.apimanagement/service', 'API Management',
    type =~ 'microsoft.search/searchservices', 'Azure AI Search',
    type
)
//
// --- Public network access (top-level property, most services) ---
| extend publicNetworkAccess = coalesce(tostring(properties.publicNetworkAccess), '')
//
// --- Firewall default action (varies per service) ---
| extend defaultAction = case(
    type =~ 'microsoft.storage/storageaccounts', tostring(properties.networkRuleSet.defaultAction),
    type =~ 'microsoft.keyvault/vaults', tostring(properties.networkAcls.defaultAction),
    type =~ 'microsoft.containerregistry/registries', tostring(properties.networkRuleSet.defaultAction),
    type =~ 'microsoft.cognitiveservices/accounts', tostring(properties.networkAcls.defaultAction),
    type =~ 'microsoft.search/searchservices', tostring(properties.networkRuleSet.defaultAction),
    ''
)
//
// --- IP whitelist rule count ---
| extend ipRuleCount = case(
    type =~ 'microsoft.storage/storageaccounts', iif(isnotempty(properties.networkRuleSet.ipRules), array_length(properties.networkRuleSet.ipRules), 0),
    type =~ 'microsoft.keyvault/vaults', iif(isnotempty(properties.networkAcls.ipRules), array_length(properties.networkAcls.ipRules), 0),
    type =~ 'microsoft.documentdb/databaseaccounts', iif(isnotempty(properties.ipRules), array_length(properties.ipRules), 0),
    type =~ 'microsoft.containerregistry/registries', iif(isnotempty(properties.networkRuleSet.ipRules), array_length(properties.networkRuleSet.ipRules), 0),
    type =~ 'microsoft.cognitiveservices/accounts', iif(isnotempty(properties.networkAcls.ipRules), array_length(properties.networkAcls.ipRules), 0),
    type =~ 'microsoft.search/searchservices', iif(isnotempty(properties.networkRuleSet.ipRules), array_length(properties.networkRuleSet.ipRules), 0),
    0
)
//
// --- VNet service endpoint rule count ---
| extend vnetRuleCount = case(
    type =~ 'microsoft.storage/storageaccounts', iif(isnotempty(properties.networkRuleSet.virtualNetworkRules), array_length(properties.networkRuleSet.virtualNetworkRules), 0),
    type =~ 'microsoft.keyvault/vaults', iif(isnotempty(properties.networkAcls.virtualNetworkRules), array_length(properties.networkAcls.virtualNetworkRules), 0),
    type =~ 'microsoft.documentdb/databaseaccounts', iif(isnotempty(properties.virtualNetworkRules), array_length(properties.virtualNetworkRules), 0),
    type =~ 'microsoft.containerregistry/registries', iif(isnotempty(properties.networkRuleSet.virtualNetworkRules), array_length(properties.networkRuleSet.virtualNetworkRules), 0),
    0
)
//
// --- Private endpoint connections ---
| extend privateEndpointConnections = properties.privateEndpointConnections
| extend hasPrivateEndpoint = isnotempty(privateEndpointConnections) and array_length(privateEndpointConnections) > 0
| extend peCount = iif(hasPrivateEndpoint, array_length(privateEndpointConnections), 0)
//
// --- Trusted services bypass (Storage & Key Vault expose this) ---
| extend trustedServicesBypass = case(
    type =~ 'microsoft.storage/storageaccounts', tostring(properties.networkRuleSet.bypass),
    type =~ 'microsoft.keyvault/vaults', tostring(properties.networkAcls.bypass),
    ''
)
//
// --- Access category classification ---
| extend accessCategory = case(
    publicNetworkAccess =~ 'SecuredByPerimeter', 'Secured by Perimeter',
    publicNetworkAccess =~ 'Disabled' and hasPrivateEndpoint, 'Fully Private',
    defaultAction =~ 'Deny' and hasPrivateEndpoint and (publicNetworkAccess =~ 'Disabled' or isempty(publicNetworkAccess)), 'Fully Private',
    publicNetworkAccess =~ 'Disabled' and not(hasPrivateEndpoint), 'Private (No PE)',
    defaultAction =~ 'Deny' and not(hasPrivateEndpoint) and ipRuleCount == 0 and vnetRuleCount == 0, 'Private (No PE)',
    defaultAction =~ 'Deny' and (ipRuleCount > 0 or vnetRuleCount > 0), 'Semi-Public (Whitelisted)',
    publicNetworkAccess =~ 'Enabled' or defaultAction =~ 'Allow', 'Fully Public',
    isempty(publicNetworkAccess) and isempty(defaultAction) and not(hasPrivateEndpoint), 'Fully Public',
    isempty(publicNetworkAccess) and isempty(defaultAction) and hasPrivateEndpoint, 'Semi-Public (Whitelisted)',
    'Unknown'
)
//
// --- Risk sort order (numeric, for stable ordering) ---
| extend riskOrder = case(
    accessCategory == 'Fully Public', 1,
    accessCategory == 'Semi-Public (Whitelisted)', 2,
    accessCategory == 'Unknown', 3,
    accessCategory == 'Private (No PE)', 4,
    accessCategory == 'Fully Private', 5,
    accessCategory == 'Secured by Perimeter', 6,
    9
)
| project
    ['Resource Name'] = name,
    ['Type'] = friendlyType,
    ['Access Category'] = accessCategory,
    ['Public Network Access'] = case(
        isempty(publicNetworkAccess), 'Not Set',
        publicNetworkAccess
    ),
    ['Firewall Default Action'] = case(
        isempty(defaultAction), 'Not Set',
        defaultAction
    ),
    ['IP Rules'] = ipRuleCount,
    ['VNet Rules'] = vnetRuleCount,
    ['Private Endpoints'] = peCount,
    ['Trusted Services Bypass'] = case(
        isempty(trustedServicesBypass), '—',
        trustedServicesBypass
    ),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    ['Location'] = location,
    riskOrder,
    ['Resource ID'] = id
| order by riskOrder asc, ['Type'] asc, ['Resource Name'] asc
| project-away riskOrder
```

---

## 8. Firewall IP Whitelist Detail (Drill-Down)

Companion query to the unified overview above. Expands the actual **IP addresses / CIDR ranges** from every resource that has firewall IP rules configured (default action = Deny). One row per IP per resource — ideal for auditing exactly which public IPs can reach your semi-public resources.

> **Note:** Each IP rule is shown on its own row. A resource with 5 whitelisted IPs will appear 5 times. The VNet rules are also expanded separately in the second part of the query.

### 8a. IP Address Whitelist per Resource

```kql
resources
| where type in~ (
    'microsoft.storage/storageaccounts',
    'microsoft.keyvault/vaults',
    'microsoft.sql/servers',
    'microsoft.documentdb/databaseaccounts',
    'microsoft.containerregistry/registries',
    'microsoft.cognitiveservices/accounts',
    'microsoft.search/searchservices'
)
//
// --- Normalize the ipRules array into a single column ---
//     networkRuleSet.ipRules : Storage, ACR, AI Search
//     networkAcls.ipRules   : Key Vault, Cognitive Services
//     ipRules (top-level)   : Cosmos DB
| extend ipRulesArray = case(
    type in~ ('microsoft.storage/storageaccounts',
              'microsoft.containerregistry/registries',
              'microsoft.search/searchservices'),           properties.networkRuleSet.ipRules,
    type in~ ('microsoft.keyvault/vaults',
              'microsoft.cognitiveservices/accounts'),      properties.networkAcls.ipRules,
    type =~  'microsoft.documentdb/databaseaccounts',      properties.ipRules,
    type =~  'microsoft.sql/servers',                      properties.ipRules,
    dynamic([])
)
| where isnotempty(ipRulesArray) and array_length(ipRulesArray) > 0
//
// --- Firewall default action (Deny = firewall active) ---
| extend defaultAction = case(
    type in~ ('microsoft.storage/storageaccounts',
              'microsoft.containerregistry/registries',
              'microsoft.search/searchservices'),           tostring(properties.networkRuleSet.defaultAction),
    type in~ ('microsoft.keyvault/vaults',
              'microsoft.cognitiveservices/accounts'),      tostring(properties.networkAcls.defaultAction),
    ''
)
//
// --- Expand one row per IP rule ---
| mv-expand ipRule = ipRulesArray
//
// --- Extract IP address (different property names per service) ---
| extend ipAddress = coalesce(
    tostring(ipRule.value),
    tostring(ipRule.ipAddressOrRange)
)
| extend ipAction = tostring(ipRule.action)
| extend friendlyType = case(
    type =~ 'microsoft.storage/storageaccounts', 'Storage Account',
    type =~ 'microsoft.keyvault/vaults', 'Key Vault',
    type =~ 'microsoft.sql/servers', 'SQL Server',
    type =~ 'microsoft.documentdb/databaseaccounts', 'Cosmos DB',
    type =~ 'microsoft.containerregistry/registries', 'Container Registry',
    type =~ 'microsoft.cognitiveservices/accounts', 'Cognitive Services',
    type =~ 'microsoft.search/searchservices', 'Azure AI Search',
    type
)
| extend ipType = case(
    ipAddress contains '/', 'CIDR Range',
    ipAddress matches regex '^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+$', 'Single IP',
    'Other'
)
| project
    ['Resource Name'] = name,
    ['Type'] = friendlyType,
    ['Firewall Default'] = defaultAction,
    ['Whitelisted IP / CIDR'] = ipAddress,
    ['IP Type'] = ipType,
    ['Rule Action'] = case(isempty(ipAction), 'Allow', ipAction),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    ['Location'] = location,
    ['Resource ID'] = id
| order by ['Type'] asc, ['Resource Name'] asc, ['Whitelisted IP / CIDR'] asc
```

### 8b. VNet Service Endpoint Whitelist per Resource

```kql
resources
| where type in~ (
    'microsoft.storage/storageaccounts',
    'microsoft.keyvault/vaults',
    'microsoft.documentdb/databaseaccounts',
    'microsoft.containerregistry/registries'
)
//
// --- Normalize the virtualNetworkRules array ---
| extend vnetRulesArray = case(
    type =~ 'microsoft.storage/storageaccounts', properties.networkRuleSet.virtualNetworkRules,
    type =~ 'microsoft.keyvault/vaults', properties.networkAcls.virtualNetworkRules,
    type =~ 'microsoft.documentdb/databaseaccounts', properties.virtualNetworkRules,
    type =~ 'microsoft.containerregistry/registries', properties.networkRuleSet.virtualNetworkRules,
    dynamic([])
)
| where isnotempty(vnetRulesArray) and array_length(vnetRulesArray) > 0
//
// --- Expand one row per VNet rule ---
| mv-expand vnetRule = vnetRulesArray
| extend subnetId = coalesce(tostring(vnetRule.id), tostring(vnetRule.['virtual-network-resource-id']))
| extend vnetName = tostring(split(subnetId, '/')[-3])
| extend subnetName = tostring(split(subnetId, '/')[-1])
| extend vnetSubscription = tostring(split(subnetId, '/')[2])
| extend friendlyType = case(
    type =~ 'microsoft.storage/storageaccounts', 'Storage Account',
    type =~ 'microsoft.keyvault/vaults', 'Key Vault',
    type =~ 'microsoft.documentdb/databaseaccounts', 'Cosmos DB',
    type =~ 'microsoft.containerregistry/registries', 'Container Registry',
    type
)
| project
    ['Resource Name'] = name,
    ['Type'] = friendlyType,
    ['VNet Name'] = vnetName,
    ['Subnet'] = subnetName,
    ['VNet Subscription'] = vnetSubscription,
    ['Cross-Sub VNet'] = case(vnetSubscription != subscriptionId, 'Yes', 'No'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    ['Full Subnet ID'] = subnetId,
    ['Resource ID'] = id
| order by ['Type'] asc, ['Resource Name'] asc, ['VNet Name'] asc
```

All queries above can be scoped by adding filters after the initial `| where type` clause:

```kql
// Filter by specific resource groups
| where resourceGroup in ('rg-prod', 'rg-dev')

// Filter by specific subscriptions
| where subscriptionId in ('sub-id-1', 'sub-id-2')

// Filter by location
| where location in ('westeurope', 'northeurope')
```

For workbook integration, use the standard parameter pattern:
```kql
| where '*' in ({ResourceGroups}) or resourceGroup in ({ResourceGroups})
```

---

## Azure Resource Graph Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| `authorizationresources` join with `resources` | Maximum 3 joins per query | Split into separate queries for user-assigned and system-assigned |
| No `make_list()` / `make_set()` | Cannot aggregate IP rules into a single cell | Show counts instead; drill down per resource in portal |
| Role definition names | Not directly in role assignments | Use `case()` with known built-in role GUIDs |
| SQL Server firewall rules | Stored as separate resource type (`microsoft.sql/servers/firewallrules`) | Not reflected in `properties.networkRuleSet`; SQL Servers show `publicNetworkAccess` only |
| Some PaaS resources don't expose `networkRuleSet` | e.g. Redis, Event Hub, Service Bus lack IP rule details in ARG | `publicNetworkAccess` property is used; IP/VNet rule columns show 0 |
| Trusted services bypass | Only Storage and Key Vault expose `bypass` | Other services show `—` in the bypass column |

---

*Last Updated: March 2026*
