# ArgoCD SSO — GitHub Login via Dex (OIDC)

This folder configures Single Sign-On (SSO) for ArgoCD using **GitHub OAuth** through the built-in **Dex** identity provider. Once set up, users log in with their GitHub credentials — no separate ArgoCD passwords needed.

---

## Folder Structure

```
sso/
├── dex-secret.yaml          # Kubernetes Secret — stores GitHub OAuth clientID & clientSecret
├── argocd-cm.yaml           # ArgoCD ConfigMap — Dex connector config + local backup user
├── argocd-rbac-cm.yaml      # RBAC — maps GitHub user/org to ArgoCD roles
└── README.md                # This file
```

---

## How It Works

```
Browser → ArgoCD UI → Dex (OIDC Bridge) → GitHub OAuth → ArgoCD Session
```

- **Dex** is a built-in OIDC identity service bundled with ArgoCD
- It acts as a bridge between ArgoCD and external identity providers (GitHub, Google, LDAP, SAML)
- Users authenticate with their **GitHub account**
- **GitHub organization membership** is used for RBAC group mapping
- `scopes: '[groups, email]'` ensures both email and org groups are available for RBAC

### SSO vs Local Users

| | Local Users | SSO (GitHub via Dex) |
|--|------------|---------------------|
| Auth method | ArgoCD password | GitHub login |
| Credential management | Manual per user | Centralized via GitHub |
| Group-based RBAC | Manual mapping | Automatic via org membership |
| Best for | Small teams / CI-CD tokens | Enterprise / teams |

---

## File Details

### 1. `dex-secret.yaml` — OAuth Credentials Secret

Stores your GitHub OAuth App credentials as a Kubernetes Secret. ArgoCD reads `dex.github.clientId` and `dex.github.clientSecret` from this secret automatically.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
type: Opaque
stringData:
  dex.github.clientId: <GITHUB_CLIENT_ID>
  dex.github.clientSecret: <GITHUB_CLIENT_SECRET>
```

> **Never commit real credentials to Git.** Use [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [External Secrets Operator](https://external-secrets.io/) in production.

---

### 2. `argocd-cm.yaml` — Dex Connector Configuration

Configures ArgoCD to use Dex with GitHub as the identity provider. Also defines a local backup user for emergencies.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # ArgoCD URL (LOCAL SETUP)
  url: https://localhost:8080

  # =========================
  # DEX CONFIG (GitHub SSO)
  # =========================
  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: <GITHUB_CLIENT_ID>
        clientSecret: <GITHUB_CLIENT_SECRET>

        # Redirect URI — must exactly match what you set in the GitHub OAuth App
        redirectURI: https://localhost:8080/api/dex/callback

        orgs:
        - name: <GITHUB_ORG_NAME>

  # =========================
  # LOCAL BACKUP USER
  # =========================
  accounts.<USERNAME>: apiKey, login

  # Default role
  policy.default: role:readonly
```

---

### 3. `argocd-rbac-cm.yaml` — RBAC for SSO Users

Maps your GitHub user (by email or username) to the `role:repo-admin` role. Also sets `scopes` to read both `groups` and `email` from GitHub.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  scopes: '[groups, email]'

  policy.csv: |
    # =====================================================
    # ROLE: repo-admin (FULL CONTROL but structured)
    # =====================================================

    # Applications
    p, role:repo-admin, applications, get,    */*, allow
    p, role:repo-admin, applications, create, */*, allow
    p, role:repo-admin, applications, update, */*, allow
    p, role:repo-admin, applications, delete, */*, allow
    p, role:repo-admin, applications, sync,   */*, allow

    # Projects
    p, role:repo-admin, projects, get,    *, allow
    p, role:repo-admin, projects, create, *, allow
    p, role:repo-admin, projects, update, *, allow

    # Repositories
    p, role:repo-admin, repositories, get,    *, allow
    p, role:repo-admin, repositories, create, *, allow
    p, role:repo-admin, repositories, update, *, allow

    # Clusters
    p, role:repo-admin, clusters, get, *, allow

    # =====================================================
    # USER → ROLE MAPPING
    # =====================================================

    # Email mapping (primary)
    g, <USER_EMAIL>, role:repo-admin

    # GitHub username mapping (optional fallback)
    g, <GITHUB_USERNAME>, role:repo-admin

  policy.default: role:readonly
```

### `role:repo-admin` Permissions Summary

| Resource | get | create | update | delete | sync |
|----------|:---:|:------:|:------:|:------:|:----:|
| applications | ✅ | ✅ | ✅ | ✅ | ✅ |
| projects | ✅ | ✅ | ✅ | ❌ | ❌ |
| repositories | ✅ | ✅ | ✅ | ❌ | ❌ |
| clusters | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## Step-by-Step Setup Guide

### Prerequisites

- Kind cluster running
- ArgoCD installed in `argocd` namespace (installed via **Manifests**, not Helm)
- `kubectl` configured and `argocd` CLI logged in

---

### Step 1 — Register GitHub OAuth App

Go to: `GitHub → Settings → Developer settings → OAuth Apps → New OAuth App`

| Field | Value |
|-------|-------|
| Application name | `ArgoCD` |
| Homepage URL | `https://<instance_public_ip>:8080` |
| Authorization callback URL | `https://<instance_public_ip>:8080/api/dex/callback` |

Click **Register Application**, then click **Generate a new client secret**.

![Client Secret Generation](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/client-secret.png)

> **Use HTTPS** — GitHub only allows HTTPS callback URLs (except `localhost`). An `http://` callback will be rejected as invalid protocol.

Note down both your **Client ID** and **Client Secret** — you'll need them in the next steps.

---

### Step 2 — Create a GitHub Organization

- Create a free org at [github.com/organizations/plan](https://github.com/organizations/plan)
- Add your GitHub user to the organization
- **Set your membership visibility to Public**

![GitHub Org People — Make Membership Public](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/make-membership-public.png)

> Your membership visibility **must be Public**. If it's set to Private, ArgoCD cannot read your organization membership and RBAC group mapping will fail silently.

![GitHub Org People View](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/github-org-people.png)

---

### Step 3 — Replace Placeholders in Config Files

Before applying, update these placeholders in each file:

| Placeholder | File | Replace With |
|-------------|------|-------------|
| `<GITHUB_CLIENT_ID>` | `dex-secret.yaml`, `argocd-cm.yaml` | Your GitHub OAuth App Client ID |
| `<GITHUB_CLIENT_SECRET>` | `dex-secret.yaml`, `argocd-cm.yaml` | Your GitHub OAuth App Client Secret |
| `<GITHUB_ORG_NAME>` | `argocd-cm.yaml` | Your GitHub Organization name |
| `<USERNAME>` | `argocd-cm.yaml` | Your desired local backup username |
| `<USER_EMAIL>` | `argocd-rbac-cm.yaml` | Your GitHub account email address |
| `<GITHUB_USERNAME>` | `argocd-rbac-cm.yaml` | Your GitHub username |

---

### Step 4 — Apply All Configuration

```bash
# 1. Apply the secret (OAuth credentials)
kubectl apply -f dex-secret.yaml

# 2. Apply the ArgoCD ConfigMap (Dex config + local backup user)
kubectl apply -f argocd-cm.yaml

# 3. Apply RBAC policies
kubectl apply -f argocd-rbac-cm.yaml

# 4. Restart ArgoCD server to pick up all changes
kubectl rollout restart -n argocd deployment argocd-server
```

---

### Step 5 — Port Forward & Access UI

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443 --address=0.0.0.0 &
```

Open `https://localhost:8080` in an **Incognito** browser window. You should now see the **"Log in via GitHub"** button:

![GitHub Login Button](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/github-login-button.png)

---

### Step 6 — Authorize Your Organization

Click **Login with GitHub** → sign in with your GitHub credentials → grant access to your organization when prompted:

![GitHub Authorize Organization](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/github-authorize.png)

> **Not seeing the authorization screen?**  
> Clear browser cache and history, then revoke existing app authorizations:
>
> ![Revoke Tokens](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/revoke-tokens.png)
>
> Go to `GitHub → Settings → Applications → Authorized OAuth Apps` → Revoke access for `ArgoCD`, then try again.

---

### Step 7 — Verify Login & Permissions

After successful login, check **User Info** in the ArgoCD UI to confirm your GitHub username appears. You can also verify in server logs:

```bash
kubectl logs -n argocd deployment/argocd-server | grep -i "login successful"
```

---

### Step 8 — Test by Adding a Repo & Creating an App

Verify your `role:repo-admin` permissions work end-to-end by connecting a repository and deploying an application:

![ArgoCD Applications Synced](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/argocd-applications-sync.png)

You should be able to add a repository and create/sync an application successfully.

---

## Repository Folder Structure Reference

For reference, here is the recommended folder structure used in this project:

![Repo Folder Structure](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/repo-folder-structure.png)

---

## Disable Built-in Admin After SSO Setup

Once SSO is working, disable the static admin user for security:

```bash
kubectl patch -n argocd configmap argocd-cm \
  --patch='{"data":{"admin.enabled": "false"}}'
```

To re-enable if needed:

```bash
kubectl patch -n argocd configmap argocd-cm \
  --patch='{"data":{"admin.enabled": "true"}}'
```

> Keep your local backup user (`accounts.<USERNAME>`) as an emergency fallback in case SSO is unavailable.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| "Log in via GitHub" button not showing | ArgoCD server not restarted after config | Run `kubectl rollout restart -n argocd deployment argocd-server` |
| GitHub authorization screen missing | Browser cache / previously authorized app | Clear cache & revoke existing OAuth tokens |
| Login succeeds but RBAC denies actions | Membership visibility set to Private | Set your GitHub org membership to **Public** |
| `http://` callback rejected by GitHub | HTTP not allowed in OAuth callbacks | Use `https://` in all URLs |
| SSO login fails silently | Wrong `clientID` or `clientSecret` | Double-check `dex-secret.yaml` values match GitHub OAuth App exactly |
| `policy.default` falling back for SSO user | `scopes` not including `groups`/`email` | Ensure `scopes: '[groups, email]'` is set in `argocd-rbac-cm.yaml` |

---

## Best Practices

| Practice | Why |
|----------|-----|
| Always use HTTPS for all URLs | GitHub rejects HTTP OAuth callbacks |
| Never commit `dex-secret.yaml` with real values | Credentials in Git are a security risk |
| Keep a local backup user | Emergency access if SSO provider is unreachable |
| Map GitHub org groups to roles | Scales better than individual user mappings |
| Set `policy.default: role:readonly` | Safe fallback for any unanticipated users |
| Set `scopes: '[groups, email]'` | Required for group-based RBAC to work with SSO |
| Use Public org membership | Required for ArgoCD to read group membership |

---

## References

- [RBAC Configuration](../rbac/README.md)
- [Official ArgoCD SSO Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/)
- [ArgoCD Dex GitHub Connector](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#github)
- [Direct OIDC Integration — Okta, Auth0, Google](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/okta/)
- [Sealed Secrets for Production](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
