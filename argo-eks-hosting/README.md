# ArgoCD HTTPS Hosting on EKS

Step-by-step guide to set up ArgoCD with HTTPS domain support on AWS EKS.

---

## Prerequisites

| Tool | Command |
|------|---------|
| AWS CLI | `aws configure` |
| eksctl | `eksctl version` |
| kubectl | `kubectl version --client` |
| Helm | `helm version` |

> AWS credentials must have sufficient permissions to create and manage EKS clusters.
> You also need a registered domain name.

---

## Setup

### Step 1 — Create EKS Cluster

```bash
eksctl create cluster --name argocd-cluster --region ap-south-1 --without-nodegroup
```

### Step 2 — Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=argocd-cluster \
  --approve
```

### Step 3 — Create Node Group

```bash
eksctl create nodegroup \
  --cluster=argocd-cluster \
  --region=ap-south-1 \
  --name=argocd-ng \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=1 \
  --nodes-max=3 \
  --node-volume-size=20 \
  --managed
```

### Step 4 — Configure kubectl

```bash
aws eks update-kubeconfig --region ap-south-1 --name argocd-cluster
kubectl get nodes
```

### Step 5 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s
```

### Step 6 — Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.enableSSLPassthrough=true
```

### Step 7 — Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager \
  --timeout=300s
```

### Step 8 — Apply Let's Encrypt Issuer

Create `letsencrypt-issuer.yaml` — replace `<your-email@example.com>` with your actual email:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <your-email@example.com>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: <your-email@example.com>
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```

> Use `letsencrypt-staging` for testing to avoid Let's Encrypt rate limits. Switch to `letsencrypt-prod` for a browser-trusted certificate.

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

### Step 9 — Apply ArgoCD Ingress

Create `argocd-ingress.yaml` — replace `argocd.yourdomain.com` with your actual domain:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.yourdomain.com
    secretName: argocd-server-tls
```

```bash
kubectl apply -f argocd-ingress.yaml
```

### Step 10 — Update DNS

Get the load balancer hostname:

```bash
kubectl get svc -n ingress-nginx
```

Add a **CNAME record** in your domain DNS pointing `argocd.yourdomain.com` to the `EXTERNAL-IP` from the output above.

---

## Verify SSL Certificate

```bash
# Check certificate status — should show Ready: True
kubectl get certificate -n argocd

# Check details if not ready
kubectl describe certificate argocd-server-tls -n argocd

# Check cert-manager logs if issues
kubectl logs -n cert-manager deployment/cert-manager

# Verify TLS secret was created
kubectl get secret argocd-server-tls -n argocd
```

> Certificate may take 5-10 minutes to be issued. `Pending` status is normal during this time.

---

## Access ArgoCD

Open `https://argocd.yourdomain.com` in Incognito mode.

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

- **Username:** `admin`
- **Password:** output of above command

> Change the password after first login.

---

## Cleanup

```bash
eksctl delete cluster --name argocd-cluster --region ap-south-1
```

> This removes all resources — nodes, VPC, subnets, security groups. Takes 10-15 minutes.

---

## References

- [eksctl Docs](https://eksctl.io/)
- [cert-manager Docs](https://cert-manager.io/docs/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [ArgoCD Installation](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)
