# Kubernetes Notes: Sections 1 & 2 - Basics, Architecture and Setup

---

## 1. The Big Idea

Kubernetes (K8s) sounds intimidating, but at its core it is just **Linux + networking + some system components, scaled across many machines.**

- On one Linux box you run processes, manage memory, handle network traffic.
- Kubernetes does the same things, but across a whole fleet of machines, from one command centre.

> **Analogy:** If one Linux machine is a single restaurant kitchen, Kubernetes is the head office running a chain of 50 kitchens. Same cooking, same rules, just coordinated at scale.

**Why "K8s"?** There are 8 letters between the K and the s in "Kubernetes".

**Why the ship wheel logo?** "Kubernetes" comes from the ancient Greek for **helmsman**, the person who steers the ship. K8s steers your containers.

### Module roadmap
1. Containers and container orchestration
2. Local cluster setup (using **Minikube**; alternatives are Kind, K3s, K3d)
3. Pods (smallest unit)
4. Deployments (manage groups of pods)
5. Services (exposing apps / networking)
6. Storage (PVs, PVCs)
7. Networking (network policies, ingress)
8. Security (RBAC)
9. Challenge / demo

---

## 2. Why Containers Exist

### The problem
Say you build a Python app. Deploying it means:
- Allocating resources
- Setting environment variables
- Installing the right libraries and runtime versions

Then if you move to a new environment or cloud provider, you repeat all of it. Slow, repetitive, error-prone. Classic "it works on my machine".

### The fix
A container packages **your code + everything it needs** (tools, libraries, config files, runtime) into one portable unit.

- The packaged blueprint is called a **container image**.
- A **container** is a running instance of that image.

> **Analogy:** An image is a recipe with all the ingredients pre-measured in a box. A container is the dish actually being cooked from that box. Hand the box to any kitchen and you get the same dish.

---

## 3. How Containers Are Built (the layers)

From bottom to top:

| Layer | What it does |
|---|---|
| **Hardware** | CPU, cores, storage, networking |
| **Host OS kernel** | Core of the OS. **All containers on the host share this one kernel.** |
| **Container runtime** | Docker, Podman, containerd etc. Manages containers and how they talk to the kernel |
| **Binaries & libraries** | Each container brings its own dependencies |
| **Application** | Your actual app, isolated from other containers |

Why this matters:
- Services are **decoupled**, so they can be run and updated independently.
- Environments are **predictable**, so fewer surprises in production.

Containers are not new. Earlier tech like **LXC** and Google's **LMCTFY** existed for over a decade. Docker made them easy and popular.

---

## 4. Containers vs Virtual Machines

| | Virtual Machine | Container |
|---|---|---|
| What's inside | App + libraries + **full guest OS** | App + libraries only |
| Typical size | App ~20-30MB, but guest OS 10GB+ | Just the app and its dependencies |
| Kernel | Each VM has its own | **Shares the host kernel** |
| Isolation | Very strong | Strong, but lighter |
| Resource cost | Heavy | Light |
| Portability | Slower to move | Very portable |

**How containers isolate without a guest OS:** two Linux kernel features.
- **Namespaces (Linux)** - control what a process can **see** (its own processes, network, filesystem view).
- **cgroups** - control what a process can **use** (how much CPU, memory).

> Note: these Linux namespaces are **not** the same thing as Kubernetes namespaces (section 11). Same word, different layer.

> **Analogy:** VMs are separate detached houses. Each has its own foundation, plumbing and wiring. Very private, but expensive. Containers are flats in one building. They share the foundation and plumbing (the kernel), but each flat has its own locked door and walls (namespaces) and its own share of the utilities (cgroups).

Bonus: container runtimes give you **base images** with an OS userland already installed, so you don't start from nothing.

---

## 5. The Problem Kubernetes Solves

Walk through the example:

1. You deploy a web app container (v1.1) on a VM with **4GB RAM, 2 cores**.
2. More users arrive, so you spin up **another VM** with the same app.
3. Devs release **v1.2**. You create a **third VM** before destroying the v1.1 ones.

Result: **3 containers using 6 cores and 12GB RAM.** One container per machine is hugely wasteful. Most of each VM sits idle.

**Kubernetes' answer:** pack as many workloads onto a node as its resources allow. Instead of one container per node, you run **many pods per node**, and the node's resources actually get used.

> **Analogy:** Sending three half-empty lorries when one full lorry would do. Kubernetes is the logistics planner that loads each lorry properly.

---

## 6. How We Got Here: Microservices

- Big organisations were stuck with **4-6 month release cycles**.
- Companies like **Netflix** moved to **microservices**: split one big app into lots of small, independent services.
- Big teams split into smaller ones. DevOps and Agile culture grew.

New problems appeared:
1. **How do we run each service anywhere?** Solved by **containers**.
2. **Now we have hundreds of containers. How do we manage them?** Solved by **container orchestration**, which is where Kubernetes comes in.

---

## 7. What Kubernetes Actually Is

**A platform that orchestrates the deployment, scaling and management of containerised applications.** Put simply: an application managing platform.

- Started at **Google**, based on how they ran containers in production.
- Now open source, part of the **CNCF** (Cloud Native Computing Foundation).
- The market leader for container orchestration.

**Core job:** schedule containers across your infrastructure, and keep them healthy.
- Dead or unhealthy containers get **replaced automatically**.

**Other capabilities:**
- Load balancing between nodes
- Rolling updates
- Mounting storage
- Accessing logs
- Distributing secrets
- Debugging apps
- Monitoring resources

> **Analogy:** An orchestra conductor. The conductor doesn't play any instrument (Kubernetes doesn't run containers itself, see section 9). It tells everyone when and what to play, and notices when someone drops out.

---

## 8. Kubernetes Architecture

Two halves: **the brain (control plane)** and **the muscles (worker nodes).**

> **Running analogy for this section: a restaurant chain.**
> - Control plane = head office and the manager's desk
> - Worker nodes = kitchens
> - Pods = plates of food going out

### Cluster
- The biggest unit. A **collection of nodes** providing compute, memory, storage and networking.
- Managed cloud versions: **EKS** (AWS), **AKS** (Azure), **GKE** (Google).
- You can have multiple clusters, but that's an advanced setup.

### Node
- **One machine** (virtual or physical). Its job is to run pods.
- Old name: **minions**. If you see "minion" in old docs, it just means node.

### Control plane (the brain)
Older name: **master node**. Decides *what* runs and *where*.

| Component | Job | Restaurant analogy |
|---|---|---|
| **kube-apiserver** | The **single entry point** for all commands. Everything talks through it. | The front counter. Every order goes through here, no exceptions. |
| **etcd** | Key-value store holding **all cluster data and desired state**. The source of truth. | The order book. If the kitchen burns down, you check the book to see what should be cooking. |
| **kube-scheduler** | Picks **which node** a new pod runs on, based on resources and rules (e.g. node affinity). | The manager deciding which kitchen has capacity for this order. |
| **kube-controller-manager** | Constantly compares **desired state vs actual state** and fixes differences. | The supervisor walking the floor: "We should have 3 of these plates out, I only see 2, make another." |
| **cloud-controller-manager** | Only on cloud providers. Talks to the cloud's API for **load balancers, storage, networking**. | The person who phones suppliers (the cloud) to get extra equipment delivered. |

### Worker nodes (the muscles)
Decide *how* things run, and do the heavy lifting.

| Component | Job | Restaurant analogy |
|---|---|---|
| **kubelet** | Agent on each node. Watches the API server, makes sure the pods assigned to its node are running and healthy, and reports back. | The head chef of one kitchen, reading tickets and making sure each plate gets made. |
| **kube-proxy** | Handles **network routing for Services**, so traffic sent to a Service reaches the right pods, wherever they are. | The waiter who knows which table each dish goes to. |
| **Pods** | Smallest deployable unit. A **wrapper around one or more containers**. | The plate. The food (containers) sits on it, and the plate is what gets served and tracked. |

> **Clarifications worth knowing (the video simplifies these):**
> - **kube-proxy** mainly handles **Service** routing. The basic pod-to-pod network itself is set up by a **CNI plugin** (e.g. Calico, Flannel). Likely covered later in the networking section.
> - The video says a pod can hold "thousands" of containers. Technically possible, but in practice a pod is usually **one main container**, sometimes with a small helper ("sidecar"). Containers in a pod share network and storage, so you only group ones that are tightly coupled.
> - "Master node" is the older term. Current docs say **control plane**.

### One-line summary
**Control plane decides the what and where. Worker nodes handle the how.**

---

## 9. Container Runtimes: Who Actually Runs Containers

Key point: **Kubernetes doesn't run containers itself.** It hands that job to a **container runtime** (the engine).

Kubernetes talks to runtimes through a standard plug called the **CRI (Container Runtime Interface)**. Any runtime that speaks CRI can be used.

> **Analogy:** CRI is like a standard plug socket. Kubernetes doesn't care which appliance you plug in, as long as it fits the socket.

### The options

| Runtime | Notes |
|---|---|
| **Docker** | The classic. Didn't speak CRI natively, so K8s needed a translator called **dockershim**. That was **removed in v1.24**. Still great for **building images locally**, just not used as the K8s runtime anymore. |
| **containerd** | **The default on EKS, GKE and AKS.** Lightweight, reliable. The standard answer. |
| **CRI-O** | Red Hat's runtime, built specifically for Kubernetes. Used in **OpenShift**. |
| **gVisor** | Sandbox runtime. Intercepts system calls in user space for extra isolation. |
| **Kata Containers** | Sandbox runtime. Runs each container in a **tiny micro VM**. |

- Use the sandbox ones (gVisor, Kata) for **untrusted workloads** where you want harder boundaries.

> **Why Docker images still work:** Docker builds images to a shared standard (OCI). containerd and CRI-O run that same format. So "Docker was removed" doesn't mean your Docker-built images stop working. Only Docker-as-the-engine inside K8s went away.

**Interview answer:** "containerd for most cases, CRI-O if you're in Red Hat / OpenShift land, the rest are niche."

**Mental model:** kubelet -> CRI -> runtime.

---

## 10. End-to-End: How a Container Actually Starts

The instructor flags this as a **favourite interview question.** Learn the chain.

```
You (kubectl apply / CI pipeline)
   -> API server        validates the request, stores it in etcd
   -> Scheduler         picks a node, writes that onto the pod
   -> kubelet           on that node, notices "this pod is mine"
   -> CRI (over gRPC)   kubelet hands the pod spec to the runtime
   -> Runtime           containerd / CRI-O: pull image, set up namespaces, cgroups, networking
   -> runc              makes the Linux system calls (clone, unshare, exec)
   -> Linux kernel      process is running = container is live
```

### Step by step

1. **You declare intent.** `kubectl apply` hits the API server. **Nothing is running yet.** You've only said "I want this pod to exist."
2. **API server** validates it and **saves it to etcd**. The cluster now knows what you want.
3. **Scheduler** checks nodes (resources, taints, affinity) and **assigns a node**.
4. **kubelet** on that node is **constantly watching the API server**. It spots a pod assigned to it and takes over. (The scheduler doesn't "send" it directly; kubelet picks it up by watching.)
5. **kubelet talks to the runtime via CRI**: "here's the spec, make it real."
6. **Runtime** (containerd / CRI-O) does the prep: pulls the image, sets up filesystem layers, namespaces, cgroups, networking.
7. **runc**, the lowest level, makes the actual system calls to start the process.
8. **Container is live.** kubelet then monitors it, runs health checks, restarts it if it crashes, and reports status back.

### The key mental shift
**A container is just a Linux process with boundaries.** Same kernel as the host. No VM. Isolation comes from namespaces (what it can see) and cgroups (what it can use).

> **Analogy:** Ordering food through an app. You place the order (kubectl). The app's system logs it (API server + etcd). Dispatch picks a restaurant (scheduler). The restaurant sees it come in (kubelet). The kitchen system passes it to a chef (CRI -> runtime). The chef preps ingredients (image, namespaces, cgroups). The hob actually cooks it (runc -> kernel).

### Why this chain helps with debugging
When something breaks, ask which link failed:
- **Scheduling problem?** Pod stuck, no node assigned (e.g. not enough resources).
- **kubelet problem?** Node assigned but nothing happening.
- **Runtime problem?** Image won't pull, container won't create.
- **Application problem?** Container starts then crashes.

---

## 11. Kubernetes Namespaces

**A virtual partition inside one cluster.** Same physical cluster, logically separated.

Example: one cluster with namespaces `default`, `dev`, `qa`. You can have an app called `app1` in both `dev` and `qa`, and there's **no conflict** because they live in different namespaces.

> **Analogy:** Departments in one office building. Same building, same power and plumbing, but HR and Finance each have their own floor, their own locked doors, and their own budget. Two people called "Sarah" can exist in different departments without confusion.

### Why use them
- **Isolation:** Dev can't accidentally wipe production pods if they're in separate namespaces.
- **Resource quotas:** Cap each namespace. E.g. dev = 4 CPUs, qa = 4 CPUs, prod = 16 CPUs. Stops one team eating the whole cluster.
- **Access control (RBAC):** RBAC can be scoped per namespace. E.g. dev team gets full access to `dev`, read-only on `prod`.

### What lives where

| Namespaced (live inside a namespace) | Cluster-wide (not namespaced) |
|---|---|
| Pods | Nodes |
| Services | PersistentVolumes |
| Deployments | ClusterRoles |
| ConfigMaps | |
| Secrets | |

- If you don't specify a namespace, everything goes into **`default`**. Fine for learning, **don't do it at work or in production.**

### Commands
```bash
# List pods in a specific namespace
kubectl get pods -n dev

# Set your current context to always use a namespace
kubectl config set-context --current --namespace=dev
```

**Bottom line:** namespaces are your first layer of organisation. Split by environment, team or application.

---

## 12. Local Cluster with kind (Hands-on Setup)
 
**kind = Kubernetes IN Docker.** It runs a real Kubernetes cluster on your own machine, where **each node is a Docker container**. Good for learning and testing, not for production.
 
### Prerequisites
- **Docker** running and reachable from WSL (`docker ps` should work). kind needs it because every node is a container.
- **kubectl** installed. kind builds the cluster, kubectl is how you talk to it.

### Installing kind on WSL
 
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```
 
### The cluster config file
```bash
nano kind-config.yaml
```
 
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```
 
| Line | Meaning |
|---|---|
| `kind: Cluster` | What this file describes. Every K8s YAML has a `kind` (later: Pod, Deployment). |
| `apiVersion` | Which version of the config format, so the tool knows how to read it. |
| `nodes:` | The machines in the cluster. |
| `role: control-plane` | The brain: API server, etcd, scheduler, controller manager. |
| `role: worker` (x2) | The machines that actually run your pods. |
 
Check it saved before creating the cluster:
```bash
ls                     # file should be listed
cat kind-config.yaml   # prints the contents
```
 
YAML cares about spacing. Use spaces, not tabs.
 
**Why bother with a config file?** Plain `kind create cluster` gives you **one node doing everything**. The config gives you **one control plane + two workers**, which matches real architecture:
- The scheduler actually has a choice of where to put pods.
- You can stop a worker and watch pods move to the other one (self-healing across machines).
- Apps run on workers, not on the brain, like in production.
- The file is reproducible: delete and recreate the same cluster any time. Your first taste of **declarative config**.
### Create and check the cluster
```bash
kind create cluster --config kind-config.yaml --name k8s-demo
kubectl cluster-info --context kind-k8s-demo
kubectl get nodes
docker ps
```
 
- kind prefixes the name with `kind-`, so the **context** is `kind-k8s-demo`. A context tells kubectl which cluster you're talking to.
- `kubectl get nodes` should show `k8s-demo-control-plane`, `k8s-demo-worker`, `k8s-demo-worker2`, all `Ready` (may say `NotReady` for the first ~30 seconds).
- `docker ps` shows three containers, one per node. Proof that a node is just a Linux machine, here running as a container.
### Run your first pod
```bash
kubectl run nginx --image=nginx     # create a pod called nginx
kubectl get pods -w                 # watch status live (Ctrl+C to stop)
kubectl get pods -o wide            # NODE column = which worker it landed on
kubectl describe pod nginx          # full details + Events (for debugging)
kubectl delete pod nginx            # clean up
```
 
### Create a deployment (video 9c)
```bash
kubectl create deployment nginx-deploy --image=nginx
kubectl get deploy
kubectl get pods
```
- A **deployment** manages pods for you and replaces them if they die. A bare pod from `kubectl run` has nothing looking after it: delete it and it's gone.
- Deployment pods get random suffixes (e.g. `nginx-deploy-7c5ddbdf54-x8k2p`) because the deployment created them.
- There's no `kubectl create pod`. Use `run` for a single pod, `create deployment` for a managed one.
**Key idea:** a pod runs your container. A deployment keeps it running.
 
### Pod statuses to recognise
| Status | Meaning | Usual cause |
|---|---|---|
| `ContainerCreating` | Pulling the image and starting up | Normal, just wait |
| `Running` | Container is up | All good |
| `ErrImagePull` / `ImagePullBackOff` | Can't download the image | Typo in image name, network, private registry |
| `CrashLoopBackOff` | Starts, crashes, restarts, repeat | App or command problem |
 
**"BackOff"** means Kubernetes keeps retrying, but waits longer between each attempt so it doesn't hammer something broken.
 
**Debugging habit:** `kubectl describe pod <name>`, then read the **Events** section at the bottom.
 
### What happened when you ran the pod (the chain from section 10, for real)
`kubectl run` -> **API server** (saves to **etcd**) -> **scheduler** picks a worker -> **kubelet** on that worker sees it -> **containerd** pulls nginx and starts it -> pod `Running`.
- `-o wide` shows you the scheduler's decision.
- `describe` shows each step as events.
### Useful kind commands
```bash
kind get clusters                    # list your kind clusters
kind delete cluster --name k8s-demo  # delete a cluster
kind delete cluster                  # deletes the default one called "kind"
```
 
- `kind delete cluster` uses the **cluster name** (`k8s-demo`), not the context name (`kind-k8s-demo`). A wrong name gives no error, it just does nothing.
- The demo uses `kx` (kubectx) to see the name. It's an add-on. Use `kind get clusters` or `kubectl config get-contexts` instead.
- Keep the cluster running while working through the module. Rebuild any time with `kind create cluster --config kind-config.yaml --name k8s-demo`.
---


## Quick Revision Cheat Sheet

- **K8s** = Linux + networking + system components, scaled across many machines.
- **Container** = a Linux process with boundaries. Shares the host kernel.
- **Namespaces (Linux)** = what a process can see. **cgroups** = what it can use.
- **VM** = full guest OS each, heavy. **Container** = just app + deps, light.
- **Problem K8s solves:** wasted resources and manual management when running many containers.
- **Control plane:** API server (entry point), etcd (source of truth), scheduler (picks node), controller manager (fixes drift), cloud controller manager (talks to cloud).
- **Worker node:** kubelet (runs and watches pods), kube-proxy (Service routing), pods (wrap containers).
- **Runtime:** K8s talks to it via CRI. containerd is the default. Docker shim removed in 1.24.
- **Start-up chain:** kubectl -> API server -> etcd -> scheduler -> kubelet -> CRI -> runtime -> runc -> kernel.
- **K8s namespaces:** virtual partitions for isolation, quotas and RBAC. Don't dump everything in `default`.