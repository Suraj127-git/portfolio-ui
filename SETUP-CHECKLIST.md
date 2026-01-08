# K3s Deployment Setup Checklist

Quick checklist for deploying Portfolio UI to K3s cluster.

## Before You Start

- [ ] K3s cluster is running on VPS
- [ ] You have SSH access to VPS
- [ ] Domain DNS points to VPS IP
- [ ] GitLab project is created

## Step 1: VPS Setup (One-time)

### Install GitLab Runner
```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner
```

### Register Runner
Get token from GitLab: Settings → CI/CD → Runners → New project runner

```bash
sudo gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token YOUR_TOKEN \
  --executor shell \
  --description "portfolio-frontend-services" \
  --tag-list "portfolio-frontend-services"
```

### Install Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
```

### Install Ingress Controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

### Install Cert-Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

### Create Let's Encrypt Issuer
```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Step 2: GitLab Configuration

### Create Deploy Token
Settings → Repository → Deploy tokens
- Name: portfolio-deploy
- Scopes: read_registry
- Save username and token

### Add CI/CD Variables
Settings → CI/CD → Variables → Add variable

1. KUBECONFIG_CONTENT
```bash
# On VPS:
cat /etc/rancher/k3s/k3s.yaml | base64 -w 0
```

2. GITLAB_DEPLOY_TOKEN_USERNAME
   - Your deploy token username

3. GITLAB_DEPLOY_TOKEN
   - Your deploy token password

## Step 3: Update Project Files

### Update helm/values-production.yaml
```yaml
deployment:
  image:
    repository: your-username/portfolio-ui  # ← Change this

ingress:
  hosts:
    - host: portfolio.yourdomain.com  # ← Change this
  tls:
    - hosts:
        - portfolio.yourdomain.com  # ← Change this
```

### Update .gitlab-ci.yml
```yaml
tags:
  - portfolio-frontend-services  # ← Match your runner tag
```

## Step 4: Deploy

- [ ] Commit and push to master/main branch
- [ ] Watch pipeline in GitLab CI/CD → Pipelines
- [ ] Pipeline stages:
  - Lint (automatic)
  - Security (automatic)
  - Build (automatic)
  - Deploy Canary (automatic)
  - Promote Canary (manual) ← Click this after testing
- [ ] Visit your domain to verify

## Verification Commands

```bash
# Check pods
kubectl get pods -n portfolio-ui

# Check ingress
kubectl get ingress -n portfolio-ui

# Check certificate
kubectl get certificate -n portfolio-ui

# View logs
kubectl logs -n portfolio-ui -l app=portfolio-ui --tail=50
```

## Common Issues

### Pods not starting?
```bash
kubectl describe pod <pod-name> -n portfolio-ui
```

### Certificate not issued?
```bash
kubectl describe certificate portfolio-ui-tls -n portfolio-ui
```

### Can't pull image?
Check deploy token and recreate secret:
```bash
kubectl delete secret gitlab-registry -n portfolio-ui
# Pipeline will recreate it
```

## Next Steps

- [ ] Test canary deployment
- [ ] Promote to stable
- [ ] Monitor logs
- [ ] Set up monitoring (optional)
- [ ] Configure backups (optional)

## Helpful Commands

```bash
# View Helm releases
helm list -n portfolio-ui

# Rollback deployment
helm rollback portfolio-ui -n portfolio-ui

# Delete deployment
helm uninstall portfolio-ui -n portfolio-ui

# View all resources
kubectl get all -n portfolio-ui
```
