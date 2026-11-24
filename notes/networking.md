# Kubernetes Notes: Networking

## The networking model

Three rules every Kubernetes cluster follows:

1. **Every pod gets its own IP**, and any pod can reach any other pod by that IP, on any node.
2. **Every node can reach every pod** directly.
3. **A pod sees the same IP for itself that everyone else uses to reach it.** One identity everywhere.

All of this happens without **NAT** (network address translation, where a router rewrites addresses on the way through, like a home router hiding all your devices behind one public IP). Pods talk to each other directly, so there's no address rewriting or port mapping to set up.

The **CNI plugin** (Calico, Cilium, Flannel) builds this network. By default **everything can talk to everything**. Network policies restrict that.

---

## Pod-to-pod communication

Two pods, one curls the other by IP:

```bash
kubectl run nginx --image=nginx       # single pod, not a deployment
kubectl run apache --image=httpd
kubectl get pods -o wide              # note each pod's IP
kubectl exec -it nginx -- sh          # shell inside the nginx pod
curl <apache-pod-IP>                  # "It works!" from Apache
```

No Service and no setup needed. Pods can reach each other by IP straight away.

Pod IPs change whenever a pod is replaced, so apps shouldn't rely on them. That's what Services and DNS are for.

---

## Service discovery and DNS

Kubernetes runs its own DNS server (**CoreDNS**, exposed as the `kube-dns` Service in `kube-system`). Every Service gets a DNS name, so pods find each other by **name** instead of IP.

```bash
kubectl create deployment apache --image=httpd
kubectl expose deployment apache --port 80     # ClusterIP service called "apache"
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port 80
kubectl get svc

kubectl run temp-shell --rm -i -t --image=nicolaka/netshoot   # throwaway pod with network tools
wget -qO- apache                                # all of these reach the same service
wget -qO- nginx.default
wget -qO- nginx.default.svc.cluster.local
cat /etc/resolv.conf                            # how it works
```

| Name | Works from |
|---|---|
| `nginx` | Same namespace |
| `nginx.default` | Any namespace |
| `nginx.default.svc.cluster.local` | Anywhere (full name) |

`/etc/resolv.conf` inside every pod contains:
- `nameserver`: the ClusterIP of the `kube-dns` Service, so every lookup goes to CoreDNS.
- `search`: domains like `default.svc.cluster.local` that get added to short names. That's why `nginx` on its own works.

Useful when debugging: if a service is reachable by IP but not by name, the problem is DNS.

---

## Network policies

Rules for **which pods can talk to which**. Without any, all pods can reach all pods, so a compromised pod could reach a database it should never touch.

Example: frontend, backend and database. Only the frontend should reach the backend, and only the backend should reach the database.

- Pods are picked with **label selectors**, like Services and Deployments.
- **Ingress rules** control incoming traffic to the selected pods.
- **Egress rules** control outgoing traffic from them.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
spec:
  podSelector:
    matchLabels:
      app: database          # applies to the database pods
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend       # only backend pods can connect
    ports:
    - port: 5432
```

- Once **any** policy selects a pod, all traffic of that type not explicitly allowed is blocked.
- Policies only work if the **CNI plugin enforces them** (Calico and Cilium do). On a CNI that doesn't, the policy is accepted but silently does nothing.

---

## Ingress and Ingress controllers

**Ingress** manages **HTTP/HTTPS access from outside the cluster**. It routes requests to Services by **domain or URL path**, through one entry point instead of a LoadBalancer per app.

| Part | What it is |
|---|---|
| **Ingress resource** | The routing rules (YAML). On its own it does nothing |
| **Ingress controller** | Pods that read the rules and actually route the traffic. Examples: Traefik, HAProxy, F5 NGINX, cloud ones like the AWS Load Balancer Controller |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  ingressClassName: nginx        # which controller handles it
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc        # example.com/api -> api-svc
            port:
              number: 80
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-svc        # example.com/app -> app-svc
            port:
              number: 80
```

- No controller installed = the Ingress is just rules nobody follows.
- The controller itself is usually exposed with **one LoadBalancer Service**, and every app sits behind it.
- An NGINX-based controller turns the rules into an NGINX config file inside its own pod. You only ever write the Ingress YAML.
- Ingress handles traffic **coming into** the cluster (north-south). Service meshes handle traffic **between** services inside it (east-west).

**Current state:** the community **ingress-nginx** controller was retired in March 2026 and gets no more security fixes. The Ingress API still works with other controllers, but **Gateway API** is the recommended replacement for new setups.

---

## Quick revision

- Every pod has its own IP. Pod to pod and node to pod, directly, no NAT.
- Pods can curl each other by IP with no setup, but pod IPs change, so use Services and DNS.
- CoreDNS (`kube-dns`) gives every Service a name: `<svc>`, `<svc>.<ns>`, `<svc>.<ns>.svc.cluster.local`.
- `/etc/resolv.conf`: nameserver = kube-dns, search domains make short names work.
- Network policies restrict pod traffic with label selectors. Ingress = incoming, egress = outgoing. Needs a CNI that enforces them.
- Ingress = HTTP routing by host and path. Needs an Ingress controller to do anything.
- ingress-nginx is retired. Gateway API is the successor.
