# Kubernetes Notes: Labels, Selectors and Annotations

How Kubernetes objects find each other through labels, and where annotations fit.

---

## 1. Labels, Selectors and Annotations

### Labels
**Key/value pairs attached to any object**, in `metadata.labels`.
```yaml
metadata:
  labels:
    app: frontend
    environment: prod
```

Two jobs:
1. **Organisation:** filter and group. `kubectl get pods -l environment=prod`
2. **Selection (the big one):** Services, deployments and network policies find their targets **by label, not by name.** A Service doesn't know pod names. It knows "send traffic to pods labelled `app: frontend`".

> **Analogy:** hashtags. You don't search for a post by its ID, you search `#prod` and get everything tagged with it.

Use as many as make sense (team, environment, version, component). Teams often standardise on the **recommended labels** from the Kubernetes docs (`app.kubernetes.io/name`, `/version`, `/component` etc.).

### Label selectors
**How you query by label.**

| Type | Example | Meaning |
|---|---|---|
| **Equality-based** | `environment=prod`, `tier!=db` | Exact match (or not) |
| **Set-based** | `environment in (dev,qa)` | Value is one of these |
| | `tier notin (db)` | Value is not one of these |
| | `!partition` | Label doesn't exist at all |

```bash
kubectl get pods -l 'environment in (dev,qa),tier notin (db)'
```

Where they're used: Services (route traffic), deployments (which ReplicaSets/pods they own), network policies (which workloads to target), Jobs (find their pods).

**Services vs deployments syntax:**
```yaml
# Service: flat selector
selector:
  app: frontend

# Deployment and other controllers: matchLabels
selector:
  matchLabels:
    app: frontend
```

**The whole Kubernetes model runs on loose coupling:** controllers don't track object names, they track label patterns.

### Annotations
Same key/value format in `metadata`, **but not for selection.** You can't query or select by annotation.

Used for:
- **Tool config:** Prometheus scrape settings, ingress controller settings
- **CI/CD info:** build timestamps, commit IDs
- **Documentation:** descriptions, owner, links
- **Operational data:** `kubernetes.io/change-cause`, last applied config (Kubernetes uses these internally)

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    kubernetes.io/change-cause: "Bump image to v1.2"
```

> **Analogy:** labels are **name tags** people use to find you. Annotations are **sticky notes** with extra info that nobody searches by.

**Rule of thumb:** if something needs to **find** the object, use a **label**. If you're just **attaching info**, use an **annotation**.

---

## Commands
```bash
kubectl get pods -l app=nginx
kubectl get pods -l 'environment in (dev,qa),tier notin (db)'
kubectl get pods --show-labels
```