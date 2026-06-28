
## Overview

The Docker Super Mario Project is a modern adaptation of the classic Infinite Mario game, reimagined using HTML5, JavaScript, Canvas, and Audio elements. This project serves as an exemplary platform for learning and implementing GitOps pipelines, offering a hands-on approach to continuous integration and continuous deployment (CI/CD) practices targeting Azure Kubernetes Service (AKS).

---

## Features

- **HTML5 Canvas**: Delivers dynamic, scriptable rendering of 2D shapes and bitmap images.
- **JavaScript**: Ensures interactive game mechanics and responsive design.
- **Audio Elements**: Enhances the gaming experience with authentic sound effects and background music.
- **Docker Integration**: Facilitates the deployment of the application in isolated environments, making it easy to share and scale.
- **GitOps Workflow**: Introduces participants to modern DevOps practices, focusing on automation, monitoring, and version control.

---

## Architecture

```
Developer Workstation
        │
        ▼
  GitHub Repository
  ├── app/              ← Game source code
  ├── Dockerfile        ← Container definition
  ├── terraform/        ← AKS infrastructure as code
  └── gitops/           ← Kubernetes manifests (ArgoCD watches this)
        │
        ▼
  GitHub Actions (CI)
  ├── Build Docker image
  ├── Run security scans (Trivy, Checkov)
  ├── Push to Azure Container Registry (ACR)
  └── Update image tag in gitops/ manifests
        │
        ▼
  ArgoCD (CD — GitOps Controller)
  └── Watches gitops/ branch → Auto-syncs to AKS
        │
        ▼
  Azure Kubernetes Service (AKS)
  ├── Namespace: mario-prod
  ├── Deployment: mario-app (replicas: 2)
  ├── Service: LoadBalancer (port 80)
  └── Ingress: NGINX + TLS (cert-manager)
```

### Infrastructure Components

| Component | Technology | Purpose |
|---|---|---|
| Source Control | GitHub | Code & manifest versioning |
| CI Pipeline | GitHub Actions | Build, scan, push |
| Container Registry | Azure Container Registry (ACR) | Image storage |
| GitOps Controller | ArgoCD | Automated AKS sync |
| Kubernetes Cluster | Azure AKS | Container orchestration |
| IaC | Terraform | AKS cluster provisioning |
| Ingress | NGINX Ingress Controller | HTTP routing + TLS |
| TLS | cert-manager + Let's Encrypt | SSL certificate automation |

---

## Prerequisites

### Local Tools

| Tool | Version | Install |
|---|---|---|
| Git | 2.40+ | [git-scm.com](https://git-scm.com/) |
| Docker Desktop | 24.0+ | [docker.com](https://www.docker.com/) |
| kubectl | 1.28+ | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| Terraform | 1.6+ | [terraform.io](https://www.terraform.io/) |
| Azure CLI | 2.50+ | [docs.microsoft.com](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) |
| Helm | 3.12+ | [helm.sh](https://helm.sh/) |

### Azure Requirements

- Azure Subscription with Contributor role
- Azure Container Registry (ACR) instance
- Service Principal with ACR push permissions
- AKS cluster (provisioned via Terraform — see below)

### GitHub Secrets Required

```
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
ACR_LOGIN_SERVER
ACR_USERNAME
ACR_PASSWORD
```

---

## Pipeline Stages (CI/CD)

### Stage 1 — Continuous Integration (GitHub Actions)

```yaml
Trigger: Push to main / Pull Request

Steps:
  1. Checkout code
  2. Build Docker image
  3. Trivy vulnerability scan (image)
  4. Checkov IaC scan (Terraform)
  5. Login to Azure Container Registry
  6. Push image → ACR (tagged with Git SHA)
  7. Update image tag in gitops/deployment.yaml
  8. Commit & push manifest update → gitops branch
```

### Stage 2 — Continuous Deployment (ArgoCD GitOps)

```yaml
Trigger: Manifest change detected in gitops/ branch

Steps:
  1. ArgoCD detects diff between desired (Git) vs live (AKS) state
  2. ArgoCD syncs → applies updated Deployment manifest to AKS
  3. AKS performs rolling update (zero downtime)
  4. ArgoCD reports sync status (Healthy / Degraded / Progressing)
```

### Pipeline Flow Diagram

```
[Code Push]
     │
     ▼
[GitHub Actions CI]
     ├─ docker build -t mario:$SHA .
     ├─ trivy image mario:$SHA
     ├─ checkov -d terraform/
     ├─ docker push acr.azurecr.io/mario:$SHA
     └─ sed -i "s|image:.*|image: acr.azurecr.io/mario:$SHA|" gitops/deployment.yaml
          │
          ▼
     [Git commit → gitops branch]
          │
          ▼
     [ArgoCD detects drift]
          │
          ▼
     [kubectl apply → AKS]
          │
          ▼
     [Rolling Deployment Complete ✅]
```

---

## How to Run Locally

### Option 1 — Docker (Quickest)

```bash
# Clone the repository
git clone https://github.com/sivaprasad789-ai/GitOps-End-To-End-Project.git
cd GitOps-End-To-End-Project

# Build the Docker image
docker build -t mario-game:local .

# Run the container
docker run -d -p 8080:80 --name mario mario-game:local

# Open in browser
start http://localhost:8080
```

### Option 2 — Deploy to Azure AKS

#### Step 1: Provision AKS with Terraform

```bash
cd terraform/

# Authenticate to Azure
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"

# Initialize and apply
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

#### Step 2: Configure kubectl

```bash
az aks get-credentials \
  --resource-group rg-mario-gitops \
  --name aks-mario-prod \
  --overwrite-existing

kubectl get nodes
```

#### Step 3: Install ArgoCD on AKS

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=LoadBalancer

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

#### Step 4: Register the App in ArgoCD

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mario-game
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/sivaprasad789-ai/GitOps-End-To-End-Project.git
    targetRevision: main
    path: gitops/
  destination:
    server: https://kubernetes.default.svc
    namespace: mario-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

#### Step 5: Access the Game

```bash
# Get the external IP of the LoadBalancer
kubectl get svc -n mario-prod

# Open in browser
start http://<EXTERNAL-IP>
```

---

## Security Hardening (DevSecOps)

### Container Security

```dockerfile
# Use minimal base image
FROM nginx:1.25-alpine

# Run as non-root user
RUN addgroup -S mario && adduser -S mario -G mario
USER mario

# Remove unnecessary packages
RUN apk del --no-cache apk-tools

# Read-only filesystem
# Set in Kubernetes securityContext (see below)
```

### Kubernetes Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

### AKS Network Policies

```yaml
# Deny all ingress by default, allow only from ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mario-netpol
  namespace: mario-prod
spec:
  podSelector:
    matchLabels:
      app: mario
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

### Image Scanning (Trivy)

```bash
# Scan image for vulnerabilities before push
trivy image --severity HIGH,CRITICAL mario-game:local

# Scan IaC
trivy config terraform/

# Fail pipeline on CRITICAL findings
trivy image --exit-code 1 --severity CRITICAL mario-game:local
```

### Checkov IaC Scan (Terraform)

```bash
checkov -d terraform/ \
  --framework terraform \
  --check CKV_AZURE_5,CKV_AZURE_6,CKV_AZURE_7
```

### Security Checklist

| Control | Tool | Status |
|---|---|---|
| Container vulnerability scanning | Trivy | ✅ CI enforced |
| IaC misconfiguration scanning | Checkov | ✅ CI enforced |
| Non-root container execution | Kubernetes securityContext | ✅ Enforced |
| Network segmentation | Kubernetes NetworkPolicy | ✅ Default-deny |
| Secret management | Azure Key Vault + CSI Driver | ✅ No hardcoded secrets |
| TLS encryption in transit | cert-manager + Let's Encrypt | ✅ Auto-renewed |
| RBAC least privilege | AKS + Azure RBAC | ✅ Service Principal scoped |
| Image signing | Azure ACR Content Trust | ⚠️ Recommended |
| Runtime threat detection | Microsoft Defender for Containers | ⚠️ Recommended |

---

## Repository Structure

```
GitOps-End-To-End-Project/
├── app/                        ← Mario game source (HTML5/JS)
├── Dockerfile                  ← Container definition
├── .github/
│   └── workflows/
│       └── ci.yaml             ← GitHub Actions CI pipeline
├── terraform/
│   ├── main.tf                 ← AKS cluster definition
│   ├── variables.tf
│   └── outputs.tf
└── gitops/
    ├── deployment.yaml         ← Kubernetes Deployment
    ├── service.yaml            ← LoadBalancer Service
    ├── ingress.yaml            ← NGINX Ingress + TLS
    └── networkpolicy.yaml      ← Default-deny NetworkPolicy
```

---

---

*Learning GitOps the right way — secure by default, automated by design.*
