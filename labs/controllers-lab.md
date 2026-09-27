# Module 02: Controllers

## Lab 1: ReplicaSets

Keeps a set number of identical pods running. Deployments create these automatically.

**1. Create a ReplicaSet** (`nginx-rs.yaml`: `kind: ReplicaSet`, 3 replicas, `nginx:1.21`)
```bash
kubectl apply -f nginx-rs.yaml
kubectl get rs                      # DESIRED 3, READY 3
kubectl get pods -l app=nginx       # only pods with label app=nginx
```

**2. Self-healing**
```bash
POD=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')   # save first pod's name
kubectl delete pod $POD
kubectl get pods -l app=nginx       # replacement already being created
```

**3. Scale**
```bash
kubectl scale rs nginx-rs --replicas=5    # adds 2 pods
kubectl scale rs nginx-rs --replicas=2    # removes 3 pods
```

**4. Selector is immutable**
```bash
kubectl get rs nginx-rs -o jsonpath='{.spec.selector.matchLabels}'   # can't be changed after creation
```

**5. Template changes don't update running pods**
```bash
kubectl set image rs/nginx-rs nginx=nginx:1.23     # updates the template only
kubectl get pods -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'   # still 1.21
kubectl delete pods -l app=nginx                   # recreated from new template (all at once = downtime)
```
This is why Deployments exist: they roll pods over to the new version gradually.

**6. Clean up**
```bash
kubectl delete rs nginx-rs          # deletes its pods too
kubectl get rs,pods -l app=nginx    # No resources found
```

---

## Lab 2: DaemonSets

Runs one pod on every node (or selected nodes). Used for log collectors, monitoring agents, CNI and CSI plugins.

**1. Create a DaemonSet** (`logging-ds.yaml`: `kind: DaemonSet`, no `replicas` field)
```bash
kubectl apply -f logging-ds.yaml
kubectl get ds                           # DESIRED/CURRENT/READY = number of nodes
kubectl get pods -l app=logging -o wide  # one pod per node
```

**2. Only run on labelled nodes** (`frontend-ds.yaml` has `nodeSelector: tier: frontend`)
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node $NODE tier=frontend
kubectl apply -f frontend-ds.yaml
kubectl get pods -l app=frontend-agent -o wide
```

**3. Inspect**
```bash
kubectl describe ds logging-agent        # Node-Selector, Tolerations, Events
kubectl logs -l app=logging --tail=5     # each pod prints its node's hostname
```

**4. Rolling update**
```bash
kubectl set image ds/logging-agent logger=busybox:1.36
kubectl rollout status ds/logging-agent
kubectl get ds logging-agent -o jsonpath='{.spec.updateStrategy}'   # RollingUpdate, maxUnavailable: 1
```

**5. Compare with a Deployment**
```bash
kubectl create deployment logger-deploy --image=busybox --replicas=3 -- sh -c "while true; do echo 'Deploy pod'; sleep 30; done"
kubectl get pods -o wide    # DaemonSet: 1 per node. Deployment: wherever there's room
```

**6. Clean up**
```bash
kubectl delete ds logging-agent frontend-agent
kubectl delete deployment logger-deploy
kubectl label node $NODE tier-     # trailing "-" removes the label
```

---

## Lab 3: StatefulSets

Pods with stable names, stable DNS and per-pod storage, created in order. Used for databases and queues.

**1. Headless Service** (`web-headless.yaml`: `clusterIP: None`)
```bash
kubectl apply -f web-headless.yaml
kubectl get svc web-headless        # CLUSTER-IP: None
```

**2. Create the StatefulSet** (`web-statefulset.yaml`: `serviceName: web-headless`, 3 replicas)
```bash
kubectl apply -f web-statefulset.yaml
kubectl get pods -l app=web -w      # web-0, then web-1, then web-2
```

**3. Stable names**
```bash
kubectl delete pod web-1
kubectl get pods -l app=web         # web-1 comes back with the same name
```

**4. Per-pod DNS** (`<pod>.<service>.<namespace>.svc.cluster.local`)
```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup web-0.web-headless   # returns web-0's IP
```

**5. Ordered scaling**
```bash
kubectl scale sts web --replicas=5  # adds web-3, then web-4
kubectl scale sts web --replicas=2  # removes web-4, web-3, web-2 (highest first)
```

**6. Storage** (reference only): `volumeClaimTemplates` gives each pod its own PVC (`data-web-0`, `data-web-1`) that survives restarts.

**7. Rolling update**
```bash
kubectl set image sts/web nginx=nginx:1.23
kubectl rollout status sts/web      # updates in reverse: web-2, web-1, web-0
```

**8. Compare naming**
```bash
kubectl create deployment nginx-deploy --image=nginx --replicas=3
kubectl get pods -l app=nginx-deploy -o name   # random suffixes
kubectl get pods -l app=web -o name            # web-0, web-1
```

**9. Clean up**
```bash
kubectl delete sts web
kubectl delete svc web-headless
kubectl delete deployment nginx-deploy
```

---

## Lab 4: Jobs & CronJobs

Jobs run a task to completion. CronJobs create Jobs on a schedule.

**1. Simple Job** (`countdown-job.yaml`: `kind: Job`, `restartPolicy: Never`)
```bash
kubectl apply -f countdown-job.yaml
kubectl get jobs -w                        # COMPLETIONS 0/1 -> 1/1
kubectl logs job/countdown                 # 5 4 3 2 1 Liftoff!
kubectl get pods -l job-name=countdown     # STATUS Completed, kept for logs
kubectl describe job countdown             # Events timeline
```

**2. Parallel Job** (`completions: 5`, `parallelism: 2`)
```bash
kubectl apply -f parallel-job.yaml
kubectl get pods -l job-name=parallel-work -w   # 2 at a time until 5 succeed
```

**3. Failing Job** (`backoffLimit: 3`, command ends in `exit 1`)
```bash
kubectl apply -f failing-job.yaml
kubectl get pods -l job-name=failing-job -w     # new Error pod for each retry
kubectl describe job failing-job | tail -10     # BackoffLimitExceeded
```

**4. CronJob** (`schedule: "*/1 * * * *"` = every minute)
```bash
kubectl apply -f hello-cron.yaml
kubectl get cronjobs
kubectl get jobs -w                              # new job appears each minute
kubectl logs job/$(kubectl get jobs -o jsonpath='{.items[-1].metadata.name}')   # logs of the newest job
```

**5. Concurrency policy**
```bash
kubectl get cronjob hello-cron -o jsonpath='{.spec.concurrencyPolicy}'   # Allow (default), Forbid, or Replace
```

**6. Trigger manually**
```bash
kubectl create job manual-hello --from=cronjob/hello-cron   # run now, no waiting
kubectl logs job/manual-hello
```

**7. Clean up**
```bash
kubectl delete job countdown parallel-work failing-job manual-hello
kubectl delete cronjob hello-cron
kubectl delete pods --field-selector=status.phase=Succeeded   # leftover Completed pods
kubectl delete pods --field-selector=status.phase=Failed      # leftover Error pods
```

---

## When to use what

| Need | Use |
|---|---|
| Stateless app | Deployment |
| Database cluster | StatefulSet |
| Agent on every node | DaemonSet |
| One-off task | Job |
| Recurring task | CronJob |