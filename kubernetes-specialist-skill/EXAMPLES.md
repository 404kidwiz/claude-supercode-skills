# Kubernetes Specialist - Code Examples & Patterns

## Blue-Green Deployment Pattern

**When to use:** Zero-downtime deployments with instant rollback capability

### Blue Deployment (current production)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myregistry.com/myapp:v1.0.0
        ports:
        - containerPort: 8080
```

### Service (switches between blue and green)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp
    version: blue  # Change to 'green' to cutover
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

### Deployment Process
```bash
# Step 1: Deploy green (new version) alongside blue
kubectl apply -f green-deployment.yaml

# Step 2: Wait for green to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/myapp-green -n production

# Step 3: Test green deployment (internal testing)
kubectl port-forward deployment/myapp-green -n production 9000:8080
curl http://localhost:9000/health

# Step 4: Cutover traffic to green (instant switch)
kubectl patch service myapp-service -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Step 5: Monitor for issues (5-10 minutes)
kubectl logs -f deployment/myapp-green -n production

# If successful: Delete blue
kubectl delete deployment myapp-blue -n production

# If issues: Instant rollback
kubectl patch service myapp-service -n production \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

## Anti-Pattern 1: No Resource Requests/Limits

### What it looks like (BAD):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    # No resources defined - pod can consume entire node!
```

### Why it fails:
- Pod scheduled to node without capacity check (causes OOMKilled on other pods)
- No QoS class (BestEffort - killed first during resource pressure)
- HPA cannot scale (requires resource requests to calculate utilization)

### Correct approach:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:  # Minimum guaranteed resources
        cpu: "500m"     # 0.5 CPU cores
        memory: "512Mi" # 512 MB
      limits:    # Maximum allowed resources
        cpu: "1000m"    # 1 CPU core
        memory: "1Gi"   # 1 GB
    # QoS class: Guaranteed (requests == limits)
```

## Anti-Pattern 2: Missing Health Probes

### What it looks like (BAD):
```yaml
containers:
- name: app
  image: myapp:latest
  # No liveness or readiness probes!
```

### Why it fails:
- Kubernetes sends traffic to pod immediately (even if app not ready)
- Crashed pods not restarted automatically
- Rolling updates don't wait for new pods to be healthy

### Correct approach:
```yaml
containers:
- name: app
  image: myapp:latest
  ports:
  - containerPort: 8080
  
  livenessProbe:  # Restart pod if this fails
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 30  # Wait for app to start
    periodSeconds: 10        # Check every 10 seconds
    timeoutSeconds: 5
    failureThreshold: 3      # Restart after 3 failures
  
  readinessProbe:  # Remove from service if this fails
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 2
```

## Network Policy Example

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow traffic from specific namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

## HorizontalPodAutoscaler Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

## PodDisruptionBudget Example

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2  # Or use maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

## StatefulSet for Databases

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi
```
