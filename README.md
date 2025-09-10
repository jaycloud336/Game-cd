# React-Based Game - CD GitOps POC

## Overview
This proof of concept (POC) demonstrates key GitOps principles by showcasing an automated continuous deployment (CD) workflow. The project utilizes a modified version of an existing containerized React application and demonstrates how to configure and deploy it using ArgoCD. It synchronizes the desired state of a Kubernetes cluster with a Git repository, ensuring that every change to the source of truth triggers a reliable and automated deployment.

## Technical Scope & Business Value
-   **Declarative Infrastructure**: The project uses Kubernetes manifests to define its deployment and service type.
-   **Automated Deployment**: It implements an automated Git-Ops triggered continuous deployment (CD) workflow.
-   **Scalable Architecture**: The architectural pattern demonstrated here is a foundational template for building more complex and comprehensive CI/CD pipelines in professional environments.

---

## Git-Ops Workflow
Developer (Manual) ➡️ Git Repository (Manifests) ➡️ ArgoCD Controller
|
v
Kubernetes Cluster
(Docker Desktop)

---

## Repository Structure
```
react-game-cd/
├── app/                    # React application source code
│   ├── src/               # React components and logic
│   ├── public/            # Static assets
│   ├── package.json       # Node.js dependencies
│   ├── package-lock.json  # Dependency lock file
│   └── Dockerfile         # Container build instructions
├── manifests/              # Kubernetes application manifests
│   ├── namespace.yaml     # Application namespace
│   ├── deployment.yaml    # Pod and container specifications
│   └── service.yaml       # Network exposure configuration
├── argocd/               # ArgoCD application manifests
│   └── application.yaml   # ArgoCD Application resource
└── README.md              # Project documentation
```

---

## Architecture Diagram
```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────────┐
│ Developer   │───▶│  Git Repository │───▶ │  ArgoCD Controller  │
│  (Manual)   │     │    (Manifests)  │     │(Reconciliation Loop)│
└─────────────┘     └─────────────────┘     └─────────────────────┘
                                                      │
                                                      ▼
           ┌────────────────────────────────────────────────────────┐
           │            Docker Desktop - Kubernetes Node            │
           │ (Single-node Cluster running on a local machine)       │
           │ ┌──────────────────┐   ┌───────────────────────────┐   │
           │ │   ArgoCD Pods    │   │      Application Pods     │   │
           │ │ (argocd-server)  │   │     (react-game-cd)       │   │
           │ └──────────────────┘   └───────────────────────────┘   │
           │           ▲                       │                    │
           │           │                 ┌───────────────┐          │
           │ ┌─────────┴─────────┐       │ ClusterIP     │          │
           │ │   ArgoCD UI       │◀────▶│ Service       │          │
           │ │  (localhost:8080) │       │ (10.96.X.X)   │          │
           │ └─────────┬─────────┘       └───────────────┘          │
           │           │ Port Forwarding      ▲ (Port Forwarding)   │
           └───────────┴───────────────────────┘────────────────────┘
                       │
             ┌──────────────────┐
             │   User Access    │
             │  (Browser/CLI)   │
             └──────────────────┘
```

---

## Technology Stack
-   **Application**: React 16.12.0 (Tetris-style game)
-   **Container**: Docker (/w nginx serving static files)
-   **Orchestration**: Kubernetes (Docker Desktop)
-   **GitOps**: ArgoCD (for automated deployment)
-   **Registry**: Docker Hub (image repository)

---

## Workflow Process

1.  **Manual CI (Image Selection):** The developer uses an existing, pre-built Docker image from a container registry. This step is intentionally manual to isolate and highlight the subsequent GitOps process.
2.  **Triggering CD with Git:** A declarative change, such as updating the image tag in the `deployment.yaml` manifest, is committed and pushed to the Git repository.
3.  **Automated Deployment (ArgoCD):** ArgoCD continuously monitors the Git repository. Upon detecting the manifest change, ArgoCD automatically pulls and applies the new configuration to the Kubernetes cluster. The new Pods are deployed, ensuring the cluster's live state matches the desired state in Git.
4.  **Application Access:** The newly deployed application is made accessible via `kubectl port-forward` for final validation.

---

## GitOps Principles Demonstrated
-   **Declarative Configuration**: The state of the system is defined in a declarative format stored in Git.
-   **Version Control**: Every change to the deployment configuration is tracked and auditable via Git.
-   **Automated Synchronization**: ArgoCD acts as a reconciliation agent, ensuring the cluster's live state matches the desired state in Git.
-   **Separation of Concerns**: The project clearly delineates the build process from the deployment process.

---

## Prerequisites
-   Docker Desktop with Kubernetes enabled
-   `kubectl` configured for local cluster
-   Git for version control
-   Familiarity with containerization and Kubernetes concepts

---

## Project Outcomes
-   **Successful Deployment**: The application consistently deploys and updates based on manifest changes.
-   **Clear Workflow**: The project provides a tangible, easy-to-follow example of a GitOps workflow.
-   **Troubleshooting**: Ability to use the ArgoCD UI to monitor and troubleshoot deployment issues.

---

## Scope & Constraints
-   **Focus on CD**: Demonstrates core functionality of a GitOps CD controller.
-   **Local Environment**: Intended for a local Kubernetes cluster (Docker Desktop) for ease of setup.
-   **Basic Configuration**: Minimal ArgoCD setup to focus on core principles rather than advanced features.


## Project Challenges
This project's focus on a minimal, CD-only workflow on a local environment introduced a specific set of challenges:

* **Port Forwarding via WSL2:** Due to Docker Desktop's reliance on WSL2 for its Kubernetes cluster on Windows, port forwarding was the most practical method for accessing the application. While effective for a POC, this approach is not scalable for production environments, where bare metal or Cloud Service Provider (CSP) clusters would utilize more robust networking solutions like **LoadBalancers** or **Ingress controllers**.
