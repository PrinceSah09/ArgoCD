# 05 — Access the Argo CD UI

## What are we doing?

The `argocd-server` Pod is running inside our cluster, but we cannot reach it directly from our browser because it is using a `ClusterIP` service — only accessible inside the cluster.

We will use `kubectl port-forward` to create a tunnel from our laptop to the service.

```
Your Browser
     │
     │ https://localhost:8080
     ▼
kubectl port-forward (tunnel)
     │
     ▼
argocd-server Service (ClusterIP)
     │
     ▼
argocd-server Pod
     │
     ▼
Argo CD
```

---

## Step 1 — Check the Argo CD service

```bash
kubectl get svc -n argocd
```

**Expected output:**

```
NAME                                      TYPE        CLUSTER-IP      PORT(S)
argocd-applicationset-controller          ClusterIP   10.96.x.x       7000/TCP
argocd-dex-server                         ClusterIP   10.96.x.x       5556/TCP,5557/TCP
argocd-metrics                            ClusterIP   10.96.x.x       8082/TCP
argocd-notifications-controller-metrics   ClusterIP   10.96.x.x       9001/TCP
argocd-redis                              ClusterIP   10.96.x.x       6379/TCP
argocd-repo-server                        ClusterIP   10.96.x.x       8081/TCP
argocd-server                             ClusterIP   10.96.x.x       80/TCP,443/TCP
argocd-server-metrics                     ClusterIP   10.96.x.x       8083/TCP
```

Notice `argocd-server` has `TYPE: ClusterIP`. That means it only has an internal IP — it is not reachable from outside the cluster yet.

**What is ClusterIP?**
A ClusterIP service gets an IP address that only exists inside the cluster. Pods inside the cluster can reach it, but your laptop cannot — not directly.

---

## Step 2 — Port-forward to access the UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Breaking this down:**

| Part | Meaning |
|------|---------|
| `kubectl port-forward` | Create a tunnel |
| `svc/argocd-server` | Forward to this service |
| `-n argocd` | The service is in the argocd namespace |
| `8080:443` | Local port 8080 → service port 443 |

**Expected output:**

```
Forwarding from 127.0.0.1:8080 -> 8443
Forwarding from [::1]:8080 -> 8443
```

Keep this terminal open. Port-forwarding stops when you close the terminal or press `Ctrl+C`.

> **Tip:** Open a new terminal tab for all other commands so the port-forward keeps running.

---

## Step 3 — Open Argo CD in your browser

Visit:

```
https://localhost:8080
```

**Browser certificate warning:**

You will see a warning like "Your connection is not private" or "This site is not secure". This is expected — Argo CD uses a self-signed TLS certificate by default in local installs.

- In Chrome: click **Advanced** → **Proceed to localhost (unsafe)**
- In Firefox: click **Advanced** → **Accept the Risk and Continue**
- In Safari: click **Show Details** → **visit this website**

This is only a concern in local development. In production, you would configure a proper TLS certificate.

---

## Step 4 — Get the initial admin password

Argo CD generates a random password during installation and stores it in a Kubernetes Secret.

Open a **new terminal** (keep the port-forward running) and run:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Breaking this down:**

| Part | Meaning |
|------|---------|
| `get secret argocd-initial-admin-secret` | Fetch this Kubernetes Secret |
| `-o jsonpath="{.data.password}"` | Extract just the password field |
| `\| base64 -d` | Decode it (Kubernetes Secrets store values as base64) |
| `&& echo` | Print a newline after the password |

**Why base64?**
Kubernetes Secrets store values encoded in base64. Base64 is **not** encryption — it is just encoding. Anyone with access to the Secret can decode it instantly. We will talk about real secret management in a later lab.

**Expected output:**

```
SomeRandomPassword123
```

Copy that password.

---

## Step 5 — Log in

In the Argo CD UI:

- **Username:** `admin`
- **Password:** the password you just retrieved

You should now see the Argo CD dashboard. It will be empty — no applications yet.

---

## Step 6 — (Optional) Change the admin password

```bash
# Install the argocd CLI first
brew install argocd

# Login via CLI
argocd login localhost:8080 --insecure --username admin --password <your-password>

# Change the password
argocd account update-password
```

The `--insecure` flag skips TLS verification for our self-signed cert. Never use this in production.

---

## What you see in the UI

When you first log in, the main page is the **Applications** view. It is empty because we have not created any Applications yet.

The navigation gives you:
- **Applications** — your deployed apps
- **Settings** — repositories, clusters, projects, users, RBAC

Explore the Settings → Clusters section. You will see `in-cluster` already registered — this represents the Kind cluster that Argo CD is running in, and it is automatically added during installation.

---

## Troubleshooting

**Port-forward dies / UI stops loading**
→ Run the port-forward command again in a terminal.

**Wrong password**
→ The password in the Secret is correct. Make sure you copied the full output without a trailing space. The `&& echo` at the end ensures a clean newline but doesn't add to the password itself.

**"Unable to connect" in browser**
→ The port-forward terminal was closed. Restart it.

**`argocd-initial-admin-secret` not found**
```bash
kubectl get secrets -n argocd
```
If it's not there, Argo CD may still be initializing. Wait a minute and try again.

---

## Summary

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
   └── Creates a tunnel: localhost:8080 → argocd-server:443

https://localhost:8080
   └── Argo CD UI

Username: admin
Password: kubectl -n argocd get secret argocd-initial-admin-secret \
            -o jsonpath="{.data.password}" | base64 -d
```

---

Next step: [06 — Create Your First Argo CD Application](../06-first-application/README.md)
