# Private Endpoints Inventory Queries

Test these queries in Azure Resource Graph Explorer to verify they work in your environment.

## 1. Resource Groups Parameter Query

```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize by resourceGroup
| order by resourceGroup asc
| project value = resourceGroup, label = resourceGroup, selected = false
```

## 2. Total Private Endpoints Count

**For Workbook:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| where '{ResourceGroups}' == 'value::all' or resourceGroup in ({ResourceGroups})
| summarize TotalCount = count()
| extend MetricName = 'Total Private Endpoints'
| project MetricName, Value = TotalCount
```

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize TotalCount = count()
| extend MetricName = 'Total Private Endpoints'
| project MetricName, Value = TotalCount
```

## 3. Subscriptions with Private Endpoints

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize SubscriptionCount = dcount(subscriptionId)
| extend MetricName = 'Subscriptions'
| project MetricName, Value = SubscriptionCount
```

## 4. Resource Groups with Private Endpoints

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize ResourceGroupCount = dcount(resourceGroup)
| extend MetricName = 'Resource Groups'
| project MetricName, Value = ResourceGroupCount
```

## 5. Private Endpoints by Location

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize Count = count() by location
| order by Count desc
| project Location = location, ['Private Endpoints'] = Count
```

## 6. Private Endpoints by Connection State

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| mv-expand privateLinkServiceConnections = properties.privateLinkServiceConnections
| extend connectionState = tostring(privateLinkServiceConnections.properties.privateLinkServiceConnectionState.status)
| summarize Count = count() by connectionState
| extend connectionState = iff(connectionState == '', 'Not Specified', connectionState)
| order by Count desc
| project ['Connection State'] = connectionState, Count
```

## 7. Detailed Private Endpoints Inventory

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend tenantId = tostring(tenantId)
| extend subscriptionId = tostring(subscriptionId)
| extend privateEndpointName = name
| extend subnet = tostring(properties.subnet.id)
| extend subnetName = tostring(split(subnet, '/')[-1])
| extend vnetName = tostring(split(subnet, '/')[-3])
| extend networkInterfaces = properties.networkInterfaces
| mv-expand networkInterface = networkInterfaces
| extend nicId = tostring(networkInterface.id)
| extend nicName = tostring(split(nicId, '/')[-1])
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend connectionName = tostring(privateLinkServiceConnection.properties.privateLinkServiceConnectionState.status)
| extend targetResourceId = tostring(privateLinkServiceConnection.properties.privateLinkServiceId)
| extend targetResourceName = tostring(split(targetResourceId, '/')[-1])
| extend targetResourceType = tostring(split(targetResourceId, '/')[-2])
| extend groupIds = tostring(privateLinkServiceConnection.properties.groupIds[0])
| project 
    TenantId = tenantId,
    SubscriptionId = subscriptionId,
    ['Private Endpoint'] = privateEndpointName,
    ['Resource Group'] = resourceGroup,
    Location = location,
    ['Connection State'] = connectionName,
    ['Target Resource Name'] = targetResourceName,
    ['Target Resource Type'] = targetResourceType,
    ['Sub Resource (Group ID)'] = groupIds,
    ['Virtual Network'] = vnetName,
    Subnet = subnetName,
    ['Network Interface'] = nicName,
    ['Target Resource ID'] = targetResourceId,
    ['Private Endpoint ID'] = id
| order by ['Private Endpoint'] asc
```

## 8. Private Endpoints by Target Resource Type

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend targetResourceId = tostring(privateLinkServiceConnection.properties.privateLinkServiceId)
| extend targetResourceType = tostring(split(targetResourceId, '/')[-2])
| extend provider = tostring(split(targetResourceId, '/')[6])
| extend fullResourceType = strcat(provider, '/', targetResourceType)
| summarize Count = count() by fullResourceType
| order by Count desc
| project ['Target Resource Type'] = fullResourceType, ['Number of Private Endpoints'] = Count
```

## 9. Cross-Subscription Private Endpoints

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend peSubscriptionId = tostring(subscriptionId)
| extend peTenantId = tostring(tenantId)
| mv-expand privateLinkServiceConnection = properties.privateLinkServiceConnections
| extend targetResourceId = tostring(privateLinkServiceConnection.properties.privateLinkServiceId)
| extend targetSubscriptionId = tostring(split(targetResourceId, '/')[2])
| extend isCrossSubscription = iff(peSubscriptionId != targetSubscriptionId and targetSubscriptionId != '', 'Yes', 'No')
| where isCrossSubscription == 'Yes'
| project 
    ['Private Endpoint'] = name,
    ['PE Tenant'] = peTenantId,
    ['PE Subscription'] = peSubscriptionId,
    ['PE Resource Group'] = resourceGroup,
    ['Target Subscription'] = targetSubscriptionId,
    ['Target Resource'] = tostring(split(targetResourceId, '/')[-1]),
    ['Connection State'] = tostring(privateLinkServiceConnection.properties.privateLinkServiceConnectionState.status),
    Location = location
| order by ['Private Endpoint'] asc
```

## 10. Private Endpoints with IP Configuration Details

**For Testing in Resource Graph Explorer:**
```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| extend networkInterfaces = properties.networkInterfaces
| mv-expand networkInterface = networkInterfaces
| extend nicId = tostring(networkInterface.id)
| project 
    ['Private Endpoint'] = name,
    ['Tenant ID'] = tenantId,
    ['Subscription ID'] = subscriptionId,
    ['Resource Group'] = resourceGroup,
    Location = location,
    ['Network Interface ID'] = nicId,
    ['Private Endpoint ID'] = id
| join kind=leftouter (
    resources
    | where type =~ 'microsoft.network/networkinterfaces'
    | extend ipConfigurations = properties.ipConfigurations
    | mv-expand ipConfig = ipConfigurations
    | project 
        nicId = id,
        ['Private IP'] = tostring(ipConfig.properties.privateIPAddress),
        ['IP Allocation'] = tostring(ipConfig.properties.privateIPAllocationMethod)
) on nicId
| project-away nicId1
| order by ['Private Endpoint'] asc
```

## 11. Private Endpoints by Tenant and Subscription

**For Testing in Resource Graph Explorer:**
```kusto
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

---

## Testing Instructions

1. Go to Azure Portal → Resource Graph Explorer
2. Select your subscriptions in the scope selector
3. Copy and paste the **"For Testing"** version of each query above
4. Click "Run query" to test
5. All parameter references have been removed for direct testing

## Simple Test Query (No Parameters)

Test this first to ensure you have private endpoints:

```kusto
resources
| where type =~ 'microsoft.network/privateendpoints'
| summarize Count = count()
```
