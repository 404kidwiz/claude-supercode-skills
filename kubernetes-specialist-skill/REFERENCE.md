# Kubernetes Specialist - Technical Reference

## Workflow: Deploy Production Kubernetes Cluster (EKS Example)

### Step 1: Cluster Design Decisions

```yaml
# cluster-requirements.yaml
cluster:
  name: production-cluster
  region: us-east-1
  version: "1.28"  # Latest stable
  
  node_groups:
    - name: system
      instance_types: [t3.large]  # 2 vCPU, 8 GB RAM
      desired: 3  # High availability
      min: 3
      max: 5
      taints:
        - key: CriticalAddonsOnly
          value: "true"
          effect: NoSchedule
      labels:
        role: system
    
    - name: applications
      instance_types: [m5.xlarge, m5.2xlarge]  # 4-8 vCPU
      desired: 5
      min: 3
      max: 20
      autoscaling: true
      labels:
        role: applications
    
    - name: spot
      instance_types: [m5.large, m5.xlarge, m5.2xlarge]
      desired: 0
      min: 0
      max: 50
      capacity_type: SPOT  # 70% cost savings
      labels:
        role: batch-processing
  
  networking:
    vpc_cidr: 10.0.0.0/16
    pod_cidr: 100.64.0.0/16  # Secondary CIDR for pods
    service_cidr: 172.20.0.0/16
  
  addons:
    - vpc-cni  # AWS networking
    - coredns  # DNS
    - kube-proxy  # Service routing
    - aws-ebs-csi-driver  # Persistent storage
    - cluster-autoscaler  # Node autoscaling
    - metrics-server  # HPA metrics
```

### Step 2: Infrastructure as Code (Terraform)

```hcl
# eks-cluster.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.20.0"

  cluster_name    = "production-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  cluster_encryption_config = {
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }

  cluster_enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  eks_managed_node_groups = {
    system = {
      name = "system-nodes"
      
      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
      
      min_size     = 3
      max_size     = 5
      desired_size = 3
      
      taints = [{
        key    = "CriticalAddonsOnly"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
      
      labels = {
        role = "system"
      }
      
      ami_type = "AL2_x86_64"
      
      metadata_options = {
        http_endpoint               = "enabled"
        http_tokens                 = "required"
        http_put_response_hop_limit = 1
      }
    }
    
    applications = {
      name = "app-nodes"
      
      instance_types = ["m5.xlarge"]
      capacity_type  = "ON_DEMAND"
      
      min_size     = 3
      max_size     = 20
      desired_size = 5
      
      labels = {
        role = "applications"
      }
      
      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 100
            volume_type           = "gp3"
            iops                  = 3000
            throughput            = 125
            encrypted             = true
            kms_key_id            = aws_kms_key.ebs.arn
            delete_on_termination = true
          }
        }
      }
    }
    
    spot = {
      name = "spot-nodes"
      
      instance_types = ["m5.large", "m5.xlarge", "m5.2xlarge"]
      capacity_type  = "SPOT"
      
      min_size     = 0
      max_size     = 50
      desired_size = 0
      
      labels = {
        role = "batch-processing"
        "karpenter.sh/capacity-type" = "spot"
      }
      
      use_mixed_instances_policy = true
    }
  }

  enable_irsa = true

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.6.2"

  set {
    name  = "clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "serviceAccount.create"
    value = "true"
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.aws_load_balancer_controller.arn
  }
}

resource "helm_release" "cluster_autoscaler" {
  name       = "cluster-autoscaler"
  repository = "https://kubernetes.github.io/autoscaler"
  chart      = "cluster-autoscaler"
  namespace  = "kube-system"
  version    = "9.29.3"

  set {
    name  = "autoDiscovery.clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "awsRegion"
    value = var.aws_region
  }
}
```

### Step 3: Deploy Application with Helm

```bash
# Create namespace
kubectl create namespace production

# Install Prometheus + Grafana monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring-values.yaml

# Install example application
helm install myapp ./charts/myapp \
  --namespace production \
  --values production-values.yaml \
  --wait \
  --timeout 10m

# Verify deployment
kubectl get pods -n production
kubectl get svc -n production
kubectl get ingress -n production
```

## Workflow: Implement GitOps with ArgoCD

### 1. Install ArgoCD
```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2. Configure Git Repository
```yaml
# argocd-repo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/myorg/k8s-manifests
  password: ghp_xxxxxxxxxxxxx
  username: git
```

## Production Readiness Checklist

```bash
# Security
[ ] Pod Security Standards enforced
kubectl label namespace production pod-security.kubernetes.io/enforce=restricted

[ ] Network policies configured (default deny, explicit allow)
[ ] RBAC configured (least privilege for service accounts)
[ ] Secrets encrypted at rest (KMS integration verified)
[ ] Image scanning enabled (Trivy, Anchore)

# High Availability
[ ] Multi-AZ node distribution
kubectl get nodes -o wide | grep -c us-east-1

[ ] Pod Disruption Budgets configured
kubectl get pdb --all-namespaces

[ ] Anti-affinity rules for critical pods
[ ] Readiness and liveness probes configured
kubectl describe pod | grep -A5 Probes

# Observability
[ ] Metrics server installed
kubectl top nodes

[ ] Prometheus scraping application metrics
[ ] Grafana dashboards configured
[ ] Logging to CloudWatch / Elasticsearch
[ ] Distributed tracing (Jaeger / Tempo)

# Resource Management
[ ] Resource requests and limits set
kubectl describe pod | grep -A5 Requests

[ ] HorizontalPodAutoscaler configured
kubectl get hpa --all-namespaces

[ ] Cluster Autoscaler working
kubectl logs -n kube-system deployment/cluster-autoscaler

# Disaster Recovery
[ ] etcd backups automated
[ ] Velero installed for application backups
[ ] Backup restoration tested
[ ] Multi-region DR strategy documented
```
