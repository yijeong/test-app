# Test-App

A Kubernetes-based application deployment repository featuring two deployment strategies: standard Kubernetes Deployment and Argo Rollouts for advanced progressive delivery with canary deployments.

## 📋 Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Helm Charts](#helm-charts)
  - [FastTest (Standard Deployment)](#fasttest-standard-deployment)
  - [FastTest-Rollout (Canary Deployment)](#fasttest-rollout-canary-deployment)
- [ArgoCD Applications](#argocd-applications)
- [Installation](#installation)
- [Configuration](#configuration)
- [Deployment Procedures](#deployment-procedures)
- [Monitoring and Maintenance](#monitoring-and-maintenance)

## 🎯 Overview

This repository contains Helm charts and ArgoCD application definitions for deploying the `fasttest` application to Kubernetes clusters. It supports two deployment strategies:

1. **Standard Deployment** (`fasttest`): Traditional Kubernetes Deployment with HPA
2. **Canary Deployment** (`fasttest-rollout`): Advanced progressive delivery using Argo Rollouts

Both strategies are managed through ArgoCD for GitOps-based continuous delivery.

## 📁 Project Structure

```
test-app/
├── argoapp/                          # ArgoCD Application definitions
│   ├── fasttest.yaml                 # ArgoCD App for standard deployment
│   └── fasttest-rollout.yaml         # ArgoCD App for canary deployment
├── helmchart/                        # Helm charts directory
│   ├── fasttest/                     # Standard deployment chart
│   │   ├── Chart.yaml                # Chart metadata
│   │   ├── values.yaml               # Default configuration values
│   │   └── templates/                # Kubernetes resource templates
│   │       ├── deployment.yaml       # Standard Deployment resource
│   │       ├── service.yaml          # Service definition
│   │       ├── serviceaccount.yaml   # ServiceAccount
│   │       ├── ingress.yaml          # ALB Ingress configuration
│   │       └── hpa.yaml              # Horizontal Pod Autoscaler
│   └── fasttest-rollout/             # Canary deployment chart
│       ├── Chart.yaml                # Chart metadata
│       ├── values.yaml               # Default configuration values
│       └── templates/                # Kubernetes resource templates
│           ├── rollout.yaml          # Argo Rollout resource
│           ├── service.yaml          # Service definition
│           ├── serviceaccount.yaml   # ServiceAccount
│           ├── ingress.yaml          # ALB Ingress configuration
│           ├── hpa.yaml              # Horizontal Pod Autoscaler
│           └── analysisTemplate.yaml # Analysis template for rollout validation
└── eksClusterConfig.yaml             # EKS cluster configuration (eksctl)
```

## 🔧 Prerequisites

Before deploying this application, ensure you have the following:

### Required Tools

- **kubectl** (v1.29+): Kubernetes command-line tool
- **helm** (v3.x): Package manager for Kubernetes
- **eksctl** (optional): For EKS cluster management
- **aws-cli** (v2.x): For AWS resource access
- **argocd CLI** (optional): For ArgoCD management

### Infrastructure Requirements

- **Kubernetes Cluster**: EKS 1.29+ (or compatible Kubernetes cluster)
- **ArgoCD**: Installed and configured in the `argocd` namespace
- **Argo Rollouts**: Installed for canary deployment support
- **AWS Load Balancer Controller**: For ALB Ingress functionality
- **Container Registry Access**: Access to AWS ECR (`339712822719.dkr.ecr.ap-northeast-2.amazonaws.com`)

### AWS Resources

- EKS Cluster with OIDC provider enabled
- VPC with subnets (specified in configuration)
- IAM roles and policies for service accounts
- ECR repository for container images

## 📦 Helm Charts

### FastTest (Standard Deployment)

The `fasttest` Helm chart deploys the application using a standard Kubernetes Deployment with the following features:

#### Key Features

- **Deployment**: Standard Kubernetes Deployment
- **Horizontal Pod Autoscaling**: 6-10 replicas based on CPU utilization (80%)
- **Service**: ClusterIP service on port 8080
- **Ingress**: AWS ALB Ingress with health checks
- **Container Image**: `339712822719.dkr.ecr.ap-northeast-2.amazonaws.com/test:v2`

#### Configuration Options

Key values in `helmchart/fasttest/values.yaml`:

```yaml
# Application settings
applicationName: "fasttest"
image:
  repository: 339712822719.dkr.ecr.ap-northeast-2.amazonaws.com/test
  tag: "v2"

# Auto-scaling
autoscaling:
  enabled: true
  minReplicas: 6
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Service configuration
service:
  type: ClusterIP
  port: 8080

# Ingress/ALB settings
ingressLbName: test-lb
ingressGroupName: test
albHealthCheck: /
```

### FastTest-Rollout (Canary Deployment)

The `fasttest-rollout` Helm chart deploys the application using Argo Rollouts for progressive canary deployments.

#### Key Features

- **Rollout**: Argo Rollouts resource for canary deployments
- **Progressive Delivery**: Gradual traffic shifting (20% → 50% → 100%)
- **Analysis**: Automated memory monitoring during rollout
- **Horizontal Pod Autoscaling**: 4-10 replicas based on CPU utilization (80%)
- **Traffic Routing**: ALB-based traffic management
- **Ping-Pong Services**: Separate canary and stable services

#### Canary Strategy

The rollout follows this progression:

```yaml
steps:
  - setWeight: 20      # Route 20% traffic to canary
  - pause: 10s         # Wait 10 seconds
  - setWeight: 50      # Route 50% traffic to canary
  - pause: {}          # Manual approval required
```

#### Configuration Options

Key values in `helmchart/fasttest-rollout/values.yaml`:

```yaml
# Application settings
applicationName: "fasttest-rollout"
image:
  repository: 339712822719.dkr.ecr.ap-northeast-2.amazonaws.com/test
  tag: "v2"

# Auto-scaling
autoscaling:
  enabled: true
  minReplicas: 4
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Resource limits
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Canary configuration
canary:
  maxSurge: "25%"
  maxUnavailable: 0
  steps:
    - setWeight: 20
    - pause: { duration: 10s }
    - setWeight: 50
    - pause: {}  # Manual promotion required
```

## 🚀 ArgoCD Applications

The `argoapp/` directory contains ArgoCD Application manifests that define how the Helm charts are deployed through GitOps.

### Purpose

ArgoCD Applications serve as the bridge between Git repository and Kubernetes cluster, enabling:

- **Continuous Deployment**: Automatic sync from Git to cluster
- **GitOps Workflow**: Infrastructure as Code approach
- **Declarative Configuration**: Desired state defined in Git
- **Sync Policies**: Automated or manual deployment strategies

### FastTest Application (`argoapp/fasttest.yaml`)

Deploys the standard deployment chart to the `fasttest` namespace.

**Key Configuration:**

```yaml
spec:
  project: default
  source:
    repoURL: https://github.com/yijeong/test-app
    targetRevision: main
    path: helmchart/fasttest
  destination:
    namespace: fasttest
  syncPolicy:
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### FastTest-Rollout Application (`argoapp/fasttest-rollout.yaml`)

Deploys the canary deployment chart to the `fasttest-rollout` namespace.

**Key Configuration:**

```yaml
spec:
  project: default
  source:
    repoURL: https://github.com/yijeong/test-app
    targetRevision: main
    path: helmchart/fasttest-rollout
  destination:
    namespace: fasttest-rollout
  syncPolicy:
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Sync Policy Options

- **Validate=true**: Helm validation before deployment
- **CreateNamespace=true**: Automatically create target namespace
- **PruneLast=true**: Delete removed resources after new ones are created
- **ApplyOutOfSyncOnly=true**: Only apply out-of-sync resources

## 🛠️ Installation

### Step 1: Create EKS Cluster (Optional)

If you need to create a new EKS cluster:

```bash
# Create cluster using eksctl
eksctl create cluster -f eksClusterConfig.yaml

# Verify cluster access
kubectl get nodes

# Update kubeconfig
aws eks update-kubeconfig --name test-cluster --region ap-northeast-2
```

### Step 2: Install Required Components

#### Install AWS Load Balancer Controller

```bash
# Add EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=test-cluster \
  --set serviceAccount.create=true \
  --set region=ap-northeast-2
```

#### Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

#### Install Argo Rollouts (for canary deployments)

```bash
# Create Argo Rollouts namespace
kubectl create namespace argo-rollouts

# Install Argo Rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Verify installation
kubectl get pods -n argo-rollouts
```

### Step 3: Configure ECR Access

Ensure your Kubernetes cluster has access to the ECR repository:

```bash
# Create ECR credentials secret (if needed)
kubectl create secret docker-registry ecr-credentials \
  --docker-server=339712822719.dkr.ecr.ap-northeast-2.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-northeast-2) \
  -n fasttest

kubectl create secret docker-registry ecr-credentials \
  --docker-server=339712822719.dkr.ecr.ap-northeast-2.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-northeast-2) \
  -n fasttest-rollout
```

### Step 4: Deploy Applications via ArgoCD

#### Deploy Standard Application

```bash
# Apply ArgoCD Application manifest
kubectl apply -f argoapp/fasttest.yaml

# Verify application is created
kubectl get application -n argocd

# Sync the application
kubectl -n argocd patch app fasttest --type merge -p '{"operation":{"sync":{}}}'

# Or use ArgoCD CLI
argocd app sync fasttest
```

#### Deploy Canary Application

```bash
# Apply ArgoCD Application manifest
kubectl apply -f argoapp/fasttest-rollout.yaml

# Verify application is created
kubectl get application -n argocd

# Sync the application
kubectl -n argocd patch app fasttest-rollout --type merge -p '{"operation":{"sync":{}}}'

# Or use ArgoCD CLI
argocd app sync fasttest-rollout
```

## ⚙️ Configuration

### Modifying Application Configuration

To modify application configuration, update the respective `values.yaml` file and commit to Git. ArgoCD will detect changes and sync automatically (if automated sync is enabled) or wait for manual sync.

#### Update Image Tag

```yaml
# Edit helmchart/fasttest/values.yaml or helmchart/fasttest-rollout/values.yaml
image:
  repository: 339712822719.dkr.ecr.ap-northeast-2.amazonaws.com/test
  tag: "v3"  # Update version
```

#### Adjust Autoscaling

```yaml
# Edit helmchart/fasttest/values.yaml
autoscaling:
  enabled: true
  minReplicas: 8  # Increase minimum replicas
  maxReplicas: 15  # Increase maximum replicas
  targetCPUUtilizationPercentage: 70  # Lower threshold
```

#### Modify Canary Strategy

```yaml
# Edit helmchart/fasttest-rollout/values.yaml
canary:
  maxSurge: "25%"
  maxUnavailable: 0
  steps:
    - setWeight: 10   # Start with 10% traffic
    - pause: { duration: 30s }
    - setWeight: 25
    - pause: { duration: 30s }
    - setWeight: 50
    - pause: {}  # Manual approval
```

### Environment-Specific Configuration

For different environments (dev, staging, prod), consider:

1. Using separate branches (e.g., `dev`, `staging`, `main`)
2. Creating environment-specific values files (e.g., `values-prod.yaml`)
3. Configuring ArgoCD to use different target revisions

## 📤 Deployment Procedures

### Standard Deployment Workflow

1. **Make Changes**: Update application code or configuration
2. **Build Image**: Build and push new Docker image to ECR
3. **Update Values**: Update image tag in `helmchart/fasttest/values.yaml`
4. **Commit & Push**: Commit changes to Git repository
5. **Sync ArgoCD**: ArgoCD detects changes and syncs (manual or automatic)
6. **Verify Deployment**: Check deployment status

```bash
# Check deployment status
kubectl get deployment -n fasttest
kubectl get pods -n fasttest

# Check HPA status
kubectl get hpa -n fasttest

# View rollout history
kubectl rollout history deployment/fasttest -n fasttest
```

### Canary Deployment Workflow

1. **Make Changes**: Update application code or configuration
2. **Build Image**: Build and push new Docker image to ECR
3. **Update Values**: Update image tag in `helmchart/fasttest-rollout/values.yaml`
4. **Commit & Push**: Commit changes to Git repository
5. **Sync ArgoCD**: ArgoCD detects changes and initiates rollout
6. **Monitor Canary**: Watch canary deployment progress
7. **Promote or Abort**: Manually promote to 100% or abort if issues detected

```bash
# Watch rollout progress
kubectl argo rollouts get rollout fasttest-rollout -n fasttest-rollout --watch

# Check rollout status
kubectl argo rollouts status fasttest-rollout -n fasttest-rollout

# Promote rollout (move to next step)
kubectl argo rollouts promote fasttest-rollout -n fasttest-rollout

# Abort rollout if issues detected
kubectl argo rollouts abort fasttest-rollout -n fasttest-rollout

# Retry failed rollout
kubectl argo rollouts retry fasttest-rollout -n fasttest-rollout
```

### Rollback Procedures

#### Standard Deployment Rollback

```bash
# Rollback to previous revision
kubectl rollout undo deployment/fasttest -n fasttest

# Rollback to specific revision
kubectl rollout undo deployment/fasttest -n fasttest --to-revision=2

# Check rollout status
kubectl rollout status deployment/fasttest -n fasttest
```

#### Canary Deployment Rollback

```bash
# Abort active rollout (automatically rolls back)
kubectl argo rollouts abort fasttest-rollout -n fasttest-rollout

# Or update values.yaml with previous image tag and sync
```

## 📊 Monitoring and Maintenance

### Check Application Health

#### Standard Deployment

```bash
# Check deployment status
kubectl get deployment fasttest -n fasttest

# Check pod status
kubectl get pods -n fasttest -l app.kubernetes.io/name=fasttest

# Check service
kubectl get service fasttest -n fasttest

# Check ingress
kubectl get ingress -n fasttest

# View logs
kubectl logs -n fasttest -l app.kubernetes.io/name=fasttest --tail=100 -f
```

#### Canary Deployment

```bash
# Check rollout status
kubectl argo rollouts get rollout fasttest-rollout -n fasttest-rollout

# Check pods
kubectl get pods -n fasttest-rollout

# Check analysis runs
kubectl get analysisrun -n fasttest-rollout

# View logs
kubectl logs -n fasttest-rollout -l app.kubernetes.io/name=fasttest-rollout --tail=100 -f
```

### ArgoCD Monitoring

```bash
# Check application status
kubectl get applications -n argocd

# View application details
argocd app get fasttest
argocd app get fasttest-rollout

# View sync history
argocd app history fasttest
argocd app history fasttest-rollout

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open browser: https://localhost:8080
```

### Troubleshooting

#### Pods Not Starting

```bash
# Describe pod to see events
kubectl describe pod <pod-name> -n <namespace>

# Check logs
kubectl logs <pod-name> -n <namespace>

# Check image pull secrets
kubectl get serviceaccount -n <namespace>
```

#### Ingress Issues

```bash
# Check ALB Ingress controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check ingress status
kubectl describe ingress -n <namespace>

# Verify AWS Load Balancer is created
aws elbv2 describe-load-balancers --region ap-northeast-2
```

#### Rollout Stuck

```bash
# Check rollout details
kubectl argo rollouts get rollout fasttest-rollout -n fasttest-rollout

# Check analysis run
kubectl describe analysisrun -n fasttest-rollout

# Manually promote if needed
kubectl argo rollouts promote fasttest-rollout -n fasttest-rollout

# Or abort and retry
kubectl argo rollouts abort fasttest-rollout -n fasttest-rollout
```

## 📝 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)

## 🤝 Contributing

1. Create a feature branch from `main`
2. Make your changes
3. Test in a non-production environment
4. Submit a pull request with detailed description

## 📞 Support

For issues or questions, please contact the platform team or create an issue in the repository.

---

**Note**: This repository uses GitOps practices. All changes should be made via Git commits and synchronized through ArgoCD. Avoid making direct changes to the Kubernetes cluster.
