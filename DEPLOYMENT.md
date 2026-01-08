# Portfolio UI Deployment Guide

Complete guide for deploying AstroZen Portfolio to K3s cluster with Helm and GitLab CI/CD.

## Prerequisites

- K3s cluster running on VPS
- Helm 3 installed
- GitLab Runner configured
- kubectl access
- Domain with DNS configured

## Quick Setup

### 1. GitLab Variables

Add these in Settings → CI/CD → Variables:

- `KUBECONFIG_CONTENT`: Your k3s kubeconfig (base64 encoded)
- `GITLAB_DEPLOY_TOKEN_USERNAME`: Deploy token username
- `GITLAB_DEPLOY_TOKEN`: Deploy token password

Get kubeconfig:
```bash
cat /etc/rancher/k3s/k3s.yaml | base64 -w 0
```

### 2. Update Configuration

Edit `helm/values-production.yaml`:
- Update image repository
- Update domain name
- Update ingress settings

Edit `.gitlab-ci.yml`:
- Update runner tags

### 3. Deploy

Push to master/main branch. Pipeline will:
1. Lint and security scan
2. Build Docker image
3. Deploy canary (automatic)
4. Promote to stable (manual)

## Manual Deployment

```bash
helm upgrade --install portfolio-ui ./helm \
  --namespace portfolio-ui \
  --create-namespace \
  --values ./helm/values-production.yaml
```

## Monitoring

```bash
kubectl get pods -n portfolio-ui
kubectl logs -n portfolio-ui -l app=portfolio-ui -f
```

See full documentation in the file for detailed troubleshooting.
