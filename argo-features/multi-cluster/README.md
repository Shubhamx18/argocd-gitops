# ArgoCD Multi-Cluster Management

One ArgoCD instance managing multiple Kubernetes clusters from a single control plane.

```
ArgoCD (argocd-cluster)  →  in-cluster (dev)     →  Nginx
                          →  prod-kind (prod)     →  Netflix
```

---

## Repository Structure

```
argocd-gitops/
└── argo-features/
    └── multi-cluster/
        ├── nginx.yml        # App manifest — deploys Nginx to in-cluster (dev)
        └── netflix.yml      # App manifest — deploys Netflix to prod-kind (prod)
```

---

## How It Works

By default ArgoCD manages only the cluster it is installed on (`in-cluster`).
By registering additional clusters with `argocd cluster add`, ArgoCD can deploy apps to multiple target clusters from a single control plane.

```
ArgoCD runs in  →  kind-argocd-cluster
                        |
                        ├── nginx.yml    →  in-cluster   (dev)   →  Nginx app
                        └── netflix.yml  →  prod-kind    (prod)  →  Netflix app
```

---

## Prerequisites

- `kind` installed and running
- ArgoCD installed in `kind-argocd-cluster`
- `argocd` CLI installed and logged in
- `kubectl` installed and configured

---

## Setup

### 1. Create the Prod Cluster

Create `kind_config.yml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "<your-EC2-private-IP>"   # run: hostname -I
  apiServerPort: 33894                          # must differ from argocd-cluster port
nodes:
  - role: control-plane
    image: kindest/node:v1.33.1
```

Create the cluster:

```bash
kind create cluster --name prod-kind --config kind_config.yml
```

Verify contexts:

```bash
kubectl config get-contexts
```

You should see:

```
kind-argocd-cluster
kind-prod-kind
```

---

### 2. Add Clusters to ArgoCD

Switch context back to the ArgoCD cluster:

```bash
kubectl config use-context kind-argocd-cluster
```

Register both clusters:

```bash
argocd cluster add kind-argocd-cluster --name argocd-cluster --insecure
argocd cluster add kind-prod-kind --name prod-cluster --insecure
```

Verify:

```bash
argocd cluster list
```

You should see:

```
SERVER                          NAME            STATUS
https://kubernetes.default.svc  in-cluster      Unknown
https://<EC2-IP>:33893          argocd-cluster  Unknown
https://<EC2-IP>:33894          prod-cluster    Unknown
```

After apps are deployed, status changes to `Successful`.

---

### 3. Deploy Nginx to in-cluster (Dev)

```bash
kubectl apply -f argo-features/multi-cluster/nginx.yml -n argocd
```

---

### 4. Deploy Netflix to prod-kind (Prod)

```bash
kubectl apply -f argo-features/multi-cluster/netflix.yml -n argocd
```

---

### 5. Verify in ArgoCD UI

Go to **Applications** in the ArgoCD UI.

You should see:

| App | Cluster | Status |
|-----|---------|--------|
| `nginx-dev` | `in-cluster` | Healthy |
| `netflix-prod` | `prod-cluster` | Healthy |

The **CLUSTER** column confirms which cluster each app is deployed to.

---

### 6. Verify via CLI

```bash
argocd app list
```

Check pods in dev cluster (in-cluster):

```bash
kubectl get pods,svc -n default
```

Check pods in prod cluster:

```bash
kubectl --context kind-prod-kind get pods,svc -n default
```

---

### 7. Access the Apps

Nginx (Dev):

```bash
kubectl port-forward svc/nginx-service 8081:80 --address=0.0.0.0 &
```

Open `http://<EC2-PUBLIC-IP>:8081`

Netflix (Prod):

```bash
kubectl --context kind-prod-kind port-forward svc/netflix-service 8082:80 --address=0.0.0.0 &
```

Open `http://<EC2-PUBLIC-IP>:8082`

> Make sure the ports are open in your EC2 Security Group inbound rules.

---

## Key Takeaways

- One ArgoCD instance can manage many clusters.
- Register clusters with `argocd cluster add`.
- Use `spec.destination.server` in each app YAML to target a specific cluster.
- ArgoCD control plane runs in `kind-argocd-cluster` and connects to remote clusters.
- In production, clusters are connected over VPN or private network peering.
