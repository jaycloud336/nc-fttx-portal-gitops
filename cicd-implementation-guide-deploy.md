# Phase 4: GitOps & Continuous Deployment (CD) with ArgoCD

The CD pipeline provides automated Deployment stages for the containerize dpplciation application. This CD repo contains the Kubernetes manifests files. ArgoCD will be installed in the cluster and monitor manifest changes pushed to the associated GitHub repo foudn here: https://github.com/jaycloud336/nc-fttx-portal
Clone bith repos and provide the necessary secrets and credentials in order to run the full pipeline.

If you'd like to build your own repo from scratch follow the instrctions below:

### Step 1: Create a Dedicated GitOps Repository
Repository for all Kubernetes manifest files ie: "single source of truth" 

```Bash
# Create a new repository on GitHub named "nc-fttx-portal-gitops"
git init
cd ~/desired directory path/nc-fttx-portal-gitops
git add .
git commit -m "Initial commit of GitOps manifests"
git remote add origin https://github.com/<repo-name>/nc-fttx-portal-gitops.git
git push -u origin main
```

## Phase 2: Git Repository Structure
A key aspect of GitOps is a dedicated Git repository for your Kubernetes manifests. You've already structured this correctly.

nc-fttx-portal-gitops/
├── argocd/                 # ArgoCD application manifests
│   ├── application.yaml    # Main application definition
├── manifests/              # Application Kubernetes manifests
│   ├── deployment.yaml     # App deployment (UPDATED with annotations)
│   ├── namespace.yaml      # Application namespace
│   └── service.yaml        # App service exposure
└── README.md               # This heirchy can be updated with monitoring setup

## Step 3: Add Kubernetes Manifests to the GitOps Repository
Add the manifests you provided to this new repository, following the structure:
nc-fttx-portal-gitops/manifests/ and nc-fttx-portal-gitops/argocd/.
**If monitoring is desired, also create nc-fttx-portal-gitops/monitoring/**

## Prerequisites & Initial Setup

1. Verify Kubernetes & ArgoCD
Ensure the Kubernetes cluster is active and that ArgoCD is properly installed and accessible.

```Bash
# Verify Kubernetes client and server versions
kubectl version --short

# Verify Kubernetes cluster status and endpoints
kubectl cluster-info

# Verify correct cluster context
kubectl config current-context

# Optional: List all available contexts if needed
kubectl config get-contexts
```

```Bash

# Create K8s Manifest files:

1. ArgoCD Application Manifest (argocd/application.yaml)
This is the manifest that tells ArgoCD what to deploy and where. The syncPolicy ensures automatic updates (selfHeal and prune) and provides a solid retry mechanism.
```

```YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nc-fttx-portal
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/jaycloud336/nc-fttx-portal-gitops.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: nc-fttx-portal
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 3
```

2. Kubernetes Manifests (manifests/)
These manifests define the actual application to be deployed. They will be managed and synced by ArgoCD.

**namespace.yaml:**

```YAML
apiVersion: v1
kind: Namespace
metadata:
  name: nc-fttx-portal
  labels:
    app: nc-fttx-portal
    environment: local
    managed-by: argocd
deployment.yaml: Defines your application's deployment. Your manifest includes important details like resource limits and a secure securityContext, which is excellent.
```

**service.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nc-fttx-portal-service
  namespace: nc-fttx-portal
  labels:
    app: nc-fttx-portal
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: nc-fttx-portal
```

**deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nc-fttx-portal-deployment
  namespace: nc-fttx-portal
  labels:
    app: nc-fttx-portal
    version: "1.0.0"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nc-fttx-portal
  template:
    metadata:
      labels:
        app: nc-fttx-portal
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: nc-fttx-portal
        image: jaycloud336/nc-fttx-portal:latest
        ports:
        - containerPort: 8080
          protocol: TCP
          name: http
        env:
        - name: PORT
          value: "8080"
        - name: GIN_MODE
          value: "debug"
        - name: NODE_ENV
          value: "development"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
```

With the manifests installed in the Git repository, the final step is to configure ArgoCD and inform it of the application.

### 2. ArgoCD Installation
Install ArgoCD onto your cluster using the Helm Chart. This approach creates a customizable template.

```bash
# Add ArgoCD Helm repository if necessary (if already installed, this step can be skipped)
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create ArgoCD namespace
kubectl create namespace argocd

# Generate ArgoCD manifests using latest Helm chart (based on recomended directory structure)
helm template argo argo/argo-cd --version 8.3.5 --namespace argocd > argo-helm.yaml

# Customize the file path as needed to match your directory structure
helm template argo argo/argo-cd --version 8.3.5 --namespace argocd > helm/argocd-helm/argo-helm.yaml

# Apply the generated manifests
kubectl apply -f argo-helm.yaml

# Wait for pods to be ready
kubectl get pods -n argocd --watch
```

### 3. Access ArgoCD UI
To access the ArgoCD dashboard, you'll need to port-forward the service and retrieve the initial admin password.

```bash
# Port forward to access ArgoCD UI
kubectl port-forward -n argocd svc/argo-argocd-server 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml
echo <encrypted password> | base64 -d

# Example:
e0Ls3xG9s4a-xvNm, 62xpCAldlNMqBPim, vWHh-PePrtEAv11p, M0-CkExCKDC704py

# Access ArgoCD at: https://localhost:8080
- Username: admin
- Password: (decoded from the secret above) # example: luhxWGTaHcolh49
```


1. Apply the ArgoCD Application

**Apply the application.yaml file to the Kubernetes cluster.**

`kubectl apply -f argocd/application.yaml`

After creating an Application custom resource in the argocd namespace. ArgoCD's controller will detect this new Application and begin its GitOps magic.

2. Monitor and Verify
ArgoCD will now automatically pull the manifests from your Git repository (nc-fttx-portal-gitops), create the nc-fttx-portal namespace, and deploy the application's resources.

```Bash

# Watch the application sync
kubectl get applications -n argocd -w

# Verify pods are running in the new namespace
kubectl get pods -n nc-fttx-portal
3. Access the Application
The ClusterIP service type can be used with the port-forward to access the portal.
```

`kubectl port-forward -n nc-fttx-portal svc/nc-fttx-portal-service 8081:80` # choose an applicable port
Access at: http://localhost:8081`


## Phase 5: Observability with Prometheus & Grafana (Optional)

This phase adds monitoring and visualization capabilities. It allows for the tracking of the health and performance of the nc-fttx-portal application using Prometheus for metrics collection and Grafana for dashboards.

### Step 1: Add Monitoring Namespace
Namespace Creation Manifest:

**monitoring/namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
    environment: local
    managed-by: argocd
```