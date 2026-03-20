# ArgoCD RBAC — Role-Based Access Control

Role-Based Access Control (RBAC) in ArgoCD controls **who** (users/groups) can perform **what actions** on **which resources**. This folder contains the complete local user setup and RBAC policy configuration.

---

## Folder Structure

```
rbac/
├── argocd-user-cm.yaml       # Defines local users in argocd-cm ConfigMap
└── argocd-rbac-cm.yaml       # Defines roles, policies & user-role mappings
```

---

## Concepts

ArgoCD uses **Casbin** syntax for RBAC with two types of rules:

### Policy Format
```
p, <role/user/group>, <resource>, <action>, <object>, <effect>
```

### Group Assignment Format
```
g, <user/group>, <role>
```

### Available Resources & Valid Actions

| Resource | get | create | update | delete | sync | action | override |
|----------|:---:|:------:|:------:|:------:|:----:|:------:|:--------:|
| applications | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| applicationsets | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| clusters | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| projects | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| repositories | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| accounts | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| logs | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| exec | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

### Action Reference

| Action | Meaning |
|--------|---------|
| `get` | Read / view resource |
| `create` | Create a new resource |
| `update` | Modify an existing resource |
| `delete` | Remove a resource |
| `sync` | Deploy / trigger an application sync |
| `override` | Force sync with parameter overrides |
| `action` | Trigger custom resource actions |

### Object Patterns

| Pattern | Meaning |
|---------|---------|
| `*/*` | All projects / all applications |
| `myproject/*` | All apps in a specific project |
| `*` | All (for non-application resources) |

---

## Local Users — `argocd-user-cm.yaml`

Local users are defined inside the `argocd-cm` ConfigMap. Each user is granted one or more capabilities.

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
  url: http://localhost:8080

  # =========================
  # USERS
  # =========================
  accounts.shubham: apiKey, login
  accounts.alex: login
  accounts.gulshankumar: apiKey

  # Default fallback role
  policy.default: role:readonly
```

### User Capabilities Explained

| User | `apiKey` | `login` | Purpose |
|------|:--------:|:-------:|---------|
| `shubham` | ✅ | ✅ | Full local user — UI login + API token generation |
| `alex` | ❌ | ✅ | UI login only — no API/CI-CD access |
| `gulshankumar` | ✅ | ❌ | API/CI-CD automation only — no UI login |

> **`apiKey`** — Allows generating authentication tokens for API access and CI/CD pipelines  
> **`login`** — Allows logging in through the ArgoCD Web UI

---

## RBAC Policies — `argocd-rbac-cm.yaml`

Roles, permissions, and user-role mappings are all defined in the `argocd-rbac-cm` ConfigMap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # =====================================================
    # ROLES
    # =====================================================

    # -------------------------
    # Admin → controlled full access
    # -------------------------
    p, role:admin, applications, get,    */*, allow
    p, role:admin, applications, create, */*, allow
    p, role:admin, applications, update, */*, allow
    p, role:admin, applications, delete, */*, allow
    p, role:admin, applications, sync,   */*, allow

    p, role:admin, projects,     get,    *, allow
    p, role:admin, projects,     create, *, allow
    p, role:admin, projects,     update, *, allow

    p, role:admin, repositories, get,    *, allow
    p, role:admin, repositories, create, *, allow

    p, role:admin, clusters,     get,    *, allow

    # -------------------------
    # Readonly → strict view only
    # -------------------------
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, projects,     get, *,   allow

    # (NO deny rule → avoids breaking UI)

    # -------------------------
    # Developer → limited deploy access
    # -------------------------
    p, role:developer, applications, get,    */*, allow
    p, role:developer, applications, sync,   */*, allow
    p, role:developer, applications, update, */*, allow

    p, role:developer, projects, get, *, allow

    # =====================================================
    # USER → ROLE MAPPING
    # =====================================================
    g, shubham,      role:admin
    g, alex,         role:readonly
    g, gulshankumar, role:developer

  # Default fallback — any authenticated user without a role gets read-only
  policy.default: role:readonly
```

### Role Summary

| Role | Applications | Projects | Repositories | Clusters |
|------|-------------|---------|-------------|---------|
| `role:admin` | get, create, update, **delete**, sync | get, create, update | get, create | get |
| `role:developer` | get, sync, update | get | ❌ | ❌ |
| `role:readonly` | get only | get only | ❌ | ❌ |

### User → Role Mapping

| User | Assigned Role | Can View | Can Sync | Can Delete | Can Create |
|------|--------------|:--------:|:--------:|:----------:|:----------:|
| `shubham` | `role:admin` | ✅ | ✅ | ✅ | ✅ |
| `gulshankumar` | `role:developer` | ✅ | ✅ | ❌ | ❌ |
| `alex` | `role:readonly` | ✅ | ❌ | ❌ | ❌ |

> **Default fallback:** `policy.default: role:readonly`  
> Any authenticated user with no explicit mapping automatically gets read-only access.

---

## Step-by-Step Setup

### Step 1 — Update Admin Password First

Before creating new users, update the built-in admin password:

```bash
argocd account update-password \
  --current-password <current-password> \
  --new-password <new-password>
```

> You can also update it via ArgoCD UI → **User Info**. After updating, log back in with the new password.

---

### Step 2 — Apply Local Users

```bash
kubectl apply -f argocd-user-cm.yaml
```

---

### Step 3 — Set Passwords for Each User

```bash
argocd account update-password --account shubham
argocd account update-password --account alex
argocd account update-password --account gulshankumar
```

> Password requirements: minimum **8 characters**, maximum **32 characters**

---

### Step 4 — List All Users

```bash
argocd account list
```

![Account List](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/account-list.png)

---

### Step 5 — Get Specific User Details

```bash
argocd account get --account shubham
```

---

### Step 6 — Apply RBAC Configuration

```bash
kubectl apply -f argocd-rbac-cm.yaml
```

---

### Step 7 — Validate RBAC Policy

```bash
argocd admin settings rbac validate --policy-file argocd-rbac-cm.yaml
```

---

## Verify Permissions with `rbac can`

Use `argocd admin settings rbac can` to test any user's exact permissions before going live. Format:

```bash
argocd admin settings rbac can <user> <action> <resource> "<object>" -n argocd
```

### Examples

```bash
# alex (readonly) — can GET applications?
argocd admin settings rbac can alex get applications "myproject/*" -n argocd
# → Yes

# alex (readonly) — can SYNC applications?
argocd admin settings rbac can alex sync applications "myproject/*" -n argocd
# → No
```

**RBAC sync denied — expected for readonly user:**

![RBAC Sync Denied](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/security-scaling/output-images/rbac-sync-denied.png)

```bash
# alex (readonly) — can DELETE applications?
argocd admin settings rbac can alex delete applications "myproject/*" -n argocd
# → No

# shubham (admin) — can SYNC applications?
argocd admin settings rbac can shubham sync applications "*/*" -n argocd
# → Yes

# shubham (admin) — can DELETE applications?
argocd admin settings rbac can shubham delete applications "*/*" -n argocd
# → Yes (admin role has all permissions)

# gulshankumar (developer) — can SYNC applications?
argocd admin settings rbac can gulshankumar sync applications "*/*" -n argocd
# → Yes

# gulshankumar (developer) — can DELETE applications?
argocd admin settings rbac can gulshankumar delete applications "*/*" -n argocd
# → No (developer role has no delete permission)
```

---

## Disable / Enable Admin User

Once users and RBAC are configured, disable the built-in admin for security:

```bash
# Disable admin user
kubectl patch -n argocd configmap argocd-cm \
  --patch='{"data":{"admin.enabled": "false"}}'

# Re-enable admin user (if needed)
kubectl patch -n argocd configmap argocd-cm \
  --patch='{"data":{"admin.enabled": "true"}}'
```

---

## Verify via UI

Log in with each user in an **Incognito** browser window to confirm permissions visually:

| User | What you can see & do |
|------|-----------------------|
| `alex` (readonly) | Can only view applications. Cannot add repos, create apps, or sync. |
| `gulshankumar` (developer) | Can view and sync applications. Cannot delete or create apps. |
| `shubham` (admin) | Full access — manages all applications, projects, and repositories. |

---

## Best Practices

| Practice | Why |
|----------|-----|
| Set `policy.default: role:readonly` | Safe fallback — unknown users can only view |
| Grant minimum required permissions | Least-privilege principle reduces blast radius |
| Use groups instead of individual users | Easier to manage at scale, especially with SSO |
| Disable the built-in `admin` user after setup | Eliminates a high-privilege static credential |
| Validate RBAC before applying | Catch policy mistakes before they affect production |
| Use `rbac can` for permission testing | Verify exact permissions without live trial-and-error |
| Avoid explicit `deny` rules on applications | Can break the ArgoCD UI unexpectedly |

---

## References

- [SSO Configuration with GitHub](../sso/README.md)
- [Official ArgoCD RBAC Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
- [Casbin Policy Syntax](https://casbin.org/docs/syntax-for-models)
