# DevOps Engineer - Technical Reference

## CI/CD Requirements Gathering

```yaml
# CI/CD Requirements Checklist
Deployment Frequency: 
  - How often? (hourly/daily/weekly)
  - Peak deployment time? (business hours/off-hours)

Tech Stack:
  - Language/framework? (Node.js, Python, Java, Go)
  - Database? (PostgreSQL, MongoDB, Redis)
  - Frontend? (React, Vue, static site)

Infrastructure:
  - Cloud provider? (AWS/Azure/GCP/multi-cloud)
  - On-premise constraints? (network, compliance)
  - Auto-scaling needed? (yes/no, triggers)

Testing:
  - Unit test coverage target? (80%+)
  - Integration tests? (API, E2E)
  - Performance tests? (load, stress)
  - Security scans? (SAST, DAST, container scanning)

Compliance:
  - Audit logging required? (SOC2, HIPAA)
  - Approval gates? (manual/automated)
  - Secrets management? (Vault, AWS Secrets Manager)
```

## GitOps with ArgoCD Pattern

**When to use:** Declarative infrastructure management, audit trail for changes

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: https://github.com/company/infrastructure
    targetRevision: main
    path: k8s/production
    
    # Helm values override
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Health checks
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

**Benefits:**
- Git as single source of truth (audit trail)
- Automatic sync keeps cluster in desired state
- Easy rollback (git revert + auto-sync)
- Multi-cluster management from single repo

## Rollback Strategies

### Kubernetes Built-in Rollback
```bash
kubectl rollout undo deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=3
```

### Helm Rollback
```bash
helm rollback webapp 5  # Rollback to revision 5
helm rollback webapp 0  # Rollback to previous release
```

### Blue-Green Instant Rollback
```bash
kubectl patch service webapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Automated Rollback with Monitoring
```yaml
# Monitor error rate and auto-rollback
apiVersion: batch/v1
kind: CronJob
metadata:
  name: auto-rollback-monitor
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: monitor
              image: curlimages/curl:latest
              command:
                - /bin/sh
                - -c
                - |
                  ERROR_RATE=$(curl -s "http://prometheus/api/v1/query?query=rate(http_errors[5m])" | jq -r '.data.result[0].value[1]')
                  
                  if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
                    echo "ERROR RATE CRITICAL: $ERROR_RATE"
                    kubectl rollout undo deployment/webapp
                    
                    curl -X POST $SLACK_WEBHOOK \
                      -H 'Content-Type: application/json' \
                      -d "{\"text\":\"🚨 Auto-rollback triggered! Error rate: $ERROR_RATE\"}"
                  fi
          restartPolicy: OnFailure
```

## Infrastructure as Code Best Practices

### Terraform State Management
```hcl
terraform {
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "production/eks-cluster.tfstate"
    region = "us-east-1"
    encrypt = true
    dynamodb_table = "terraform-lock"
  }
}
```

### Tagging Strategy
```hcl
provider "aws" {
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "webapp"
      ManagedBy   = "Terraform"
      CostCenter  = var.cost_center
    }
  }
}
```

### Module Composition
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  # configuration
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  vpc_id  = module.vpc.vpc_id
  # configuration
}
```

## Monitoring Setup

### Prometheus + Grafana Configuration
```yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

alertmanager:
  config:
    global:
      slack_api_url: "${SLACK_WEBHOOK_URL}"
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      receiver: 'slack'
    
    receivers:
      - name: 'slack'
        slack_configs:
          - channel: '#alerts'
            title: 'Alert: {{ .GroupLabels.alertname }}'
```

### Installation Commands
```bash
# Install Prometheus stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring-values.yaml

# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

## Container Security Best Practices

### Multi-stage Dockerfile
```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Runtime stage
FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
WORKDIR /app
COPY --from=builder --chown=nodejs:nodejs /app .
USER nodejs
EXPOSE 3000
CMD ["node", "server.js"]
```

### Security Scanning
```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```
