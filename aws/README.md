# AWS Cloud Core, Architecture & Enterprise Implementation Master Guide

This repository module contains a comprehensive, production-grade curriculum designed for Cloud Engineers preparing for technical interviews, systems design whiteboard rounds, coding interviews, and real-world enterprise infrastructure delivery.

---

## Modular Domain Directory Index

### [Domain 01: Cloud Engineering Role & Core Principles](01-cloud-engineering-role-and-principles/)
* [`01-cloud-principles-and-shared-responsibility.md`](01-cloud-engineering-role-and-principles/01-cloud-principles-and-shared-responsibility.md): Capex vs. Opex, Elasticity, High Availability vs Fault Tolerance, and the AWS Shared Responsibility Model across IaaS, PaaS, and Serverless.
* [`02-cloud-engineer-role-and-cross-functional-workflows.md`](01-cloud-engineering-role-and-principles/02-cloud-engineer-role-and-cross-functional-workflows.md): Cross-functional collaboration with Developers, SecOps, SRE, and executive stakeholders, executing requirements step-by-step, and trade-off negotiation.
* [`03-workload-types-and-well-architected-framework.md`](01-cloud-engineering-role-and-principles/03-workload-types-and-well-architected-framework.md): Workload archetypes (Stateless, Stateful, Batch, AI/ML), the 6 Pillars of the AWS Well-Architected Framework, and the 6 Rs of Cloud Migration.

### [Domain 02: Systems Foundations, Protocols & OS Internals](02-foundations-networking-and-systems/)
* [`01-how-the-internet-works-packet-journey-and-dns.md`](02-foundations-networking-and-systems/01-how-the-internet-works-packet-journey-and-dns.md): The nanosecond packet lifecycle from browser cache and recursive DNS to ARP, TCP 3-way handshake, TLS 1.3, and HTTP/2 edge routing.
* [`02-ssl-tls-handshake-and-pki-certificates.md`](02-foundations-networking-and-systems/02-ssl-tls-handshake-and-pki-certificates.md): TLS 1.2 vs 1.3 handshake mechanics, Public Key Infrastructure (PKI), AWS Certificate Manager (ACM) DNS validation, and mutual TLS (mTLS).
* [`03-linux-boot-ssh-and-access-key-management.md`](02-foundations-networking-and-systems/03-linux-boot-ssh-and-access-key-management.md): Linux boot sequence (BIOS to systemd PID 1), SSH asymmetric cryptography, authorized_keys permissions, and AWS Systems Manager (SSM) Session Manager.
* [`04-cloud-native-os-and-systems-troubleshooting.md`](02-foundations-networking-and-systems/04-cloud-native-os-and-systems-troubleshooting.md): Cloud-native OS distributions (Bottlerocket, AL2023), and the USE method for triaging CPU, memory (OOM killer), disk IOPS, and socket exhaustion.

### [Domain 03: AWS Networking & Hybrid Connectivity](03-aws-networking-and-hybrid-connectivity/)
* [`01-vpc-architecture-subnets-and-route-tables.md`](03-aws-networking-and-hybrid-connectivity/01-vpc-architecture-subnets-and-route-tables.md): IPv4 CIDR planning, the 5 AWS reserved IPs per subnet, Public vs Private vs Database subnets, and Route Table routing logic.
* [`02-security-groups-nacls-and-flow-logs.md`](03-aws-networking-and-hybrid-connectivity/02-security-groups-nacls-and-flow-logs.md): Stateful Security Groups vs stateless NACLs, ephemeral return ports (1024-65535), and VPC Flow Log packet rejection analysis.
* [`03-nat-gateways-internet-gateways-and-endpoints.md`](03-aws-networking-and-hybrid-connectivity/03-nat-gateways-internet-gateways-and-endpoints.md): Multi-AZ NAT Gateway high availability vs Gateway Endpoints (free S3/DynamoDB) and Interface Endpoints (AWS PrivateLink).
* [`04-transit-gateway-and-vpc-peering.md`](03-aws-networking-and-hybrid-connectivity/04-transit-gateway-and-vpc-peering.md): Non-transitive VPC peering limits, Hub-and-Spoke Transit Gateway routing domains, and solving firewall asymmetric routing using Appliance Mode.
* [`05-site-to-site-vpn-and-direct-connect.md`](03-aws-networking-and-hybrid-connectivity/05-site-to-site-vpn-and-direct-connect.md): IPsec Phase 1 (IKE) and Phase 2 (ESP) protocols, AWS Direct Connect (Private/Public/Transit VIF), and deterministic BGP AS-path failover.
* [`06-multicloud-interconnects-and-network-triage.md`](03-aws-networking-and-hybrid-connectivity/06-multicloud-interconnects-and-network-triage.md): Multi-cloud interconnects (AWS, Azure, GCP), overlapping CIDR resolution via PrivateLink, and the 7-layer network triage framework.

### [Domain 04: Compute, Containers & Kubernetes](04-compute-containers-and-kubernetes/)
* [`01-ec2-lifecycle-instance-types-and-auto-scaling.md`](04-compute-containers-and-kubernetes/01-ec2-lifecycle-instance-types-and-auto-scaling.md): EC2 instance families (M, C, R, G, Graviton), On-Demand vs Spot economics, Target Tracking Auto Scaling, and rolling instance refreshes.
* [`02-load-balancing-alb-nlb-gwlb-and-waf.md`](04-compute-containers-and-kubernetes/02-load-balancing-alb-nlb-gwlb-and-waf.md): ALB (Layer 7) vs NLB (Layer 4) vs GWLB (GENEVE), client IP preservation, and AWS WAF OWASP rule mitigation.
* [`03-cloudfront-cdn-and-route53-hosted-zones.md`](04-compute-containers-and-kubernetes/03-cloudfront-cdn-and-route53-hosted-zones.md): CloudFront CDN edge caching, Route 53 Public/Private hosted zones, Zone Apex Alias records, and routing policies (Weighted, Failover, Latency).
* [`04-docker-fundamentals-and-amazon-ecs-fargate.md`](04-compute-containers-and-kubernetes/04-docker-fundamentals-and-amazon-ecs-fargate.md): Docker multi-stage builds, Amazon ECS architecture, Fargate vs EC2 launch types, awsvpc networking, and Task Execution Role vs Task Role.
* [`05-kubernetes-architecture-and-manifest-engineering.md`](04-compute-containers-and-kubernetes/05-kubernetes-architecture-and-manifest-engineering.md): K8s Control Plane & Worker components, writing production YAML manifests (Deployments, Services, HPA, Probes, Security Contexts).
* [`06-amazon-eks-deployment-scaling-and-maintenance.md`](04-compute-containers-and-kubernetes/06-amazon-eks-deployment-scaling-and-maintenance.md): Amazon EKS architecture, AWS VPC CNI prefix delegation, IRSA vs EKS Pod Identity, Karpenter node autoscaling, and zero-downtime upgrades.

### [Domain 05: Storage, Databases & Caching Systems](05-storage-databases-and-caching/)
* [`01-amazon-s3-end-to-end-security-and-lifecycle.md`](05-storage-databases-and-caching/01-amazon-s3-end-to-end-security-and-lifecycle.md): S3 strong consistency, storage class economics, Block Public Access, bucket policies, SSE-KMS envelope encryption, and Object Lock.
* [`02-rds-and-aurora-architecture-and-failover.md`](05-storage-databases-and-caching/02-rds-and-aurora-architecture-and-failover.md): RDS Multi-AZ sync standby vs async read replicas, Aurora 6-copy distributed shared storage, Global Databases, and RDS Proxy connection pooling.
* [`03-dynamodb-nosql-modeling-and-performance.md`](05-storage-databases-and-caching/03-dynamodb-nosql-modeling-and-performance.md): DynamoDB partition key hashing, 10GB/1000WCU limits, On-Demand vs Provisioned RCU/WCU math, GSI vs LSI, Streams, and TTL.
* [`04-elasticache-redis-and-caching-strategies.md`](05-storage-databases-and-caching/04-elasticache-redis-and-caching-strategies.md): Redis vs Memcached, caching patterns (Cache-Aside, Write-Through, Write-Behind), cache stampede mitigation, and LRU eviction policies.

### [Domain 06: Security, Identity & Compliance](06-security-identity-and-compliance/)
* [`01-iam-mastery-roles-policies-and-conditions.md`](06-security-identity-and-compliance/01-iam-mastery-roles-policies-and-conditions.md): Official AWS Policy Evaluation logic, STS AssumeRole temporary credentials, conditional keys (aws:SourceIp, aws:PrincipalOrgID), and boundaries.
* [`02-kms-envelope-encryption-and-secrets-manager.md`](06-security-identity-and-compliance/02-kms-envelope-encryption-and-secrets-manager.md): KMS Customer Managed Keys, Envelope Encryption with Data Encryption Keys (DEKs), and Secrets Manager vs SSM Parameter Store.
* [`03-cloudtrail-cloudwatch-and-auditing-guardrails.md`](06-security-identity-and-compliance/03-cloudtrail-cloudwatch-and-auditing-guardrails.md): CloudTrail Management vs Data events, cryptographic log integrity validation, CloudWatch Metric Filters, and Composite Alarms.
* [`04-guardduty-security-hub-and-access-analyzer.md`](06-security-identity-and-compliance/04-guardduty-security-hub-and-access-analyzer.md): Agentless threat detection with GuardDuty, compliance posture scoring via Security Hub (CIS Benchmarks), and automated EventBridge remediation.

### [Domain 07: Infrastructure as Code (IaC) & Automation](07-iac-automation-and-scripting/)
* [`01-top-10-coding-and-algorithmic-patterns.md`](07-iac-automation-and-scripting/01-top-10-coding-and-algorithmic-patterns.md): The 10 core coding and algorithmic patterns for cloud engineers (Sliding window, Hash maps, Interval merging, Trie LPM, DAG traversal, Token bucket).
* [`02-boto3-python-and-bash-automation-scripts.md`](07-iac-automation-and-scripting/02-boto3-python-and-bash-automation-scripts.md): Production Boto3 client standards, pagination, waiters, adaptive retries, Bash strict mode (`set -euo pipefail`), and error traps.
* [`03-golang-for-cloud-engineers.md`](07-iac-automation-and-scripting/03-golang-for-cloud-engineers.md): Why Go powers cloud infrastructure, Goroutines vs OS threads, Context timeouts, and the AWS SDK for Go v2.
* [`04-terraform-gitops-state-and-modules.md`](07-iac-automation-and-scripting/04-terraform-gitops-state-and-modules.md): Terraform remote state backends (S3 + DynamoDB LockID), state drift detection, `import {}` blocks, directory isolation, and GitOps.
* [`05-aws-cdk-cloudformation-and-ansible.md`](07-iac-automation-and-scripting/05-aws-cdk-cloudformation-and-ansible.md): CloudFormation StackSets, AWS CDK Construct hierarchy (L1/L2/L3), synthesis, and Ansible agentless idempotent playbook automation.

### [Domain 08: Serverless & Event-Driven Computing](08-serverless-and-event-driven-architecture/)
* [`01-lambda-internals-concurrency-and-vpc-egress.md`](08-serverless-and-event-driven-architecture/01-lambda-internals-concurrency-and-vpc-egress.md): Firecracker microVM lifecycle, Cold starts, Reserved vs Provisioned Concurrency, and AWS Hyperplane shared ENI networking.
* [`02-sqs-sns-and-eventbridge-decoupling.md`](08-serverless-and-event-driven-architecture/02-sqs-sns-and-eventbridge-decoupling.md): SQS Standard vs FIFO, Visibility Timeout, DLQs, Long Polling, SNS pub/sub fanout, and EventBridge content-based JSON filtering.
* [`03-step-functions-and-serverless-resilience.md`](08-serverless-and-event-driven-architecture/03-step-functions-and-serverless-resilience.md): Step Functions Standard vs Express state machines, the Saga Pattern for distributed transactions, and native ASL Retry/Catch blocks.

### [Domain 09: CI/CD, DevOps, SRE & Observability](09-cicd-devops-sre-and-observability/)
* [`01-github-actions-and-jenkins-pipelines.md`](09-cicd-devops-sre-and-observability/01-github-actions-and-jenkins-pipelines.md): GitHub Actions vs Jenkins, keyless AWS authentication via OpenID Connect (OIDC), and production pipeline security guardrails.
* [`02-aws-codepipeline-and-non-managed-devops.md`](09-cicd-devops-sre-and-observability/02-aws-codepipeline-and-non-managed-devops.md): AWS native DevOps (CodePipeline, CodeBuild, CodeDeploy AppSpec lifecycle hooks) vs building non-managed, self-hosted CI/CD pipelines.
* [`03-blue-green-and-canary-deployment-patterns.md`](09-cicd-devops-sre-and-observability/03-blue-green-and-canary-deployment-patterns.md): Zero-downtime deployment patterns, Route 53/ALB traffic shifting, CodeDeploy automated rollback on alarms, and database Expand-Contract schema evolution.
* [`04-metrics-logs-and-distributed-tracing.md`](09-cicd-devops-sre-and-observability/04-metrics-logs-and-distributed-tracing.md): The 3 Pillars of Observability, distributed trace context propagation (`X-Amzn-Trace-Id`), AWS X-Ray service maps, and CloudWatch Insights SQL queries.
* [`05-sre-playbooks-runbooks-and-incident-triage.md`](09-cicd-devops-sre-and-observability/05-sre-playbooks-runbooks-and-incident-triage.md): SRE reliability metrics (SLI, SLO, SLA, Error Budgets), the 5-phase incident response lifecycle, and production runbook authoring standards.

### [Domain 10: Enterprise Systems Design & Whiteboard Scenarios](10-system-designs-and-interview-scenarios/)
* [`01-three-tier-production-aws-architecture.md`](10-system-designs-and-interview-scenarios/01-three-tier-production-aws-architecture.md): Complete end-to-end 3-tier enterprise architecture blueprint, component justifications, traffic flows, and interview defense walkthrough.
* [`02-monolith-to-microservices-migration-runbook.md`](10-system-designs-and-interview-scenarios/02-monolith-to-microservices-migration-runbook.md): The Strangler Fig migration pattern, database decomposition, Change Data Capture (CDC) via AWS DMS, and dark launching.
* [`03-hybrid-datacenter-extension-and-dns.md`](10-system-designs-and-interview-scenarios/03-hybrid-datacenter-extension-and-dns.md): Direct Connect + VPN over Transit Gateway, Active Directory trust, and bidirectional Hybrid DNS via Route 53 Inbound and Outbound Resolvers.
* [`04-finops-cost-optimization-and-client-negotiation.md`](10-system-designs-and-interview-scenarios/04-finops-cost-optimization-and-client-negotiation.md): FinOps principles, GP2 to GP3 migration, S3 Glacier Deep Archive tiering, Savings Plans vs Spot, and executive trade-off negotiation.

### [Domain 11: Interview Scenario Scripts](11-interview-scenario-scripts/)
* [`01-aws-python-boto3-interview-scenarios.md`](11-interview-scenario-scripts/01-aws-python-boto3-interview-scenarios.md): Scenario-based Python Boto3 interview scripts (Unattached EBS cleanup with pagination, S3 Public Access Block enforcer, EC2 AMI backup rotation, Security Group synchronizer).
* [`02-bash-systems-and-cloud-automation-scenarios.md`](11-interview-scenario-scripts/02-bash-systems-and-cloud-automation-scenarios.md): Scenario-based Bash scripts (Production EC2 User Data bootstrap with IMDSv2, memory-safe access log analyzer, automated disk/inode alert with SNS).
* [`03-golang-cloud-tooling-scenarios.md`](11-interview-scenario-scripts/03-golang-cloud-tooling-scenarios.md): Scenario-based Go scripts (High-concurrency worker pool health checker with context timeouts, AWS SDK for Go v2 S3 tiering auditor).
* [`04-json-yaml-and-task-definition-wrangling.md`](11-interview-scenario-scripts/04-json-yaml-and-task-definition-wrangling.md): Scenario-based JSON/YAML wrangling (Dynamic ECS task definition image/env injection in CI/CD, multi-document K8s manifest CPU/memory boundary validator).

### [Domain 12: LeetCode & Coding Question Bank](12-leetcode-cloud-engineer-question-bank/)
* [`01-top-cloud-engineer-coding-problems-and-patterns.md`](12-leetcode-cloud-engineer-question-bank/01-top-cloud-engineer-coding-problems-and-patterns.md): High-frequency Cloud Engineer algorithmic problems with Python code, complexity analysis, and cloud contexts (Merge Intervals, Validate IP / Trie Route Table, Log Spike Sliding Window, Top K IPs, DAG Topological Sort).
* [`02-system-and-concurrency-problem-bank.md`](12-leetcode-cloud-engineer-question-bank/02-system-and-concurrency-problem-bank.md): Systems and concurrency problem bank (Token Bucket Rate Limiter, LRU Cache, Producer-Consumer Pipeline with Backpressure).

### [Domain 13: Event Streaming (Kinesis & Apache Kafka MSK)](13-event-streaming-kinesis-and-msk/)
* [`01-event-streaming-architectures-and-use-cases.md`](13-event-streaming-kinesis-and-msk/01-event-streaming-architectures-and-use-cases.md): Event streaming foundations, Shard/Partition hash math, Kinesis Data Streams vs Amazon Data Firehose vs Apache Kafka MSK vs SQS/SNS, and production use cases.
* [`02-kinesis-data-streams-and-firehose-setup.md`](13-event-streaming-kinesis-and-msk/02-kinesis-data-streams-and-firehose-setup.md): Complete Terraform IaC (`main.tf`, `variables.tf`, `outputs.tf`) and AWS Console step-by-step setup for Kinesis Data Streams + Firehose + S3 Data Lake + CloudWatch.
* [`03-amazon-msk-apache-kafka-setup.md`](13-event-streaming-kinesis-and-msk/03-amazon-msk-apache-kafka-setup.md): Complete Terraform IaC and AWS Console clickthrough guide for Multi-AZ Amazon MSK Apache Kafka Cluster with AWS IAM Client Authentication.

### [Domain 14: Production IaC & Console Setups (Core Services)](14-production-iac-and-console-setups/)
* [`01-three-tier-vpc-and-networking-setup.md`](14-production-iac-and-console-setups/01-three-tier-vpc-and-networking-setup.md): Dual Terraform & Console setup for 3-Tier Multi-AZ VPC (Public, App, Isolated DB subnets, Multi-AZ NAT Gateways, S3 Gateway Endpoint, VPC Flow Logs).
* [`02-ec2-auto-scaling-and-alb-setup.md`](14-production-iac-and-console-setups/02-ec2-auto-scaling-and-alb-setup.md): Dual Terraform & Console setup for Internet-Facing ALB, Target Tracking ASG (Graviton), IMDSv2 Launch Template, and Security Group chaining.
* [`03-ecs-fargate-container-service-setup.md`](14-production-iac-and-console-setups/03-ecs-fargate-container-service-setup.md): Dual Terraform & Console setup for ECR, ECS Cluster, Task Definition (Task Role vs Execution Role), and Fargate Service behind ALB.
* [`04-rds-aurora-postgresql-multi-az-setup.md`](14-production-iac-and-console-setups/04-rds-aurora-postgresql-multi-az-setup.md): Dual Terraform & Console setup for Aurora PostgreSQL Serverless v2 Multi-AZ Cluster, Secrets Manager password generation, and Amazon RDS Proxy.
* [`05-amazon-s3-enterprise-security-setup.md`](14-production-iac-and-console-setups/05-amazon-s3-enterprise-security-setup.md): Dual Terraform & Console setup for KMS-CMK encrypted S3 bucket, Block Public Access, TLS-enforcing bucket policy, and automated Glacier Deep Archive tiering.
* [`06-amazon-dynamodb-and-lambda-serverless-setup.md`](14-production-iac-and-console-setups/06-amazon-dynamodb-and-lambda-serverless-setup.md): Dual Terraform & Console setup for On-Demand DynamoDB table (PITR), VPC-attached Lambda function, and API Gateway HTTP API.
* [`07-amazon-eks-cluster-and-karpenter-setup.md`](14-production-iac-and-console-setups/07-amazon-eks-cluster-and-karpenter-setup.md): Dual Terraform & Console setup for Amazon EKS Control Plane (K8s 1.30), Managed Node Group, OIDC provider, and Karpenter autoscaling controller role.

