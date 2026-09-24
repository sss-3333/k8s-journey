# Kubernetes Notes: Headless and ExternalName Services

Two special Service types that work through **DNS** instead of a normal cluster IP.

---

## 1. Headless Services

### Normal vs headless
| | Normal Service | Headless Service |
|---|---|---|
| Cluster IP | Yes, one virtual IP | **None** |
| DNS lookup returns | **One IP** (the cluster IP) | **Every pod's IP** (one A record each) |
| Load balancing | Done by Kubernetes (kube-proxy) | Done by the **client** (it picks a pod) |

All it takes is one line:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  clusterIP: None      # this is what makes it headless
  selector:
    app: my-db
  ports:
  - port: 5432
```

> **Analogy:** a normal Service is a **switchboard**: you ring one number and get put through to someone. A headless Service hands you the **staff phone book** with everyone's direct line, and you choose who to call.

### Why use it
- **StatefulSets:** each pod needs a **stable identity** and its own DNS name, so you can reach one specific pod, not a random one.
- **Client-side load balancing:** the client gets all the IPs and picks one itself. Some databases work this way, such as Cassandra.
- **Direct pod access:** testing, debugging, targeting one particular pod.

### DNS behaviour
**Normal Service:**
```
my-svc.default.svc.cluster.local   ->  10.96.0.15   (one cluster IP)
```

**Headless Service:**
```
my-db.default.svc.cluster.local    ->  10.244.1.5
                                       10.244.2.7
                                       10.244.1.9   (one A record per pod)
```

With a **StatefulSet**, each pod also gets its **own DNS name**:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

e.g.  my-db-0.my-db.default.svc.cluster.local
```
- The pod name is stable (`my-db-0`, `my-db-1`...), so the DNS name is stable.
- If the pod restarts and gets a **new IP**, the DNS name **stays the same** and updates to point to the new IP.

**Why this matters for databases:** replicas need to sync with **specific** other replicas ("talk to `my-db-0`, the primary"), not whichever pod happens to answer.

> **Note:** the per-pod DNS names come from StatefulSets (covered later). A headless Service in front of a normal deployment still returns all pod IPs, but the pods don't get stable individual names.

---

## 2. ExternalName Services

**A DNS alias. No cluster IP, no pods, no endpoints.** It points a Kubernetes Service name at a DNS name **outside** the cluster.

> **Course description:** "a DNS trick".

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.company.com
```

### What happens
1. A pod looks up `external-db` (full name `external-db.default.svc.cluster.local`).
2. Cluster DNS replies with a **CNAME** (an alias) pointing to `db.company.com`.
3. The pod looks up `db.company.com` itself and **connects directly**.

**Kubernetes doesn't proxy anything.** Traffic goes straight out to the external target.

> **Analogy:** saving a contact as "Dentist" in your phone. Your apps always call "Dentist". If you switch dentists, you update the contact once, and nothing else changes.

### Why use it
- **Integrate external systems:** your app talks to `external-db`, which is really an AWS RDS database or an external API.
- **Abstraction / easy migration:** app config always says `external-db`. Move from RDS to something else, update the one Service, and the apps don't change.
- **Consistency:** internal and external dependencies all use the same Kubernetes DNS naming pattern.

### Limitations
- **No port remapping.** You connect on whatever port the external service actually uses.
- **Some clients handle CNAMEs badly.** The most common real problem is with **HTTP and HTTPS**: the app sends `external-db` as the hostname, but the external server (and its TLS certificate) expects `db.company.com`, so requests or certificate checks can fail. (The transcript is garbled at this point. This is the usual gotcha.)

---

