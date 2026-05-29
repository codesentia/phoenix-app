# Phoenix Team Applications

This repository contains Kubernetes applications for the Phoenix team deployed on the EKS shared platform via ArgoCD.

## Repository Structure

```
phoenix-apps/
├── apps/
│   ├── webapp/          # Simple nginx webapp with custom HTML
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── api/             # HTTPBin API for testing
│       ├── deployment.yaml
│       └── service.yaml
└── applicationset.yaml  # ArgoCD ApplicationSet for auto-discovery
```

## Setup Instructions

### 1. Create GitHub Repository

```bash
# Create a new repository on GitHub named 'phoenix-apps'
# Then push this code:

cd /tmp/phoenix-apps
git init
git add .
git commit -m "Initial commit: Phoenix team applications"
git branch -M main
git remote add origin https://github.com/YOUR_ORG/phoenix-apps.git
git push -u origin main
```

### 2. Update ApplicationSet

Edit `applicationset.yaml` and replace `YOUR_ORG` with your GitHub organization/username.

### 3. Onboard Team to EKS Platform

```bash
make onboard-team \
  TEAM_NAME=phoenix \
  REPO_URL=https://github.com/YOUR_ORG/phoenix-apps \
  CPU_QUOTA=4 \
  MEMORY_QUOTA=8Gi \
  CONTACT_EMAIL=phoenix@example.com
```

### 4. Deploy ApplicationSet

```bash
kubectl apply -f applicationset.yaml
```

### 5. Verify Deployment

```bash
# Check ArgoCD Applications
kubectl get applications -n argocd | grep phoenix

# Check pods in team namespace
kubectl get pods -n team-phoenix

# Port-forward to webapp
kubectl port-forward -n team-phoenix svc/webapp 8080:80

# Access at http://localhost:8080
```

## Applications

### Webapp
- **Image**: nginx:1.27-alpine
- **Replicas**: 2
- **Resources**: 100m CPU / 128Mi memory (request), 200m CPU / 256Mi memory (limit)
- **Features**: Custom HTML page, health checks, horizontal scaling ready

### API
- **Image**: kennethreitz/httpbin (public testing API)
- **Replicas**: 1
- **Resources**: 50m CPU / 64Mi memory (request), 100m CPU / 128Mi memory (limit)
- **Endpoints**: /get, /post, /status/*, /headers, etc.

## Testing the Setup

### Test Webapp
```bash
# Port-forward
kubectl port-forward -n team-phoenix svc/webapp 8080:80

# Access in browser: http://localhost:8080
# You should see a purple gradient page with platform info
```

### Test API
```bash
# Port-forward
kubectl port-forward -n team-phoenix svc/api 8081:80

# Test endpoints
curl http://localhost:8081/get
curl http://localhost:8081/status/200
curl -X POST http://localhost:8081/post -d '{"test":"data"}'
```

### Test ArgoCD Sync
```bash
# Make a change to webapp replicas
# Edit apps/webapp/deployment.yaml, change replicas: 2 to replicas: 3
# Commit and push

git add apps/webapp/deployment.yaml
git commit -m "Scale webapp to 3 replicas"
git push

# ArgoCD will auto-sync within ~3 minutes
# Watch the sync:
kubectl get applications -n argocd -w
kubectl get pods -n team-phoenix -w
```

## Resource Quotas

The team-phoenix namespace has the following quotas:
- CPU requests: 4 cores
- Memory requests: 8Gi
- Pods: 50 max

Current usage:
- webapp: 2 pods × 100m CPU = 200m CPU, 256Mi memory
- api: 1 pod × 50m CPU = 50m CPU, 64Mi memory
- **Total**: 250m CPU, 320Mi memory (well within quota)

## Network Policies

The namespace has default-deny ingress and allows:
- Egress to kube-dns (UDP 53)
- Egress within same namespace
- Egress to Kubernetes API (TCP 443)

To allow external API calls, create additional NetworkPolicy resources.

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod -n team-phoenix <pod-name>
kubectl logs -n team-phoenix <pod-name>
```

### ArgoCD sync issues
```bash
# Check Application status
kubectl get application phoenix-webapp -n argocd -o yaml

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### ImagePullBackOff errors
- For public images (nginx, httpbin): Check cluster internet connectivity
- For ECR images: Verify ArgoCD IRSA role has ECR pull permissions

## Next Steps

1. **Add Ingress**: Expose services via AWS Load Balancer Controller
2. **Add Monitoring**: Configure Prometheus ServiceMonitor
3. **Add Secrets**: Use Kubernetes Secrets or AWS Secrets Manager
4. **CI/CD Integration**: Add GitHub Actions to build/push custom images
