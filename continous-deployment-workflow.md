# React Game - Continous Deployment with ArgoCD

## Project Overview
This project demonstrates a complete GitOps workflow using **ArgoCD** to deploy a Dockerized React-based game to a local Kubernetes cluster. The setup showcases continuous deployment principles, where application changes are managed through a Git repository and automatically synchronized with the Kubernetes cluster. 

## Architecture Components
- **Application**: React-based Game
- **Container Registry**: Docker Hub (`jaycloud336/malgus-game-app:cicd`)
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

# Customize the file path as needed to match your directory structure
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
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml
echo <encrypted password> | base64 -d


# Access ArgoCD at: https://localhost:8080
- Username: admin
- Password: (decoded from the secret above) # example: luhxWGTaHcolh49
```

## GitOps Workflow

### 1. Project Files & Manifests
Project Git repository structure including Kubernetes manifests.

```
Game-cd/
├── manifests/             # Kubernetes application manifests
│   ├── namespace.yaml     # Defines the application's namespace
│   ├── deployment.yaml    # App deployment 
│   └── service.yaml       # App service exposure
├── argocd/               # ArgoCD application manifest
│   └── application.yaml   # Tells ArgoCD where to find the manifests
└── helm/                 # Helm-generated manifests
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
        image: jaycloud336/malgus-game-app:cicd
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: react-game-service
  namespace: react-game-cd # Add this line
spec:
  selector:
    app: react-game
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
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
## *Note: Three Different "Namespaces"*

1. **ArgoCD's Own Namespace**
   - Name: `argocd` | Contains: ArgoCD pods/services | How: Manual `kubectl create namespace argocd` for Helm install
   - *View in: `kubectl get pods -n argocd` or ArgoCD UI server components*

2. **Application Definition Namespace**  
   - Name: `argocd` (same) | Contains: Application YAML objects | How: Auto when `kubectl apply -f application.yaml`
   - *View in: `kubectl get applications -n argocd` or ArgoCD UI applications list*

3. **Target Deployment Namespace**
   - Name: `react-game-cd` | Contains: Your app pods/services | How: Manual `manifests/namespace.yaml` deployed by ArgoCD
   - *View in: `kubectl get pods -n react-game-cd` or ArgoCD UI application resources tree*

**Key:** ArgoCD lives in `argocd`, manages apps in other namespaces

### Github Repo Creation / Intialization

***Required: Create a new repository on GitHub named Game-cd, then proceed with the following commands:***

```bash
# Clone your GitHub repository (or create from scratch in githiub as necesary)
git clone https://github.com/jaycloud336/react-game-cd.git
cd react-game-cd

# Commit and push the manifest files into the repository
git add .
git commit -m "Add ArgoCD application and Kubernetes manifests"
git push origin main
```

### 4. Deployment
Once your files are pushed to the Git repository, you can apply the ArgoCD application manifest.

```bash
# Apply the ArgoCD application from your repository
kubectl apply -f argocd/application.yaml

# Verify application is created
kubectl get applications -n argocd
```

### 5. Monitor Deployment
ArgoCD will automatically detect and deploy the manifests. 

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
 http://localhost:3000

```

### 7. Docker Desktop Single-Node Cluster Teardown & Restore

```bash
# Reset Kubernetes cluster in Docker Desktop
# Option 1: Clean removal (preserves local files)
kubectl delete -f argocd/application.yaml
kubectl delete namespace react-game-cd
kubectl delete namespace argocd

# Option 2: Nuclear option - Reset entire cluster
Docker Desktop → Settings → Kubernetes → Reset Kubernetes Cluster
# ⚠️ This removes ALL cluster resources but keeps local files

# 1. Reinstall argocd application
bash# 1. Reinstall ArgoCD
kubectl create namespace argocd
kubectl apply -f helm/argocd-helm/argo-helm.yaml

# 2. Re-apply application.yaml
kubectl apply -f argocd/application.yaml
# Restore Access
bash# Get new ArgoCD admin password (changes on each install)
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml
echo <encrypted password> | base64 -d

# 3. Access ArgoCD UI
kubectl port-forward -n argocd svc/argocd-server 8080:443

# 4. Access your application (after ArgoCD syncs)
kubectl port-forward -n react-game-cd svc/react-game-service 3000:80
```
#### Important Note: When Restarting Configuration Image & Port Compatibility

*This project references two different images in order to demonstrated CD principles:*

*• `jaycloud336/malgus-game-app:v1.0.0` - Runs on port 3000*
*• `jaycloud336/malgus-game-app:cicd` - Runs on port 80 (nginx)*

*Ensure your manifest files match the image you want to deploy:*

*• For `:v1.0.0` → `containerPort: 3000` and `targetPort: 3000`*
*• For `:cicd` → `containerPort: 80` and `targetPort: 80`*

*Note: ArgoCD will automatically detect pushed Git repository changes and redeploy application within 1-3 minutes. Everything returns to the exact state defined in Git.*