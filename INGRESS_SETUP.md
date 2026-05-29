# Exposing Phoenix Apps to the Internet

Quick guide to make your applications accessible from the internet using AWS Load Balancer Controller.

---

## Prerequisites

- EKS cluster `thor` running
- Team `phoenix` onboarded
- Applications deployed in `team-phoenix` namespace

---

## Step 1: Install AWS Load Balancer Controller

### 1.1 Create IAM Policy

```bash
curl -o /tmp/iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file:///tmp/iam-policy.json
```

### 1.2 Create IAM Role with IRSA

Get your AWS account ID and OIDC issuer:

```bash
# Get AWS account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Get OIDC issuer (without https://)
OIDC_ISSUER=$(aws ssm get-parameter --name /eks/thor/oidc-issuer-url --query "Parameter.Value" --output text | sed 's|https://||')

echo "Account: $AWS_ACCOUNT_ID"
echo "OIDC Issuer: $OIDC_ISSUER"
```

Create IAM role trust policy:

```bash
cat > /tmp/lb-controller-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ISSUER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_ISSUER}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller",
          "${OIDC_ISSUER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name eks-thor-lb-controller-role \
  --assume-role-policy-document file:///tmp/lb-controller-trust-policy.json

# Attach the policy
aws iam attach-role-policy \
  --role-name eks-thor-lb-controller-role \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy

# Get role ARN
LB_ROLE_ARN=$(aws iam get-role --role-name eks-thor-lb-controller-role --query Role.Arn --output text)
echo "Role ARN: $LB_ROLE_ARN"
```

### 1.3 Install Controller via Helm

```bash
# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=thor \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${LB_ROLE_ARN} \
  --set region=us-east-1 \
  --set vpcId=$(aws cloudformation describe-stacks --stack-name vpc-dev --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)
```

### 1.4 Verify Installation

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller

# Expected output:
# NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
# aws-load-balancer-controller   2/2     2            2           1m
```

---

## Step 2: Add Ingress to Your Applications

### 2.1 Create Ingress for Webapp

Create `apps/webapp/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp
  namespace: team-phoenix
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: phoenix-apps
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp
                port:
                  number: 80
```

### 2.2 Create Ingress for API

Create `apps/api/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: team-phoenix
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: phoenix-apps
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /get
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

### 2.3 Commit and Push

```bash
cd /tmp/phoenix-app

git add apps/webapp/ingress.yaml apps/api/ingress.yaml
git commit -m "Add Ingress resources for internet access"
git push origin main
```

### 2.4 Wait for ArgoCD to Sync

```bash
# Watch for Ingress resources (~3 minutes)
kubectl get ingress -n team-phoenix -w
```

---

## Step 3: Access Your Applications

### 3.1 Get ALB URL

```bash
# Wait for ALB to be created (~2-3 minutes)
kubectl get ingress webapp -n team-phoenix

# Example output:
# NAME     CLASS   HOSTS   ADDRESS                                          PORTS   AGE
# webapp   alb     *       k8s-phoenix-apps-abc123-456789.us-east-1.elb... 80      2m
```

### 3.2 Get the Full URL

```bash
ALB_URL=$(kubectl get ingress webapp -n team-phoenix -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Webapp URL: http://${ALB_URL}"
echo "API URL: http://${ALB_URL}/api/get"
```

### 3.3 Test Access

```bash
# Test webapp
curl http://${ALB_URL}

# Test API
curl http://${ALB_URL}/api/get
```

Or open in browser:
- Webapp: `http://<ALB_URL>`
- API: `http://<ALB_URL>/api/get`

---

## Step 4: Add Custom Domain (Optional)

### 4.1 Get ALB DNS Name

```bash
ALB_DNS=$(kubectl get ingress webapp -n team-phoenix -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB DNS: $ALB_DNS"
```

### 4.2 Create DNS Record

In your DNS provider (Route53, Cloudflare, etc.), create a CNAME record:

```
phoenix.yourdomain.com  →  CNAME  →  k8s-phoenix-apps-abc123.us-east-1.elb.amazonaws.com
```

### 4.3 Update Ingress with Host

Update `apps/webapp/ingress.yaml`:

```yaml
spec:
  ingressClassName: alb
  rules:
    - host: phoenix.yourdomain.com  # Add this
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp
                port:
                  number: 80
```

Commit and push to update.

---

## Step 5: Add HTTPS/TLS (Optional)

### 5.1 Request Certificate in ACM

```bash
aws acm request-certificate \
  --domain-name phoenix.yourdomain.com \
  --validation-method DNS \
  --region us-east-1
```

Follow ACM instructions to validate the certificate (add DNS record).

### 5.2 Update Ingress with Certificate

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT_ID
```

---

## Troubleshooting

### ALB Not Created

```bash
# Check controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check Ingress events
kubectl describe ingress webapp -n team-phoenix
```

### Unhealthy Targets

```bash
# Check target group health in AWS Console
# Or via CLI:
aws elbv2 describe-target-health --target-group-arn <target-group-arn>

# Common issues:
# - Security groups: ALB needs access to NodePort range (not needed with IP target-type)
# - Health check path wrong
# - Pods not ready
```

### 403/502 Errors

```bash
# Check pod logs
kubectl logs -n team-phoenix -l app=webapp

# Check service endpoints
kubectl get endpoints -n team-phoenix
```

---

## Cost Optimization

**Current Setup:** 1 ALB shared by both apps (~$16-20/month)

The `alb.ingress.kubernetes.io/group.name: phoenix-apps` annotation makes both Ingress resources share the same ALB.

**To add more apps:** Use the same group name to share the ALB:

```yaml
annotations:
  alb.ingress.kubernetes.io/group.name: phoenix-apps  # Reuse same ALB
```

---

## Summary

You now have:
- ✅ AWS Load Balancer Controller installed
- ✅ Public ALB created automatically
- ✅ Webapp accessible at `http://<ALB_URL>`
- ✅ API accessible at `http://<ALB_URL>/api`
- ✅ Shared ALB for cost optimization

**Next Steps:**
1. Add custom domain
2. Add HTTPS with ACM certificate
3. Configure WAF rules (optional)
4. Set up CloudWatch alarms for ALB
