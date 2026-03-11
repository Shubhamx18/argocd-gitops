# ArgoCD Notifications

ArgoCD Notifications automatically send alerts about application events (sync, health, errors) to external channels like **Slack, Email, Teams, Webhooks**, etc.

---

## How it Works

| Component | What it does |
|-----------|-------------|
| **Notification Controller** | Watches ArgoCD apps and fires notifications |
| **Triggers** | Define **when** to send (e.g., on sync failure) |
| **Templates** | Define **what** to send (message content) |
| **Subscriptions** | Define **where** to send (Email, Slack, etc.) |

---

## Prerequisites

- `kind` cluster running with ArgoCD installed
- `argocd` CLI installed and logged in
- Gmail account with **App Password** created

**Create Gmail App Password:**

1. Enable **2-Step Verification** on your Google account → [Google Help](https://support.google.com/accounts/answer/185839)
2. Go to https://myaccount.google.com/apppasswords
3. Enter app name: `ArgoCD SMTP` → click **Create**
4. Copy the **16-character password** (format: `xxxx xxxx xxxx xxxx`) — you won't see it again

---

## Repo Structure

```
argocd-gitops/
└── notifications/
    ├── secret-smtp.yaml
    ├── argocd-notifications-cm.yaml
    └── portfolio-app.yaml
```

![Notifications repo structure on GitHub](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/notifications/output-images/notifications_repo_structure.png)

---

## Setup

### Step 0 — Install catalog triggers and templates

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/notifications_catalog/install.yaml
```

---

### Step 1 — Create SMTP Secret

Use: `secret-smtp.yaml`

Replace `your-email@gmail.com` with your sender Gmail.
Replace `your-app-password` with the 16-character app password (keep the spaces).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  email-username: your-email@gmail.com
  email-password: "xxxx xxxx xxxx xxxx"
```

```bash
kubectl apply -f secret-smtp.yaml
```

---

### Step 2 — Apply Notification ConfigMap

Use: `argocd-notifications-cm.yaml`

Replace `<your-argocd-server>` with your instance public IP where ArgoCD is running.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.email.gmail: |
    host: smtp.gmail.com
    port: 587
    from: $email-username
    username: $email-username
    password: $email-password

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]

  template.app-deployed: |
    email:
      subject: "[ArgoCD] {{.app.metadata.name}} successfully deployed"
    message: |
      Application: {{.app.metadata.name}}
      Sync Status: {{.app.status.sync.status}}
      Health Status: {{.app.status.health.status}}
      Revision: {{.app.status.sync.revision}}

      Deployment completed successfully.

      ArgoCD Link: http://<your-argocd-server>/applications/{{.app.metadata.name}}

  template.app-health-degraded: |
    email:
      subject: "[ArgoCD] {{.app.metadata.name}} health is Degraded"
    message: |
      Application: {{.app.metadata.name}}
      Health Status: {{.app.status.health.status}}
      Sync Status: {{.app.status.sync.status}}

      Application health has degraded. Please investigate.

      ArgoCD Link: http://<your-argocd-server>/applications/{{.app.metadata.name}}
```

```bash
kubectl apply -f argocd-notifications-cm.yaml
```

> **How Secret + ConfigMap work together:**
> The `$email-username` and `$email-password` in the ConfigMap are **automatically resolved** from `argocd-notifications-secret` at runtime.
> This keeps credentials out of Git (Secret) while keeping config in Git (ConfigMap).

---

### Step 3 — Annotate the Application

Use: `portfolio-app.yaml`

Replace `<receiver@example.com>` with the recipient email.
Replace `<your-username>` with your GitHub username.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: portfolio-app
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.email: "<receiver@example.com>"
    notifications.argoproj.io/subscribe.on-health-degraded.email: "<receiver@example.com>"
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/argocd-gitops.git
    targetRevision: main
    path: practicals/portfolio-manifest
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f portfolio-app.yaml
```

Verify app is running in ArgoCD UI:

![portfolio-app resource tree in ArgoCD](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/notifications/output-images/portfolio_app_tree.png)

Access the app:

```bash
kubectl port-forward svc/portfolio-service -n default 3000:80 --address=0.0.0.0 &
```

Open: `http://<instance_public_ip>:3000`

---

## Demo

### Successful Deployment Email

Once `portfolio-app` syncs and becomes Healthy, you will receive:

![Deployment success email](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/notifications/output-images/deploy_success_email.png)

---

### Trigger a Degraded Alert

Edit the deployment manifest with a wrong image tag:

```yaml
image: your-image:v1    # v1 does not exist → ImagePullBackOff → Degraded
```

Commit and push. Then monitor the controller logs:

```bash
kubectl -n argocd logs deploy/argocd-notifications-controller --follow
```

When app becomes Degraded, the email fires automatically:

```
[ArgoCD] portfolio-app health is Degraded
```

---

## Common Triggers Reference

| Trigger | When it fires |
|---------|--------------|
| `on-sync-failed` | Sync failed (invalid manifest, cluster error) |
| `on-health-degraded` | App health becomes Degraded (e.g., CrashLoopBackOff) |
| `on-deployed` | Sync succeeded + app is Healthy |
| `on-sync-succeeded` | Sync completed successfully |
| `on-sync-running` | Sync operation started |
| `on-created` | Application resource created in ArgoCD |
| `on-deleted` | Application removed from ArgoCD |
| `on-sync-status-unknown` | ArgoCD cannot determine sync state |

---

> For more info read: [ArgoCD Notifications Docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/)

**Happy Learning!**
