# Kubernetes Notes: Services (Exposing Applications)

What Services are, how they find pods, and the three main types: ClusterIP, NodePort, LoadBalancer. Includes the ClusterIP demo and how to test a Service.

---

## 1. Why Services Exist

**The problem:** pods come and go. Every time a pod is replaced (crash, update, scale), it gets a **new IP**. If your frontend talked to a backend pod by its IP, it would break the moment that pod restarted.

**The fix:** a **Service** gives a group of pods **one stable address and name** that never changes, and passes traffic on to whichever pods are currently healthy.

Example app:
- Frontend pods need to reach backend pods.
- Backend pods need to reach the database.
- Services connect each group without anyone needing to know individual pod IPs.

> **Analogy:** a company switchboard number. Staff come and go and change desks, but you always ring the same main number and get put through to someone who can help.

**Why it matters:** parts of your app stay **loosely coupled**. You can update or scale the backend without the frontend noticing.

---

## 2. How a Service Finds Its Pods

**By labels.** A Service has a `selector`, and it sends traffic to every pod whose labels match (see the Labels notes).

```yaml
selector:
  app: nginx        # "send traffic to any pod labelled app=nginx"
```

- The selector must match the **pod labels** (in a deployment, that's the labels under `template.metadata.labels`).
- Kubernetes keeps a live list of the matching, ready pod IPs. These are the Service's **endpoints**.
- **No matching ready pods = no endpoints.** That's the "no endpoints available" 503 from the Dashboard lab: the Service existed, but its only pod kept crashing.

```bash
kubectl get endpoints <service-name>    # which pod IPs the Service is sending to
```

**Behind the scenes:** **kube-proxy** on each node writes networking rules (usually **iptables**) so traffic sent to the Service's address gets forwarded to one of its pods.

> **Correction:** the video says Services work at **layer 3** of the OSI model. They actually work at **layer 4** (transport), handling TCP and UDP connections. Layer 7 (HTTP-aware) routing is what **Ingress** adds later.

---

## 3. The Three Main Service Types

| Type | Reachable from | Use for |
|---|---|---|
| **ClusterIP** (default) | **Inside the cluster only** | Internal communication (frontend to backend) |
| **NodePort** | Outside, via `<node IP>:<30000-32767>` | Local dev, testing, small setups |
| **LoadBalancer** | Outside, via a **cloud load balancer** | Production on AWS, Azure, GCP |

They build on each other: a **NodePort** Service also gets a ClusterIP, and a **LoadBalancer** Service also gets a NodePort and a ClusterIP.

---

## 4. ClusterIP (the default)

Kubernetes gives the Service a **stable internal IP** (the cluster IP) that only things **inside the cluster** can reach.

> **Course analogy:** the **internal phone line** of the cluster. Extensions work between desks, but nobody outside can dial them.

**Why use it:**
- **Secure by default:** nothing outside the cluster can reach it.
- **Simple:** it's the default type, so if you don't set `type`, you get ClusterIP.

### The ports
```yaml
ports:
- protocol: TCP
  port: 80          # the Service's own port (what other pods connect to)
  targetPort: 80    # the port the container is listening on
```
- **`port`** = the Service's port.
- **`targetPort`** = the container's port. If the app listened on 8080, this would be 8080.
- They're often the same, but don't have to be.

### Demo YAML: deployment and Service in one file
`---` separates multiple objects in one YAML file.

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
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f nginx-service.yaml    # creates both
kubectl get deploy
kubectl get svc
```
- Applying again after changing only the Service: the deployment shows `unchanged`, the Service is `created` or `configured`. `apply` only changes what's different.

> **Correction:** the demo says the `kubernetes` Service that's always in the default namespace is "for DNS". It's actually the Service for the **API server**, so pods can reach it. Cluster DNS runs separately as `kube-dns` in `kube-system`.

---

## 5. Testing a ClusterIP Service

A ClusterIP is internal, so you can't just open it in your browser. Two ways to test:

### Option 1: Port-forward (from your machine)
```bash
kubectl port-forward svc/nginx-service 8080:80
```
- Format is **`local-port:service-port`**. This means "my laptop's port 8080 goes to the Service's port 80".
- Open `http://localhost:8080` and you'll see the nginx page.
- It's a temporary tunnel. `Ctrl+C` stops it.
- A testing and debugging tool, not how you expose apps for real.

### Option 2: From inside the cluster (a temporary pod)
```bash
kubectl run tmp-shell --rm -it --image=nicolaka/netshoot -- bash
```
- `--rm` deletes the pod when you exit. `-it` gives you an interactive shell.
- **netshoot** is the network debugging image from the Multi-Container Pods notes (curl, dig, nslookup etc.).

Inside it:
```bash
curl http://nginx-service
exit
```
If you get the nginx page back, two things are proven: the Service works, and **pods can reach Services by name**.

### Service DNS names
Kubernetes gives every Service a DNS name:

```
<service-name>.<namespace>.svc.cluster.local
```

- Same namespace: just `nginx-service` works.
- Different namespace: `nginx-service.default` or the full name `nginx-service.default.svc.cluster.local`.

---

## 6. NodePort

Opens the **same port on every node** in the cluster and forwards traffic from it to the Service.

```yaml
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 8080         # Service port (inside the cluster)
    targetPort: 80     # container port
    nodePort: 30080    # opened on every node
```

Traffic path:
```
http://<node-IP>:30080  ->  Service :8080  ->  pod :80
```

- **NodePort range:** `30000-32767`. If you don't set `nodePort`, Kubernetes picks one from the range.
- Works on **every** node, even ones not running the pod. kube-proxy forwards it to wherever the pod is.

> **Analogy:** every building on a street agrees that door number 30080 leads to the same office, whichever building you walk into.

**Good for:** local dev, testing, small setups. **Downsides:** awkward high port numbers, you need to know node IPs, and nodes can change.

**Seen already:** the Dashboard lab used a NodePort on `30090`.

---

## 7. LoadBalancer

Kubernetes asks your **cloud provider** (AWS, Azure, GCP) to create a **real external load balancer**, which spreads incoming traffic across your pods.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Traffic path:
```
Users on the internet  ->  cloud load balancer (public IP)  ->  Service  ->  pods
```
- The "internal load balancer" in the video's diagram is the Service itself (kube-proxy spreading traffic across pods).
- On AWS this creates an actual AWS load balancer. That costs money, and each LoadBalancer Service gets its own.
- `kubectl get svc` shows the public address under `EXTERNAL-IP`.

> **On a local cluster (kind, Minikube):** there's no cloud provider to create the load balancer, so `EXTERNAL-IP` stays `<pending>` forever. That's expected, not a mistake. Local workarounds exist (e.g. `minikube tunnel`, or MetalLB), but for learning, stick with NodePort or port-forward.

**Good for:** production apps that need to be reachable from outside.

**Coming later:** an **Ingress controller** is usually exposed with one LoadBalancer Service, then routes to many apps behind it. That avoids paying for one load balancer per app.

---

## Commands
```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl port-forward svc/<name> 8080:80
kubectl run tmp-shell --rm -it --image=nicolaka/netshoot -- bash
kubectl expose deployment <name> --port=80 --target-port=80 --type=ClusterIP   # imperative way
kubectl delete svc <name>
```