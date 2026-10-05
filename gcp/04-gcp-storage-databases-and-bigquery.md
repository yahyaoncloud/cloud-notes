# GCP Storage, Databases & BigQuery Architecture

Data storage and analytics architectures on Google Cloud Platform.

---

## 1. Cloud Storage (GCS) Class Selection

| Storage Class | Min Storage Duration | Typical Latency | Best Use Case |
| :--- | :--- | :--- | :--- |
| **Standard** | None | Milliseconds | Hot website assets, active data feeds, mobile downloads |
| **Nearline** | 30 days | Milliseconds | Monthly reports, 30-day backups, multimedia archives |
| **Coldline** | 90 days | Milliseconds | Quarterly disaster recovery backups, legal records |
| **Archive** | 365 days | Milliseconds | Compliance data stored for 5-10 years, raw sensor logs |

*Note: GCS cold storage retrieves data in milliseconds without the multi-hour restore delay required by AWS S3 Glacier.*

### Lifecycle Configuration (JSON)
```json
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30, "matchesPrefix": ["logs/"]}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365, "matchesPrefix": ["logs/"]}
    }
  ]
}
```

---

## 2. Managed Relational & Distributed Databases

* **Cloud SQL**: Fully managed MySQL, PostgreSQL, and SQL Server. Supports automated read replicas, automated failover, private IP connectivity via VPC Service Peering.
* **Cloud Spanner**: Globally distributed, synchronously replicated relational database with ACID transactions, external consistency (using TrueTime atomic clocks), and horizontal scalability without sharding.
* **Firestore**: Serverless NoSQL document database with real-time listeners and multi-region replication.

---

## 3. BigQuery Architecture & Optimization

BigQuery decouples compute (Dremel execution engine) from storage (Colossus distributed filesystem) using Google's Jupiter petabit-scale network.

### Performance & Cost Optimization Rules
1. **Partitioning**: Partition tables by date/timestamp or integer range to avoid scanning full tables.
2. **Clustering**: Cluster on high-cardinality search fields (e.g. `customer_id`, `status`) to co-locate related records.
3. **Avoid `SELECT *`**: BigQuery charges per byte scanned in the queried columns.
4. **Use BI Engine**: In-memory caching for real-time dashboards (Looker / Tableau).
