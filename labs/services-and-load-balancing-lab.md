# Module 04: Services & Load Balancing

## Lab 1: ClusterIP

**1. Deployment to expose**
```bash
kubectl create deployment nginx --image=nginx:1.21 --replicas=3
kubectl wait --for=condition=Available deployment/nginx --timeout=60s   # blocks until ready
kubectl get pods -l app=nginx -o wide                                   # each pod has its own IP, changes on restart
```

**2. Expose it**
```bash
kubectl expose deployment nginx --port=80 --target-port=80   # ClusterIP by default, selector copied from the deployment
kubectl get svc nginx                                        # stable CLUSTER-IP
kubectl describe svc nginx | grep -A2 Selector               # app=nginx
kubectl get endpoints nginx                                  # the pod IPs it sends traffic to
```

**3. Test from inside the cluster**
```bash
kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- nginx
kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- nginx.default.svc.cluster.local
```
- Throwaway pod: `--rm` deletes it after, `-it` shows the output, `--restart=Never` makes it a plain pod.
- `wget -qO-` fetches the page and prints it.
- Short name works in the same namespace. Full name: `<svc>.<namespace>.svc.cluster.local`.

**4. Load balancing** (`web` deployment, 3 `hashicorp/http-echo` pods on port 5678)
```bash
kubectl expose deployment web --port=80 --target-port=5678   # service port 80 -> container 5678
for i in {1..6}; do
  kubectl run curl-$i --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- web
done
```
Every pod replies `hello`, so the output doesn't show which pod answered. `kubectl logs -l app=web --prefix` shows the requests spread across pods. kube-proxy picks a pod at random, not strictly round-robin.

**5. Service YAML with different ports** (`backend`: `port: 8080`, `targetPort: 80`)
```bash
kubectl apply -f backend-svc.yaml
kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- backend:8080
```

**6. Multiple ports** (`multi-port`: 80 and 443)
```bash
kubectl apply -f multi-port-svc.yaml
kubectl get svc multi-port    # 80/TCP,443/TCP
```
With more than one port, each one needs a `name`.

---

## Lab 2: NodePort

```
nodePort (30000-32767, on every node) -> port (service) -> targetPort (container)
```

**1. Create and find the port**
```bash
kubectl expose deployment nginx --type=NodePort --port=80 --name=nginx-nodeport
kubectl get svc nginx-nodeport       # PORT(S) 80:3XXXX/TCP, the second number is the nodePort
NODE_PORT=$(kubectl get svc nginx-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl $NODE_IP:$NODE_PORT             # nginx page from outside the cluster
```
`?(@.type=="InternalIP")` is a jsonpath filter: pick the address whose type is InternalIP.

In Killercoda: **Traffic / Ports** > enter the nodePort.

**2. Fixed nodePort** (`nginx-fixed-port`: `nodePort: 30080`)
```bash
kubectl apply -f specific-nodeport.yaml
curl $NODE_IP:30080
```

**3. Out-of-range port is rejected** (`nodePort: 8080`)
```bash
kubectl apply -f invalid-nodeport.yaml 2>&1 || echo "Failed as expected!"
```
`2>&1` shows the error, `|| echo` runs only if the command failed.

**4. A NodePort service still has a ClusterIP**
```bash
kubectl get svc nginx-nodeport -o jsonpath='{.spec.clusterIP}'
kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- nginx-nodeport   # internal access still works
```

Downsides: only 30000-32767, opens the port on every node, no TLS. Fine for dev and testing, use Ingress or LoadBalancer in production.

---

## Lab 3: LoadBalancer, ExternalName, services without selectors

**1. LoadBalancer without a cloud**
```bash
kubectl expose deployment nginx --type=LoadBalancer --port=80 --name=nginx-lb
kubectl get svc nginx-lb                 # EXTERNAL-IP <pending>: no cloud provider to create the LB
NODE_PORT=$(kubectl get svc nginx-lb -o jsonpath='{.spec.ports[0].nodePort}')
curl $NODE_IP:$NODE_PORT                 # works, because a LoadBalancer also creates a NodePort
```
LoadBalancer = NodePort + a cloud load balancer in front. On bare metal, MetalLB can hand out the external IPs.

**2. ExternalName** (`external-db` -> `database.example.com`)
```bash
kubectl apply -f external-db.yaml
kubectl get svc external-db              # no ClusterIP, EXTERNAL-IP shows the DNS name
kubectl run dns-test --image=busybox:1.36 --restart=Never -it --rm -- nslookup external-db   # CNAME to database.example.com
```
Just DNS: no proxying, no port mapping, no health checks. The HTTP Host header must match the external name.

**3. Service without a selector + manual Endpoints** (`external-service` pointing at `93.184.216.34:80`)
```bash
kubectl apply -f external-endpoint.yaml
kubectl get endpoints external-service   # the IP I set, not pods
```
- No selector means Kubernetes doesn't manage the endpoints. I create an `Endpoints` object with the **same name** as the service.
- Unlike ExternalName, this points at an IP, not a DNS name, and traffic does go through kube-proxy.
- Newer clusters print a warning that `Endpoints` is deprecated in favour of `EndpointSlice`. Still works.

---

## Lab 4: DNS and service discovery

**1. Long-running debug pod**
```bash
kubectl run dns-debug --image=busybox -- sleep 3600    # stays up so I can exec into it
kubectl wait --for=condition=Ready pod/dns-debug --timeout=30s
kubectl exec dns-debug -- nslookup nginx               # resolves to the ClusterIP
kubectl exec dns-debug -- nslookup nginx.default.svc.cluster.local
```

**2. Cross-namespace**
```bash
kubectl create namespace other
kubectl create deployment httpd --image=httpd -n other
kubectl expose deployment httpd --port=80 -n other
kubectl exec dns-debug -- nslookup httpd.other.svc.cluster.local
kubectl exec dns-debug -- wget -qO- httpd.other        # <svc>.<namespace> is enough
```
Short name only works in the same namespace. Other namespaces need `.namespace` added.

**3. Pod DNS records**
```bash
POD_IP=$(kubectl get pod dns-debug -o jsonpath='{.status.podIP}' | tr '.' '-')   # 10.244.0.5 -> 10-244-0-5
kubectl exec dns-debug -- nslookup $POD_IP.default.pod.cluster.local
```
Format: `<ip-with-dashes>.<namespace>.pod.cluster.local`. Rarely used.

**4. Regular vs headless** (`nginx-headless`: `clusterIP: None`, selector `app=nginx`)
```bash
kubectl exec dns-debug -- nslookup nginx            # one ClusterIP
kubectl exec dns-debug -- nslookup nginx-headless   # every pod IP
```

**5. CoreDNS and troubleshooting**
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns   # CoreDNS pods running?
kubectl get svc -n kube-system kube-dns               # the DNS service
kubectl get configmap coredns -n kube-system -o yaml  # config: cluster.local zone, caching, forwarding outside queries
kubectl exec dns-debug -- cat /etc/resolv.conf        # nameserver = kube-dns ClusterIP, search domains
```
The `search` lines in resolv.conf are why short names like `nginx` work: the pod tries `nginx.default.svc.cluster.local` for me.

**6. SRV records**
```bash
kubectl exec dns-debug -- nslookup -type=SRV _http._tcp.nginx.default.svc.cluster.local
```
Format: `_<port-name>._<protocol>.<svc>.<namespace>.svc.cluster.local`. Only exists for **named** ports, so it doesn't resolve for `nginx`, which was made with `kubectl expose` and has no port name.

**7. Clean up**
```bash
kubectl delete deployment nginx web --ignore-not-found
kubectl delete svc nginx nginx-nodeport nginx-fixed-port nginx-lb web backend multi-port nginx-headless external-db external-service --ignore-not-found
kubectl delete endpoints external-service --ignore-not-found
kubectl delete namespace other          # deletes httpd with it
kubectl delete pod dns-debug
```
- `--ignore-not-found` skips anything already gone instead of erroring.
- `-n` applies to every name in a command, so `kubectl delete deployment nginx web httpd -n other` would only look in `other` and leave `nginx` and `web` running in `default`.

---

## Summary

| Type | Reachable from | Built on |
|---|---|---|
| ClusterIP | Inside the cluster | Default |
| NodePort | `<NodeIP>:30000-32767` | ClusterIP + a port on every node |
| LoadBalancer | Cloud LB external IP | NodePort + cloud LB |
| ExternalName | Inside the cluster, as a DNS alias | Nothing, just a CNAME |

```yaml
ports:
- port: 80          # what clients connect to
  targetPort: 8080  # where the container listens
  nodePort: 30080   # external port on every node (NodePort/LoadBalancer only)
```

| DNS name | Resolves to |
|---|---|
| `<svc>` | ClusterIP, same namespace |
| `<svc>.<ns>` | ClusterIP, any namespace |
| `<svc>.<ns>.svc.cluster.local` | Full name |
| Headless service | Pod IPs |

**Common mistakes**
- Selector doesn't match pod labels: empty endpoints, service goes nowhere. Check `kubectl get endpoints` first.
- `targetPort` doesn't match the container's port.
- NodePort in production: use Ingress or LoadBalancer.
- Unnamed ports: needed for multi-port services and SRV records.

**Useful commands**
```bash
kubectl expose deployment <name> --port=80 --target-port=8080
kubectl create service clusterip <name> --tcp=80:8080
kubectl create service nodeport <name> --tcp=80:8080
kubectl get endpoints <name>
kubectl run curl --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- <service>
```