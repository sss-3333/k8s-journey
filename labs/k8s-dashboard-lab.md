# Lab: Kubernetes Dashboard (Killercoda)

**Environment:** Killercoda browser lab (Kubernetes v1.36.1, Dashboard v2.6.0)  
**Lab link:** https://killercoda.com/coderco-live/course/kubernetes/k8s-dash-test

## The task
1. Deploy the Kubernetes Dashboard
2. Access it (via NodePort, then via `kubectl proxy`)
3. Explore cluster resources in the UI

**What the Dashboard is:** a web UI showing nodes, pods, deployments, services etc. It runs **inside the cluster as ordinary pods** in the `kubernetes-dashboard` namespace.

## Steps run
```bash
# 1. Deploy the dashboard from the lab's pre-made YAML
kubectl apply -f /root/dashboard.yaml
kubectl -n kubernetes-dashboard wait --for=condition=ready pod --all

# 2. Create an account with admin rights and get a login token
kubectl -n kubernetes-dashboard create sa admin-user
kubectl create clusterrolebinding admin-user --clusterrole cluster-admin --serviceaccount kubernetes-dashboard:admin-user
kubectl -n kubernetes-dashboard create token admin-user
```
- **ServiceAccount (`sa`):** an account for software/tools, not a person.
- **ClusterRoleBinding to `cluster-admin`:** gives that account full access, so the Dashboard can show everything. (RBAC, covered later.)
- **Token:** temporary password for that account, pasted into the Dashboard login.

## Two ways to access it

| Route | Path | Notes |
|---|---|---|
| **NodePort** (lab default) | Browser -> node port 30090 -> Dashboard pod (9090) | Opens a port on every node. Anyone who can reach the node can reach it. |
| **kubectl proxy** | Browser -> proxy (8001) -> API server -> Dashboard | Private tunnel using your kubectl login. Safer. What you'd use for internal tools. |

**A proxy = a middleman.** You go to it, it forwards your request on for you.

```bash
# In a second terminal tab (it keeps running)
kubectl proxy --address 0.0.0.0 --accept-hosts '.*'
```
- `--address 0.0.0.0` and `--accept-hosts` are only needed because Killercoda is viewed from your own browser, not the lab machine. By default the proxy only listens on localhost.

Proxy URL (Killercoda: take the port 30090 URL and change the port to 8001):
```
https://<session-id>-1-8001.spch.r.killercoda.com/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```
Meaning: "API server, in namespace `kubernetes-dashboard`, find service `kubernetes-dashboard`, forward me over HTTP."

**Lab-only settings (never do this for real):** skip login, HTTP instead of HTTPS, listening on all interfaces, `cluster-admin` for the Dashboard account.

## Issues and fixes

**Issue 1: 404 on the Dashboard home page**
- **Symptom:** "Not Found (404) - the server could not find the requested resource" on the Workloads overview.
- **Cause:** old Dashboard (v2.6.0) on a new cluster (v1.36.1). The overview loads Cron Jobs using an API version newer Kubernetes has removed.
- **Fix:** workaround only. Skip the overview and open individual pages (Pods, Deployments, Services, Nodes). Cron Jobs still 404s.

**Issue 2: 503, pod in CrashLoopBackOff (the real problem)**
- **Symptom:** `kubectl get pods -n kubernetes-dashboard` showed `0/1 CrashLoopBackOff`, 11 restarts in 28 minutes. NodePort route only worked in the brief windows the pod was up.
- **Investigation:**
  ```bash
  kubectl logs -n kubernetes-dashboard <pod-name> --previous    # no errors, app started fine
  kubectl describe pod -n kubernetes-dashboard <pod-name>       # read Events
  ```
- **Key Events:**
  ```
  Liveness probe failed: Get "https://192.168.0.150:8443/": connect: connection refused
  Container kubernetes-dashboard failed liveness probe, will be restarted
  ```
- **Cause:** the lab changed the Dashboard to serve **HTTP on 9090**, but the **liveness probe** still checked **HTTPS on 8443**. Every check failed, so the kubelet kept killing and restarting a healthy app.
- **Fix:** patch the deployment's probe to match:
  ```bash
  kubectl -n kubernetes-dashboard patch deployment kubernetes-dashboard --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/port","value":9090},{"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/scheme","value":"HTTP"}]'
  kubectl get pods -n kubernetes-dashboard -w     # new pod, 1/1 Running, 0 restarts
  ```
  Changing the pod template made the deployment replace the pod automatically.

**Liveness probe:** a regular health check Kubernetes runs on a container. If it fails several times in a row, the kubelet restarts the container. A misconfigured probe can kill a perfectly healthy app.

**`kubectl patch`:** edits one part of an existing resource without rewriting the whole YAML.

## Debugging flow used (reusable)
1. **Status:** `kubectl get pods` -> spotted CrashLoopBackOff and restart count
2. **Logs:** `kubectl logs <pod> --previous` -> output from the last crashed attempt. Clean logs = the app isn't failing itself
3. **Events:** `kubectl describe pod <pod>` -> showed Kubernetes killing it (liveness probe)
4. **Root cause:** probe port/scheme didn't match the app's config
5. **Fix and verify:** patch, then watch the new pod stay Running

## Takeaways
- Clean logs + restarts usually means **Kubernetes is killing the container** (probe failure or out of memory), not the app crashing. Check Events.
- `describe` Events show the whole chain: scheduler assigned it, kubelet pulled/created/started it, kubelet killed it.
- Config changes must stay consistent: changing a port in one place (args) but not another (probe) breaks things.
- Old tools on new clusters break when Kubernetes removes old APIs (same story as Docker/dockershim).