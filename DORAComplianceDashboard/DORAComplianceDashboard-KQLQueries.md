# DORA 2022/2554 Compliance Dashboard - KQL Queries

**Version 1.0** | January 2026

This document contains all KQL (Kusto Query Language) queries used in the DORA Compliance Dashboard workbook. **All queries are ready to run directly in Azure Resource Graph Explorer** without modification.

## ✅ Ready for Resource Graph Explorer

All queries in this document have been updated to work directly in Azure Resource Graph Explorer:

- **No parameters required** - Workbook parameters (`{ResourceGroups}`, `{Subscriptions}`) have been removed
- **Full subscription scope** - Queries scan all accessible subscriptions
- **Direct execution** - Copy, paste, and run immediately

## 🎯 Quick Start

1. Open [Azure Portal](https://portal.azure.com) → **Resource Graph Explorer**
2. Copy any query from this document
3. Paste into the query window
4. Click **Run query**
5. Export results as CSV/JSON if needed

## ⚙️ Threshold Configuration

**Important:** Many queries include compliance thresholds and rating scales (e.g., "90% = Excellent", "75% = Good"). These are **contextual examples only** and should be adjusted based on:

- Your organization's risk tolerance
- Industry-specific regulatory requirements
- Infrastructure size and complexity
- Internal governance policies
- DORA implementation interpretation

Look for **⚠️ Note** sections in queries to identify configurable thresholds.

## 🔍 Optional Filtering

To scope queries to specific resource groups or subscriptions, add filters after the initial `where` clause:

```kql
// Filter by specific resource groups
| where resourceGroup in ('rg-prod', 'rg-dev')

// Filter by specific subscriptions  
| where subscriptionId in ('sub-id-1', 'sub-id-2')

// Filter by location
| where location in ('eastus', 'westeurope')
```

---

## Workbook Structure

The DORA Compliance Dashboard is organized into **2 main tabs**:

1. **📖 Overview Tab** - Documentation hub with:
   - Introduction and getting started guide
   - Changelog & version history (expandable)
   - Features supported by section (expandable)
   - Usage instructions and troubleshooting (expandable)

2. **📊 Dashboard Tab** (Default) - Contains 8 compliance assessment tabs:
   - Executive Summary
   - Article 9.3a (Availability)
   - Article 9.3b (Access Control)
   - Article 9.3c (Data Protection)
   - Article 9.3d (Identity)
   - Article 10.1 (Monitoring)
   - Article 10.2 (Logging)
   - Analytics

All queries below correspond to the Dashboard tab assessments.

---

## Table of Contents

1. [Parameter Queries](#parameter-queries)
2. [Executive Summary Queries](#executive-summary-queries)
3. [Article 9.3a - ICT Systems Availability](#article-93a---ict-systems-availability)
4. [Article 9.3b - Access Control](#article-93b---access-control)
5. [Article 9.3c - Data Protection](#article-93c---data-protection)
6. [Article 9.3d - Identity & Authentication](#article-93d---identity--authentication)
7. [Article 10.1 - Detection & Monitoring](#article-101---detection--monitoring)
8. [Article 10.2 - Incident Response & Logging](#article-102---incident-response--logging)
9. [Analytics Queries](#analytics-queries)

---

## Parameter Queries

### Resource Groups List

```kql
resources
| summarize by resourceGroup
| order by resourceGroup asc
| project value = resourceGroup, label = resourceGroup, selected = false
```

---

## Executive Summary Queries

### Zone Redundancy Compliance Score (Article 9.3a)

> **⚠️ Note:** Compliance thresholds (90% = Excellent, 75% = Good, 50% = Needs Improvement) are **contextual examples**. Adjust these percentages based on your organization's risk tolerance and DORA interpretation.

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
| extend ComplianceStatus = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 75, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Improvement',
    '🔴 Critical'
)
| project 
    Metric = 'Article 9.3a - Zone Redundancy',
    Status = ComplianceStatus,
    Value = strcat(ComplianceRate, '%'),
    Compliant = ZoneCompliant,
    NonCompliant
```

### PaaS Network Isolation Summary (Article 9.3a)

> **⚠️ Note:** This query assesses individual PaaS resources (not private endpoint counts). One resource can have multiple private endpoints but is counted once. Compliance thresholds (90% = Excellent, 75% = Good) are **contextual examples**. Adjust based on your organization's network isolation requirements.

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
| extend ComplianceRate = round(todouble(FullyIsolated) / todouble(TotalPaaSResources) * 100, 2)
| extend ComplianceStatus = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 75, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Improvement',
    '🔴 Critical'
)
| project 
    Metric = 'Article 9.3a - PaaS Network Isolation',
    Status = ComplianceStatus,
    Value = strcat(ComplianceRate, '%'),
    ['Fully Isolated'] = FullyIsolated,
    ['Total PaaS'] = TotalPaaSResources
```

### Secure Transfer Compliance (Article 9.3a)

> **⚠️ Note:** Compliance thresholds (95% = Excellent, 80% = Good) are **contextual examples**. Adjust based on your organization's security baseline.

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
| extend ComplianceRate = round(todouble(HttpsCompliant) / todouble(Total) * 100, 2)
| extend TlsRate = round(todouble(TlsCompliant) / todouble(Total) * 100, 2)
| extend ComplianceStatus = case(
    ComplianceRate >= 95 and TlsRate >= 95, '🟢 Excellent',
    ComplianceRate >= 80, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Work',
    '🔴 Critical'
)
| project 
    Metric = 'Article 9.3a - Secure Transfer',
    Status = ComplianceStatus,
    Value = strcat(ComplianceRate, '% HTTPS'),
    TlsCompliance = strcat(TlsRate, '% TLS 1.2+'),
    NonCompliant = Total - HttpsCompliant
```

### Key Vault RBAC Summary (Article 9.3b)

> **⚠️ Note:** Compliance thresholds (90% = Excellent, 70% = Good, 40% = Needs Work) are **contextual examples**. Adjust based on your organization's access control policies.

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
| extend ComplianceRate = round(todouble(RbacEnabled) / todouble(Total) * 100, 2)
| extend ComplianceStatus = case(
    ComplianceRate >= 90 and Total > 0, '🟢 Excellent',
    ComplianceRate >= 70, '🟡 Good',
    ComplianceRate >= 40, '🟠 Needs Work',
    Total == 0, '⚪ No Key Vaults',
    '🔴 Critical'
)
| project 
    Metric = 'Article 9.3b - Key Vault RBAC',
    Status = ComplianceStatus,
    Value = strcat(RbacEnabled, '/', Total),
    SoftDelete = SoftDeleteEnabled,
    PurgeProtected
```

### AAD-Only Authentication Summary (Article 9.3d)

> **⚠️ Note:** Compliance thresholds (90% = Excellent, 70% = Good) are **contextual examples**. Adjust based on your organization's identity governance requirements.

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
| extend ComplianceRate = round(todouble(AadOnlyCompliant) / todouble(Total) * 100, 2)
| extend ComplianceStatus = case(
    Total == 0, '⚪ No Resources',
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 70, '🟡 Good',
    ComplianceRate >= 40, '🟠 Needs Work',
    '🔴 Critical'
)
| project 
    Metric = 'Article 9.3d - AAD-Only Auth',
    Status = ComplianceStatus,
    Value = strcat(ComplianceRate, '%'),
    Compliant = AadOnlyCompliant,
    NonCompliant = Total - AadOnlyCompliant
```

### Microsoft Defender Summary (Article 10.1)

> **⚠️ Note:** Compliance thresholds (70% = Good, 40% = Partial) are **contextual examples**. Adjust based on your organization's security monitoring requirements and budget.

```kql
securityresources
| where type =~ 'microsoft.security/pricings'
| extend pricingTier = tostring(properties.pricingTier)
| extend planName = name
| summarize 
    TotalPlans = dcount(planName),
    EnabledPlans = dcountif(planName, pricingTier == 'Standard')
| extend ComplianceRate = iff(TotalPlans == 0, 0.0, round(todouble(EnabledPlans) / todouble(TotalPlans) * 100, 0))
| extend ComplianceStatus = case(
    TotalPlans == 0, '⚪ No Data',
    EnabledPlans == TotalPlans, '🟢 All Plans Enabled',
    ComplianceRate >= 70, '🟡 Most Plans Enabled',
    ComplianceRate >= 40, '🟠 Partial Coverage',
    '🔴 Limited Coverage'
)
| project 
    Metric = 'Article 10.1 - Defender Plans',
    Status = ComplianceStatus,
    Value = strcat(EnabledPlans, '/', TotalPlans, ' Plans Enabled'),
    Compliant = EnabledPlans,
    NotEnabled = TotalPlans - EnabledPlans
```

### NSG Flow Logs Summary (Article 10.2)

> **⚠️ Note:** Compliance thresholds (90% = Excellent, 70% = Good) are **contextual examples**. Adjust based on your organization's logging and auditing requirements.

```kql
resources
| where type =~ 'microsoft.network/networksecuritygroups'
| extend hasFlowLogs = isnotnull(properties.flowLogs) and array_length(properties.flowLogs) > 0
| summarize 
    Total = count(),
    WithFlowLogs = countif(hasFlowLogs == true)
| extend ComplianceRate = round(todouble(WithFlowLogs) / todouble(Total) * 100, 2)
| extend ComplianceStatus = case(
    Total == 0, '⚪ No NSGs',
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 70, '🟡 Good',
    ComplianceRate >= 40, '🟠 Needs Work',
    '🔴 Critical'
)
| project 
    Metric = 'Article 10.2 - NSG Flow Logs',
    Status = ComplianceStatus,
    Value = strcat(ComplianceRate, '%'),
    WithLogs = WithFlowLogs,
    WithoutLogs = Total - WithFlowLogs
```

### Business Continuity Risk Assessment (Article 10)

> **⚠️ Note:** Risk percentage thresholds (0% = Low, ≤10% = Medium, ≤25% = High, >25% = Critical) are **contextual examples**. Adjust based on your organization's risk tolerance and DORA interpretation.

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
| extend RiskLevel = case(
    RiskPercentage == 0, '🟢 Low Risk',
    RiskPercentage <= 10, '🟡 Medium Risk',
    RiskPercentage <= 25, '🟠 High Risk',
    '🔴 Critical Risk'
)
| project 
    Metric = 'Business Continuity Risk',
    Level = RiskLevel,
    ['Risk %'] = strcat(RiskPercentage, '%'),
    ['At Risk'] = NonCompliant,
    ['Total Resources'] = TotalResources
```

---

## Article 9.3a - ICT Systems Availability

### Virtual Machines Without Availability Zones

```kql
resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend hasZones = isnotempty(zones)
| where not(hasZones)
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend osType = tostring(properties.storageProfile.osDisk.osType)
| extend powerState = tostring(properties.extended.instanceView.powerState.code)
| project 
    ['Virtual Machine'] = name,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['VM Size'] = vmSize,
    ['OS Type'] = osType,
    ['Power State'] = powerState,
    ['DORA Article'] = '9.3a',
    ['DORA Risk'] = '⚠️ No Zone Redundancy',
    ['Resource ID'] = id
| order by ['Virtual Machine'] asc
```

### Network Infrastructure Without Zone Redundancy

```kql
resources
| where type in~ (
    'microsoft.network/loadbalancers',
    'microsoft.network/publicipaddresses',
    'microsoft.network/applicationgateways',
    'microsoft.network/azurefirewalls'
)
| extend hasZones = isnotempty(zones)
| where not(hasZones)
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
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['SKU/Tier'] = skuTier,
    ['DORA Article'] = '9.3a',
    ['DORA Impact'] = '🔴 High - Critical Infrastructure',
    ['Resource ID'] = id
| order by ['Resource Type'] asc, ['Resource Name'] asc
```

### Redis Cache SSL/TLS Compliance

```kql
resources
| where type =~ 'microsoft.cache/redis'
| extend sslEnabled = tobool(properties.enableNonSslPort) != true
| extend minTlsVersion = tostring(properties.minimumTlsVersion)
| extend DORACompliance = case(
    sslEnabled == true and minTlsVersion == '1.2', '✅ Compliant',
    sslEnabled == true, '⚠️ Partial - TLS Version',
    '❌ Non-Compliant - SSL Disabled'
)
| project 
    ['Redis Cache'] = name,
    ['DORA Compliance'] = DORACompliance,
    ['SSL Only'] = case(sslEnabled, 'Yes', 'No'),
    ['Min TLS Version'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['DORA Compliance'] asc
```

### Database SSL Enforcement

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
| extend DORACompliance = case(
    sslEnforcement =~ 'Enabled' and minTlsVersion in ('TLS1_2', '1.2'), '✅ Compliant',
    sslEnforcement =~ 'Enabled', '⚠️ Partial - Check TLS',
    '❌ Non-Compliant'
)
| project 
    ['Database'] = name,
    ['Type'] = dbType,
    ['DORA Compliance'] = DORACompliance,
    ['SSL Enforcement'] = sslEnforcement,
    ['Min TLS Version'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['DORA Compliance'] asc, ['Type'] asc
```

### Recovery Services Vaults - Business Continuity & Backup Compliance

> **⚠️ Note:** This query validates compliance with **Article 11 (Business Continuity & Disaster Recovery Plans)** and **Article 9.3a (Network Security)**. Thresholds for backup storage redundancy, soft delete retention, and immutability should be adjusted based on your organization's business continuity requirements and regulatory compliance needs.

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
| extend privateEndpointState = case(
    hasPrivateEndpoint, tostring(privateEndpointConnections[0].properties.privateLinkServiceConnectionState.status),
    'No Private Endpoint'
)
| extend geoRedundant = backupStorageType in~ ('GeoRedundant', 'ReadAccessGeoZoneRedundant', 'ZoneRedundant')
| extend softDeleteEnabled = softDeleteState =~ 'Enabled'
| extend immutableVault = immutableState =~ 'Enabled'
| extend DORAArticle11Compliance = case(
    geoRedundant and crossRegionRestore and softDeleteEnabled and immutableVault, '✅ Article 11 Fully Compliant',
    geoRedundant and crossRegionRestore and softDeleteEnabled, '⚠️ Missing Immutability Protection',
    geoRedundant and softDeleteEnabled, '⚠️ Missing Cross-Region Restore',
    geoRedundant, '⚠️ Missing Soft Delete or CRR',
    '❌ Not Geo-Redundant'
)
| extend DORAArticle93aCompliance = case(
    hasPrivateEndpoint and privateEndpointState =~ 'Approved', '✅ Article 9.3a Compliant',
    hasPrivateEndpoint, '⚠️ Private Endpoint Pending',
    '❌ No Private Endpoint'
)
| extend serviceType = 'Recovery Services'
| project 
    ['Recovery Vault'] = name,
    ['Service Type'] = serviceType,
    ['Article 11 Compliance'] = DORAArticle11Compliance,
    ['Article 9.3a Compliance'] = DORAArticle93aCompliance,
    ['Backup Storage'] = backupStorageType,
    ['Cross-Region Restore'] = case(crossRegionRestore == true, '✅ Enabled', '❌ Disabled'),
    ['Soft Delete'] = case(softDeleteEnabled, '✅ Enabled', '❌ Disabled'),
    ['Immutable Vault'] = case(immutableVault, '✅ Enabled', '❌ Disabled'),
    ['Private Endpoint'] = case(hasPrivateEndpoint, '✅ Yes', '❌ No'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Articles'] = 'Article 11 (BC) & 9.3a (Network)',
    ['Resource ID'] = id
| order by ['Article 11 Compliance'] asc, ['Article 9.3a Compliance'] asc
```

### API Management - Protocol Security

```kql
resources
| where type =~ 'microsoft.apimanagement/service'
| extend skuName = tostring(sku.name)
| extend protocols = properties.customProperties
| extend http2Enabled = tobool(protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Protocols.Server.Http2'])
| extend tlsEnabled = tobool(protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls12'])
| extend tls10Disabled = protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10'] == 'False'
| extend tls11Disabled = protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11'] == 'False'
| extend ssl30Disabled = protocols['Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Ssl30'] == 'False'
| extend hasVnet = isnotnull(properties.virtualNetworkConfiguration)
| extend vnetType = tostring(properties.virtualNetworkType)
| extend hasPrivateEndpoint = isnotnull(properties.privateEndpointConnections) and array_length(properties.privateEndpointConnections) > 0
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend DORACompliance = case(
    tls10Disabled and tls11Disabled and ssl30Disabled and hasPrivateEndpoint, '✅ Fully Compliant',
    tls10Disabled and tls11Disabled and ssl30Disabled, '⚠️ Secure TLS - No PE',
    hasPrivateEndpoint, '⚠️ Has PE - Check TLS Settings',
    '❌ Non-Compliant - Legacy Protocols'
)
| project 
    ['APIM Service'] = name,
    ['DORA Status'] = DORACompliance,
    ['SKU'] = skuName,
    ['TLS 1.0 Disabled'] = case(tls10Disabled, '✅ Yes', '❌ No'),
    ['TLS 1.1 Disabled'] = case(tls11Disabled, '✅ Yes', '❌ No'),
    ['SSL 3.0 Disabled'] = case(ssl30Disabled, '✅ Yes', '❌ No'),
    ['Private Endpoint'] = case(hasPrivateEndpoint, '✅ Yes', '❌ No'),
    ['VNet Type'] = coalesce(vnetType, 'None'),
    ['Public Access'] = coalesce(publicNetworkAccess, 'Enabled'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

### Private Endpoints by Location

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize Count = count() by location
| order by Count desc
```

### Private Endpoints by Connection State

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend connectionState = tostring(privateLinkServiceConnection.properties.privateLinkServiceConnectionState.status)
| summarize Count = count() by connectionState
| order by Count desc
```

### Private Endpoints Detailed Inventory

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend tenantId = tostring(tenantId)
| extend subscriptionId = tostring(subscriptionId)
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
| extend DORACompliance = case(
    connectionState == 'Approved', '✅ Article 9.3a Compliant',
    connectionState == 'Pending', '⚠️ Pending Approval',
    '❌ Non-Compliant'
)
| project 
    ['Private Endpoint'] = privateEndpointName,
    ['DORA Status'] = DORACompliance,
    ['Connection State'] = connectionState,
    ['Target Resource'] = targetResourceName,
    ['Target Type'] = targetResourceType,
    ['Sub Resource'] = groupIds,
    ['Virtual Network'] = vnetName,
    ['Subnet'] = subnetName,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['Target Resource ID'] = targetResourceId
| order by ['DORA Status'] asc, ['Private Endpoint'] asc
```

### Private Endpoints by Target Resource Type

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend targetResourceId = tostring(privateLinkServiceConnection.properties.privateLinkServiceId)
| extend targetResourceType = tostring(split(targetResourceId, '/providers/')[-1])
| extend targetType = tostring(split(targetResourceType, '/')[0])
| summarize Count = count() by targetType
| order by Count desc
| project ['Target Resource Type'] = targetType, ['Private Endpoint Count'] = Count
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
| extend DORACompliance = case(
    publicNetworkAccess =~ 'SecuredByPerimeter', '🔒 Network Security Perimeter',
    hasPrivateEndpoint and publicNetworkAccess =~ 'Disabled', '✅ Fully Isolated',
    hasPrivateEndpoint and publicNetworkAccess =~ 'Enabled', '⚠️ Has PE - Public Access Enabled',
    publicNetworkAccess =~ 'Disabled', '🔐 Public Access Disabled - No PE',
    publicNetworkAccess =~ 'Unknown' or isempty(publicNetworkAccess), '🔍 Review Required - Unknown Status',
    '❌ No Private Endpoint'
)
| project 
    ['Resource Name'] = name,
    ['Resource Type'] = resourceType,
    ['DORA Status'] = DORACompliance,
    ['Private Endpoints'] = peCount,
    ['Public Network'] = publicNetworkAccess,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['DORA Status'] asc, ['Resource Type'] asc, ['Resource Name'] asc
```

---

## Article 9.3b - Access Control

### Key Vault RBAC Authorization & Protection

```kql
resources
| where type =~ 'microsoft.keyvault/vaults'
| extend enableRbac = tobool(properties.enableRbacAuthorization)
| extend softDelete = tobool(properties.enableSoftDelete)
| extend purgeProtection = tobool(properties.enablePurgeProtection)
| extend networkAcls = properties.networkAcls
| extend defaultAction = tostring(networkAcls.defaultAction)
| extend DORACompliance = case(
    enableRbac == true and softDelete == true and purgeProtection == true, '✅ Fully Compliant',
    enableRbac == true and softDelete == true, '⚠️ Missing Purge Protection',
    enableRbac == true, '⚠️ Partial Compliance',
    '❌ Non-Compliant - No RBAC'
)
| project 
    ['Key Vault'] = name,
    ['DORA Status'] = DORACompliance,
    ['RBAC Enabled'] = case(enableRbac == true, '✅ Yes', '❌ No'),
    ['Soft Delete'] = case(softDelete == true, '✅ Yes', '❌ No'),
    ['Purge Protection'] = case(purgeProtection == true, '✅ Yes', '❌ No'),
    ['Network Default'] = defaultAction,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3b',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

### Storage Accounts Encryption & Redundancy

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
| extend DORACompliance = case(
    isZoneRedundant and httpsOnly == true and (minTlsVersion == 'TLS1_2' or minTlsVersion == 'TLS1_3'), '✅ Fully Compliant',
    isGeoRedundant and httpsOnly == true, '⚠️ Geo-Redundant Only',
    httpsOnly == true, '⚠️ HTTPS Only - No Zone/Geo',
    '❌ Non-Compliant'
)
| extend RiskLevel = case(
    isZoneRedundant and httpsOnly == true, 'Low',
    isGeoRedundant and httpsOnly == true, 'Medium',
    httpsOnly == true, 'Medium',
    'High'
)
| project 
    ['Storage Account'] = name,
    ['DORA Compliance'] = DORACompliance,
    ['Risk Level'] = RiskLevel,
    ['Replication'] = skuName,
    ['CMK Enabled'] = case(cmkEnabled, '✅ Yes', '❌ No'),
    ['HTTPS Only'] = case(httpsOnly == true, '✅ Yes', '❌ No'),
    ['Min TLS'] = minTlsVersion,
    ['Kind'] = storageKind,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3b/9.3c',
    ['Resource ID'] = id
| order by ['Risk Level'] desc, ['Storage Account'] asc
```

### Container Registry Security Configuration

```kql
resources
| where type =~ 'microsoft.containerregistry/registries'
| extend skuName = tostring(sku.name)
| extend adminEnabled = tobool(properties.adminUserEnabled)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend encryption = properties.encryption
| extend cmkEnabled = tostring(encryption.status) == 'enabled'
| extend zoneRedundancy = tostring(properties.zoneRedundancy)
| extend DORACompliance = case(
    adminEnabled == false and publicNetworkAccess == 'Disabled', '✅ Fully Compliant',
    adminEnabled == false, '⚠️ Admin Disabled - Public Access',
    '❌ Non-Compliant - Admin Enabled'
)
| project 
    ['Container Registry'] = name,
    ['DORA Status'] = DORACompliance,
    ['SKU'] = skuName,
    ['Admin User'] = case(adminEnabled == true, '❌ Enabled', '✅ Disabled'),
    ['Public Access'] = publicNetworkAccess,
    ['CMK Enabled'] = case(cmkEnabled, '✅ Yes', '❌ No'),
    ['Zone Redundant'] = zoneRedundancy,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3b',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

### AKS Clusters - RBAC & Security Configuration

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
| extend DORACompliance = case(
    enableRbac == true and aadEnabled and azureRbacEnabled == true and localAccounts == false, '✅ Fully Compliant',
    enableRbac == true and aadEnabled and azureRbacEnabled == true, '⚠️ Local Accounts Enabled',
    enableRbac == true and aadEnabled, '⚠️ Azure RBAC Not Enabled',
    enableRbac == true, '⚠️ AAD Not Integrated',
    '❌ Non-Compliant - RBAC Disabled'
)
| project 
    ['AKS Cluster'] = name,
    ['DORA Status'] = DORACompliance,
    ['K8s RBAC'] = case(enableRbac == true, '✅ Enabled', '❌ Disabled'),
    ['AAD Integrated'] = case(aadEnabled, '✅ Yes', '❌ No'),
    ['Azure RBAC'] = case(azureRbacEnabled == true, '✅ Yes', '❌ No'),
    ['Local Accounts'] = case(localAccounts, '❌ Enabled', '✅ Disabled'),
    ['Network Policy'] = coalesce(networkPolicy, 'None'),
    ['Tier'] = tier,
    ['Zone Redundant'] = case(hasZones, '✅ Yes', '❌ No'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3b',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

---

## Article 9.3c - Data Protection

### Databases High Availability & Zone Redundancy

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
| extend DORACompliance = case(
    hasZones, '✅ Article 9.3c Compliant',
    '❌ Non-Compliant'
)
| project 
    ['Database'] = name,
    ['Type'] = databaseType,
    ['DORA Status'] = DORACompliance,
    ['HA Mode'] = haMode,
    ['Zone Redundant'] = case(hasZones, '✅ Yes', '❌ No'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3c',
    ['Resource ID'] = id
| order by ['DORA Status'] asc, ['Database'] asc
```

### Container Services Zone Redundancy

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
| extend DORACompliance = case(
    hasZones, '✅ Zone Redundant',
    '❌ No Zone Redundancy'
)
| project 
    ['Service Name'] = name,
    ['Type'] = serviceType,
    ['DORA Status'] = DORACompliance,
    ['Tier'] = tier,
    ['Network Plugin'] = networkProfile,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3c',
    ['Resource ID'] = id
| order by ['DORA Status'] asc, ['Service Name'] asc
```

### App Services HTTPS/FTPS/TLS Compliance

```kql
resources
| where type =~ 'microsoft.web/sites'
| extend hasZones = isnotempty(zones)
| extend httpsOnly = tobool(properties.httpsOnly)
| extend minTlsVersion = tostring(properties.siteConfig.minTlsVersion)
| extend ftpsState = tostring(properties.siteConfig.ftpsState)
| extend kind = tostring(kind)
| extend DORACompliance = case(
    httpsOnly == true and ftpsState == 'FtpsOnly' and (minTlsVersion == '1.2' or minTlsVersion == '1.3'), '✅ Fully Compliant',
    httpsOnly == true and ftpsState == 'FtpsOnly', '⚠️ Check TLS Version',
    httpsOnly == true, '⚠️ FTPS Not Enforced',
    '❌ Non-Compliant'
)
| project 
    ['App Service'] = name,
    ['Kind'] = kind,
    ['DORA Status'] = DORACompliance,
    ['HTTPS Only'] = case(httpsOnly == true, '✅ Yes', '❌ No'),
    ['FTPS State'] = ftpsState,
    ['Min TLS'] = minTlsVersion,
    ['Zone Redundant'] = case(hasZones, '✅ Yes', '❌ No'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a/9.3c',
    ['Resource ID'] = id
| order by ['DORA Status'] asc, ['App Service'] asc
```

### Key Vault - Key Protection Settings

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
| extend vaultUri = tostring(properties.vaultUri)
| extend DORAKeyRotation = case(
    softDelete == true and purgeProtection == true and softDeleteRetention >= 90, '✅ Key Protection Ready',
    softDelete == true and purgeProtection == true, '⚠️ Short Retention Period',
    softDelete == true, '⚠️ Missing Purge Protection',
    '❌ Non-Compliant'
)
| project 
    ['Key Vault'] = name,
    ['DORA Status'] = DORAKeyRotation,
    ['Soft Delete'] = case(softDelete == true, '✅ Yes', '❌ No'),
    ['Purge Protection'] = case(purgeProtection == true, '✅ Yes', '❌ No'),
    ['Retention Days'] = softDeleteRetention,
    ['RBAC Enabled'] = case(enableRbac == true, '✅ Yes', '❌ No'),
    ['For Deployment'] = case(enabledForDeployment == true, '✅ Yes', '❌ No'),
    ['For Disk Encryption'] = case(enabledForDiskEncryption == true, '✅ Yes', '❌ No'),
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3c',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

### Integration Services Zone Redundancy

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
| extend 
    tier = tostring(sku.name)
| extend DORACompliance = case(
    hasZones, '✅ Zone Redundant',
    '❌ No Zone Redundancy'
)
| project 
    ['Service Name'] = name,
    ['Type'] = serviceType,
    ['DORA Status'] = DORACompliance,
    ['SKU'] = tier,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3c',
    ['Resource ID'] = id
| order by ['DORA Status'] asc, ['Service Name'] asc
```

### Analytics & AI Services - Private Link Compliance

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
| extend DORACompliance = case(
    hasPrivateEndpoint and publicNetworkAccess =~ 'Disabled', '✅ Fully Compliant',
    hasPrivateEndpoint, '⚠️ PE Enabled - Public Access On',
    publicNetworkAccess =~ 'Disabled', '⚠️ Public Disabled - No PE',
    '❌ Non-Compliant'
)
| project 
    ['Service Name'] = name,
    ['Type'] = serviceType,
    ['DORA Status'] = DORACompliance,
    ['Private Endpoint'] = case(hasPrivateEndpoint, '✅ Yes', '❌ No'),
    ['Public Access'] = publicNetworkAccess,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['DORA Status'] asc, ['Service Name'] asc
```

### Azure Databricks - Public IP & Private Link

```kql
resources
| where type =~ 'microsoft.databricks/workspaces'
| extend enableNoPublicIp = tobool(properties.parameters.enableNoPublicIp.value)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend hasPrivateEndpoint = isnotnull(properties.privateEndpointConnections) and array_length(properties.privateEndpointConnections) > 0
| extend pricingTier = tostring(sku.name)
| extend DORACompliance = case(
    enableNoPublicIp == true and hasPrivateEndpoint, '✅ Fully Compliant',
    enableNoPublicIp == true, '⚠️ No Public IP - No PE',
    hasPrivateEndpoint, '⚠️ PE Only - Public IP Enabled',
    '❌ Non-Compliant'
)
| project 
    ['Databricks Workspace'] = name,
    ['DORA Status'] = DORACompliance,
    ['No Public IP'] = case(enableNoPublicIp == true, '✅ Yes', '❌ No'),
    ['Private Endpoint'] = case(hasPrivateEndpoint, '✅ Yes', '❌ No'),
    ['Pricing Tier'] = pricingTier,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3a',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

---

## Article 9.3d - Identity & Authentication

### SQL Servers Azure AD Authentication

```kql
resources
| where type =~ 'microsoft.sql/servers'
| extend aadOnlyAuth = tobool(properties.administrators.azureADOnlyAuthentication)
| extend aadAdmin = tostring(properties.administrators.login)
| extend publicNetworkAccess = tostring(properties.publicNetworkAccess)
| extend minTlsVersion = tostring(properties.minimalTlsVersion)
| extend DORACompliance = case(
    aadOnlyAuth == true, '✅ AAD-Only Compliant',
    isnotnull(aadAdmin) and aadAdmin != '', '⚠️ AAD Admin Set - SQL Auth Enabled',
    '❌ Non-Compliant - No AAD'
)
| project 
    ['SQL Server'] = name,
    ['DORA Status'] = DORACompliance,
    ['AAD-Only Auth'] = case(aadOnlyAuth == true, '✅ Enabled', '❌ Disabled'),
    ['AAD Admin'] = aadAdmin,
    ['Public Network'] = publicNetworkAccess,
    ['Min TLS'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '9.3d',
    ['Resource ID'] = id
| order by ['DORA Status'] asc
```

---

## Article 10.1 - Detection & Monitoring

### Microsoft Defender for Cloud Plan Status

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
| extend DORACompliance = case(
    pricingTier == 'Standard', '✅ Enabled',
    '❌ Not Enabled'
)
| project 
    ['Defender Plan'] = defenderPlan,
    ['Resource Name'] = resourceName,
    ['DORA Status'] = DORACompliance,
    ['Pricing Tier'] = pricingTier,
    ['Subscription'] = subscriptionId,
    ['DORA Article'] = '10.1'
| order by ['DORA Status'] asc, ['Defender Plan'] asc
```

---

## Article 10.2 - Incident Response & Logging

### Network Security Groups - Flow Log Requirements

```kql
resources
| where type =~ 'microsoft.network/networksecuritygroups'
| extend nsgId = id
| extend nsgName = name
| extend subnetCount = array_length(properties.subnets)
| extend nicCount = array_length(properties.networkInterfaces)
| extend rulesCount = array_length(properties.securityRules)
| extend hasFlowLogs = isnotnull(properties.flowLogs) and array_length(properties.flowLogs) > 0
| extend DORAStatus = case(
    hasFlowLogs, '✅ Flow Logs Enabled',
    subnetCount > 0 or nicCount > 0, '⚠️ Attached - Verify Flow Logs',
    '📋 Not Attached'
)
| project 
    ['NSG Name'] = nsgName,
    ['DORA Status'] = DORAStatus,
    ['Security Rules'] = rulesCount,
    ['Attached Subnets'] = subnetCount,
    ['Attached NICs'] = nicCount,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '10.2',
    ['Resource ID'] = nsgId
| order by ['Attached Subnets'] desc, ['Attached NICs'] desc
```

### PostgreSQL Flexible Servers - Logging Configuration

```kql
resources
| where type =~ 'microsoft.dbforpostgresql/flexibleservers'
| extend version = tostring(properties.version)
| extend haMode = tostring(properties.highAvailability.mode)
| extend state = tostring(properties.state)
| extend logCheckpoints = tobool(properties.dataEncryption)
| extend DORACompliance = case(
    state == 'Ready', '✅ Server Ready',
    '⚠️ Check Status'
)
| project 
    ['PostgreSQL Server'] = name,
    ['DORA Status'] = DORACompliance,
    ['Version'] = version,
    ['HA Mode'] = haMode,
    ['State'] = state,
    ['Resource Group'] = resourceGroup,
    ['Subscription'] = subscriptionId,
    Location = location,
    ['DORA Article'] = '10.2 - Logging',
    ['Resource ID'] = id
| order by ['PostgreSQL Server'] asc
```

---

## Analytics Queries

### Zone Redundancy Compliance by Resource Type

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
    Compliant = countif(hasZones),
    NonCompliant = countif(not(hasZones))
    by resourceType
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 2)
| order by NonCompliant desc
| project 
    ['Resource Type'] = resourceType,
    ['Total Resources'] = Total,
    ['DORA Compliant'] = Compliant,
    ['Non-Compliant'] = NonCompliant,
    ['Compliance %'] = ComplianceRate
```

### DORA Compliance by Azure Region

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
    Compliant = countif(hasZones),
    NonCompliant = countif(not(hasZones))
    by location
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 2)
| order by NonCompliant desc
| project 
    ['Azure Region'] = location,
    ['Total Resources'] = Total,
    ['Compliant'] = Compliant,
    ['At Risk'] = NonCompliant,
    ['Compliance Rate %'] = ComplianceRate
```

### DORA Compliance by Subscription

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
    ZoneCompliant = countif(hasZones),
    AtRisk = countif(not(hasZones)),
    ResourceTypes = dcount(type)
    by subscriptionId
| extend ComplianceRate = round(todouble(ZoneCompliant) / todouble(TotalResources) * 100, 2)
| extend DORAStatus = case(
    ComplianceRate >= 90, '✅ Excellent',
    ComplianceRate >= 75, '🟡 Good',
    ComplianceRate >= 50, '🟠 Needs Action',
    '🔴 Critical'
)
| order by AtRisk desc
| project 
    ['Subscription ID'] = subscriptionId,
    ['DORA Status'] = DORAStatus,
    ['Total Resources'] = TotalResources,
    ['Compliant'] = ZoneCompliant,
    ['At Risk'] = AtRisk,
    ['Compliance %'] = ComplianceRate,
    ['Resource Types'] = ResourceTypes
```

### Private Endpoints by Tenant and Subscription

```kql
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend tenantId = tostring(tenantId)
| extend subscriptionId = tostring(subscriptionId)
| summarize 
    ['Private Endpoints Count'] = count(),
    ['Resource Groups'] = dcount(resourceGroup),
    Locations = dcount(location)
    by tenantId, subscriptionId
| order by ['Private Endpoints Count'] desc
| project 
    ['Tenant ID'] = tenantId,
    ['Subscription ID'] = subscriptionId,
    ['Private Endpoints Count'],
    ['Resource Groups'],
    Locations
```

### Cross-Subscription & Cross-Tenant Private Endpoints

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
| extend isCrossSubscription = iff(peSubscriptionId != targetSubscriptionId and targetSubscriptionId != '', 'Yes', 'No')
| join kind=leftouter (
    resources
    | extend targetTenantId = tostring(tenantId)
    | extend idLower = tolower(id)
    | project idLower, targetTenantId
) on $left.targetResourceIdLower == $right.idLower
| extend finalTargetTenant = iff(isempty(targetTenantId), iff(isempty(targetSubscriptionId), 'N/A', peTenantId), targetTenantId)
| extend isCrossTenant = iff(isempty(targetTenantId) and isnotempty(targetSubscriptionId), iff(peSubscriptionId != targetSubscriptionId, 'Unknown - Verify Access', 'No'), iff(isempty(targetTenantId), 'Unknown', iff(peTenantId != targetTenantId, 'Yes', 'No')))
| extend DORAStatus = case(
    isCrossTenant == 'Yes', '🔴 Cross-Tenant - Review Required',
    isCrossTenant == 'Unknown - Verify Access', '⚠️ Verify Target Resource Access',
    isCrossSubscription == 'Yes', '🟡 Cross-Subscription',
    '✅ Same Subscription'
)
| project 
    ['Private Endpoint'] = name,
    ['DORA Status'] = DORAStatus,
    ['Connection State'] = connectionState,
    ['PE Tenant'] = peTenantId,
    ['PE Subscription'] = peSubscriptionId,
    ['PE Resource Group'] = resourceGroup,
    ['Target Tenant'] = finalTargetTenant,
    ['Target Subscription'] = targetSubscriptionId,
    ['Target Resource'] = tostring(split(targetResourceId, '/')[-1]),
    Location = location
| order by ['DORA Status'] asc, ['Private Endpoint'] asc
```

---

## 📝 Usage Notes

### Running Queries in Azure Resource Graph Explorer

1. Navigate to **Azure Portal** → **Resource Graph Explorer**
2. Copy and paste any query from this document
3. Click **Run query** - queries work immediately without modification
4. Use the export options to download results as CSV, JSON, or Excel

### Scoping to Specific Resources (Optional)

All queries scan your entire accessible scope by default. To filter to specific resources, add filters:

**By Resource Group:**
```kql
resources
| where type =~ 'microsoft.compute/virtualmachines'
| where resourceGroup in ('my-rg-1', 'my-rg-2')  // Add this line
| extend hasZones = isnotempty(zones)
// ... rest of query
```

**By Subscription:**
```kql
resources  
| where type =~ 'microsoft.compute/virtualmachines'
| where subscriptionId in ('sub-id-1', 'sub-id-2')  // Add this line
| extend hasZones = isnotempty(zones)
// ... rest of query
```

**By Location:**
```kql
resources
| where type =~ 'microsoft.compute/virtualmachines'
| where location in ('eastus', 'westeurope')  // Add this line
| extend hasZones = isnotempty(zones)
// ... rest of query
```

---

## 🔧 Removing Compliance Scores and Thresholds

Many queries in this document include **compliance scoring logic** (e.g., "✅ Compliant", "⚠️ Partial", "❌ Non-Compliant") and **threshold-based assessments** (e.g., "90% = Excellent"). If you need **raw resource data** without these assessments, you can simplify queries by removing the scoring logic.

### Why Remove Compliance Scoring?

- **Custom Analysis**: You want to apply your own compliance logic in Excel/Power BI
- **Data Export**: You need raw data for integration with other tools
- **Simplified Reports**: You just want resource lists without assessment logic
- **Performance**: Reducing `case` statements can slightly improve query performance on large datasets

### How to Simplify Queries

**Step 1: Remove the `DORACompliance` or `ComplianceStatus` field**

Find and delete lines like:
```kql
| extend DORACompliance = case(
    condition1, '✅ Compliant',
    condition2, '⚠️ Partial',
    '❌ Non-Compliant'
)
```

**Step 2: Remove compliance columns from the `project` statement**

Change:
```kql
| project 
    ['Resource Name'] = name,
    ['DORA Status'] = DORACompliance,  // Remove this
    ['Zone Redundant'] = hasZones,
    ...
```

To:
```kql
| project 
    ['Resource Name'] = name,
    ['Zone Redundant'] = hasZones,
    ...
```

**Step 3: Remove threshold-based summarizations (optional)**

For summary queries, remove the `ComplianceStatus` calculation:

Change:
```kql
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 2)
| extend ComplianceStatus = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 75, '🟡 Good',
    '🔴 Critical'
)
```

To:
```kql
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 2)
```

### Example Transformation

**Original Query (with compliance scoring):**
```kql
resources
| where type =~ 'microsoft.storage/storageaccounts'
| extend httpsOnly = tobool(properties.supportsHttpsTrafficOnly)
| extend minTlsVersion = tostring(properties.minimumTlsVersion)
| extend DORACompliance = case(
    httpsOnly == true and minTlsVersion == 'TLS1_2', '✅ Compliant',
    httpsOnly == true, '⚠️ Partial - Check TLS',
    '❌ Non-Compliant'
)
| project 
    ['Storage Account'] = name,
    ['DORA Status'] = DORACompliance,
    ['HTTPS Only'] = httpsOnly,
    ['Min TLS'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    Location = location
| order by ['DORA Status'] asc
```

**Simplified Query (raw data only):**
```kql
resources
| where type =~ 'microsoft.storage/storageaccounts'
| extend httpsOnly = tobool(properties.supportsHttpsTrafficOnly)
| extend minTlsVersion = tostring(properties.minimumTlsVersion)
| project 
    ['Storage Account'] = name,
    ['HTTPS Only'] = httpsOnly,
    ['Min TLS'] = minTlsVersion,
    ['Resource Group'] = resourceGroup,
    Location = location,
    ['Resource ID'] = id
| order by ['Storage Account'] asc
```

**What Changed:**
- ❌ Removed `DORACompliance` calculation
- ❌ Removed `['DORA Status']` column from output
- ✅ Changed sort from compliance status to alphabetical by name
- ✅ Added `['Resource ID']` for reference

### Example: Simplifying Summary Queries

**Original (with thresholds):**
```kql
resources
| where type =~ 'microsoft.keyvault/vaults'
| extend enableRbac = tobool(properties.enableRbacAuthorization)
| summarize 
    Total = count(),
    RbacEnabled = countif(enableRbac == true)
| extend ComplianceRate = round(todouble(RbacEnabled) / todouble(Total) * 100, 2)
| extend ComplianceStatus = case(
    ComplianceRate >= 90, '🟢 Excellent',
    ComplianceRate >= 70, '🟡 Good',
    '🔴 Critical'
)
| project 
    Metric = 'Key Vault RBAC',
    Status = ComplianceStatus,
    Value = strcat(RbacEnabled, '/', Total),
    ['Compliance %'] = ComplianceRate
```

**Simplified (counts only):**
```kql
resources
| where type =~ 'microsoft.keyvault/vaults'
| extend enableRbac = tobool(properties.enableRbacAuthorization)
| summarize 
    Total = count(),
    RbacEnabled = countif(enableRbac == true),
    RbacDisabled = countif(enableRbac == false)
| extend ComplianceRate = round(todouble(RbacEnabled) / todouble(Total) * 100, 2)
| project 
    ['Total Key Vaults'] = Total,
    ['RBAC Enabled'] = RbacEnabled,
    ['RBAC Disabled'] = RbacDisabled,
    ['Percentage Enabled'] = ComplianceRate
```

**What Changed:**
- ❌ Removed `ComplianceStatus` with emoji indicators
- ✅ Added `RbacDisabled` count for completeness
- ✅ Renamed columns to be more descriptive
- ✅ Kept percentage calculation for reference

### Quick Tips

1. **Keep the `extend` logic**: Even when removing compliance scoring, keep the `extend` statements that calculate technical properties (e.g., `hasZones`, `httpsOnly`) - these are useful data points
2. **Use boolean outputs**: Replace compliance strings with simple boolean columns: `case(hasZones, 'Yes', 'No')` or just `hasZones`
3. **Add Resource IDs**: Always include `['Resource ID'] = id` in simplified queries for cross-referencing
4. **Sort by name**: Change `order by ['DORA Status']` to `order by name` or `order by ['Resource Group']`
5. **Export-friendly**: Simplified queries work better with CSV exports and data analysis tools

### Query Performance Tips

- Use specific resource type filters (`where type =~ '...'`) to reduce scan scope
- Add subscription/resource group filters early in the query pipeline
- For large environments, consider filtering by location or tags
- Use `| take 100` at the end to limit results during testing

### Working with Results

- **CSV Export**: Best for Excel analysis and reporting
- **JSON Export**: Best for automation and integration
- **Save Query**: Use the "Save" option to keep queries in your Resource Graph Explorer library
- **Pin to Dashboard**: Create Azure Portal dashboards with query results

---

## 🔐 Permissions Required

To run these queries, you need:

- **Reader** role on subscriptions/resource groups you want to query
- **Security Reader** role for queries using `securityresources` table (Microsoft Defender queries)

---

## 📚 References

- [DORA Regulation (EU) 2022/2554](https://eur-lex.europa.eu/eli/reg/2022/2554/oj)
- [Azure Resource Graph Query Language](https://learn.microsoft.com/azure/governance/resource-graph/concepts/query-language)
- [Azure Policy Initiative for DORA](https://learn.microsoft.com/azure/governance/policy/samples/built-in-initiatives#regulatory-compliance)
