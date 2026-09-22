# 01 — Create a Kind Cluster

## What is Kind?

Kind stands for **Kubernetes IN Docker**.

Instead of needing a real server or a cloud provider, Kind runs a full Kubernetes cluster inside Docker containers on your laptop. Each "node" in the cluster is just a Docker container.

```
Your Mac
  └── Docker
        └── Kind
              └── Kubernetes Cluster
                    └── Nodes (Docker containers)
```

It is the fastest way to get a real Kubernetes cluster running locally for learning and testing.

---

## Why are we using Kind?

- Free — no cloud costs
- Fast to create and destroy
- Runs on Apple Silicon (ARM64) natively
- Feels like a real Kubernetes cluster
- Easy to reset if something breaks

---

## Prerequisites

Make sure Docker Desktop is running before you start. Kind needs Docker.

```bash
# Check Docker is running
docker info
```

If this command returns an error, open Docker Desktop and wait for it to start.

---

## Install Kind

```bash
brew install kind
```

Verify the installation:

```bash
kind version
```

Expected output:

```
kind v0.23.0 go1.21.x darwin/arm64
```

The exact version numbers may differ — that is fine.

---

## Create the cluster

```bash
kind create cluster --name argocd-lab
```

**What this does:**
- Pulls the Kind node image (a Docker image that contains Kubernetes)
- Creates a Docker container called `argocd-lab-control-plane`
- Bootstraps a full single-node Kubernetes cluster inside that container
- Configures `kubectl` to point at this new cluster automatically

**Expected output:**

```
Creating cluster "argocd-lab" ...
 ✓ Ensuring node image (kindest/node:v1.30.x) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-argocd-lab"
You can now use your cluster with:

kubectl cluster-info --context kind-argocd-lab
```

This takes about 1–2 minutes the first time (Docker needs to pull the image).

---

## Verify the cluster

```bash
kubectl get nodes
```

**What this does:** Lists all the nodes (machines) in your Kubernetes cluster.

**Expected output:**

```
NAME                       STATUS   ROLES           AGE   VERSION
argocd-lab-control-plane   Ready    control-plane   60s   v1.30.x
```

Let's understand each column:

| Column | Meaning |
|--------|---------|
| `NAME` | The name of the node — here it's our Docker container |
| `STATUS` | `Ready` means the node is healthy and accepting workloads |
| `ROLES` | `control-plane` means this node manages the cluster |
| `AGE` | How long ago the node was created |
| `VERSION` | The Kubernetes version running on this node |

---

## What is a control-plane node?

In Kubernetes, the **control-plane** is the brain of the cluster. It:

- Stores the cluster state (in `etcd`)
- Schedules workloads onto nodes
- Watches for changes and responds

In a production cluster you would have separate worker nodes where your applications run. For our local lab, one node doing everything is perfectly fine.

---

## What is kubectl?

`kubectl` is the command-line tool for talking to Kubernetes. Every command you run against your cluster goes through `kubectl`.

```
You
 └── kubectl
       └── Kubernetes API (control-plane)
             └── Cluster
```

When Kind created the cluster, it automatically updated your `~/.kube/config` file so `kubectl` knows which cluster to talk to.

Confirm which cluster kubectl is currently pointing at:

```bash
kubectl config current-context
```

Expected output:

```
kind-argocd-lab
```

---

## Useful Kind commands

```bash
# List all clusters you have created
kind get clusters

# Delete the cluster when you are done
kind delete cluster --name argocd-lab

# Create the cluster again fresh
kind create cluster --name argocd-lab
```

---

## Troubleshooting

**`kind: command not found`**
→ Run `brew install kind` and try again.

**Docker errors when creating cluster**
→ Make sure Docker Desktop is running. Check with `docker info`.

**`kubectl get nodes` shows `NotReady`**
→ Wait 30 seconds and try again. The node takes a moment to fully initialize.

**Wrong context — kubectl pointing at a different cluster**
→ Run `kubectl config use-context kind-argocd-lab` to switch.

---

## Summary

You now have a real Kubernetes cluster running on your Mac inside Docker.

```
Mac
 └── Docker Desktop
       └── Container: argocd-lab-control-plane
             └── Kubernetes cluster (1 node)
```

Next step: [02 — Kubernetes Basics](../02-kubernetes-basics/README.md)
