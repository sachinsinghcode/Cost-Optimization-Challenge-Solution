# Solution for Cost Optimization Challenge

## Inputs

 1. Cosmos DB as the primary storage solution.
 2. Read heavy for Latest data, Records stored > 3 Months are rarely accessed
 3. Record Size <= 300KB for around 2 Million Records
 4. API Endpoints to remain same for querying

## Calculations and Assumptions:

Total Data stored would be 300KBs x 2x10^6 Records = ~**600GBs**

Assuming Data is 3 Year old and write speed is constant over the Years
Last 3 months Active Data = 10% of total data = ~60GBs
Data to be archived/cooled = 90% of the data = ~540GBs

### 💵 Cosmos DB Monthly Cost: With vs Without Archiving

| Scenario                      | Storage Used | Cosmos DB Storage Cost - 0.25$/GB | RU/s Provisioned - 0.008$/hour | RU/s Cost | **Total Monthly Cost** |
|------------------------------|--------------|-------------------------|------------------|-----------|-------------------------|
| **1. No Archiving**          | 600 GB       | 0.25$ x 600 = $150                    |  1000 RU/s = ~$0.008/hour × 1000 = $0.008 × 24 × 30 = **$5.76/day**          | $172.80   | **$322.80**             |
| **2. Archiving 90% of Data** | 60 GB        | 0.25$ x 60 = $15                     |  1000 RU/s = ~$0.008/hour × 1000 = $0.008 × 24 × 30 = **$5.76/day**          | $172.80   | **$187.80** + nominal archival cost of 560 GBs             |

## First most and Simplest Choice

If we can afford an additional ~ 322.90 - 187.80 = 135$ cost per month then the best option is to not archive the data and avoid introducing complexity into the system risking integrity of the data.
But if the above option is not acceptable then we can go with below approach.

## Proposed Architecture

Once records become older than three months, they can be offloaded to a more cost-effective storage solution. In our case, we have two suitable options: Azure Blob Storage and Azure Data Lake Storage. Between the two, the **cool tier of Azure Data Lake Storage** is a better fit for our scenario. We're dealing with large volumes of data that are infrequently accessed, but when accessed, they need to be available within a few seconds. Azure Data Lake also offers better support for querying and analytics, which aligns well with our use case.

To further optimize cost, we can define a lifecycle policy that moves very old data, say beyond a year to the **archive tier**, which is ideal for data that is rarely accessed but must still be retained for compliance or analysis.

For transferring data from Cosmos DB to Data Lake once it crosses the 3-month threshold, we can use a trigger-based approach. This can be implemented using **Azure Logic Apps** or a custom **Azure Function App** running in the consumption plan to keep costs low.

Assuming a write-once model (i.e., records are written once and not updated), we can enable **Time to Live (TTL)** in Cosmos DB to automatically flag records for deletion after three months plus a buffer. This buffer period ensures that data isn't deleted prematurely due to delays, failures, or exceptions in the transfer pipeline. However, if the data is subject to updates after creation, we’ll need to track the "last updated" timestamp manually and use that to determine aging instead of relying solely on TTL.

There's some ambiguity around the requirement to “keep the existing API contracts the same.” If that refers to a custom REST API layer, this is straightforward to handle. We can simply modify the existing endpoint logic to first attempt a lookup in Cosmos DB, and if the record isn't found, fall back to the archived data in Data Lake. For example:

    async function getItemById(id) {
    try {
        // Try Cosmos DB first
        const { resource } = await container.item(id, partitionKey).read();
        return resource;
    } catch (err) {
        if (err.code === 404) {
            // Try Azure Data lake
            const archived = await readFromDataLake(id);
            return archived || null;
        } 
        else {
            throw err;
        }
	   }
	}

This pattern allows clients to continue using the same REST endpoint, without knowing whether the data is being served from hot or cold storage.

However, if "existing API contracts" refers to direct access to Cosmos DB through its native endpoint using the Cosmos DB connection string, SDKs, or SQL-like queries, then it's not possible to transparently integrate archived data into that model. Once data is deleted from Cosmos DB ,it is no longer accessible via its SDKs or endpoints, and any archival layer would require a separate access path.


I used chatgpt to to read about to research about connectivity in cosmosDB. I feel the architecture is quite simple and doesn't need an architecture diagram to explain itself



