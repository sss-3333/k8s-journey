# Notes

My Kubernetes study notes.

## Contents

| # | Notes | Topics |
|---|---|---|
| 01 | [Basics & Architecture](./Basics_and_architecture.md) | Containers vs VMs, control plane and worker nodes, container runtimes, how a pod starts, namespaces, local cluster with kind |
| 02 | [Pods](./pods.md) | Pod YAML, declarative vs imperative, pod phases, CrashLoopBackOff |
| 03 | [Multi-Container Pods](./multi-container-pods.md) | Init containers, sidecars, ambassadors, adapters, ephemeral debug containers |
| 04 | [Deployments](./deployments.md) | Deployments, ReplicaSets, rolling updates, Recreate, rollbacks |
| 05 | [Probes](./probes.md) | Liveness, readiness and startup probes |
| 06 | [Resources & QoS](./resources&qos.md) | Requests and limits, QoS classes, LimitRanges, ResourceQuotas |
| 07 | [Labels, Selectors & Annotations](./labels-selectors-annotations.md) | Labels, equality and set-based selectors, annotations |
| 08 | [Services](./services.md) | ClusterIP, NodePort, LoadBalancer, endpoints, testing services |
| 09 | [Headless & ExternalName Services](./headless&externalNameServices.md) | Headless services, StatefulSet DNS, ExternalName |
| 10 | [Networking Model & Service Mesh](./modelServiceMesh.md) | Flat pod network, CNI, service meshes, Ingress vs mesh |
| 11 | [Storage](./storage.md) | PVs and PVCs, access modes, StorageClasses, dynamic provisioning, ephemeral volumes, CSI |
| 12 | [ConfigMaps](./configmaps.md) | Creating ConfigMaps, env vars vs volume mounts, envFrom, verifying in a pod |