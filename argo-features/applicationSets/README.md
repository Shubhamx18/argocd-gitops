# ApplicationSets in ArgoCD

ApplicationSets are a way to **dynamically generate multiple ArgoCD Applications** from a single manifest.

Instead of writing many `Application` YAMLs manually, you define a **template + a generator**, and ArgoCD automatically creates all the apps for you.

---

## Theory

Normally, **each app = one `Application` CRD**.
With ApplicationSets, you write **one manifest** and let the generator create all the apps.

Think of it like this:

```
Without ApplicationSet:
  nginx-app.yml        → apply → nginx Application
  netflix-app.yml      → apply → netflix Application
  portfolio-app.yml    → apply → portfolio Application

With ApplicationSet:
  appset.yml           → apply → nginx + netflix + portfolio Applications (auto!)
```

**Generators** decide HOW apps are created:

| Generator | What it does |
|-----------|-------------|
| **List Generator** | You define a static list of apps manually |
| **Cluster Generator** | Deploys the same app across all registered clusters |
| **Git Generator** | Scans a Git repo and creates one app per folder |

### Why use ApplicationSets?

- **Less YAML** — one manifest instead of many
- **Multi-cluster & multi-env** deployments become trivial
- **True DRY** (Don't Repeat Yourself) in GitOps
- **Auto-creates apps** when new folders/clusters are added

---

## Repo Structure

Your repo should look like this before applying any ApplicationSet:

```
argocd-gitops/
├── practicals/
│   ├── netflix-manifests/        ← netflix app manifests
│   │   ├── deployment.yml
│   │   └── service.yml
│   ├── nginx-manifests/          ← nginx app manifests
│   │   ├── deployment.yml
│   │   └── service.yml
│   └── portfolio-manifest/       ← portfolio app manifests
│       ├── deployment.yml
│       └── service.yml
├── list_generator_appset.yml
├── cluster_generator.yml
└── git_generator.yml
```

---

## Generator YAMLs

### 1. List Generator — `list_generator_appset.yml`

Use: `list_generator_appset.yml`

Replace `<your-username>` with your GitHub username.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: app-sets
  namespace: argocd
spec:
  # The list generator lets you enumerate a static set of elements (apps)
  generators:
    - list:
        elements:
          - app: netflix
            path: practicals/netflix-manifests    # path in repo for this app
          - app: nginx
            path: practicals/nginx-manifests      # path in repo for this app
          - app: portfolio
            path: practicals/portfolio-manifest   # path in repo for this app
  # Template that will be used to generate individual Application CRs
  template:
    metadata:
      # name template uses the element key 'app'
      name: '{{app}}-list'
    spec:
      project: default
      source:
        repoURL: https://github.com/<your-username>/argocd-gitops.git
        targetRevision: main
        path: '{{path}}'          # resolved from element
      destination:
        server: https://kubernetes.default.svc
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

![List Generator YAML in GitHub](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/list_generator_yaml.png)

This will create **three apps**:
- `netflix-list` → Netflix/StreamFlix app
- `nginx-list` → Nginx web server
- `portfolio-list` → Personal portfolio website

---

### 2. Git Generator — `git_generator.yml`

Use: `git_generator.yml`

Replace `<your-username>` with your GitHub username.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo-git
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/<your-username>/argocd-gitops.git
        revision: main
        directories:
          - path: git_generator/*    # scan all folders inside git_generator/
  template:
    metadata:
      name: '{{path.basename}}-git'
    spec:
      project: default
      source:
        repoURL: https://github.com/<your-username>/argocd-gitops.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

This will **scan repo directories** and auto-create apps for each folder found (e.g., `apache-git`, `online-shop-git`, `chai-app-git`).

---

### 3. Cluster Generator — `cluster_generator.yml`

Use: `cluster_generator.yml`

Replace `<your-username>` with your GitHub username.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo-cluster
  namespace: argocd
spec:
  generators:
    - clusters: {}    # targets ALL clusters registered in ArgoCD
  template:
    metadata:
      name: '{{name}}-chai-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/<your-username>/argocd-gitops.git
        targetRevision: main
        path: applicationsets/chai-app
      destination:
        server: '{{server}}'     # resolved per cluster
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

This will deploy **chai-app into ALL clusters** registered in ArgoCD.

---

## Implementation Steps

### List Generator

#### Prerequisites

- `kind` cluster running (where ArgoCD is deployed)
- ArgoCD installed and running
- `argocd` CLI installed and logged in
- `kubectl` CLI installed and configured

#### Step 1 — Apply the ApplicationSet manifest

```bash
kubectl apply -f list_generator_appset.yml -n argocd
```

#### Step 2 — Check ApplicationSet status

```bash
argocd appset list
```

You will get something like:

```
NAME             PROJECT  SYNCPOLICY  CONDITIONS                                                                         REPO                                               PATH      TARGET
argocd/app-sets  default  nil         [{ParametersGenerated Successfully generated parameters ... True}                  https://github.com/shubhamx18/argocd-gitops.git   {{path}}  main
                                       {ResourcesUpToDate ApplicationSet up to date ... True ApplicationSetUpToDate}]
```

#### Step 3 — Verify in ArgoCD UI

Apps will appear automatically — `netflix-list`, `nginx-list`, `portfolio-list`:

![ArgoCD UI showing all 3 apps - Healthy and Synced](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/argocd_apps_list.png)

---

#### Step 4 — Explore each app

**netflix-list** → Deploys the StreamFlix (Netflix Clone) app

ArgoCD resource tree — Service → Deployment → ReplicaSet → Pods:

![netflix-list resource tree in ArgoCD](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/netflix_list_tree.png)

Access the app:

```bash
kubectl port-forward svc/netflix-service 8080:80 --address=0.0.0.0 &
```

> Open inbound rule for port `8080`, then open browser → [http://localhost:8080](http://localhost:8080)

You should see the **StreamFlix homepage**:

![StreamFlix app running in browser](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/streamflix_app.png)

---

**nginx-list** → Deploys the Nginx web server

ArgoCD resource tree — Service → Deployment → ReplicaSet → 4 Pods running:

![nginx-list resource tree in ArgoCD](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/nginx_list_tree.png)

Access the app:

```bash
kubectl port-forward svc/nginx-service 8081:80 --address=0.0.0.0 &
```

> Open inbound rule for port `8081`, then open browser → [http://localhost:8081](http://localhost:8081)

You should see the **Nginx Welcome page**:

![Nginx welcome page](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/nginx_welcome.png)

---

**portfolio-list** → Deploys the personal portfolio website

ArgoCD resource tree — Service → Deployment → ReplicaSet → Pod:

![portfolio-list resource tree in ArgoCD](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/portfolio_list_tree.png)

Access the app:

```bash
kubectl port-forward svc/portfolio-service 8082:80 --address=0.0.0.0 &
```

> Open inbound rule for port `8082`, then open browser → [http://localhost:8082](http://localhost:8082)

You should see **Shubham's Portfolio website**:

![Portfolio website running in browser](https://raw.githubusercontent.com/Shubhamx18/argocd-gitops/main/argo-features/applicationSets/output-images/portfolio_app.png)

---

#### Step 5 — Verify all apps via CLI

```bash
argocd app list
```

You will get:

```
NAME                      CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                               PATH                          TARGET
argocd/netflix-list       https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>      https://github.com/shubhamx18/argocd-gitops.git   practicals/netflix-manifests   main
argocd/nginx-list         https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>      https://github.com/shubhamx18/argocd-gitops.git   practicals/nginx-manifests     main
argocd/portfolio-list     https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>      https://github.com/shubhamx18/argocd-gitops.git   practicals/portfolio-manifest  main
```

#### Step 6 — Delete ApplicationSet

```bash
argocd appset delete argocd/app-sets
```

> **Note:** Deleting an ApplicationSet will also delete all the Applications it generated.

---

### Cluster Generator

#### Prerequisites

- `kind` cluster running (where ArgoCD is deployed)
- ArgoCD installed and running
- `argocd` CLI installed and logged in
- `kubectl` CLI installed and configured
- **At least 2 clusters registered in ArgoCD** (e.g., in-cluster + kind-argocd-cluster)

#### Step 1 — Get cluster contexts

```bash
kubectl config get-contexts
```

#### Step 2 — Add your second cluster to ArgoCD

```bash
argocd cluster add kind-argocd-cluster
```

> You can add any other cluster too. This cluster will now appear in ArgoCD under **Settings → Clusters**.

#### Step 3 — Apply the ApplicationSet manifest

```bash
kubectl apply -f cluster_generator.yml -n argocd
```

#### Step 4 — Check ApplicationSet status

```bash
argocd appset list
```

#### Step 5 — Verify all apps via CLI

```bash
argocd app list
```

You will get:

```
NAME                                 CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS
argocd/in-cluster-chai-app           https://kubernetes.default.svc  default    default  OutOfSync  Healthy  Auto-Prune  SharedResourceWarning(3)
argocd/kind-argocd-cluster-chai-app  https://172.31.19.178:33893     default    default  Synced     Healthy  Auto-Prune  <none>
```

> **Note:** The `in-cluster` app shows `OutOfSync` because `chai-app` is also deployed to `kind-argocd-cluster` using the same resources — this is just a `SharedResourceWarning`. You can safely ignore it.

#### Step 6 — Access the app from kind-argocd-cluster

```bash
kubectl port-forward --context kind-argocd-cluster svc/chai-app-service 3002:3000 --address=0.0.0.0 &
```

> Open inbound rule for port `3002`, then open browser → [http://localhost:3002](http://localhost:3002)

#### Step 7 — Delete ApplicationSet

```bash
argocd appset delete argocd/demo-cluster
```

---

### Git Generator

#### Prerequisites

- `kind` cluster running (where ArgoCD is deployed)
- ArgoCD installed and running
- `argocd` CLI installed and logged in
- `kubectl` CLI installed and configured
- A GitHub repo with **multiple app folders** inside `git_generator/` directory

Your repo structure should look like:

```
argocd-gitops/
└── git_generator/
    ├── apache/
    │   ├── deployment.yml
    │   └── service.yml
    ├── online-shop/
    │   ├── deployment.yml
    │   └── service.yml
    └── chai-app/
        ├── deployment.yml
        └── service.yml
```

#### Step 1 — Apply the ApplicationSet manifest

```bash
kubectl apply -f git_generator.yml -n argocd
```

#### Step 2 — Check ApplicationSet status

```bash
argocd appset list
```

#### Step 3 — Verify all apps via CLI

```bash
argocd app list
```

You will get:

```
NAME                    CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  REPO                                               PATH                       TARGET
argocd/apache-git       https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  https://github.com/shubhamx18/argocd-gitops.git   git_generator/apache       main
argocd/chai-app-git     https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  https://github.com/shubhamx18/argocd-gitops.git   git_generator/chai-app     main
argocd/online-shop-git  https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  https://github.com/shubhamx18/argocd-gitops.git   git_generator/online-shop  main
```

> Notice how ArgoCD **automatically discovered** all 3 app folders and created apps — you never had to write individual Application YAMLs!

#### Step 4 — Access the apps

Get services first:

```bash
kubectl get svc -n default
```

Then port-forward as needed and open in your browser.

#### Step 5 — Delete ApplicationSet

```bash
argocd appset delete argocd/demo-git
```

---

## Comparison of Generators

| Generator Type | How it Works | Best Use Case | Example Outcome |
|----------------|-------------|---------------|-----------------|
| **List Generator** | You define a static list of apps (name + path) | Simple multi-app deployments where the set is known in advance | `netflix-list`, `nginx-list`, `portfolio-list` from one manifest |
| **Cluster Generator** | Iterates over all clusters registered in ArgoCD (`argocd cluster add`) | Deploying the same app across multiple clusters (Dev, Stg, Prod) | Chai-App automatically deployed to all clusters |
| **Git Generator** | Scans Git repo for folders or manifests, creates one app per folder | Microservices or monorepos where each folder = one app | Apps auto-created for `apache`, `online-shop`, `chai-app` etc. |

---

## Key Takeaways

- **ApplicationSet = one manifest → many apps**
- **Generators** control how apps are created (List, Cluster, Git)
- Adding a new folder to your repo (Git Generator) **auto-creates** a new ArgoCD app — zero manual YAML
- Adding a new cluster to ArgoCD (Cluster Generator) **auto-deploys** the app there — zero manual YAML
- Perfect for **multi-cluster + multi-env** deployments at scale
- Reduces YAML duplication and keeps your GitOps workflow truly DRY

---

> For more info read: [ApplicationSet Official Docs](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)

**Happy Learning! **
