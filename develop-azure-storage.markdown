---
layout: page
title: Develop for Azure Storage
permalink: /develop-azure-storage/
---

## (10 - 15%)

### Develop Solutions using Cosmos DB storage
An Azure Cosmos container is the unit of scalability both for provisioned throughput and storage. A container is horizontally particioned, then replicated across multiple regions. The items added to a Cosmos container, and the throughput provisioned on it, are automatically distributed across a set of logical partitions based on partition key.
- Cosmos DB account > dbs > containers/collections > documents/items
- Select the appropriate API for your solution
  - SQL API: returns data in JSON format.
  - Table API: returns data as a table, structured in rows and columns.
  - More API specific entities for Azure Cosmos container:
  
  |Azure Cosmos Entity| SQL API |Cassandra API|Azure Cosmos DB API for MongoDB|Gremlin API|Table API|
  :-------:|:-------:|:----:|:---:|:---:|:---:|
  |Azure Cosmos Container|Container|Table|Collection|Graph|Table|

- Implement partitioning schemes
  - Azure Cosmos container can scale elastically, using either of the below throughput modes.
  - **Dedicated provisioned throughput mode**: Throughput provisioned on a container is exclusively reserved for that container and is backed by SLAs.
  - **Shared provisioned throughput mode**: Containers share the provisioned throughput with other container in the same database (excluding containers configured with dedicated provisioned throughput on the database). Other words, the provisioned throughput on the databse is shared among all the "shared throughput" containers.
  - Partitioning is used to scale individual containers in a db to meet performance needs.
  - Items in a container are divided into distinct subsets called *logical partitions*.
    - All items in a logical partition have the same partition key value, t.
  - Azure Cosmos DB uses hash-based partitioning to spread logical parititons across physical partitions. Azure Cosmos DB hashes the parition key value of an item. The hashed result determines the physical partition. Azure Cosmos DB allocates the key space of partition key hashes evenly across the physical partitions.
  - For **all** containers, your partition key should:
    - Be a property with a value which **does note** change
    - Have high cardinality
    - Spread request unit (RU) consumption and data storage evenly across all logical partitions, ensuring even distribution also across physical partitions.
- Interact with data using the appropriate SDK
  - Access the **first item in the Items container** of the AZ Cosmos DB:

  ```
  static void QueryDocuments(Uri uri, string accessKey)
  {
    // DocumentClient class represents connection to a Cosmos DB account
    DocumentClient dc = new DocumentClient(uri, accessKey);

    // CreateDatabaseQuery method creates a query returning the dbs in the account.
    // In this case, it returns the Tasks db
    var tasks = dc.CreateDatabaseQuery()
        .Where(d => d.Id == "Tasks").AsEnumerable().First();
    // CreateDocumentCollectionQuery method creates a query returning containers, or collections, in the db.
    // Returns one collection named "Items"
    var items = dc.CreateDocumentCollectionQuery(tasks.SelfLink)
        .Where(d => d.Id == "Items").AsEnumberable().First();
    // CreateDocumentQuery method creates a query returning the documents in a collection
    var item = dc.CreateDocumentQuery(items.SelfLink).AsEnumerable().First();
  }
  ```

- Set the appropriate consistency level for operations
  - All consistency levels are region-agnostic. And are therefore guaranteed for all operations, regardless of region settings.
  - **Stronger(more latency, less availability) > Weaker consistency(lower latency, more availability)**: Strong > Bounded Staleness > Session > Consistent Prefix > Eventual
  - **Strong**: Always returns the most recent committed version of an item. Ensures clients never receive a partial write, or uncommitted data during a read operation.
  - **Bounded staleness**: reads might lag behind writes based on configured updated versions (K) or time interval (t).
  - **Session**: *Most widely used consistency level*. Reads honor the consistent prefix level. This level does not prevent a client from receiving partial writes.
  - **Consistent prefix**: Ensures reads never see out of order writes. But does not prevent reads from seeing partial writes.
  - **Eventual**: Lowest latency, higher availability for data reads. Does not guarantee ordering of reads. Client may read values older than ones read previously. Is not an issue for applications not requiring a specific order (ex. count of Retweets, Likes). Clients can potentially receive a partial write, or uncommitted data during a read operation.
- Create Cosmos DB containers
- Implement scaling (partitions, containers)
- Implement server-side programming including stored procedures, triggers, and change feed notifications
  - Writing stored procedures, triggers, and user-defined functions (UDFs) in JS allows building of rich applications with the following advantages:
    - **Procedural logic**: JS is a high-level programming language providing rich and familiar interface to express business logic.
    - **Atomic transactions**: Operations performed within a single stored procedure or trigger are stomic. Atomic functionality lets an application combine related operations into a single batch, such that all operations succeed or none at all.
    - **Performance**: JSON data is intrinsically mapped to the JS language type system. Mapping allows a number of optimizations including lazy materialization of JSON documents in the buffer pool, and making them available on-demand to the executing code. Other performance benefits of shipping business logic to the db, including: 
    - *Batching*: group operations like inserts and submit them in bulk, this reducing network traffic latency costs and store overhead.
    - *Pre-compilation*: Stored procedures, triggers, and UDFs are implicitly pre-compiled to the byte code format in order to avoid compilation cost at the time of each script invocation.
    - *Sequencing*: Sometimes operations need a triggering mechanism to perform one or additional updates to the data.
    - **Encapsulation**: Stored procedures can be used to group logic in one place. Encapsulation adds an abstraction layer on top of the data, enabling you to evolve your applications independently from the data. This layer of abstraction is helpful when the data is schema-less, and you don't have to manage adding additional logic directly into your application. Abstraction keeps the data secure by streamlining access from the scripts.

### Develop solutions using blob storage

- Move items in Blob storage between storage accounts or containers
  - (Blob > Container > Storage account)
  - **AzCopy command**: command-line tool to move data into and out of storage in Azure. It includes the ability for asynchronous copy between blob storage accounts, specifying the source and destination accounts when running the command. AzCopy copies the blob, while leaving the original blob in place. Supports restart at point of failure, if a copy operation fails.
  - **az storage blob** Azure CLI command: supports asynchronous copying between storage accounts, but does NOT support restart at point of failure, if a copy operation fails.
  - **Start-AzureStorageBlobCopy** PowerShell cmdlet: copies blob between containers, NOT between storage accounts
  - **Start-AzureStorageBlobIncrementalCopy** PowerShell cmdlet: copy data from a Page blob snapshot to a specified destination Page blob.
  - Must define default request option for the location mode, in order for PrimaryThenSecondary mode to be used. If default request location mode is ommitted, PrimaryOnly mode will be used.
  - blobClient.DefaultRequestOptions.LocationMode options: **PrimaryOnly** (the default blob client mode), **PrimaryThenSecondary**, **SecondaryThenPrimary**, **SecondaryOnly**.
  - RA-GRS or RA-GZRS: read access geo-zone-redundant storage, a replication type.
- Set and retrieve properties and metadata
  - Blob storage account URLs have the following format,
    `https://{storage account name}.blob.core.windows.net/{container}/{blob}`
  - Retrive a blob snapshot, using the snapshot query string,
    `https://company1.blob.core.windows.net/books/azure.pdf?snapshot=2019-03-12T11:26:09:9360000Z`
- Interact with data using the appropriate SDK
- Implement data archiving and retention

  - To implement data archiving in Azure Blob Storage using access tiers and storage lifecycle, the Azure Storage Account must be upgraded to **General Purpose v2 (GPv2)**. This can also be done by converting an existing GPv1 to GPv2, through the Azure portal.
  - Blob leasing: prevents blob from being deleted or overwritten. Example: prevent the 2019.pdf blob from being deleted or overwritten for one minute (60 seconds), using comp query string set to lease,

  ```
  PUT https://company1.blob.core.windows.net/taxreturns/2019.pdf?comp=lease HTTP/1.1

  x-ms-version: 2018-03-28
  x-ms-lease-action: acquire
  x-ms-lease-duration: 60
  x-ms-proposed-lease-id: 18f12-b342bbdsf-sfdsfgfdg45-dsfdsfv
  x-ms-date: Tue, 12 Mar 2019 10:23:27 GMT
  Authorization: SharedKey company1:esSLMOYaK40+dsfsdghbdsg+0=
  ```

- Implement hot, cool, and archive storage
  - snapshot: enables versioning in a blob container. Each time a blob is modified, a new snapshot version is created.
  - hot tier: storing data that is accessed frequently
  - tierToArchive: moves data from expired catalogs to the archive tier (offline storage). Data cannot be accessed directly because it is stored in offline storage. Archive data cannot be read, modified, copied, overwirrte, or have snapshots taken. A rehydratrion process (takes a few hours to complete) is required, to a cool or hot tier, before accessing the data.
  - delete: Catalog data is deleted.
  - tierToCool: Data is moved to a cool tier, satisfying the infrequent data access, therefore minimizing storage costs.
  ```
  {
      "rules": [
          {
              "name": "lifecycleRule",
              "enabled": true,
              "type": "Lifecycle",
              "definition": {
                  "actions": {
                      "baseBlob" : {
                          "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
                          "delete": { "daysAfterModificationGreaterThan": 365 },
                          "tierToCool": { "daysAfterModificationGreaterThan": 30 }
                      }
                  }
              }
          }
      ]
  }
  ```
