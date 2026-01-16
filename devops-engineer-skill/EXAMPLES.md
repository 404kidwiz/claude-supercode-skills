# DevOps Engineer - Code Examples & Patterns

## Complete GitHub Actions CI/CD Pipeline

```yaml
# GitHub Actions Multi-Stage Pipeline
name: Production CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Stage 1: Build and Test
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Run unit tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
      
      - name: Build application
        run: npm run build
      
      - name: Build Docker image
        run: |
          docker build -t ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker build -t ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:latest .
      
      - name: Push Docker image
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker push ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  # Stage 2: Security Scanning
  security:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Run SAST with SonarQube
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  # Stage 3: Deploy to Staging
  deploy-staging:
    needs: [build, security]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Deploy to EKS staging
        run: |
          aws eks update-kubeconfig --name staging-cluster --region us-east-1
          kubectl set image deployment/app app=${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n staging
          kubectl rollout status deployment/app -n staging --timeout=5m
      
      - name: Run smoke tests
        run: |
          curl -f https://staging.example.com/health || exit 1
          npm run test:e2e -- --env=staging

  # Stage 4: Deploy to Production
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to EKS production
        run: |
          aws eks update-kubeconfig --name production-cluster --region us-east-1
          kubectl set image deployment/app app=${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n production
          kubectl rollout status deployment/app -n production --timeout=10m
      
      - name: Verify deployment
        run: |
          curl -f https://api.example.com/health || (kubectl rollback deployment/app -n production && exit 1)
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Production deployment successful: ${{ github.sha }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "✅ *Production Deployment*\nCommit: `${{ github.sha }}`\nAuthor: ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Terraform EKS Infrastructure

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "production/eks-cluster.tfstate"
    region = "us-east-1"
    encrypt = true
    dynamodb_table = "terraform-lock"
  }
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "webapp"
      ManagedBy   = "Terraform"
    }
  }
}

# VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "${var.environment}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_dns_hostnames = true
  
  tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  
  cluster_name    = var.cluster_name
  cluster_version = "1.27"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    general = {
      min_size     = 2
      max_size     = 10
      desired_size = 3
      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
    }
    
    spot = {
      min_size     = 1
      max_size     = 5
      desired_size = 2
      instance_types = ["t3.medium", "t3a.medium"]
      capacity_type  = "SPOT"
    }
  }
}

# Outputs
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}
```

## Anti-Patterns & Fixes

### Anti-Pattern 1: Manual Server Configuration (Snowflake Servers)

**BAD:**
```bash
# SSH into production server
ssh prod-server-01
sudo apt-get install nginx
sudo vi /etc/nginx/nginx.conf  # Hand-edit config
sudo systemctl restart nginx
# Result: Undocumented changes, impossible to replicate
```

**GOOD:**
```yaml
# ansible/playbooks/nginx.yml
- name: Configure Nginx web server
  hosts: webservers
  become: yes
  
  vars:
    nginx_version: "1.24"
  
  tasks:
    - name: Install Nginx
      apt:
        name: nginx={{ nginx_version }}*
        state: present
      
    - name: Configure Nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Reload Nginx
  
  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

### Anti-Pattern 2: No Rollback Strategy

**BAD:**
```bash
kubectl set image deployment/webapp webapp=webapp:v2.0.0
# Something breaks...
# Panic! How do we go back?
# Downtime: 45 minutes
```

**GOOD:**
```bash
# Method 1: Kubernetes built-in rollback
kubectl rollout undo deployment/webapp

# Method 2: Helm rollback
helm rollback webapp 5

# Method 3: Blue-Green (instant rollback)
kubectl patch service webapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

## Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: ghcr.io/company/webapp:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: production
spec:
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```
