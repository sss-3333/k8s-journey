# ConfigMaps

## Why ConfigMaps

Apps need config: environment variables, URLs, file paths, feature flags. Hard-coding them means rebuilding the image whenever they change, and they often **differ per environment** (dev, staging, prod).

A ConfigMap is a storage box for **non-confidential** config data that pods can read. Change the config without touching the application code.

| | PV / PVC | ConfigMap |
|---|---|---|
| Handles | Data storage | Configuration |
| Holds | App data that must persist | Key-value settings / config files |
| Pod gets it as | A mounted volume | Env vars or mounted files |

> **Not for secrets.** ConfigMaps are plain text. Passwords, tokens and keys go in a **Secret**.

---

## Creating a ConfigMap

### Imperative
```bash
kubectl create configmap my-config \
  --from-literal=app_color=blue \
  --from-literal=app_mode=production
```
`cm` works as shorthand for `configmap`.

Other sources: `--from-file=app.properties` (file contents become a value), `--from-env-file=.env`.

### Declarative
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  app_color: blue
  app_mode: production
```

### Check it
```bash
kubectl get cm
kubectl get cm my-config -o yaml    # keys appear under data:
```

---

## Using a ConfigMap in a pod

Two main ways to get the values in:

| Method | Appears as | Picks up changes? |
|---|---|---|
| Env vars (`configMapKeyRef`) | `$app_color` inside the container | No, only read at container start. Restart the pod |
| Volume mount | One **file per key** in the mount path | Yes, the kubelet updates the files (after a short delay) |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
spec:
  containers:
  - name: demo
    image: busybox
    command: ["sleep", "3600"]
    env:
    - name: app_color
      valueFrom:
        configMapKeyRef:
          name: my-config        # which ConfigMap
          key: app_color         # which key in it
    - name: app_mode
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: app_mode
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config     # each key becomes a file here
  volumes:                       # same level as containers
  - name: config-volume
    configMap:
      name: my-config
```

Shortcut to load **every** key as env vars at once:
```yaml
envFrom:
- configMapRef:
    name: my-config
```

---

## Verifying inside the pod

```bash
kubectl apply -f configmap-pod.yaml
kubectl get pods

# As environment variables
kubectl exec -it cm-demo -- env | grep app_
# app_color=blue
# app_mode=production

# As mounted files
kubectl exec -it cm-demo -- cat /etc/config/app_color   # blue
kubectl exec -it cm-demo -- cat /etc/config/app_mode    # production
```

**Rule:** simple settings -> env vars. Whole config files (nginx.conf, app.properties) or values that should update live -> volume mount.

---
