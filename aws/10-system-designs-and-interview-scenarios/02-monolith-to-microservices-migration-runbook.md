# Monolith to Microservices Cloud Migration: The Strangler Fig Runbook

Migrating a legacy, business-critical monolithic application to cloud-native microservices is one of the most high-stakes initiatives a cloud engineer will encounter. Attempting a "Big Bang" migration—shutting down the old system on Sunday night and turning on 30 new microservices on Monday morning—almost always results in catastrophic failure. This guide breaks down the enterprise **Strangler Fig Pattern**, database decomposition, and zero-downtime cutover.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Strangler Fig Tree
In tropical rainforests, a Strangler Fig seed lands in the high canopy branches of a massive, ancient host tree:
* The fig tree slowly grows roots downward along the host tree's trunk toward the ground soil.
* Over several years, the fig tree roots multiply, thicken, and encase the host tree trunk.
* Gradually, the fig tree absorbs more nutrients, sunlight, and water than the host tree.
* Eventually, the original host tree rots away and disappears, leaving a hollow, strong, freestanding fig tree in its place.
* In software architecture, the **Strangler Fig Pattern** places an API proxy in front of the legacy monolith, intercepts specific endpoints (e.g. `/payments`, `/users`), and routes them to new cloud microservices while leaving the rest of the monolith intact until the monolith rots away and is retired!

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### The 4 Phases of Strangler Fig Migration

```
PHASE 1: BASELINE PROXY INTERCEPTION
[ Client Traffic ] ===> [ AWS API Gateway / ALB ] ===> 100% Traffic ===> [ Legacy Monolith ]

PHASE 2: FIRST STRANGLED SERVICE (Payments)
[ Client Traffic ] ===> [ AWS API Gateway / ALB ]
                               |
                               |-- Path: /api/v1/payments ===> [ New Payment Microservice ]
                               |                                 (ECS Fargate / DynamoDB)
                               `-- Path: /* (Everything else)===> [ Legacy Monolith ]

PHASE 3: CONTINUOUS DECOMPOSITION (Orders, Inventory, Auth)
[ Client Traffic ] ===> [ AWS API Gateway / ALB ]
                               |-- /api/v1/payments ===> [ Payment Service ]
                               |-- /api/v1/orders   ===> [ Orders Service ]
                               |-- /api/v1/auth     ===> [ Auth Service ]
                               `-- /*               ===> [ Shrunken Monolith ]

PHASE 4: COMPLETE RETIREMENT
[ Legacy Monolith Decommissioned & Turned Off! ]
```

---

### Database Decomposition: Breaking the Shared Database

The biggest hurdle in microservice migration is the shared monolithic database:
* **The Anti-Pattern:** Building 5 microservices that all directly query the legacy MySQL database. This creates tight schema coupling; a single column change breaks multiple services.
* **The Goal: Database-per-Service:** Each microservice must own its private database. No other service may query that database except through the service's published REST/gRPC API.

---

### Change Data Capture (CDC) via AWS DMS

During the multi-month migration window, the Monolith and the new Microservices must run concurrently and share live customer data.
Use **AWS Database Migration Service (DMS)** in Change Data Capture (CDC) mode:

```
[ Legacy Monolith Database (Oracle / SQL Server / MySQL) ]
                           |
                           v (Reads Transaction Redo / Binary Logs)
          [ AWS DMS Replication Instance (CDC Mode) ]
                           |
            +--------------+--------------+
            | (Sub-second replication)    |
            v                             v
[ Aurora PostgreSQL (Orders DB) ]    [ DynamoDB (Payments DB) ]
```

#### How AWS DMS CDC Works:
1. DMS takes an initial full load snapshot of existing tables.
2. DMS continuously reads the source database's transaction log (e.g. MySQL `binlog` or Oracle `redo log`).
3. As legacy users make changes in the monolith, DMS writes those changes to the microservices' databases within 500 milliseconds, keeping data perfectly synchronized!

---

### Step-by-Step Production Cutover Checklist

```text
[ ] Step 1: Pre-synchronize databases using AWS DMS CDC. Verify zero replication lag.
[ ] Step 2: Deploy new microservice to ECS Fargate / EKS behind an internal ALB.
[ ] Step 3: Enable "Shadow Traffic" (Dark Launching):
            Use an ALB or Envoy proxy to duplicate incoming production requests to the
            new microservice in the background. Compare responses without serving them to users.
[ ] Step 4: Shift 5% of real user traffic via Route 53 or API Gateway canary release.
[ ] Step 5: Monitor SLIs: Error rates, database CPU, and latency percentiles.
[ ] Step 6: Incrementally ramp traffic: 25% -> 50% -> 100%.
[ ] Step 7: Sever write permissions on legacy monolith tables; make them read-only.
[ ] Step 8: Archive legacy tables to S3 Glacier Deep Archive; terminate legacy monolith EC2.
```
