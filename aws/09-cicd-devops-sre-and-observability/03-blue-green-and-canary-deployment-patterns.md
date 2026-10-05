# Zero-Downtime Deployment Strategies: Rolling, Blue/Green & Canary Patterns

Deploying software directly to live production instances without a deployment strategy leads to dropped customer connections, broken database sessions, and catastrophic downtime. Modern cloud engineering uses three primary deployment patterns: **Rolling Updates**, **Blue/Green Deployments**, and **Canary Releases**.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: Replacing Lightbulbs in an Operating Room
Imagine a hospital operating room that must stay illuminated 24/7:
* **The In-Place / Rolling Update:** An electrician walks in with 4 new LED bulbs. They unscrew Bulb #1 and screw in the replacement while surgery is underway. For 3 minutes, the room is 25% dimmer, and if the new bulb explodes, 25% of the room goes dark (**Incremental replacement, mixed version state**).
* **The Blue/Green Deployment:** The hospital builds an identical, fully illuminated secondary operating room right next door (**Green Environment**). The surgical team inspects every lightbulb, confirms all monitors work, and then rolls the patient's gurney across the hall in 5 seconds (**Instant Traffic Shift**). If a monitor fails, they wheel the patient right back (**Zero-Risk Instant Rollback**).
* **The Canary Release (The Miner's Canary):** Before entering a dangerous underground coal mine, miners carried a caged canary bird. Because canaries are hypersensitive to odorless toxic gases, if the bird showed distress, miners evacuated immediately before humans were harmed. In software, you route **10% of real customer traffic** to the new code. If the "canary" sounds an alarm (**5xx error rate spikes**), the deployment rolls back automatically, protecting 90% of your users from experiencing a bug.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Architectural Comparison: The 3 Deployment Patterns

```
+--------------------------+--------------------------+--------------------------+
| ROLLING UPDATE           | BLUE / GREEN DEPLOYMENT  | CANARY DEPLOYMENT        |
+--------------------------+--------------------------+--------------------------+
| How it works:            | How it works:            | How it works:            |
| Updates instances in     | Provisions 100% duplicate| Routes small % of traffic|
| batches (e.g. 2 at time) | environment; flips switch| (10%), monitors alarms,  |
| behind load balancer     | via DNS or ALB weights   | then ramps to 100%       |
+--------------------------+--------------------------+--------------------------+
| Downtime: ZERO           | Downtime: ZERO           | Downtime: ZERO           |
+--------------------------+--------------------------+--------------------------+
| Extra Cost: Minimal (10%)| Extra Cost: +100% during | Extra Cost: Small (10%)  |
| (Only temporary surge)   | deployment window        |                          |
+--------------------------+--------------------------+--------------------------+
| Rollback Speed: SLOW     | Rollback Speed: INSTANT  | Rollback Speed: FAST     |
| (Must re-roll backwards) | (1 click to flip back)   | (Automated rollback)     |
+--------------------------+--------------------------+--------------------------+
| Blast Radius: Medium     | Blast Radius: 100% of    | Blast Radius: MINIMAL    |
| (Some users see errors)  | users if switch happens  | (Only 10% exposed)       |
+--------------------------+--------------------------+--------------------------+
```

---

### Blue/Green Architecture on AWS

```
                                  [ Amazon Route 53 / ALB Listener ]
                                                   |
                     +-----------------------------+-----------------------------+
                     | (Active Traffic: 100%)                                    | (Test Traffic: 0%)
                     v                                                           v
         [ BLUE TARGET GROUP (v1.0) ]                                [ GREEN TARGET GROUP (v2.0) ]
                     |                                                           |
         +-----------+-----------+                                   +-----------+-----------+
         |                       |                                   |                       |
         v                       v                                   v                       v
    [ EC2 / ECS Task ]      [ EC2 / ECS Task ]                  [ EC2 / ECS Task ]      [ EC2 / ECS Task ]
```

#### Step-by-Step Blue/Green Cutover:
1. Current production is **Blue** (v1.0), serving 100% of customer traffic.
2. CI/CD provisions **Green** (v2.0) in the same VPC with identical configuration.
3. QA / SRE runs automated synthetic integration tests against Green's private test port.
4. When tests pass, the ALB Listener flips the target group rule: **Green gets 100%, Blue gets 0%**.
5. Blue instances are kept warm on standby for 60 minutes.
6. If critical bugs appear, flip back to Blue in 2 seconds. If no bugs appear, terminate Blue!

---

### Canary Deployment with Automated CloudWatch Rollback

Using AWS CodeDeploy or AWS App Mesh, you define a canary traffic shifting schedule:

```
[ INCOMING TRAFFIC ]
        |
        v
[ ALB Weighted Routing ]
  |-- Minute 0 to 15:   90% to Prod (v1.0)  |  10% to Canary (v2.0)
  |-- CloudWatch Alarm evaluates 5xx errors on Canary Target Group!
  |
  +---> IF 5xx Errors > 1% ==> ABORT & REVERT 100% TO PROD IMMEDIATELY!
  |
  `---> IF 5xx Errors == 0 ==> Minute 15+: Shift 100% to Canary (v2.0)!
```

#### Standard CodeDeploy Canary Configuration:
* **`CodeDeployDefault.ECSCanary10Percent15Minutes`**: Shifts 10% of traffic in the first increment, monitors for 15 minutes, then shifts remaining 90% if no CloudWatch alarms fire.
* **`CodeDeployDefault.ECSLinear10PercentEvery3Minutes`**: Shifts 10% every 3 minutes linearly over 30 minutes.

---

### The Database Migration Trap (N-1 Backward Compatibility)

The most common failure in Blue/Green deployments is the **Shared Database**:
> If Blue (v1.0) and Green (v2.0) share the same RDS database, and Green drops a column or alters a table schema before the traffic flip, **Blue immediately crashes for 100% of active users!**

#### The Expand-Contract (Parallel Run) Pattern:
1. **Expand Phase:** Add the new column as `NULLABLE` without deleting the old column.
2. Deploy Green (v2.0). Both Blue and Green can read and write without breaking.
3. Flip traffic 100% to Green.
4. **Contract Phase:** In a subsequent deployment 1 week later, drop the obsolete legacy column.

---

##### 3. Visual Flow Diagram: Expand-Contract Database Schema Evolution

```
STATE 1: Base State            STATE 2: Expand (Both Work)    STATE 3: Contract (Cleanup)
+-----------------------+      +-----------------------+      +-----------------------+
| Table: Users          |      | Table: Users          |      | Table: Users          |
| - id                  | ===> | - id                  | ===> | - id                  |
| - full_name (v1 code) |      | - full_name (v1 code) |      | - first_name (v2 code)|
+-----------------------+      | - first_name (v2 code)|      | - last_name  (v2 code)|
                               | - last_name  (v2 code)|      +-----------------------+
                               +-----------------------+
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Why did my Blue/Green deployment cause cache inconsistencies?"**
  * If user session state or shopping cart data is stored in local instance memory, shifting a user from Blue to Green wipes their active session!
  * **The Prerequisite:** All state **must be externalized** to an in-memory cache (Amazon ElastiCache Redis) or client-side signed JWTs before Blue/Green deployments are possible.
