<div align="center">

# ArgoCD GitOps Repository

![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Package_Manager-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Modules](https://img.shields.io/badge/Modules-8-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)

A complete hands-on GitOps repository for managing Kubernetes deployments using ArgoCD — covering everything from basic setup to production-grade security, SSO, RBAC, monitoring, image automation, notifications, and multi-cluster deployments on EKS.

</div>

---

## Repository Structure

```
argocd-gitops/
│
├── 01_introduction/                    # GitOps concepts and ArgoCD overview
│
├── 02_core_concepts/                   # Applications, Projects, Sync Policies
│
├── 03_setup_installation/              # ArgoCD install on Kind and EKS
│   ├── README.md
│   └── setup_argocd.sh
│
├── argo-features/                      # Advanced ArgoCD features
│   ├── applicationsets/                # Auto-generate apps across clusters/envs
│   ├── app-projects/                   # Team and environment isolation
│   ├── sync-waves/                     # Control deployment order
│   └── resource-hooks/                 # PreSync, PostSync, SyncFail hooks
│
├── argocd-monitoring/                  # Prometheus + Grafana observability
│
├── image-updater/                      # Automatic image tag updates from registry
│
├── notifications/                      # Slack, email, webhook alerts
│
├── practicals/                         # Hands-on exercises and real examples
│
├── security-scaling/                   # RBAC, SSO, High Availability
│   ├── output-images/                  # Documentation screenshots
│   ├── rbac/                           # Local users and role-based access control
│   │   ├── argocd-user-cm.yaml
│   │   ├── argocd-rbac-cm.yaml
│   │   └── README.md
│   └── sso/                            # GitHub SSO via Dex OIDC
│       ├── argocd-cm.yaml
│       ├── dex-secret.yaml
│       ├── argocd-rbac-cm.yaml
│       └── README.md
│
└── README.md
```

---

## Module Index

### Setup & Installation — `03_setup_installation/`

| File | Description |
|------|-------------|
| `README.md` | Step-by-step ArgoCD install on Kind cluster and AWS EKS |
| `setup_argocd.sh` | Shell script to automate full ArgoCD setup |

---

### ArgoCD Features — `argo-features/`

| Topic | Description |
|-------|-------------|
| `applicationsets/` | Template and auto-generate applications across multiple clusters or environments using generators |
| `app-projects/` | Isolate teams and environments — restrict which repos, clusters, and namespaces a team can access |
| `sync-waves/` | Control the order resources are deployed using `argocd.argoproj.io/sync-wave` annotations |
| `resource-hooks/` | Run Kubernetes Jobs at specific sync phases — `PreSync`, `PostSync`, `SyncFail` |

---

### ArgoCD Monitoring — `argocd-monitoring/`

| Topic | Description |
|-------|-------------|
| Prometheus | Scrapes ArgoCD metrics — sync status, app health, controller queue depth |
| Grafana | Pre-built dashboards for application health, sync history, cluster performance |
| Alerting | Alert rules for sync failures, degraded applications, and controller errors |

---

### Image Updater — `image-updater/`

| Topic | Description |
|-------|-------------|
| Registry Watching | Monitors Docker Hub, ECR, GCR for new image tags |
| Auto Update | Automatically commits updated image tags back to Git |
| Update Strategies | Supports `semver`, `latest`, and `digest` strategies |
| CI/CD Loop | Closes the loop — no manual manifest changes needed after a build |

---

### Notifications — `notifications/`

| Topic | Description |
|-------|-------------|
| Slack | Alerts for sync success, sync failure, and app health degradation |
| Email | Email notifications on critical application events |
| Webhooks | Custom webhook integrations for any external system |
| Triggers | Rule-based triggers — notify only on specific events and conditions |

---

### Security & Scaling — `security-scaling/`

#### RBAC — `security-scaling/rbac/`

| File | Description |
|------|-------------|
| `argocd-user-cm.yaml` | Defines local users with `apiKey` and `login` capabilities |
| `argocd-rbac-cm.yaml` | Defines roles, permission policies, and user-role mappings |
| `README.md` | Full hands-on guide — create users, apply RBAC, verify permissions |

Local users and roles configured in this repo:

| User | Role | Access |
|------|------|--------|
| `shubham` | `role:admin` | Full access — get, create, update, delete, sync |
| `gulshankumar` | `role:developer` | Limited — get, sync, update applications only |
| `alex` | `role:readonly` | View only — get applications and projects |

#### SSO — `security-scaling/sso/`

| File | Description |
|------|-------------|
| `dex-secret.yaml` | Kubernetes Secret storing GitHub OAuth clientID and clientSecret |
| `argocd-cm.yaml` | ArgoCD ConfigMap with Dex connector and GitHub org configuration |
| `argocd-rbac-cm.yaml` | RBAC mapping GitHub user/email to ArgoCD roles |
| `README.md` | Full hands-on guide — GitHub OAuth app, Dex setup, org membership, RBAC |

#### High Availability

| Component | HA Configuration |
|-----------|-----------------|
| ArgoCD API Server | Multiple replicas behind LoadBalancer |
| Repo Server | Scales horizontally for parallel Git operations |
| Application Controller | Leader election across replicas |
| Redis | HA mode with Sentinel |

---

### Practicals — `practicals/`

| Topic | Description |
|-------|-------------|
| Deploy applications | Deploy real apps using ArgoCD UI and CLI |
| Helm integration | Manage Helm chart deployments through ArgoCD |
| Sync management | Manual sync, force sync, rollback, and diff |
| Multi-cluster | Register and deploy to multiple clusters from one ArgoCD instance |

---

## Getting Started

### Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `kubectl` | Interact with Kubernetes | [Install](https://kubernetes.io/docs/tasks/tools/) |
| `argocd` CLI | Manage ArgoCD from terminal | [Install](https://argo-cd.readthedocs.io/en/stable/cli_installation/) |
| `helm` | Install ArgoCD and components | [Install](https://helm.sh/docs/intro/install/) |
| `kind` | Local Kubernetes cluster | [Install](https://kind.sigs.k8s.io/docs/user/quick-start/) |
| `eksctl` | AWS EKS cluster management | [Install](https://eksctl.io/) |

---

### Quick Setup — Local (Kind Cluster)

**1. Create Kind cluster:**

```bash
kind create cluster --name argocd-demo
```

**2. Install ArgoCD:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**3. Wait for pods:**

```bash
kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s
```

**4. Access UI:**

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Open `https://localhost:8080`

**5. Get admin password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

**6. Login via CLI:**

```bash
argocd login localhost:8080 --username admin --password <password> --insecure
```

> Change the admin password immediately after first login.

---

## Key Concepts

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Shubhamx18/argocd-gitops.git
    targetRevision: HEAD
    path: practicals/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Sync Policy

| Policy | Description |
|--------|-------------|
| Manual | Sync triggered manually via UI or CLI |
| `automated` | ArgoCD syncs automatically on every Git push |
| `selfHeal: true` | Reverts any manual cluster changes back to Git state |
| `prune: true` | Deletes cluster resources that are removed from Git |

### Application Health Status

| Status | Meaning |
|--------|---------|
| `Healthy` | All resources running as expected |
| `Degraded` | One or more resources are failing |
| `Progressing` | Deployment in progress |
| `Suspended` | Application is paused |
| `Missing` | Resource does not exist in cluster |
| `Unknown` | Health cannot be determined |

### Sync Status

| Status | Meaning |
|--------|---------|
| `Synced` | Cluster state matches Git state |
| `OutOfSync` | Cluster differs from what is defined in Git |

---

## RBAC Quick Reference

```
# Policy format
p, <role>, <resource>, <action>, <object>, <effect>

# Group mapping
g, <user/group>, <role>

# Examples
p, role:developer, applications, sync,   myproject/*, allow
p, role:developer, applications, get,    myproject/*, allow
p, role:readonly,  applications, get,    */*, allow
g, alice, role:developer
g, bob,   role:readonly
```

### Available Actions per Resource

| Resource | get | create | update | delete | sync |
|----------|:---:|:------:|:------:|:------:|:----:|
| applications | ✅ | ✅ | ✅ | ✅ | ✅ |
| projects | ✅ | ✅ | ✅ | ✅ | ❌ |
| repositories | ✅ | ✅ | ✅ | ✅ | ❌ |
| clusters | ✅ | ✅ | ✅ | ✅ | ❌ |
| accounts | ✅ | ❌ | ✅ | ❌ | ❌ |

---

## Best Practices

| Practice | Why |
|----------|-----|
| Use Git branches and PRs for all changes | Never edit the cluster directly — all changes must be traceable in Git |
| Set `policy.default: role:readonly` | Safe fallback — unauthenticated or unmapped users get no write access |
| Enable `selfHeal: true` | Prevents configuration drift when someone edits the cluster manually |
| Use `AppProjects` to isolate teams | Restrict which repos, clusters, and namespaces each team can access |
| Use `ApplicationSets` for multi-env/cluster | Manage hundreds of apps from a single template instead of duplicating YAML |
| Never commit plain secrets to Git | Use Sealed Secrets or External Secrets Operator for all sensitive values |
| Use SSO for team environments | Centralized auth via GitHub/Okta instead of managing individual local users |
| Enable ArgoCD Notifications | Alert the team immediately on sync failures or degraded applications |
| Disable built-in `admin` after setup | Eliminate a high-privilege static credential once proper users/SSO are configured |

---

## References

| Topic | Link |
|-------|------|
| ArgoCD Official Docs | [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/stable/) |
| ArgoCD RBAC | [Operator Manual — RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/) |
| ArgoCD SSO | [User Management](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/) |
| ArgoCD High Availability | [HA Setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/) |
| ArgoCD Notifications | [Notifications](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/) |
| ArgoCD Image Updater | [Image Updater Docs](https://argocd-image-updater.readthedocs.io/en/stable/) |
| ArgoCD ApplicationSets | [ApplicationSet Docs](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) |
| Sealed Secrets | [bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) |

---

<div align="center">

**Shubham**

[![GitHub](https://img.shields.io/badge/GitHub-Shubhamx18-181717?style=for-the-badge&logo=github)](https://github.com/Shubhamx18)

⭐ Star this repo if it helped you! ⭐

</div>
