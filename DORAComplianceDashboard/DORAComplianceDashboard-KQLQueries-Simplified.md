# DORA 2022/2554 Compliance Dashboard - Simplified KQL Queries

**Version 1.0** | January 2026

This document contains **simplified versions** of the DORA Compliance Dashboard queries **without compliance scoring or thresholds**. These queries return raw resource data for custom analysis, data exports, or integration with other tools.

## 📘 About This Document

**Purpose**: Provide raw Azure resource data without DORA compliance assessments

**Use Cases**:
- Export data to Excel/Power BI for custom analysis
- Apply your own compliance thresholds and logic
- Integrate with third-party tools and dashboards
- Generate simple resource inventories
- Performance-optimized queries for large environments

**Companion Document**: For queries with compliance scoring and threshold-based assessments, see [DORAComplianceDashboard-KQLQueries.md](DORAComplianceDashboard-KQLQueries.md)

---

## ✅ Ready for Resource Graph Explorer

All queries work directly in Azure Resource Graph Explorer:

1. Open [Azure Portal](https://portal.azure.com) → **Resource Graph Explorer**
2. Copy any query from this document
3. Paste and click **Run query**
4. Export results as CSV/JSON if needed

---

## Table of Contents

1. [Executive Summary Queries](#executive-summary-queries)
2. [Article 9.3a - ICT Systems Availability](#article-93a---ict-systems-availability)
3. [Article 9.3b - Access Control](#article-93b---access-control)
4. [Article 9.3c - Data Protection](#article-93c---data-protection)
5. [Article 9.3d - Identity & Authentication](#article-93d---identity--authentication)
6. [Article 10.1 - Detection & Monitoring](#article-101---detection--monitoring)
7. [Article 10.2 - Incident Response & Logging](#article-102---incident-response--logging)
8. [Analytics Queries](#analytics-queries)

---

## Executive Summary Queries

### Zone Redundancy Coverage

```kql
resources
| where type in~ (
    'microsoft.compute/virtualmachines',
    'microsoft.compute/virtualmachinescalesets',
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.network/applicationgateways',
    'microsoft.network/azurefirewalls',
    'microsoft.storage/storageaccounts',
    'microsoft.sql/servers/databases',
    'microsoft.dbforpostgresql/flexibleservers',
    'microsoft.dbformysql/flexibleservers',
    'microsoft.cache/redis',
    'microsoft.containerservice/managedclusters',
    'microsoft.web/sites',
    'microsoft.app/containerapps',
    'microsoft.eventhub/namespaces',
    'microsoft.servicebus/namespaces',
    'microsoft.apimanagement/service',
    'microsoft.keyvault/vaults',
    'microsoft.containerregistry/registries',
    'microsoft.documentdb/databaseaccounts'
)
| extend hasZones = isnotempty(zones)
| summarize 
    Total = count(),
    ZoneCompliant = countif(hasZones),
    NonCompliant = countif(not(hasZones))
| extend ComplianceRate = round(todouble(ZoneCompliant) / todouble(Total) * 100, 2)
| project 
    ['Total Resources'] = Total,
    ['Zone Redundant'] = ZoneCompliant,
    ['Not Zone Redundant'] = NonCompliant,
    ['Percentage Zone Redundant'] = ComplianceRate
```

### PaaS Network Isolation

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
| extend publicNetworkAccess = case(
    type =~ 'microsoft.storage/storageaccounts', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.keyvault/vaults', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.sql/servers', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.containerregistry/registries', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.cognitiveservices/accounts', tostring(properties.publicNetworkAccess),
    'Unknown'
)
| extend privateEndpointConnections = properties.privateEndpointConnections
| extend hasPrivateEndpoint = isnotempty(privateEndpointConnections)
| extend isFullyIsolated = hasPrivateEndpoint and (publicNetworkAccess =~ 'Disabled' or publicNetworkAccess =~ 'SecuredByPerimeter')
| summarize 
    TotalPaaSResources = count(),
    FullyIsolated = countif(isFullyIsolated),
    HasPEWithPublic = countif(hasPrivateEndpoint and not(isFullyIsolated)),
    NoPE = countif(not(hasPrivateEndpoint))
| extend IsolationRate = round(todouble(FullyIsolated) / todouble(TotalPaaSResources) * 100, 2)
| project 
    ['Total PaaS Resources'] = TotalPaaSResources,
    ['Fully Isolated'] = FullyIsolated,
    ['Has PE + Public Access'] = HasPEWithPublic,
    ['No Private Endpoint'] = NoPE,
    ['Percentage Fully Isolated'] = IsolationRate
```

### Storage Accounts - Secure Transfer

```kql
resources
| where type =~ 'microsoft.storage/storageaccounts'
| extend httpsOnly = tobool(properties.supportsHttpsTrafficOnly)
| extend minTlsVersion = tostring(properties.minimumTlsVersion)
| extend tlsCompliant = (minTlsVersion == 'TLS1_2' or minTlsVersion == 'TLS1_3')
| summarize 
    Total = count(),
    HttpsCompliant = countif(httpsOnly == true),
    TlsCompliant = countif(tlsCompliant)
| extend HttpsRate = round(todouble(HttpsCompliant) / todouble(Total) * 100, 2)
| extend TlsRate = round(todouble(TlsCompliant) / todouble(Total) * 100, 2)
| project 
    ['Total Storage Accounts'] = Total,
    ['HTTPS Enforced'] = HttpsCompliant,
    ['TLS 1.2+ Enforced'] = TlsCompliant,
    ['Percentage HTTPS'] = HttpsRate,
    ['Percentage TLS 1.2+'] = TlsRate
```

### Key Vaults - RBAC Enabled

```kql
resources
| where type =~ 'microsoft.keyvault/vaults'
| extend enableRbac = tobool(properties.enableRbacAuthorization)
| extend softDelete = tobool(properties.enableSoftDelete)
| extend purgeProtection = tobool(properties.enablePurgeProtection)
| summarize 
    Total = count(),
    RbacEnabled = countif(enableRbac == true),
    SoftDeleteEnabled = countif(softDelete == true),
    PurgeProtected = countif(purgeProtection == true)
| extend RbacRate = round(todouble(RbacEnabled) / todouble(Total) * 100, 2)
| project 
    ['Total Key Vaults'] = Total,
    ['RBAC Enabled'] = RbacEnabled,
    ['Soft Delete Enabled'] = SoftDeleteEnabled,
    ['Purge Protected'] = PurgeProtected,
    ['Percentage RBAC'] = RbacRate
```

### Databases - AAD-Only Authentication

```kql
resources
| where type in~ (
    'microsoft.sql/servers',
    'microsoft.dbformysql/flexibleservers'
)
| extend aadOnlyAuth = case(
    type =~ 'microsoft.sql/servers', tobool(properties.administrators.azureADOnlyAuthentication),
    type =~ 'microsoft.dbformysql/flexibleservers', tobool(properties.authConfig.activeDirectoryAuth) == true and tobool(properties.authConfig.passwordAuth) == false,
    false
)
| summarize 
    Total = count(),
    AadOnlyCompliant = countif(aadOnlyAuth == true)
| extend AadRate = round(todouble(AadOnlyCompliant) / todouble(Total) * 100, 2)
| project 
    ['Total Database Servers'] = Total,
    ['AAD-Only Auth Enabled'] = AadOnlyCompliant,
    ['SQL Auth Allowed'] = Total - AadOnlyCompliant,
    ['Percentage AAD-Only'] = AadRate
```

### Microsoft Defender Coverage

```kql
securityresources
| where type =~ 'microsoft.security/pricings'
| extend pricingTier = tostring(properties.pricingTier)
| extend planName = name
| summarize 
    TotalPlans = dcount(planName),
    EnabledPlans = dcountif(planName, pricingTier == 'Standard')
| extend CoverageRate = round(todouble(EnabledPlans) / todouble(TotalPlans) * 100, 0)
| project 
    ['Total Defender Plans'] = TotalPlans,
    ['Plans Enabled'] = EnabledPlans,
    ['Plans Not Enabled'] = TotalPlans - EnabledPlans,
    ['Percentage Enabled'] = CoverageRate
```

### NSG Flow Logs

```kql
resources
| where type =~ 'microsoft.network/networksecuritygroups'
| extend hasFlowLogs = isnotnull(properties.flowLogs) and array_length(properties.flowLogs) > 0
| summarize 
    Total = count(),
    WithFlowLogs = countif(hasFlowLogs == true)
| extend FlowLogsRate = round(todouble(WithFlowLogs) / todouble(Total) * 100, 2)
| project 
    ['Total NSGs'] = Total,
    ['With Flow Logs'] = WithFlowLogs,
    ['Without Flow Logs'] = Total - WithFlowLogs,
    ['Percentage With Logs'] = FlowLogsRate
```

### Business Continuity Risk

```kql
resources
| where type in~ (
    'microsoft.compute/virtualmachines',
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.storage/storageaccounts',
    'microsoft.sql/servers/databases',
    'microsoft.containerservice/managedclusters'
)
| extend hasZones = isnotempty(zones)
| summarize 
    TotalResources = count(),
    NonCompliant = countif(not(hasZones)),
    Compliant = countif(hasZones)
| extend RiskPercentage = round(todouble(NonCompliant) / todouble(TotalResources) * 100, 2)
| project 
    ['Total Critical Resources'] = TotalResources,
    ['Zone Redundant'] = Compliant,
    ['Not Zone Redundant'] = NonCompliant,
    ['Percentage At Risk'] = RiskPercentage
```

---

## Article 9.3a - ICT Systems Availability

### Virtual Machines - Zone Redundancy

```kql
resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend hasZones = isnotempty(zones)
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend osType = tostring(properties.storageProfile.osDisk.osType)
| extend powerState = tostring(properties.extended.instanceView.powerState.code)
| project 
    ['Virtual Machine'] = name,
    ['Zone Redundant'] = hasZones,
    ['Zones'] = zones,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['VM Size'] = vmSize,
    ['OS Type'] = osType,
    ['Power State'] = powerState,
    ['Resource ID'] = id
| order by ['Zone Redundant'] asc, ['Virtual Machine'] asc
```

### Network Infrastructure - Zone Redundancy

```kql
resources
| where type in~ (
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.network/applicationgateways',
    'microsoft.network/azurefirewalls'
)
| extend hasZones = isnotempty(zones)
| extend 
    resourceType = case(
        type =~ 'microsoft.network/loadbalancers', 'Load Balancer',
        type =~ 'microsoft.network/publicipaddresses', 'Public IP',
        type =~ 'microsoft.network/applicationgateways', 'Application Gateway',
        type =~ 'microsoft.network/azurefirewalls', 'Azure Firewall',
        type
    )
| extend skuTier = tostring(sku.tier)
| project 
    ['Resource Name'] = name,
    ['Resource Type'] = resourceType,
    ['Zone Redundant'] = hasZones,
    ['Zones'] = zones,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['SKU/Tier'] = skuTier,
    ['Resource ID'] = id
| order by ['Zone Redundant'] asc, ['Resource Type'] asc, ['Resource Name'] asc
```

### Redis Cache - SSL/TLS Configuration

```kql
resources
| where type =~ 'microsoft.cache/redis'
| extend sslEnabled = tobool(properties.enableNonSslPort) != true
| extend minTlsVersion = tostring(properties.minimumTlsVersion)
| project 
    ['Redis Cache'] = name,
    ['SSL Only'] = sslEnabled,
    ['Min TLS Version'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['SSL Only'] desc, ['Min TLS Version'] desc
```

### Databases - SSL Enforcement

```kql
resources
| where type in~ (
    'microsoft.dbforpostgresql/flexibleservers',
    'microsoft.dbformysql/flexibleservers',
    'microsoft.dbformariadb/servers'
)
| extend sslEnforcement = case(
    type =~ 'microsoft.dbforpostgresql/flexibleservers', tostring(properties.sslEnforcement),
    type =~ 'microsoft.dbformysql/flexibleservers', tostring(properties.sslEnforcement),
    type =~ 'microsoft.dbformariadb/servers', tostring(properties.sslEnforcement),
    'Unknown'
)
| extend minTlsVersion = tostring(properties.minimalTlsVersion)
| extend dbType = case(
    type =~ 'microsoft.dbforpostgresql/flexibleservers', 'PostgreSQL',
    type =~ 'microsoft.dbformysql/flexibleservers', 'MySQL',
    type =~ 'microsoft.dbformariadb/servers', 'MariaDB',
    type
)
| project 
    ['Database'] = name,
    ['Type'] = dbType,
    ['SSL Enforcement'] = sslEnforcement,
    ['Min TLS Version'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Type'] asc, ['Database'] asc
```

### Recovery Services Vaults - Backup Configuration

```kql
resources
| where type =~ 'microsoft.recoveryservices/vaults'
| extend vaultType = tostring(properties.type)
| extend redundancySettings = properties.redundancySettings
| extend backupStorageType = tostring(redundancySettings.standardTierStorageRedundancy)
| extend crossRegionRestore = tobool(redundancySettings.crossRegionRestore)
| extend softDeleteState = tostring(properties.securitySettings.softDeleteSettings.softDeleteState)
| extend immutableState = tostring(properties.securitySettings.immutabilitySettings.immutabilityState)
| extend privateEndpointConnections = properties.privateEndpointConnections
| extend hasPrivateEndpoint = isnotnull(privateEndpointConnections) and array_length(privateEndpointConnections) > 0
| extend softDeleteEnabled = softDeleteState =~ 'Enabled'
| extend immutableVault = immutableState =~ 'Enabled'
| project 
    ['Recovery Vault'] = name,
    ['Backup Storage Type'] = backupStorageType,
    ['Cross-Region Restore'] = crossRegionRestore,
    ['Soft Delete'] = softDeleteEnabled,
    ['Immutable Vault'] = immutableVault,
    ['Private Endpoint'] = hasPrivateEndpoint,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Recovery Vault'] asc
```

### API Management - Protocol Configuration

```kql
resources
| where type =~ 'microsoft.apimanagement/service'
| extend skuName = tostring(sku.name)
| extend protocols = properties.customProperties
| extend tls10Disabled = protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10'] == 'False'
| extend tls11Disabled = protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11'] == 'False'
| extend ssl30Disabled = protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Ssl30'] == 'False'
| extend hasPrivateEndpoint = isnotnull(properties.privateEndpointConnections) and array_length(properties.privateEndpointConnections) > 0
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| project 
    ['APIM Service'] = name,
    ['SKU'] = skuName,
    ['TLS 1.0 Disabled'] = tls10Disabled,
    ['TLS 1.1 Disabled'] = tls11Disabled,
    ['SSL 3.0 Disabled'] = ssl30Disabled,
    ['Private Endpoint'] = hasPrivateEndpoint,
    ['Public Access'] = publicNetworkAccess,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['APIM Service'] asc
```

### Private Endpoints - Detailed Inventory

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend privateEndpointName = name
| extend subnet = tostring(properties.subnet.id)
| extend subnetName = tostring(split(subnet, '/')[-1])
| extend vnetName = tostring(split(subnet, '/')[-3])
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend connectionState = tostring(privateLinkServiceConnection.properties.privateLinkServiceConnectionState.status)
| extend targetResourceId = tostring(privateLinkServiceConnection.properties.privateLinkServiceId)
| extend targetResourceName = tostring(split(targetResourceId, '/')[-1])
| extend targetResourceType = tostring(split(targetResourceId, '/')[-2])
| extend groupIds = tostring(privateLinkServiceConnection.properties.groupIds[0])
| project 
    ['Private Endpoint'] = privateEndpointName,
    ['Connection State'] = connectionState,
    ['Target Resource'] = targetResourceName,
    ['Target Type'] = targetResourceType,
    ['Sub Resource'] = groupIds,
    ['Virtual Network'] = vnetName,
    ['Subnet'] = subnetName,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Target Resource ID'] = targetResourceId,
    ['Resource ID'] = id
| order by ['Connection State'] asc, ['Private Endpoint'] asc
```

### PaaS Resources - Private Endpoint Coverage

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
| extend resourceType = case(
    type =~ 'microsoft.storage/storageaccounts', 'Storage Account',
    type =~ 'microsoft.keyvault/vaults', 'Key Vault',
    type =~ 'microsoft.sql/servers', 'SQL Server',
    type =~ 'microsoft.dbforpostgresql/flexibleservers', 'PostgreSQL',
    type =~ 'microsoft.dbformysql/flexibleservers', 'MySQL',
    type =~ 'microsoft.documentdb/databaseaccounts', 'Cosmos DB',
    type =~ 'microsoft.containerregistry/registries', 'Container Registry',
    type =~ 'microsoft.cache/redis', 'Redis Cache',
    type =~ 'microsoft.eventhub/namespaces', 'Event Hub',
    type =~ 'microsoft.servicebus/namespaces', 'Service Bus',
    type =~ 'microsoft.web/sites', 'App Service',
    type =~ 'microsoft.cognitiveservices/accounts', 'Cognitive Services',
    type =~ 'microsoft.apimanagement/service', 'API Management',
    type =~ 'microsoft.search/searchservices', 'Search Service',
    type
)
| extend publicNetworkAccess = case(
    type =~ 'microsoft.storage/storageaccounts', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.keyvault/vaults', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.sql/servers', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.containerregistry/registries', tostring(properties.publicNetworkAccess),
    type =~ 'microsoft.cognitiveservices/accounts', tostring(properties.publicNetworkAccess),
    'Unknown'
)
| extend privateEndpointConnections = properties.privateEndpointConnections
| extend hasPrivateEndpoint = isnotempty(privateEndpointConnections) and array_length(privateEndpointConnections) > 0
| extend peCount = iff(hasPrivateEndpoint, array_length(privateEndpointConnections), 0)
| project 
    ['Resource Name'] = name,
    ['Resource Type'] = resourceType,
    ['Private Endpoints'] = peCount,
    ['Has Private Endpoint'] = hasPrivateEndpoint,
    ['Public Network Access'] = publicNetworkAccess,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Resource Type'] asc, ['Has Private Endpoint'] desc, ['Resource Name'] asc
```

---

## Article 9.3b - Access Control

### Key Vaults - RBAC & Protection

```kql
resources
| where type =~ 'microsoft.keyvault/vaults'
| extend enableRbac = tobool(properties.enableRbacAuthorization)
| extend softDelete = tobool(properties.enableSoftDelete)
| extend purgeProtection = tobool(properties.enablePurgeProtection)
| extend networkAcls = properties.networkAcls
| extend defaultAction = tostring(networkAcls.defaultAction)
| project 
    ['Key Vault'] = name,
    ['RBAC Enabled'] = enableRbac,
    ['Soft Delete'] = softDelete,
    ['Purge Protection'] = purgeProtection,
    ['Network Default'] = defaultAction,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['RBAC Enabled'] desc, ['Key Vault'] asc
```

### Storage Accounts - Encryption & Redundancy

```kql
resources
| where type =~ 'microsoft.storage/storageaccounts'
| extend skuName = tostring(sku.name)
| extend isZoneRedundant = (skuName contains 'ZRS' or skuName contains 'GZRS')
| extend isGeoRedundant = (skuName contains 'GRS' or skuName contains 'GZRS' or skuName contains 'RA-GRS' or skuName contains 'RA-GZRS')
| extend httpsOnly = tobool(properties.supportsHttpsTrafficOnly)
| extend minTlsVersion = tostring(properties.minimumTlsVersion)
| extend encryption = properties.encryption
| extend cmkEnabled = isnotnull(encryption.keyvaultproperties)
| extend storageKind = tostring(kind)
| extend accessTier = tostring(properties.accessTier)
| project 
    ['Storage Account'] = name,
    ['Replication'] = skuName,
    ['Zone Redundant'] = isZoneRedundant,
    ['Geo Redundant'] = isGeoRedundant,
    ['HTTPS Only'] = httpsOnly,
    ['Min TLS'] = minTlsVersion,
    ['CMK Enabled'] = cmkEnabled,
    ['Kind'] = storageKind,
    ['Access Tier'] = accessTier,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Storage Account'] asc
```

### Container Registries - Security Configuration

```kql
resources
| where type =~ 'microsoft.containerregistry/registries'
| extend skuName = tostring(sku.name)
| extend adminEnabled = tobool(properties.adminUserEnabled)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend encryption = properties.encryption
| extend cmkEnabled = tostring(encryption.status) == 'enabled'
| extend zoneRedundancy = tostring(properties.zoneRedundancy)
| project 
    ['Container Registry'] = name,
    ['SKU'] = skuName,
    ['Admin User Enabled'] = adminEnabled,
    ['Public Access'] = publicNetworkAccess,
    ['CMK Enabled'] = cmkEnabled,
    ['Zone Redundant'] = zoneRedundancy,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Admin User Enabled'] asc, ['Container Registry'] asc
```

### AKS Clusters - RBAC Configuration

```kql
resources
| where type =~ 'microsoft.containerservice/managedclusters'
| extend enableRbac = tobool(properties.enableRBAC)
| extend aadEnabled = isnotnull(properties.aadProfile)
| extend azureRbacEnabled = tobool(properties.aadProfile.enableAzureRBAC)
| extend localAccounts = tobool(properties.disableLocalAccounts) != true
| extend networkPolicy = tostring(properties.networkProfile.networkPolicy)
| extend tier = tostring(sku.tier)
| extend hasZones = isnotempty(zones) or isnotempty(properties.agentPoolProfiles[0].availabilityZones)
| project 
    ['AKS Cluster'] = name,
    ['K8s RBAC Enabled'] = enableRbac,
    ['AAD Integrated'] = aadEnabled,
    ['Azure RBAC Enabled'] = azureRbacEnabled,
    ['Local Accounts Enabled'] = localAccounts,
    ['Network Policy'] = networkPolicy,
    ['Tier'] = tier,
    ['Zone Redundant'] = hasZones,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['K8s RBAC Enabled'] desc, ['AAD Integrated'] desc
```

---

## Article 9.3c - Data Protection

### Databases - High Availability & Zones

```kql
resources
| where type in~ (
    'microsoft.sql/servers/databases',
    'microsoft.dbforpostgresql/flexibleservers',
    'microsoft.dbformysql/flexibleservers',
    'microsoft.cache/redis',
    'microsoft.documentdb/databaseaccounts'
)
| extend hasZones = isnotempty(zones) or tobool(properties.zoneRedundant) or tobool(properties.highAvailability.mode == 'ZoneRedundant')
| extend 
    databaseType = case(
        type =~ 'microsoft.sql/servers/databases', 'SQL Database',
        type =~ 'microsoft.dbforpostgresql/flexibleservers', 'PostgreSQL',
        type =~ 'microsoft.dbformysql/flexibleservers', 'MySQL',
        type =~ 'microsoft.cache/redis', 'Redis Cache',
        type =~ 'microsoft.documentdb/databaseaccounts', 'Cosmos DB',
        type
    )
| extend haMode = case(
    type =~ 'microsoft.dbforpostgresql/flexibleservers', tostring(properties.highAvailability.mode),
    type =~ 'microsoft.dbformysql/flexibleservers', tostring(properties.highAvailability.mode),
    type =~ 'microsoft.documentdb/databaseaccounts', case(tobool(properties.enableMultipleWriteLocations), 'Multi-Region', 'Single-Region'),
    'N/A'
)
| project 
    ['Database'] = name,
    ['Type'] = databaseType,
    ['Zone Redundant'] = hasZones,
    ['HA Mode'] = haMode,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Zone Redundant'] desc, ['Type'] asc
```

### Container Services - Zone Redundancy

```kql
resources
| where type in~ (
    'microsoft.containerservice/managedclusters',
    'microsoft.app/containerapps'
)
| extend hasZones = isnotempty(zones)
| extend 
    serviceType = case(
        type =~ 'microsoft.containerservice/managedclusters', 'AKS Cluster',
        type =~ 'microsoft.app/containerapps', 'Container App',
        type
    )
| extend 
    tier = case(
        type =~ 'microsoft.containerservice/managedclusters', tostring(sku.tier),
        type =~ 'microsoft.app/containerapps', tostring(properties.managedEnvironmentId),
        'N/A'
    )
| extend networkProfile = case(
    type =~ 'microsoft.containerservice/managedclusters', tostring(properties.networkProfile.networkPlugin),
    'N/A'
)
| project 
    ['Service Name'] = name,
    ['Type'] = serviceType,
    ['Zone Redundant'] = hasZones,
    ['Tier'] = tier,
    ['Network Plugin'] = networkProfile,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Zone Redundant'] desc, ['Service Name'] asc
```

### App Services - HTTPS/FTPS/TLS

```kql
resources
| where type =~ 'microsoft.web/sites'
| extend hasZones = isnotempty(zones)
| extend httpsOnly = tobool(properties.httpsOnly)
| extend minTlsVersion = coalesce(tostring(properties.siteConfig.minTlsVersion), 'Not Set')
| extend ftpsState = coalesce(tostring(properties.siteConfig.ftpsState), 'Not Set')
| extend appKind = coalesce(tostring(kind), 'app')
| project 
    ['App Service'] = name,
    ['Kind'] = appKind,
    ['HTTPS Only'] = httpsOnly,
    ['FTPS State'] = ftpsState,
    ['Min TLS'] = minTlsVersion,
    ['Zone Redundant'] = hasZones,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['HTTPS Only'] desc, ['App Service'] asc
```

### Key Vaults - Protection Settings

```kql
resources
| where type =~ 'microsoft.keyvault/vaults'
| extend enableRbac = tobool(properties.enableRbacAuthorization)
| extend softDelete = tobool(properties.enableSoftDelete)
| extend purgeProtection = tobool(properties.enablePurgeProtection)
| extend softDeleteRetention = toint(properties.softDeleteRetentionInDays)
| extend enabledForDeployment = tobool(properties.enabledForDeployment)
| extend enabledForDiskEncryption = tobool(properties.enabledForDiskEncryption)
| extend enabledForTemplateDeployment = tobool(properties.enabledForTemplateDeployment)
| project 
    ['Key Vault'] = name,
    ['Soft Delete'] = softDelete,
    ['Purge Protection'] = purgeProtection,
    ['Retention Days'] = softDeleteRetention,
    ['RBAC Enabled'] = enableRbac,
    ['For Deployment'] = enabledForDeployment,
    ['For Disk Encryption'] = enabledForDiskEncryption,
    ['For Template Deployment'] = enabledForTemplateDeployment,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Soft Delete'] desc, ['Purge Protection'] desc
```

### Integration Services - Zone Redundancy

```kql
resources
| where type in~ (
    'microsoft.apimanagement/service',
    'microsoft.eventhub/namespaces',
    'microsoft.servicebus/namespaces'
)
| extend hasZones = isnotempty(zones)
| extend 
    serviceType = case(
        type =~ 'microsoft.apimanagement/service', 'API Management',
        type =~ 'microsoft.eventhub/namespaces', 'Event Hubs',
        type =~ 'microsoft.servicebus/namespaces', 'Service Bus',
        type
    )
| extend tier = tostring(sku.name)
| project 
    ['Service Name'] = name,
    ['Type'] = serviceType,
    ['Zone Redundant'] = hasZones,
    ['SKU'] = tier,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Zone Redundant'] desc, ['Type'] asc
```

### Analytics & AI Services - Private Link

```kql
resources
| where type in~ (
    'microsoft.databricks/workspaces',
    'microsoft.machinelearningservices/workspaces',
    'microsoft.synapse/workspaces',
    'microsoft.search/searchservices',
    'microsoft.cognitiveservices/accounts',
    'microsoft.openai/accounts'
)
| extend serviceType = case(
    type =~ 'microsoft.databricks/workspaces', 'Azure Databricks',
    type =~ 'microsoft.machinelearningservices/workspaces', 'Azure ML',
    type =~ 'microsoft.synapse/workspaces', 'Azure Synapse',
    type =~ 'microsoft.search/searchservices', 'Azure AI Search',
    type =~ 'microsoft.cognitiveservices/accounts', 'Cognitive Services',
    type =~ 'microsoft.openai/accounts', 'Azure OpenAI',
    type
)
| extend hasPrivateEndpoint = isnotnull(properties.privateEndpointConnections) and array_length(properties.privateEndpointConnections) > 0
| extend publicNetworkAccess = coalesce(
    tostring(properties.publicNetworkAccess),
    tostring(properties.networkAcls.defaultAction),
    'Not Set'
)
| project 
    ['Service Name'] = name,
    ['Type'] = serviceType,
    ['Private Endpoint'] = hasPrivateEndpoint,
    ['Public Access'] = publicNetworkAccess,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['Type'] asc, ['Private Endpoint'] desc
```

### Azure Databricks - Network Configuration

```kql
resources
| where type =~ 'microsoft.databricks/workspaces'
| extend enableNoPublicIp = tobool(properties.parameters.enableNoPublicIp.value)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend hasPrivateEndpoint = isnotnull(properties.privateEndpointConnections) and array_length(properties.privateEndpointConnections) > 0
| extend pricingTier = tostring(sku.name)
| project 
    ['Databricks Workspace'] = name,
    ['No Public IP'] = enableNoPublicIp,
    ['Private Endpoint'] = hasPrivateEndpoint,
    ['Public Network Access'] = publicNetworkAccess,
    ['Pricing Tier'] = pricingTier,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['No Public IP'] desc, ['Private Endpoint'] desc
```

---

## Article 9.3d - Identity & Authentication

### SQL Servers - Azure AD Authentication

```kql
resources
| where type =~ 'microsoft.sql/servers'
| extend aadOnlyAuth = tobool(properties.administrators.azureADOnlyAuthentication)
| extend aadAdmin = tostring(properties.administrators.login)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend minTlsVersion = tostring(properties.minimalTlsVersion)
| project 
    ['SQL Server'] = name,
    ['AAD-Only Auth'] = aadOnlyAuth,
    ['AAD Admin'] = aadAdmin,
    ['Public Network'] = publicNetworkAccess,
    ['Min TLS'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['AAD-Only Auth'] desc, ['SQL Server'] asc
```

---

## Article 10.1 - Detection & Monitoring

### Microsoft Defender Plans

```kql
securityresources
| where type =~ 'microsoft.security/pricings'
| extend pricingTier = tostring(properties.pricingTier)
| extend resourceName = name
| extend defenderPlan = case(
    resourceName == 'VirtualMachines', 'Defender for Servers',
    resourceName == 'SqlServers', 'Defender for SQL',
    resourceName == 'AppServices', 'Defender for App Service',
    resourceName == 'StorageAccounts', 'Defender for Storage',
    resourceName == 'KeyVaults', 'Defender for Key Vault',
    resourceName == 'Containers', 'Defender for Containers',
    resourceName == 'Arm', 'Defender for Resource Manager',
    resourceName == 'Dns', 'Defender for DNS',
    resourceName == 'OpenSourceRelationalDatabases', 'Defender for Open Source DBs',
    resourceName == 'CosmosDbs', 'Defender for Cosmos DB',
    resourceName == 'CloudPosture', 'Defender CSPM',
    resourceName
)
| project 
    ['Defender Plan'] = defenderPlan,
    ['Resource Name'] = resourceName,
    ['Pricing Tier'] = pricingTier,
    ['Enabled'] = (pricingTier == 'Standard'),
    ['Subscription'] = subscriptionId
| order by ['Enabled'] desc, ['Defender Plan'] asc
```

---

## Article 10.2 - Incident Response & Logging

### Network Security Groups - Flow Logs

```kql
resources
| where type =~ 'microsoft.network/networksecuritygroups'
| extend nsgId = id
| extend nsgName = name
| extend subnetCount = array_length(properties.subnets)
| extend nicCount = array_length(properties.networkInterfaces)
| extend rulesCount = array_length(properties.securityRules)
| extend hasFlowLogs = isnotnull(properties.flowLogs) and array_length(properties.flowLogs) > 0
| project 
    ['NSG Name'] = nsgName,
    ['Flow Logs Enabled'] = hasFlowLogs,
    ['Security Rules'] = rulesCount,
    ['Attached Subnets'] = subnetCount,
    ['Attached NICs'] = nicCount,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = nsgId
| order by ['Flow Logs Enabled'] desc, ['Attached Subnets'] desc
```

### PostgreSQL - Server Status

```kql
resources
| where type =~ 'microsoft.dbforpostgresql/flexibleservers'
| extend version = tostring(properties.version)
| extend haMode = tostring(properties.highAvailability.mode)
| extend state = tostring(properties.state)
| project 
    ['PostgreSQL Server'] = name,
    ['Version'] = version,
    ['HA Mode'] = haMode,
    ['State'] = state,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Resource ID'] = id
| order by ['State'] desc, ['PostgreSQL Server'] asc
```

---

## Analytics Queries

### Zone Redundancy by Resource Type

```kql
resources
| where type in~ (
    'microsoft.compute/virtualmachines',
    'microsoft.compute/virtualmachinescalesets',
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.network/applicationgateways',
    'microsoft.network/azurefirewalls',
    'microsoft.storage/storageaccounts',
    'microsoft.sql/servers/databases',
    'microsoft.containerservice/managedclusters',
    'microsoft.web/sites',
    'microsoft.keyvault/vaults',
    'microsoft.containerregistry/registries',
    'microsoft.documentdb/databaseaccounts',
    'microsoft.dbforpostgresql/flexibleservers',
    'microsoft.dbformysql/flexibleservers',
    'microsoft.cache/redis',
    'microsoft.eventhub/namespaces',
    'microsoft.servicebus/namespaces',
    'microsoft.apimanagement/service'
)
| extend hasZones = isnotempty(zones)
| extend 
    resourceType = case(
        type =~ 'microsoft.compute/virtualmachines', 'Virtual Machines',
        type =~ 'microsoft.compute/virtualmachinescalesets', 'VM Scale Sets',
        type =~ 'microsoft.network/loadbalancers', 'Load Balancers',
        type =~ 'microsoft.network/publicipaddresses', 'Public IPs',
        type =~ 'microsoft.network/applicationgateways', 'App Gateways',
        type =~ 'microsoft.network/azurefirewalls', 'Azure Firewalls',
        type =~ 'microsoft.storage/storageaccounts', 'Storage Accounts',
        type =~ 'microsoft.sql/servers/databases', 'SQL Databases',
        type =~ 'microsoft.containerservice/managedclusters', 'AKS Clusters',
        type =~ 'microsoft.web/sites', 'App Services',
        type =~ 'microsoft.keyvault/vaults', 'Key Vaults',
        type =~ 'microsoft.containerregistry/registries', 'Container Registries',
        type =~ 'microsoft.documentdb/databaseaccounts', 'Cosmos DB',
        type =~ 'microsoft.dbforpostgresql/flexibleservers', 'PostgreSQL',
        type =~ 'microsoft.dbformysql/flexibleservers', 'MySQL',
        type =~ 'microsoft.cache/redis', 'Redis Cache',
        type =~ 'microsoft.eventhub/namespaces', 'Event Hubs',
        type =~ 'microsoft.servicebus/namespaces', 'Service Bus',
        type =~ 'microsoft.apimanagement/service', 'API Management',
        type
    )
| summarize 
    Total = count(),
    ZoneRedundant = countif(hasZones),
    NotZoneRedundant = countif(not(hasZones))
    by resourceType
| extend PercentageZoneRedundant = round(todouble(ZoneRedundant) / todouble(Total) * 100, 2)
| order by NotZoneRedundant desc
| project 
    ['Resource Type'] = resourceType,
    ['Total'] = Total,
    ['Zone Redundant'] = ZoneRedundant,
    ['Not Zone Redundant'] = NotZoneRedundant,
    ['% Zone Redundant'] = PercentageZoneRedundant
```

### Resource Distribution by Azure Region

```kql
resources
| where type in~ (
    'microsoft.compute/virtualmachines',
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.storage/storageaccounts',
    'microsoft.sql/servers/databases',
    'microsoft.containerservice/managedclusters',
    'microsoft.keyvault/vaults',
    'microsoft.containerregistry/registries'
)
| extend hasZones = isnotempty(zones)
| summarize 
    Total = count(),
    ZoneRedundant = countif(hasZones),
    NotZoneRedundant = countif(not(hasZones))
    by location
| extend PercentageZoneRedundant = round(todouble(ZoneRedundant) / todouble(Total) * 100, 2)
| order by NotZoneRedundant desc
| project 
    ['Azure Region'] = location,
    ['Total Resources'] = Total,
    ['Zone Redundant'] = ZoneRedundant,
    ['Not Zone Redundant'] = NotZoneRedundant,
    ['% Zone Redundant'] = PercentageZoneRedundant
```

### Resource Distribution by Subscription

```kql
resources
| where type in~ (
    'microsoft.compute/virtualmachines',
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.storage/storageaccounts',
    'microsoft.sql/servers/databases',
    'microsoft.containerservice/managedclusters',
    'microsoft.web/sites',
    'microsoft.keyvault/vaults',
    'microsoft.containerregistry/registries'
)
| extend hasZones = isnotempty(zones)
| summarize 
    TotalResources = count(),
    ZoneRedundant = countif(hasZones),
    NotZoneRedundant = countif(not(hasZones)),
    ResourceTypes = dcount(type)
    by subscriptionId
| extend PercentageZoneRedundant = round(todouble(ZoneRedundant) / todouble(TotalResources) * 100, 2)
| order by NotZoneRedundant desc
| project 
    ['Subscription ID'] = subscriptionId,
    ['Total Resources'] = TotalResources,
    ['Zone Redundant'] = ZoneRedundant,
    ['Not Zone Redundant'] = NotZoneRedundant,
    ['% Zone Redundant'] = PercentageZoneRedundant,
    ['Resource Types'] = ResourceTypes
```

### Private Endpoints by Tenant and Subscription

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend tenantId = tostring(tenantId)
| extend subscriptionId = tostring(subscriptionId)
| summarize 
    ['Private Endpoints'] = count(),
    ['Resource Groups'] = dcount(resourceGroup),
    ['Locations'] = dcount(location)
    by tenantId, subscriptionId
| order by ['Private Endpoints'] desc
| project 
    ['Tenant ID'] = tenantId,
    ['Subscription ID'] = subscriptionId,
    ['Private Endpoints'],
    ['Resource Groups'],
    ['Locations']
```

### Cross-Tenant Private Endpoints

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend peSubscriptionId = tostring(subscriptionId)
| extend peTenantId = tostring(tenantId)
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend targetResourceId = tostring(privateLinkServiceConnection.properties.privateLinkServiceId)
| extend targetResourceIdLower = tolower(targetResourceId)
| extend targetSubscriptionId = tostring(split(targetResourceId, '/')[2])
| extend connectionState = tostring(privateLinkServiceConnection.properties.privateLinkServiceConnectionState.status)
| extend isCrossSubscription = (peSubscriptionId != targetSubscriptionId and targetSubscriptionId != '')
| join kind=leftouter (
    resources
    | extend targetTenantId = tostring(tenantId)
    | extend idLower = tolower(id)
    | project idLower, targetTenantId
) on $left.targetResourceIdLower == $right.idLower
| extend finalTargetTenant = iff(isempty(targetTenantId), peTenantId, targetTenantId)
| extend isCrossTenant = (peTenantId != finalTargetTenant and isnotempty(finalTargetTenant))
| project 
    ['Private Endpoint'] = name,
    ['Cross-Tenant'] = isCrossTenant,
    ['Cross-Subscription'] = isCrossSubscription,
    ['Connection State'] = connectionState,
    ['PE Tenant'] = peTenantId,
    ['PE Subscription'] = peSubscriptionId,
    ['PE Resource Group'] = resourceGroup,
    ['Target Tenant'] = finalTargetTenant,
    ['Target Subscription'] = targetSubscriptionId,
    ['Target Resource'] = tostring(split(targetResourceId, '/')[-1]),
    Location = location
| order by ['Cross-Tenant'] desc, ['Cross-Subscription'] desc
```

---

## 📝 Additional Filtering

To filter any query to specific resources, add filters after the initial `where` clause:

```kql
// By Resource Group
| where resourceGroup in ('my-rg-1', 'my-rg-2')

// By Subscription
| where subscriptionId in ('sub-id-1', 'sub-id-2')

// By Location
| where location in ('eastus', 'westeurope')

// By Tag
| where tags['Environment'] == 'Production'
```

---

## 🔐 Permissions Required

- **Reader** role on subscriptions/resource groups
- **Security Reader** role for queries using `securityresources` table

---

## 📚 References

- [Main Query Document (with compliance scoring)](DORAComplianceDashboard-KQLQueries.md)
- [DORA Regulation (EU) 2022/2554](https://eur-lex.europa.eu/eli/reg/2022/2554/oj)
- [Azure Resource Graph Query Language](https://learn.microsoft.com/azure/governance/resource-graph/concepts/query-language)
