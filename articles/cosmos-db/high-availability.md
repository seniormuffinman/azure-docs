---
title: High availability in Azure Cosmos DB 
description: This article describes how Azure Cosmos DB provides high availability
author: markjbrown
ms.service: cosmos-db
ms.topic: conceptual
ms.date: 05/21/2019
ms.author: mjbrown
ms.reviewer: sngun

---

# High availability with Azure Cosmos DB

Azure Cosmos DB transparently replicates your data across all the Azure regions associated with your Cosmos account. Cosmos DB employs multiple layers of redundancy for your data as shown in the following image:

![Physical partitioning](./media/high-availability/cosmosdb-data-redundancy.png)

- The data within Cosmos containers is [horizontally partitioned](partitioning-overview.md).

- Within each region, every partition is protected by a replica-set with all writes replicated and durably committed by a majority of replicas. Replicas are distributed across as many as 10-20 fault domains.

- Each partition across all the regions is replicated. Each region contains all the data partitions of a Cosmos container and can accept writes and serve reads.  

If your Cosmos account is distributed across *N* Azure regions, there will be at least *N* x 4 copies of all your data. In addition to providing low latency data access and scaling write/read throughput across the regions associated with your Cosmos account, having more regions (higher *N*) further improves availability.  

## SLAs for availability

As a globally distributed database, Cosmos DB provides comprehensive SLAs that encompass throughput, latency at the 99th percentile, consistency, and high availability. The table below shows the guarantees for high availability provided by Cosmos DB for single and multi-region accounts. For high availability, always configure your Cosmos accounts to have multiple write regions.

|Operation type  | Single region |Multi-region (single region writes)|Multi-region (multi-region writes) |
|---------|---------|---------|-------|
|Writes    | 99.99    |99.99   |99.999|
|Reads     | 99.99    |99.999  |99.999|

> [!NOTE]
> In practice, the actual write availability for bounded staleness, session, consistent prefix and eventual consistency models is significantly higher than the published SLAs. The actual read availability for all consistency levels is significantly higher than the published SLAs.

## High availability with Cosmos DB in the event of regional outages

Regional outages aren't uncommon, and Azure Cosmos DB makes sure your database is always highly available. The following details capture Cosmos DB behavior during an outage, depending on your Cosmos account configuration:

- With Cosmos DB, before a write operation is acknowledged to the client, the data is durably committed by a quorum of replicas within the region that accepts the write operations.

- Multi-region accounts configured with multiple-write regions will be highly available for both writes and reads. Regional failovers are instantaneous and don't require any changes from the application.

- **Multi-region accounts with a single-write region (write region outage):** During a write region outage, these accounts will remain highly available for reads. However, for writes you must **“enable automatic failover”** on your Cosmos account to failover the impacted region to another region. The failover will occur in the order of region priority you’ve specified. When the impacted region is back online, the un-replicated data present in the impacted write region during the outage is made available through the [conflicts feed](how-to-manage-conflicts.md#read-from-conflict-feed). Applications can read the conflicts feed, resolve the conflicts based on the application-specific logic, and write the updated data back to the Cosmos container as appropriate. Once the previously impacted write region recovers, it becomes automatically available as a read region. You can invoke a manual failover and configure the impacted region as the write region. Again you can do a manual failover by using [Azure CLI or Azure portal](how-to-manage-database-account.md#manual-failover). There is **no data or availability loss** before, during or after the manual failover. Your application continues to be highly available. 

- **Multi-region accounts with a single-write region (read region outage):** During a read region outage, these accounts will remain highly available for reads and writes. The impacted region is automatically disconnected from the write region and will be marked offline. The [Cosmos DB SDKs](sql-api-sdk-dotnet.md) will redirect read calls to the next available region in the preferred region list. If none of the regions in the preferred region list is available, calls automatically fall back to the current write region. No changes are required in your application code to handle read region outage. Eventually, when the impacted region is back online, the previously impacted read region will automatically sync with the current write region and will be available again to serve read requests. Subsequent reads are redirected to the recovered region without requiring any changes to your application code. During both failover and rejoining of a previously failed region, read consistency guarantees continue to be honored by Cosmos DB.

- Single-region accounts may lose availability following a regional outage. It's always recommended to set up **at least two regions** (preferably, at least two write regions) with your Cosmos account to ensure high availability at all times.

- Even in an extremely rare and unfortunate event when the Azure region is permanently irrecoverable, there is no data loss if your multi-region Cosmos account is configured with the default consistency level of *Strong*. In the event of a permanently irrecoverable write region, for the multi-region Cosmos accounts configured with bounded-staleness consistency, the potential data loss window is restricted to the staleness window (*K* or *T*); for session, consistent-prefix and eventual consistency levels, the potential data loss window is restricted to a maximum of five seconds.

## Building highly available applications

- To ensure high write and read availability, configure your Cosmos account to span at least two regions with multiple-write regions. This configuration will provide the highest availability, lowest latency, and best scalability for both reads and writes backed by SLAs. To learn more, see how to [configure your Cosmos account with multiple write-regions](tutorial-global-distribution-sql-api.md).

- For multi-region Cosmos accounts that are configured with a single-write region, [enable automatic-failover by using Azure CLI or Azure portal](how-to-manage-database-account.md#automatic-failover). After you enable automatic failover, whenever there is a regional disaster, Cosmos DB will automatically failover your account.  

- Even if your Cosmos account is highly available, your application may not be correctly designed to remain highly available. To test the end-to-end high availability of your application, periodically invoke the [manual failover by using Azure CLI or Azure portal](how-to-manage-database-account.md#manual-failover), as a part of your application testing or disaster-recovery (DR) drills.

- Within a globally distributed database environment there is a direct relationship between the consistency level and data durability in the presence of a region-wide outage. As you develop your business continuity plan, you need to understand the maximum acceptable time before the application fully recovers after a disruptive event. The time required for an application to fully recover is known as recovery time objective (RTO). You also need to understand the maximum period of recent data updates the application can tolerate losing when recovering after a disruptive event. The time period of updates that you might afford to lose is known as recovery point objective (RPO). To see the RPO and RTO for Azure Cosmos DB, see [Consistency levels and data durability](consistency-levels-tradeoffs.md#rto)

## Next steps

Next you can read the following articles:

* [Availability and performance tradeoffs for various consistency levels](consistency-levels-tradeoffs.md)
* [Globally scaling provisioned throughput](scaling-throughput.md)
* [Global distribution - under the hood](global-dist-under-the-hood.md)
* [Consistency levels in Azure Cosmos DB](consistency-levels.md)
* [How to configure your Cosmos account with multiple write regions](how-to-multi-master.md)
