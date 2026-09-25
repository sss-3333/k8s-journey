# Lab: Module 01 - Pods & Deployments

**Platform:** Killercoda (CoderCo Live)  
**Status:** Completed

My first hands-on Kubernetes module. It starts with a single pod and builds up to production-style deployments with scaling, rolling updates and health probes.

## Why it matters

Everything in Kubernetes runs in pods: every service, application and job. Deployments are how real workloads are run. They keep the app running, roll out updates without downtime, roll back when something breaks, and scale up or down.

## Labs

| # | Lab | Covered |
|---|---|---|
| 1 | Your First Pod | Writing a pod manifest and applying it declaratively |
| 2 | Pod Deep Dive | Pod lifecycle: how pods are created, run and terminated |
| 3 | Multi-Container Pods | Sidecar and init container patterns |
| 4 | Creating Deployments | Declarative application management |
| 5 | Scaling & Rolling Updates | Manual and declarative scaling, zero-downtime updates |
| 6 | Health Probes | Liveness, readiness and startup probes |

The examples below are the core manifests and commands each lab covers.

---

## Lab 1: Your First Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-declarative
  labels:
    app: nginx
    environment: lab
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f nginx-pod.yaml             # declarative
kubectl run nginx-imperative --image=nginx  # imperative, for comparison
kubectl get pods -o wide
```

**Got it wrong first:**

```
error parsing nginx-pod.yaml: error converting YAML to JSON: yaml: line 2: mapping values are not allowed in this context
```

- **Cause:** I pasted a heredoc (`cat <<EOF > nginx-pod.yaml ... EOF`) into vi, so the shell command lines ended up inside the file.
- **Fix:** removed the first and last lines in vi (`dd`, `G`, `dd`, `:wq`). The other option is to paste the heredoc straight into the terminal.
- **Lesson:** a heredoc is a shell command that writes a file for you. It goes in the terminal, not in an editor. Always `cat` the file before applying it.

---

## Lab 2: Pod Deep Dive

```bash
kubectl describe pod nginx-declarative                          # full details + Events
kubectl get pod nginx-declarative -o jsonpath='{.status.phase}' # just the phase
kubectl get pod nginx-declarative -o yaml                       # everything Kubernetes filled in
kubectl logs nginx-declarative                                  # container output
kubectl exec -it nginx-declarative -- bash                      # shell inside the container
kubectl delete pod nginx-declarative
kubectl get pods -w                                             # watch it go Terminating, then disappear
```

- **Events** in `describe` show the lifecycle in order: Scheduled, Pulling, Pulled, Created, Started.
- **Phases:** Pending, Running, Succeeded, Failed, Unknown. Running only means the container started, not that the app is healthy.
- A bare pod has nothing looking after it. Deleted means gone.

---

## Lab 3: Multi-Container Pods

An init container writes a web page before nginx starts. A sidecar tails nginx's access log. They share files through `emptyDir` volumes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
spec:
  initContainers:
  - name: setup
    image: busybox
    command: ['sh', '-c', 'echo "<h1>Hello from the init container</h1>" > /html/index.html']
    volumeMounts:
    - name: html
      mountPath: /html
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: logs
      mountPath: /var/log/nginx
  - name: log-sidecar
    image: busybox
    command: ['sh', '-c', 'tail -F /var/log/nginx/access.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: html
    emptyDir: {}
  - name: logs
    emptyDir: {}
```

```bash
kubectl apply -f web-with-sidecar.yaml
kubectl get pods -w                                                  # Init:0/1 -> PodInitializing -> Running 2/2
kubectl exec web-with-sidecar -c log-sidecar -- wget -qO- localhost  # sidecar reaches nginx over localhost
kubectl logs web-with-sidecar -c log-sidecar                         # the request shows up in the sidecar's output
```

- **Init container** runs first and must finish before the main containers start.
- **Sidecar** runs alongside the app for the pod's whole life.
- Containers in a pod share the network (`localhost`) and any volumes mounted into both.
- With more than one container, `-c` picks which one `logs` and `exec` talk to.

---

## Lab 4: Creating Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx          # must match the selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deploy,rs,pods     # Deployment -> ReplicaSet -> 3 Pods
```

**Tested self-healing:**

```bash
kubectl delete pod --all
kubectl get pods        # 3 pods back within seconds: new names, AGE 46s, RESTARTS 0
```

- The ReplicaSet saw 0 of 3 pods running and immediately created replacements.
- The ReplicaSet hash in the names stayed the same, but the pod suffixes were new, so these were brand new pods, not restarts.
- To actually remove them, delete the **deployment**, not the pods.

---

## Lab 5: Scaling & Rolling Updates

**Scaling:**

```bash
kubectl scale deployment nginx-deployment --replicas=5   # imperative
# or change replicas: 5 in the YAML and re-apply          # declarative
kubectl get pods -o wide                                  # pods spread across nodes
```

**Rolling update:**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Update nginx to 1.22"
kubectl rollout status deployment/nginx-deployment
kubectl get rs                                            # old RS scaled to 0, new RS at 5
```

**Rollback:**

```bash
kubectl rollout history deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment
```

**Controlling the pace:**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # at most 1 extra pod during the update
      maxUnavailable: 0    # never drop below the desired count
```

- An update creates a **new ReplicaSet** and shifts pods over gradually.
- The old ReplicaSet is kept at 0, which is what makes a rollback instant.
- The declarative way is better for real work, because the YAML stays the record of what's running.

---

## Lab 6: Health Probes

```yaml
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        startupProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 30
          periodSeconds: 5       # up to 150s to start before liveness kicks in
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 10      # fails = container restarted
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5       # fails = no traffic, no restart
```

**Breaking the readiness probe on purpose** (`path: /does-not-exist`):

```bash
kubectl get pods        # STATUS Running, READY 0/1, RESTARTS 0
kubectl describe pod <name>   # Events: Readiness probe failed: HTTP probe failed with statuscode: 404
```

- The pod keeps running but gets no traffic until the probe passes.
- The same mistake on a **liveness** probe would restart the container over and over (see the Dashboard lab).
- Liveness should only check the app itself, never external dependencies like a database.

---

## Key takeaways

- Pod manifests always have `apiVersion`, `kind`, `metadata` and `spec`. Pods use `v1`, deployments use `apps/v1`.
- Deployment -> ReplicaSet -> Pods. The deployment keeps the desired number of pods running.
- Rolling updates replace pods gradually. Zero downtime depends on readiness probes.
- Liveness probe fails = restart. Readiness probe fails = traffic stops. Startup probe = protects slow starters.

## Related notes

- `pods.md`
- `multi-container-pods.md`
- `deployments.md`
- `probes.md`