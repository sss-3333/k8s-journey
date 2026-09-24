# Kubernetes Notes: Pod Networking Model and Service Meshes

The rule that underpins all Kubernetes networking, why service meshes exist, how they work, and how they differ from Ingress.

---

## 1. The Flat Pod Networking Model

> The course: "If you learn anything about Kubernetes networking, please learn this."

**The fundamental rule:** every pod gets its **own IP**, and every pod can talk to every other pod **directly, without NAT**.

(**NAT** = network address translation, where addresses are rewritten along the way, like a home router hiding all your devices behind one public IP.)

### The three rules
1. **Pods can talk to all other pods**, on any node, using the pod's IP directly. A pod on node A reaches a pod on node B with no translation.
2. **Nodes can talk to all pods** directly.
3. **A pod sees its own real IP.** The IP inside the container is the same IP everyone else uses to reach it. Nothing hidden.

> **Analogy:** one big open-plan office. Everyone has their own desk number, and anyone can walk straight to anyone else's desk. No receptionist in between rewriting who's who.

### What this gives you
- **A simple mental model:** treat each pod like its own small VM on one network.
- **No NAT headaches**, no "what's my real IP" confusion, no port mapping tables.

### What makes it work: the CNI plugin
The **CNI (Container Network Interface) plugin** builds this network. Examples: **Flannel, Calico, Cilium, Weave.** Each builds it differently under the hood, but from your point of view, pods just have IPs and can reach each other.

This is the same idea as the CRI from the architecture notes: Kubernetes defines a standard interface, and you plug in whichever implementation you want.

```bash
kubectl get pods -o wide     # the IP column shows real, routable pod IPs
```

### Where Services fit
Pods **can** talk directly by IP, but pod IPs change whenever pods are replaced. **Services** (see the Services notes) give a stable name and IP on top of this flat network.

### Security note
**By default, everything can talk to everything.** That's convenient, but not secure. **Network Policies** (covered later) are how you restrict it.

---

## 2. Why Service Meshes Exist

Picture 50 to 100 microservices. **Every one** needs:
- TLS between services, plus certificate rotation
- Retries, timeouts, circuit breakers
- Metrics, distributed tracing, logging
- Rate limiting, auth policies

Now multiply that by every service, every language, every team.

**What goes wrong:**
- **Duplicated logic everywhere.** Every team writes retries slightly differently.
- **Inconsistent** metrics, and security policies drift apart.
- **Debugging is a nightmare**, because everyone does their own thing.
- **Networking code leaks into the app.** Business logic gets tangled up with retry, TLS and timeout code.

**A service mesh pulls all of that out of the application and into the infrastructure.** The app just makes normal HTTP calls, and the mesh handles the rest.

> **Analogy:** instead of every department in a company running its own post room, security and couriers, the building provides one shared service. Departments just drop letters in the tray.

---

## 3. How a Service Mesh Works

Two parts, the same split as Kubernetes itself:

| Part | What it is | Examples |
|---|---|---|
| **Data plane** | A **proxy sidecar** injected into every pod. All traffic in and out goes through it. | Envoy (used by Istio), Linkerd proxy |
| **Control plane** | The **brain**. Pushes config (certificates, routing rules, policies) to every proxy. | istiod (Istio), Linkerd controller |

### Data plane
- A proxy container is added **alongside your app container, in the same pod** (the sidecar pattern from the Multi-Container Pods notes).
- The app makes a normal call. The proxy **intercepts** it, since they share the pod's network, and handles the rest.
- **The app doesn't know the proxy is there.**

### Control plane
- You change a policy once, and the control plane distributes it to every proxy in the mesh.

### What you get
- **mTLS everywhere:** encrypted, authenticated service-to-service traffic, with **no app code changes**. Proxies rotate certificates automatically.
- **Traffic control:** canary deployments, blue-green, retries, timeouts, all configured in the mesh.
- **Observability:** every request passes through a proxy, so you get consistent traces, logs and metrics across all services.
- **Consistent policies:** rate limiting and access control defined once, enforced everywhere.

**Result: the application gets simpler.** It just makes HTTP calls.

**Trade-off (from the sidecar notes):** a proxy in every pod costs CPU and memory. This is why Istio is moving to "ambient mode", which doesn't need a per-pod proxy.

---

## 4. Ingress vs Service Mesh

Often confused. **They handle different directions of traffic.**

| | Ingress | Service Mesh |
|---|---|---|
| Direction | **North-South**: from **outside** the cluster coming **in** | **East-West**: **inside** the cluster, service to service |
| Sits | At the **cluster edge** | Inside **every pod** (sidecar) |
| Handles | Routing external traffic to Services, TLS for external connections, host and path-based routing | mTLS between services, retries, timeouts, circuit breaking, internal observability |
| Question it answers | "How does outside traffic get **into** my cluster?" | "How do my services talk to each other **reliably and securely**?" |
| Examples | NGINX, Traefik, HAProxy | Istio, Linkerd |

> **Analogy:** Ingress is the **front door and reception**, deciding which visitor goes to which department. The service mesh is the **internal mail and security system** between departments.

**Most production setups use both:** Ingress at the edge, mesh inside. Don't use Ingress alone to secure internal traffic, and don't use a mesh alone to handle external traffic. Right tool for the job.

---