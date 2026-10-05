# Kubernetes Architecture & Production Manifest Engineering

Kubernetes (K8s) has become the de facto operating system for cloud-native infrastructure. As a cloud engineer, you must understand both the distributed control plane mechanics that keep a cluster in consensus and how to engineer secure, production-grade YAML manifests.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Automated Mega-Warehouse
Imagine a high-volume automated distribution warehouse:
* **The Control Plane (The Headquarters Tower):**
  * **`kube-apiserver` (The Front Receptionist & Order Desk):** The only person allowed to receive customer requests. They check your ID badges and stamp approvals on paperwork.
  * **`etcd` (The Central Indelible Ledger):** The indestructible bank vault containing the official ledger recording where every item is stored. If an item isn't written in the ledger, it doesn't exist.
  * **`kube-scheduler` (The Logistics Planner):** Looks at new incoming cargo boxes, checks which warehouse rooms have free shelf space and strong enough floors, and assigns the box to Room 3.
  * **`kube-controller-manager` (The Plant Supervisors):** Constantly walks the warehouse floor comparing reality against the ledger: *"The ledger says we should have 4 security guards standing at Door B; I only see 3, so I will hire another guard right now."*
* **The Worker Nodes (The Warehouse Rooms):**
  * **`kubelet` (The Room Foreman):** Lives in each room. Receives orders from headquarters, commands the robotic arms (**Container Runtime / containerd**) to unpack the containers, and reports back on temperature and health.
  * **`kube-proxy` (The Internal Intercom & Conveyor Switch):** Programs the automated conveyor belts (**iptables / IPVS**) so packages slide seamlessly between workbenches.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### The Control Plane & Worker Node Architecture

```
========================= KUBERNETES CONTROL PLANE =========================
               +---------------------------------------+
               |             kube-apiserver            |<==== kubectl / Helm
               +---+---------------+---------------+---+
                   |               |               |
                   v               v               v
            +------------+  +--------------+  +---------------------------+
            |    etcd    |  |kube-scheduler|  |  kube-controller-manager  |
            |(Consensus) |  | (Placement)  |  | (Reconciliation Loop)     |
            +------------+  +--------------+  +---------------------------+
=============================================================================
                                   | (HTTPS)
                                   v
========================== WORKER NODE 1 ====================================
+---------------------------------------------------------------------------+
| [ kubelet ] <==== Reports Node & Pod Status =====> [ kube-apiserver ]     |
|      |                                                                    |
|      v                                                                    |
| [ containerd (CRI) ] === Spawns ===> [ Pod: Container A | Container B ]   |
|                                                                           |
| [ kube-proxy ] === Programs ===> [ Linux iptables / IPVS Network Rules ]  |
+---------------------------------------------------------------------------+
```

---

### Engineering Production Kubernetes Manifests

In interviews, you will often be asked to write or review a Kubernetes manifest. A production manifest must include:
1. Resource requests and limits (preventing node starvation).
2. Liveness and Readiness probes (enabling zero-downtime rolling updates).
3. Security Context (running as non-root with read-only root filesystems).
4. Pod Anti-Affinity (spreading replicas across distinct nodes and zones).

```yaml
# ==============================================================================
# Production Deployment Manifest: payment-service
# ==============================================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  labels:
    app.kubernetes.io/name: payment-service
    app.kubernetes.io/part-of: payment-system
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%         # Can spin up 1 extra pod during deployment
      maxUnavailable: 0     # ZERO pods allowed down during rolling update!
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      # Spread pods across distinct Availability Zones and Nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: ["payment-service"]
                topologyKey: "topology.kubernetes.io/zone"
      
      # Pod Security Hardening
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
      
      containers:
        - name: payment-api
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-api:v1.4.2
          imagePullPolicy: IfNotPresent
          
          ports:
            - name: http-server
              containerPort: 8080
              protocol: TCP
          
          # Compute Boundaries (Requests = Reserved; Limits = Hard Cap)
          resources:
            requests:
              cpu: "250m"      # 0.25 vCPU
              memory: "512Mi"  # 512 MB RAM
            limits:
              cpu: "1000m"     # 1 full vCPU
              memory: "1Gi"    # OOM-Killed if it exceeds 1 GB
          
          # Zero-Downtime Health Probes
          readinessProbe:
            httpGet:
              path: /ready
              port: http-server
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
            
          livenessProbe:
            httpGet:
              path: /healthz
              port: http-server
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: password
---
# ==============================================================================
# Production ClusterIP Service Manifest
# ==============================================================================
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: payment-service
  ports:
    - name: http
      port: 80
      targetPort: http-server
---
# ==============================================================================
# Horizontal Pod Autoscaler (HPA) Manifest
# ==============================================================================
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

##### 3. Visual Flow Diagram: Liveness vs. Readiness Probe Execution

```
[ Pod Container Launches ]
           |
           v
[ Readiness Probe Pings /ready ]
  |-- Returns 503 (App is warming up caches):
  |   ==> Kube-proxy REMOVES Pod from Service endpoints! No user traffic sent!
  |-- Returns 200 OK (App is ready):
  |   ==> Kube-proxy ADDS Pod to Service endpoints! User traffic flows!
           |
           v
[ Liveness Probe Pings /healthz periodically ]
  |-- Returns 200 OK: Everything healthy.
  `-- Returns 500 or Times Out 3 consecutive times:
      ==> Kubelet KILLS the container and restarts it automatically!
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"What happens if CPU limit is exceeded vs. Memory limit?"**
  * **CPU Limit Exceeded:** CPU is a compressible resource. If your container tries to use 2000m on a 1000m limit, Kubernetes **throttles the CPU clock cycles (CFS quota)**. The application slows down, but it does **not** crash.
  * **Memory Limit Exceeded:** Memory is an incompressible resource. If your container reaches 1025 MiB on a 1024 MiB limit, the Linux kernel immediately invokes the Out-Of-Memory killer and terminates the process with **`Exit Code 137 (OOMKilled)`**!

---

##### 5. Quick Hands-On Check

Check why a pod is stuck or restarting using `kubectl describe` and inspect its termination reason:

```bash
# Query pod status and inspect event stream for OOMKilled or probe failures
kubectl describe pod payment-service-6789abcdef-xyz -n production
```
