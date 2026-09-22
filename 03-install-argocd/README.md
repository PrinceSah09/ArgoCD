# 03 — Install Argo CD

## What are we doing?

Installing Argo CD inside our Kind Kubernetes cluster. Argo CD itself runs as a set of Pods in a dedicated namespace called `argocd`.

```
Kind Cluster
  └── argocd namespace
        ├── argocd-server
        ├── argocd-application-controller
        ├── argocd-repo-server
        ├── argocd-applicationset-controller
        ├── argocd-redis
        ├── argocd-dex-server
        └── argocd-notifications-controller
```

---

## Step 1 — Create the argocd namespace

```bash
kubectl create namespace argocd
```

**What this does:** Creates a logical folder inside Kubernetes called `argocd`. All Argo CD components will live here, separated from your applications.

**Why a separate namespace?**
It keeps things organized. Argo CD's own Pods, Secrets, and Services won't mix with your application resources.

**Expected output:**

```
namespace/argocd created
```

Verify it exists:

```bash
kubectl get namespaces
```

You should now see `argocd` in the list.

---

## Step 2 — Install Argo CD

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Breaking down this command:**

| Part | Meaning |
|------|---------|
| `kubectl apply` | Create or update resources defined in a YAML file |
| `-n argocd` | Apply everything into the `argocd` namespace |
| `-f <url>` | The YAML file to apply — fetched directly from GitHub |

**What is in that YAML file?**

It is the official Argo CD install manifest. It contains all the Deployments, Services, ConfigMaps, RBAC roles, and CRDs (Custom Resource Definitions) needed to run Argo CD.

**Expected output:**

```
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
...
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
statefulset.apps/argocd-application-controller created
...
```

You will see many lines — that is normal.

---

## Step 3 — Wait for Argo CD to be ready

```bash
kubectl get pods -n argocd
```

Run this every 20–30 seconds until all pods show `Running`.

**Expected output (when ready):**

```
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          2m
argocd-applicationset-controller-xxx                1/1     Running   0          2m
argocd-dex-server-xxx                               1/1     Running   0          2m
argocd-notifications-controller-xxx                 1/1     Running   0          2m
argocd-redis-xxx                                    1/1     Running   0          2m
argocd-repo-server-xxx                              1/1     Running   0          2m
argocd-server-xxx                                   1/1     Running   0          2m
```

**Column meanings:**

| Column | Meaning |
|--------|---------|
| `READY` | `1/1` means the container inside the pod is ready |
| `STATUS` | `Running` is what we want |
| `RESTARTS` | How many times the container restarted — 0 is healthy |

If you want to watch continuously instead of running the command repeatedly:

```bash
kubectl get pods -n argocd --watch
```

Press `Ctrl+C` to stop watching once all pods are `Running`.

---

## Step 4 — Verify everything is up

```bash
kubectl get all -n argocd
```

This shows every resource Argo CD created: Pods, Services, Deployments, StatefulSets, and ReplicaSets.

You should see 7 services, including `argocd-server`.

---

## What just happened?

```
Your terminal
     │
     │ kubectl apply -f install.yaml
     ▼
Kubernetes API
     │
     │ Creates all resources defined in the YAML
     ▼
argocd namespace
     │
     ├── Deployments created
     │     └── Kubernetes starts creating Pods
     │
     └── Pods become Running
           └── Argo CD is alive
```

Argo CD is now running inside your cluster. It is watching for `Application` resources and waiting for you to tell it what to deploy.

---

## Troubleshooting

**Pods stuck in `Pending`**
```bash
kubectl describe pod <pod-name> -n argocd
```
Look at the `Events` section at the bottom. Usually this means not enough resources — try restarting Docker Desktop.

**Pods in `CrashLoopBackOff`**
```bash
kubectl logs <pod-name> -n argocd
```
Read the logs to find the error.

**Image pull errors (`ErrImagePull` or `ImagePullBackOff`)**
→ Your Docker Desktop may not have internet access. Check Docker Desktop network settings.

**`kubectl apply` fails with connection refused**
→ Your cluster isn't running. Run `kind get clusters` and recreate if needed.

---

## Quick reference

```bash
# Check all argocd pods
kubectl get pods -n argocd

# Check all argocd services
kubectl get svc -n argocd

# See logs from argocd-server
kubectl logs deployment/argocd-server -n argocd

# Describe a specific pod (for troubleshooting)
kubectl describe pod <pod-name> -n argocd
```

---

Next step: [04 — Argo CD Components](../04-argocd-components/README.md)
