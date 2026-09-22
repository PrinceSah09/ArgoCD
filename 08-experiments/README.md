# 08 — Experiments

Now that nginx is deployed and Argo CD is managing it, we run small experiments to understand exactly how GitOps and Argo CD behave.

Each experiment follows the same pattern:

```
Change something
    ↓
Observe what happens
    ↓
Understand why
    ↓
Restore state
```

---

## Experiment 1 — Change replicas in Git

**What we are testing:** When you change a value in Git, Argo CD detects it and the app goes OutOfSync.

### Setup

Current state:

```yaml
# deployment.yaml
replicas: 2
```

Kubernetes is running 2 Pods. Argo CD shows Synced.

### Action

Edit `06-first-application/deployment.yaml` on your local machine:

```yaml
replicas: 3   # changed from 2
```

Commit and push to GitHub:

```bash
git add 06-first-application/deployment.yaml
git commit -m "scale nginx to 3 replicas"
git push
```

### Observe

Argo CD polls Git every **3 minutes** by default. Wait up to 3 minutes, or force a refresh immediately:

```bash
# Force Argo CD to re-check Git right now
argocd app get nginx-demo --refresh
```

Or in the UI: click the **Refresh** button on the app.

Now check:

```bash
kubectl get applications -n argocd
```

Expected output:

```
NAME         SYNC STATUS   HEALTH STATUS
nginx-demo   OutOfSync     Healthy
```

**Why OutOfSync?**

```
Git says:        replicas: 3
Kubernetes has:  replicas: 2

They are different → OutOfSync
```

Notice the Health is still **Healthy** — the Deployment is still working fine with 2 replicas. OutOfSync does not mean broken.

### Sync it

In the UI click **SYNC → SYNCHRONIZE**, or:

```bash
argocd app sync nginx-demo
```

Then verify:

```bash
kubectl get pods
```

Expected: 3 Pods now running.

```bash
kubectl get applications -n argocd
```

Expected:

```
NAME         SYNC STATUS   HEALTH STATUS
nginx-demo   Synced        Healthy
```

### What you learned

- Changing Git → Argo CD detects the difference → OutOfSync
- OutOfSync is not an error — it means "pending change"
- Manual Sync applies the change from Git to Kubernetes
- Argo CD does not automatically apply changes unless auto-sync is configured

---

## Experiment 2 — Delete a Pod manually

**What we are testing:** Kubernetes self-healing. This is Kubernetes behavior, not Argo CD.

### Action

Get a Pod name:

```bash
kubectl get pods
```

Delete one Pod:

```bash
kubectl delete pod <pod-name>
```

### Observe immediately

```bash
kubectl get pods
```

You will see one Pod `Terminating` and a new one `ContainerCreating` almost instantly.

Wait a few seconds:

```bash
kubectl get pods
```

Expected: 3 Pods are Running again.

**Why?**

The Deployment said `replicas: 3`. When you deleted a Pod, Kubernetes noticed the actual count (2) was less than the desired count (3). It immediately created a new Pod.

```
Desired: 3 Pods (from Deployment spec)
Actual:  2 Pods (after manual delete)
         ↓
Kubernetes creates Pod 3
         ↓
Actual: 3 Pods again
```

**This is Kubernetes doing this, not Argo CD.** Kubernetes continuously reconciles Deployments. Argo CD operates at a higher level — it reconciles the Deployment resource itself, not individual Pods.

### Check Argo CD

```bash
kubectl get applications -n argocd
```

Argo CD still shows **Synced + Healthy**. It does not care that a Pod was briefly deleted — the Deployment resource itself never changed.

### What you learned

- Kubernetes self-heals Pods automatically (Deployment controller)
- Argo CD self-heals at the resource level (Deployment, Service, etc.)
- These are different layers of reconciliation

---

## Experiment 3 — Manually scale Kubernetes (bypass Git)

**What we are testing:** What happens when someone changes Kubernetes directly, without going through Git. This is called **drift** — the actual state drifts away from the desired state in Git.

### Action

Scale the Deployment directly using kubectl, bypassing Git entirely:

```bash
kubectl scale deployment nginx-demo --replicas=1
```

### Observe

```bash
kubectl get pods
```

Only 1 Pod now running.

Check Argo CD immediately:

```bash
kubectl get applications -n argocd
```

Expected (within 3 minutes):

```
NAME         SYNC STATUS   HEALTH STATUS
nginx-demo   OutOfSync     Healthy
```

**Why OutOfSync?**

```
Git says:        replicas: 3   (your last commit)
Kubernetes has:  replicas: 1   (you just changed it manually)

They are different → OutOfSync
```

This is **drift**. Someone (you, a colleague, a script) changed the cluster without going through Git. Argo CD detected it.

In the UI, you will also see a yellow **OutOfSync** indicator on the Deployment resource in the resource tree. You can click the Deployment to see the diff between what Git says and what Kubernetes has.

### Options from here

**Option A — Sync (apply Git back to Kubernetes)**
```bash
argocd app sync nginx-demo
```
This restores Git state (replicas: 3) to Kubernetes. The manual change is overwritten.

**Option B — Do nothing**
Argo CD will keep showing OutOfSync but will not change anything automatically (sync policy is Manual).

For now, choose Option A and sync to restore the correct state.

### What you learned

- Manually changing Kubernetes → Argo CD detects drift → OutOfSync
- In GitOps, Git is the source of truth
- Manual `kubectl` changes should generally not be used in a GitOps workflow
- When you do sync, Git wins — the manual change is gone

---

## Experiment 4 — Remove a resource from Git

**What we are testing:** What happens when you delete a manifest from Git. Does Argo CD delete it from Kubernetes too?

### Action

Let's simulate this by creating a simple ConfigMap to observe with:

```bash
# Create a configmap directly in Kubernetes first
kubectl create configmap test-config --from-literal=env=test
```

Verify it exists:

```bash
kubectl get configmap test-config
```

Now, **without** prune enabled (default for manual sync), check what Argo CD does.

In this case, Argo CD did not create this ConfigMap — you created it manually. Let's instead test what happens with Git-managed resources.

### What to actually test

Edit `06-first-application/service.yaml`. Rename it temporarily to `service.yaml.disabled` in Git (or delete it entirely from the commit):

```bash
# Rename the file temporarily to simulate deletion
git mv 06-first-application/service.yaml 06-first-application/service.yaml.bak
git commit -m "test: remove service from git"
git push
```

Force a refresh:

```bash
argocd app get nginx-demo --refresh
```

### Observe

```bash
kubectl get applications -n argocd
```

Argo CD shows OutOfSync.

Now sync:

```bash
argocd app sync nginx-demo
```

```bash
kubectl get svc nginx-demo
```

**Expected result:**

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx-demo   ClusterIP   10.96.x.x    <none>        80/TCP    10m
```

The Service **still exists**. Argo CD did **not** delete it.

**Why?**

By default, Argo CD does not delete resources that were removed from Git. This is called **pruning** and it is disabled by default for safety. You have to explicitly enable it.

This is a deliberate safety choice — you do not want Argo CD to accidentally delete a resource just because someone made a typo in a path.

### Restore

```bash
git mv 06-first-application/service.yaml.bak 06-first-application/service.yaml
git commit -m "restore: add service back to git"
git push
argocd app sync nginx-demo
```

### What you learned

- Removing a manifest from Git makes the app OutOfSync
- Without `prune: true`, Argo CD will NOT delete resources from Kubernetes
- Pruning must be explicitly enabled — it's a safety feature
- This becomes important when you want Git to fully control everything (including deletions)

---

## Summary — what you now understand

| Concept | What it means |
|---------|--------------|
| **Desired state** | What Git says should exist |
| **Actual state** | What is currently running in Kubernetes |
| **Synced** | Desired == Actual |
| **OutOfSync** | Desired != Actual |
| **Healthy** | Resources are working correctly |
| **Missing** | Resources don't exist in Kubernetes yet |
| **Drift** | Someone changed Kubernetes without going through Git |
| **Reconciliation** | Argo CD comparing desired and actual state |
| **Manual sync** | You trigger the sync yourself |
| **Pruning** | Deleting resources from Kubernetes when removed from Git |
| **Kubernetes self-healing** | Deployment controller recreates deleted Pods (not Argo CD) |

---

## The GitOps rule

> **Git is the source of truth. Never run `kubectl apply` or `kubectl scale` in production when using GitOps. Change Git instead.**

If you want to scale from 3 to 5 replicas: change the YAML in Git and sync. That way:
- The change is tracked in Git history
- Anyone can see what changed and why (commit message)
- It can be reviewed (pull request)
- It can be rolled back (revert the commit)

---

## What to explore next

Now that you understand the fundamentals, here are good next topics to add to this lab:

- **Auto-sync** — Argo CD syncs automatically when Git changes (no manual click)
- **Self-heal** — Argo CD reverts manual kubectl changes automatically
- **Prune** — Argo CD deletes resources removed from Git
- **Helm** — manage apps with Helm charts instead of raw YAML
- **Kustomize** — manage multiple environments (dev/staging/prod) with overlays
- **ApplicationSet** — auto-generate multiple Applications from a template
- **Argo CD Image Updater** — automatically update image tags when a new image is pushed

Each of these is a small, practical addition to what you have already built.
