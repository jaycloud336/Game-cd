# React Game - Continous Deployment with ArgoCD

## Project Overview
This project demonstrates a complete GitOps workflow using **ArgoCD** to deploy a Dockerized React-based game to a local Kubernetes cluster. The setup showcases continuous deployment principles, where application changes are managed through a Git repository and automatically synchronized with the Kubernetes cluster. 

## Architecture Components
- **Application**: React-based Game
- **Container Registry**: Docker Hub (`jaycloud336/malgus-game-app:v1.0.0`)
- **Kubernetes**: Docker Desktop K8s cluster
- **GitOps Tool**: ArgoCD
- **Git Repository**: Github
- **K9s**: (Optional) Terminal-based UI 

---

## Prerequisites & Setup

### 1. Docker Desktop Kubernetes Setup
Ensure Docker Desktop is running with Kubernetes enabled. You can verify the setup using these commands:

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

### 2. ArgoCD Installation
Install ArgoCD onto your cluster using the Helm Chart. This approach creates a customizable template.

```bash
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create ArgoCD namespace
kubectl create namespace argocd

# Generate ArgoCD manifests using latest Helm chart (based on reccomended directory structure)
helm template argo argo/argo-cd --version 8.3.5 --namespace argocd > argo-helm.yaml

# Customize the file path as needed to match your direcoty structure
helm template argo argo/argo-cd --version 8.3.5 --namespace argocd > helm/argocd-helm/argo-helm.yaml

# Apply the generated manifests
kubectl apply -f argo-helm.yaml

# Wait for pods to be ready
kubectl get pods -n argocd --watch
```

### 3. Access ArgoCD UI
To access the ArgoCD dashboard, you'll need to port forward the service and retrieve the initial admin password.

```bash
# Port forward to access ArgoCD UI
kubectl port-forward -n argocd svc/argo-argocd-server 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d


# Access ArgoCD at: https://localhost:8080
- Username: admin
- Password: (decoded from the secret above) # example: luhxWGTaHcolh49
```

## GitOps Workflow

### 1. Project Files & Manifests
The core of this project is the Git repository structure, which separates the application's source code from its Kubernetes manifests.

```
Game-cd/
├── manifests/             # Kubernetes application manifests
│   ├── namespace.yaml     # Defines the application's namespace
│   ├── deployment.yaml    # How to run your app
│   └── service.yaml       # How to expose your app internally
├── argocd/               # ArgoCD application manifest
│   └── application.yaml   # Tells ArgoCD where to find the manifests
└── helm/                 # Helm-generated manifests (NEW)
    └── argocd-helm/      # ArgoCD installation manifests
        └── argo-helm.yaml # Generated Helm template output
```
### Recommended Directory Structure

```bash
# Create local repository structure
mkdir Game-cd
cd Game-cd
mkdir manifests argocd helm
mkdir helm/argocd-helm

# Create manifest files (add content as shown in sections 2-3)
touch manifests/namespace.yaml
touch manifests/deployment.yaml  
touch manifests/service.yaml
touch argocd/application.yaml
```

### 2. The Kubernetes Manifests
These manifests are responsible for defining the desired state of your application within the cluster.

**namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: react-game-cd
```

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-game-deployment
  namespace: react-game-cd
spec:
  selector:
    matchLabels:
      app: react-game
  replicas: 2
  template:
    metadata:
      labels:
        app: react-game
    spec:
      containers:
      - name: react-game-container
        image: jaycloud336/malgus-game-app:v1.0.0
        ports:
        - containerPort: 3000
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: react-game-service
  namespace: react-game-cd
spec:
  selector:
    app: react-game
  ports:
  - port: 80
    protocol: TCP
    targetPort: 3000
```

### 3. ArgoCD Application Manifest
This file is the GitOps manifest that points ArgoCD to your Kubernetes manifests. It is located in the argocd/ directory.

**application.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: react-game-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/jaycloud336/Game-cd # Your GitOps repository URL
    targetRevision: HEAD
    path: manifests # The directory containing your app's YAML files
  destination:
    server: https://kubernetes.default.svc
    namespace: react-game-cd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```
### Github Repo Creation / Intialization

***Required: Create a new repository on GitHub named Game-cd, then proceed with the following commands:***

```bash
# Clone your GitHub repository
git clone https://github.com/jaycloud336/react-game-cd.git
cd react-game-cd

# Push your manifest files into the cloned repository 

# Commit and push the manifest files
git add .
git commit -m "Add ArgoCD application and Kubernetes manifests"
git push origin main
```

### 4. Deployment
Once your files are pushed to the Git repository, you can deploy the ArgoCD application.


```bash
# Apply the ArgoCD application from your repository
kubectl apply -f argocd/application.yaml

# Verify application is created
kubectl get applications -n argocd
```

### 5. Monitor Deployment
ArgoCD will automatically detect and deploy your manifests. You can monitor the process via the CLI or the UI.

```bash
# Watch ArgoCD sync the application
kubectl get applications -n argocd -w

# Check pods in the application namespace
kubectl get pods -n react-game-cd

# Check service status
kubectl get services -n react-game-cd
```

### 6. Access the Application
Since we are using a ClusterIP service, we will use port forwarding to access the game.

```bash
# Port forward the service to your local machine
kubectl port-forward -n react-game-cd svc/react-game-service 3000:80

# Access the application at:
# http://localhost:3000
```