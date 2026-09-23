# Kubernetes Notes: Multi-Container Pods and Debugging

Init containers, sidecars, ambassadors, adapters, and ephemeral debug containers.

---

## 1. Multi-Container Patterns

All of these are just **extra containers in the same pod**, sharing its network (same IP, localhost) and optionally its volumes. The difference is **what the helper is for.**

| Pattern | Job | Analogy |
|---|---|---|
| **Init container** | Runs **before** the main app, then exits | Setting the table before dinner |
| **Sidecar** | Runs **alongside** the app, adds a capability | Motorbike sidecar: rides along, doesn't steer |
| **Ambassador** | Proxies the app's **outbound** connections | A receptionist who handles all your outgoing calls |
| **Adapter** | **Transforms** the app's output into another format | A UK-to-EU plug adapter |

### Init containers
"Init" = initialise, like `git init`, `terraform init`, or Linux's `init` process (PID 1, the first process at boot).

- Run **in order**: init 1 finishes, then init 2, then the main app.
- If one fails, the pod doesn't start. Kubernetes keeps retrying.
- **No probes.** They either complete or fail. They're not long-running.
- Defined under `spec.initContainers`, same structure as normal containers.

Uses: wait for a database to be reachable, run database migrations, fetch config from Vault or S3, set file permissions, create directories, set up proxy networking for service meshes like Istio.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
  - name: app
    image: my-app
```

### Sidecar containers
Extend the app **without changing its code.** One job each.

- **Log shipping:** app writes logs to a file, a Fluent Bit sidecar ships them to Elasticsearch.
- **Service mesh:** Istio/Linkerd inject a proxy sidecar. All traffic goes through it, giving mTLS, retries, tracing. The app just makes normal HTTP calls.
- **Secrets:** a Vault agent sidecar writes secrets to a shared volume and refreshes them when they rotate.

**Native sidecars (Kubernetes 1.29+):** built-in support that keeps the sidecar running for the pod's whole life.

> **Correction:** the video says native sidecars go in `containers`. They're actually defined in **`initContainers` with `restartPolicy: Always`**. That's what tells Kubernetes "start this first, but keep it running".
> ```yaml
> spec:
>   initContainers:
>   - name: log-shipper
>     image: fluent/fluent-bit
>     restartPolicy: Always
>   containers:
>   - name: app
>     image: my-app
> ```

**Trade-off:** every sidecar costs CPU and memory, and is one more thing to monitor. Across thousands of pods that adds up, which is why Istio is moving to "ambient mode" (no per-pod proxy).

### Adapter containers
For when you **can't change the app** (legacy, third-party).
- **Metrics:** legacy app exposes metrics in its own format. The adapter converts them to Prometheus format on `/metrics`. Exporters (Redis exporter, MySQL exporter) are adapters.
- **Logs:** convert odd log formats to JSON.
- **APIs:** translate REST to gRPC.

**Ambassador vs adapter:** ambassador = **where** traffic goes (routing). Adapter = **what shape** the data is in (translation).

---

## 2. Ephemeral Containers (Debugging)

**Problem:** production images are often minimal or "distroless", with no shell, no curl, no tools. When something breaks, you can't `exec` in and look around.

**Solution:** attach a **temporary debug container** to a running pod, without restarting it.

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```
- `--target` shares the **process namespace** with that container, so you can see its processes and reach its filesystem via `/proc`.
- Popular debug image: **netshoot** (`nicolaka/netshoot`), loaded with curl, dig, tcpdump, nmap for network debugging.

Rules:
- Only added to a **running** pod. You can't put them in the original spec.
- Share the pod's network (same IP, ports).
- **Don't restart or affect the pod.**
- Deleted when the pod is deleted. Nothing persists.
- Stable since Kubernetes 1.25.

> **Analogy:** a paramedic arriving with a kit. They work on the patient where they are, without moving them, and leave when done.

Use for production issues you can't reproduce elsewhere. A debugging tool, not somewhere to live.

---

## Commands
```bash
kubectl debug -it <pod> --image=busybox --target=<container>
```