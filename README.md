# Orange Wallet - GitOps Repository

**GitOps manifests for Orange Wallet microservices platform**

This repository contains Kubernetes manifests managed by ArgoCD for continuous deployment using the GitOps pattern.

---

## 📋 Repository Structure

```
orange-wallet-gitops/
├── base/                                # Base Kustomize manifests
│   ├── authentication-service/
│   ├── user-service/
│   ├── transaction-service/
│   ├── wallet-service/
│   └── api-gateway/
│
├── production/                          # Production overlays (image tags)
│   ├── authentication-service/
│   │   └── kustomization.yaml           # Updated by CI/CD
│   └── ... (other services)
│
├── infrastructure/                      # Infrastructure resources
│   ├── namespaces/
│   ├── ingress/
│   └── network-policies/
│
└── argocd/                              # ArgoCD Applications
    ├── app-of-apps.yaml                 # Root application
    └── applications/                    # Individual service apps
        ├── authentication-service.yaml
        └── ... (other services)
```

---

## 🚀 Deployment

### Prerequisites

- Kubernetes cluster (K3s)
- ArgoCD installed
- kubectl access to cluster

### Deploy All Services

Deploy the App-of-Apps to automatically create all applications:

```bash
kubectl apply -f argocd/app-of-apps.yaml
```

This single command will:
1. Create the `orange-wallet-apps` parent application
2. Automatically create all 5 child applications
3. Sync all manifests to the cluster
4. Monitor for changes continuously

### Verify Deployment

```bash
# View all applications
kubectl get applications -n argocd

# View pods
kubectl get pods -n orange-wallet

# View services
kubectl get svc -n orange-wallet
```

---

## 🔄 GitOps Workflow

### How It Works

```
Developer Push → CI/CD Pipeline → Build & Test → Docker Push
                                                      ↓
                                          Update image tag in GitOps repo
                                                      ↓
                                          ArgoCD detects change (3-min poll)
                                                      ↓
                                          ArgoCD syncs to cluster automatically
                                                      ↓
                                          New version deployed!
```

### Image Tag Updates

CI/CD pipeline automatically updates image tags in `production/{service}/kustomization.yaml`:

```yaml
images:
  - name: authentication-service
    newName: docker.io/teukumunawar/orange-pay-authentication-service
    newTag: abc123  # Updated by CI/CD with commit SHA
```

### Automatic Sync

ArgoCD is configured with automatic sync enabled:
- **Poll interval:** 3 minutes
- **Self-heal:** Yes (auto-correct drift)
- **Prune:** Yes (auto-delete removed resources)

---

## 📊 Services

| Service | Description | Port |
|---------|-------------|------|
| **authentication-service** | JWT authentication & authorization | 8080 |
| **user-service** | User management & profiles | 8080 |
| **transaction-service** | BNI VA transactions | 8080 |
| **wallet-service** | E-wallet balance management | 8080 |
| **api-gateway** | API gateway & routing | 8080 |

---

## 🔧 Manual Operations

### Sync Specific Service

```bash
# Via kubectl
kubectl patch application authentication-service -n argocd \
  --type merge -p '{"spec":{"syncPolicy":{"syncOptions":["CreateNamespace=true"]}}}'

# Via ArgoCD CLI
argocd app sync authentication-service
```

### Rollback Service

#### Via Git Revert

```bash
# Find the commit that updated the service
git log --oneline production/authentication-service/

# Revert the commit
git revert <commit-sha>
git push origin main

# ArgoCD will automatically sync the rollback
```

#### Via ArgoCD UI

1. Open ArgoCD UI
2. Click on service application
3. Go to "History and Rollback"
4. Select previous revision
5. Click "Rollback"

#### Via kubectl

```bash
# Rollback deployment directly
kubectl rollout undo deployment/authentication-service -n orange-wallet

# To specific revision
kubectl rollout undo deployment/authentication-service \
  --to-revision=3 -n orange-wallet
```

### Force Refresh

```bash
# Hard refresh (re-query Git repo)
argocd app get authentication-service --hard-refresh

# Force sync (ignore differences)
argocd app sync authentication-service --force
```

---

## 📈 Monitoring

### ArgoCD UI

Access ArgoCD dashboard:

```bash
# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Or via Ingress (if configured)
# https://argocd.orange-wallet.com
```

Login credentials:
- **Username:** admin
- **Password:**
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d
  ```

### Health Checks

```bash
# Check all applications
argocd app list

# Get application details
argocd app get authentication-service

# View sync status
argocd app get authentication-service -o yaml | grep -A10 syncStatus
```

### Logs

```bash
# Application logs
kubectl logs -n orange-wallet -l app=authentication-service --tail=100 -f

# ArgoCD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f

# ArgoCD server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server -f
```

---

## 🛠️ Development

### Local Testing

Test Kustomize builds locally:

```bash
# Test base manifests
kustomize build base/authentication-service/

# Test production overlay
kustomize build production/authentication-service/

# Validate YAML
kustomize build production/authentication-service/ | kubeval --strict
```

### Updating Manifests

**For base manifests (structure changes):**

1. Update files in `base/{service}/`
2. Commit and push
3. ArgoCD will detect and sync

**For image tags (deployments):**

CI/CD automatically updates `production/{service}/kustomization.yaml`. Do not update manually.

---

## 🔒 Security

### Secrets Management

Secrets are managed via External Secrets Operator:
- Database credentials
- JWT keys
- API keys
- Docker registry credentials

**Never commit secrets to this repository!**

### Access Control

ArgoCD RBAC controls who can:
- View applications
- Sync applications
- Rollback deployments
- Modify sync policies

---

## 🐛 Troubleshooting

### Application OutOfSync

**Symptoms:** ArgoCD shows "OutOfSync" status

**Solutions:**
```bash
# Hard refresh
argocd app get <app-name> --hard-refresh

# Force sync
argocd app sync <app-name> --force
```

### Sync Failed

**Symptoms:** Sync operation fails with errors

**Solutions:**
```bash
# View detailed error
argocd app get <app-name>

# Check events
kubectl get events -n orange-wallet --sort-by='.lastTimestamp'

# Check pod status
kubectl describe pod -n orange-wallet -l app=<service>
```

### Image Pull Failed

**Symptoms:** Pods in `ImagePullBackOff` state

**Solutions:**
```bash
# Verify image exists
docker pull docker.io/teukumunawar/orange-pay-authentication-service:<tag>

# Check image pull secret
kubectl get secret -n orange-wallet dockerhub-secret

# Check image tag in kustomization
cat production/authentication-service/kustomization.yaml | grep newTag
```

### Pods Not Starting

**Symptoms:** Pods stuck in `Pending` or `CrashLoopBackOff`

**Solutions:**
```bash
# Check pod logs
kubectl logs -n orange-wallet <pod-name>

# Check pod events
kubectl describe pod -n orange-wallet <pod-name>

# Check resource availability
kubectl describe nodes
```

---

## 📚 References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [GitOps Principles](https://opengitops.dev/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

## 🤝 Contributing

This repository is managed by CI/CD automation. Manual changes should be:
1. Discussed with the team
2. Tested locally with Kustomize
3. Validated in development environment first
4. Reviewed via Pull Request

---

## 📞 Support

- **CI/CD Issues:** Check [GITOPS-MIGRATION-CHECKLIST.md](../be-capstone-project/.github/GITOPS-MIGRATION-CHECKLIST.md)
- **Application Repo:** [be-capstone-project](https://github.com/group-2-odp-bni/be-capstone-project)
- **ArgoCD UI:** https://argocd.orange-wallet.com

---

**Last Updated:** 2025-11-10
**Maintained by:** Orange Wallet Team (Group 2 - ODP BNI)
