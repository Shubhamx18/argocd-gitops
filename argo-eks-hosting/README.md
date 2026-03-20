# ArgoCD HTTPS Hosting on EKS

Step-by-step guide to set up ArgoCD with HTTPS domain support on AWS EKS using `eksctl`.

---

## Prerequisites

Before starting, ensure you have the following tools installed and configured:

### 1. AWS CLI

Install and configure with your credentials:

```bash
aws configure
```

You will be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region: `ap-south-1`
- Output format: `json`

> Your AWS credentials must have sufficient permissions to create and manage EKS clusters and related resources. Use an account with `AdministratorAccess` or a custom policy covering EKS, EC2, IAM, and VPC.

### 2. eksctl

```bash
# Linux/WSL
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Verify:

```bash
eksctl version
```

### 3. kubectl

```bash
kubectl version --client
```

> Install guide: https://kubernetes.io/docs/tasks/tools/

### 4. Helm

```bash
helm version
```

> Install guide: https://helm.sh/docs/intro/install/

### 5. Domain Name

You need a registered domain name to configure HTTPS. You will point a subdomain (e.g. `argocd.yourdomain.com`) to the AWS load balancer created in later steps.

---

## Folder Structure

```
eks-https/
├── letsencrypt-issuer.yaml     # ClusterIssuer for Let's Encrypt (staging + prod)
└── argocd-ingress.yaml         # Ingress resource for ArgoCD with TLS
```

---

## Step-by-Step Setup

### Step 1 — Create EKS Cluster

Create an EKS cluster without a node group first. The node group will be added separately in the next step.

```bash
eksctl create cluster \
  --name argocd-cluster \
  --region ap-south-1 \
  --without-nodegroup
```

> This takes around 10-15 minutes. eksctl will create the VPC, subnets, and control plane automatically.

---

### Step 2 — Verify Cluster Creation

```bash
eksctl get clusters --region ap-south-1
```

You should see `argocd-cluster` listed with status `ACTIVE`.

---

### Step 3 — Associate IAM OIDC Provider

This is required for IAM roles for service accounts (IRSA) to work correctly with EKS.

```bash
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=argocd-cluster \
  --approve
```

---

### Step 4 — Create Node Group

Add a managed node group to the cluster. This creates EC2 instances that will run your workloads.

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

| Parameter | Value | Description |
|-----------|-------|-------------|
| `--node-type` | `t3.medium` | Instance type — 2 vCPU, 4GB RAM |
| `--nodes` | `2` | Initial number of nodes |
| `--nodes-min` | `1` | Minimum nodes for autoscaling |
| `--nodes-max` | `3` | Maximum nodes for autoscaling |
| `--node-volume-size` | `20` | Disk size in GB per node |

---

### Step 5 — Configure kubectl

Update your local kubeconfig to connect to the new cluster:

```bash
aws eks update-kubeconfig --region ap-south-1 --name argocd-cluster
```

Verify the nodes are in `Ready` state:

```bash
kubectl get nodes
```

You should see 2 nodes listed with status `Ready`.

---

### Step 6 — Install ArgoCD

Create a dedicated namespace for ArgoCD:

```bash
kubectl create namespace argocd
```

Install ArgoCD using the official manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all ArgoCD pods are running and ready:

```bash
kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s
```

Verify all pods are running:

```bash
kubectl get pods -n argocd
```

---

### Step 7 — Install NGINX Ingress Controller

The NGINX Ingress Controller acts as the entry point for all external traffic into the cluster. We need to enable SSL passthrough so HTTPS traffic is forwarded directly to ArgoCD without being terminated at the ingress layer.

Add the Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install the ingress controller with SSL passthrough enabled:

```bash
helm install my-ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.enableSSLPassthrough=true
```

Verify the ingress controller is running:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

> The `ingress-nginx-controller` service should have an `EXTERNAL-IP` assigned. This is the AWS Load Balancer address you will use for DNS.

---

### Step 8 — Install cert-manager

cert-manager is a Kubernetes add-on that automatically provisions and manages TLS certificates from Let's Encrypt.

Install cert-manager:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

Wait for all cert-manager pods to be ready:

```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager \
  --timeout=300s
```

Verify cert-manager is running:

```bash
kubectl get pods -n cert-manager
```

You should see 3 pods running: `cert-manager`, `cert-manager-cainjector`, and `cert-manager-webhook`.

---

### Step 9 — Configure Let's Encrypt Issuer

A `ClusterIssuer` tells cert-manager how to request certificates from Let's Encrypt.

Create `letsencrypt-issuer.yaml` — replace `<your-email@example.com>` with your actual email in both issuers:

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

Two issuers are defined:

| Issuer | Purpose |
|--------|---------|
| `letsencrypt-staging` | For testing — avoids rate limits, certificate is not browser-trusted |
| `letsencrypt-prod` | For production — issues a fully trusted certificate |

> Start with `letsencrypt-staging` to test your setup without hitting Let's Encrypt rate limits. Once everything works, switch to `letsencrypt-prod`.

Apply the issuer:

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

Verify the issuers were created:

```bash
kubectl get clusterissuer
```

Both issuers should show `Ready: True`.

---

### Step 10 — Create ArgoCD Ingress

The Ingress resource defines how external traffic reaches ArgoCD and instructs cert-manager to issue a TLS certificate for your domain.

Create `argocd-ingress.yaml` — replace `argocd.yourdomain.com` with your actual domain:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    # Forwards HTTPS traffic directly to ArgoCD without terminating SSL at ingress
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # Tells NGINX the backend (ArgoCD) is running HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # Tells cert-manager which issuer to use for the TLS certificate
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
    secretName: argocd-server-tls  # cert-manager will create this secret automatically
```

Apply the ingress:

```bash
kubectl apply -f argocd-ingress.yaml
```

Verify the ingress was created:

```bash
kubectl get ingress -n argocd
```

---

### Step 11 — Update DNS

Get the external load balancer hostname:

```bash
kubectl get svc -n ingress-nginx
```

Look for the `EXTERNAL-IP` of the `my-ingress-nginx-controller` service. Copy this value.

Now go to your domain registrar's DNS settings and add a **CNAME record**:

- **Type:** `CNAME`
- **Host / Name:** `argocd` (or `argocd.yourdomain.com` depending on your registrar)
- **Value / Points to:** the `EXTERNAL-IP` from the command above

> DNS propagation can take a few minutes to up to 48 hours depending on your registrar. You can check propagation at https://dnschecker.org.

---

## Verify SSL Certificate

After applying the ingress, cert-manager will automatically request a certificate from Let's Encrypt.

Check if the certificate was issued:

```bash
kubectl get certificate -n argocd
```

The `READY` column should show `True`. If it shows `False`, check the details:

```bash
kubectl describe certificate argocd-server-tls -n argocd
```

Check cert-manager logs for any errors:

```bash
kubectl logs -n cert-manager deployment/cert-manager
```

Verify the TLS secret was created by cert-manager:

```bash
kubectl get secret argocd-server-tls -n argocd
```

> **Note:** It may take 5-10 minutes for the certificate to be issued. A `Pending` or `False` status is normal during this time — it will automatically update to `True` once Let's Encrypt completes the challenge.

If the certificate stays in `Pending` for more than 15 minutes, check:
- DNS is correctly pointing to the load balancer
- The domain is publicly accessible from the internet
- cert-manager pods are all running: `kubectl get pods -n cert-manager`

---

## Access ArgoCD

Once the certificate is ready, open your browser in **Incognito mode** and go to `https://argocd.yourdomain.com`.

> Use Incognito mode to avoid browser cache issues. If you previously visited the URL over HTTP, your browser may have cached a redirect.

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

- **URL:** `https://argocd.yourdomain.com`
- **Username:** `admin`
- **Password:** output of the command above

> Change the default password immediately after first login via **User Info → Update Password** in the ArgoCD UI.

---

## Deploy an Application

Once logged in, you can deploy applications through the ArgoCD UI or CLI. For example, deploy the online-shop app from the `argocd-demos` repo using path `multicluster/online-shop`.

---

## Cleanup

When you are done, delete the entire cluster and all associated AWS resources:

```bash
eksctl delete cluster --name argocd-cluster --region ap-south-1
```

This will automatically remove:
- All running pods and applications
- Node groups and EC2 instances
- Load balancers
- VPC, subnets, and all networking components
- Security groups and IAM roles created by eksctl

> Cleanup takes approximately 10-15 minutes. Do not interrupt the process.

---

## References

- [eksctl Documentation](https://eksctl.io/)
- [ArgoCD Installation Guide](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
