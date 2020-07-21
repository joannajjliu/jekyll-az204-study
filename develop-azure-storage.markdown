---
layout: page
title: Develop for Azure Storage
permalink: /develop-azure-storage/
---
##  (10 - 15%)
### Develop Solutions using Cosmost DB storage
- Select the appropriate API for your solution
  - SQL API: returns data in JSON format.
  - Table API: returns data as a table, structured in rows and columns.
  
- Implement partitioning schemes
- Interact with data using the appropriate SDK
  - Access the **first item in the Items container** of the AZ Cosmos DB:
  - Cosmos DB account > dbs > containers/collections > documents/items
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
  - **Bounded staleness**: reads might lag behind writes based on configured updated version (K) or time (t).
  - **Session**: Reads honor the consistent prefix level. This level does not prevent a client from receiving partial writes.
  - **Consistent prefix**: Ensures reads never see out of order writes. But does not prevent reads from seeing partial writes.
  - **Eventual**: Lowest latency, higher availability for data reads. Does not guarantee ordering of reads; not an issue for a static database not receiving new data writes. Clients can potentially receive a partial write, or uncommitted data during a read operation.
- Create Cosmos DB containers
- Implement scaling (partitions, containers)
- Implement server-side programming including stored procedures, triggers, and change feed notifications

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