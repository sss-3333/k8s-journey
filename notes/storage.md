# Storage

## Why persistent storage

Pods are ephemeral. When a pod dies, anything written inside its container goes with it. Fine for a web server, a disaster for a database.

Persistent storage lives **outside the pod's lifecycle**, so a replacement pod picks up the same data. Essential for stateful apps: databases, message queues, anything that needs to remember previous state.

---

## PV and PVC

| | PersistentVolume (PV) | PersistentVolumeClaim (PVC) |
|---|---|---|
| What | A piece of storage in the cluster | A request for storage |
| Made by | Admin (static) or a provisioner (dynamic) | The app / developer |
| Says | "Here's 10Gi of EBS" | "I need 10Gi, one node read-write" |
| Scope | Cluster-wide | Namespaced |

- A PV **hides the backend**. The app doesn't care if it's a local disk, NFS, AWS EBS or a GCP persistent disk.
- Kubernetes finds a PV that satisfies the PVC and **binds** them. One-to-one: that PVC owns that PV.
- The pod only ever references the **PVC**, never the PV directly.

> **Analogy:** PV = a parking space. PVC = your permit asking for "one space, standard size". The pod just shows the permit.

```bash
kubectl get pv     # STATUS Bound, CLAIM shows the PVC
kubectl get pvc    # STATUS Bound, VOLUME shows the PV
```

---

## Access modes

| Mode | Short | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | Read-write by **one node** |
| ReadOnlyMany | ROX | Read-only by many nodes |
| ReadWriteMany | RWX | Read-write by many nodes. Not every storage type supports it (EBS doesn't, EFS/NFS do) |
| ReadWriteOncePod | RWOP | Read-write by **one pod** only |

RWO is per **node**, not per pod: several pods on the same node can still share an RWO volume. RWOP is the strict one-pod version.

---

## Dynamic provisioning

**Old way (static):** developer asks admin for 10Gi, admin creates the disk in the cloud, registers it as a PV, developer claims it. Admin becomes a bottleneck at scale.

**Dynamic:** define a **StorageClass** once ("how to create EBS volumes"). Developers just create PVCs and volumes appear automatically. Self-service storage.

### The flow
```
StorageClass  ->  PVC requests it  ->  provisioner creates the disk + PV  ->  PV binds to PVC  ->  pod mounts the PVC
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com      # who creates the volumes
parameters:
  type: gp3
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: gp3
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data             # where it appears in the container
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data          # reference the PVC by name
```

1. PVC created, pointing at the `gp3` StorageClass.
2. The EBS CSI driver sees it, calls the AWS API, creates an EBS volume.
3. The volume is registered as a PV and bound to the PVC.
4. The pod mounts the PVC. The volume is attached to the node and mounted at `/data`.

---

## StatefulSets and storage

Deployments treat pods as **cattle**: interchangeable, parallel, no identity. Stateful workloads need them as **pets**.

| Problem with Deployments | StatefulSet fix |
|---|---|
| New name and IP on restart | Stable names (`mysql-0`, `mysql-1`) + stable DNS via headless Service |
| Data lost with the pod | Own PVC per pod via `volumeClaimTemplates`, reattached on restart |
| Pods start in parallel | Ordered: 0, 1, 2 up. 2, 1, 0 down |

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
```
Creates `data-mysql-0`, `data-mysql-1`... one PVC per pod.

Used for: MySQL, PostgreSQL, Cassandra, Kafka, RabbitMQ, Elasticsearch, ZooKeeper.

---

## Ephemeral volumes

Storage that **lives and dies with the pod**. Simpler and faster when data doesn't need to survive.

| Type | Use |
|---|---|
| `emptyDir` | Empty folder created at pod start, deleted at pod end. Scratch space, caches, sharing files between containers in a pod |
| `emptyDir` with `medium: Memory` | RAM disk. Very fast, but counts against the container's memory limit |
| ConfigMap / Secret volumes | Config or credentials mounted as files |
| Downward API | Pod's own metadata (name, labels, annotations) as files |
| Ephemeral CSI | Temporary storage from a CSI driver (only some drivers) |

```yaml
volumes:
- name: cache
  emptyDir: {}
```

**Rule:** data must survive a restart -> PVC. Temporary space for the pod's life -> ephemeral volume.

---

## CSI (Container Storage Interface)

The standard plug-in interface for storage. Same idea as CRI for runtimes and CNI for networking.

- **Before:** storage drivers were built into Kubernetes itself. New backend = change Kubernetes core and wait for a release.
- **Now:** vendors write their own drivers that run as pods in the cluster. Kubernetes talks to all of them through the same API.

Benefits: vendor independent, driver updates don't wait on Kubernetes releases, consistent interface.

| Component | Runs | Does |
|---|---|---|
| Controller plugin | Somewhere in the cluster (usually a Deployment) | Creates, deletes and attaches volumes via the storage backend's API |
| Node plugin | Every node (DaemonSet) | Mounts volumes into pods on that node |

Common drivers: AWS EBS, AWS EFS, Azure Disk, GCP Persistent Disk, OpenEBS.

```bash
kubectl get csidrivers
kubectl get storageclass
```

---