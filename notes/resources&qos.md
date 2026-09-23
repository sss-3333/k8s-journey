# Kubernetes Notes: Resources, QoS, LimitRanges and ResourceQuotas

Requests and limits, QoS classes and eviction, and namespace-level resource governance.

---

## 1. Resource Requests and Limits

Every container can declare what it **needs** and what it's **allowed** to take.

| | Requests | Limits |
|---|---|---|
| Meaning | **Minimum guaranteed** | **Maximum allowed** |
| Used by | The **scheduler**, to find a node with enough room | The **kubelet/kernel**, to cap usage (via cgroups) |

```yaml
resources:
  requests:
    cpu: 250m          # guarantee a quarter of a CPU core
    memory: 128Mi
  limits:
    cpu: 500m          # can burst to half a core
    memory: 256Mi
```

### Units
- **CPU:** millicores. `1000m` = 1 core. `250m` = a quarter of a core.
- **Memory:** bytes, with suffixes.

> **Correction:** the video mixes these up. **`Mi` = mebibytes** (1024 x 1024 bytes). **`M` = megabytes** (1000 x 1000 bytes). They're slightly different sizes. `Mi` and `Gi` are what you'll see most.

### CPU vs memory when the limit is hit
- **CPU over limit:** **throttled.** The app slows down but keeps running.
- **Memory over limit:** **OOMKilled.** The container is terminated.

That's why getting **memory limits right matters more** than CPU.

### If you don't set them
- **No requests:** the scheduler assumes zero, so the pod might land on an already overloaded node.
- **No limits:** the container can use everything available. Fine until one container eats the whole node.

---

## 2. QoS (Quality of Service) Classes

When a node runs short of resources, Kubernetes has to **evict** some pods. QoS classes decide the order.

> **Course analogy:** plane classes. When turbulence hits, first class gets looked after first.

**You don't set QoS directly.** Kubernetes assigns it from your requests and limits:

| Class | Rule | Evicted |
|---|---|---|
| **Guaranteed** | **Every** container has CPU **and** memory requests **equal to** limits | **Last** |
| **Burstable** | Doesn't meet Guaranteed, but at least one container has a request or limit set | Middle |
| **BestEffort** | **No** requests or limits on any container | **First** |

- Critical workloads (databases, core APIs): aim for **Guaranteed**.
- Batch jobs or things that are fine to restart: **BestEffort** is OK.
- Most workloads end up **Burstable** (requests lower than limits to allow bursting).

**Shortcut to Guaranteed:** set only `limits`. Kubernetes copies them into `requests` automatically, so requests = limits.

Check a pod's class:
```bash
kubectl describe pod <name>      # look for "QoS Class"
```
(The Dashboard pod in the lab was `BestEffort`: no resources set at all.)

---

## 3. LimitRanges and ResourceQuotas

Both are **namespace-scoped**. The difference is the level they work at.

| | LimitRange | ResourceQuota |
|---|---|---|
| Applies to | **Each individual** container / pod / PVC | The **whole namespace combined** |
| Does | Sets **defaults**, **minimums**, **maximums** per object | Caps **total** CPU, memory, pod count etc. |
| Breaking it | Object rejected at admission | New object rejected if it would push the namespace over |

> **Analogy:** a department at work. The **ResourceQuota** is the department's total budget. The **LimitRange** is the per-person spending rule ("each claim between £10 and £200, and if you don't fill in an amount we'll assume £50").

### LimitRange
Fixes people forgetting to set resources, or setting silly values.
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:            # limits injected if none set
      cpu: 500m
      memory: 256Mi
    defaultRequest:     # requests injected if none set
      cpu: 250m
      memory: 128Mi
    min:
      cpu: 100m
    max:
      cpu: "1"
```
You can have several LimitRanges in a namespace. All of them apply.

### ResourceQuota
Stops the **noisy neighbour** problem: one team eating the whole cluster.
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    limits.cpu: "8"
    requests.memory: 8Gi
    limits.memory: 16Gi
    pods: "20"
```
Can also cap PVCs, services, secrets, ConfigMaps.

### Why use them together ("namespace and friends")
- **Without a LimitRange**, a ResourceQuota **rejects any pod that doesn't set resources** (it can't count what isn't declared).
- **Without a ResourceQuota**, a LimitRange alone doesn't cap the namespace total.

Layered enforcement:
```
ResourceQuota  -> how much can this whole namespace use?
LimitRange     -> what are the rules for each container?
Pod spec       -> what does this container actually need?
```

---