# Azure_Engineer_position-_assignment

Cost Optimization in Azure Serverless Architecture with Cosmos DB

To manage storage costs effectively in a serverless architecture while ensuring data accessibility and performance, we propose a hybrid data management approach. This solution retains active data in Azure Cosmos DB for low-latency access, while offloading historical records to cost-effective Azure Blob Storage tiers.

Solution Summary
The system is read-intensive, and most queries target billing records generated within the last 90 days. Historical data (older than 3 months) is rarely accessed but must remain retrievable.

Optimization Approach
Active Data (< 3 months): Stored in Cosmos DB for fast, low-latency access.

Archived Data (> 3 months): Moved to Azure Blob Storage (Cold or Archive tier) to reduce storage costs.

Key Components & Flow
1. Data Archival Workflow
An Azure Function, triggered on a schedule (e.g., daily), moves aged records from Cosmos DB to Blob Storage.

Uses Cosmos DB Change Feed or time-based queries to identify outdated records.

Records are serialized as JSON and uploaded to Blob Storage (Cold/Archive tier).

Optional: Delete original records from Cosmos DB after successful archival.

2. On-Demand Data Retrieval
When a request is made:

The API or Azure Function first checks Cosmos DB.

If the record is missing, it fetches it from Blob Storage.

Fetched records are temporarily cached in Redis or optionally rehydrated into Cosmos DB for future access.

3. Caching Layer (Optional)
Use Azure Redis Cache to store recently retrieved records from Blob Storage.

Minimizes repeated Blob access, reducing both latency and egress costs.

Architecture Diagram
pgsql
Copy
Edit
+---------------------+          +----------------------+
|    API Gateway /    | ------>  |     Cosmos DB        |
|    Frontend Client  |          |  (Active Records)     |
+---------------------+          +----------+-----------+
                                            |
                                 +----------v-----------+
                                 |   Azure Function      |  (Scheduled)
                                 |   (Archival Trigger)  |
                                 +----------+-----------+
                                            |
                                 +----------v-----------+
                                 |  Azure Blob Storage   |  (Cold/Archive Tier)
                                 | (Archived Records)    |
                                 +----------+-----------+
                                            |
                                 +----------v-----------+
                                 | Azure Function/API    |  (On-demand fetch)
                                 +----------+-----------+
                                            |
                                 +----------v-----------+
                                 |   Azure Redis Cache   |  (Optional caching)
                                 +-----------------------+
Implementation Pseudocode
Archival Logic (Azure Function)
python
Copy
Edit
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import json, datetime

cosmos_client = CosmosClient(endpoint_url, credential)
blob_client = BlobServiceClient.from_connection_string(blob_connection_string)

container = cosmos_client.get_database_client("billing-db").get_container_client("records")
blob_container = blob_client.get_container_client("archived-records")

def archive_old_records():
    cutoff_date = datetime.datetime.utcnow() - datetime.timedelta(days=90)
    query = f"SELECT * FROM c WHERE c.creationDate < '{cutoff_date.isoformat()}'"
    old_records = container.query_items(query, enable_cross_partition_query=True)

    for record in old_records:
        blob_name = f"record_{record['id']}.json"
        blob_container.upload_blob(name=blob_name, data=json.dumps(record), overwrite=True)
        container.delete_item(item=record['id'], partition_key=record['id'])
Retrieval Logic (Azure Function)
python
Copy
Edit
def retrieve_record(record_id):
    try:
        return container.read_item(item=record_id, partition_key=record_id)
    except:
        blob_name = f"record_{record_id}.json"
        try:
            blob_data = blob_container.download_blob(blob_name)
            record_json = blob_data.readall()
            record = json.loads(record_json)

            # Rehydrate into Cosmos DB
            container.upsert_item(record)
            return record
        except Exception as e:
            print(f"Error retrieving archived record: {str(e)}")
            return None
Cost Optimization Highlights
Optimization Area	Strategy
Storage Tiering	Use Cold/Archive tiers for aged data in Blob.
Serverless Execution	Leverage Azure Functions for pay-per-use model.
Autoscaling	Configure Cosmos DB with autoscale throughput.
Data Caching	Use Redis to reduce latency for archive reads.
Minimal Egress	Serve repeated fetches from cache/Cosmos DB.


