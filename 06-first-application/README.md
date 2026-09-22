# 06 — Create Your First Argo CD Application

## The core idea

Before we touch the UI, understand what we are about to do.

Argo CD works on a simple principle:

```
Git                         Kubernetes
────────────────            ────────────────
deployment.yaml    vs.      actual Deployment
service.yaml       vs.      actual Service

If they match  →  Synced
If they differ →  OutOfSync
```

An **Argo CD Application** is a resource that tells Argo CD:
1. Where is my configuration? (**Source** — a Git repo + path)
2. Where should it be deployed? (**Destination** — a cluster + namespace)

That is it. Everything else flows from that.

---

## What is in this folder?

This folder contains the Kubernetes manifests for our Nginx app. These are the files Argo CD will read from Git and apply to Kubernetes.

```
06-first-application/
  ├── deployment.yaml    ← tells Kubernetes: run 2 nginx pods
  ├── service.yaml       ← tells Kubernetes: give them a stable network address
  └── README.md          ← this file
```

---

## Understanding deployment.yaml

```yaml
apiVersion: apps/v1          # Use the apps/v1 API (handles Deployments)
kind: Deployment             # This resource is a Deployment
metadata:
  name: nginx-demo           # The Deployment is called nginx-demo
  namespace: default         # Put it in the default namespace
  labels:
    app: nginx-demo          # Tag this resource with app=nginx-demo
spec:
  replicas: 2                # Run 2 copies of our Pod
  selector:
    matchLabels:
      app: nginx-demo        # This Deployment manages Pods with label app=nginx-demo
  template:                  # This is the Pod template — what each Pod looks like
    metadata:
      labels:
        app: nginx-demo      # Every Pod gets this label
    spec:
      containers:
        - name: nginx        # Container name
          image: nginx:1.27  # Docker image to run
          ports:
            - containerPort: 80   # Nginx listens on port 80
          resources:
            requests:
              cpu: "50m"     # Minimum CPU (50 millicores = 5% of 1 CPU)
              memory: "64Mi" # Minimum memory
            limits:
              cpu: "100m"    # Maximum CPU
              memory: "128Mi"# Maximum memory
```

**What `replicas: 2` means:**

```
Deployment (replicas: 2)
  ├── Pod 1  (nginx container)
  └── Pod 2  (nginx container)
```

Kubernetes will always try to keep exactly 2 Pods running. If one dies, it creates a new one.

---

## Understanding service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo           # The Service is called nginx-demo
  namespace: default
spec:
  type: ClusterIP            # Only reachable inside the cluster
  selector:
    app: nginx-demo          # Route traffic to Pods with this label
  ports:
    - port: 80               # The Service listens on port 80
      targetPort: 80         # Forward traffic to port 80 on the Pod
```

**Why do we need a Service?**

Pods get random IP addresses that change every time a Pod restarts. A Service gives you a stable address that always points to healthy Pods.

```
Service: nginx-demo (stable IP: 10.96.x.x:80)
         │
         ├──► Pod 1  (10.244.0.5:80)
         └──► Pod 2  (10.244.0.6:80)
```

The `selector: app: nginx-demo` is how the Service knows which Pods to send traffic to. It matches the label on our Pods.

---

## Step 1 — Push this folder to GitHub

Argo CD needs to read these files from a **Git repository**. Argo CD cannot read files from your local filesystem.

**Option A — Use this repo directly**

If you have pushed `argocd-learning-lab` to GitHub, Argo CD can read directly from it. The path to give Argo CD would be `06-first-application`.

**Option B — Create a separate app repo**

You can also create a new GitHub repo (e.g., `nginx-demo`), copy `deployment.yaml` and `service.yaml` into the root, and push. Then Argo CD reads from path `.` (root).

Either approach works. For this lab we will use Option A (this same repo).

> Make sure your repository is public, or that you have configured a private repo credential in Argo CD Settings → Repositories.

---

## Step 2 — Create the Argo CD Application via UI

Make sure your port-forward is running:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080` and log in.

Click **+ New App** (top left).

Fill in the form:

### General section

| Field | Value |
|-------|-------|
| Application Name | `nginx-demo` |
| Project | `default` |
| Sync Policy | `Manual` |

### Source section

| Field | Value |
|-------|-------|
| Repository URL | Your GitHub repo URL (e.g. `https://github.com/your-username/argocd-learning-lab`) |
| Revision | `HEAD` |
| Path | `06-first-application` |

### Destination section

| Field | Value |
|-------|-------|
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

**What does `https://kubernetes.default.svc` mean?**

This is the internal DNS name of the Kubernetes API server — it refers to the same cluster that Argo CD is running in. This is automatically registered and available as "in-cluster".

Click **Create**.

---

## Step 3 — Alternatively, create via YAML

If you prefer YAML over the UI (the GitOps way!), apply this manifest:

```yaml
# Save this as argocd-app.yaml (do NOT commit this file — it's just for reference)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-USERNAME/argocd-learning-lab
    targetRevision: HEAD
    path: 06-first-application
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
# Replace YOUR-USERNAME first, then apply
kubectl apply -f argocd-app.yaml
```

```bash
# Verify the Application was created
kubectl get applications -n argocd
```

Expected output:

```
NAME         SYNC STATUS   HEALTH STATUS
nginx-demo   OutOfSync     Missing
```

---

## Step 4 — Understand what you are seeing

After creating the Application, Argo CD immediately checks the current state.

It finds:
- **Git says:** Deployment `nginx-demo` and Service `nginx-demo` should exist in namespace `default`
- **Kubernetes says:** Neither of those resources exist yet

Therefore:

```
Desired State (Git)     ≠     Actual State (Kubernetes)
                  ↓
              OutOfSync
```

And because the resources don't exist at all yet:

```
Health Status: Missing
```

**This is not an error.** OutOfSync just means "Git and Kubernetes don't match." We haven't deployed anything yet — that is expected.

```
Git
 │
 │  deployment.yaml  ←── "This should exist"
 │  service.yaml     ←── "This should exist"
 ▼
Argo CD
 │
 │  Checks Kubernetes...
 ▼
Kubernetes
 └── (nothing in default namespace yet)

Result: OutOfSync + Missing
```

Once synced, the application card in the UI will look like this — showing **Healthy** and **Synced**, along with the source repo, revision, path, destination cluster and namespace:

![Argo CD application card showing nginx-demo Healthy and Synced](../00-media-files/argo_application.png)

---

## Key terms (crystal clear version)

### Sync Status

**Synced** → What Git says matches what is running in Kubernetes. They are identical.

**OutOfSync** → Git and Kubernetes are different. Could mean:
- Resources exist in Git but not in Kubernetes (not deployed yet)
- Resources exist in Kubernetes but not in Git (manually created)
- Resources exist in both but with different values (e.g., replicas differ)

### Health Status

**Healthy** → Resources are running correctly.

**Missing** → Resources are defined in Git but do not exist in Kubernetes at all.

**Progressing** → Resources exist but are not ready yet (e.g., Pods are starting up).

**Degraded** → Resources exist but are not working correctly (e.g., Pod is CrashLooping).

---

## What is next?

We have created the Application. Argo CD knows what should exist, but we have not told it to actually deploy yet.

The next step is to Sync — which tells Argo CD to apply the manifests from Git to Kubernetes.

Continue in: [07 — Nginx with Argo CD](../07-nginx-with-argocd/README.md)
