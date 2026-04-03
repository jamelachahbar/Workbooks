# Managed Identity & Firewall Dashboard - KQL Queries

Ready-to-use queries for [Azure Resource Graph Explorer](https://portal.azure.com/#blade/HubsExtension/ArgQueryBlade).

> **Note:** Workbook parameters (`{Subscriptions}`, `{ResourceGroups}`) have been removed. Filter by subscription scope directly in Resource Graph Explorer. Add `| where resourceGroup in ('rg1','rg2')` if you need resource group filtering.

---

## 1. User-Assigned Managed Identities

```kql
resources
| where type =~ 'microsoft.managedidentity/userassignedidentities'
| extend principalId = tostring(properties.principalId)
| extend clientId = tostring(properties.clientId)
| project
    ['Identity Name'] = name,
    ['Type'] = 'User-Assigned',
    ['Principal ID'] = principalId,
    ['Client ID'] = clientId,
    ['Subscription'] = subscriptionId,
    ['Resource Group'] = resourceGroup,
    ['Location'] = location,
    ['Resource ID'] = id
| order by ['Identity Name'] asc
```

---

## 2. System-Assigned Managed Identities

```kql
resources
| where isnotempty(identity)
| where tostring(identity.type) has 'SystemAssigned'
| extend principalId = tostring(identity.principalId)
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

## 3. High-Privilege Role Assignments (Service Principals)

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

## 4. Cross-Subscription Assignments (User-Assigned MIs)

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

## 5. Cross-Subscription Assignments (System-Assigned MIs)

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

## 6. Resource Network Access Posture (Firewall Overview)

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
| extend publicNetworkAccess = coalesce(tostring(properties.publicNetworkAccess), '')
| extend defaultAction = case(
    type =~ 'microsoft.storage/storageaccounts', tostring(properties.networkRuleSet.defaultAction),
    type =~ 'microsoft.keyvault/vaults', tostring(properties.networkAcls.defaultAction),
    type =~ 'microsoft.containerregistry/registries', tostring(properties.networkRuleSet.defaultAction),
    type =~ 'microsoft.cognitiveservices/accounts', tostring(properties.networkAcls.defaultAction),
    type =~ 'microsoft.search/searchservices', tostring(properties.networkRuleSet.defaultAction),
    ''
)
| extend ipRuleCount = case(
    type =~ 'microsoft.storage/storageaccounts', iif(isnotempty(properties.networkRuleSet.ipRules), array_length(properties.networkRuleSet.ipRules), 0),
    type =~ 'microsoft.keyvault/vaults', iif(isnotempty(properties.networkAcls.ipRules), array_length(properties.networkAcls.ipRules), 0),
    type =~ 'microsoft.documentdb/databaseaccounts', iif(isnotempty(properties.ipRules), array_length(properties.ipRules), 0),
    type =~ 'microsoft.containerregistry/registries', iif(isnotempty(properties.networkRuleSet.ipRules), array_length(properties.networkRuleSet.ipRules), 0),
    type =~ 'microsoft.cognitiveservices/accounts', iif(isnotempty(properties.networkAcls.ipRules), array_length(properties.networkAcls.ipRules), 0),
    type =~ 'microsoft.search/searchservices', iif(isnotempty(properties.networkRuleSet.ipRules), array_length(properties.networkRuleSet.ipRules), 0),
    0
)
| extend vnetRuleCount = case(
    type =~ 'microsoft.storage/storageaccounts', iif(isnotempty(properties.networkRuleSet.virtualNetworkRules), array_length(properties.networkRuleSet.virtualNetworkRules), 0),
    type =~ 'microsoft.keyvault/vaults', iif(isnotempty(properties.networkAcls.virtualNetworkRules), array_length(properties.networkAcls.virtualNetworkRules), 0),
    type =~ 'microsoft.documentdb/databaseaccounts', iif(isnotempty(properties.virtualNetworkRules), array_length(properties.virtualNetworkRules), 0),
    type =~ 'microsoft.containerregistry/registries', iif(isnotempty(properties.networkRuleSet.virtualNetworkRules), array_length(properties.networkRuleSet.virtualNetworkRules), 0),
    0
)
| extend privateEndpointConnections = properties.privateEndpointConnections
| extend hasPrivateEndpoint = isnotempty(privateEndpointConnections) and array_length(privateEndpointConnections) > 0
| extend peCount = iif(hasPrivateEndpoint, array_length(privateEndpointConnections), 0)
| extend trustedServicesBypass = case(
    type =~ 'microsoft.storage/storageaccounts', tostring(properties.networkRuleSet.bypass),
    type =~ 'microsoft.keyvault/vaults', tostring(properties.networkAcls.bypass),
    ''
)
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

## 7. Whitelisted IP Addresses per Resource

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
| extend defaultAction = case(
    type in~ ('microsoft.storage/storageaccounts',
              'microsoft.containerregistry/registries',
              'microsoft.search/searchservices'),           tostring(properties.networkRuleSet.defaultAction),
    type in~ ('microsoft.keyvault/vaults',
              'microsoft.cognitiveservices/accounts'),      tostring(properties.networkAcls.defaultAction),
    ''
)
| mv-expand ipRule = ipRulesArray
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

---

## 8. VNet Service Endpoint Rules per Resource

```kql
resources
| where type in~ (
    'microsoft.storage/storageaccounts',
    'microsoft.keyvault/vaults',
    'microsoft.documentdb/databaseaccounts',
    'microsoft.containerregistry/registries'
)
| extend vnetRulesArray = case(
    type =~ 'microsoft.storage/storageaccounts', properties.networkRuleSet.virtualNetworkRules,
    type =~ 'microsoft.keyvault/vaults', properties.networkAcls.virtualNetworkRules,
    type =~ 'microsoft.documentdb/databaseaccounts', properties.virtualNetworkRules,
    type =~ 'microsoft.containerregistry/registries', properties.networkRuleSet.virtualNetworkRules,
    dynamic([])
)
| where isnotempty(vnetRulesArray) and array_length(vnetRulesArray) > 0
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
