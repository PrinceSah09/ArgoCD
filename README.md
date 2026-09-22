# Argo CD Learning Lab

A practical, step-by-step Argo CD learning repository.

I document exactly what I do in my local Kind cluster — every command, every concept, every experiment.

**Stack:** Kind · Kubernetes · Argo CD · GitHub · Nginx · macOS (Apple Silicon)

---

## What I am learning

How to use Argo CD to deploy applications to Kubernetes using GitOps — where Git is the single source of truth and Argo CD keeps the cluster in sync automatically.

---

## Learning Path

| Step | Topic | Folder |
|------|-------|--------|
| 01 | Create a Kind cluster | [01-kind](./01-kind/README.md) |
| 02 | Understand basic Kubernetes | [02-kubernetes-basics](./02-kubernetes-basics/README.md) |
| 03 | Install Argo CD | [03-install-argocd](./03-install-argocd/README.md) |
| 04 | Understand Argo CD components | [04-argocd-components](./04-argocd-components/README.md) |
| 05 | Access the Argo CD UI | [05-argocd-ui](./05-argocd-ui/README.md) |
| 06 | Create first Argo CD Application | [06-first-application](./06-first-application/README.md) |
| 07 | Deploy Nginx with Argo CD | [07-nginx-with-argocd](./07-nginx-with-argocd/README.md) |
| 08 | Run experiments | [08-experiments](./08-experiments/README.md) |

---

## The core idea (before you read anything else)

```
You push YAML to Git
        ↓
Argo CD reads Git
        ↓
Argo CD applies it to Kubernetes
        ↓
Your app is running
```

If someone manually changes something in the cluster, Argo CD notices.
If your Git changes, Argo CD notices.
That is GitOps.

---

## Prerequisites

- macOS with Apple Silicon
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- [Kind](https://kind.sigs.k8s.io/) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) installed
- A GitHub account
- Basic Kubernetes knowledge (Pods, Deployments, Services)

Install prerequisites quickly:

```bash
# Kind
brew install kind

# kubectl
brew install kubectl

# Verify
kind version
kubectl version --client
```

---

## Quick start

```bash
# 1. Create the cluster
kind create cluster --name argocd-lab

# 2. Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for pods to be ready
kubectl get pods -n argocd --watch

# 4. Get the admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# 5. Open the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then visit: https://localhost:8080
```

Then follow the steps in order starting from [01-kind](./01-kind/README.md).

---

## Key concepts you will understand by the end

- **GitOps** — Git as the source of truth
- **Desired state vs actual state** — what Git says vs what's running
- **Reconciliation** — Argo CD continuously comparing the two
- **Application** — the core Argo CD resource
- **Sync / OutOfSync** — whether desired == actual
- **Healthy / Missing / Degraded** — health of the actual resources
- **Manual sync** — you tell Argo CD to apply changes
- **Self-healing** — Argo CD reverts manual changes (introduced later)

---

## Versions used

| Tool | Version |
|------|---------|
| Kind | v0.23+ |
| Kubernetes | v1.30+ |
| Argo CD | stable (v2.12+) |
| Nginx | 1.27 |

---

> This is a personal learning lab. I build it step by step, breaking things intentionally to understand what goes wrong and why.
