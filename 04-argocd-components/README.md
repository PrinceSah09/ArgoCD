# 04 — Argo CD Components

## Why does this matter?

Before touching the UI or deploying anything, it helps to understand what is actually running inside the `argocd` namespace and what each piece does. When something breaks, you will know which component to look at.

---

## The full picture

```
                        Git Repository
                             │
                             │ (reads YAML/Helm/Kustomize)
                             ▼
                       argocd-repo-server
                             │
                             │ (rendered manifests)
                             ▼
                 argocd-application-controller
                             │
                             │ (applies changes)
                             ▼
                      Kubernetes API
                             │
                             ▼
                    Your application Pods


                         ─────────

                           User
                             │
                             │ (browser / argocd CLI)
                             ▼
                       argocd-server
                             │
                             ▼
                 (reads state from application-controller)
```

---

## argocd-server

**What it is:** The front door of Argo CD.

It serves:
- The **web UI** you open in your browser
- The **REST API** used by the CLI and external tools
- The **gRPC API** used internally

**Analogy:** Think of it as the receptionist. It handles all incoming requests from humans and tools, but it does not do the actual deployment work.

```bash
# See it running
kubectl get pod -n argocd -l app.kubernetes.io/name=argocd-server

# See its service (the network endpoint)
kubectl get svc argocd-server -n argocd
```

---

## argocd-application-controller

**What it is:** The brain of Argo CD.

This is the most important component. It runs a continuous **reconciliation loop**:

```
Every few seconds:
  1. Read the Application resource (what Git says should exist)
  2. Read the actual state from Kubernetes (what is actually running)
  3. Compare them
  4. If different → mark as OutOfSync
  5. If sync is enabled → apply the difference
```

**Analogy:** A thermostat. It constantly checks the current temperature (actual state) against the target temperature (desired state) and adjusts if there is a difference.

This runs as a `StatefulSet` (not a regular Deployment) because it maintains local state.

```bash
kubectl get statefulset argocd-application-controller -n argocd
kubectl logs statefulset/argocd-application-controller -n argocd
```

---

## argocd-repo-server

**What it is:** The component that talks to Git.

When Argo CD needs to know what should be deployed, the repo-server:
1. Clones or fetches the Git repository
2. Reads the files at the specified path
3. If it is plain YAML → returns as-is
4. If it is a Helm chart → runs `helm template` to render manifests
5. If it is Kustomize → runs `kustomize build`
6. Returns the final rendered Kubernetes manifests to the application-controller

**Analogy:** The researcher. It fetches and prepares the information, then hands it to the brain (application-controller) to act on.

```bash
kubectl logs deployment/argocd-repo-server -n argocd
```

---

## argocd-applicationset-controller

**What it is:** Used to automatically generate multiple Argo CD Applications from a single template.

**Example use case:** Instead of manually creating 3 Applications for dev, staging, and prod, you write one `ApplicationSet` and it generates all three automatically.

We will cover this in a later lab. For now, just know it exists.

```bash
kubectl get deployment argocd-applicationset-controller -n argocd
```

---

## argocd-dex-server

**What it is:** Handles authentication and SSO (Single Sign-On) integrations.

Dex allows you to log into Argo CD using external identity providers like:
- GitHub
- Google
- LDAP
- SAML

For our local lab we use the built-in `admin` user and don't need Dex.

---

## argocd-redis

**What it is:** A Redis in-memory cache used by Argo CD internally.

Argo CD uses it to cache application state, repo data, and other information to avoid constantly re-computing or re-fetching things.

You will never interact with Redis directly in normal usage.

```bash
kubectl get pod -n argocd -l app.kubernetes.io/name=argocd-redis
```

---

## argocd-notifications-controller

**What it is:** Sends notifications when application state changes.

It can notify you via:
- Slack
- Email
- PagerDuty
- Webhooks

Example: "Your app `nginx-demo` just went OutOfSync in production."

We will not configure notifications in this lab, but knowing it exists is useful.

---

## How they all work together (sync flow)

Here is the full flow from Git push to running Pod:

```
1. You push a change to Git
        │
        ▼
2. argocd-repo-server detects the change
   (polls Git every 3 minutes, or webhook triggers immediately)
        │
        ▼
3. argocd-repo-server clones the repo and renders the manifests
        │
        ▼
4. argocd-application-controller compares rendered manifests
   with what is actually in Kubernetes
        │
        ▼
5. If different → Application becomes OutOfSync
        │
        ▼
6. If auto-sync is on (or you click Sync) →
   application-controller calls the Kubernetes API
        │
        ▼
7. Kubernetes creates/updates resources
        │
        ▼
8. Pods start running
        │
        ▼
9. Application becomes Synced + Healthy
```

---

## Quick check — all components running

```bash
kubectl get pods -n argocd
```

You should see one Pod for each component (some may have random suffixes on their names):

```
argocd-application-controller-0          1/1   Running
argocd-applicationset-controller-xxx     1/1   Running
argocd-dex-server-xxx                    1/1   Running
argocd-notifications-controller-xxx      1/1   Running
argocd-redis-xxx                         1/1   Running
argocd-repo-server-xxx                   1/1   Running
argocd-server-xxx                        1/1   Running
```

---

## Summary

| Component | Role |
|-----------|------|
| `argocd-server` | UI, API, CLI — the front door |
| `argocd-application-controller` | Reconciliation loop — the brain |
| `argocd-repo-server` | Reads Git and renders manifests |
| `argocd-applicationset-controller` | Auto-generates multiple Applications |
| `argocd-dex-server` | SSO / authentication |
| `argocd-redis` | Internal caching |
| `argocd-notifications-controller` | Sends alerts and notifications |

The two you will interact with most when troubleshooting are:
- `argocd-server` (UI issues, API issues)
- `argocd-application-controller` (sync issues, reconciliation issues)

---

Next step: [05 — Access the Argo CD UI](../05-argocd-ui/README.md)
