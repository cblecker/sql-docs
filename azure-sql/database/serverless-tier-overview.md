---
title: Serverless compute tier
description: This article describes the new serverless compute tier and compares it with the existing provisioned compute tier for Azure SQL Database.
author: oslake
ms.author: moslake
ms.reviewer: wiassaf, mathoma
ms.date: 04/25/2023
ms.service: sql-database
ms.subservice: service-overview
ms.topic: conceptual
ms.custom:
  - "test sqldbrb=1"
  - "devx-track-azurecli"
  - "devx-track-azurepowershell"
---
# Serverless compute tier for Azure SQL Database
[!INCLUDE[appliesto-sqldb](../includes/appliesto-sqldb.md)]

Serverless is a [compute tier](service-tiers-sql-database-vcore.md#compute) for single databases in Azure SQL Database that automatically scales compute based on workload demand and bills for the amount of compute used per second. The serverless compute tier also automatically pauses databases during inactive periods when only storage is billed and automatically resumes databases when activity returns. The serverless compute tier is available in the [General Purpose](service-tier-general-purpose.md) service tier and currently in preview in the [Hyperscale](service-tier-hyperscale.md) service tier. 

> [!NOTE]
> - Serverless in the Hyperscale service tier is currently in preview.
> - Auto-pausing and auto-resuming is currently only supported in the General Purpose service tier. 

## Overview

The serverless compute tier for single databases in Azure SQL Database is parameterized by a compute autoscaling range and an auto-pause delay. The configuration of these parameters shapes the database performance experience and compute cost.

![serverless billing](./media/serverless-tier-overview/serverless-billing.png)

### Performance configuration

- The **minimum vCores** and **maximum vCores** are configurable parameters that define the range of compute capacity available for the database. Memory and IO limits are proportional to the vCore range specified.  
- The **auto-pause delay** is a configurable parameter that defines the period of time the database must be inactive before it is automatically paused. The database is automatically resumed when the next login or other activity occurs.  Alternatively, automatic pausing can be disabled.

### Cost

- The cost for a serverless database is the summation of the compute cost and storage cost.
- When compute usage is between the min and max limits configured, the compute cost is based on vCore and memory used.
- When compute usage is below the min limits configured, the compute cost is based on the min vCores and min memory configured.
- When the database is paused, the compute cost is zero and only storage costs are incurred.
- The storage cost is determined in the same way as in the provisioned compute tier.

For more cost details, see [Billing](serverless-tier-overview.md#billing).

## Scenarios

Serverless is price-performance optimized for single databases with intermittent, unpredictable usage patterns that can afford some delay in compute warm-up after idle usage periods. In contrast, the [provisioned compute tier](service-tiers-sql-database-vcore.md#compute) is price-performance optimized for single databases or multiple databases in [elastic pools](elastic-pool-overview.md) with higher average usage that cannot afford any delay in compute warm-up.

### Scenarios well suited for serverless compute

- Single databases with intermittent, unpredictable usage patterns interspersed with periods of inactivity, and lower average compute utilization over time.
- Single databases in the provisioned compute tier that are frequently rescaled and customers who prefer to delegate compute rescaling to the service.
- New single databases without usage history where compute sizing is difficult or not possible to estimate prior to deployment in SQL Database.

### Scenarios well suited for provisioned compute

- Single databases with more regular, predictable usage patterns and higher average compute utilization over time.
- Databases that cannot tolerate performance trade-offs resulting from more frequent memory trimming or delays in resuming from a paused state.
- Multiple databases with intermittent, unpredictable usage patterns that can be consolidated into elastic pools for better price-performance optimization.

### Compare compute tiers

The following table summarizes distinctions between the serverless compute tier and the provisioned compute tier:

| | **Serverless compute** | **Provisioned compute** |
|:---|:---|:---|
|**Database usage pattern**| Intermittent, unpredictable usage with lower average compute utilization over time. | More regular usage patterns with higher average compute utilization over time, or multiple databases using elastic pools.|
| **Performance management effort** |Lower|Higher|
|**Compute scaling**|Automatic|Manual|
|**Compute responsiveness**|Lower after inactive periods|Immediate|
|**Billing granularity**|Per second|Per hour|

## Purchasing model and service tier


The following table describes serverless support based on purchasing model, service tiers and hardware:: 

| **Category** | **Supported** | **Not supported**|
|:---|:---|:---|
| **Purchasing model** | [vCore](service-tiers-vcore.md) | [DTU](service-tiers-dtu.md) |
| **Service tier** | [General Purpose](service-tier-general-purpose.md) <br/> [Hyperscale](service-tier-hyperscale.md) (in Preview) | Business Critical| 
| **Hardware** | Standard-series (Gen5) | All other hardware |  

## Autoscaling

### Scaling responsiveness

In general, serverless databases are run on a machine with sufficient capacity to satisfy resource demand without interruption for any amount of compute requested within limits set by the max vCores value. Occasionally, load balancing automatically occurs if the machine is unable to satisfy resource demand within a few minutes. For example, if the resource demand is 4 vCores, but only 2 vCores are available, then it may take up to a few minutes to load balance before 4 vCores are provided. The database remains online during load balancing except for a brief period at the end of the operation when connections are dropped.

### Memory management

In both the General Purpose and Hyperscale service tiers, memory for serverless databases is reclaimed more frequently than for provisioned compute databases. This behavior is important to control costs in serverless and can impact performance.


#### Cache reclamation

Unlike provisioned compute databases, memory from the SQL cache is reclaimed from a serverless database when CPU or active cache utilization is low.

- Active cache utilization is considered low when the total size of the most recently used cache entries falls below a threshold for a period of time.
- When cache reclamation is triggered, the target cache size is reduced incrementally to a fraction of its previous size and reclaiming only continues if usage remains low.
- When cache reclamation occurs, the policy for selecting cache entries to evict is the same selection policy as for provisioned compute databases when memory pressure is high.
- The cache size is never reduced below the min memory limit as defined by min vCores, that can be configured.

In both serverless and provisioned compute databases, cache entries may be evicted if all available memory is used.

When CPU utilization is low, active cache utilization can remain high depending on the usage pattern and prevent memory reclamation.  Also, there can be other delays after user activity stops before memory reclamation occurs due to periodic background processes responding to prior user activity.  For example, delete operations and Query Store cleanup tasks generate ghost records that are marked for deletion, but are not physically deleted until the ghost cleanup process runs. Ghost cleanup may involve reading additional data pages into cache.

#### Cache hydration

The SQL memory cache grows as data is fetched from disk in the same way and with the same speed as for provisioned databases. When the database is busy, the cache is allowed to grow unconstrained while there is available memory.


### <a name="disk-cache-mgmt"></a> Disk cache management

In the Hyperscale service tier for both serverless and provisioned compute tiers, each compute replica uses a Resilient Buffer Pool Extension (RBPEX) cache, which stores data pages on local SSD to improve IO performance.  However, in the serverless compute tier for Hyperscale, the RBPEX cache for each compute replica automatically grows and shrinks in response to increasing and decreasing workload demand.  The maximum size the RBPEX cache can grow to is three times the maximum memory configured for the database.  For details on maximum memory and RBPEX auto-scaling limits in serverless, see [serverless Hyperscale resource limits](resource-limits-vcore-single-databases.md#hyperscale---serverless-compute---gen-5).

## Auto-pausing and auto-resuming

Currently, serverless auto-pausing and auto-resuming are only supported in the General Purpose tier. 

### Auto-pausing

Auto-pausing is triggered if all of the following conditions are true for the duration of the auto-pause delay:

- Number of sessions = 0
- CPU = 0 for user workload running in the user resource pool

An option is provided to disable auto-pausing if desired.

The following features do not support auto-pausing, but do support auto-scaling. If any of the following features are used, then auto-pausing must be disabled and the database will remain online regardless of the duration of database inactivity:

- Geo-replication ([active geo-replication](active-geo-replication-overview.md) and [auto-failover groups](auto-failover-group-sql-db.md)).
- [Long-term backup retention](long-term-retention-overview.md) (LTR).
- The sync database used in [SQL Data Sync](sql-data-sync-data-sql-server-sql-database.md).  Unlike sync databases, hub and member databases support auto-pausing.
- [DNS alias](dns-alias-overview.md) created for the logical server containing a serverless database.
- [Elastic Jobs (preview)](elastic-jobs-overview.md), Auto-pause enabled serverless database is not supported as a *Job Database*. Serverless databases targeted by elastic jobs do support auto-pausing, and will be resumed by job connections.

Auto-pausing is temporarily prevented during the deployment of some service updates which require the database be online.  In such cases, auto-pausing becomes allowed again once the service update completes.

#### Auto-pause troubleshooting

If auto-pausing is enabled, but a database does not auto-pause after the delay period, and the features listed above are not used, the application or user sessions may be preventing auto-pausing. To see if there are any application or user sessions currently connected to the database, connect to the database using any client tool, and execute the following query:

```sql
SELECT session_id,
       host_name,
       program_name,
       client_interface_name,
       login_name,
       status,
       login_time,
       last_request_start_time,
       last_request_end_time
FROM sys.dm_exec_sessions AS s
INNER JOIN sys.dm_resource_governor_workload_groups AS wg
ON s.group_id = wg.group_id
WHERE s.session_id <> @@SPID
      AND
      (
      (
      wg.name like 'UserPrimaryGroup.DB%'
      AND
      TRY_CAST(RIGHT(wg.name, LEN(wg.name) - LEN('UserPrimaryGroup.DB') - 2) AS int) = DB_ID()
      )
      OR
      wg.name = 'DACGroup'
      );
```

> [!TIP]
> After running the query, make sure to disconnect from the database. Otherwise, the open session used by the query will prevent auto-pausing.

If the result set is non-empty, it indicates that there are sessions currently preventing auto-pausing. 

If the result set is empty, it is still possible that sessions were open, possibly for a short time, at some point earlier during the auto-pause delay period. To see if such activity has occurred during the delay period, you can use [Azure SQL Auditing](auditing-overview.md) and examine audit data for the relevant period.

The presence of open sessions, with or without concurrent CPU utilization in the user resource pool, is the most common reason for a serverless database to not auto-pause as expected.

### Auto-resuming

Auto-resuming is triggered if any of the following conditions are true at any time:

|Feature|Auto-resume trigger|
|---|---|
|Authentication and authorization|Login|
|Threat detection|Enabling/disabling threat detection settings at the database or server level.<br>Modifying threat detection settings at the database or server level.|
|Data discovery and classification|Adding, modifying, deleting, or viewing sensitivity labels|
|Auditing|Viewing auditing records.<br>Updating or viewing auditing policy.|
|Data masking|Adding, modifying, deleting, or viewing data masking rules|
|Transparent data encryption|Viewing state or status of transparent data encryption|
|Vulnerability assessment|Ad hoc scans and periodic scans if enabled|
|Query (performance) data store|Modifying or viewing query store settings|
|Performance recommendations|Viewing or applying performance recommendations|
|Auto-tuning|Application and verification of auto-tuning recommendations such as auto-indexing|
|Database copying|Create database as copy.<br>Export to a BACPAC file.|
|SQL data sync|Synchronization between hub and member databases that run on a configurable schedule or are performed manually|
|Modifying certain database metadata|Adding new database tags.<br>Changing max vCores, min vCores, or auto-pause delay.|
|SQL Server Management Studio (SSMS)|Using SSMS versions earlier than 18.1 and opening a new query window for any database in the server will resume any auto-paused database in the same server. This behavior does not occur if using SSMS version 18.1 or later.|

Monitoring, management, or other solutions performing any of the operations listed above will trigger auto-resuming.

Auto-resuming is also triggered during the deployment of some service updates that require the database be online.

### Connectivity

If a serverless database is paused, then the first login will resume the database and return an error stating that the database is unavailable with error code 40613. Once the database is resumed, the login must be retried to establish connectivity. Database clients with connection retry logic should not need to be modified.  For connection retry logic options that are built-in to the SqlClient driver, see [configurable retry logic in SqlClient](/sql/connect/ado-net/configurable-retry-logic).

### Latency

The latency to auto-resume and auto-pause a serverless database is generally order of 1 minute to auto-resume and 1-10 minutes after the expiration of the delay period to auto-pause.

### Customer managed transparent data encryption (BYOK)

#### Key deletion or revocation

If using [customer managed transparent data encryption](transparent-data-encryption-byok-overview.md) (BYOK) and the serverless database is auto-paused when key deletion or revocation occurs, then the database remains in the auto-paused state.  In this case, after the database is next resumed, the database becomes inaccessible within approximately 10 minutes. Once the database becomes inaccessible, the recovery process is the same as for provisioned compute databases. If the serverless database is online when key deletion or revocation occurs, then the database also becomes inaccessible within approximately 10 minutes in the same way as with provisioned compute databases.

#### Key rotation

If using [customer managed transparent data encryption](transparent-data-encryption-byok-overview.md) (BYOK) and the serverless database is auto-paused, then automated key rotation is deferred until the database is auto-resumed.

## <a name="create-serverless-db"></a> Create a new serverless database 

Creating a new database or moving an existing database into a serverless compute tier follows the same pattern as creating a new database in provisioned compute tier and involves the following two steps:

1. Specify the service objective. The service objective prescribes the service tier, hardware configuration, and max vCores. For service objective options, see [serverless resource limits](resource-limits-vcore-single-databases.md#general-purpose---serverless-compute---gen5)


2. Optionally, specify the min vCores and auto-pause delay to change their default values. The following table shows the available values for these parameters. 

   |Parameter|Value choices|Default value|
   |---|---|---|---|
   |Min vCores|Depends on max vCores configured - see [resource limits](resource-limits-vcore-single-databases.md#general-purpose---serverless-compute---gen5).|0.5 vCores|
   |Autopause delay|Minimum: 60 minutes (1 hour)<br>Maximum: 10080 minutes (7 days)<br>Increments: 10 minutes<br>Disable autopause: -1|60 minutes|

The following examples create a new database in the serverless compute tier.

#### Use Azure portal

See [Quickstart: Create a single database in Azure SQL Database using the Azure portal](single-database-create-quickstart.md).


#### Use PowerShell

# [General Purpose](#tab/general-purpose)

Create a new serverless General Purpose database with the following PowerShell example: 

```powershell
New-AzSqlDatabase -ResourceGroupName $resourceGroupName -ServerName $serverName -DatabaseName $databaseName `
  -Edition GeneralPurpose -ComputeModel Serverless -ComputeGeneration Gen5 `
  -MinVcore 0.5 -MaxVcore 2 -AutoPauseDelayInMinutes 720
```

# [Hyperscale](#tab/hyperscale)

Create a new serverless Hyperscale database with the following PowerShell example: 

```powershell
New-AzSqlDatabase -ResourceGroupName $resourceGroupName -ServerName $serverName -DatabaseName $databaseName ` 
  -Edition Hyperscale -ComputeModel Serverless -ComputeGeneration Gen5 ` 
  -MinVcore 0.5 -MaxVcore 2
```

Create a new serverless Hyperscale database with 1 high availability replica and zone redundancy using the following PowerShell example:

```powershell
New-AzSqlDatabase -ResourceGroupName $resourceGroupName -ServerName $serverName -DatabaseName $databaseName `
  -Edition Hyperscale -ComputeModel Serverless -ComputeGeneration Gen5 `
  -MinVcore 0.5 -MaxVcore 2 `
  -HighAvailabilityReplicaCount 1 -BackupStorageRedundancy Zone -ZoneRedundant
```

---

#### Use Azure CLI

# [General Purpose](#tab/general-purpose)

Create a new serverless General Purpose database with the following Azure CLI example: 

```azurecli
az sql db create -g $resourceGroupName -s $serverName -n $databaseName `
  -e GeneralPurpose --compute-model Serverless -f Gen5 `
  --min-capacity 0.5 -c 2 --auto-pause-delay 720
```

# [Hyperscale](#tab/hyperscale)

Create a new serverless Hyperscale database with the following Azure CLI example: 

```azurecli
az sql db create -g $resourceGroupName -s $serverName -n $databaseName ` 
  -e Hyperscale --compute-model Serverless -f Gen5 `
  --min-capacity 0.5 -c 2 
```

Create a new serverless Hyperscale database with 1 high availability replica and zone redundancy using the following Azure CLI example:

```azurecli
az sql db create -g $resourceGroupName -s $serverName -n $databaseName `
  -e Hyperscale --compute-model Serverless -f Gen5 `
  --min-capacity 0.5 -c 2 `
  --ha-replicas 1 --backup-storage-redundancy Zone --zone-redundant
```

---


#### Use Transact-SQL (T-SQL)

When using T-SQL, default values are applied for the min vcores and autopause delay. They can later be changed from the portal or via other management APIs (PowerShell, Azure CLI, REST API).

For details, see [CREATE DATABASE](/sql/t-sql/statements/create-database-transact-sql?view=azuresqldb-current&preserve-view=true).  

# [General Purpose](#tab/general-purpose)

Create a new General Purpose serverless database with the following T-SQL example: 

```sql
CREATE DATABASE testdb
( EDITION = 'GeneralPurpose', SERVICE_OBJECTIVE = 'GP_S_Gen5_1' ) ;
```

# [Hyperscale](#tab/hyperscale)

Create a new Hyperscale serverless database with the following T-SQL example: 

```sql
ALTER DATABASE testdb  
MODIFY ( SERVICE_OBJECTIVE = 'HS_S_Gen5_2') ; 
```

---



## Move a database between compute tiers

It's possible to move your database from the provisioned compute tier to the serverless compute tier, and back again. 

> [!NOTE]
> It's also possible to upgrade your database in the General Purpose tier to the Hyperscale tier. Review [Manage Hyperscale databases](manage-hyperscale-database.md#migrate-an-existing-database-to-hyperscale) to learn more. 

When moving your database between compute tiers, provide the **Compute model** parameter as either `Serverless` or `Provisioned` when using PowerShell and the Azure CLI, and the compute size for the  **SERVICE_OBJECTIVE** when using T-SQL. Review [resource limits](resource-limits-vcore-single-databases.md) to identify your appropriate compute size. 

The examples in this section show you how to move your provisioned database to serverless. Modify the service objective as needed, as these examples set the max vCores to 4. 


#### Use PowerShell

# [General Purpose](#tab/general-purpose)

Move a provisioned compute General Purpose database to the serverless compute tier with the following PowerShell example: 

```powershell
Set-AzSqlDatabase -ResourceGroupName $resourceGroupName -ServerName $serverName -DatabaseName $databaseName `
  -Edition GeneralPurpose -ComputeModel Serverless -ComputeGeneration Gen5 `
  -MinVcore 1 -MaxVcore 4 -AutoPauseDelayInMinutes 1440
```

# [Hyperscale](#tab/hyperscale)

Move a provisioned compute Hyperscale database to the serverless compute tier with the following PowerShell example: 

```powershell
Set-AzSqlDatabase -ResourceGroupName $resourceGroupName -ServerName $serverName -DatabaseName $databaseName ` 
  -Edition Hyperscale -ComputeModel Serverless -ComputeGeneration Gen5 ` 
  -MinVcore 1 -MaxVcore 4 
```

---

#### Use Azure CLI

# [General Purpose](#tab/general-purpose)

Move a provisioned compute General Purpose database to the serverless compute tier with the following Azure CLI example: 

```azurecli
az sql db update -g $resourceGroupName -s $serverName -n $databaseName `
  --edition GeneralPurpose --compute-model Serverless --family Gen5 `
  --min-capacity 1 --capacity 4 --auto-pause-delay 1440
```

# [Hyperscale](#tab/hyperscale)

Move a provisioned compute Hyperscale database to the serverless compute tier with the following Azure CLI example: 

```azurecli
az sql db update -g $resourceGroupName -s $serverName -n $databaseName ` 
  --edition Hyperscale --compute-model Serverless --family Gen5 `
  --min-capacity 1 --capacity 4 
```

---

#### Use Transact-SQL (T-SQL)

When using T-SQL, default values are applied for the min vcores and auto-pause delay. They can later be changed from the portal or via other management APIs (PowerShell, Azure CLI, REST API). For details, see [ALTER DATABASE](/sql/t-sql/statements/alter-database-transact-sql?view=azuresqldb-current&preserve-view=true).

# [General Purpose](#tab/general-purpose)

Move a provisioned compute General Purpose database to the serverless compute tier with the following T-SQL example: 

```sql
ALTER DATABASE testdb 
MODIFY ( SERVICE_OBJECTIVE = 'GP_S_Gen5_1') ;
```

# [Hyperscale](#tab/hyperscale)

Move a provisioned compute Hyperscale database to the serverless compute tier with the following T-SQL example: 

```sql
ALTER DATABASE testdb  
MODIFY ( SERVICE_OBJECTIVE = 'HS_S_Gen5_2') ; 
```

---

## Modify serverless configuration

### Use PowerShell

Use [Set-AzSqlDatabase](/powershell/module/az.sql/set-azsqldatabase) to modify the maximum or minimum vCores, and autopause delay. Use the `MaxVcore`, `MinVcore`, and `AutoPauseDelayInMinutes` arguments.  Serverless auto-pausing is not currently supported in the Hyperscale tier, so the auto-pause delay argument is only applicable to the General Purpose tier. 

### Use Azure CLI

Use [az sql db update](/cli/azure/sql/db#az-sql-db-update) to modify the maximum or minimum vCores, and autopause delay. Use the `capacity`, `min-capacity`, and `auto-pause-delay` arguments. Serverless auto-pausing is not currently supported in the Hyperscale tier, so the auto-pause delay argument is only applicable to the General Purpose tier. 


## Monitoring

### Resources used and billed

The resources of a serverless database are encapsulated by app package, SQL instance, and user resource pool entities.

#### App package

The app package is the outer most resource management boundary for a database, regardless of whether the database is in a serverless or provisioned compute tier. The app package contains the SQL instance and external services like Full-text Search that all together scope all user and system resources used by a database in SQL Database. The SQL instance generally dominates the overall resource utilization across the app package.

#### User resource pool

The user resource pool is an inner resource management boundary for a database, regardless of whether the database is in a serverless or provisioned compute tier. The user resource pool scopes CPU and IO for user workload generated by DDL queries such as CREATE and ALTER, DML queries such as INSERT, UPDATE, DELETE, and MERGE, and SELECT queries. These queries generally represent the most substantial proportion of utilization within the app package.

### Metrics

Metrics for monitoring the resource usage of the app package and user resource pool of a serverless database and any geo-replicas are listed in the following table:

|Entity|Metric|Description|Units|
|---|---|---|---|
|App package|app_cpu_percent|Percentage of vCores used by the app relative to max vCores allowed for the app. For serverless Hyperscale, this metric is exposed for all primary replicas, named replicas, and geo-replicas. |Percentage|
|App package|app_cpu_billed|The amount of compute billed for the app during the reporting period. The amount paid during this period is the product of this metric and the vCore unit price. <br><br>Values of this metric are determined by aggregating over time the maximum of CPU used and memory used each second. If the amount used is less than the minimum amount provisioned as set by the min vCores and min memory, then the minimum amount provisioned is billed. In order to compare CPU with memory for billing purposes, memory is normalized into units of vCores by rescaling the amount of memory in GB by 3 GB per vCore. For serverless Hyperscale, this metric is exposed for the primary replica and any named replicas. |vCore seconds|
|App package| app_cpu_billed_HA_replicas| Only applicable to serverless Hyperscale.  Sum of the compute billed across all apps for HA replicas during the reporting period.  This sum is scoped either to the HA replicas belonging to the primary replica or the HA replicas belonging to a given named replica.  Before calculating this sum across HA replicas, the amount of compute billed for an individual HA replica is determined in the same way as for the primary replica or a named replica.  For serverless Hyperscale, this metric is exposed for all primary replicas, named replicas, and geo-replicas.  The amount paid during the reporting period is the product of this metric and the vCore unit price.  |vCore seconds| 
|App package|app_memory_percent|Percentage of memory used by the app relative to max memory allowed for the app. For serverless Hyperscale, this metric is exposed for all primary replicas, named replicas, and geo-replicas. |Percentage|
|User resource pool|cpu_percent|Percentage of vCores used by user workload relative to max vCores allowed for user workload. |Percentage|
|User resource pool|data_IO_percent|Percentage of data IOPS used by user workload relative to max data IOPS allowed for user workload.|Percentage|
|User resource pool|log_IO_percent|Percentage of log MB/s used by user workload relative to max log MB/s allowed for user workload.|Percentage|
|User resource pool|workers_percent|Percentage of workers used by user workload relative to max workers allowed for user workload.|Percentage|
|User resource pool|sessions_percent|Percentage of sessions used by user workload relative to max sessions allowed for user workload.|Percentage|

### Pause and resume status

In the Azure portal, the database status is displayed in the overview pane of the server that lists the databases it contains. The database status is also displayed in the overview pane for the database.

Using the following commands to query the pause and resume status of a database:

#### Use PowerShell

```powershell
Get-AzSqlDatabase -ResourceGroupName $resourcegroupname -ServerName $servername -DatabaseName $databasename `
  | Select -ExpandProperty "Status"
```

#### Use Azure CLI

```azurecli
az sql db show --name $databasename --resource-group $resourcegroupname --server $servername --query 'status' -o json
```

## Resource limits

For resource limits, see [serverless compute tier](resource-limits-vcore-single-databases.md#general-purpose---serverless-compute---gen5).

## Billing

The amount of compute billed for a serverless database is the maximum of CPU used and memory used each second. If the amount of CPU and memory used is less than the minimum amount provisioned for each resource, then the provisioned amount is billed. In order to compare CPU with memory for billing purposes, memory is normalized into units of vCores by rescaling the number of  GB by 3 GB per vCore.

- **Resource billed**: CPU and memory
- **Amount billed**: vCore unit price * max (min vCores, vCores used, min memory GB * 1/3, memory GB used * 1/3) 
- **Billing frequency**: Per second

The vCore unit price is the cost per vCore per second. For Hyperscale, the vCore unit price for an HA replica or named replica is lower than for the primary replica. 

Refer to the [Azure SQL Database pricing page](https://azure.microsoft.com/pricing/details/sql-database/single/) for specific unit prices in a given region.

The amount of compute billed in serverless for a General Purpose database, or a Hyperscale primary or named replica is exposed by the following metric: 

- **Metric**: app_cpu_billed (vCore seconds)
- **Definition**: max (min vCores, vCores used, min memory GB * 1/3, memory GB used * 1/3)
- **Reporting frequency**: Per minute based on per second measurements aggregated over 1 minute.

The amount of compute billed in serverless for Hyperscale HA replicas belonging to the primary replica or any named replica is exposed by the following metric: 

- **Metric**: app_cpu_billed_HA_replicas (vCore seconds) 
- **Definition**: Sum of max (min vCores, vCores used, min memory GB * 1/3, memory GB used * 1/3) for any HA replicas belonging to their parent resource.
- **Parent resource and metric endpoint**: The primary replica and any named replica each separately expose this metric which measures the compute billed for any associated HA replicas.  
- **Reporting frequency**: Per minute based on per second measurements aggregated over 1 minute. 


### Minimum compute bill

If a serverless database is paused, then the compute bill is zero.  If a serverless database is not paused, then the minimum compute bill is no less than the amount of vCores based on max (min vCores, min memory GB * 1/3).

Examples:

- Suppose a serverless database in the General Purpose tier is not paused and configured with 8 max vCores and 1 min vCore corresponding to 3.0 GB min memory.  Then the minimum compute bill is based on max (1 vCore, 3.0 GB * 1 vCore / 3 GB) = 1 vCore.
- Suppose a serverless database in the General Purpose tier is not paused and configured with 4 max vCores and 0.5 min vCores corresponding to 2.1 GB min memory.  Then the minimum compute bill is based on max (0.5 vCores, 2.1 GB * 1 vCore / 3 GB) = 0.7 vCores.
- Suppose a serverless database in the Hyperscale tier has a primary replica with one HA replica and one named replica with no HA replicas.  Suppose each replica is configured with 8 max vCores and 1 min vCore corresponding to 3 GB min memory. Then the minimum compute bill for the primary replica, HA replica, and named replica are each based on max (1 vCore, 3 GB * 1 vCore / 3 GB) = 1 vCore. 

The [Azure SQL Database pricing calculator](https://azure.microsoft.com/pricing/calculator/?service=sql-database) for serverless can be used to determine the min memory configurable based on the number of max and min vCores configured.  As a rule, if the min vCores configured is greater than 0.5 vCores, then the minimum compute bill is independent of the min memory configured and based only on the number of min vCores configured.

### <a name="scenario-examples"></a> Scenario examples 

# [General Purpose](#tab/general-purpose)

Consider a serverless database in the General Purpose tier configured with 1 min vCore and 4 max vCores.  This configuration corresponds to around 3 GB min memory and 12 GB max memory.  Suppose the auto-pause delay is set to 6 hours and the database workload is active during the first 2 hours of a 24-hour period and otherwise inactive.    

In this case, the database is billed for compute and storage during the first 8 hours.  Even though the database is inactive starting after the second hour, it is still billed for compute in the subsequent 6 hours based on the minimum compute provisioned while the database is online.  Only storage is billed during the remainder of the 24-hour period while the database is paused.

More precisely, the compute bill in this example is calculated as follows:

|Time Interval|vCores used each second|GB used each second|Compute dimension billed|vCore seconds billed over time interval|
|---|---|---|---|---|
|0:00-1:00|4|9|vCores used|4 vCores * 3600 seconds = 14400 vCore seconds|
|1:00-2:00|1|12|Memory used|12 GB * 1/3 * 3600 seconds = 14400 vCore seconds|
|2:00-8:00|0|0|Min memory provisioned|3 GB * 1/3 * 21600 seconds = 21600 vCore seconds|
|8:00-24:00|0|0|No compute billed while paused|0 vCore seconds|
|Total vCore seconds billed over 24 hours||||50400 vCore seconds|

Suppose the compute unit price is $0.000145/vCore/second.  Then the compute billed for this 24-hour period is the product of the compute unit price and vCore seconds billed: $0.000145/vCore/second * 50400 vCore seconds ~ $7.31.



# [Hyperscale](#tab/hyperscale)

Consider a serverless database in the Hyperscale tier configured with 1 min vCore and 8 max vCores.  Suppose that the primary replica has enabled one HA replica and that a named replica with 1 min vCore and 8 max vCores has also been provisioned.  For each replica, this configuration corresponds to 3 GB min memory and 24 GB max memory.  Further suppose that write workload occurs throughout a 24-hour period, but that read-only workload occurs just during the first 8 hours of this time period. 

In this example, the compute billed for the database is summation of the compute billed for each replica and calculated as follows based on the usage pattern described in the tables below: 

**Primary replica**

| Time Interval    | vCores used each second    | GB used each second    | Compute dimension billed | vCore seconds billed over time interval | 
|---|---|---|---|---|
|0:00-2:00 | 8    | 15 |    vCores used    | 8 vCores * 7200 seconds = 57600 vCore seconds |
|2:00-14:00 |    1.5    | 6     | Memory used |    6 GB * 1/3 * 43200 seconds = 86400 vCore seconds |
|14:00-24:00 |    0.5    | 2     | Min vCores provisioned    | 1 vCore * 36000 seconds = 36000 vCore seconds | 
|**Total vCore seconds billed over 24 hours** |||| 180000 vCore seconds |

Suppose the compute unit price for the primary replica is $0.000163/vCore/second. Then the compute billed for the primary replica over this 24-hour period is the product of the compute unit price and vCore seconds billed: $0.000163/vCore/second * 180000 vCore seconds ~ $29.34.

**HA replica**

|Time Interval    | vCores used each second    | GB used each second    | Compute dimension billed    | vCore seconds billed over time interval |
|---|---|---|---|---|
|0:00-2:00 |    8 |    9    | vCores used    | 8 vCores * 7200 seconds = 57600 vCore seconds |
| 2:00-8:00    | 1.5     | 3    | Memory used    | 3 GB * 1/3 * 43200 seconds = 43200 vCore seconds|
|8:00-24:00|    0|    2    |Min memory provisioned    |3 GB * 1/3 * 36000 seconds = 36000 vCore seconds|
|Total vCore seconds billed over 24 hours||||136800 vCore seconds |

Suppose the compute unit price for an HA replica is $0.000105/vCore/second. Then the compute billed for the HA replica over this 24-hour period is $0.000105/vCore/second * 136800 vCore seconds ~ $14.36.

**Named replica** 

Similarly for the named replica, suppose the total vCore seconds billed over 24 hours is 150000 vCore seconds and that the compute unit price for a named replica is $0.000105/vCore/second. Then the compute billed for the named replica over this time period is $0.000105/vCore/second * 150000 vCore seconds ~ $15.75.

**Total compute cost**

Therefore, the total compute bill for all three replicas of the database is around $29.34 + ~ $14.36 + $15.75 = $59.45.

---


## Azure Hybrid Benefit and reserved capacity

Azure Hybrid Benefit (AHB) and reserved capacity discounts do not apply to the serverless compute tier.

## Available regions

The serverless compute tier with support up to 40 max vCores is available worldwide except the following regions: China East, China North, Germany Central, Germany Northeast, and US Gov Central (Iowa).

### Regions supporting 80 max vCores

Currently, 80 max vCores in serverless is supported in the following regions with more regions planned: Australia East, Australia Southeast, Canada Central, Central US, East Asia, East US, East US 2, France Central, France South, India Central, Japan East, Japan West, North Central US, North Europe, Norway East, Qatar Central, South Africa North, South Central US, Switzerland North, UK South, UK West, West Europe, West US, West US 2, and West US 3.

### Regions supporting availability zones for 80 max vCores

Currently, 80 max vCores in serverless with availability zone support is limited to the following regions with more regions planned: East US, West Europe, West US 2, and West US 3.

## Next steps

- To get started, see [Quickstart: Create a single database in Azure SQL Database using the Azure portal](single-database-create-quickstart.md).
- For serverless service tier choices, see [General Purpose](service-tier-general-purpose.md) and [Hyperscale (preview)](service-tier-hyperscale.md). 
