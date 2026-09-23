# Kubernetes Notes: Deployments, ReplicaSets, Updates and Rollbacks

How deployments manage pods through ReplicaSets, rolling updates vs recreate, and rolling back.

---

## 1. Deployments

**The "father of pods".** A controller that manages many identical pods for you.

What it does:
- Creates the pods and **keeps the right number running.** Delete one or let it fail, and a replacement appears.
- **Scales** up or down.
- **Rolling updates** with no downtime.
- **Rollbacks** to a previous version.

> **Analogy:** a project manager. You say "I always want 3 of these", and the manager makes it happen and keeps it that way.

### Deployment YAML
```yaml
apiVersion: apps/v1              # NOT v1 like pods
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                    # always keep 3 pods running
  selector:
    matchLabels:
      app: nginx                 # "I manage pods with this label"
  template:                      # the blueprint for each pod
    metadata:
      labels:
        app: nginx               # MUST match the selector above
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

How to read it:
- **`replicas`:** how many pods.
- **`selector.matchLabels`:** how the deployment finds the pods it owns.
- **`template`:** a pod definition nested inside the deployment. Everything under it is what you'd write in a standalone pod YAML.
- **Template labels must match the selector.** That's the link between the deployment and its pods.

```bash
kubectl apply -f nginx-deploy.yaml
kubectl get deploy
kubectl get rs                   # the ReplicaSet it created
kubectl get pods -o wide         # each pod has its own unique IP
```

### Deleting
```bash
kubectl delete deploy nginx-deployment
# or
kubectl delete -f nginx-deploy.yaml
```
- **Deleting a pod** managed by a deployment: it comes straight back.
- **Deleting the deployment:** all its pods go too.

---

## 2. ReplicaSets

**A ReplicaSet makes sure a set number of identical pods is running.** If one fails, it creates another.

**You rarely create them yourself.** A deployment creates and manages one for you:

```
Deployment  ->  ReplicaSet  ->  Pods
```

- **ReplicaSet:** keeps the count right. That's all.
- **Deployment:** manages ReplicaSets and adds rolling updates, rollbacks and scaling.

> **Course analogy:** ReplicaSets are the **worker bees** keeping the pod count right. The deployment is the **queen bee** deciding when pods should be added or replaced.

**Pod names show the chain:** `nginx-deployment-7c5ddbdf54-x8k2p`
- `nginx-deployment` = deployment name
- `7c5ddbdf54` = ReplicaSet ID
- `x8k2p` = unique pod ID

**Two versions under one deployment:** during an update, the deployment runs **two ReplicaSets**, one per version, and shifts pods between them. This is how rolling updates and rollbacks work (see Deployment Strategies and Rollbacks below).

Demo: create a deployment with 5 replicas, delete a pod, and `kubectl get rs` briefly shows 4 ready before the ReplicaSet brings it back to 5.

---

## 3. Deployment Strategies

### Rolling update (the default)
Gradually replaces old pods with new ones. **No downtime.**

1. You change the image or config.
2. Kubernetes creates a **new ReplicaSet**.
3. It scales the new one up while scaling the old one down.
4. For a while you have a **mix of old and new pods**, and traffic goes to both.
5. Old pods are removed as new ones become ready.

| Setting | Meaning | Default |
|---|---|---|
| `maxSurge` | How many **extra** pods can exist during the rollout | 25% |
| `maxUnavailable` | How many pods can be **down** at once | 25% |

Example: 4 pods, 25% surge = 1 extra, so you briefly have 5.

**Zero downtime depends on readiness probes.** New pods only get traffic once ready, and old pods keep serving until then.

```bash
kubectl rollout pause deployment/<name>     # freeze mid-rollout to investigate
kubectl rollout resume deployment/<name>
```

> **Analogy:** swapping shop staff one at a time so the shop never closes.

### Recreate
**Kills all old pods first, then starts the new ones.** Causes **downtime**.

```yaml
spec:
  strategy:
    type: Recreate
```

Use when old and new **can't run at the same time**:
- Database migration changes the schema, and the old version can't work with it.
- Storage conflict: a `ReadWriteOnce` volume can only be attached by one pod at a time.
- Breaking changes: incompatible protocols or shared caches.

> **Analogy:** closing the shop for a refit.

**Rule:** can old and new versions coexist? **Yes** = rolling update. **No** = recreate.

---

## 4. Rollbacks

Every update keeps the **old ReplicaSet**, scaled to 0 but still there. That's your rollback point.

```bash
kubectl rollout history deployment/<name>                    # list revisions
kubectl rollout undo deployment/<name>                       # back one version
kubectl rollout undo deployment/<name> --to-revision=2       # back to a specific revision
```

- A rollback **scales the old ReplicaSet back up and the current one down.** Same mechanism as a rolling update, in reverse. Fast.
- Kubernetes keeps **10 revisions** by default. Change with `revisionHistoryLimit`. Old ReplicaSets are stored in etcd, so they take up space.

**Pro tip:** add a change-cause annotation so the history shows what changed, not just numbers:
```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Upgrade nginx to 1.27"
```
The old `--record` flag is deprecated. Use the annotation.

---

## Commands
```bash
kubectl apply -f deploy.yaml
kubectl create deployment <name> --image=nginx --replicas=3
kubectl get deploy
kubectl get rs
kubectl delete deploy <name>

kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>s
```