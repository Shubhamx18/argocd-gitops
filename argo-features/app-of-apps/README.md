# ArgoCD App of Apps

One root app that manages all other apps automatically.
Push a new app YAML to Git — ArgoCD deploys it. No manual steps needed.

```
main.yml (root)  →  watches apps/  →  deploys each app  →  All apps running
```

---

## Repository Structure

```
argocd-gitops/
├── argo-features/
│   └── app-of-apps/
│       ├── apps/
│       │   ├── netflix_app.yml        # Child app — deploys Netflix
│       │   └── portfolio-app.yaml     # Child app — deploys Portfolio
│       └── main.yml                   # Root app — watches apps/ folder
└── practicals/
    ├── netflix-manifests/
    │   ├── deployment.yaml
    │   └── service.yaml
    └── portfolio-manifest/
        ├── deployment.yml
        └── service.yml
```

---

## How the Flow Works

```
main.yml  (root app)
    └── Watches argo-features/app-of-apps/apps/
            |
            ├── netflix_app.yml
            │       └── Points to practicals/netflix-manifests/
            │               └── Deploys Netflix Deployment + Service
            │
            └── portfolio-app.yaml
                    └── Points to practicals/portfolio-manifest/
                            └── Deploys Portfolio Deployment + Service
```

Add a new `.yml` file inside `apps/` and ArgoCD automatically picks it up and deploys it.

---

## Setup

### 1. Apply the Root App

This single command bootstraps everything:

```bash
kubectl apply -f argo-features/app-of-apps/main.yml
```

ArgoCD registers `main.yml` as the root app and immediately starts watching the `apps/` folder.

### 2. Verify Child Apps Are Created

ArgoCD auto-creates `netflix-app` and `portfolio-app` from the files inside `apps/`:

```bash
argocd app list
```

You should see:

```
NAME            STATUS   SYNC    HEALTH
root-app        Synced   Synced  Healthy
netflix-app     Synced   Synced  Healthy
portfolio-app   Synced   Synced  Healthy
```

### 3. Verify the Deployments

```bash
kubectl get pods
kubectl get svc
```

---

## main.yml — Root App

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Shubhamx18/argocd-gitops.git
    targetRevision: main
    path: argo-features/app-of-apps/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

| Field | What it does |
|-------|--------------|
| `path: argo-features/app-of-apps/apps` | Root app watches this folder for child apps |
| `namespace: argocd` | Child app objects are created in the argocd namespace |
| `prune: true` | Removes child apps deleted from Git automatically |
| `selfHeal: true` | Restores any manually changed or deleted resource |

---

## Adding a New App

Create a new YAML file inside `apps/` and push to Git:

```bash
git add . && git commit -m "Add my-new-app" && git push
```

ArgoCD detects the new file and deploys the app automatically.

---

## Testing GitOps

### Self-Healing Test

Delete a child app manually — ArgoCD recreates it from Git:

```bash
argocd app delete netflix-app
argocd app list
```

`root-app` detects the missing app and restores it automatically via `selfHeal: true`.

### Drift Detection Test

Delete a deployment — ArgoCD restores it:

```bash
kubectl delete deployment netflix-deployment
kubectl get deployment
```

Git is the single source of truth. Any drift from it gets auto-corrected.
