# Nginx GitOps Deployment with ArgoCD

Deploy an Nginx application to Kubernetes using ArgoCD's GitOps workflow.

![ArgoCD Nginx Application](argocd_nginx.png)

---

## 📁 Repository Structure

```
argocd-gitops/
└── practicals/
    └── nginx-manifests/
        ├── deployment.yaml
        └── service.yaml
```

---

## 🚀 Create & Deploy the Application

### 1. Create the Application in ArgoCD UI

Open the ArgoCD UI and click **NEW APP**, then fill in:

**General**

| Field            | Value     |
|------------------|-----------|
| Application Name | `nginx`   |
| Project          | `default` |
| Sync Policy      | `Manual`  |

**Source**

| Field          | Value                                              |
|----------------|----------------------------------------------------|
| Repository URL | `https://github.com/Shubhamx18/argocd-gitops.git` |
| Revision       | `HEAD`                                             |
| Path           | `practicals/nginx-manifests`                       |

**Destination**

| Field     | Value                            |
|-----------|----------------------------------|
| Cluster   | `https://kubernetes.default.svc` |
| Namespace | `default`                        |

Click **Create**.

---

### 2. Sync the Application

Click **SYNC** → **SYNCHRONIZE** in the ArgoCD UI.

ArgoCD will pull the manifests from GitHub and deploy them to the cluster.

---

### 3. Verify the Deployment

```bash
# Check pods are running
kubectl get pods

# Check service is created
kubectl get svc
```

Expected pod output:
```
NAME                                READY   STATUS    RESTARTS
nginx-deployment-xxxxx-xxxxx        1/1     Running   0
nginx-deployment-xxxxx-xxxxx        1/1     Running   0
```

---

### 4. Access the Application via Port Forward

```bash
kubectl port-forward svc/nginx-service 8081:80
```

Open your browser and navigate to:

```
http://localhost:8081
```

You should see the default **Welcome to nginx!** page.

---

## 🔄 Making Updates

1. Push changes to `practicals/nginx-manifests` in GitHub
2. Click **SYNC** in ArgoCD to apply the changes
3. ArgoCD will reconcile the cluster state with the new manifests

> 💡 To automate this, enable **Auto-Sync** in the application settings.
