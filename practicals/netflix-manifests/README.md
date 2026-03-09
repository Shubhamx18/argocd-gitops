# Netflix Deployment with ArgoCD — Declarative Approach

> 📌 **Declarative = You describe the desired state in YAML, push to Git, and ArgoCD handles the rest.**  
> No manual `kubectl apply`. No running commands every time. Git is the single source of truth.

```
📝 Write YAML  →  ⬆️ Push to Git  →  👁️ ArgoCD watches  →  ✅ Cluster matches Git
```

![ArgoCD Netflix](declarative_netflix.png)

---

## Why Declarative?

| ❌ Imperative (manual) | ✅ Declarative (ArgoCD) |
|---|---|
| You run `kubectl apply` every time | ArgoCD syncs automatically from Git |
| Easy to forget what's deployed | Git history = exact record of everything |
| Config drifts silently | ArgoCD detects drift and auto-heals |

---

## Folder Structure

```
netflix-manifests/
├── deployment.yaml   # What to run (image, replicas)
├── service.yaml      # How to expose it (port)
└── application.yaml  # Tells ArgoCD: watch this Git repo
```

> `application.yaml` is the key file — it's what makes this **declarative**.  
> You apply it once and ArgoCD takes ownership from there.

---

## Deploy

```bash
git clone https://github.com/Shubhamx18/argocd-gitops.git
cd argocd-gitops/practicals/netflix-manifests
kubectl apply -f application.yaml
```

That's it. ArgoCD reads the repo and deploys `deployment.yaml` + `service.yaml` automatically.

---

## Destination Server in `application.yaml`

| Where | Set this |
|---|---|
| Local | `https://kubernetes.default.svc` |
| EC2 | Run `argocd cluster list` → copy your cluster URL |

---

## Verify

```bash
kubectl get pods
kubectl get svc
```

---

## Access the App

**Local:**
```bash
kubectl port-forward svc/netflix-service 3000:3000
```
→ `http://localhost:3000`

**EC2:**
```bash
kubectl port-forward svc/netflix-service 3000:3000 --address 0.0.0.0
```
→ `http://<EC2-PUBLIC-IP>:3000`

> Ensure port `3000` is open in your EC2 Security Group inbound rules.
