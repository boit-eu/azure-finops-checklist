# azure-finops-checklist
Here you find a Quick Checklist for Azure Cost Optimization.

## General
Here are some general suggestions for your cloud cost optimization.

1. Commit to Reservations & Savings Plans
* Azure Reservations (RIs): Best for static, predictable resources (e.g., a SQL database or a Domain Controller running 24/7). Savings: Up to 72%.
* Azure Savings Plans: Best for dynamic environments where compute usage fluctuates but a "baseline" spend exists. It is more flexible than RIs (applies across regions and instance families). Savings: Up to 65%.

2. Leverage Azure Hybrid Benefit
* Allow customers to use their existing on-premises Windows Server and SQL Server licenses (with Software Assurance) in Azure.
* Can save up to 40% on VMs and up to 55% on SQL Database. When combined with Reservations (Point 1), savings can hit 80%.

3. Eliminate "Orphaned" Resources
* Unattached Managed Disks: Leftover when a VM is deleted but the disk wasn't.
* Public IPs: Unassociated with any NIC.
* Network Interfaces (NICs): Leftover from deleted VMs.
* App Service Plans: Empty plans with no apps running.

4. Right-Sizing & Auto-Scaling
* Static (Right-Sizing): Downsize VMs or SQL Databases that consistently run at <20% CPU utilization.
* Dynamic (Auto-Scaling): Implement Virtual Machine Scale Sets (VMSS) or Standard/Premium App Service Plans to automatically add/remove instances based on demand (e.g., CPU > 70%).

5. Storage Lifecycle Management
* Configure **Lifecycle Management Policies**
* Hot Tier: For active data (accessed daily).
* Cool Tier: For data not accessed for 30 days (lower storage cost, higher access cost).
* Archive Tier: For backups/compliance data not accessed for 180+ days (lowest cost, long retrieval time).
* Can reduce storage costs by 50-80% for data-heavy clients (backups, logs, media).

## Find orphaned resources with KQL
Navigate to the **Azure Resource Graph Explorer** and run the queries.

### Orphaned disks
```kql
resources
| where type =~ 'microsoft.compute/disks'
| where properties.diskState == 'Unattached'
// Filter out disks that are arguably "intentional" (optional, usually not needed)
| where name !contains "ASR" // Exclude Azure Site Recovery disks if necessary
| project 
    DiskName = name, 
    ResourceGroup = resourceGroup, 
    Location = location, 
    SizeGB = toint(properties.diskSizeGB), 
    Tier = sku.name, 
    DiskState = properties.diskState, 
    SubscriptionId = subscriptionId,
    CreatedTime = properties.timeCreated
| order by SizeGB desc
```

### Orphaned public IP addresses
```kql
resources
| where type =~ 'microsoft.network/publicipaddresses'
| where properties.ipConfiguration == "" or isnull(properties.ipConfiguration)
| project 
    IPName = name, 
    ResourceGroup = resourceGroup, 
    Location = location, 
    SKU = sku.name, 
    IPAddress = properties.ipAddress, 
    SubscriptionId = subscriptionId
```

## Check long running VMs without Resource Reservation
Check with Azure Cost Management

1. Go to Cost Management + Billing -> Cost Analysis.
2. In the view settings (top), change "Actual Cost" to Amortized Cost.
3. Group By: Pricing Model (This is the magic filter).
4. Add Filter: Resource Type = virtual machines.
5. Add Filter: Pricing Model = On Demand.
