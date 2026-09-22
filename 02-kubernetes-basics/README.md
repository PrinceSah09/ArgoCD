# 02 — Kubernetes Basics

This is not a full Kubernetes course. We cover only the concepts you need to follow the Argo CD labs.

If something here feels new, that is fine — just keep moving. It will make more sense once you see it in action.

---

## The big picture

```
Kubernetes Cluster
  │
  ├── Nodes            (the machines that run things)
  │
  └── Namespaces       (logical groups, like folders)
        │
        └── Workloads
              ├── Deployments    (describe what to run)
              ├── Pods           (the actual running containers)
              └── Services       (network access to pods)
```

---

## Cluster

A **cluster** is a set of machines (nodes) managed by Kubernetes as a single unit.

In our case, it is one Docker container acting as the entire cluster.

```bash
kubectl cluster-info
```

Expected output:

```
Kubernetes control plane is running at https://127.0.0.1:XXXXX
```

---

## Node

A **node** is a machine (real or virtual) that runs your workloads. Kubernetes schedules pods onto nodes.

```bash
kubectl get nodes
```

Expected output:

```
NAME                       STATUS   ROLES           AGE   VERSION
argocd-lab-control-plane   Ready    control-plane   5m    v1.30.x
```

We only have one node. In production you would have many.

---

## Namespace

A **namespace** is a way to divide a cluster into logical sections. Think of it like a folder inside the cluster.

Different teams, applications, or environments often get their own namespace.

```bash
# List all namespaces
kubectl get namespaces
```

Expected output:

```
NAME              STATUS   AGE
default           Active   5m
kube-node-lease   Active   5m
kube-public       Active   5m
kube-system       Active   5m
```

| Namespace | Purpose |
|-----------|---------|
| `default` | Where things go if you don't specify a namespace |
| `kube-system` | Kubernetes internal components |
| `kube-public` | Publicly readable data (rarely used) |
| `kube-node-lease` | Used for node heartbeats |

We will create an `argocd` namespace in the next step.

```bash
# See everything running across all namespaces
kubectl get pods -A
```

The `-A` flag means "all namespaces". Right now you will mostly see `kube-system` pods.

---

## Pod

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers.

When you deploy Nginx, Kubernetes runs it inside a Pod.

```
Pod
 └── Container (nginx:1.27)
```

Pods are **ephemeral** — they can be killed and restarted at any time. You should never rely on a specific pod being alive forever. That is why we use Deployments.

```bash
# List pods in the default namespace
kubectl get pods

# List pods in a specific namespace
kubectl get pods -n kube-system

# Get more detail about a specific pod
kubectl describe pod <pod-name>
```

---

## Deployment

A **Deployment** tells Kubernetes:

> "I want 2 replicas of this container always running."

Kubernetes then creates Pods to match that desired state. If a Pod dies, Kubernetes creates a new one.

```
Deployment (replicas: 2)
  ├── Pod 1 (nginx)
  └── Pod 2 (nginx)
```

```bash
kubectl get deployments
kubectl get deployments -n argocd
kubectl describe deployment <name>
```

This is the self-healing you have heard about in Kubernetes — but it is Kubernetes, not Argo CD, that does this.

---

## Service

Pods have their own IP addresses, but those IPs change every time a Pod restarts. A **Service** gives you a stable endpoint to reach a group of Pods.

```
Service (stable IP / DNS name)
  └── Routes traffic to Pods matching a label selector
        ├── Pod 1
        └── Pod 2
```

**Service types you need to know now:**

| Type | What it does |
|------|-------------|
| `ClusterIP` | Only reachable inside the cluster (default) |
| `NodePort` | Exposes a port on every node |
| `LoadBalancer` | Creates a cloud load balancer (AWS ELB etc.) |

For local labs we use `ClusterIP` and access it through `kubectl port-forward`.

```bash
kubectl get services
kubectl get svc          # svc is short for services
kubectl get svc -n argocd
```

---

## The YAML pattern

Every Kubernetes resource is defined as YAML. The structure is always the same:

```yaml
apiVersion: apps/v1       # Which API version handles this resource
kind: Deployment          # What type of resource this is
metadata:
  name: my-app            # Name of the resource
  namespace: default      # Which namespace it lives in
spec:                     # The desired state — what you want
  replicas: 2
  ...
```

You write YAML → Kubernetes makes it real. That is the declarative model.

---

## Commands cheatsheet for this lab

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Namespaces
kubectl get namespaces
kubectl get ns               # short form

# Pods
kubectl get pods
kubectl get pods -n <namespace>
kubectl get pods -A          # all namespaces
kubectl describe pod <name> -n <namespace>
kubectl logs <pod-name> -n <namespace>

# Deployments
kubectl get deployments
kubectl get deploy           # short form

# Services
kubectl get services
kubectl get svc              # short form

# Everything in a namespace
kubectl get all -n <namespace>
```

---

## What you need to remember

| Concept | One sentence |
|---------|-------------|
| Cluster | A group of machines managed by Kubernetes |
| Node | A machine in the cluster |
| Namespace | A logical folder inside the cluster |
| Pod | A running container (or group of containers) |
| Deployment | Describes how many Pods to keep running |
| Service | Stable network access to a group of Pods |

---

Next step: [03 — Install Argo CD](../03-install-argocd/README.md)
