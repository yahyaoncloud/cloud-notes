# Amazon Elastic Kubernetes Service (EKS): Scaling, IRSA & Karpenter

Amazon EKS delivers a production-grade, highly available Kubernetes control plane backed by AWS SLAs. Running enterprise workloads on EKS requires mastering AWS-specific integrations: **AWS VPC CNI**, **Karpenter intelligent node autoscaling**, **EKS Pod Identity / IRSA**, and zero-downtime cluster upgrades.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The City Metro Transit System
Imagine building a high-speed metro train network:
* **The Control Plane (The City Transit Authority HQ):** AWS owns the central dispatch tower, signal lights, and train tracking servers. You cannot enter their building, but AWS guarantees the signal computers will never crash.
* **AWS VPC CNI (Direct Street Address for Every Passenger):** In standard Kubernetes (like Flannel), pods live on a private, isolated overlay network (like a hidden subway track). To talk to a database on the street outside, traffic must be translated through complex proxy booths. With **AWS VPC CNI**, every single passenger (Pod) receives their own official street address (**a real VPC IP from your subnet**), allowing pods to talk directly to RDS databases, on-premises servers, and ALBs at native line rate without overlay translation!
* **Karpenter (The Dynamic Fleet Commander):** Traditional Cluster Autoscaler is like a slow bus depot that only orders buses in fixed batches of 5. **Karpenter** watches the passenger queue in real time: if 12 passengers arrive, it immediately orders an agile 12-seater electric van; when they leave, it consolidates remaining passengers onto smaller vans to save fuel.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Amazon EKS Control Plane Architecture

When you provision an Amazon EKS cluster:
* AWS provisions a single-tenant, dedicated control plane running across at least 3 Availability Zones.
* Contains 2 API server instances and 3 `etcd` nodes.
* Control plane upgrades, etcd snapshots, and node communication are fully managed by AWS.
* You connect your worker nodes to the control plane using an **EKS Cluster Endpoint** (Public, Public & Private, or Private-Only).

---

### The AWS VPC CNI & IP Address Planning

Unlike overlay networks (Calico / Flannel) that wrap packets in VXLAN headers, the **AWS VPC CNI plugin**:
1. Assigns a real secondary IPv4 address from your VPC subnet directly to each Pod's network namespace (`eth0`).
2. **The ENI & IP Limit Dilemma:** An EC2 instance (e.g., `m5.large`) can only attach a fixed number of ENIs (3 ENIs) and each ENI can only hold 10 IPv4 addresses. Total pods per node = $(3 \times 10) - 3 = 27$ pods!
3. **Prefix Delegation (`ENABLE_PREFIX_DELEGATION=true`):** AWS VPC CNI can allocate entire `/28` IPv4 CIDR blocks (16 IPs) to each ENI slot instead of individual IPs, boosting pod density to 110 pods per node on small instances!

---

### Security: IRSA vs. EKS Pod Identity

How should a pod in Kubernetes authenticate to Amazon S3 or DynamoDB?

```
+------------------------------------+------------------------------------+
| IAM ROLES FOR SERVICE ACCOUNTS     | EKS POD IDENTITY                   |
| (IRSA - Legacy / Standard)         | (Modern Best Practice)             |
+------------------------------------+------------------------------------+
| Uses OIDC (OpenID Connect) Identity| Uses native EKS Pod Identity Agent |
| Provider configured in IAM         | daemonset installed on worker nodes|
+------------------------------------+------------------------------------+
| Pod spec must annotate:            | Pod spec needs NO annotations!     |
| eks.amazonaws.com/role-arn         | Mapped directly in EKS API console |
+------------------------------------+------------------------------------+
| Generates STS WebIdentity tokens   | Automatically mounts credentials   |
| mounted via projected volume       | via AWS SDK credential chain       |
+------------------------------------+------------------------------------+
| Works across all AWS regions       | Simplifies multi-cluster IAM roles |
+------------------------------------+------------------------------------+
```

---

### Node Autoscaling: Cluster Autoscaler vs. Karpenter

```
+------------------------------------+------------------------------------+
| KUBERNETES CLUSTER AUTOSCALER      | KARPENTER (Next-Gen Autoscaler)    |
+------------------------------------+------------------------------------+
| Tied to EC2 Auto Scaling Groups    | GROUP-LESS (Zero ASGs needed)      |
| Slow scaling: 3 - 6 minutes        | Ultra-fast scaling: 40 - 60 seconds|
| Provisions uniform instance types  | Selects exact compute instance size|
| based on Launch Templates          | needed for pending pods (Bin-pack) |
| Minimal cost optimization          | Active Spot consolidation & defrag |
+------------------------------------+------------------------------------+
```

#### How Karpenter Works:
1. Karpenter watches for pods marked `Pending` by `kube-scheduler` due to insufficient CPU/memory.
2. Karpenter evaluates the pending pod's resource requests, node selectors, taints, and architecture constraints.
3. Karpenter calls the AWS EC2 Fleet API directly, provisions the most cost-effective instance (e.g. an ARM64 `c6g.xlarge` Spot instance), and registers it to the cluster in under 45 seconds!
4. **Consolidation:** When workloads decrease, Karpenter drains pods and swaps the node for a cheaper, smaller instance automatically.

---

### Enterprise Zero-Downtime EKS Upgrade Workflow

AWS releases new Kubernetes versions approximately every 4 months, and supports each version for 14 months. Upgrades must follow this strict 4-step sequence:

```
STEP 1: UPGRADE K8S CONTROL PLANE
* Run: aws eks update-cluster-version --name prod-cluster --kubernetes-version 1.30
* Takes ~15 minutes; API server has zero downtime (rolling update across 2+ nodes).

STEP 2: UPGRADE CLUSTER ADD-ONS (In this exact order!)
1. Amazon VPC CNI plugin
2. CoreDNS
3. kube-proxy
4. AWS Load Balancer Controller

STEP 3: UPGRADE WORKER NODES (Managed Node Groups or Karpenter)
* Karpenter: Update NodePool AMI reference; Karpenter rolls nodes automatically.
* Managed Node Groups: Trigger rolling update. Nodes are cordoned, drained, and replaced.

STEP 4: VERIFY APPLICATION WORKLOADS
* Ensure PodDisruptionBudgets (PDB) prevented outages during drain operations!
```

---

##### 3. Visual Flow Diagram: Pod Requesting S3 via IRSA / OIDC

```
[ Pod Container ]
       | (Calls: aws s3 ls)
       v
[ AWS SDK in Container ]
       | (Reads temporary OIDC Web Identity Token injected by Kubelet)
       v
[ AWS Security Token Service (STS) ]
       |
       +---> Validates OIDC JWT signature against EKS Cluster OIDC Provider
       |
       `---> Returns temporary AWS credentials (AccessKey, SecretKey, SessionToken)
                 |
                 v
[ Pod successfully reads data from Amazon S3! ]
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **The Missing PodDisruptionBudget (PDB) Catastrophe:**
  * During an EKS node upgrade, the node is cordoned and drained.
  * If a microservice only has 1 replica, draining the node takes the service completely offline!
  * If a microservice has 3 replicas, but all 3 happen to be running on the same worker node being drained, all 3 get evicted simultaneously!
  * **The Guardrail:** Every production deployment **must** have a `PodDisruptionBudget` declaring `minAvailable: 1` or `maxUnavailable: 1` and `podAntiAffinity` rules spreading pods across nodes.

---

##### 5. Quick Hands-On Check

Verify your EKS cluster add-on versions and compatibility via the AWS CLI:

```bash
# List all active add-ons installed on your EKS cluster
aws eks list-addons \
    --cluster-name production-eks-cluster \
    --region us-east-1 \
    --output table
```
