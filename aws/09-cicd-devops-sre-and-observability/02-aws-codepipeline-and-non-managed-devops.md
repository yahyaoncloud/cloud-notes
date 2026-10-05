# AWS CodePipeline, CodeDeploy & Non-Managed Self-Hosted DevOps

In enterprise cloud environments, organizations frequently debate whether to adopt **AWS-native managed DevOps services** (AWS CodePipeline, CodeBuild, CodeDeploy) or engineer **self-managed, non-managed CI/CD pipelines** (self-hosted GitLab CI or Jenkins on EC2/EKS) to eliminate cloud vendor lock-in. This guide explores both architectures and deconstructs CodeDeploy AppSpec lifecycle hooks.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Monorail System vs. Building Your Own Highway
* **AWS Managed DevOps (The Proprietary Monorail):**
  * You buy into AWS CodePipeline, CodeBuild, and CodeDeploy.
  * AWS lays the magnetic tracks, powers the train, and guarantees it stops precisely at the EC2 and ECS stations.
  * It integrates effortlessly with IAM roles and CloudWatch alarms, but it only travels to destinations built by AWS. You cannot easily drive the monorail train into a Google Cloud or Microsoft Azure station.
* **Non-Managed Self-Hosted DevOps (Building Your Own Multi-Lane Highway):**
  * You launch an EC2 Auto Scaling group or Kubernetes cluster running self-hosted **GitLab CI runners** or **Jenkins agents**.
  * You maintain the asphalt, paint the lane lines, and fix the potholes (**OS patching, backup scripts, disk maintenance**).
  * However, any vehicle in the world (Docker, Kubernetes, AWS, Azure, on-premises bare metal) can drive on your highway with zero cloud vendor lock-in.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### The AWS Native DevOps Suite

```
[ GITHUB / CODECOMMIT / S3 ] ===> [ AWS CODEBUILD ] ===> [ AWS CODEDEPLOY ] ===> [ EC2 / ECS / LAMBDA ]
        ^                                 ^                      ^
        |                                 |                      |
+-------+---------------------------------+----------------------+---------------------------------+
|                                    AWS CODEPIPELINE                                              |
|                      (Central Visual Pipeline Orchestrator)                                      |
+--------------------------------------------------------------------------------------------------+
```

1. **AWS CodePipeline:** Visual workflow orchestrator that coordinates actions across stages (Source -> Build -> Test -> Manual Approval -> Deploy).
2. **AWS CodeBuild:** Fully managed build service. Spawns an ephemeral Docker container based on `buildspec.yml`, runs compilation and tests, outputs artifacts to S3, and terminates. Zero servers to manage.
3. **AWS CodeDeploy:** The deployment engine capable of executing rolling, blue/green, and canary deployments to EC2 instances, ECS Fargate services, or Lambda functions.

---

### CodeDeploy AppSpec Lifecycle Hooks (`appspec.yml`)

When deploying software to EC2 instances via the AWS CodeDeploy Agent, CodeDeploy executes deployment scripts in an exact, deterministic lifecycle order:

```
====================== CODEDEPLOY AGENT LIFECYCLE SEQUENCE ======================
1. ApplicationStop     ===> Gracefully stop the current running application
2. DownloadBundle       ===> Agent downloads new zip/tar artifact from Amazon S3
3. BeforeInstall        ===> Pre-installation scripts (decrypt secrets, backup DB)
4. Install              ===> Copies artifact files from temp dir to destination
5. AfterInstall         ===> Configures permissions, symlinks, runs migrations
6. ApplicationStart    ===> Starts systemd service (e.g. systemctl start app)
7. ValidateService      ===> CRUCIAL: Pings http://localhost:8080/healthz!
                             If health check fails, CodeDeploy triggers ROLLBACK!
=================================================================================
```

#### Production `appspec.yml` Example:
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/payment-api
hooks:
  BeforeInstall:
    - location: scripts/backup_current_version.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_systemd_service.sh
      timeout: 60
      runas: root
  ValidateService:
    - location: scripts/verify_health_status.sh
      timeout: 60
      runas: root
```

---

### Engineering Non-Managed DevOps (Zero Vendor Lock-In)

If an enterprise client requires an architecture that can be migrated between AWS, Azure, and on-premises data centers without rewriting deployment scripts, deploy **Self-Hosted GitLab CI / Jenkins on Kubernetes**:

```
+----------------------------------------------------------------------------------------------------+
|                         NON-MANAGED ENTERPRISE DEVOPS INFRASTRUCTURE                               |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | KUBERNETES CLUSTER (EKS / Self-Hosted)                                                       |  |
|  |                                                                                              |  |
|  |  [ GitLab CI Runner Controller ]                                                             |  |
|  |              |                                                                               |  |
|  |              v (Dynamically spawns ephemeral Pods per build job)                             |  |
|  |  +---------------------------+  +---------------------------+  +--------------------------+  |  |
|  |  | Build Job Pod #1 (Kaniko) |  | Test Job Pod #2 (PyTest)  |  | Deploy Job Pod #3 (Helm) |  |  |
|  |  | - Builds Docker without   |  | - Unit & Integration tests|  | - Deploys to EKS / AKS   |  |  |
|  |  |   privileged Docker daemon|  |   in isolated sandbox     |  |   using generic manifests|  |  |
|  |  +---------------------------+  +---------------------------+  +--------------------------+  |  |
|  +----------------------------------------------------------------------------------------------+  |
+----------------------------------------------------------------------------------------------------+
```

#### Key Architectural Standards for Non-Managed Pipelines:
* **Kaniko for Docker Builds:** In Kubernetes, running `docker build` requires mounting the host's `/var/run/docker.sock` in privileged mode (a major security risk!). Use **Google Kaniko**, which builds container images entirely inside userspace without requiring root host access.
* **Helm & Kustomize for Deployment:** Keep deployment manifests cloud-agnostic. The exact same Helm chart that deploys to AWS EKS can deploy to Azure AKS or Google GKE.

---

##### 3. Visual Flow Diagram: Architectural Trade-Off Comparison

```
+------------------------------------+------------------------------------+
| AWS MANAGED (CodePipeline)         | NON-MANAGED (GitLab / K8s Runners) |
+------------------------------------+------------------------------------+
| Zero server maintenance overhead   | High operational maintenance       |
| Native IAM & CloudWatch integration| Requires custom secrets management |
| Vendor lock-in to AWS APIs         | 100% portable across any cloud     |
| Costs: Charged per active pipeline | Cost: Run on EC2 Spot / K8s nodes  |
| ($1.00/mo) + CodeBuild per-minute  | (Massive economies of scale)       |
+------------------------------------+------------------------------------+
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Why did my CodeDeploy fail at ValidateService even though my code compiled?"**
  * `ValidateService` is the final gatekeeper hook. If your script runs `curl http://localhost:8080/health` immediately after `systemctl start app`, but the application takes 10 seconds to warm up database connections, `curl` receives `Connection Refused` and CodeDeploy marks the deployment as **FAILED**.
  * **The Fix:** Add a retry loop inside `verify_health_status.sh` that polls the endpoint 6 times with 5-second sleep intervals before giving up!
