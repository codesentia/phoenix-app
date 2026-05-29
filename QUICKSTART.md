# Quick Start Guide

## Prerequisites
- EKS cluster `thor` deployed and running
- ArgoCD installed (`make install-argocd`)
- GitHub account
- kubectl configured for thor cluster

## Step 1: Create GitHub Repository

1. Go to https://github.com/new
2. Repository name: `phoenix-apps`
3. Visibility: Public (or Private if you configure ArgoCD with GitHub token)
4. Don't initialize with README (we already have one)
5. Click "Create repository"

## Step 2: Push Code to GitHub

```bash
cd /tmp/phoenix-apps

# Initialize git
git init
git add .
git commit -m "Initial commit: Phoenix team applications"

# Add your remote (replace YOUR_USERNAME)
git remote add origin https://github.com/YOUR_USERNAME/phoenix-apps.git
git branch -M main
git push -u origin main
```

## Step 3: Update ApplicationSet

Edit `applicationset.yaml` and replace `YOUR_ORG` with your GitHub username:

```yaml
repoURL: https://github.com/YOUR_USERNAME/phoenix-apps  # Line 9
repoURL: https://github.com/YOUR_USERNAME/phoenix-apps  # Line 19
```

Commit and push:
```bash
git add applicationset.yaml
git commit -m "Update repo URL"
git push
```

## Step 4: Onboard Team to EKS

```bash
cd ~/Documents/my-repos/eks-infra

make onboard-team \
  TEAM_NAME=phoenix \
  REPO_URL=https://github.com/YOUR_USERNAME/phoenix-apps \
  CPU_QUOTA=4 \
  MEMORY_QUOTA=8Gi \
  CONTACT_EMAIL=your.email@example.com
```

Expected output:
```
Onboarding team: phoenix
  Repository: https://github.com/YOUR_USERNAME/phoenix-apps
  CPU quota: 4
  Memory quota: 8Gi
  Contact: your.email@example.com

Rendering namespace templates...
Validating manifests with kubectl dry-run...
✓ Manifest validation passed

Applying manifests to cluster...
namespace/team-phoenix created
resourcequota/team-phoenix-quota created
limitrange/team-phoenix-limits created
networkpolicy/team-phoenix-isolation created
serviceaccount/team-phoenix-admin created
rolebinding.rbac.authorization.k8s.io/team-phoenix-admin-binding created
appproject.argoproj.io/team-phoenix created

✓ Team onboarding complete!

Namespace: team-phoenix
AppProject: team-phoenix (in argocd namespace)
```

## Step 5: Validate Onboarding

```bash
python scripts/validate_team_setup.py --team phoenix
```

Expected output:
```
Validating onboarding for team: phoenix
Namespace: team-phoenix

✓ Namespace team-phoenix exists with correct labels
✓ ResourceQuota team-phoenix-quota exists
✓ LimitRange team-phoenix-limits exists
✓ NetworkPolicy team-phoenix-isolation exists
✓ ServiceAccount and RoleBinding exist
✓ AppProject team-phoenix exists in argocd namespace
✓ NetworkPolicy configured with Ingress and Egress rules

✓ All validation checks passed!
```

## Step 6: Deploy ApplicationSet

```bash
kubectl apply -f /tmp/phoenix-apps/applicationset.yaml
```

Expected output:
```
applicationset.argoproj.io/phoenix-apps created
```

## Step 7: Watch ArgoCD Sync

```bash
# Watch ApplicationSet create Applications
kubectl get applications -n argocd | grep phoenix

# Watch pods come up
kubectl get pods -n team-phoenix -w
```

Within 1-3 minutes, you should see:
```
NAME                      READY   STATUS    RESTARTS   AGE
api-xxxxxxxxxx-xxxxx      1/1     Running   0          45s
webapp-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
webapp-xxxxxxxxxx-yyyyy   1/1     Running   0          45s
```

## Step 8: Test the Applications

### Test Webapp
```bash
# Port-forward to webapp
kubectl port-forward -n team-phoenix svc/webapp 8080:80

# Open in browser: http://localhost:8080
# You should see a purple gradient page with platform info
```

### Test API
```bash
# Port-forward to api
kubectl port-forward -n team-phoenix svc/api 8081:80

# Test in another terminal:
curl http://localhost:8081/get
curl http://localhost:8081/headers
```

### Check ArgoCD UI
```bash
# Port-forward to ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8082:443

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d

# Open: https://localhost:8082
# Login: admin / <password from above>
# Navigate to Applications → phoenix-webapp and phoenix-api
```

## Step 9: Test GitOps Workflow

Make a change to trigger ArgoCD sync:

```bash
cd /tmp/phoenix-apps

# Edit apps/webapp/deployment.yaml
# Change line: replicas: 2
# To: replicas: 3

sed -i 's/replicas: 2/replicas: 3/' apps/webapp/deployment.yaml

# Commit and push
git add apps/webapp/deployment.yaml
git commit -m "Scale webapp to 3 replicas"
git push

# Watch ArgoCD sync (takes ~3 minutes)
kubectl get pods -n team-phoenix -w
```

You should see a third webapp pod start up automatically!

## Troubleshooting

### Applications not appearing in ArgoCD
```bash
# Check ApplicationSet status
kubectl get applicationset phoenix-apps -n argocd -o yaml

# Check ArgoCD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller
```

### Pods not starting
```bash
# Describe the pod
kubectl describe pod -n team-phoenix <pod-name>

# Check logs
kubectl logs -n team-phoenix <pod-name>

# Common issues:
# - ImagePullBackOff: Check internet connectivity
# - CrashLoopBackOff: Check application logs
# - Pending: Check resource quotas
```

### ArgoCD sync failures
```bash
# Check Application details
kubectl get application phoenix-webapp -n argocd -o yaml

# Check sync status
kubectl describe application phoenix-webapp -n argocd
```

## Success Indicators

You've successfully onboarded a team when:
- ✅ Namespace `team-phoenix` exists
- ✅ ArgoCD shows 2 applications (phoenix-webapp, phoenix-api)
- ✅ Both applications are "Synced" and "Healthy" in ArgoCD UI
- ✅ 3 pods running in team-phoenix namespace (2 webapp, 1 api)
- ✅ Webapp accessible via port-forward shows custom HTML page
- ✅ Changes to git repo automatically sync to cluster

## Next Steps

1. **Configure Ingress**: Expose webapp via AWS Load Balancer
2. **Add Custom Images**: Build and push to ECR, update deployments
3. **Set up Monitoring**: Add Prometheus metrics
4. **Configure Secrets**: Use Kubernetes Secrets or AWS Secrets Manager
5. **Onboard More Teams**: Repeat this process for other teams
