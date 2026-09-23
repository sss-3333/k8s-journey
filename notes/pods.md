# Kubernetes Notes: Pods

Pods, YAML structure, declarative vs imperative, pod phases, CrashLoopBackOff.

---

## 1. Pods

**A pod is the smallest thing Kubernetes creates and manages.** It wraps one or more containers.

Containers in the same pod:
- Are **always scheduled together** on the same node.
- **Share one IP address and port space**, so they talk to each other over `localhost`.
- Can share storage, but only if a **volume is explicitly mounted** into each container. They don't get shared storage by default.

> **Analogy:** a pod is a flat, and the containers are housemates. They share one address (IP) and can shout to each other across the hall (localhost). They only share a cupboard (volume) if one is deliberately set up for them.

---

## 2. Kubernetes YAML: The Four Required Fields

Every Kubernetes YAML file has four top-level fields:

| Field | What it is | Pod example |
|---|---|---|
| `apiVersion` | Which version of the Kubernetes API handles this object | `v1` |
| `kind` | What type of object you're creating | `Pod` |
| `metadata` | Identity info: name, labels, namespace, annotations | `name: nginx-pod` |
| `spec` | What the object should actually contain. **Different for every kind.** | containers, images, ports |

- `apiVersion` and `kind` are simple strings.
- `metadata` and `spec` are **dictionaries**, so everything under them is indented.
- `metadata` only accepts set fields (name, labels, namespace, annotations). But **inside labels** you can use any key/value you like.
- `spec` differs per object, so check the Kubernetes docs for the right format.

### Pod YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:            # a list, because a pod can have more than one container
  - name: nginx-container
    image: nginx         # no tag = defaults to :latest
    ports:
    - containerPort: 80  # nginx listens on port 80 (HTTP)
```

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods -w
kubectl get pods -o wide     # shows the pod's internal IP and which node it's on
```

- `.yaml` and `.yml` both work.
- **Indentation errors are the most common YAML mistake.** In the demo, `apply` failed with a parsing error because of one missing space on line 10. The error message gives you the line number, so start there.
- `containerPort` is mostly documentation. It records which port the app listens on. It doesn't open or expose anything by itself (that's what Services do).

---

## 3. Declarative vs Imperative

| | Declarative | Imperative |
|---|---|---|
| How | Write a YAML file, `kubectl apply -f file.yaml` | Type the whole thing as a command |
| Pod | YAML above | `kubectl run nginx --image=nginx` |
| Deployment | YAML (see the Deployments notes) | `kubectl create deployment nginx-deployment --image=nginx --replicas=2` |
| Good for | Real work: repeatable, reviewable, stored in Git | Quick tests |

The course recommends **declarative**, because you control exactly what goes in and the file becomes the record of what should exist.

See the full YAML Kubernetes generated for an imperative object:
```bash
kubectl get pod nginx -o yaml
```
It includes lots of fields you never set (creation timestamp, IPs, defaults). Kubernetes fills them in.

> **Useful tip (not in the videos):** generate a starter YAML without creating anything:
> ```bash
> kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
> ```

---

## 4. Pod Phases

The `STATUS` column in `kubectl get pods`. Every pod moves through these:

| Phase | Meaning | If stuck, check |
|---|---|---|
| **Pending** | Accepted by the API server, not running yet. Scheduler still deciding, or image still pulling. | Resources, node selectors, taints |
| **Running** | Bound to a node, **at least one container is up.** | Logs and events, because running doesn't mean healthy |
| **Succeeded** | All containers exited with **code 0** (success). What you want for Jobs/CronJobs. | Nothing, it finished |
| **Failed** | All containers stopped, **at least one with a non-zero exit code.** | Logs, exit codes |
| **Unknown** | Kubernetes can't get the pod's state. Usually the kubelet isn't responding or there's a network issue. | The node first |

**Key point:** phases describe the **pod's state, not your app's health.** A pod can be `Running` while the app returns errors. That's why probes exist (see the Probes notes).

**Exit code 0 = success. Anything else = something went wrong.**

---

## 5. CrashLoopBackOff

**Not a phase. It's a container state.** The pod can even show as Running while a container inside keeps dying.

The loop: container starts, crashes, kubelet restarts it, crashes again, kubelet waits, restarts, crashes, waits longer...

- The waiting is the **back-off**. It's **exponential**: 10s, 20s, 40s, 80s... capped at **5 minutes**.
- Kubernetes **never gives up**. It keeps retrying until you fix the cause.

### Common causes
- **Misconfiguration:** bad environment variables, wrong config path, missing secret. App starts, reads something that isn't there, crashes.
- **Unreachable dependency:** app tries to connect to a database that isn't ready, gets connection refused, crashes.
- **Probe failure:** a liveness or startup probe keeps failing, so Kubernetes kills the container. (This was the Dashboard lab bug.)
- **Memory limit too low (OOMKilled):** app needs 256Mi, limit is 128Mi, it gets killed.

### How to debug
```bash
kubectl describe pod <name>          # read Events at the bottom: why it died
kubectl logs <name> --previous       # logs from the last crashed attempt
```
- `--previous` matters because by the time you look, the container may have restarted and the current logs won't show the crash.

| Exit code | Usually means |
|---|---|
| `0` | Success |
| `1` | Generic app error |
| `137` | Killed by the system. Often **OOMKilled** (out of memory) |

---

## Commands
```bash
kubectl apply -f pod.yaml
kubectl run nginx --image=nginx
kubectl get pods -w
kubectl get pods -o wide
kubectl get pod <name> -o yaml
kubectl describe pod <name>
kubectl logs <name> --previous
kubectl delete pod <name>
```