# Module 03: Deployments & ReplicaSets

Only covers what's new since Modules 01 and 02 (basic deployments, scaling, `set image`, `rollout undo`, standalone ReplicaSets and self-healing are in those).

`cat <<EOF | kubectl apply -f - ... EOF` pipes the YAML straight into `apply` without saving a file. `-f -` means "read from input".

---

## Lab 1: Deployment and ReplicaSet relationship

**1. Create and inspect the ReplicaSet**
```bash
kubectl create deployment nginx --image=nginx:1.21 --replicas=3
kubectl get rs                        # name = deployment name + hash
kubectl describe rs $(kubectl get rs -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```
- `Controlled By: Deployment/nginx`
- Selector includes `pod-template-hash`, which ties pods to one version of the template.

**2. Scale by editing the live manifest**
```bash
kubectl get deployment nginx -o yaml > nginx-deployment.yaml   # export the running config
sed -i 's/replicas: 5/replicas: 2/' nginx-deployment.yaml     # find and replace in place
kubectl apply -f nginx-deployment.yaml
```

**3. Scaling doesn't create a new ReplicaSet**
```bash
kubectl get rs -l app=nginx           # still one RS
kubectl describe deployment nginx     # ScalingReplicaSet events
```
Only a change to the pod template (image, env, etc.) creates a new ReplicaSet.

---

## Lab 2: Rollouts in more depth

**1. Inspect a specific revision**
```bash
kubectl rollout history deployment/webapp
LATEST=$(kubectl rollout history deployment/webapp | awk '/^[0-9]+/ {r=$1} END {print r}')   # latest revision number
kubectl rollout history deployment/webapp --revision=$LATEST   # image and template for that revision
```
Revision numbers change after a rollback: `undo` reuses the old template and gives it a new, higher number, so the old entry disappears. Don't hardcode revision numbers.

**2. Broken rollout**
```bash
kubectl set image deployment/webapp nginx=nginx:broken-tag
kubectl rollout status deployment/webapp   # hangs, Ctrl+C
kubectl get pods -l app=webapp             # new pods ImagePullBackOff, old pods still Running
```
RollingUpdate stops at the first failed pods, so the app stays up on the old version.

**3. Roll back to a specific revision**
```bash
kubectl rollout undo deployment/webapp --to-revision=1
kubectl get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].image}'   # nginx:1.21
```

**4. Pause and resume (canary)**
```bash
kubectl set image deployment/webapp nginx=nginx:1.23
kubectl rollout pause deployment/webapp    # freezes mid-rollout: mix of old and new pods
kubectl rollout resume deployment/webapp
kubectl rollout status deployment/webapp
```

---

## Lab 3: ResourceQuotas and LimitRanges

**1. Namespace + ResourceQuota** (total for the namespace: `requests.cpu: 4`, `requests.memory: 8Gi`, `limits.cpu: 10`, `limits.memory: 16Gi`, `pods: 10`)
```bash
kubectl create namespace limited
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limited
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "10"
    limits.memory: "16Gi"
    pods: "10"
EOF
kubectl describe resourcequota compute-quota -n limited   # Used vs Hard
```

**2. LimitRange** (per container: default limit 500m/512Mi, default request 100m/128Mi, max 2/4Gi, min 50m/64Mi)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: limited
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
EOF
kubectl describe limitrange default-limits -n limited
```

**3. Deployment with explicit resources** (`resource-app`, 2 replicas, requests 200m/256Mi, limits 500m/512Mi)
```bash
kubectl describe resourcequota compute-quota -n limited   # Used: 400m CPU, 512Mi memory
```

**4. Push against the quota**
```bash
kubectl scale deployment resource-app --replicas=7 -n limited
kubectl get pods -n limited
kubectl describe deployment resource-app -n limited
kubectl scale deployment resource-app --replicas=2 -n limited
```
- 7 replicas is still inside the quota (1400m of 4 CPU, 7 of 10 pods). Any `Pending` pods here are the node running out of CPU (`kubectl describe pod` shows `Insufficient cpu`).
- A real quota breach (e.g. `--replicas=12`, over the 10-pod limit) doesn't give Pending pods. The pods are never created, and the error shows in `kubectl get events -n limited`: `exceeded quota`.

**5. Pod with no resources gets the LimitRange defaults** (`default-resources` pod, no `resources` section)
```bash
kubectl get pod default-resources -n limited -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"500m","memory":"512Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

**6. Pod over the LimitRange max is rejected** (3 CPU / 5Gi)
```
Error: maximum cpu usage per Container is 2, but limit is 3
```
Rejected at creation (admission), so the pod never exists.

**Notes**
- A quota on `requests.cpu` means every pod must declare requests, or get them from a LimitRange. Without one or the other, pods are rejected.
- LimitRange defaults only apply when a pod is created, not to pods already running.

---

## Lab 4: Label selectors

**1. Labels must be on the pod template** (`spec.template.metadata.labels`). Labels on the Deployment's own metadata don't reach the pods.

**2. Viewing labels**
```bash
kubectl get pods --show-labels        # all labels in one column
kubectl get pods -L tier,env,app      # each label as its own column
```

**3. Selectors**
```bash
kubectl get pods -l 'tier in (web,api)'           # value is one of
kubectl get pods -l 'app!=cache'                  # not equal
kubectl get pods -l 'tier in (web,api),env=prod'  # comma = AND
kubectl get pods -l tier                          # label exists (any value)
kubectl get pods -l '!tier'                       # label doesn't exist
```

**4. Act on everything matching a label**
```bash
kubectl scale deployment -l tier=api --replicas=4   # scales every matching deployment
```

**5. Add labels to existing objects**
```bash
kubectl label deployment frontend version=v1.0 team=platform
kubectl get deployments -l tier=web,env=prod,version=v1.0
kubectl get all -l team=platform      # deployments, replicasets, pods in one go
```
`kubectl label` on a deployment only labels the deployment. The pods don't get it unless it's in the template.

**Common label keys:** `env`, `tier`, `team`, `version`, `release` (canary / stable / beta).

---

## Lab 5: ReplicaSet internals

**1. Patch instead of set image**
```bash
kubectl patch rs standalone-rs -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'
kubectl get pods -l app=standalone -o jsonpath='{.items[*].spec.containers[0].image}'   # still 1.21
```
`patch` changes specific fields with a JSON snippet.

**2. Orphan pods**
```bash
kubectl delete rs standalone-rs --cascade=orphan              # delete the RS, keep its pods
kubectl get rs                                                # gone
kubectl get pods -l app=standalone                            # still running
kubectl describe pod -l app=standalone | grep "Controlled By" # nothing: no controller
```
Orphaned pods aren't recreated if they die.

**3. Adopt with a new controller**
```bash
# Deployment "adopted" with selector app=standalone, version=v1 and image nginx:1.22
kubectl get pods -l app=standalone
kubectl describe pod -l app=standalone | grep "Controlled By"
```
A controller takes ownership of existing pods whose labels match its selector.

**4. Show custom columns**
```bash
kubectl get pods -l app=standalone -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

**5. Restart all pods onto the current template**
```bash
kubectl rollout restart deployment adopted   # rolling replacement of every pod, no config change needed
```
Also useful for picking up changed ConfigMaps or Secrets.

**6. Selector change is refused**
```bash
kubectl get rs -l app=standalone -o name | head -1 | xargs kubectl patch -p '{"spec":{"selector":{"matchLabels":{"app":"different"}}}}'
# field is immutable
```
`xargs` passes the RS name from the first command into `kubectl patch`.

---

## Lab 6: RollingUpdate vs Recreate

**1. Watch each strategy** (`rolling-app`: `maxUnavailable: 2`, `maxSurge: 2`, 6 replicas. `recreate-app`: `strategy.type: Recreate`, 6 replicas)
```bash
kubectl get pods -l app=rolling-app -w     # terminal 1
kubectl set image deployment/rolling-app nginx=nginx:1.22   # terminal 2: up to 8 pods, never fewer than 4

kubectl get pods -l app=recreate-app -w
kubectl set image deployment/recreate-app nginx=nginx:1.22  # all 6 terminate, zero running, then 6 new
```

**2. Broken image on each**
```bash
kubectl set image deployment/rolling-app nginx=nginx:broken
kubectl rollout status deployment/rolling-app --timeout=30s   # gives up after 30s instead of hanging
kubectl get pods -l app=rolling-app       # old pods still serving
kubectl rollout undo deployment/rolling-app

kubectl set image deployment/recreate-app nginx=nginx:broken
kubectl get pods -l app=recreate-app      # every pod ImagePullBackOff: total outage
kubectl rollout undo deployment/recreate-app
```

**Tuning RollingUpdate**

| Setting | Effect |
|---|---|
| `maxUnavailable: 0`, `maxSurge: 1` | Safest. Full capacity the whole time, one extra pod at a time |
| `maxUnavailable: 1`, `maxSurge: 1` | Balanced |
| `25%` / `25%` (default) | Faster, needs spare capacity |

Lower `maxUnavailable` = safer, slower. Higher `maxSurge` = faster, needs more room on the nodes.

| | RollingUpdate | Recreate |
|---|---|---|
| Downtime | None (with readiness probes) | Yes |
| Old and new together | Yes, briefly | Never |
| Bad image | Old pods keep serving | Total outage |
| Use for | Stateless apps, APIs, production | Apps that can't run two versions at once, schema migrations |

**Other patterns built on these**
- **Blue-green:** run a second deployment, switch the Service selector to it, delete the old one.
- **Canary:** `rollout pause` part-way, check the new pods, then `resume`.