# Kubernetes Notes: Probes

Liveness, readiness and startup probes: what they check and what happens when they fail.

---

## 1. Probes

Without probes, Kubernetes only knows if the container **process** is running. Not whether the app is working.

> **Course analogy:** a doctor checking a patient. "Are you alive? Are you well enough to work?"

| Probe | Question | If it fails |
|---|---|---|
| **Liveness** | Is the container still alive (not stuck/deadlocked)? | **Kubelet restarts the container** |
| **Readiness** | Is it ready to receive traffic? | **Removed from Service endpoints.** No restart. Traffic stops until it passes again |
| **Startup** | Has a slow app finished starting? | Liveness and readiness are **disabled until it passes** |

### How they check
- **HTTP GET:** call an endpoint (e.g. `/healthz`), expect a success response. Most common.
- **TCP socket:** can a connection be opened on this port?
- **Exec:** run a command inside the container, check the exit code.
- **gRPC:** for apps that speak gRPC natively (GA since 1.27).

### Key settings
- `initialDelaySeconds`: wait before the first check
- `periodSeconds`: how often to check
- `failureThreshold`: how many failures before acting

### Example
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10          # check every 10s, restart if it fails
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10          # 30 x 10s = up to 5 minutes to start
```

**Separate endpoints, separate questions:** `/health` = "am I alive?", `/ready` = "are my dependencies connected?". Some apps use one endpoint for all three.

### Common mistake
**Don't make the liveness probe depend on external services.** If the database goes down and liveness checks the database, every pod gets restarted, the database is still down, and you've created a restart loop for nothing. **Liveness should only check the process itself.** Dependencies belong in readiness.

> **Real example from the Dashboard lab:** the liveness probe checked HTTPS on 8443 while the app served HTTP on 9090. The app was fine, but Kubernetes kept killing it. A misconfigured probe can take down a healthy app.

---