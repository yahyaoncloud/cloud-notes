# Enterprise CI/CD Pipelines: GitHub Actions & Jenkins Architecture

Continuous Integration and Continuous Delivery (CI/CD) pipelines automate the journey of software from a developer's `git push` to live production environments. This guide compares **GitHub Actions** and **Jenkins**, demonstrates how to authenticate securely to AWS using **OpenID Connect (OIDC)** without hardcoding secrets, and details production pipeline standards.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Automated Automotive Assembly Line
Imagine a vehicle factory:
* **The Developer Commit:** A designer drops a newly engineered car door onto the conveyer belt.
* **The CI Pipeline (Automated Crash Tests & Paint Scanners):**
  * Robotic arms immediately inspect the door dimensions (**Linters & Unit Tests**).
  * X-ray scanners check the metal density for microscopic cracks (**Security Vulnerability Scanning & SAST**).
  * If the door fails by 1 millimeter, the conveyor belt stops immediately with an amber flashing light, rejecting the piece before it ever reaches the car frame.
* **The CD Pipeline (Automated Chassis Mounting):**
  * If tests pass, automated robotic cranes carry the door to the live assembly room, screw it onto the vehicle chassis (**Staging / Production Deployment**), and run a quick electronic sensor check to ensure the window rolls down cleanly (**Synthetic Health Check**).

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Architectural Comparison: GitHub Actions vs. Jenkins

```
+------------------------------------+------------------------------------+
| GITHUB ACTIONS                     | JENKINS (Self-Hosted)              |
+------------------------------------+------------------------------------+
| Architecture: Fully managed SaaS   | Architecture: Self-hosted master   |
| (or self-hosted runner agents)     | and agent controller cluster       |
+------------------------------------+------------------------------------+
| Configuration: Declarative YAML    | Configuration: Groovy DSL          |
| inside `.github/workflows/`        | (`Jenkinsfile` - Declarative/Script|
+------------------------------------+------------------------------------+
| Secrets: OIDC AWS STS integration  | Secrets: Jenkins Credentials Store |
| (Zero permanent credentials!)      | or HashiCorp Vault plugin          |
+------------------------------------+------------------------------------+
| Maintenance: Zero server patching  | Maintenance: High (Must patch JVM, |
| for GitHub-hosted runners          | Jenkins core, and 40+ plugins)     |
+------------------------------------+------------------------------------+
| Scalability: Auto-scales instantly | Scalability: Ephemeral K8s agent   |
| via cloud runner pool              | pods via Kubernetes plugin         |
+------------------------------------+------------------------------------+
```

---

### Keyless AWS Authentication: GitHub Actions via OIDC

**Never store static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in GitHub Secrets!** If an attacker breaches the repository or an open-source dependency exfiltrates secrets, your entire cloud account is compromised.
Use **OpenID Connect (OIDC)** and **AWS STS AssumeRoleWithWebIdentity**:

```
[ GitHub Actions Runner ]
           |
           | (1) Requests a signed cryptographic JWT OIDC Token from GitHub
           v
[ GitHub OIDC Token Mint: token.actions.githubusercontent.com ]
           |
           | (2) Returns signed JWT carrying: repo, branch, commit SHA
           v
[ AWS Security Token Service (STS) ]
           |
           | (3) Verifies GitHub's cryptographic signature against IAM OIDC Provider
           | (4) Validates Trust Policy: repo:my-org/my-repo:ref:refs/heads/main
           v
[ AWS STS issues short-lived (15-min) temporary credentials to the runner! ]
```

#### Production GitHub Actions Workflow:
```yaml
name: Production Deployment Pipeline

on:
  push:
    branches:
      - main

# Request permissions to fetch GitHub OIDC JWT token
permissions:
  id-token: write
  contents: read

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Authenticate to AWS via OIDC (No Static Keys!)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeploymentRole
          aws-region: us-east-1
          role-session-name: GitHubActions-${{ github.sha }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, Tag, and Push Docker Image
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          REPOSITORY: payment-api
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
          docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG

      - name: Deploy Task to Amazon ECS
        run: |
          aws ecs update-service \
            --cluster production-cluster \
            --service payment-service \
            --force-new-deployment
```

---

### Enterprise Jenkins Pipeline Architecture

In high-compliance on-premises environments, Jenkins is run on EC2 or Kubernetes using ephemeral Docker agents:

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: jenkins-agent
spec:
  containers:
  - name: aws-tooling
    image: amazon/aws-cli:latest
    command: ['cat']
    tty: true
'''
        }
    }
    
    stages {
        stage('Code Quality & SAST') {
            steps {
                container('aws-tooling') {
                    echo "Running security scans..."
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                container('aws-tooling') {
                    sh 'aws ecs update-service --cluster staging-cluster --service web-app --force-new-deployment'
                }
            }
        }
    }
    
    post {
        failure {
            echo "Deployment failed! Alerting Slack on-call channel..."
        }
    }
}
```

---

##### 3. Visual Flow Diagram: CI/CD Guardrail Stages

```
[ Developer Push ]
       |
       v
+---------------------------------------------------------------------------------+
| STAGE 1: LINT & STATIC ANALYSIS (tfsec, checkov, sonarqube, bandit)            |
+---------------------------------------+-----------------------------------------+
                                        | (Must Pass)
                                        v
+---------------------------------------------------------------------------------+
| STAGE 2: CONTAINER IMAGE BUILD & SECURITY SCAN (Trivy / ECR Basic Scan)        |
+---------------------------------------+-----------------------------------------+
                                        | (Zero Critical Vulnerabilities)
                                        v
+---------------------------------------------------------------------------------+
| STAGE 3: EPHEMERAL STAGING DEPLOYMENT & INTEGRATION TESTS                       |
+---------------------------------------+-----------------------------------------+
                                        | (Automated Tests Pass)
                                        v
+---------------------------------------------------------------------------------+
| STAGE 4: PRODUCTION CANARY DEPLOYMENT (10% Traffic -> 100% Traffic)             |
+---------------------------------------------------------------------------------+
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Why did my GitHub Actions runner fail to assume the IAM role?"**
  * *The Trap:* In the IAM Role's **Trust Policy**, the `StringEquals` condition strictly inspects the GitHub `sub` (subject) claim. If your branch is `main` but the trust policy specifies `refs/heads/master`, or if your repo name has a typo, STS rejects the token instantly with `AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity`!
