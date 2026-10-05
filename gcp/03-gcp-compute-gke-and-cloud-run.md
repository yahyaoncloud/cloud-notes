# GCP Compute Engine, GKE & Cloud Run Architecture

Google Cloud provides three primary compute options spanning Infrastructure-as-a-Service, Container Orchestration, and Serverless Containers.

---

## 1. Compute Paradigm Comparison

| Feature | Compute Engine (GCE) | Google Kubernetes Engine (GKE) | Cloud Run |
| :--- | :--- | :--- | :--- |
| **Model** | IaaS Virtual Machines | Managed Kubernetes (PaaS/Container) | Serverless Container Runtime |
| **Scaling** | Managed Instance Groups (MIGs) | Cluster Autoscaler + HPA + Karpenter/GKE Auto | Concurrency-based scale-to-zero |
| **Startup Latency** | 30 - 90 seconds | Pod spin-up (seconds) | Sub-second cold starts |
| **Pricing** | Per-second VM billing | Node VM costs + $0.10/hr cluster fee (Standard) | Per-millisecond vCPU/memory during request |
| **Best For** | Legacy stateful apps, custom OS kernels | Complex microservices, multi-container pods, stateful workloads | Stateless web services, APIs, webhook handlers, event processors |

---

## 2. GKE Standard vs. GKE Autopilot

* **GKE Autopilot**: Google provisions, hardens, and manages worker nodes automatically. You only pay for the requested Pod resources (vCPU, Memory, Ephemeral Storage). Node pools, OS patching, and security baselines are fully automated by Google.
* **GKE Standard**: You manage worker node pools, machine types, custom OS images, and cluster upgrades manually. Gives full access to node privileges (e.g. hostNetwork, privileged containers, custom daemonsets).

### Production Autopilot Manifest Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payments-api
  template:
    metadata:
      labels:
        app: payments-api
    spec:
      containers:
      - name: payments-api
        image: gcr.io/yahya-workload-prod/payments-api:v1.2.0
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

---

## 3. Cloud Run Enterprise Deployment

Cloud Run runs any container image that listens on HTTP/HTTPS or gRPC:

```bash
# Build container and push to Artifact Registry
gcloud builds submit --tag us-central1-docker.pkg.dev/my-project/my-repo/api:v1

# Deploy to Cloud Run with concurrency tuning and min-instances to prevent cold starts
gcloud run deploy api \
  --image us-central1-docker.pkg.dev/my-project/my-repo/api:v1 \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 50 \
  --concurrency 80 \
  --cpu 1 \
  --memory 512Mi \
  --set-env-vars ENV=production
```
