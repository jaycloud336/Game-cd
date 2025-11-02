# React-Based Game - CD GitOps POC

## Overview
Proof of concept (POC) demonstrates key GitOps principles by showcasing an automated continuous deployment (CD) workflow. The project utilizes a modified version of an existing containerized React application and demonstrates how to configure and deploy it using ArgoCD. It synchronizes the desired state of a Kubernetes cluster with a Git repository, ensuring that every change to the source of truth triggers a reliable and automated deployment.

## Technical Scope & Business Value
-   **Declarative Infrastructure**: The project uses Kubernetes manifests to define its deployment and service type.
-   **Automated Deployment**: It implements an automated Git-Ops triggered continuous deployment (CD) workflow.
-   **Scalable Architecture**: The architectural pattern demonstrated here is a foundational template for building more complex and comprehensive CI/CD pipelines in professional environments.

---

## Technology Stack
-   **Application**: React 16.12.0 (Tetris-style game)
-   **Container**: Docker (w/ nginx serving static files)
-   **Orchestration**: Kubernetes (Docker Desktop)
-   **GitOps**: ArgoCD (installed via Helm template method for reliability)
-   **Registry**: Docker Hub (image repository)

---

## Git-Ops Workflow
Developer (Manual) ➡️ Git Repository (Manifests) ➡️ ArgoCD Controller
|
v
Kubernetes Cluster
(Docker Desktop)

---

## Prerequisites
-   Docker Desktop with Kubernetes enabled
-   `kubectl` configured for local cluster
-   Helm CLI for ArgoCD installation
-   Git for version control
-   Familiarity with containerization and Kubernetes concepts

---

## Installation & Setup

### 1. Cluster Verification
Ensure Docker Desktop is running with Kubernetes enabled:

```bash
# Verify cluster status
kubectl version
kubectl cluster-info
kubectl config get-contexts
kubectl config use-context docker-desktop

# Check cluster health
kubectl get nodes
kubectl get pods -n kube-system
```

### 2. ArgoCD Installation (Helm Template Method)
Install ArgoCD using Helm templates for improved reliability:

```bash
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create ArgoCD namespace
kubectl create namespace argocd

# Generate ArgoCD manifests using Helm template
helm template argocd argo/argo-cd --version 8.3.5 --namespace argocd > helm/argocd-helm/argo-helm.yaml

# Apply the generated manifests
kubectl apply -f helm/argocd-helm/argo-helm.yaml

# Wait for pods to be ready
kubectl get pods -n argocd --watch
```

### 3. Access ArgoCD UI
```bash
# Port forward to access ArgoCD UI
kubectl port-forward -n argocd svc/argocd-server 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Access ArgoCD at: https://localhost:8080
- Username: admin
- Password: (decoded from the secret above)

### 4. Repository Setup
```bash
# Create local repository structure
mkdir Game-cd
cd Game-cd
mkdir manifests argocd helm
mkdir helm/argocd-helm

# Create manifest files
touch manifests/namespace.yaml
touch manifests/deployment.yaml  
touch manifests/service.yaml
touch argocd/application.yaml
```

### 5. Deploy Application
```bash
# Apply the ArgoCD application
kubectl apply -f argocd/application.yaml

# Verify application is created
kubectl get applications -n argocd

# Monitor deployment
kubectl get pods -n react-game-cd
kubectl get services -n react-game-cd
```

### 6. Access Application
```bash
# Port forward the service to access the game
kubectl port-forward -n react-game-cd svc/react-game-service 3000:80

# Access the application at:
# http://localhost:3000
```

---

## Workflow Process

1.  **Manual CI (Image Selection):** Utilize existing, pre-built Docker image from a container registry. This step is intentionally manual to isolate and highlight the subsequent GitOps process.
2.  **Triggering CD with Git:** A declarative change, such as updating the image tag in the `deployment.yaml` manifest, is committed and pushed to the Git repository.
3.  **Automated Deployment (ArgoCD):** ArgoCD continuously monitors the Git repository. Upon detecting the manifest change, ArgoCD automatically pulls and applies the new configuration to the Kubernetes cluster. The new Pods are deployed, ensuring the cluster's live state matches the desired state in Git.
4.  **Application Access:** The newly deployed application is made accessible via `kubectl port-forward` for final validation.

---

## GitOps Principles
-   **Declarative Configuration**: The state of the system is defined in a declarative format stored in Git.
-   **Version Control**: Every change to the deployment configuration is tracked and auditable via Git.
-   **Automated Synchronization**: ArgoCD acts as a reconciliation agent, ensuring the cluster's live state matches the desired state in Git.
-   **Separation of Concerns**: The project clearly delineates the build process from the deployment process.

---

## Project Outcomes
-   **Successful Deployment**: The application consistently deploys and updates based on manifest changes.
-   **Clear Workflow**: The project provides a tangible, easy-to-follow example of a GitOps workflow.
-   **Troubleshooting**: Ability to use the ArgoCD UI to monitor and troubleshoot deployment issues.
-   **Installation Reliability**: Helm template method provides more stable ArgoCD installation compared to raw YAML manifests.

---

## Scope & Constraints
-   **Focus on CD**: Demonstrates core functionality of a GitOps CD controller.
-   **Local Environment**: Intended for a local Kubernetes cluster (Docker Desktop) for ease of setup.
-   **Basic Configuration**: Minimal ArgoCD setup to focus on core principles rather than advanced features.

## Challenges:
This project's focus on a minimal, CD-only workflow on a local environment introduced a specific set of challenges:

* **Port Forwarding via WSL2:** Due to Docker Desktop's reliance on WSL2 for its Kubernetes cluster on Windows, port forwarding was the most practical method for accessing the application. While effective for a POC, this approach is not scalable for production environments, where bare metal or Cloud Service Provider (CSP) clusters would utilize more robust networking solutions like **LoadBalancers** or **Ingress controllers**.

* **ArgoCD Installation Methods:** Initial attempts using raw YAML manifests resulted in repo-server connectivity issues. The Helm template method proved more reliable for Docker Desktop environments, providing better component coordination and service mesh configuration.