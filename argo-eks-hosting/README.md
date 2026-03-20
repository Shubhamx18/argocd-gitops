# ArgoCD HTTPS Hosting on EKS

Simple step-by-step guide to set up ArgoCD with HTTPS domain support on AWS EKS using direct `eksctl` commands.

---

## Prerequisites

Before starting, ensure you have:

1. **AWS CLI** installed and configured (with `eks` related policy, use `admin` for this guide)

   ```bash
   aws configure
   ```

   > You need AWS Access Key ID and Secret Access Key with appropriate permissions to create and manage EKS clusters and related resources.

2. **eksctl** installed

   ```bash
   # Linux/WSL
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```

   Verify installation:

   ```bash
   eksctl version
   ```

3. **kubectl** installed

   ```bash
   kubectl version --client
   ```

4. **Helm** installed

   ```bash
   helm version
   ```

   > Install Guide: https://helm.sh/docs/intro/install/

5. **Domain name** registered.

---

## Step-by-Step Setup

### Step 1 — Create EKS Cluster

```bash
eksctl create cluster --name argocd-cluster --region ap-south-1 --without-nodegroup
```

---

### Step 2 — Verify Cluster Creation

```bash
eksctl get clusters --region ap-south-1
```

---

### Step 3 — Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=argocd-cluster \
  --approve
```

---

### Step 4 — Create Node Group

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

---

### Step 5 — Verify Cluster Access

Update kubeconfig:

```bash
aws eks update-kubeconfig --region ap-south-1 --name argocd-cluster
```

Verify nodes are ready:

```bash
kubectl get nodes
```

---

### Step 6 — Install ArgoCD

Create namespace:

```bash
kubectl create namespace argocd
```

Install ArgoCD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for pods to be ready:

```bash
kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s
```

---

### Step 7 — Install NGINX Ingress Controller

Add Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install ingress controller with SSL passthrough enabled:

```bash
helm install my-ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.enableSSLPassthrough=true
```

---

### Step 8 — Install cert-manager for SSL Certificates

Install cert-manager:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager \
  --timeout=300s
```

---

### Step 9 — Configure Let's Encrypt and HTTPS

#### 9.1 — Create `letsencrypt-issuer.yaml`

This file defines two ClusterIssuers — one for **staging** (for testing, avoids Let's Encrypt rate limits) and one for **production** (for a trusted certificate).

Replace `<your-email@example.com>` with your actual email in both issuers:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <your-email@example.com>  # Replace with your email
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
    email: <your-email@example.com>  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```

| Issuer | Server | Use Case |
|--------|--------|---------|
| `letsencrypt-staging` | `acme-staging-v02.api.letsencrypt.org` | Testing — avoids rate limits, certificate not trusted by browsers |
| `letsencrypt-prod` | `acme-v02.api.letsencrypt.org` | Production — fully trusted certificate |

Apply the issuer:

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

---

#### 9.2 — Create `argocd-ingress.yaml`

Replace `argocd.yourdomain.com` with your actual domain:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    # Forward HTTPS traffic directly to the backend without terminating SSL
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # Specify that the backend service uses HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # Use production issuer for a trusted certificate
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.yourdomain.com  # Replace with your domain
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
    - argocd.yourdomain.com  # Replace with your domain
    secretName: argocd-server-tls  # TLS secret created automatically by cert-manager
```

Apply the ingress:

```bash
kubectl apply -f argocd-ingress.yaml
```

> **Note:** To test first with staging, change `cert-manager.io/cluster-issuer: letsencrypt-prod` to `letsencrypt-staging`. Once verified, switch back to `letsencrypt-prod` and re-apply.

---

### Step 10 — Update DNS and Access ArgoCD

Get the load balancer hostname:

```bash
kubectl get svc -n ingress-nginx
```

Copy the `EXTERNAL-IP` value from the output and point your domain `argocd.yourdomain.com` to it by adding a **CNAME record** in your domain's DNS settings.

| DNS Record Type | Host | Value |
|----------------|------|-------|
| `CNAME` | `argocd.yourdomain.com` | `<EXTERNAL-IP from above>` |

---

## Verify SSL Certificate

After applying the ingress, cert-manager will automatically request a Let's Encrypt certificate.

Check certificate status:

```bash
kubectl get certificate -n argocd
```

Check certificate details:

```bash
kubectl describe certificate argocd-server-tls -n argocd
```

Check cert-manager logs if there are issues:

```bash
kubectl logs -n cert-manager deployment/cert-manager
```

Verify the TLS secret was created:

```bash
kubectl get secret argocd-server-tls -n argocd
```

The certificate should show `Ready: True` status. If not, verify:

| Check | Command |
|-------|---------|
| DNS is pointing to load balancer | `nslookup argocd.yourdomain.com` |
| Domain is publicly accessible | Test from browser or external tool |
| cert-manager pods are running | `kubectl get pods -n cert-manager` |
| cert-manager logs for errors | `kubectl logs -n cert-manager deployment/cert-manager` |

> **Note:** It may take 5-10 minutes for the SSL certificate to be issued by Let's Encrypt. A `Pending` status is normal during this time — it will automatically update to `True` once issued.

---

## Access ArgoCD via HTTPS

Open your browser in **Incognito mode** and navigate to `https://argocd.yourdomain.com` (replace with your actual domain).

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

| Field | Value |
|-------|-------|
| URL | `https://argocd.yourdomain.com` |
| Username | `admin` |
| Password | Output of command above (change after first login) |

---

## Deploy an Application

Deploy the online-shop (or any other app from `argocd-demos` repo) used in previous examples. Use path `multicluster/online-shop`.

---

## Cleanup

To destroy the cluster and all resources:

```bash
eksctl delete cluster --name argocd-cluster --region ap-south-1
```

This command will:

| Resource | Action |
|----------|--------|
| All applications and pods | Deleted |
| Node groups and EC2 instances | Terminated |
| VPC, subnets, networking components | Removed |
| Security groups | Removed |

> Complete cleanup takes approximately 10-15 minutes.

---

## References

- [eksctl Documentation](https://eksctl.io/)
- [ArgoCD Installation Guide](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Let's Encrypt](https://letsencrypt.org/)
