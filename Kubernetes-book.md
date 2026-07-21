# Kubernetes: Complete Mastery Curriculum
### 28 Lessons — Full, Unabridged Reference (Zero to Production Expert)

Every lesson below contains its complete original content: What/Why/Where/How, full ASCII architecture diagrams, complete YAML examples, kubectl commands, best practices, common mistakes, debugging steps, comparison tables, interview questions (beginner/intermediate/advanced), mini quizzes with answers, hands-on labs, and summaries.

---

## Table of Contents

1. [Kubernetes Architecture & Cluster Components](#lesson-1-kubernetes-architecture--cluster-components)
2. [Nodes](#lesson-2-nodes)
3. [Pods](#lesson-3-pods)
4. [Labels, Selectors & Annotations](#lesson-4-labels-selectors--annotations)
5. [ReplicaSets](#lesson-5-replicasets)
6. [Deployments](#lesson-6-deployments)
7. [StatefulSets](#lesson-7-statefulsets)
8. [DaemonSets](#lesson-8-daemonsets)
9. [Jobs & CronJobs](#lesson-9-jobs--cronjobs)
10. [Services](#lesson-10-services)
11. [Ingress](#lesson-11-ingress)
12. [Gateway API](#lesson-12-gateway-api)
13. [Namespaces, Resource Quotas & Limit Ranges](#lesson-13-namespaces-resource-quotas--limit-ranges)
14. [ConfigMaps & Secrets](#lesson-14-configmaps--secrets)
15. [Volumes, PV, PVC & Storage Classes](#lesson-15-volumes-pv-pvc--storage-classes)
16. [CSI Drivers](#lesson-16-csi-drivers-container-storage-interface)
17. [Networking, DNS & CNI](#lesson-17-networking-dns--cni)
18. [kube-proxy](#lesson-18-kube-proxy)
19. [RBAC, Service Accounts, Authentication & Authorization](#lesson-19-rbac-service-accounts-authentication--authorization)
20. [NetworkPolicies](#lesson-20-networkpolicies)
21. [Taints, Tolerations, Node & Pod Affinity](#lesson-21-taints-tolerations-node-affinity--pod-affinityanti-affinity)
22. [HPA, VPA & Cluster Autoscaler](#lesson-22-horizontal-pod-autoscaler-hpa-vertical-pod-autoscaler-vpa--cluster-autoscaler)
23. [Probes & Init/Sidecar Containers](#lesson-23-probes-livenessreadinessstartup--initsidecar-containers-in-depth)
24. [Helm & Kustomize](#lesson-24-helm--kustomize)
25. [Operators & CRDs](#lesson-25-operators--crds-custom-resource-definitions)
26. [Admission Controllers](#lesson-26-admission-controllers)
27. [Monitoring & Logging](#lesson-27-monitoring--logging-prometheus-grafana-metrics-server-efkloki)
28. [Production Best Practices](#lesson-28-production-best-practices--high-availability-disaster-recovery-security-hardening--upgrade-strategies)

---


# Lesson 1: Kubernetes Architecture & Cluster Components

## 1. What is it?

Kubernetes (K8s) is a **container orchestration platform** — it automates deploying, scaling, healing, and managing containerized applications across a cluster of machines.

**Problem it solves:** Running one container with `docker run` is easy. Running 200 containers across 50 servers, keeping them healthy, restarting failures, distributing load, rolling out updates without downtime, and scaling based on traffic — that's a job no human (or shell script) can do reliably. Kubernetes is the "operating system for your cluster" that handles this.

## 2. Why do we need it?

**Without it**, teams hand-roll: custom health-check scripts, manual load balancer config updates, SSH-based deployments, ad-hoc scaling scripts, and inconsistent environments between dev/staging/prod. This is fragile and doesn't scale past a handful of servers.

**Industry motivation:** Google ran billions of containers/week internally (via Borg) before open-sourcing the ideas as Kubernetes in 2014. Every major cloud now offers managed Kubernetes because it's become the de facto standard for cloud-native deployment.

## 3. Where is it used?

- **Enterprise microservices platforms** — hundreds of services, each independently deployable
- **Managed offerings:** AWS EKS, Azure AKS, Google GKE, Red Hat OpenShift, on-prem via kubeadm/Rancher
- **CI/CD pipelines** deploying to ephemeral preview environments
- **ML/data platforms** (Kubeflow, Spark-on-K8s) for scheduling GPU workloads

## 4. How does it work internally? (Architecture)

A cluster has two types of nodes: **Control Plane** (the brain) and **Worker Nodes** (where your app actually runs).

```
                         +-------------------------------------------+
                         |            CONTROL PLANE                   |
                         |                                             |
   kubectl apply  ------>|  +--------------+                          |
                         |  | kube-apiserver|<------ All components   |
                         |  +------+-------+        talk ONLY to API  |
                         |         |                server (never     |
                         |         v                to each other)    |
                         |  +--------------+                          |
                         |  |     etcd     |  (key-value store,       |
                         |  | (cluster DB) |   the single source      |
                         |  +--------------+   of truth)              |
                         |         ^                                  |
                         |         |                                  |
                         |  +------+-------+   +--------------------+ |
                         |  |  scheduler   |   | controller manager | |
                         |  | (assigns pod |   | (reconciliation    | |
                         |  |  to a node)  |   |  loops: Deployment, | |
                         |  +--------------+   |  Node, Job, etc.)   | |
                         |                     +--------------------+ |
                         +---------------------------------------------+
                                       |
                         +-------------+-------------+
                         v             v             v
                 +-----------+ +-----------+ +-----------+
                 | WORKER    | | WORKER    | | WORKER    |
                 |  NODE 1   | |  NODE 2   | |  NODE 3   |
                 |           | |           | |           |
                 | kubelet   | | kubelet   | | kubelet   |
                 | kube-proxy| | kube-proxy| | kube-proxy|
                 | container | | container | | container |
                 | runtime   | | runtime   | | runtime   |
                 | (containerd)| |(containerd)| |(containerd)|
                 |           | |           | |           |
                 |  [Pods]   | |  [Pods]   | |  [Pods]   |
                 +-----------+ +-----------+ +-----------+
```

**Component responsibilities:**

| Component | Role |
|---|---|
| **kube-apiserver** | Front door to the cluster. Validates and processes all REST requests (from `kubectl`, controllers, kubelets). The *only* component that talks to etcd directly. |
| **etcd** | Distributed, consistent key-value store. Holds the entire cluster state (every object's desired + observed spec). If etcd is lost, the cluster's "memory" is gone. |
| **kube-scheduler** | Watches for newly created Pods with no assigned node, decides which node to place them on based on resource requests, taints/tolerations, affinity rules. |
| **kube-controller-manager** | Runs control loops (Deployment controller, ReplicaSet controller, Node controller, etc.) that continuously reconcile actual state -> desired state. |
| **kubelet** | Agent on every worker node. Talks to the API server, ensures containers described in PodSpecs are actually running (via the container runtime). |
| **kube-proxy** | Maintains network rules on each node so traffic to a Service gets routed to the correct Pod (via iptables/IPVS). |
| **Container Runtime** | (containerd, CRI-O) Actually pulls images and runs containers per the CRI (Container Runtime Interface). |

**The golden rule:** components never talk to each other directly — everything flows through the **kube-apiserver**, and the **desired state lives in etcd**. Every controller just watches for changes and reconciles.

## 5. What happens from `kubectl apply` to a running Pod?

1. `kubectl apply -f deployment.yaml` -> kubectl sends an HTTP request to the **kube-apiserver**.
2. API server **authenticates**, **authorizes** (RBAC), runs **admission controllers**, validates the YAML schema.
3. API server writes the desired state into **etcd**.
4. **Deployment controller** (in controller-manager) notices a new Deployment -> creates a **ReplicaSet**.
5. **ReplicaSet controller** notices it needs N Pods -> creates Pod objects (unscheduled, no node yet).
6. **kube-scheduler** watches for unscheduled Pods -> picks a node based on resource fit, affinity, taints -> writes the binding back to the API server.
7. The **kubelet** on that node sees a Pod assigned to it -> tells the **container runtime** to pull the image and start containers.
8. Kubelet continuously reports Pod status back to the API server -> written to etcd.
9. **kube-proxy** updates networking rules so the new Pod is reachable via any Service pointing to it.

## 6. Comparison: Control Plane vs Worker Node

| Aspect | Control Plane | Worker Node |
|---|---|---|
| Purpose | Decision-making, cluster state | Running actual workloads |
| Key components | apiserver, etcd, scheduler, controller-manager | kubelet, kube-proxy, container runtime |
| Failure impact | Cluster becomes unmanageable (new work) but running Pods keep running briefly | That node's Pods go down; rescheduled elsewhere |
| Typical count | 3 or 5 (odd, for etcd quorum) in HA setups | Scales with workload, often dozens-thousands |

## 7. Interview Questions

**Beginner**
- What is the role of etcd in a Kubernetes cluster?
- Name the main control plane components.

**Intermediate**
- What happens if the kube-apiserver goes down? (Answer: no new changes/scheduling, but existing Pods keep running since kubelet operates independently short-term.)
- Why does etcd typically run with an odd number of nodes (3, 5)?

**Advanced / Scenario**
- A Pod is stuck in `Pending` state. Walk through your debugging steps across scheduler, node capacity, and taints.
- Your control plane is HA across 3 nodes and one etcd member fails. What's the cluster behavior, and when do you lose quorum?

## 8. Mini Quiz

1. Which component is the *only* one that talks directly to etcd? -> **kube-apiserver**
2. Which component decides *where* a Pod runs? -> **kube-scheduler**
3. True/False: kubelet and kube-scheduler communicate directly. -> **False** (they only talk via the API server)

## Summary

Kubernetes separates **desired state** (declared via YAML, stored in etcd) from **actual state** (constantly reconciled by controllers and kubelet). The control plane decides *what should happen*; worker nodes *make it happen*. Every single interaction — from `kubectl` to internal controllers — flows through the API server.


# Lesson 2: Nodes

## 1. What is it?

A **Node** is a worker machine in the Kubernetes cluster — physical or virtual — where your actual application containers run inside Pods. Every node runs the three worker-side components from Lesson 1: **kubelet**, **kube-proxy**, and a **container runtime**.

**Purpose:** Nodes are the compute capacity of the cluster. Kubernetes abstracts away "which specific machine" your app runs on — you just declare what you need, and the scheduler picks a node.

## 2. Why do we need it?

Without the Node abstraction, you'd manually decide which physical/virtual machine runs which container — exactly the toil Kubernetes exists to remove. Nodes let you add/remove capacity (autoscaling, hardware failure replacement) without touching application definitions at all.

**Industry motivation:** A fleet of nodes lets you scale horizontally — need more capacity? Add nodes. A node dies? Pods reschedule elsewhere automatically (self-healing).

## 3. Where is it used?

- **Cloud managed node groups:** EKS Managed Node Groups, AKS Node Pools, GKE Node Pools — auto-provisioned VMs joined to the cluster
- **On-prem bare metal:** physical servers joined via `kubeadm join`
- **Mixed fleets:** GPU node pools for ML workloads, spot/preemptible node pools for cost savings, separate pools for different workload types (tainted for isolation)

## 4. How does it work internally?

```
+--------------------------- NODE ---------------------------+
|                                                               |
|  +------------+        registers with          +----------+|
|  |  kubelet   |--------- API server ----------->| apiserver||
|  |            |<-------- watches for ------------| (control ||
|  |            |          assigned Pods           |  plane)  ||
|  +-----+------+                                  +----------+|
|        | CRI (Container Runtime Interface)                   |
|        v                                                     |
|  +------------+                                              |
|  | containerd |--> pulls images, starts/stops containers    |
|  | (runtime)  |                                              |
|  +------------+                                              |
|                                                               |
|  +------------+                                              |
|  | kube-proxy |--> maintains iptables/IPVS rules for Service |
|  +------------+    routing to Pods on this node              |
|                                                               |
|  Node status reported: CPU/Mem capacity, conditions           |
|  (Ready, MemoryPressure, DiskPressure, PIDPressure,           |
|   NetworkUnavailable), allocatable resources                  |
+-----------------------------------------------------------------+
```

**Node lifecycle:**

1. Node boots -> kubelet starts -> kubelet registers itself with the API server (creates a Node object)
2. kubelet sends periodic **heartbeats** (via NodeStatus updates, roughly every 10s, lease-based since v1.13+)
3. If the control plane doesn't hear from a node for `node-monitor-grace-period` (default 40s), it's marked `NotReady`
4. After `pod-eviction-timeout` (default 5 min), Pods on that node are evicted and rescheduled elsewhere
5. kubelet continuously watches the API server for Pods assigned to it and runs them via the container runtime

**Node Conditions** (visible via `kubectl describe node`):

- `Ready` — node can accept Pods
- `MemoryPressure` — node is low on memory
- `DiskPressure` — node is low on disk
- `PIDPressure` — too many processes
- `NetworkUnavailable` — network not correctly configured

## 5. How do we create and manage it?

You rarely create Node objects directly by hand — they're auto-registered when kubelet starts. But you interact with them constantly:

```bash
# List all nodes with status
kubectl get nodes

# Detailed info: capacity, conditions, running pods, taints
kubectl describe node <node-name>

# See resource usage (requires metrics-server)
kubectl top node

# Cordon a node (mark unschedulable - for maintenance)
kubectl cordon <node-name>

# Drain a node (evict all pods gracefully, then it's safe to remove)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon after maintenance
kubectl uncordon <node-name>

# Add a label to a node (used for nodeSelector/affinity)
kubectl label node <node-name> disktype=ssd

# Add a taint (repel Pods unless they tolerate it)
kubectl taint node <node-name> key=value:NoSchedule
```

**A minimal Node YAML** (for reference — you'd rarely write this manually since kubelet self-registers it):

```yaml
apiVersion: v1
kind: Node
metadata:
  name: worker-node-1
  labels:
    kubernetes.io/hostname: worker-node-1
    node-role.kubernetes.io/worker: ""
spec:
  taints: []          # no taints = schedulable by any pod
status:
  capacity:
    cpu: "4"
    memory: 16Gi
  allocatable:
    cpu: "3800m"       # slightly less than capacity - reserved for system daemons
    memory: 15Gi
```

**Best practices:**
- Always `cordon` + `drain` before decommissioning a node — never just power it off
- Set resource reservations (`--system-reserved`, `--kube-reserved`) so system processes aren't starved
- Use node pools/groups with taints to separate workload types (e.g., GPU nodes tainted so only GPU-requesting Pods land there)

**Common mistakes:**
- Powering off a node without draining first -> Pods die ungracefully, no clean termination
- Forgetting `--ignore-daemonsets` on drain -> drain hangs forever (DaemonSet pods aren't meant to be evicted this way)
- Not setting resource reservations -> node runs out of memory and the *kubelet itself* gets OOM-killed alongside your app

**Debugging a `NotReady` node:**
```bash
kubectl describe node <node-name>     # check Conditions section for the reason
kubectl get events --field-selector involvedObject.name=<node-name>
# SSH into node and check:
systemctl status kubelet
journalctl -u kubelet -f
```

## 6. When should we use manual node management vs. Cluster Autoscaler?

Covered in depth later in the autoscaling lesson — short version: manual node management fits small/static clusters; Cluster Autoscaler fits variable-load production environments.

## 7. Comparison: Master (Control Plane) Node vs Worker Node

| Aspect | Control Plane Node | Worker Node |
|---|---|---|
| Runs user workloads? | Typically **no** (tainted `NoSchedule` by default) | Yes — this is its purpose |
| Components | apiserver, etcd, scheduler, controller-manager | kubelet, kube-proxy, runtime |
| Count in HA | Odd number (3, 5) for etcd quorum | Scales freely, even hundreds |

## 8. Interview Questions

**Beginner**
- What's the difference between Node capacity and allocatable resources?
- What command safely removes a node from scheduling for maintenance?

**Intermediate**
- What happens to Pods on a node that goes `NotReady`? How long before they're rescheduled?
- Why might `kubectl drain` hang indefinitely?

**Advanced / Scenario-based**
- A node shows `Ready` but Pods scheduled to it are stuck in `ContainerCreating`. Where do you look? (Answer path: kubelet logs -> container runtime status -> image pull errors -> CNI plugin issues -> disk pressure)
- You have a node with a hardware GPU. How do you ensure only GPU-workload Pods get scheduled there, and normal Pods don't consume the GPU node's other resources unnecessarily? (Answer: taint the node `nvidia.com/gpu=true:NoSchedule` + Pods must add matching toleration + resource request for `nvidia.com/gpu`)

## 9. Mini Quiz

1. What triggers a Node to be marked `NotReady`? -> Missing heartbeats beyond `node-monitor-grace-period` (default 40s)
2. Command to safely evict all Pods before maintenance? -> `kubectl drain <node> --ignore-daemonsets`
3. True/False: You must manually create a Node object before a machine can join the cluster. -> **False** — kubelet self-registers on startup

## Hands-on Lab

```bash
kubectl get nodes -o wide
kubectl describe node <your-node-name>
kubectl label node <your-node-name> env=lab
kubectl get nodes --show-labels
kubectl cordon <your-node-name>
kubectl get nodes    # observe SchedulingDisabled status
kubectl uncordon <your-node-name>
```

**Expected output:** After cordon, `STATUS` column shows `Ready,SchedulingDisabled`. After uncordon, back to `Ready`.

## Summary

A Node is the unit of compute in Kubernetes — it self-registers via kubelet, reports health via heartbeats, and hosts Pods assigned by the scheduler. Managing nodes well (labels, taints, draining) is core to safe cluster operations.


# Lesson 3: Pods

## 1. What is it?

A **Pod** is the smallest deployable unit in Kubernetes — a wrapper around one or more containers that share the same **network namespace** (same IP, same port space) and can share **storage volumes**.

**Definition:** Not "one container = one Pod" necessarily — a Pod can hold multiple tightly-coupled containers that must run together, always on the same node.

**Problem it solves:** Kubernetes doesn't manage raw containers directly. It needs an abstraction that groups containers which are logically "one unit" (e.g., an app + a helper that must share network/storage), and that abstraction is the Pod.

## 2. Why do we need it?

**Without Pods**, you'd manage individual containers with no built-in concept of "these two containers belong together, share localhost, and should live/die as one." You'd need external glue to co-locate a logging sidecar with your app container, for instance.

**Industry motivation:** Real-world apps often need helper processes — a log shipper, a proxy (Envoy/Istio sidecar), a config-reloader — that must share the same network/filesystem as the main app. Pods make this a first-class concept instead of a hack.

## 3. Where is it used?

- Every single workload in Kubernetes ultimately runs as a Pod — Deployments, StatefulSets, DaemonSets, Jobs all just create Pods under the hood
- **Service mesh sidecars** (Istio/Linkerd inject a proxy container into every app Pod)
- **Log aggregation sidecars** (Fluent Bit container alongside app container)

## 4. How does it work internally?

```
+----------------------- POD (shared network namespace) -----------------------+
|  Pod IP: 10.244.1.7                                                          |
|                                                                               |
|  +------------------+   +------------------+   +------------------+        |
|  |  Init Container 1 |   |  Main Container   |   | Sidecar Container |        |
|  |  (runs & exits    |-->|  (app, e.g.       |<->|  (e.g. log shipper)|        |
|  |   before others   |   |   nginx:80)       |   |  localhost:9090   |        |
|  |   start)          |   |  localhost:80     |   |                    |        |
|  +------------------+   +------------------+   +------------------+        |
|                                   |                        |                  |
|                          +--------+------------------------+                  |
|                          v                                                    |
|                  Shared Volume (emptyDir, ConfigMap, etc.)                    |
|                                                                               |
|  All containers: same IP, communicate via localhost, share volumes           |
+-------------------------------------------------------------------------------+
```

**Key internal facts:**
- Every Pod gets exactly **one IP address** (from the CNI plugin) — shared by all containers inside it
- Containers within a Pod can reach each other via `localhost:<port>` — no networking needed
- **Init containers** run sequentially, to completion, before any main containers start (used for setup tasks like waiting for a dependency, or seeding config)
- The **pause container** (a hidden infrastructure container) actually holds the network namespace open — your containers attach to it
- Pods are **ephemeral and immutable by design** — you don't "update" a running Pod's image; you replace it with a new Pod (this is why Deployments exist)

**Pod lifecycle phases:**
```
Pending -> Running -> Succeeded / Failed
             |
             +-- (if container crashes) -> CrashLoopBackOff (via restart policy)
```

- `Pending` — Pod accepted but containers not yet running (image pulling, scheduling)
- `Running` — bound to a node, at least one container running
- `Succeeded` — all containers terminated successfully (exit code 0) — typical for Jobs
- `Failed` — at least one container terminated with failure
- `Unknown` — node unreachable, kubelet can't report status

## 5. How do we create and manage it?

**Simple single-container Pod YAML — line by line:**
```yaml
apiVersion: v1              # API version for this resource type
kind: Pod                   # This object is a Pod
metadata:
  name: my-app-pod          # Unique name within the namespace
  labels:                   # Key-value tags for selection (used by Services, etc.)
    app: my-app
    tier: backend
spec:
  containers:
  - name: app-container     # Container name within the pod
    image: nginx:1.25       # Image to pull and run
    ports:
    - containerPort: 80     # Port the container listens on (informational)
    resources:
      requests:              # Minimum guaranteed resources (used for scheduling)
        cpu: "250m"          # 250 millicores = 0.25 CPU
        memory: "128Mi"
      limits:                # Hard ceiling - container is throttled/killed if exceeded
        cpu: "500m"
        memory: "256Mi"
    env:
    - name: ENV_MODE
      value: "production"
  restartPolicy: Always      # Always | OnFailure | Never - what to do when container exits
```

**Multi-container Pod with init container:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
    # This runs to completion BEFORE main containers start
  containers:
  - name: main-app
    image: myapp:1.0
    ports:
    - containerPort: 8080
  - name: log-sidecar
    image: fluent-bit:2.1
    # Shares network + can mount the same volume as main-app
```

**Essential kubectl commands:**
```bash
kubectl run my-pod --image=nginx --port=80        # quick imperative pod creation
kubectl apply -f pod.yaml                          # declarative creation
kubectl get pods                                   # list pods
kubectl get pods -o wide                           # + node, IP info
kubectl describe pod my-app-pod                    # events, conditions, container statuses
kubectl logs my-app-pod                             # logs from single-container pod
kubectl logs my-app-pod -c log-sidecar               # logs from specific container
kubectl logs my-app-pod --previous                    # logs from CRASHED previous instance
kubectl exec -it my-app-pod -- /bin/sh                # shell into a running pod
kubectl delete pod my-app-pod                          # delete (triggers graceful termination)
kubectl port-forward pod/my-app-pod 8080:80             # tunnel local port to pod port
```

**Best practices:**
- Never deploy bare Pods directly in production — always use a Deployment/StatefulSet/DaemonSet wrapper (bare Pods aren't self-healing; if the node dies, the Pod is gone forever)
- Always set `resources.requests` and `limits` — unbounded Pods can starve the node
- Use `restartPolicy: Never` or `OnFailure` for Jobs, `Always` for long-running services

**Common mistakes:**
- Deploying naked Pods -> no auto-restart if the node fails
- Forgetting resource limits -> one Pod can consume all node memory, causing `OOMKilled` cascades for neighbors
- Assuming Pod IP is stable -> Pod IPs change every time a Pod is recreated; never hardcode them (use Services instead)

**Debugging:**
```bash
kubectl describe pod <name>       # check Events section FIRST - most issues show here
kubectl get pod <name> -o yaml    # full spec/status dump
kubectl logs <name> --previous    # if CrashLoopBackOff, see why it crashed last time
```

Common states you'll debug:

| State | Typical Cause |
|---|---|
| `Pending` | Insufficient node resources, unschedulable (taints/affinity), PVC not bound |
| `ImagePullBackOff` | Wrong image name, private registry auth missing |
| `CrashLoopBackOff` | App crashes on startup - check `logs --previous` |
| `OOMKilled` | Container exceeded memory limit |
| `ContainerCreating` (stuck) | Volume mount issue, CNI network plugin problem |

**Cleanup:**
```bash
kubectl delete pod my-app-pod --grace-period=30   # graceful termination window
kubectl delete pod my-app-pod --force --grace-period=0   # force delete (use cautiously)
```

## 6. When should we use it?

Directly authoring bare Pods is appropriate almost **only** for: quick debugging/testing (`kubectl run`), one-off diagnostic pods, or as the underlying object created by higher-level controllers. You conceptually "use" Pods everywhere, but you rarely **author** them directly.

## 7. When should we NOT use it (bare Pods)?

- **Never for production workloads directly** — no self-healing, no rolling updates, no replica management
- Use a **Deployment** instead for stateless apps needing self-healing and scaling
- Use a **StatefulSet** for anything needing stable identity/storage (databases)
- Use a **Job/CronJob** for run-to-completion tasks

## 8. Comparison Table: Pod vs Container

| Aspect | Container | Pod |
|---|---|---|
| Definition | Single process + its dependencies | Wrapper for one or more containers |
| Networking | Has its own namespace normally | Shares one IP across all its containers |
| Kubernetes schedules | Never schedules bare containers | Schedules Pods (as a whole unit) |
| Restart granularity | N/A | Whole Pod is the recreate unit typically |

## 9. Interview Questions

**Beginner**
- Why does Kubernetes use Pods instead of scheduling containers directly?
- What's the difference between `restartPolicy: Always` and `OnFailure`?

**Intermediate**
- What's the purpose of an init container? Give a real example.
- How do containers within the same Pod communicate?

**Advanced / Scenario**
- A Pod is stuck in `CrashLoopBackOff`. Walk through your full debugging process.
- You need a Pod that mounts a shared config file both containers use, but only one container should ever write to it. How would you structure volumes and permissions?
- Explain exactly why deploying bare Pods in production is an anti-pattern, with a concrete failure scenario.

## 10. Mini Quiz

1. How many IP addresses does a multi-container Pod get? -> **One**, shared by all containers
2. What phase does a Pod reach when a Job's container exits with code 0? -> **Succeeded**
3. True/False: If a node crashes, a bare Pod (no controller) will automatically be rescheduled elsewhere. -> **False**

## Hands-on Lab

```bash
kubectl run test-pod --image=nginx:1.25 --port=80
kubectl get pods
kubectl describe pod test-pod
kubectl logs test-pod
kubectl exec -it test-pod -- curl localhost:80
kubectl delete pod test-pod
```

**Expected:** Pod reaches `Running`, curl returns nginx welcome HTML, delete removes it (and it does NOT come back — proving the "no self-healing without a controller" point).

## Summary

The Pod is Kubernetes' atomic unit — one IP, shared storage, one scheduling decision. Understanding Pods deeply (lifecycle, init containers, debugging states) is foundational because *every* higher-level object (Deployment, StatefulSet, Job) is ultimately just a Pod factory with extra behavior layered on top.


# Lesson 4: Labels, Selectors & Annotations

## 1. What is it?

**Labels** — key-value pairs attached to objects (Pods, Nodes, Services, etc.) used to **identify and group** resources semantically (e.g., `app: frontend`, `env: production`, `version: v2`).

**Selectors** — queries that **filter objects by their labels** (e.g., "give me all Pods where `app=frontend`"). This is how Services find Pods, how Deployments know which Pods they own, how `kubectl get pods -l app=frontend` works.

**Annotations** — key-value pairs attached to objects, but **not used for selection** — purely for storing arbitrary metadata (build info, git commit SHA, tooling config, contact info) that tools or humans read but Kubernetes itself doesn't query against.

**Problem they solve:** Kubernetes needs a flexible, non-hierarchical way to group and query arbitrary objects without hardcoding relationships. Labels give you SQL-like `WHERE` filtering over your whole cluster.

## 2. Why do we need them?

**Without labels**, a Deployment would have no generic way to say "these are MY Pods" — you'd need hardcoded names or IDs, which breaks the moment you scale or replace Pods. Labels decouple **identity** from **relationships**: any object can dynamically discover related objects purely by matching key-value pairs.

**Real-world motivation:**
- A Service needs to route traffic to "all healthy Pods of this app" — regardless of how many exist or their names -> **label selector**
- You want to `kubectl get pods -l env=staging,tier=backend` during an incident -> instant filtering
- CI/CD wants to record which git commit built an image, without affecting scheduling/selection -> **annotation**

## 3. Where is it used?

- **Services** select backend Pods via label selectors (`selector: app: my-app`)
- **Deployments/ReplicaSets** track "which Pods do I own" via labels + `matchLabels`
- **NetworkPolicies** select which Pods a policy applies to
- **Node affinity/anti-affinity** matches Pods to Nodes via labels
- **Enterprise tooling:** Prometheus scrapes Pods based on annotations (`prometheus.io/scrape: "true"`); Istio injects sidecars based on namespace labels; ArgoCD/Helm track ownership via labels

## 4. How does it work internally?

```
+--------------- Labels are just metadata stored with the object in etcd ---------------+
|                                                                                          |
|   Pod A                    Pod B                    Pod C                              |
|   labels:                  labels:                  labels:                            |
|     app: frontend            app: frontend             app: backend                     |
|     env: prod                 env: staging              env: prod                       |
|     version: v2                version: v1               version: v1                    |
|                                                                                          |
|   +-----------------------------------------------------------------+                   |
|   |  Service selector: { app: frontend, env: prod }                  |                   |
|   |  -> API server queries etcd for Pods matching BOTH labels        |                   |
|   |  -> Matches: Pod A only                                          |                   |
|   +-----------------------------------------------------------------+                   |
+------------------------------------------------------------------------------------------+
```

**Mechanically:** the API server indexes objects by their labels, and any controller (Service endpoint controller, ReplicaSet controller, etc.) issues a **label query** (`LIST` with a `labelSelector` query param) against the API server to find matching objects. This is a **live, continuous match** — if you relabel a Pod so it no longer matches a Service's selector, it's instantly removed from that Service's endpoints (no restart needed).

**Two types of selectors:**
1. **Equality-based:** `app=frontend`, `env!=staging`
2. **Set-based:** `environment in (production, staging)`, `tier notin (frontend)`, `partition` (key exists, any value)

## 5. How do we create and manage them?

**Labels on a Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend          # what application this belongs to
    tier: web               # architectural layer
    env: production          # environment
    version: v2.1.0          # for canary/version-based routing
  annotations:
    kubernetes.io/change-cause: "Deployed by CI pipeline run #482"
    git.commit-sha: "a1b2c3d4"
    prometheus.io/scrape: "true"     # read by Prometheus tooling, not by K8s core
    prometheus.io/port: "9090"
spec:
  containers:
  - name: web
    image: frontend:v2.1.0
```

**Service using an equality-based selector:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend       # matches ANY pod with label app=frontend
    env: production      # AND env=production (implicit AND across keys)
  ports:
  - port: 80
    targetPort: 8080
```

**Deployment using set-based `matchLabels`/`matchExpressions`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
spec:
  selector:
    matchExpressions:
    - key: app
      operator: In
      values: ["frontend"]
    - key: tier
      operator: NotIn
      values: ["experimental"]
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: web
        image: frontend:v2.1.0
```

**Essential kubectl commands:**
```bash
# Add a label
kubectl label pod frontend-pod release=stable

# Overwrite an existing label
kubectl label pod frontend-pod env=staging --overwrite

# Remove a label
kubectl label pod frontend-pod env-

# Filter by label (equality)
kubectl get pods -l app=frontend

# Filter by label (set-based)
kubectl get pods -l 'env in (staging,production)'
kubectl get pods -l 'tier notin (experimental)'
kubectl get pods -l 'app'          # key exists, any value

# Multiple conditions (AND)
kubectl get pods -l app=frontend,env=production

# Show labels in output
kubectl get pods --show-labels

# Add/edit an annotation
kubectl annotate pod frontend-pod git.commit-sha=e5f6g7h
```

**Best practices:**
- Use **recommended standard labels** (from Kubernetes docs) for consistency across tools:
  ```yaml
  app.kubernetes.io/name: frontend
  app.kubernetes.io/instance: frontend-abc123
  app.kubernetes.io/version: "2.1.0"
  app.kubernetes.io/component: web
  app.kubernetes.io/part-of: ecommerce-platform
  app.kubernetes.io/managed-by: helm
  ```
- Keep labels **short and queryable** — they're for selection, not storage of large data
- Put large/structured metadata (build logs URLs, JSON blobs, config) in **annotations**, since labels have stricter size/format constraints (label values: max 63 chars, alphanumeric + `-_.`)
- Never use labels for information that changes extremely frequently (e.g., a live CPU percentage) — that's what metrics/status fields are for

**Common mistakes:**
- Using an annotation where a selector is needed -> Service silently matches zero Pods (annotations are invisible to selectors)
- Deployment `selector.matchLabels` not matching the Pod template's labels -> **immutable field error** on apply; this is a very common early mistake
- Overly broad selectors (e.g., Service selecting `app: web` when multiple unrelated Deployments share that label) -> traffic routed to wrong Pods

**Debugging a Service with no endpoints:**
```bash
kubectl get endpoints frontend-svc     # empty? selector isn't matching anything
kubectl get pods -l app=frontend,env=production --show-labels   # verify the label actually exists on pods
kubectl describe svc frontend-svc      # confirm the selector itself is what you expect
```

## 6. When should we use which?

| Need | Use |
|---|---|
| Group/query objects (Services, Deployments matching Pods) | **Label** |
| Store build metadata, tooling config, human-readable notes | **Annotation** |
| Filter `kubectl get` output during debugging | **Label** |
| Track a Helm release name, rollback history, webhook config | **Annotation** |

## 7. When should we NOT use labels?

- Don't store non-identifying, large, or frequently-changing data in labels — they're indexed and meant for selection, not payload storage
- Don't rely on labels alone for security boundaries — combine with **RBAC** and **NetworkPolicies**, since labels are just metadata anyone with edit access can change

## 8. Comparison Table

| Aspect | Label | Annotation |
|---|---|---|
| Used in selectors? | **Yes** | No |
| Size limit | 63 chars per value | Much larger (up to 256KB total per object) |
| Purpose | Identification/grouping | Arbitrary metadata |
| Indexed by API server? | Yes | No |
| Example | `app: frontend` | `git.commit-sha: a1b2c3d` |

## 9. Interview Questions

**Beginner**
- What's the difference between a label and an annotation?
- How does a Service find which Pods to send traffic to?

**Intermediate**
- What happens if you change a Pod's labels so it no longer matches its Deployment's selector? (Answer: The Deployment's ReplicaSet no longer "sees" it as owned — it becomes an orphan Pod, and the ReplicaSet spins up a *new* Pod to satisfy its replica count, since it thinks it's short one.)
- Explain the difference between equality-based and set-based selectors with an example each.

**Advanced / Scenario**
- Your Service has zero endpoints but Pods are `Running` and healthy. Walk through your full diagnostic process.
- You want to do a canary rollout where 10% of traffic goes to `version: v2` Pods. How would labels/selectors factor into this design?

## 10. Mini Quiz

1. True/False: Annotations can be used in a Service's `selector` field. -> **False**
2. What happens if a Deployment's `matchLabels` doesn't match its own Pod template labels? -> API rejects it as invalid (or on update, it's an immutable-field error)
3. Command to list Pods with `env=staging` and label key `app.kubernetes.io/instance` set to anything? -> `kubectl get pods -l env=staging,app.kubernetes.io/instance`

## Hands-on Lab

```bash
kubectl run pod-a --image=nginx --labels="app=frontend,env=prod"
kubectl run pod-b --image=nginx --labels="app=frontend,env=staging"
kubectl get pods --show-labels
kubectl get pods -l app=frontend
kubectl get pods -l env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl label pod pod-b env=prod --overwrite
kubectl get pods -l env=prod    # now both pods appear
```

**Expected:** First filter shows both pods (both `app=frontend`); second shows only `pod-a`; after overwrite, both show under `env=prod`.

## Summary

Labels are the **query language** of Kubernetes — every relationship between objects (Service->Pod, Deployment->Pod, NetworkPolicy->Pod) is expressed as a live label match, not a hardcoded reference. Annotations carry metadata that humans/tools need but Kubernetes itself doesn't act on.


# Lesson 5: ReplicaSets

## 1. What is it?

A **ReplicaSet (RS)** is a controller that ensures a specified number of identical Pod replicas are running at all times. If a Pod dies, the ReplicaSet creates a new one; if there are too many, it deletes the excess.

**Purpose:** Solve the gap identified in Lesson 3 — bare Pods have no self-healing. A ReplicaSet is the "babysitter" that watches a set of Pods (via label selector) and reconciles actual count -> desired count.

## 2. Why do we need it?

**Without it**, if a Pod crashes or its node dies, nothing brings it back — your app silently loses capacity. Manually running `kubectl run` again after every failure doesn't scale and isn't automatable.

**Industry motivation:** High availability requires *continuous* reconciliation, not one-time creation. A ReplicaSet's control loop runs forever: watch actual state, compare to desired state (`replicas: 3`), take corrective action.

## 3. Where is it used?

In practice, **you almost never create a ReplicaSet directly** — Deployments create and manage ReplicaSets for you (next lesson). But understanding RS is essential because:
- Every `kubectl get pods` running under a Deployment is actually owned by a ReplicaSet
- Rolling updates work by creating a *new* ReplicaSet and scaling the old one down while scaling the new one up
- Debugging rollout issues requires you to inspect ReplicaSets directly

## 4. How does it work internally?

```
+--------------------- ReplicaSet Controller (control loop) ---------------------+
|                                                                                   |
|   Desired: replicas: 3                                                          |
|   Selector: matchLabels: { app: my-app }                                        |
|                                                                                   |
|   Step 1: LIST Pods matching selector { app: my-app } from API server           |
|   Step 2: Count matching Pods currently existing                                |
|   Step 3: Compare count to desired replicas                                     |
|                                                                                   |
|        if actual < desired  -> CREATE new Pods (from podTemplate)               |
|        if actual > desired  -> DELETE excess Pods                                |
|        if actual == desired -> do nothing (steady state)                        |
|                                                                                   |
|   This loop runs continuously, triggered by watch events (Pod deleted, node down)|
+------------------------------------------------------------------------------------+

        ReplicaSet (my-app-rs)
                |
     +----------+----------+
     v          v          v
   Pod-1      Pod-2      Pod-3      <- all labeled app: my-app, owned via ownerReferences
  (Running)  (Running)  (CRASHED -> deleted, new Pod-4 created automatically)
```

**Key internal mechanism — `ownerReferences`:** Every Pod created by a ReplicaSet has an `ownerReferences` field pointing back to that RS's UID. This is how Kubernetes knows "this Pod belongs to that controller" — and it's also how **garbage collection** works: delete the RS, and (by default) all owned Pods are cascade-deleted too.

**Important subtlety:** the ReplicaSet's selector must match its own Pod template's labels — but it can also **adopt** pre-existing Pods that happen to match its selector but weren't created by it.

## 5. How do we create and manage it?

**ReplicaSet YAML — line by line:**
```yaml
apiVersion: apps/v1              # ReplicaSet lives in the apps/v1 API group
kind: ReplicaSet
metadata:
  name: my-app-rs
  labels:
    app: my-app
spec:
  replicas: 3                     # desired number of Pod replicas
  selector:
    matchLabels:
      app: my-app                 # MUST match template.metadata.labels below
  template:                       # Pod template - blueprint for Pods this RS creates
    metadata:
      labels:
        app: my-app               # must satisfy the selector above
    spec:
      containers:
      - name: app-container
        image: my-app:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "400m"
            memory: "512Mi"
```

**Essential kubectl commands:**
```bash
kubectl apply -f replicaset.yaml
kubectl get rs                                # list ReplicaSets
kubectl describe rs my-app-rs                 # see events, replica status
kubectl get pods -l app=my-app                # see Pods it owns
kubectl scale rs my-app-rs --replicas=5       # manually scale
kubectl delete rs my-app-rs                   # deletes RS AND its Pods (cascade)
kubectl delete rs my-app-rs --cascade=orphan  # deletes RS but LEAVES Pods running (orphaned)
```

**Best practices:**
- Don't manage ReplicaSets directly in production — use a Deployment (it manages RS lifecycle for you, enabling rollbacks and rolling updates)
- Always let the RS selector be as specific as needed to avoid accidentally adopting unrelated Pods

**Common mistakes:**
- Manually creating Pods with labels that happen to match an existing RS's selector -> the RS "adopts" them, then may delete them later if it's already at capacity (surprising behavior!)
- Editing a running Pod's spec directly instead of updating the RS template — the RS doesn't retroactively update existing Pods; only *new* Pods get the new template
- Forgetting that scaling down doesn't let you pick *which* Pod dies — RS chooses (generally: not-ready Pods first, then newest Pods)

**Debugging:**
```bash
kubectl describe rs my-app-rs
# Look at "Replicas: 3 current / 3 desired" and Events section
kubectl get events --field-selector involvedObject.name=my-app-rs
```

**Cleanup:**
```bash
kubectl delete rs my-app-rs   # cascade deletes Pods too, by default
```

## 6. When should we use it?

Almost never directly — but understanding it is essential since Deployments are built on top of it.

## 7. When should we NOT use it?

- **Never use bare ReplicaSets for apps needing rolling updates** — RS has no concept of revision history or gradual rollout; changing the image in a RS's template does **not** trigger a rolling update of existing Pods.
- Use a **Deployment** instead almost always.

## 8. Comparison Table: Pod vs ReplicaSet vs Deployment (preview)

| Aspect | Bare Pod | ReplicaSet | Deployment |
|---|---|---|---|
| Self-healing | No | Yes | Yes |
| Maintains replica count | No | Yes | Yes |
| Rolling updates | No | No | **Yes** |
| Rollback history | No | No | **Yes** |
| Typically authored directly? | Rarely | Rarely | **Yes, commonly** |

## 9. Interview Questions

**Beginner**
- What does a ReplicaSet do if a Pod it manages is deleted manually?
- What field links a Pod to its owning ReplicaSet?

**Intermediate**
- If you update the container image in a ReplicaSet's `template`, do existing Pods get updated? Why or why not?
- What happens if you create a Pod manually with labels matching an existing ReplicaSet's selector?

**Advanced / Scenario**
- You scale a ReplicaSet from 5 to 2. Which 3 Pods get deleted, and what's the general selection logic?
- Why does Kubernetes recommend **never editing a Deployment's underlying ReplicaSet directly**? What could go wrong?

## 10. Mini Quiz

1. True/False: Updating a ReplicaSet's Pod template image automatically restarts existing Pods with the new image. -> **False**
2. What mechanism allows cascade deletion of Pods when their ReplicaSet is deleted? -> `ownerReferences`
3. What object almost always sits "on top of" a ReplicaSet in real-world usage? -> **Deployment**

## Hands-on Lab

```bash
kubectl apply -f replicaset.yaml
kubectl get pods -l app=my-app
kubectl delete pod <one-of-the-pod-names>
kubectl get pods -l app=my-app    # watch it come back within seconds
kubectl scale rs my-app-rs --replicas=1
kubectl get pods -l app=my-app    # watch 2 pods terminate
```

**Expected:** After manual Pod deletion, a replacement appears almost immediately. After scaling down, excess Pods terminate to match desired count.

## Summary

ReplicaSet introduces the **reconciliation loop** pattern that underlies almost everything in Kubernetes: watch desired state, compare to actual state, act to close the gap. It solves self-healing and replica-count enforcement — but lacks rollout/rollback intelligence, which is exactly the gap the **Deployment** fills.


# Lesson 6: Deployments

## 1. What is it?

A **Deployment** is a higher-level controller that manages **ReplicaSets** on your behalf, adding **rolling updates**, **rollback history**, and **declarative version management** on top of everything ReplicaSets give you.

**Definition:** You declare "I want 3 replicas of image `v2`," and the Deployment controller figures out how to get there safely — including how to transition from `v1` to `v2` without downtime.

**Problem it solves:** ReplicaSets can maintain a replica count, but they have zero concept of "roll from old version to new version gradually." Deployments add that entire orchestration layer.

## 2. Why do we need it?

**Without it**, updating an app means either deleting all Pods at once (downtime) or manually orchestrating a partial rollout Pod-by-Pod (error-prone, unscalable). Rollbacks after a bad release would mean manually recreating the old ReplicaSet from memory.

**Industry motivation:** Zero-downtime deployments are table stakes in production. Deployments make "ship a new version safely, and revert instantly if it's broken" a one-line `kubectl` command instead of a manual incident response.

## 3. Where is it used?

- **The default choice for virtually all stateless workloads**: web servers, APIs, backend services, frontends
- CI/CD pipelines update Deployments via `kubectl set image` or GitOps tools (ArgoCD/Flux) applying new YAML
- Every managed cloud K8s offering (EKS/AKS/GKE) — Deployments are the bread-and-butter object

## 4. How does it work internally?

```
+-------------------------- Deployment "my-app" --------------------------+
|  spec.template: image = v2                                              |
|  strategy: RollingUpdate (maxSurge: 1, maxUnavailable: 1)                |
+---------------------------------+------------------------------------+-+
                                 |                                       |
                    +------------v-----------+            +-------------v----------+
                    |  ReplicaSet (v1) - OLD  |            | ReplicaSet (v2) - NEW  |
                    |  desired: 0 (scaling v) |            | desired: 3 (scaling ^) |
                    +------------+------------+            +-------------+----------+
                                 |                                        |
                    Pod(v1) --> terminated                     Pod(v2) --> created
                    (one at a time, respecting maxUnavailable/maxSurge)
```

**Rolling update mechanics:**
1. You change `spec.template.spec.containers[0].image` from `v1` to `v2` and `kubectl apply`
2. Deployment controller creates a **new ReplicaSet** (RS-v2) with `replicas: 0`
3. It gradually scales RS-v2 up and RS-v1 down, respecting:
   - `maxSurge` — how many *extra* Pods above desired count are allowed during rollout (default 25%)
   - `maxUnavailable` — how many Pods can be unavailable during rollout (default 25%)
4. Once RS-v2 reaches full desired replica count and all Pods are `Ready`, RS-v1 is scaled to 0 (but **not deleted** — kept for rollback history, up to `revisionHistoryLimit`, default 10)
5. Each such change creates a new **revision** you can roll back to

**Deployment strategies:**

| Strategy | Behavior |
|---|---|
| `RollingUpdate` (default) | Gradual replace, controlled by maxSurge/maxUnavailable - zero downtime |
| `Recreate` | Kill ALL old Pods first, then create new ones - brief downtime, but guarantees no two versions run simultaneously |

## 5. How do we create and manage it?

**Deployment YAML — line by line:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deploy
  labels:
    app: my-app
spec:
  replicas: 3                         # desired Pod count
  revisionHistoryLimit: 10            # how many old ReplicaSets to keep for rollback
  selector:
    matchLabels:
      app: my-app                     # must match template labels below
  strategy:
    type: RollingUpdate               # or Recreate
    rollingUpdate:
      maxSurge: 1                     # allow 1 extra Pod during rollout (or "25%")
      maxUnavailable: 0               # zero downtime - never go below desired count
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app-container
        image: my-app:v2.0            # <-- change this to trigger a rollout
        ports:
        - containerPort: 8080
        readinessProbe:               # Deployment waits for this before considering
          httpGet:                    # a new Pod "successfully rolled out"
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

**Essential kubectl commands:**
```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get rs                                    # see old + new ReplicaSets
kubectl rollout status deployment/my-app-deploy   # watch rollout progress live
kubectl rollout history deployment/my-app-deploy  # see revision history
kubectl rollout history deployment/my-app-deploy --revision=2   # details of a specific revision

# Trigger a new rollout by changing the image
kubectl set image deployment/my-app-deploy app-container=my-app:v2.1

# Roll back to the previous revision
kubectl rollout undo deployment/my-app-deploy

# Roll back to a SPECIFIC revision
kubectl rollout undo deployment/my-app-deploy --to-revision=3

# Pause/resume a rollout (useful for canary-style manual gating)
kubectl rollout pause deployment/my-app-deploy
kubectl rollout resume deployment/my-app-deploy

# Scale
kubectl scale deployment my-app-deploy --replicas=5

# Delete (cascades to RS and Pods)
kubectl delete deployment my-app-deploy
```

**Best practices:**
- Always set a `readinessProbe` — without it, the Deployment thinks a new Pod is "ready" the instant its process starts
- Use `maxUnavailable: 0` for true zero-downtime rollouts
- Add `kubernetes.io/change-cause` annotation so `rollout history` is meaningful
- Set `revisionHistoryLimit` deliberately

**Common mistakes:**
- Forgetting readiness probes -> rollout "succeeds" but users hit errors
- Using `Recreate` strategy by accident for a service that can't tolerate downtime
- Assuming `kubectl rollout undo` reverts config changes too — it only reverts the **Pod template**, not Service definitions or ConfigMap contents referenced by name

**Debugging a stuck rollout:**
```bash
kubectl rollout status deployment/my-app-deploy    # will hang if rollout stuck
kubectl describe deployment my-app-deploy          # check Conditions: Progressing / Available
kubectl get rs -l app=my-app                        # see which RS is stuck scaling
kubectl describe pod <new-pod-name>                # check why new Pods aren't becoming Ready
```

## 6. When should we use it?

- Any **stateless** workload: web servers, APIs, workers that don't need stable network identity or persistent per-replica storage
- When you need rolling updates, easy rollbacks, and horizontal scaling

## 7. When should we NOT use it?

- **Stateful apps needing stable identity/storage** (databases, Kafka, etcd itself) -> use **StatefulSet**
- **One-per-node workloads** (log collectors, node monitoring agents) -> use **DaemonSet**
- **Run-to-completion batch work** -> use **Job**/**CronJob**

## 8. Comparison Table: Deployment vs StatefulSet vs DaemonSet (preview)

| Aspect | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| Pod identity | Interchangeable | Stable, ordered (`pod-0`, `pod-1`...) | One per node |
| Use case | Stateless apps | Databases, queues | Node-level agents |
| Scaling | Any order | Ordered (sequential) | Tied to node count |
| Storage | Typically shared/none | Per-replica persistent storage | Usually none |

## 9. Interview Questions

**Beginner**
- What's the difference between a ReplicaSet and a Deployment?
- How do you trigger a rolling update?

**Intermediate**
- What do `maxSurge` and `maxUnavailable` control, and what happens if you set `maxUnavailable: 0` and `maxSurge: 0` simultaneously? (Answer: rollout would deadlock)
- Why does Kubernetes keep old ReplicaSets around after a successful rollout instead of deleting them?

**Advanced / Scenario**
- A rollout appears stuck at "2/3 new Pods ready." How do you diagnose whether it's an app problem vs. a readiness probe misconfiguration vs. a resource constraint?
- You rolled back a Deployment, but the app is still broken because an accompanying ConfigMap change wasn't reverted. How would you architect Deployments + ConfigMaps to avoid this class of problem?

## 10. Mini Quiz

1. What creates a new ReplicaSet: scaling a Deployment, or changing its Pod template image? -> **Changing the Pod template**
2. Command to view rollout history? -> `kubectl rollout history deployment/<name>`
3. True/False: `kubectl rollout undo` can revert a ConfigMap's contents. -> **False**

## Hands-on Lab

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app-deploy
kubectl set image deployment/my-app-deploy app-container=nginx:1.26
kubectl rollout status deployment/my-app-deploy
kubectl get rs -l app=my-app          # observe old RS scaled to 0, new RS at desired count
kubectl rollout undo deployment/my-app-deploy
kubectl rollout status deployment/my-app-deploy
kubectl get rs -l app=my-app          # observe roles reversed
```

## Summary

Deployment is the **workhorse object** of real-world Kubernetes — it wraps ReplicaSets with rolling update orchestration and revision history, turning "ship a new version" into a safe, observable, reversible operation.


# Lesson 7: StatefulSets

## 1. What is it?

A **StatefulSet** is a controller for workloads that need **stable, unique network identities** and **stable, persistent per-replica storage** — even across rescheduling. Unlike a Deployment where Pods are interchangeable, StatefulSet Pods have a **fixed identity** (`myapp-0`, `myapp-1`, `myapp-2`) that persists across restarts.

**Problem it solves:** Deployments give every replica the same template — Pods are cattle, fully interchangeable. But databases (PostgreSQL, MongoDB, Kafka, etcd, Cassandra) need each replica to: keep its *own* disk across restarts, have a *predictable, stable* network name other replicas can address, and start up/shut down in a defined *order*.

## 2. Why do we need it?

**Without it**, running a 3-node database cluster on Kubernetes would be nearly impossible declaratively — you'd need external tooling to track "which Pod is node 0," ensure it always mounts the *same* disk after a restart, and manage startup ordering for cluster bootstrap.

**Industry motivation:** Distributed databases fundamentally rely on stable member identity. StatefulSet makes this a native Kubernetes primitive instead of a hand-rolled system.

## 3. Where is it used?

- **Databases:** PostgreSQL, MySQL, MongoDB, Cassandra clusters
- **Message queues/streaming:** Kafka, RabbitMQ clusters
- **Distributed coordination systems:** etcd, ZooKeeper
- **Kubernetes Operators** almost always manage a StatefulSet under the hood

## 4. How does it work internally?

```
+----------------------- StatefulSet "postgres" (replicas: 3) -----------------------+
|                                                                                        |
|  Ordered creation: pod-0 created & Ready BEFORE pod-1 starts, etc. (sequential)      |
|                                                                                        |
|   +--------------+        +--------------+        +--------------+                  |
|   | postgres-0   |        | postgres-1   |        | postgres-2   |                  |
|   | stable name: |        | stable name: |        | stable name: |                  |
|   | postgres-0.  |        | postgres-1.  |        | postgres-2.  |                  |
|   | postgres-svc |        | postgres-svc |        | postgres-svc |                  |
|   +------+-------+        +------+-------+        +------+-------+                  |
|          |                       |                       |                           |
|          v                       v                       v                           |
|   +--------------+        +--------------+        +--------------+                  |
|   | PVC: data-   |        | PVC: data-   |        | PVC: data-   |  <- each Pod gets its |
|   | postgres-0   |        | postgres-1   |        | postgres-2   |    OWN PVC             |
|   +--------------+        +--------------+        +--------------+                  |
|                                                                                        |
|   Headless Service (postgres-svc, clusterIP: None) provides DNS:                     |
|   postgres-0.postgres-svc.default.svc.cluster.local  -> always resolves              |
|   to whichever node is currently "postgres-0"                                        |
+----------------------------------------------------------------------------------------+
```

**Key mechanisms:**
1. **Stable network identity:** Requires a **headless Service** (`clusterIP: None`) — DNS returns a record *per Pod*: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.
2. **Stable storage via `volumeClaimTemplates`:** Each replica gets its own PersistentVolumeClaim, following the pattern `<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`. Reattaches to the **same PVC** on Pod recreation.
3. **Ordered, sequential lifecycle** (default `podManagementPolicy: OrderedReady`): scale up sequentially, scale down highest ordinal first.
4. **`Parallel` podManagementPolicy** (optional) skips ordering.

## 5. How do we create and manage it?

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
spec:
  clusterIP: None          # Headless Service - required for StatefulSet stable DNS
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-svc      # must match the headless Service above
  replicas: 3
  podManagementPolicy: OrderedReady   # default; sequential start/stop
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data                  # must match volumeClaimTemplate name below
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
  volumeClaimTemplates:                # StatefulSet auto-creates ONE PVC per replica
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```

**Essential kubectl commands:**
```bash
kubectl apply -f statefulset.yaml
kubectl get statefulset
kubectl get pods -l app=postgres -w              # watch ordered creation: -0, then -1, then -2
kubectl get pvc                                   # see per-replica PVCs: data-postgres-0, -1, -2
kubectl exec -it postgres-0 -- psql -U postgres   # connect to a SPECIFIC named replica

# Scale (respects ordering)
kubectl scale statefulset postgres --replicas=5

# Rolling update (default: reverse ordinal order, highest first)
kubectl set image statefulset/postgres postgres=postgres:16.1

# Delete StatefulSet WITHOUT deleting PVCs (data survives)
kubectl delete statefulset postgres --cascade=orphan

# PVCs are NOT deleted automatically even on normal statefulset delete
kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2
```

**Best practices:**
- Always pair with a **headless Service**
- Use `Retain` reclaim policy so you never accidentally lose database data
- For zero-downtime updates on clustered databases, consider `partition` in `updateStrategy.rollingUpdate`
- Combine with **PodDisruptionBudgets** so voluntary disruptions don't take down quorum-sensitive systems

**Common mistakes:**
- Forgetting the headless Service -> stable per-Pod DNS doesn't work
- Deleting a StatefulSet and assuming PVCs are gone too -> PVCs persist by default
- Treating StatefulSet Pods as interchangeable like Deployment Pods

**Debugging:**
```bash
kubectl describe statefulset postgres     # check Events, replica status
kubectl get pvc -l app=postgres           # confirm each replica has a bound PVC
kubectl describe pvc data-postgres-0      # check if PVC is stuck Pending
kubectl logs postgres-0                   # check the specific ordinal's logs
```

## 6. When should we use it?

Any workload requiring: stable network identity, ordered startup/shutdown, or per-replica persistent storage.

## 7. When should we NOT use it?

- **Stateless apps** -> use Deployment
- If you can use a **managed database service** (RDS, Cloud SQL) instead of self-hosting on K8s — often the better production choice

## 8. Comparison Table: Deployment vs StatefulSet

| Aspect | Deployment | StatefulSet |
|---|---|---|
| Pod naming | Random suffix | Stable ordinal (`postgres-0`, `postgres-1`) |
| Storage | Shared or none | Dedicated PVC per replica |
| Network identity | None stable | Stable DNS per Pod |
| Startup order | Parallel, any order | Sequential by default |
| Scale-down order | Any Pod | Highest ordinal first |
| Typical use | Stateless web/API services | Databases, queues, coordination systems |

## 9. Interview Questions

**Beginner**
- What's the key difference between how Deployment and StatefulSet Pods are named?
- Why does a StatefulSet require a headless Service?

**Intermediate**
- If you delete a StatefulSet, what happens to its PVCs by default?
- Explain what `OrderedReady` podManagementPolicy guarantees during scale-up and scale-down.

**Advanced / Scenario**
- You need to upgrade a 5-node Cassandra cluster running as a StatefulSet with zero data loss. How would you use `updateStrategy.rollingUpdate.partition` to stage this safely?
- A StatefulSet Pod (`postgres-1`) is stuck `Pending`. You check and its PVC is `Pending` too. What's your diagnostic path?

## 10. Mini Quiz

1. True/False: Deleting a StatefulSet automatically deletes its PVCs. -> **False**
2. What type of Service is required for stable per-Pod DNS in a StatefulSet? -> **Headless Service**
3. In default scale-down behavior, which Pod is terminated first — `myapp-0` or `myapp-2`? -> **myapp-2**

## Hands-on Lab

```bash
kubectl apply -f statefulset.yaml
kubectl get pods -l app=postgres -w        # observe sequential creation
kubectl get pvc
kubectl exec -it postgres-0 -- hostname    # confirm stable hostname
kubectl delete pod postgres-1
kubectl get pods -l app=postgres -w        # postgres-1 recreated with SAME name
kubectl get pvc                             # confirm same PVC reused
```

## Summary

StatefulSet exists because not everything is stateless cattle — some workloads are "pets" that need consistent identity and storage across their lifetime.

# Lesson 8: DaemonSets

## 1. What is it?

A **DaemonSet** ensures that a copy of a specific Pod runs on **every node** in the cluster (or a selected subset of nodes) — automatically. When a new node joins, the DaemonSet's Pod is scheduled there without you doing anything; when a node leaves, that Pod is garbage collected with it.

**Problem it solves:** Some workloads are inherently "infrastructure-per-node" — you want **exactly one per node**, tied to that node's lifecycle.

## 2. Why do we need it?

**Without it**, running a log collector or monitoring agent on every node would mean manually tracking node count and creating/deleting Pods as nodes scale up/down — completely unworkable with autoscaling clusters.

**Industry motivation:** Nearly every production cluster needs node-level agents: log shippers (Fluent Bit), metrics collectors (node-exporter), security/runtime agents (Falco), and the CNI networking plugin itself (Calico, Cilium) — all deployed as DaemonSets.

## 3. Where is it used?

- **Logging:** Fluent Bit/Fluentd DaemonSet reads container logs from each node's disk
- **Monitoring:** `node-exporter` (Prometheus) runs on every node
- **Networking:** CNI plugins (Calico, Cilium, Flannel) run as DaemonSets
- **Security:** Falco, security scanners
- **Storage:** CSI node plugins for volume mounting

## 4. How does it work internally?

```
+---------------------- DaemonSet "fluent-bit" ----------------------+
|  No "replicas" field - count is implicitly "one per matching node"  |
+---------------+--------------+--------------+------------------------+
                 |              |              |
        +--------v---+ +-------v----+ +-------v----+
        |  Node 1    | |  Node 2    | |  Node 3    |
        | fluent-bit | | fluent-bit | | fluent-bit |  <- one Pod per node,
        |   Pod      | |   Pod      | |   Pod      |    auto-scheduled
        +------------+ +------------+ +------------+

   New Node 4 joins the cluster
                 |
                 v
        +------------+
        |  Node 4    |
        | fluent-bit |  <- DaemonSet controller automatically schedules
        |   Pod      |    a Pod here too, no manual action needed
        +------------+
```

**Key internal mechanism:** Since Kubernetes 1.12+, the DaemonSet controller directly assigns Pods to nodes matching its `nodeSelector`/affinity rules (bypassing normal scheduling preferences, though it still respects taints/tolerations and resource fit checks).

**Tolerations are critical:** Control-plane nodes are tainted `NoSchedule` by default, so regular Pods won't land there. But many DaemonSets (CNI plugins, monitoring agents) *need* to run on control-plane nodes too — so DaemonSet Pods commonly include explicit **tolerations**.

## 5. How do we create and manage it?

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit             # must match template labels
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      tolerations:                 # allow scheduling onto control-plane nodes too
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        volumeMounts:
        - name: varlog                # mount the HOST's log directory
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlog
        hostPath:                     # hostPath volume - reads directly from node's filesystem
          path: /var/log
```

Restricting to a subset of nodes (e.g., only GPU nodes):
```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu: "true"                   # only schedules on nodes labeled gpu=true
```

**Essential kubectl commands:**
```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset                          # shows DESIRED / CURRENT / READY / UP-TO-DATE per node
kubectl get pods -l app=fluent-bit -o wide     # confirm one Pod per node
kubectl describe daemonset fluent-bit          # events, node selector, tolerations

# Update the image (triggers RollingUpdate by default)
kubectl set image daemonset/fluent-bit fluent-bit=fluent/fluent-bit:2.3

kubectl rollout status daemonset/fluent-bit
kubectl delete daemonset fluent-bit            # removes Pods from ALL nodes
```

**Best practices:**
- Always set tight resource `requests`/`limits` — DaemonSet Pods run on *every* node
- Use `nodeSelector` to scope DaemonSets that only need to run on specific node types
- Add appropriate tolerations if the DaemonSet must run on tainted nodes

**Common mistakes:**
- Forgetting tolerations -> DaemonSet Pods silently skip control-plane or specially-tainted nodes
- Using `hostPath` volumes carelessly -> potential security risk
- Treating a DaemonSet like it can be scaled with `kubectl scale` -> **this doesn't work**

**Debugging (missing DaemonSet Pod on a specific node):**
```bash
kubectl get daemonset fluent-bit    # compare DESIRED vs CURRENT counts
kubectl describe daemonset fluent-bit   # check Events for scheduling failures
kubectl describe node <node-name>   # check for taints the DaemonSet doesn't tolerate
```

## 6. When should we use it?

Node-level infrastructure: logging agents, metrics exporters, CNI plugins, security/runtime monitoring, storage node plugins.

## 7. When should we NOT use it?

- **Application workloads** that need horizontal scaling independent of node count -> use Deployment
- If you only need the agent on a **few specific nodes permanently** -> a Deployment with node affinity might be simpler

## 8. Comparison Table: Deployment vs DaemonSet

| Aspect | Deployment | DaemonSet |
|---|---|---|
| Replica count | Manually specified, arbitrary | Implicitly = number of matching nodes |
| Scaling | `kubectl scale` | Not applicable |
| Scheduling target | Any node with capacity | Every (matching) node, one Pod each |
| Typical workload | Stateless applications | Node-level infrastructure/agents |
| New node behavior | No automatic Pod placement | Pod automatically created there |

## 9. Interview Questions

**Beginner**
- Why can't you use `kubectl scale` on a DaemonSet?
- Give three real-world examples of DaemonSet use cases.

**Intermediate**
- Why do many DaemonSets need explicit tolerations that regular Deployment Pods don't need?
- What happens to a DaemonSet's Pod when its node is removed from the cluster?

**Advanced / Scenario**
- Your logging DaemonSet is missing logs from 2 out of 10 nodes. Walk through your full diagnostic process.
- You want a DaemonSet to run only on nodes with SSD storage, but also tolerate a custom taint `dedicated=monitoring:NoSchedule` on those same nodes. Write the relevant YAML snippet.

## 10. Mini Quiz

1. True/False: You manually set `replicas: N` in a DaemonSet spec. -> **False**
2. What Kubernetes construct do control-plane nodes use by default to repel non-DaemonSet Pods? -> **Taints**
3. Command to update a DaemonSet's image? -> `kubectl set image daemonset/<name> <container>=<new-image>`

## Hands-on Lab

```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset fluent-bit
kubectl get pods -l app=fluent-bit -o wide
kubectl get nodes
```

**Expected:** `DESIRED`, `CURRENT`, and `READY` columns all equal your node count.

## Summary

DaemonSet flips the usual "how many replicas do I want" question into "run this on every node, automatically, forever."


# Lesson 9: Jobs & CronJobs

## 1. What is it?

A **Job** runs one or more Pods to **completion** — unlike Deployments/ReplicaSets which keep Pods running forever, a Job's Pods are expected to **finish and exit successfully**, and the Job tracks that success.

A **CronJob** is a Job that runs **on a schedule** (cron syntax) — it creates a new Job object at each scheduled time.

**Problem it solves:** Batch work — data migrations, report generation, backups, one-off scripts, nightly cleanup tasks — has a fundamentally different lifecycle: **run, finish, stop**.

## 2. Why do we need them?

**Without Jobs**, you'd run batch scripts outside Kubernetes entirely, losing all the benefits of container packaging, resource limits, retries, and cluster-native scheduling.

**Industry motivation:** Nearly every production system needs scheduled maintenance: database backups at 2 AM, report generation, cache warming, certificate renewal checks.

## 3. Where is it used?

- **Database backups** (nightly `pg_dump` Jobs)
- **Batch data processing** (ETL pipelines, ML training jobs)
- **One-off migrations** (schema migration Job run once during a deploy)
- **Scheduled maintenance**: cert renewal checks, log rotation, cache invalidation
- **CI/CD**: ephemeral test-runner Jobs

## 4. How does it work internally?

```
+--------------------------- Job "data-migration" ---------------------------+
|  spec.completions: 3   (need 3 successful Pod completions total)            |
|  spec.parallelism: 1   (run 1 at a time)                                    |
|  spec.backoffLimit: 4  (retry up to 4 times on failure before giving up)   |
|                                                                              |
|  Pod-1 runs -> exits 0 (Succeeded)  -> completions: 1/3                    |
|  Pod-2 runs -> exits 1 (Failed)     -> Job creates Pod-2-retry automatically|
|  Pod-2-retry runs -> exits 0        -> completions: 2/3                    |
|  Pod-3 runs -> exits 0              -> completions: 3/3 -> Job COMPLETE    |
+------------------------------------------------------------------------------+

+---------------------- CronJob "nightly-backup" ----------------------+
|  schedule: "0 2 * * *"   (2 AM daily, cron syntax)                    |
|                                                                        |
|  At 2:00 AM  -> CronJob controller creates a new Job object           |
|                     |                                                 |
|                     v                                                 |
|              Job runs to completion, tracked independently            |
|                                                                        |
|  At 2:00 AM next day -> ANOTHER new Job object created                |
|  (old Job objects retained per successfulJobsHistoryLimit)            |
+--------------------------------------------------------------------+
```

**Job completion modes:**
- **Non-parallel** (default): one Pod runs, Job is done when it succeeds
- **Fixed completion count** (`completions: N`): Job needs N total successful Pod completions
- **Work queue** (`completions` unset, `parallelism: N`): Pods coordinate among themselves

**Retry logic:** if a Pod fails, the Job controller creates a replacement Pod (bounded by `backoffLimit`, default 6), with exponential backoff between retries.

**CronJob concurrency handling** (`concurrencyPolicy`):
- `Allow` (default) — multiple Job runs can overlap
- `Forbid` — skip the new run entirely if the previous one hasn't finished
- `Replace` — cancel the still-running Job and start the new one

## 5. How do we create and manage them?

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 3            # total successful completions needed
  parallelism: 1            # how many Pods run concurrently
  backoffLimit: 4           # max retries before marking Job Failed
  activeDeadlineSeconds: 600  # kill the whole Job if it runs longer than this (10 min)
  template:
    spec:
      restartPolicy: OnFailure   # Job Pods MUST use OnFailure or Never (never "Always")
      containers:
      - name: migrate
        image: my-migration-tool:1.0
        command: ["python", "migrate.py"]
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"              # cron syntax: min hour day month weekday
  concurrencyPolicy: Forbid           # don't start a new run if previous still active
  successfulJobsHistoryLimit: 3       # keep last 3 successful Job objects
  failedJobsHistoryLimit: 1           # keep last 1 failed Job object
  startingDeadlineSeconds: 100        # how late a missed schedule can still start
  jobTemplate:                        # this IS a Job spec, nested
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres-backup-tool:1.0
            command: ["/bin/sh", "-c", "pg_dump -h db-service > /backup/dump.sql"]
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

**Essential kubectl commands:**
```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl describe job data-migration
kubectl get pods -l job-name=data-migration    # auto-added label to track owned pods
kubectl logs -l job-name=data-migration          # see logs from job's pods
kubectl delete job data-migration                # cleans up completed pods too

# CronJobs
kubectl apply -f cronjob.yaml
kubectl get cronjob
kubectl get jobs --watch                          # watch Jobs get created on schedule
kubectl create job --from=cronjob/nightly-backup manual-run-now   # trigger manually
kubectl patch cronjob nightly-backup -p '{"spec":{"suspend":true}}'  # pause the schedule
```

**Best practices:**
- Always set `restartPolicy: OnFailure` or `Never` for Job Pods
- Set `activeDeadlineSeconds` to prevent runaway Jobs
- Set `backoffLimit` deliberately
- For CronJobs, use `concurrencyPolicy: Forbid` for tasks that must never overlap
- Set history limits low enough to avoid cluttering the cluster

**Common mistakes:**
- Forgetting `restartPolicy: OnFailure`/`Never` -> API rejects the Job spec
- Not setting `activeDeadlineSeconds` on a Job with a bug that hangs forever
- Using `concurrencyPolicy: Allow` (default) for a task that isn't safe to run concurrently
- Confusing cron schedule timezone — runs in the kube-controller-manager's configured timezone (UTC by default unless `timeZone` field set)

**Debugging:**
```bash
kubectl describe job <name>              # check Events, completions status
kubectl get pods -l job-name=<name>      # see individual Pod statuses/retries
kubectl logs <pod-name> --previous       # logs from a failed retry attempt
kubectl describe cronjob <name>          # check Last Schedule Time, Active jobs
```

**Cleanup:**
```bash
kubectl delete job data-migration          # deletes Job + its Pods
kubectl delete cronjob nightly-backup      # stops future schedules
```

## 6. When should we use it?

- **Job:** one-off or batch tasks that must run to completion
- **CronJob:** recurring scheduled tasks

## 7. When should we NOT use it?

- **Long-running services** -> use Deployment, never a Job
- **Sub-second or very high-frequency scheduling needs** -> CronJob's minimum granularity is 1 minute
- **Tasks requiring guaranteed exact-time execution** -> CronJob scheduling has inherent delay/jitter

## 8. Comparison Table: Job vs CronJob vs Deployment

| Aspect | Deployment | Job | CronJob |
|---|---|---|---|
| Pod lifecycle | Runs forever | Runs to completion | Creates Jobs on schedule |
| restartPolicy | `Always` | `OnFailure`/`Never` | Same as Job |
| Use case | Long-running services | One-off batch tasks | Recurring batch tasks |
| Self-healing | Yes, continuous | Retries up to `backoffLimit` | Each scheduled Job retries independently |

## 9. Interview Questions

**Beginner**
- Why can't a Job Pod use `restartPolicy: Always`?
- What does `backoffLimit` control?

**Intermediate**
- Explain the difference between `concurrencyPolicy: Forbid` and `Replace` for CronJobs.
- What's the difference between `completions` and `parallelism` in a Job spec?

**Advanced / Scenario**
- A CronJob is supposed to run every 5 minutes but you notice it's skipping runs entirely during a period of controller downtime. What field controls how late a missed run can still execute?
- You need a Job that processes 100 items from a queue using 10 parallel worker Pods. How would you configure `completions` and `parallelism`?

## 10. Mini Quiz

1. True/False: A Job Pod can use `restartPolicy: Always`. -> **False**
2. What CronJob field prevents overlapping runs? -> `concurrencyPolicy: Forbid`
3. What field auto-deletes a Job N seconds after it finishes? -> `ttlSecondsAfterFinished`

## Hands-on Lab

```bash
kubectl apply -f job.yaml
kubectl get pods -l job-name=data-migration -w    # watch completions happen
kubectl logs -l job-name=data-migration
kubectl get job data-migration                     # check COMPLETIONS column, e.g. 3/3

kubectl apply -f cronjob.yaml
kubectl get cronjob nightly-backup
kubectl create job --from=cronjob/nightly-backup test-run-now
kubectl get jobs
```

## Summary

Jobs and CronJobs cover the "run to completion" half of workloads that always-running controllers can't handle.


# Lesson 10: Services

## 1. What is it?

A **Service** is a stable networking abstraction that provides a **single, unchanging IP address and DNS name** for a dynamic, ever-changing set of Pods.

**Definition:** A Service continuously watches for Pods matching its label selector and load-balances traffic across whichever Pods currently match.

## 2. Why do we need it?

**Without it**, every time a Deployment replaced a Pod, every client would need to discover the new Pod IP somehow — utterly unworkable at scale.

**Industry motivation:** Microservices need to call each other by a stable name regardless of how many replicas exist or how often they're replaced.

## 3. Where is it used?

- **Every microservice architecture** — internal service-to-service calls
- **Exposing apps externally** — via `LoadBalancer` type
- **Database access** — StatefulSet databases exposed via headless Services
- **Ingress backends**

## 4. How does it work internally?

```
+--------------------- Service "backend-svc" (ClusterIP: 10.96.5.10) ---------------------+
|  selector: { app: backend }                                                                |
|  port: 80 -> targetPort: 8080                                                              |
+---------------------------------+---------------------------------------------------------+
                                 |
                    +------------v-----------+
                    |   Endpoints/EndpointSlice |  <- continuously updated list of matching Pod IPs
                    |  10.244.1.5:8080          |
                    |  10.244.2.9:8080          |
                    |  10.244.3.2:8080          |
                    +------------+-----------+
                                 |
              +------------------+------------------+
              v                  v                  v
         Pod (backend)      Pod (backend)      Pod (backend)
         10.244.1.5         10.244.2.9         10.244.3.2

    +--------------- How traffic actually gets routed ---------------+
    |  Client sends request to 10.96.5.10:80 (the Service's ClusterIP)|
    |           |                                                      |
    |           v                                                      |
    |  kube-proxy on the client's node intercepts via iptables/IPVS   |
    |  rules, rewrites destination to one of the Endpoint IPs         |
    |           |                                                      |
    |           v                                                      |
    |  Packet delivered directly to the chosen Pod's IP:port          |
    +--------------------------------------------------------------+
```

**Key internal facts:**
- The **ClusterIP is virtual** — implemented via iptables/IPVS rules on every node, maintained by **kube-proxy**
- An **EndpointSlice** is auto-created and continuously updated to reflect the current set of Ready Pods
- **Only `Ready` Pods** are included in Endpoints
- DNS resolution: CoreDNS resolves `backend-svc.namespace.svc.cluster.local` -> the Service's ClusterIP

## Service Types

| Type | Behavior | Use Case |
|---|---|---|
| `ClusterIP` (default) | Internal-only virtual IP | Service-to-service communication |
| `NodePort` | Exposes the Service on a static port (30000-32767) on every node's IP | Simple external access, dev/test |
| `LoadBalancer` | Provisions an actual cloud load balancer | Production external-facing services |
| `ExternalName` | Maps the Service to an external DNS name (CNAME) | Referencing external databases/APIs |
| `Headless` (`clusterIP: None`) | No load-balancing; DNS returns individual Pod IPs | StatefulSets needing per-Pod addressing |

## 5. How do we create and manage them?

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP              # default; can be omitted
  selector:
    app: backend                # matches Pods with this label
  ports:
  - port: 80                    # port the SERVICE listens on
    targetPort: 8080             # port on the POD to forward to
    protocol: TCP
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080              # optional - auto-assigned from 30000-32767 if omitted
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

Multi-port Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-svc
spec:
  selector:
    app: my-app
  ports:
  - name: http                   # names REQUIRED when multiple ports exist
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
```

**Essential kubectl commands:**
```bash
kubectl apply -f service.yaml
kubectl get svc                                # list services + ClusterIPs
kubectl describe svc backend-svc               # see selector, endpoints, events
kubectl get endpoints backend-svc              # verify which Pod IPs are backing it
kubectl get endpointslices -l kubernetes.io/service-name=backend-svc

# Quick expose an existing Deployment as a Service
kubectl expose deployment my-app --port=80 --target-port=8080 --type=ClusterIP

# Test connectivity from within the cluster
kubectl run tmp-test --rm -it --image=busybox -- wget -qO- backend-svc:80

kubectl delete svc backend-svc
```

**Best practices:**
- Always name multi-port Services' ports explicitly
- Use `ClusterIP` by default; only use `NodePort`/`LoadBalancer` for services genuinely needing external access
- Combine with **readiness probes**
- Always use the **DNS name**, never the ClusterIP directly

**Common mistakes:**
- Selector doesn't match any Pod labels -> Service has zero Endpoints
- Confusing `port` and `targetPort`
- Using `NodePort` in production for external traffic instead of `LoadBalancer`/`Ingress`
- Forgetting that `LoadBalancer` type provisions a **real cloud resource** — costs money

**Debugging a Service with connectivity issues:**
```bash
kubectl get endpoints <svc-name>              # empty? -> selector/label mismatch
kubectl describe svc <svc-name>                # confirm selector, ports match expectations
kubectl get pods -l <selector-from-above>      # confirm Pods actually exist and are Ready
kubectl run debug --rm -it --image=busybox -- nslookup <svc-name>   # DNS resolution test
kubectl run debug --rm -it --image=busybox -- wget -qO- <svc-name>:<port>   # connectivity test
```

## 6. When should we use each type?

- **ClusterIP:** default for all internal service-to-service communication
- **NodePort:** quick dev/test external access
- **LoadBalancer:** production external-facing services on a cloud provider
- **Headless:** StatefulSets, or per-Pod identity needs
- **ExternalName:** abstracting an external dependency behind an internal-looking name

## 7. When should we NOT use certain types?

- Don't use `LoadBalancer` for every internal microservice — use `ClusterIP` + Ingress instead
- Don't use `NodePort` for production external-facing traffic if `LoadBalancer` or `Ingress` is available

## 8. Comparison Table

| Type | Internal Access | External Access | Cloud Resource Created | Typical Use |
|---|---|---|---|---|
| ClusterIP | Yes | No | None | Service-to-service |
| NodePort | Yes | Yes (via node IP:port) | None | Dev/test, on-prem |
| LoadBalancer | Yes | Yes (via LB IP) | Cloud LB | Production external |
| Headless | Yes (per-Pod DNS) | No | None | StatefulSet peer discovery |
| ExternalName | N/A (CNAME) | N/A | None | External dependency abstraction |

## 9. Interview Questions

**Beginner**
- What problem does a Service solve that Pods alone can't?
- What's the difference between `port` and `targetPort`?

**Intermediate**
- How does a Service actually route traffic at the packet level — what role does kube-proxy play?
- Why are only `Ready` Pods included in a Service's Endpoints?

**Advanced / Scenario**
- A Service shows healthy Endpoints, but clients still get intermittent connection failures. What are 3 possible causes beyond the Service/Endpoints layer itself?
- Explain exactly what happens at the iptables/IPVS level when a Pod is added to a Service's Endpoints.

## 10. Mini Quiz

1. True/False: A Service's ClusterIP is assigned to a real network interface. -> **False**
2. What excludes a Pod from a Service's Endpoints even though it's `Running`? -> Failing its **readiness probe**
3. Which Service type provisions an actual cloud load balancer? -> `LoadBalancer`

## Hands-on Lab

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl get svc web
kubectl get endpoints web                       # should show 3 Pod IPs
kubectl run test --rm -it --image=busybox -- wget -qO- web:80   # should return nginx HTML
kubectl scale deployment web --replicas=1
kubectl get endpoints web                        # should now show only 1 IP
```

## Summary

Services decouple "who's calling" from "which specific Pod answers," providing a live abstraction over ephemeral Pod IPs.


# Lesson 11: Ingress

## 1. What is it?

**Ingress** is an API object that manages **external HTTP/HTTPS access** to Services inside the cluster, providing **host-based and path-based routing**, **TLS termination**, and a **single entry point**.

**Important distinction:** Ingress itself is just a set of **routing rules**. It requires an **Ingress Controller** (NGINX, Traefik, HAProxy, AWS ALB Controller, etc.) actually running in the cluster to read those rules and configure a real proxy/load balancer.

## 2. Why do we need it?

**Without it**, exposing 10 microservices externally would mean 10 separate `LoadBalancer` Services — 10 separate cloud load balancers, 10 separate IPs, 10x the cost.

**Industry motivation:** Real-world apps expose one domain with path-based or subdomain-based routing — all through **one** load balancer, one IP, one TLS certificate setup.

## 3. Where is it used?

- **Every production web-facing Kubernetes deployment**
- **TLS termination** at the edge — via cert-manager
- **Multi-tenant platforms**

## 4. How does it work internally?

```
                     Internet
                        |
                        v
              +-------------------+
              |  Cloud Load        |  <- ONE external IP
              |  Balancer          |
              +---------+---------+
                        |
                        v
        +-------------------------------+
        |   Ingress Controller Pod        |  <- e.g., nginx-ingress-controller
        |   (watches Ingress objects       |     running as a Deployment
        |    via API server, reconfigures  |
        |    its internal proxy rules)     |
        +---------------+-----------------+
                         |
         reads Ingress rules:
         host: example.com
           /api    -> backend-svc:80
           /       -> frontend-svc:80
                         |
           +-------------+-------------+
           v                           v
    backend-svc (ClusterIP)     frontend-svc (ClusterIP)
           |                           |
           v                           v
      backend Pods                frontend Pods
```

**Key internal mechanism:**
1. You create an **Ingress object** declaring routing rules
2. The **Ingress Controller** continuously watches Ingress objects via the API server
3. On any change, it **regenerates its underlying proxy config** and reloads
4. Traffic flow: Client -> Cloud LB -> Ingress Controller Pod -> routes based on Host header/path -> target ClusterIP Service -> Pod

## 5. How do we create and manage it?

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # controller-specific behavior tweaks
spec:
  ingressClassName: nginx              # which Ingress Controller handles this
  rules:
  - host: example.com                   # only matches requests with this Host header
    http:
      paths:
      - path: /api                      # example.com/api/* -> backend-svc
        pathType: Prefix                # Prefix | Exact | ImplementationSpecific
        backend:
          service:
            name: backend-svc
            port:
              number: 80
      - path: /                         # example.com/* (catch-all) -> frontend-svc
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

Ingress with TLS termination:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # cert-manager auto-issues cert
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-com-tls          # cert-manager writes the cert/key here
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

**Essential kubectl commands:**
```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress main-ingress       # see rules, backend, TLS, events
kubectl get ingressclass                     # see which controllers are available
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx <controller-pod-name>
kubectl delete ingress main-ingress
```

**Best practices:**
- Always specify `ingressClassName` explicitly
- Use `cert-manager` for automated TLS certificate issuance/renewal
- Use `pathType: Prefix` deliberately understood

**Common mistakes:**
- Forgetting to install an Ingress Controller at all -> Ingress objects exist but do **nothing**
- Path/host mismatches -> 404s from the controller itself, not your app
- Relying on annotations specific to one controller while running a different one
- Missing TLS secret -> HTTPS requests fail or serve self-signed/default certs

**Debugging:**
```bash
kubectl get ingress main-ingress -o yaml         # confirm rules are as expected
kubectl describe ingress main-ingress            # check "Events" for controller sync issues
kubectl get pods -n ingress-nginx                # confirm controller is actually Running
kubectl logs -n ingress-nginx <controller-pod>   # see actual routing decisions/errors
curl -H "Host: example.com" http://<ingress-controller-external-ip>/api
```

## 6. When should we use it?

Any time you're exposing **HTTP/HTTPS** services externally and want: a single entry point, path/host-based routing, centralized TLS termination.

## 7. When should we NOT use it?

- **Non-HTTP(S) traffic** (raw TCP/UDP, gRPC) -> generally still need `LoadBalancer` Services
- **Extremely simple single-service clusters**
- For advanced traffic splitting, consider a **Service Mesh**

## 8. Comparison Table: Service (LoadBalancer) vs Ingress

| Aspect | LoadBalancer Service | Ingress |
|---|---|---|
| Layer | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| Routing | One Service = one LB, no path/host logic | Path/host-based routing to many Services |
| TLS termination | Manual, per-Service | Centralized, often automated |
| Cost (cloud) | One LB per Service | One LB total, shared |
| Requires extra component? | No | **Yes** — an Ingress Controller |

## 9. Interview Questions

**Beginner**
- Why doesn't creating an Ingress object alone route any traffic?
- What's the difference between `pathType: Prefix` and `pathType: Exact`?

**Intermediate**
- Explain the full traffic path from a browser request to `https://example.com/api` down to the Pod actually handling it.
- Why might annotations behave differently when switching from an NGINX Ingress Controller to Traefik?

**Advanced / Scenario**
- Users report intermittent 502 errors only for `/api` routes, while `/` works fine. Walk through your diagnostic path.
- You need to migrate from one Ingress Controller (NGINX) to another (Traefik) with zero downtime.

## 10. Mini Quiz

1. True/False: Ingress works out-of-the-box without installing any additional component. -> **False**
2. What OSI layer does Ingress operate at? -> **Layer 7 (HTTP/HTTPS)**
3. What field specifies which Ingress Controller should handle a given Ingress object? -> `ingressClassName`

## Hands-on Lab

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx -w
kubectl create deployment web --image=nginx:1.25
kubectl expose deployment web --port=80
kubectl apply -f ingress.yaml
kubectl get ingress main-ingress
curl -H "Host: example.com" http://<ingress-controller-external-ip>/
```

## Summary

Ingress adds the **Layer 7 smart routing** that plain Services (Layer 4) can't provide. The critical mental model: **Ingress is config; the Ingress Controller is the engine that acts on it**.


# Lesson 12: Gateway API

## 1. What is it?

The **Gateway API** is the next-generation successor to Ingress — a more expressive, **role-oriented**, and **protocol-flexible** set of Kubernetes APIs (`GatewayClass`, `Gateway`, `HTTPRoute`, `TCPRoute`, `GRPCRoute`, etc.).

## 2. Why do we need it?

**Ingress's core limitations:**
- **No role separation** — infra concerns and app routing mixed in the same object
- **No standard way to express advanced routing** — traffic splitting, header-based routing required vendor annotations
- **HTTP-only in practice**
- **Annotation sprawl** — unstructured strings with no schema validation

**Industry motivation:** Platform teams manage "how traffic enters" (Gateway); app teams manage "how my service's traffic is routed" (HTTPRoute).

## 3. Where is it used?

- **Multi-team platforms** where infra teams provision shared Gateways and app teams self-serve routing rules
- **Service mesh integration** — Istio adopting Gateway API as configuration surface
- **Advanced traffic management** — canary releases, A/B testing, cross-namespace routing

## 4. How does it work internally?

```
+------------------------- Role Separation -------------------------+
|                                                                      |
|  Platform/Infra Team owns:          Application Team owns:         |
|  +------------------+               +------------------+          |
|  |   GatewayClass     |              |    HTTPRoute       |          |
|  |  (which controller |              |  (path/header      |          |
|  |   implementation)  |              |   matching rules,  |          |
|  +---------+----------+              |   backend, weight) |          |
|            |                          +---------+----------+          |
|            v                                    |                    |
|  +------------------+                          |                    |
|  |      Gateway       |<-------------------------+                    |
|  | (listeners: ports,  |   HTTPRoutes attach TO a Gateway            |
|  |  protocols, TLS     |   via parentRefs                            |
|  |  certs, hostnames)  |                                              |
|  +---------+----------+                                              |
+------------+---------------------------------------------------------+
             v
    Actual load balancer / proxy provisioned by the GatewayClass's controller
             |
             v
    Routes traffic per HTTPRoute rules -> Services -> Pods
```

**Resource hierarchy:**
1. **GatewayClass** (cluster-scoped, infra-managed) — defines which controller implementation handles Gateways of this class
2. **Gateway** (infra-managed) — listener configuration: ports, protocols, hostnames, TLS
3. **HTTPRoute/TCPRoute/GRPCRoute/TLSRoute** (app-team-managed) — routing rules

**Advanced capabilities:**
- **Weighted traffic splitting** (canary) — natively in the spec
- **Header/query-param-based matching**
- **Cross-namespace routing** with explicit `ReferenceGrant`
- **Request/response mutation** — standardized

## 5. How do we create and manage it?

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.io/nginx-gateway-controller
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"          # wildcard supported natively
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-com-tls
    allowedRoutes:
      namespaces:
        from: All                        # or "Selector" / "Same"
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-route
spec:
  parentRefs:
  - name: main-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: backend-v1-svc
      port: 80
      weight: 90                          # 90% of traffic -> v1
    - name: backend-v2-svc
      port: 80
      weight: 10                          # 10% of traffic -> v2 (native canary!)
```

**Essential kubectl commands:**
```bash
kubectl apply -f gatewayclass.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httproute.yaml
kubectl get gatewayclass
kubectl get gateway
kubectl get httproute
kubectl describe gateway main-gateway     # check listener status, attached routes
kubectl describe httproute backend-route  # check parentRefs resolution, backend status
```

**Best practices:**
- Let infra teams own `GatewayClass`/`Gateway`; app teams own `HTTPRoute`
- Use `ReferenceGrant` explicitly for cross-namespace routing
- Prefer native weighted `backendRefs` for canary rollouts

**Common mistakes:**
- Forgetting `ReferenceGrant` when an HTTPRoute wants to attach to a Gateway in another namespace
- Not checking `status` conditions on Gateway/HTTPRoute objects
- Assuming Gateway API is a drop-in replacement everywhere

**Debugging:**
```bash
kubectl get gateway main-gateway -o yaml    # check status.conditions
kubectl get httproute backend-route -o yaml # check status.parents conditions
```

## 6. When should we use it?

New clusters/platforms starting fresh, especially with multi-team ownership needs; native traffic splitting/header routing without vendor lock-in.

## 7. When should we NOT use it?

If your existing Ingress setup works fine and you don't need advanced capabilities; if your controller doesn't have mature Gateway API support yet.

## 8. Comparison Table: Ingress vs Gateway API

| Aspect | Ingress | Gateway API |
|---|---|---|
| Role separation | None | Explicit: GatewayClass/Gateway vs HTTPRoute |
| Traffic splitting | Vendor annotations only | **Native** (`weight` field) |
| Protocol support | HTTP/HTTPS-centric | HTTP, HTTPS, TCP, UDP, TLS, gRPC |
| Cross-namespace routing | Implicit, less secure by default | Explicit via `ReferenceGrant` |
| Status/debuggability | Limited | Rich `status.conditions` |
| Maturity/adoption | Extremely widespread | Growing, GA since 2023 |

## 9. Interview Questions

**Beginner**
- What are the three core Gateway API resources, and who typically owns each?
- How does Gateway API express canary traffic splitting natively?

**Intermediate**
- Why does Gateway API require an explicit `ReferenceGrant` for cross-namespace routing?
- What's the relationship between `GatewayClass` and `Gateway`?

**Advanced / Scenario**
- Design a Gateway API setup for a platform with 5 app teams sharing one central Gateway with strict namespace controls.
- Migrating a complex NGINX Ingress setup with rate-limiting/auth annotations to Gateway API — what translates natively vs. needs vendor extensions?

## 10. Mini Quiz

1. True/False: In Gateway API, the same team typically owns both the Gateway and the HTTPRoute. -> **False**
2. What object is required for secure cross-namespace route attachment? -> `ReferenceGrant`
3. What native field enables canary/weighted traffic splitting? -> `weight` (on `backendRefs`)

## Hands-on Lab

```bash
kubectl apply -f gatewayclass.yaml
kubectl apply -f gateway.yaml
kubectl get gateway main-gateway -o yaml     # check status.conditions.Programmed=True
kubectl apply -f httproute.yaml
kubectl get httproute backend-route -o yaml  # check status.parents for Accepted=True
curl -H "Host: api.example.com" http://<gateway-external-ip>/v1
```

## Summary

Gateway API is Ingress's more mature successor — splitting infra and app concerns, standardizing advanced routing, and adding real debuggability via status conditions.


# Lesson 13: Namespaces, Resource Quotas & Limit Ranges

## 1. What is it?

**Namespace** — a virtual cluster partition within a physical cluster. Scopes names, provides a boundary for RBAC, quotas, and network policies.

**ResourceQuota** — a namespace-level object that caps the **total** resource consumption within that namespace.

**LimitRange** — a namespace-level object that sets **default, minimum, and maximum** resource values for individual Pods/Containers within that namespace.

## 2. Why do we need them?

**Without Namespaces**, every object lives in one flat space — naming collisions, no logical grouping for RBAC, no way to apply different policies.

**Without ResourceQuotas**, a single misconfigured Deployment could consume all cluster resources — the classic "noisy neighbor" problem.

**Without LimitRanges**, Pods with no `resources` block get scheduled with no resource accounting.

## 3. Where is it used?

- **Multi-team clusters**: `team-a-dev`, `team-a-prod`, `team-b-dev` namespaces
- **Environment separation**: `dev`, `staging`, `prod`
- **Platform-as-a-service offerings**: each customer gets a namespace with quotas matching subscription tier
- **Default namespaces**: `default`, `kube-system`, `kube-public`, `kube-node-lease`

## 4. How does it work internally?

```
+--------------------------------- Cluster ---------------------------------+
|                                                                              |
|  +---------------- Namespace: team-a ----------------+                    |
|  |  ResourceQuota: max 10 CPU, 20Gi memory, 20 Pods     |                  |
|  |  LimitRange: default request 100m CPU/128Mi, max     |                  |
|  |              per-container 1 CPU/1Gi                  |                  |
|  |  Deployments, Services, ConfigMaps, Pods...           |                  |
|  +------------------------------------------------------+                  |
|                                                                              |
|  +---------------- Namespace: team-b ----------------+                    |
|  |  ResourceQuota: max 5 CPU, 10Gi memory, 10 Pods      |                  |
|  |  (completely independent from team-a's usage)         |                  |
|  +------------------------------------------------------+                  |
|                                                                              |
|  Cluster-scoped objects (NOT namespaced): Nodes, PersistentVolumes,        |
|  StorageClasses, ClusterRoles, Namespaces themselves                       |
+-------------------------------------------------------------------------+
```

**Enforcement mechanism:** When you `kubectl apply` a Pod into a namespace, the request passes through **admission control**:
1. **LimitRange admission** runs first — injects defaults if missing, rejects if out of min/max bounds
2. **ResourceQuota admission** runs next — checks if creating this Pod would push namespace total over any quota limit

## 5. How do we create and manage them?

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    env: production
    team: team-a
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "20"
    services: "10"
    services.loadbalancers: "2"          # max 2 LoadBalancer-type Services (cost control!)
    persistentvolumeclaims: "5"
    count/deployments.apps: "15"
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:                            # auto-applied if a container doesn't specify limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:                     # auto-applied if a container doesn't specify requests
      cpu: "100m"
      memory: "128Mi"
    max:                                 # hard ceiling
      cpu: "2"
      memory: "2Gi"
    min:                                 # hard floor
      cpu: "50m"
      memory: "64Mi"
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

**Essential kubectl commands:**
```bash
kubectl create namespace team-a
kubectl get namespaces
kubectl config set-context --current --namespace=team-a

kubectl apply -f resourcequota.yaml
kubectl get resourcequota -n team-a
kubectl describe resourcequota team-a-quota -n team-a     # shows Used vs Hard limits

kubectl apply -f limitrange.yaml
kubectl get limitrange -n team-a
kubectl describe limitrange team-a-limits -n team-a

kubectl get pods -n team-a
kubectl apply -f deployment.yaml -n team-a

kubectl delete namespace team-a    # WARNING: cascades and deletes EVERYTHING inside it
```

**Best practices:**
- Always pair **ResourceQuota** with **LimitRange**
- Set **object count quotas** (not just CPU/memory)
- Use **namespace labels** consistently
- Never put unrelated environments in the same namespace if quota/RBAC boundaries matter

**Common mistakes:**
- Setting a ResourceQuota with `requests.cpu` defined, but Pods don't specify `resources.requests` -> **all Pod creation fails**
- Assuming `kubectl delete namespace` is reversible
- Forgetting that quotas are **per-namespace**, not per-team-across-namespaces
- Confusing `LimitRange` max with `ResourceQuota` hard limit

**Debugging a rejected Pod:**
```bash
kubectl describe resourcequota team-a-quota -n team-a   # see current Used vs Hard
kubectl describe limitrange team-a-limits -n team-a     # see min/max/default bounds
```

## 6. When should we use it?

- **Namespaces:** any cluster shared by more than one team, environment, or workload category
- **ResourceQuota:** any multi-tenant cluster
- **LimitRange:** any namespace where Pods might be created without explicit resource specs

## 7. When should we NOT use it?

- Single-team, single-purpose small clusters may not need elaborate structures
- Namespaces are a soft boundary — true hard multi-tenancy needs separate clusters or node-level isolation (gVisor, Kata Containers)

## 8. Comparison Table

| Aspect | Namespace | ResourceQuota | LimitRange |
|---|---|---|---|
| Scope | Logical partition | Total usage across namespace | Per-container/Pod bounds |
| Enforces | Naming, RBAC, NetworkPolicy scope | Sum limits (CPU, memory, object counts) | Min/max/default per object |
| Without it | Flat namespace, naming collisions | One team can consume entire cluster | Pods can have no resource accounting |

## 9. Interview Questions

**Beginner**
- What's the difference between a namespaced and a cluster-scoped resource?
- What happens if you delete a namespace?

**Intermediate**
- A ResourceQuota exists with `requests.cpu`/`requests.memory` set. A user tries to create a Pod with no `resources` block. What happens, and why? How does a LimitRange fix this?
- Explain the difference between what ResourceQuota and LimitRange each enforce.

**Advanced / Scenario**
- Design a namespace/quota/limitrange strategy for a platform with 3 tiers of customers sharing one cluster.
- A team reports Pods rejected with "exceeded quota" errors, but `kubectl describe resourcequota` shows usage well below the hard limit. What are possible explanations?

## 10. Mini Quiz

1. True/False: PersistentVolumes are namespaced objects. -> **False**
2. If a ResourceQuota sets `requests.memory` but a Pod spec omits memory requests entirely, what happens without a LimitRange default? -> **Pod creation is rejected**
3. What's the difference between LimitRange's `max` and ResourceQuota's `hard` limit for CPU? -> LimitRange bounds a single container; ResourceQuota bounds the sum across the entire namespace

## Hands-on Lab

```bash
kubectl create namespace lab-ns
kubectl apply -f limitrange.yaml -n lab-ns
kubectl apply -f resourcequota.yaml -n lab-ns
kubectl run test-pod --image=nginx -n lab-ns
kubectl get pod test-pod -n lab-ns -o jsonpath='{.spec.containers[0].resources}'
kubectl describe resourcequota team-a-quota -n lab-ns
kubectl run test-pod-2 --image=nginx -n lab-ns
kubectl run test-pod-3 --image=nginx -n lab-ns
```

## Summary

Namespaces carve one cluster into many logical ones; ResourceQuota is the total-consumption governor per namespace; LimitRange is the per-object bounds-setter that makes quotas actually enforceable.


# Lesson 14: ConfigMaps & Secrets

## 1. What is it?

**ConfigMap** — stores **non-sensitive configuration data** as key-value pairs, injected into Pods as environment variables, command-line args, or mounted files.

**Secret** — structurally almost identical to a ConfigMap, but intended for **sensitive data**. Values are stored **base64-encoded** (not encrypted by default).

## 2. Why do we need them?

**Without ConfigMaps**, you'd hardcode config into images or use fragile shell scripts — breaking "build once, deploy anywhere."

**Without Secrets**, credentials would need to live in plain environment variables baked into YAML.

**Industry motivation:** The **12-factor app** methodology calls for config to be stored in the environment, separate from code.

## 3. Where is it used?

- **Database connection strings, feature flags, log levels** -> ConfigMaps
- **Database passwords, API keys, TLS certificates, Docker registry credentials, OAuth tokens** -> Secrets
- **cert-manager** writes issued TLS certs into Secrets automatically
- **Service accounts** automatically get a token stored as a Secret

## 4. How does it work internally?

```
+--------------------- ConfigMap: app-config ---------------------+
|  data:                                                              |
|    LOG_LEVEL: "info"                                                |
|    DB_HOST: "postgres-svc"                                          |
|    app.properties: |                                                |
|      timeout=30                                                     |
|      retries=3                                                      |
+------------------------------+------------------------------------+
                                |  injected via 3 possible mechanisms
        +-----------------------+-----------------------+
        v                       v                       v
+---------------+     +-----------------------+   +---------------------+
| Env variable    |     | Volume mount            |   | Command-line arg      |
| LOG_LEVEL=info  |     | /etc/config/             |   | via valueFrom in       |
| (via envFrom or |     |   app.properties         |   | args: [] referencing   |
|  env.valueFrom) |     | (each key = a file)      |   | an env var             |
+---------------+     +-----------------------+   +---------------------+
```

**Secret internal storage:** stored in **etcd**, base64-encoded — this is **encoding, not encryption**. For actual encryption:
- Enable **encryption at rest** for etcd
- Use **external secret managers** (HashiCorp Vault, AWS Secrets Manager, External Secrets Operator)

**Live update behavior:** if a ConfigMap/Secret is mounted as a **volume**, updates are eventually reflected (~1 minute) **without restarting the Pod**. Environment variables injected via `env`/`envFrom` are **NOT updated live**.

## 5. How do we create and manage them?

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  DB_HOST: "postgres-svc"
  app.properties: |                  # multi-line file content
    timeout=30
    retries=3
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque                          # generic key-value secret (most common type)
data:
  username: YWRtaW4=                  # base64-encoded "admin"
  password: c3VwZXJzZWNyZXQ=          # base64-encoded "supersecret"
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:1.0
    env:
    - name: LOG_LEVEL                       # single key from ConfigMap -> single env var
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_PASSWORD                      # single key from Secret -> single env var
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    envFrom:
    - configMapRef:
        name: app-config                     # ALL keys from ConfigMap become env vars
    - secretRef:
        name: db-secret                      # ALL keys from Secret become env vars
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config                 # each ConfigMap key becomes a file here
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400                      # restrict file permissions
```

**Common Secret types:**
```yaml
type: Opaque                          # generic (default)
type: kubernetes.io/tls               # TLS cert + key (tls.crt, tls.key)
type: kubernetes.io/dockerconfigjson  # private registry credentials
type: kubernetes.io/basic-auth        # username/password
type: kubernetes.io/service-account-token  # SA token
```

**Essential kubectl commands:**
```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=info --from-literal=DB_HOST=postgres-svc
kubectl create configmap app-config --from-file=app.properties
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=supersecret
kubectl create secret tls my-tls-secret --cert=path/to/cert.crt --key=path/to/key.key
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=user --docker-password=pass

kubectl get configmap app-config -o yaml
kubectl get secret db-secret -o yaml           # values shown base64-encoded, NOT plaintext
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d   # manually decode to verify

kubectl describe configmap app-config          # safe - shows values
kubectl describe secret db-secret              # SAFE by default - does NOT print values

kubectl delete configmap app-config
kubectl delete secret db-secret
```

**Best practices:**
- **Never commit raw Secret YAML to git** — use Sealed Secrets, SOPS, or an external secrets operator
- Enable **etcd encryption at rest** in any production cluster
- Use **RBAC** to tightly restrict who/what can `get`/`list` Secrets
- Mount Secrets as **volumes with restrictive `defaultMode`** rather than environment variables where possible
- Use the **"immutable ConfigMap/Secret" pattern** (`immutable: true`) for config that shouldn't change without a full Pod replacement

**Common mistakes:**
- Assuming base64 = encryption
- Updating a ConfigMap and expecting env-var-injected Pods to pick up the change automatically — **they won't**
- Committing Secret manifests to version control in plaintext-adjacent form
- Forgetting `readOnly: true` on Secret volume mounts

**Debugging:**
```bash
kubectl exec -it app-pod -- env | grep LOG_LEVEL          # verify env var actually landed
kubectl exec -it app-pod -- cat /etc/config/app.properties  # verify volume-mounted file content
kubectl describe pod app-pod                                # check Events for "configmap not found" etc.
```

## 6. When should we use each?

- **ConfigMap:** non-sensitive settings
- **Secret:** anything sensitive
- **Volume mount vs env var:** volume mounts for config that might change live; env vars for simple values read once at startup

## 7. When should we NOT use them?

- Don't use ConfigMaps/Secrets for **large binary blobs or datasets** — 1MiB size limit per object
- Don't use raw Kubernetes Secrets as your **only** secrets management strategy in security-sensitive production

## 8. Comparison Table: ConfigMap vs Secret

| Aspect | ConfigMap | Secret |
|---|---|---|
| Intended data | Non-sensitive config | Sensitive data |
| Storage encoding | Plaintext | Base64 (NOT encryption) |
| `kubectl describe` shows values? | Yes | No (hidden by default) |
| Size limit | 1MiB | 1MiB |
| Real security requires | Nothing extra | RBAC + etcd encryption + external secret manager |

## 9. Interview Questions

**Beginner**
- What's the actual difference in how Kubernetes stores ConfigMap vs Secret data?
- Name the three ways to inject a ConfigMap's data into a Pod.

**Intermediate**
- If you update a ConfigMap mounted as a volume, does the Pod see the change without restarting? What about `envFrom`?
- Why is base64 encoding NOT a security control?

**Advanced / Scenario**
- Design a secrets management strategy for a production fintech application that must pass a security audit.
- A Pod's ConfigMap-mounted file hasn't updated 10 minutes after you changed the ConfigMap. What are possible causes? (Hint: `subPath` mounts do NOT get live updates at all.)

## 10. Mini Quiz

1. True/False: Base64 encoding in Secrets provides real encryption. -> **False**
2. Which injection method updates live when the underlying ConfigMap changes? -> **Volume mount**
3. What K8s feature encrypts Secret data at rest in etcd? -> **Encryption at rest (KMS provider configuration)**

## Hands-on Lab

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=info
kubectl create secret generic db-secret --from-literal=password=supersecret123
kubectl apply -f pod.yaml
kubectl exec -it app-pod -- env | grep LOG_LEVEL
kubectl exec -it app-pod -- cat /etc/secrets/password
kubectl edit configmap app-config      # change LOG_LEVEL to "debug"
sleep 70
kubectl exec -it app-pod -- cat /etc/config/LOG_LEVEL   # should reflect "debug"
kubectl exec -it app-pod -- env | grep LOG_LEVEL          # should STILL show "info"
```

## Summary

ConfigMaps/Secrets decouple config from images, implementing the 12-factor app principle natively.

# Lesson 15: Volumes, PV, PVC & Storage Classes

## 1. What is it?

**Volume** — storage attached to a Pod that outlives individual **container** restarts within that Pod.

**PersistentVolume (PV)** — a cluster-level storage resource, provisioned manually or dynamically, representing an actual piece of storage — independent of any Pod's lifecycle.

**PersistentVolumeClaim (PVC)** — a **request** for storage made by a user/Pod, which Kubernetes **binds** to a matching PV. Pods reference PVCs, never PVs directly.

**StorageClass** — a template that defines **how** to dynamically provision PVs on-demand.

## 2. Why do we need them?

**Without Volumes**, every container restart wipes local state.

**Without PV/PVC abstraction**, application YAML would need to hardcode exactly which physical disk to use.

**Without StorageClasses**, every PVC would require a human admin to manually pre-provision a matching PV.

**Industry motivation:** The PV/PVC split implements separation of concerns: developers ask for storage abstractly; infra teams (or the cloud provider) handle the how.

## 3. Where is it used?

- **Databases** (StatefulSets) — each replica's PVC backed by a real cloud disk
- **File upload storage**
- **Shared configuration/data** across multiple Pods via `ReadWriteMany`
- **Ephemeral scratch space** — `emptyDir` volumes

## 4. How does it work internally?

```
+------------------ Dynamic Provisioning Flow ------------------+
|                                                                   |
|  1. User creates a PVC:                                         |
|     "I need 20Gi, ReadWriteOnce, storageClassName: fast-ssd"    |
|                          |                                       |
|                          v                                       |
|  2. PVC controller sees no matching PV exists yet                |
|                          |                                       |
|                          v                                       |
|  3. StorageClass "fast-ssd" defines provisioner:                 |
|     ebs.csi.aws.com (AWS EBS CSI driver)                          |
|                          |                                       |
|                          v                                       |
|  4. CSI driver provisions an ACTUAL EBS volume in AWS             |
|                          |                                       |
|                          v                                       |
|  5. A PersistentVolume object is auto-created representing        |
|     that EBS volume, and BOUND to the PVC (1:1 relationship)     |
|                          |                                       |
|                          v                                       |
|  6. Pod references the PVC by name -> kubelet mounts the          |
|     underlying volume into the container via the CSI driver       |
+--------------------------------------------------------------------+

     PVC (namespaced) <---- 1:1 bind ----> PV (cluster-scoped)
          |
          | referenced by
          v
        Pod
```

**Volume types (non-persistent, Pod-lifetime only):**
- `emptyDir` — created empty when the Pod starts, deleted when the Pod is removed
- `hostPath` — mounts a path from the **node's** filesystem directly
- `configMap`/`secret`

**Access Modes:**

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Mounted read-write by a single node |
| `ReadOnlyMany` (ROX) | Mounted read-only by many nodes simultaneously |
| `ReadWriteMany` (RWX) | Mounted read-write by many nodes simultaneously (needs NFS/EFS) |
| `ReadWriteOncePod` (RWOP) | Mounted read-write by a single **Pod** |

**Reclaim Policies:**
- `Delete` (default for dynamically provisioned volumes) — the actual disk is destroyed
- `Retain` — the disk survives, requires manual admin cleanup/re-binding

## 5. How do we create and manage them?

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com          # which CSI driver handles this
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Retain                  # override default "Delete" - protect data
volumeBindingMode: WaitForFirstConsumer  # delay provisioning until a Pod actually needs it
allowVolumeExpansion: true              # allow PVCs using this class to be resized later
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:                                   # example: NFS-backed volume
    server: nfs-server.example.com
    path: /exports/data
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
  - name: db
    image: postgres:16
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc                 # Pod references the PVC, never the PV directly
```

emptyDir example:
```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: cache
      mountPath: /tmp/cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 1Gi
```

**Essential kubectl commands:**
```bash
kubectl get storageclass
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
kubectl get pvc                               # check STATUS: Pending -> Bound
kubectl get pv                                # see the auto-provisioned PV
kubectl describe pvc data-pvc                 # check Events if stuck Pending

kubectl patch pvc data-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

kubectl delete pvc data-pvc                   # triggers reclaim policy (Delete or Retain)
kubectl delete pv manual-pv
```

**Best practices:**
- Use `reclaimPolicy: Retain` for any storage holding data you cannot afford to lose
- Use `volumeBindingMode: WaitForFirstConsumer` — delays disk provisioning until a Pod is scheduled, ensuring same zone
- Enable `allowVolumeExpansion: true`
- Never use `hostPath` for anything except node-level infrastructure

**Common mistakes:**
- PVC stuck `Pending` forever — no StorageClass matches, provisioner not installed, unsupported access mode
- Forgetting `WaitForFirstConsumer` -> zone-mismatch failure
- Deleting a PVC accidentally with `reclaimPolicy: Delete` -> permanent data loss
- Assuming PVCs can shrink — you can only expand

**Debugging a stuck PVC:**
```bash
kubectl describe pvc data-pvc          # check Events
kubectl get storageclass               # confirm the referenced StorageClass exists
kubectl describe storageclass fast-ssd # confirm provisioner is correct
kubectl get pods -n kube-system | grep csi   # confirm CSI driver Pods are Running
```

## 6. When should we use each?

- **emptyDir:** temporary scratch space, inter-container sharing
- **PVC + dynamic StorageClass:** standard modern approach for persistent data
- **Static PV:** legacy/on-prem environments without dynamic provisioner
- **hostPath:** only for node-level infrastructure agents

## 7. When should we NOT use them?

- Don't use `emptyDir` for anything that needs to survive Pod deletion
- Don't use `hostPath` for regular application data
- Don't request `ReadWriteMany` unless your storage backend supports it

## 8. Comparison Table

| Aspect | emptyDir | PV/PVC | hostPath |
|---|---|---|---|
| Survives container restart | Yes | Yes | Yes |
| Survives Pod deletion | **No** | **Yes** | Depends |
| Survives node failure | No | Yes (cloud-backed) | No |
| Portable across nodes | Yes (recreated) | Yes (reattaches) | **No** |
| Typical use | Scratch space | Databases, persistent app data | Node-level agents only |

## 9. Interview Questions

**Beginner**
- What's the difference between a PV and a PVC?
- What happens to `emptyDir` data when a Pod is deleted?

**Intermediate**
- Explain the full flow from creating a PVC to a Pod having a mounted, writable disk, in dynamic provisioning.
- Why does `volumeBindingMode: WaitForFirstConsumer` matter for zonal cloud storage?

**Advanced / Scenario**
- A StatefulSet Pod is stuck `Pending` with its PVC also `Pending`. What else could be wrong beyond StorageClass/CSI driver?
- Design a backup/disaster-recovery strategy for StatefulSet PVCs backed by cloud block storage.

## 10. Mini Quiz

1. True/False: A Pod can reference a PersistentVolume directly, bypassing the PVC. -> **False** (discouraged)
2. What reclaim policy prevents data loss when a PVC is deleted? -> `Retain`
3. Which access mode allows multiple nodes to mount the same volume read-write simultaneously? -> `ReadWriteMany`

## Hands-on Lab

```bash
kubectl get storageclass
kubectl apply -f pvc.yaml
kubectl get pvc data-pvc -w
kubectl get pv
kubectl apply -f pod.yaml
kubectl exec -it db-pod -- sh -c "echo hello > /var/lib/postgresql/data/test.txt"
kubectl delete pod db-pod
kubectl apply -f pod.yaml
kubectl exec -it db-pod -- cat /var/lib/postgresql/data/test.txt   # "hello" survives!
```

## Summary

PV/PVC/StorageClass decouple developers (abstract requests) from infra teams (actual provisioning).


# Lesson 16: CSI Drivers (Container Storage Interface)

## 1. What is it?

The **Container Storage Interface (CSI)** is a standardized API that lets storage vendors write **plugins** that Kubernetes can use to provision, attach, mount, and manage storage — without vendor-specific code living inside Kubernetes' core codebase.

## 2. Why do we need it?

**Without CSI**, adding support for a new storage backend meant contributing to and waiting on the core Kubernetes project — vendors couldn't iterate independently.

**Industry motivation:** CSI decouples storage vendor development entirely from Kubernetes releases — the same architectural motivation as CNI (networking) and CRI (container runtimes).

## 3. Where is it used?

- **Every managed cloud Kubernetes offering**: `ebs.csi.aws.com` (EKS), `pd.csi.storage.gke.io` (GKE), `disk.csi.azure.com` (AKS)
- **On-prem/enterprise storage**: NetApp Trident, Portworx, Ceph CSI, Longhorn
- Referenced directly in **StorageClass** `provisioner` field

## 4. How does it work internally?

```
+----------------------- CSI Driver Architecture -----------------------+
|                                                                          |
|  Deployed as TWO components (typical pattern):                         |
|                                                                          |
|  +-----------------------------+                                       |
|  |  CSI Controller Pod           |  <- runs as a Deployment              |
|  |  (usually 1, on any node)     |     Handles: CreateVolume,           |
|  |  - external-provisioner        |     DeleteVolume, Attach/Detach,    |
|  |  - external-attacher           |     Snapshot creation                |
|  |  - CSI driver container        |                                    |
|  +-----------------------------+                                       |
|                                                                          |
|  +-----------------------------+                                       |
|  |  CSI Node Plugin (DaemonSet)  |  <- ONE per node                     |
|  |  - node-driver-registrar       |     Handles: mounting the already- |
|  |  - CSI driver container        |     attached volume into the        |
|  |  (runs on EVERY node)          |     actual Pod's filesystem path    |
|  +-----------------------------+                                       |
+---------------------------------------------------------------------+

  Flow when a Pod needs a volume:
  1. PVC created -> external-provisioner (Controller) calls CSI CreateVolume RPC
  2. Cloud API creates the actual disk (e.g., EBS volume)
  3. external-attacher (Controller) calls CSI ControllerPublishVolume -> attaches
     disk to the target NODE (at the cloud API level)
  4. kubelet on that node calls the CSI Node Plugin's NodeStageVolume/NodePublishVolume
     -> actually mounts the disk into the Pod's container filesystem path
```

**Key insight:** this mirrors the DaemonSet pattern from Lesson 8 — the **Node Plugin runs on every node** (mounting must happen locally), while the **Controller Plugin runs centrally**.

**CSI defines a small set of gRPC calls** every driver must implement: `CreateVolume`, `DeleteVolume`, `ControllerPublishVolume`, `ControllerUnpublishVolume`, `NodeStageVolume`, `NodePublishVolume`, `NodeUnpublishVolume` — plus optional `CreateSnapshot`.

## 5. How do we create and manage them?

```bash
kubectl get csidrivers
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system -l app=ebs-csi-node       # should be one per node
kubectl get volumeattachments
kubectl describe csidriver ebs.csi.aws.com
```

VolumeSnapshot (a CSI-enabled feature):
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Retain
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: data-pvc
```

Restoring a PVC from a snapshot:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  storageClassName: fast-ssd
  dataSource:
    name: data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**Best practices:**
- Verify the CSI driver's node plugin (DaemonSet) is `Running` on every node before troubleshooting
- Use **VolumeSnapshots** for backup/restore workflows
- Check driver-specific documentation for supported `accessModes` and `parameters`

**Common mistakes:**
- Assuming any CSI driver supports snapshots/resizing/RWX by default
- Not noticing the Node Plugin DaemonSet isn't running on a specific node (taint it doesn't tolerate)
- Manually deleting a `VolumeAttachment` object while a Pod is still using it

**Debugging a volume mount failure:**
```bash
kubectl describe pod <stuck-pod>              # look for "FailedMount"/"FailedAttachVolume"
kubectl get volumeattachments                  # confirm attached to the right node
kubectl logs -n kube-system <csi-node-plugin-pod> -c <driver-container>
kubectl get pods -n kube-system -l app=ebs-csi-node -o wide
```

## 6. When should we use it?

Always used indirectly the moment you use PVCs with dynamic provisioning — no scenario in a modern cluster avoids it.

## 7. When should we NOT use it?

Not applicable — CSI is the standard, not an optional pattern. The only decision is which CSI driver to install.

## 8. Comparison Table: In-tree Plugins (legacy) vs CSI

| Aspect | In-tree Plugins (deprecated) | CSI |
|---|---|---|
| Location | Compiled into Kubernetes core | Separate, independently deployed Pods |
| Update cadence | Tied to Kubernetes releases | Independent |
| Isolation | Runs in-process with kubelet (risky) | Runs as separate containers |
| Current status | Being removed across K8s versions | The standard, actively developed approach |

## 9. Interview Questions

**Beginner**
- What problem does CSI solve compared to in-tree storage plugins?
- What two main components make up a typical CSI driver deployment?

**Intermediate**
- Why does the CSI Node Plugin need to run as a DaemonSet specifically?
- Walk through what happens, component by component, from PVC creation to a mounted volume.

**Advanced / Scenario**
- Pod stuck `ContainerCreating` with a `FailedMount` event, but PVC shows `Bound` and PV exists. Full diagnostic path?
- Migrate persistent data from AWS EBS to Portworx with minimal downtime — how would VolumeSnapshots factor in?

## 10. Mini Quiz

1. True/False: You must write custom Kubernetes core code to add support for a new storage backend today. -> **False**
2. Which CSI component runs as a DaemonSet, and why? -> **Node Plugin** — mounting must happen locally
3. What object tracks which node a given volume is currently attached to? -> `VolumeAttachment`

## Hands-on Lab

```bash
kubectl get csidrivers
kubectl get pods -n kube-system -l app=ebs-csi-node -o wide
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl apply -f pvc.yaml
kubectl get volumeattachments -w
```

## Summary

CSI is the plumbing that makes PV/PVC/StorageClass work against real storage backends — standardizing "create, attach, mount" into a vendor-neutral gRPC interface.


# Lesson 17: Networking, DNS & CNI

## 1. What is it?

**Kubernetes Networking** is governed by a foundational set of rules every cluster must satisfy. **CNI (Container Network Interface)** is the standardized plugin API that lets networking vendors implement Pod networking. **Cluster DNS (CoreDNS)** is the internal DNS service that resolves Service/Pod names to IPs.

## 2. Why do we need them?

**Without a defined networking model**, every implementation could behave differently. **Without CNI**, every network vendor would need Kubernetes core changes. **Without cluster DNS**, every Service consumer would need to hardcode ClusterIPs.

## 3. Where is it used?

- Every single Pod-to-Pod, Pod-to-Service interaction
- **Network policy enforcement** implemented by the CNI plugin
- **Service meshes** layer on top of, or partially replace, base CNI networking

## 4. How does it work internally?

**The Kubernetes Networking Model — 3 fundamental rules:**
1. Every Pod gets its **own unique IP** — no port-mapping/NAT between Pods
2. **All Pods can communicate with all other Pods** across all nodes, without NAT, by default
3. **Agents on a node** (like kubelet) can communicate with all Pods on that node

```
+---------------------------- Pod-to-Pod Communication ----------------------------+
|                                                                                     |
|   Node A (10.0.1.5)                        Node B (10.0.2.8)                     |
|   +---------------------+                 +---------------------+               |
|   |  Pod X: 10.244.1.5    |                 |  Pod Y: 10.244.2.9   |               |
|   |  (via CNI-managed      |                 |  (via CNI-managed     |               |
|   |   virtual interface)   |                 |   virtual interface)  |               |
|   +----------+----------+                 +----------+----------+               |
|              |                                          |                          |
|              v                                          v                          |
|      +---------------+                          +---------------+                |
|      | CNI bridge/veth|                          | CNI bridge/veth|                |
|      +-------+-------+                          +-------+-------+                |
|              |                                          |                          |
|              +---------- Overlay network or ------------+                          |
|                          direct routing (BGP, VXLAN)                              |
|                          between nodes, managed entirely                           |
|                          by the CNI plugin                                         |
+-----------------------------------------------------------------------------------+
```

**How CNI plugs in:**
1. kubelet creates the Pod's network namespace
2. kubelet invokes the **CNI plugin binary** with a config file
3. The plugin assigns an IP, creates a veth pair, sets up routes
4. Cross-node routing implementations:
   - **Overlay networks** (VXLAN, Flannel's default) — encapsulate traffic
   - **Native/BGP routing** (Calico's default) — direct Layer 3 routing
   - **eBPF-based** (Cilium) — bypasses much of the traditional kernel networking stack

**DNS resolution flow (CoreDNS):**
```
Pod queries: backend-svc.default.svc.cluster.local
        |
        v
Pod's /etc/resolv.conf points to CoreDNS's ClusterIP (a Service itself!)
        |
        v
CoreDNS Pod resolves the name by querying the Kubernetes API
        |
        v
Returns the Service's ClusterIP (or Pod IPs for headless Services)
```

**DNS naming convention:**
```
<service-name>.<namespace>.svc.cluster.local     -> Service DNS
<pod-ip-dashes>.<namespace>.pod.cluster.local     -> Pod DNS (rarely used)
<pod-hostname>.<service-name>.<namespace>.svc.cluster.local   -> StatefulSet Pod DNS
```

## 5. How do we create and manage it?

```bash
kubectl get pods -n kube-system | grep -E 'calico|cilium|flannel|weave'
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
kubectl describe node <node-name>    # check "PodCIDR"

kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns

kubectl run dns-test --rm -it --image=busybox -- nslookup backend-svc
kubectl run dns-test --rm -it --image=busybox -- nslookup backend-svc.default.svc.cluster.local
kubectl exec -it my-pod -- cat /etc/resolv.conf
```

Customizing Pod DNS behavior:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - custom.svc.cluster.local
  containers:
  - name: app
    image: my-app:1.0
```

**Best practices:**
- Choose a CNI plugin based on real requirements: Calico for NetworkPolicy + BGP at scale; Cilium for eBPF performance; Flannel for simplicity
- Monitor CoreDNS resource usage on large clusters
- Use **NodeLocal DNSCache** in large clusters to reduce load on central CoreDNS

**Common mistakes:**
- Choosing a CNI plugin without checking NetworkPolicy support (Flannel by default does NOT enforce it)
- Assuming Pod IPs are routable outside the cluster
- Misdiagnosing DNS resolution failures as app bugs
- Overlapping CNI Pod CIDR with existing VPC ranges

**Debugging network connectivity issues:**
```bash
kubectl exec -it my-pod -- nslookup backend-svc
kubectl exec -it my-pod -- wget -qO- <pod-ip>:8080
kubectl get pods -n kube-system -o wide | grep <cni-plugin-name>
kubectl get networkpolicy -A
kubectl logs -n kube-system -l k8s-app=kube-dns
```

## 6. When should we use which CNI plugin?

| Need | Recommended CNI |
|---|---|
| Strong NetworkPolicy enforcement, mature ecosystem | **Calico** |
| Maximum performance, deep eBPF-based observability | **Cilium** |
| Simplicity, small clusters | **Flannel** |
| Multi-cloud/hybrid networking | **Cilium** or **Calico** |

## 7. When should we NOT rely on default CNI behavior?

- Need NetworkPolicy enforcement but chose a non-supporting plugin
- Need multi-cluster service mesh networking — base CNI alone doesn't solve this

## 8. Comparison Table: CNI Plugin Approaches

| Aspect | Overlay (Flannel/VXLAN) | Native Routing (Calico/BGP) | eBPF (Cilium) |
|---|---|---|---|
| Performance | Some encapsulation overhead | Near-native | Highest |
| NetworkPolicy support | No (needs Calico paired) | Yes | Yes, advanced |
| Setup complexity | Low | Medium | Medium-high |
| Observability | Basic | Good | Excellent |

## 9. Interview Questions

**Beginner**
- What are the three fundamental rules of the Kubernetes networking model?
- What does CoreDNS resolve, and where does it run?

**Intermediate**
- Why doesn't every CNI plugin support NetworkPolicy enforcement?
- Explain the difference between overlay networking and native BGP routing.

**Advanced / Scenario**
- Pods across two nodes can't reach each other, but same-node Pods communicate fine. Diagnose fully.
- Intermittent DNS resolution failures cluster-wide during peak traffic. What feature reduces CoreDNS load?

## 10. Mini Quiz

1. True/False: All CNI plugins enforce NetworkPolicy by default. -> **False**
2. What component resolves `backend-svc.default.svc.cluster.local`? -> **CoreDNS**
3. Name one CNI plugin that uses eBPF. -> **Cilium**

## Hands-on Lab

```bash
kubectl get pods -n kube-system -o wide
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl run pod-a --image=busybox --command -- sleep 3600
kubectl run pod-b --image=busybox --command -- sleep 3600
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')
kubectl exec -it pod-a -- ping -c 3 $POD_B_IP
kubectl exec -it pod-a -- nslookup kubernetes.default
```

## Summary

CNI standardizes Pod networking; CoreDNS makes Service names usable — together fulfilling Kubernetes' flat-network, no-NAT guarantee.


# Lesson 18: kube-proxy

## 1. What is it?

**kube-proxy** is a node-level component (running as a DaemonSet) responsible for implementing the actual **traffic routing rules** that make Services work. It watches the API server for Service/EndpointSlice changes and programs the node's networking stack.

## 2. Why do we need it?

**Without kube-proxy**, Services would be nothing but metadata — nothing would actually route packets to a Service's virtual ClusterIP.

## 3. Where is it used?

Runs on **every node** in every Kubernetes cluster (unless replaced by a CNI plugin's own implementation, e.g., Cilium's eBPF mode).

## 4. How does it work internally?

```
+--------------------------- kube-proxy on Node A ---------------------------+
|                                                                               |
|  1. Watches API server for Service + EndpointSlice changes (continuous)     |
|                                                                               |
|  2. On any change, reprograms this node's routing rules:                    |
|                                                                               |
|     +-------------+        +-------------+       +-------------+           |
|     |  iptables    |   OR   |    IPVS      |  OR   |    eBPF      |         |
|     |  mode        |        |    mode      |       |  (Cilium,    |         |
|     |  (legacy      |        |  (higher     |       |   replaces    |       |
|     |   default)    |        |  performance)|       |  kube-proxy   |       |
|     +-------------+        +-------------+       |  entirely)    |         |
|                                                     +-------------+         |
|                                                                               |
|  3. Incoming packet to ClusterIP:port -> rules rewrite destination to        |
|     a chosen backend Pod IP:port (round-robin/random/session-affinity)      |
+------------------------------------------------------------------------------+
```

**iptables mode (long-time default):**
- Writes a chain of `iptables` rules for every Service/Endpoint
- Packet matching ClusterIP gets DNAT to a random backend Pod
- Downside: roughly linear performance with Service count

**IPVS mode (recommended for larger clusters):**
- Uses the Linux kernel's IP Virtual Server, hash tables -> O(1) lookup
- Supports more sophisticated load balancing algorithms

**eBPF mode (Cilium replacing kube-proxy):**
- Bypasses iptables/IPVS entirely, packet routing decisions directly in kernel via eBPF
- Lower latency/CPU overhead, richer observability (Hubble)

## 5. How do we create and manage it?

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl get configmap kube-proxy -n kube-system -o yaml
kubectl logs -n kube-system <kube-proxy-pod-name>
kubectl debug node/<node-name> -it --image=busybox -- chroot /host iptables -t nat -L -n | grep <service-name>
kubectl debug node/<node-name> -it --image=busybox -- chroot /host ipvsadm -Ln
```

Switching modes:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"          # or "iptables" (default) or "" (auto)
```

**Best practices:**
- Use IPVS mode for clusters with 1000+ Services
- Consider eBPF-based CNI plugins for the most performance-sensitive clusters
- Never manually edit iptables/IPVS rules that kube-proxy manages

**Common mistakes:**
- Diagnosing "Service not reachable" without checking kube-proxy Pod health first
- Assuming a Service issue is a CNI issue (or vice versa) — CNI handles Pod-to-Pod connectivity; kube-proxy handles Service routing on top of that
- Running iptables mode at very high Service counts without realizing the performance cliff

**Debugging Service connectivity by isolating kube-proxy vs CNI:**
```bash
kubectl exec -it pod-a -- ping <pod-b-direct-ip>        # tests CNI
kubectl exec -it pod-a -- wget -qO- <service-clusterip>:<port>   # tests kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl logs -n kube-system <kube-proxy-pod-on-that-node>
```

## 6. When should we use which mode?

| Cluster Size / Need | Recommended Mode |
|---|---|
| Small/medium clusters, simplicity | iptables (default) |
| Large clusters (1000+ Services) | IPVS |
| Maximum performance | eBPF (Cilium kube-proxy replacement) |

## 7. When should we NOT rely on default kube-proxy behavior?

- Large scale sticking with iptables mode without evaluating IPVS/eBPF
- Need L7-aware load balancing — beyond kube-proxy's scope, need a service mesh

## 8. Comparison Table: iptables vs IPVS vs eBPF

| Aspect | iptables | IPVS | eBPF (Cilium) |
|---|---|---|---|
| Lookup complexity | O(n) | O(1) | O(1), kernel-level |
| Load balancing algorithms | Random only | Round-robin, least-conn, hash | Highly flexible |
| Scalability ceiling | Degrades at high Service count | Scales well | Best at extreme scale |
| Maturity/default status | Long-time default | Available, opt-in | Requires Cilium |

## 9. Interview Questions

**Beginner**
- What is kube-proxy's core responsibility?
- What objects does kube-proxy watch?

**Intermediate**
- Practical performance difference between iptables and IPVS mode?
- How would you tell whether a connectivity issue is CNI or kube-proxy?

**Advanced / Scenario**
- 3,000 Services experiencing latency spikes correlating with Service create/delete events. Hypothesis and recommendation?
- Evaluating Cilium kube-proxy replacement mode — what should you validate still works?

## 10. Mini Quiz

1. True/False: kube-proxy runs as a single centralized component. -> **False** — DaemonSet, one per node
2. Which mode offers O(1) lookup regardless of Service count? -> **IPVS**
3. Direct Pod-to-Pod ping works but Service ClusterIP fails — fault? -> **kube-proxy**

## Hands-on Lab

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --port=80
kubectl run test --rm -it --image=busybox -- wget -qO- web:80
WEB_POD_IP=$(kubectl get pods -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl run test2 --rm -it --image=busybox -- ping -c 2 $WEB_POD_IP
kubectl run test3 --rm -it --image=busybox -- wget -qO- web:80
```

## Summary

kube-proxy turns a Service's abstract ClusterIP into real, working packet routing on every node.


# Lesson 19: RBAC, Service Accounts, Authentication & Authorization

## 1. What is it?

**Authentication (AuthN)** — establishing *who* is making a request. **Authorization (AuthZ)** — determining *what* that identity is allowed to do (RBAC is Kubernetes' primary mechanism). **RBAC** — `Role`/`ClusterRole` (permissions) bound to identities via `RoleBinding`/`ClusterRoleBinding`. **ServiceAccount** — identity for processes running inside Pods.

## 2. Why do we need it?

**Without AuthN**, the API server couldn't distinguish legitimate requests. **Without AuthZ/RBAC**, every authenticated identity would have equal, unrestricted access — a compromised Pod's token could delete the entire cluster.

**Industry motivation:** The principle of least privilege.

## 3. Where is it used?

- Every single API request
- CI/CD pipelines authenticate via a dedicated ServiceAccount with narrowly-scoped RBAC
- Monitoring tools use read-only, cluster-wide permissions
- Human users authenticate via cloud-IAM/OIDC, authorized via RBAC

## 4. How does it work internally?

```
+---------------------- Every API Request Flow ----------------------+
|                                                                        |
|   Request arrives at kube-apiserver                                  |
|            |                                                          |
|            v                                                          |
|   +-----------------+                                                 |
|   |  AUTHENTICATION   |  "WHO are you?"                                |
|   |  - Client certs    |  Checks: client cert, bearer token, OIDC     |
|   |  - Bearer tokens   |  -> produces username + group memberships     |
|   |  - OIDC             |                                              |
|   +--------+--------+                                                 |
|            | (if fails -> 401 Unauthorized)                            |
|            v                                                          |
|   +-----------------+                                                 |
|   |  AUTHORIZATION     |  "Are you ALLOWED to do this?"                 |
|   |  (RBAC)            |  Checks: does any Role/ClusterRole permit     |
|   |                    |  this verb+resource combo?                    |
|   +--------+--------+                                                 |
|            | (if fails -> 403 Forbidden)                               |
|            v                                                          |
|   +-----------------+                                                 |
|   | ADMISSION CONTROL |  "Should this SPECIFIC request be allowed/      |
|   |  (mutating +        |   modified?" - ResourceQuota, LimitRange,    |
|   |   validating)       |  PodSecurity admission, custom webhooks       |
|   +--------+--------+                                                 |
|            v                                                          |
|      Request processed, written to etcd                                |
+------------------------------------------------------------------------+
```

**RBAC's four core objects:**

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespaced | Defines permissions within one namespace |
| `ClusterRole` | Cluster-wide | Defines permissions across all namespaces, or on cluster-scoped resources |
| `RoleBinding` | Namespaced | Grants a Role (or ClusterRole) to a subject, scoped to one namespace |
| `ClusterRoleBinding` | Cluster-wide | Grants a ClusterRole across the entire cluster |

**Important subtlety:** you CAN bind a `ClusterRole` via a namespaced `RoleBinding` — a common pattern (reuse one ClusterRole across many namespaces).

**ServiceAccount mechanics:**
- Every namespace has a `default` ServiceAccount automatically
- Every Pod runs as some ServiceAccount (explicit or `default`)
- The token is automatically mounted (modern clusters use projected, time-bound tokens via `TokenRequest` API)

## 5. How do we create and manage them?

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: team-a
rules:
- apiGroups: [""]                      # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: team-a
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: team-a
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]                  # Nodes are cluster-scoped
  verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: Group
  name: system:monitoring-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: team-a
---
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-pod
  namespace: team-a
spec:
  serviceAccountName: monitoring-sa       # explicitly assign this SA (not "default")
  automountServiceAccountToken: true
  containers:
  - name: monitor
    image: monitoring-agent:1.0
```

**Essential kubectl commands:**
```bash
kubectl create serviceaccount monitoring-sa -n team-a
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n team-a
kubectl create rolebinding read-pods-binding --role=pod-reader --serviceaccount=team-a:monitoring-sa -n team-a
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrolebinding node-reader-binding --clusterrole=node-reader --group=system:monitoring-team

kubectl get roles,rolebindings -n team-a
kubectl get clusterroles,clusterrolebindings
kubectl describe role pod-reader -n team-a
kubectl describe rolebinding read-pods-binding -n team-a

# CRITICAL debugging command
kubectl auth can-i get pods --as=system:serviceaccount:team-a:monitoring-sa -n team-a
kubectl auth can-i delete deployments --as=system:serviceaccount:team-a:monitoring-sa -n team-a
kubectl auth can-i '*' '*' --as=system:serviceaccount:team-a:monitoring-sa
```

**Built-in default ClusterRoles:**

| ClusterRole | Grants |
|---|---|
| `view` | Read-only access, excludes Secrets/RBAC objects |
| `edit` | Read/write access, excludes RBAC objects |
| `admin` | Full access within a namespace, including RBAC objects there |
| `cluster-admin` | Full, unrestricted access to everything, cluster-wide |

**Best practices:**
- **Never use `cluster-admin`** for application workloads or CI/CD pipelines
- Give every Pod that needs API access its own dedicated ServiceAccount
- Set `automountServiceAccountToken: false` on Pods that don't need API access
- Use `kubectl auth can-i --as=...` proactively during development
- Regularly audit `ClusterRoleBindings`

**Common mistakes:**
- Binding a `ClusterRole` via `ClusterRoleBinding` when a namespaced `RoleBinding` would suffice
- Leaving Pods on the `default` ServiceAccount with no scoped RBAC
- Granting `cluster-admin` to a CI/CD ServiceAccount "just to make deployments work"
- Forgetting that RBAC is **additive only** — no explicit "deny" rule

**Debugging a "Forbidden" error:**
```bash
kubectl auth can-i list pods --as=system:serviceaccount:team-a:monitoring-sa -n team-a
kubectl get rolebindings -n team-a -o yaml
kubectl describe role <role-name> -n team-a
```

## 6. When should we use Role vs ClusterRole?

- **Role + RoleBinding:** permissions scoped to a single namespace
- **ClusterRole + ClusterRoleBinding:** cluster-scoped resources or all-namespace permissions
- **ClusterRole + RoleBinding:** hybrid — reuse a ClusterRole (like built-in `view`) within specific namespaces

## 7. When should we NOT grant broad access?

- Never grant `cluster-admin` except to genuine human cluster administrators
- Avoid wildcard permissions (`resources: ["*"]`, `verbs: ["*"]`) in custom Roles

## 8. Comparison Table

| Aspect | Role | ClusterRole | RoleBinding | ClusterRoleBinding |
|---|---|---|---|---|
| Scope | Single namespace | Cluster-wide | Single namespace | Cluster-wide |
| Can grant access to cluster-scoped resources? | No | Yes | No | Yes |
| Can bind a ClusterRole? | N/A | N/A | Yes | Yes |

## 9. Interview Questions

**Beginner**
- What's the difference between Authentication and Authorization?
- What identity does a Pod use when it calls the Kubernetes API?

**Intermediate**
- Explain the difference between binding a ClusterRole via a RoleBinding vs a ClusterRoleBinding.
- Why is `kubectl auth can-i` useful?

**Advanced / Scenario**
- A CI/CD ServiceAccount was found with `cluster-admin` access after a security audit. Redesign using least-privilege for a pipeline deploying to 3 specific namespaces.
- Explain step-by-step what happens (AuthN -> AuthZ -> Admission) when a compromised Pod's token attempts `kubectl delete namespace production`.

## 10. Mini Quiz

1. True/False: RBAC supports explicit "deny" rules. -> **False**
2. What object type is required to grant permissions on a cluster-scoped resource like Nodes? -> **ClusterRole**
3. Command to check if a ServiceAccount can delete Deployments? -> `kubectl auth can-i delete deployments --as=system:serviceaccount:<ns>:<sa> -n <ns>`

## Hands-on Lab

```bash
kubectl create namespace rbac-lab
kubectl create serviceaccount limited-sa -n rbac-lab
kubectl create role pod-viewer --verb=get,list --resource=pods -n rbac-lab
kubectl create rolebinding pod-viewer-binding --role=pod-viewer --serviceaccount=rbac-lab:limited-sa -n rbac-lab

kubectl auth can-i get pods --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab       # yes
kubectl auth can-i list pods --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab      # yes
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab    # no
kubectl auth can-i get secrets --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab    # no
kubectl auth can-i get pods --as=system:serviceaccount:rbac-lab:limited-sa -n default        # no
```

## Summary

RBAC is Kubernetes' answer to "who can do what, where" — a strictly additive permission model built from four objects, combined with ServiceAccounts as Pod identities.


# Lesson 20: NetworkPolicies

## 1. What is it?

A **NetworkPolicy** is a namespace-scoped object that controls **which Pods can talk to which other Pods** at the network level — a firewall for Pod-to-Pod traffic, enforced by the CNI plugin.

## 2. Why do we need it?

**Without NetworkPolicies**, network segmentation simply doesn't exist inside the cluster — a compromised frontend Pod could freely reach database Pods, internal admin APIs, anything.

**Industry motivation:** Zero-trust networking and defense in depth; compliance frameworks (PCI-DSS, SOC 2, HIPAA) frequently mandate explicit network segmentation.

## 3. Where is it used?

- **Multi-tier applications** — restrict database Pods to only accept traffic from backend Pods
- **Multi-tenant clusters** — prevent `team-a` from reaching `team-b`
- **Compliance-driven environments**

## 4. How does it work internally?

```
+---------------------------- Default (no NetworkPolicy) ----------------------------+
|                                                                                        |
|   Frontend Pod ---------------> Backend Pod ---------------> Database Pod            |
|        |                                                            ^                 |
|        +------------------------------------------------------------+                 |
|                    (frontend can ALSO reach database directly - no restriction)      |
+---------------------------------------------------------------------------------------+

+-------------------------- With NetworkPolicy on Database Pods --------------------------+
|                                                                                             |
|   Frontend Pod --------X (BLOCKED)-------->  Database Pod                                 |
|                                               ^                                            |
|   Backend Pod  -------------(ALLOWED)---------+                                            |
|                                                                                             |
|   NetworkPolicy selects Database Pods via podSelector, and only allows                    |
|   ingress from Pods matching { app: backend } - everything else is denied                  |
+--------------------------------------------------------------------------------------------+
```

**Key internal facts:**
- **NetworkPolicies are enforced by the CNI plugin**, not the API server itself
- **If your CNI doesn't support NetworkPolicy, these objects are silently ignored**
- **Default behavior is allow-all** — a Pod with no NetworkPolicy remains fully open
- **Once ANY NetworkPolicy selects a Pod, that Pod becomes default-deny** for the direction(s) that policy specifies
- Policies are **additive**

## 5. How do we create and manage them?

Default-deny-all ingress (recommended baseline):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # empty selector = applies to ALL Pods in this namespace
  policyTypes:
  - Ingress
```

Allow specific ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```

Namespace-scoped isolation:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace-only
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}        # empty podSelector INSIDE "from" = any Pod in the SAME namespace
```

Allow ingress from a specific OTHER namespace:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-ns
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

Egress control:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

**Essential kubectl commands:**
```bash
kubectl apply -f networkpolicy.yaml
kubectl get networkpolicy -n production
kubectl describe networkpolicy allow-backend-to-db -n production
kubectl exec -it frontend-pod -- wget -qO- --timeout=3 database-svc:5432   # should FAIL/timeout
kubectl exec -it backend-pod -- wget -qO- --timeout=3 database-svc:5432    # should SUCCEED
kubectl delete networkpolicy allow-backend-to-db -n production
```

**Best practices:**
- Start every security-conscious namespace with **default-deny-all**
- Never forget to allow **DNS egress (UDP/TCP port 53)**
- Combine `podSelector` + `namespaceSelector` for precision
- Test in non-production first

**Common mistakes:**
- Assuming NetworkPolicies work regardless of CNI choice
- Forgetting DNS egress rules -> total application breakage
- Confusing `podSelector: {}` semantics in different contexts
- Believing NetworkPolicy provides encryption or identity verification (it doesn't)

**Debugging blocked traffic that "should" be allowed:**
```bash
kubectl get networkpolicy -n production -o yaml
kubectl describe pod <target-pod> -n production
kubectl get pods --show-labels -n production
kubectl get pods -n kube-system | grep -E 'calico|cilium'
```

## 6. When should we use it?

Multi-tier applications, multi-tenant clusters, compliance-driven environments.

## 7. When should we NOT rely on it alone?

Pair with RBAC, Pod Security Standards, and ideally a service mesh for mTLS if actual traffic encryption is needed.

## 8. Comparison Table: NetworkPolicy vs RBAC vs Service Mesh Policies

| Aspect | NetworkPolicy | RBAC | Service Mesh |
|---|---|---|---|
| Layer | L3/L4 | API-level | L7, plus encryption |
| Controls | Which Pods can send packets to which Pods | Who can create/read/modify objects | Service identity, encrypted transport |
| Enforced by | CNI plugin | kube-apiserver | Sidecar proxies |
| Requires extra component | Policy-capable CNI | No | Yes (Istio/Linkerd) |

## 9. Interview Questions

**Beginner**
- What is the default Pod-to-Pod network behavior without any NetworkPolicy?
- What happens the moment ANY NetworkPolicy selects a Pod?

**Intermediate**
- Why might a NetworkPolicy have zero effect even though syntactically correct?
- Difference between empty `podSelector: {}` at top level vs inside `ingress.from`?

**Advanced / Scenario**
- Design a NetworkPolicy strategy for a 3-tier app plus a monitoring namespace scraping all three tiers, using default-deny baseline.
- After deploying default-deny-all egress, an app fails intermittently with DNS errors under load. Likely cause and fix?

## 10. Mini Quiz

1. True/False: NetworkPolicies are enforced by the API server itself. -> **False**
2. What happens applying a NetworkPolicy on a cluster running plain Flannel? -> **Silently ignored**
3. What port/protocol must almost always remain allowed in egress-restricted namespaces? -> **UDP/TCP port 53 (DNS)**

## Hands-on Lab

```bash
kubectl create namespace netpol-lab
kubectl label namespace netpol-lab kubernetes.io/metadata.name=netpol-lab
kubectl run frontend --image=busybox -n netpol-lab --labels="app=frontend" --command -- sleep 3600
kubectl run backend --image=nginx -n netpol-lab --labels="app=backend"
kubectl expose pod backend --port=80 -n netpol-lab
kubectl exec -it frontend -n netpol-lab -- wget -qO- --timeout=3 backend:80    # SUCCEEDS

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-frontend-to-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress: []
EOF

kubectl exec -it frontend -n netpol-lab -- wget -qO- --timeout=3 backend:80    # NOW FAILS/TIMES OUT
```

## Summary

NetworkPolicy brings network-layer zero-trust — RBAC controls API access; NetworkPolicy controls network access.


# Lesson 21: Taints, Tolerations, Node Affinity & Pod Affinity/Anti-Affinity

## 1. What is it?

**Taints** — applied to Nodes; repel Pods unless the Pod "tolerates" that taint. **Tolerations** — applied to Pods; let a Pod be scheduled onto a Node with a matching taint (permission, not requirement). **Node Affinity** — a preference/requirement for which Nodes a Pod should be scheduled on. **Pod Affinity/Anti-Affinity** — preference about being scheduled near or away from other Pods.

## 2. Why do we need them?

**Without taints/tolerations**, you couldn't reserve nodes for specific workloads. **Without affinity/anti-affinity**, you couldn't express "spread these replicas across zones for HA" or "co-locate this cache with its consumer."

**Industry motivation:** HA design depends on not putting all replicas on the same node/zone; cost optimization requires dedicating GPU nodes.

## 3. Where is it used?

- **Control-plane node protection** — tainted `NoSchedule` by default
- **Dedicated node pools:** GPU nodes tainted for ML workloads
- **Spot/preemptible instances:** tainted so only fault-tolerant workloads opt in
- **High availability:** anti-affinity spreading replicas across zones
- **Performance-sensitive co-location**

## 4. How does it work internally?

```
+------------------------------ Taints & Tolerations ------------------------------+
|                                                                                      |
|   Node (GPU-1) - TAINTED: nvidia.com/gpu=true:NoSchedule                          |
|                                                                                      |
|   Pod A (no toleration)  --->  REJECTED, scheduler skips this node                 |
|   Pod B (has matching toleration) --->  ALLOWED to be scheduled here               |
|                                          (but NOT forced to)                        |
+--------------------------------------------------------------------------------+
```

**Taint effects:**

| Effect | Behavior |
|---|---|
| `NoSchedule` | New Pods without matching toleration never scheduled here |
| `PreferNoSchedule` | Scheduler tries to avoid, but will use if no other option (soft) |
| `NoExecute` | New Pods never scheduled AND existing non-tolerating Pods evicted |

```
+--------------------------- Node Affinity ---------------------------+
|                                                                        |
|  Pod spec says: "I MUST run on a node labeled disktype=ssd"          |
|  (requiredDuringSchedulingIgnoredDuringExecution - hard requirement) |
|                                                                        |
|  OR: "I PREFER a node labeled zone=us-east-1a, but not required"     |
|  (preferredDuringSchedulingIgnoredDuringExecution - soft preference, |
|   weighted 1-100)                                                    |
+-----------------------------------------------------------------+

+--------------------------- Pod Anti-Affinity ---------------------------+
|                                                                            |
|  Deployment replicas of "web-app" say:                                  |
|  "Don't schedule me on a node that ALREADY has another Pod labeled      |
|   app=web-app" (topologyKey: kubernetes.io/hostname)                    |
|                                                                            |
|   Node 1: web-app-pod-1 [ok]                                             |
|   Node 2: web-app-pod-2 [ok]  (different node - anti-affinity satisfied)|
|   Node 3: web-app-pod-3 [ok]  (different node again)                    |
+---------------------------------------------------------------------+
```

**Critical distinction — "Required" vs "Preferred":**
- `requiredDuringSchedulingIgnoredDuringExecution` — hard rule; unsatisfiable means Pod stays `Pending` forever
- `preferredDuringSchedulingIgnoredDuringExecution` — soft rule; scheduler tries but still schedules elsewhere if unsatisfiable
- "IgnoredDuringExecution" means: label changes after scheduling don't retroactively evict already-running Pods

## 5. How do we create and manage them?

```bash
kubectl taint node gpu-node-1 nvidia.com/gpu=true:NoSchedule
kubectl taint node gpu-node-1 nvidia.com/gpu=true:NoSchedule-    # remove a taint
kubectl describe node gpu-node-1 | grep Taints
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
spec:
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: trainer
    image: ml-training:1.0
    resources:
      limits:
        nvidia.com/gpu: 1
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]
  containers:
  - name: web
    image: nginx
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["web-app"]
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web
        image: nginx
```

Pod Affinity (co-locate with a related Pod):
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["redis-cache"]
        topologyKey: "kubernetes.io/hostname"
```

**Essential kubectl commands:**
```bash
kubectl taint node <node> key=value:NoSchedule
kubectl label node <node> disktype=ssd
kubectl describe node <node> | grep -A5 Taints
kubectl get pods -o wide
kubectl describe pod <pod-name>
```

**Best practices:**
- Use taints for exclusion, affinity for inclusion/preference
- Prefer `preferredDuringScheduling` over `requiredDuringScheduling` unless truly hard requirement
- Use anti-affinity with `topologyKey: topology.kubernetes.io/zone` for true cross-zone HA
- Consider **Pod Topology Spread Constraints** for even distribution

**Common mistakes:**
- Adding a toleration but forgetting node affinity -> merely permitted, not directed
- Using `requiredDuringScheduling` anti-affinity with more replicas than nodes -> stuck Pending
- Confusing taint `effect` values with toleration `effect`
- Forgetting `NoExecute` taints evict running Pods, unlike `NoSchedule`

**Debugging a Pod stuck `Pending` due to scheduling constraints:**
```bash
kubectl describe pod <pod-name>
# "0/5 nodes are available: 3 node(s) didn't match Pod's node affinity/selector,
#  2 node(s) had taint {nvidia.com/gpu: true}, that the pod didn't tolerate"
kubectl get nodes --show-labels
kubectl describe node <node> | grep Taints
```

## 6. When should we use each?

| Need | Mechanism |
|---|---|
| Reserve nodes for specific workloads | Taints + Tolerations |
| Direct Pods TO specific nodes based on labels | Node Affinity |
| Spread replicas for HA | Pod Anti-Affinity |
| Co-locate related Pods for performance | Pod Affinity |

## 7. When should we NOT use them?

- Don't over-constrain scheduling with excessive `requiredDuringScheduling` rules
- For simple HA spreading needs, consider Pod Topology Spread Constraints instead

## 8. Comparison Table

| Aspect | Taint/Toleration | Node Affinity | Pod Affinity/Anti-Affinity |
|---|---|---|---|
| Applied to | Node (taint) + Pod (toleration) | Pod (references Node labels) | Pod (references other Pod labels) |
| Direction | Repel unless tolerated | Attract/require based on Node labels | Attract/repel based on Pod placement |
| Typical use | Dedicate nodes to workload types | Direct to specific hardware/zone | HA spreading, co-location |

## 9. Interview Questions

**Beginner**
- What's the difference between a taint and a toleration?
- What are the three taint effects, and how does `NoExecute` differ from `NoSchedule`?

**Intermediate**
- Why might adding a toleration to a Pod NOT guarantee it lands on the tainted node?
- Explain `requiredDuringSchedulingIgnoredDuringExecution` in plain language.

**Advanced / Scenario**
- Deployment with 5 replicas and required pod anti-affinity on `kubernetes.io/hostname`, but only 3 nodes. What happens, and fix?
- Design a scheduling strategy for a GPU-based ML platform with bidirectional exclusivity.

## 10. Mini Quiz

1. True/False: A toleration guarantees a Pod will be scheduled on the tainted node. -> **False**
2. Which taint effect evicts already-running Pods? -> `NoExecute`
3. What does "IgnoredDuringExecution" mean? -> Label changes after scheduling don't retroactively evict

## Hands-on Lab

```bash
kubectl taint node <node-name> dedicated=ml:NoSchedule
kubectl label node <node-name> gpu=true

kubectl run test-no-toleration --image=nginx --overrides='
{
  "spec": {
    "nodeSelector": {"gpu": "true"}
  }
}' --dry-run=client -o yaml | kubectl apply -f -
kubectl get pod test-no-toleration -o wide
kubectl describe pod test-no-toleration

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-with-toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ml"
    effect: "NoSchedule"
  nodeSelector:
    gpu: "true"
  containers:
  - name: app
    image: nginx
EOF
kubectl get pod test-with-toleration -o wide
```

## Summary

Taints/tolerations and affinity/anti-affinity are two complementary halves of scheduling control — real production setups combine both.

# Lesson 22: Horizontal Pod Autoscaler (HPA), Vertical Pod Autoscaler (VPA) & Cluster Autoscaler

## 1. What is it?

**HPA** — automatically adjusts the **number of Pod replicas** based on observed metrics (CPU, memory, custom metrics). **VPA** — automatically adjusts the **resource requests/limits** of containers within existing Pods. **Cluster Autoscaler (CA)** — automatically adjusts the **number of Nodes** in the cluster.

## 2. Why do we need them?

**Without HPA**, you'd manually scale replicas before expected traffic spikes — reactive and error-prone. **Without VPA**, you'd guess resource requests/limits once and rarely revisit them. **Without Cluster Autoscaler**, HPA could try to scale Pods up, but with no Node capacity, those Pods just sit `Pending` forever.

**Industry motivation:** Modern cloud economics reward elastic infrastructure — pay for what you use.

## 3. Where is it used?

- **Web/API services with variable traffic** — HPA
- **Batch/ML workloads with unpredictable resource needs** — VPA
- **Cloud-native clusters using managed node groups** — Cluster Autoscaler (or Karpenter on AWS)
- **Cost optimization initiatives**

## 4. How does it work internally?

```
+--------------------------- HPA Control Loop ---------------------------+
|                                                                            |
|  1. Metrics Server (or Prometheus Adapter) collects CPU/memory usage      |
|     from kubelet's cAdvisor on each node                                 |
|                          |                                                 |
|                          v                                                 |
|  2. HPA controller polls metrics every 15s (default) via the Metrics API |
|                          |                                                 |
|                          v                                                 |
|  3. Compares CURRENT average utilization to the TARGET                     |
|     e.g., target: 50% CPU, current: 80% CPU -> need MORE replicas         |
|                          |                                                 |
|                          v                                                 |
|  4. Computes desired replicas:                                            |
|     desiredReplicas = currentReplicas x (currentMetric / targetMetric)    |
|     e.g., 4 x (80/50) = 6.4 -> rounds to 7                                |
|                          |                                                 |
|                          v                                                 |
|  5. Patches the Deployment's replica count -> normal ReplicaSet scaling   |
|     logic takes over from there                                           |
+------------------------------------------------------------------------------+
```

**HPA specifics:**
- Requires the **metrics-server** add-on installed
- For **custom metrics** — requires a **Prometheus Adapter** or similar
- Has built-in **stabilization windows** to prevent thrashing (default: no delay scaling up, 5-minute stabilization window scaling down)

```
+--------------------------- VPA Components ---------------------------+
|                                                                         |
|  VPA Recommender  -> analyzes historical usage, computes recommended    |
|                      CPU/memory requests                               |
|  VPA Updater       -> decides WHEN a Pod needs updating                 |
|  VPA Admission     -> when a Pod is (re)created, MUTATES its resource   |
|  Controller           requests/limits to the recommended values        |
|                                                                         |
|  CRITICAL: applying new resource values requires Pod RECREATION        |
+------------------------------------------------------------------------+
```

```
+---------------------- Cluster Autoscaler Loop ----------------------+
|                                                                          |
|  Scale-UP trigger: Pods stuck `Pending` due to insufficient Node       |
|                      capacity (unschedulable)                          |
|                          |                                              |
|                          v                                              |
|  CA calls the cloud provider API (AWS ASG, GCP MIG, Azure VMSS)        |
|  to add a new Node matching a compatible node group/instance type      |
|                          |                                              |
|                          v                                              |
|  New Node joins -> kube-scheduler places the Pending Pods there        |
|                                                                          |
|  Scale-DOWN trigger: a Node's utilization stays low for a sustained    |
|                       period AND its Pods could be safely rescheduled  |
|                          |                                              |
|                          v                                              |
|  CA cordons + drains the Node, then removes it from the cloud provider |
+---------------------------------------------------------------------+
```

## 5. How do we create and manage them?

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app-deploy
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
```

HPA with custom metrics:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app-deploy
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app-deploy
  updatePolicy:
    updateMode: "Auto"        # "Auto", "Initial", "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
```

**Essential kubectl commands:**
```bash
kubectl apply -f hpa.yaml
kubectl get hpa
kubectl describe hpa web-app-hpa
kubectl top pods
kubectl top nodes
kubectl autoscale deployment web-app-deploy --cpu-percent=50 --min=3 --max=20

kubectl apply -f vpa.yaml
kubectl get vpa
kubectl describe vpa web-app-vpa

kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler
kubectl describe configmap cluster-autoscaler-status -n kube-system
```

**Best practices:**
- Always set both `minReplicas` and `maxReplicas`
- **Never run HPA and VPA on the same metric for the same workload simultaneously**
- Set `behavior.scaleDown.stabilizationWindowSeconds` deliberately
- Ensure PDBs are correctly configured for Cluster Autoscaler
- Use readiness probes correctly

**Common mistakes:**
- Deploying HPA without metrics-server -> `<unknown>` metrics
- Running HPA and VPA on the same CPU metric -> fighting autoscalers
- Not setting resource requests -> HPA percentage meaningless
- VPA `Auto` mode recreates Pods -> unexpected disruptions
- Assuming Cluster Autoscaler scales down aggressively — it's conservative, respects PDBs

**Debugging:**
```bash
kubectl describe hpa web-app-hpa
kubectl top pods
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl describe vpa web-app-vpa
kubectl logs -n kube-system -l app=cluster-autoscaler | grep -i "scale-up"
kubectl describe pod <pending-pod>
```

## 6. When should we use each?

- **HPA:** stateless services with variable, unpredictable traffic
- **VPA:** workloads where right-sizing matters more than replica count
- **Cluster Autoscaler:** any cluster using cloud-managed node groups

## 7. When should we NOT use them?

- Don't use VPA in `Auto` mode on restart-sensitive singleton workloads
- Don't rely on HPA alone without Cluster Autoscaler in a capacity-constrained cluster
- Don't use Cluster Autoscaler in on-prem/bare-metal without a compatible provider integration

## 8. Comparison Table

| Aspect | HPA | VPA | Cluster Autoscaler |
|---|---|---|---|
| Scales what | Pod replica count | Pod resource requests/limits | Node count |
| Trigger | CPU/memory/custom metric thresholds | Historical usage patterns | Pending unschedulable Pods |
| Causes disruption? | No | Yes (recreates Pods, Auto mode) | No direct app disruption |
| Requires | metrics-server | VPA components | Cloud provider API integration |

## 9. Interview Questions

**Beginner**
- What's the fundamental difference between HPA and VPA?
- What component must be installed for HPA to function at all?

**Intermediate**
- Why is it risky to run HPA and VPA on the same resource metric?
- Why can Cluster Autoscaler's scale-down be blocked even when a node appears underutilized?

**Advanced / Scenario**
- Design a complete autoscaling strategy for a flash sale expecting 10x traffic.
- HPA shows target replicas increasing correctly, but new Pods stay `Pending`. Full diagnostic path?

## 10. Mini Quiz

1. True/False: VPA can resize a running Pod without recreating it, in most standard configurations. -> **False**
2. What add-on is required for basic CPU/memory-based HPA? -> **metrics-server**
3. What triggers Cluster Autoscaler to add a new node? -> **Pods stuck Pending due to insufficient capacity**

## Hands-on Lab

```bash
kubectl create deployment php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m
kubectl expose deployment php-apache --port=80
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa -w
kubectl run load-generator --rm -it --image=busybox -- /bin/sh -c \
  "while true; do wget -q -O- http://php-apache; done"
kubectl get hpa php-apache -w
kubectl get pods -w
```

## Summary

Three autoscalers cover three layers of elasticity: Pod count (HPA), Pod size (VPA), and Node count (Cluster Autoscaler).


# Lesson 23: Probes (Liveness/Readiness/Startup) & Init/Sidecar Containers in Depth

## 1. What is it?

**Probes** are health-check mechanisms the kubelet runs against containers:
- **Liveness Probe:** "Is this container still alive/functioning, or should it be restarted?"
- **Readiness Probe:** "Is this container ready to receive traffic right now?"
- **Startup Probe:** "Has this container finished its (potentially slow) startup process yet?"

**Init Containers** run sequentially to completion before any main containers start. **Sidecar Containers** run alongside the main container for the Pod's entire lifetime.

## 2. Why do we need them?

**Without liveness probes**, a deadlocked container sits there forever, marked `Running`, silently serving nothing. **Without readiness probes**, a Pod still initializing receives live traffic immediately, causing failed requests. **Without startup probes**, apps with slow, variable startup times get killed by an impatient liveness probe before they've even finished starting.

**Industry motivation:** Zero-downtime deployments, accurate Service routing, and self-healing all fundamentally depend on probes being configured correctly.

## 3. Where is it used?

- Every production workload should have readiness probes at minimum
- Slow-starting legacy apps (JVM, large in-memory caches) -> startup probes
- Service meshes inject sidecar proxies needing their own readiness state
- Database migration Jobs commonly use init containers to wait for reachability

## 4. How does it work internally?

```
+---------------------- Pod Startup Timeline ----------------------+
|                                                                      |
|  t=0s   Init Container 1 starts -> runs to completion (exit 0)      |
|  t=5s   Init Container 2 starts -> runs to completion (exit 0)      |
|  t=8s   Main container starts                                      |
|                                                                      |
|         +---------- Startup Probe (if defined) ----------+         |
|         |  Runs FIRST. Liveness/Readiness probes are      |         |
|         |  DISABLED until Startup Probe succeeds.         |         |
|         |  Failing this too many times -> container killed |         |
|         +----------------+---------------------------------+         |
|                            | (once startup probe succeeds)           |
|                            v                                          |
|         +---- Liveness Probe ----+    +---- Readiness Probe ----+   |
|         |  Runs continuously      |    |  Runs continuously        |   |
|         |  FAIL -> kubelet KILLS  |    |  FAIL -> Pod REMOVED from |   |
|         |  and RESTARTS container |    |  Service Endpoints (NOT   |   |
|         |  (per restartPolicy)    |    |  killed, just no traffic) |   |
|         +------------------------+    +------------------------+   |
+----------------------------------------------------------------------+
```

**Critical distinction — different failure actions:**

| Probe | On Failure | Purpose |
|---|---|---|
| Liveness | Kill and restart the container | Recover from hangs/deadlocks |
| Readiness | Remove from Service Endpoints (Pod keeps running) | Prevent traffic to a not-yet-ready Pod |
| Startup | Kill and restart (blocks other probes until success) | Protect slow-starting apps |

**Probe mechanisms:** `httpGet`, `tcpSocket`, `exec`, `grpc` (newer, GA since 1.24+)

**Init Container mechanics:**
- Run sequentially, in order — each must complete before the next starts
- If an init container fails, retried per `restartPolicy` — the entire Pod is stuck until all init containers succeed
- Can have different images/tools than the main container

**Native Sidecar Containers (GA since Kubernetes 1.29):** init containers marked with `restartPolicy: Always` behave as "native sidecars" — start before the main container, but keep running for the Pod's entire lifetime, guaranteed to start before and terminate after the main application containers.

## 5. How do we create and manage them?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z postgres-svc 5432; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 3
      successThreshold: 1
```

Using `exec` and `tcpSocket`:
```yaml
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  periodSeconds: 10

readinessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 10
```

Native sidecar container (K8s 1.29+):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  initContainers:
  - name: log-shipper
    image: fluent-bit:2.2
    restartPolicy: Always                 # KEY field
  containers:
  - name: main-app
    image: my-app:1.0
```

**Essential kubectl commands:**
```bash
kubectl apply -f pod.yaml
kubectl get pods                              # READY column shows X/Y containers ready
kubectl describe pod web-app                  # check probe failure Events
kubectl logs web-app -c wait-for-db
kubectl logs web-app -c app
kubectl exec -it web-app -- wget -qO- localhost:8080/healthz
```

**Best practices:**
- **Always** set a readiness probe on anything receiving Service traffic
- Use a **startup probe** for any app with variable/slow startup time
- Make readiness checks more strict/comprehensive than liveness checks
- Never make a liveness probe depend on external dependencies
- Use native sidecars (1.29+) instead of the older pattern

**Common mistakes:**
- Liveness probe checking downstream dependencies -> cascading restart storms
- No startup probe + short `initialDelaySeconds` -> slow-starting apps killed in infinite restart loop
- Readiness probe too lenient
- Forgetting init container failures block the entire Pod

**Debugging:**
```bash
kubectl describe pod <name>
kubectl get pods    # READY column: 0/1 = failing readiness; RESTARTS climbing = liveness failures
kubectl logs <pod> --previous
```

## 6. When should we use each?

- **Startup probe:** slow, variable, or unpredictable startup time
- **Liveness probe:** apps that can genuinely hang/deadlock
- **Readiness probe:** virtually always, for anything behind a Service
- **Init containers:** one-time setup/pre-condition tasks
- **Native sidecars:** logging agents, service mesh proxies

## 7. When should we NOT use them?

- Don't add a liveness probe reflexively to every container
- Don't check external dependencies in liveness probes
- Don't use the older sidecar pattern on 1.29+ clusters for new workloads

## 8. Comparison Table

| Aspect | Liveness | Readiness | Startup |
|---|---|---|---|
| Question answered | "Is it alive?" | "Is it ready for traffic?" | "Has it finished starting?" |
| On failure | Restart container | Remove from Endpoints | Restart container (blocks other probes) |
| Runs when | Continuously | Continuously | Only until first success |
| Should check dependencies? | **No** | **Yes** | No |

## 9. Interview Questions

**Beginner**
- What's the difference in outcome between a failed liveness probe and a failed readiness probe?
- Why do init containers run before the main container, and what happens if one fails?

**Intermediate**
- Why is it bad practice for a liveness probe to check downstream database connectivity?
- What problem does a startup probe solve that a large `initialDelaySeconds` doesn't?

**Advanced / Scenario**
- App has intermittent 2-3 second GC pauses; liveness probe has `timeoutSeconds: 1, failureThreshold: 3, periodSeconds: 5`. Explain the risk and reconfigure.
- Design a full probe strategy for a Java Spring Boot app with 60-90s variable startup and a flaky Redis dependency.

## 10. Mini Quiz

1. True/False: A failed readiness probe causes the container to restart. -> **False**
2. What field makes an init container behave as a "native sidecar"? -> `restartPolicy: Always`
3. Should a liveness probe check an external database dependency? -> **No**

## Hands-on Lab

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: nginx
    readinessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /nonexistent-path
        port: 80
      periodSeconds: 5
      failureThreshold: 2
EOF
kubectl get pod probe-demo -w
kubectl describe pod probe-demo
```

## Summary

Probes bridge "the process is running" and "the application is actually healthy" — liveness handles recovery-via-restart, readiness handles traffic-safety, startup protects slow-booting apps.


# Lesson 24: Helm & Kustomize

## 1. What is it?

**Helm** is the de facto package manager for Kubernetes — bundles manifests into a reusable, versioned, templated **Chart**. **Kustomize** takes a template-free approach — plain manifests + patches/overlays, built into `kubectl`.

## 2. Why do we need them?

**Without Helm/Kustomize**, teams maintain fully duplicated YAML per environment or build ad-hoc templating with shell scripts — fragile and unversioned.

## 3. Where is it used?

- **Installing third-party software:** Prometheus, Grafana, cert-manager, ingress-nginx — virtually all ship as Helm charts
- **Application deployment pipelines:** ArgoCD/Flux support both
- **Environment-specific configuration**
- **GitOps workflows**

## 4. How does it work internally?

```
+---------------------------- Helm Chart Structure ----------------------------+
|  mychart/                                                                     |
|  |-- Chart.yaml           # metadata: name, version, description             |
|  |-- values.yaml          # DEFAULT configuration values                     |
|  |-- templates/           # Go-template YAML files, referencing values       |
|  |   |-- deployment.yaml                                                     |
|  |   |-- service.yaml                                                        |
|  |   `-- _helpers.tpl     # reusable template snippets/functions             |
|  `-- charts/              # dependency sub-charts (nested Helm charts)       |
+-------------------------------------------------------------------------------+

  helm install my-release mychart/ -f prod-values.yaml
             |
             v
  1. Helm reads values.yaml, MERGES with any -f overrides / --set flags
             |
             v
  2. Renders every template file, substituting {{ .Values.xxx }} placeholders
             |
             v
  3. Produces final, complete Kubernetes YAML manifests
             |
             v
  4. Applies them to the cluster via the API server
             |
             v
  5. Records this as a "Release" - a tracked, versioned deployment,
     stored as a Secret in the cluster, enabling rollback
```

```
+----------------------- Kustomize Base + Overlays -----------------------+
|  base/                                                                    |
|  |-- deployment.yaml       # plain, complete K8s YAML - no template syntax!|
|  |-- service.yaml                                                         |
|  `-- kustomization.yaml    # lists which resources belong to this base    |
|                                                                             |
|  overlays/                                                                  |
|  |-- dev/                                                                   |
|  |   |-- kustomization.yaml   # references ../../base, applies dev patches |
|  |   `-- replica-patch.yaml    # e.g., replicas: 1                         |
|  |-- prod/                                                                   |
|  |   |-- kustomization.yaml   # references ../../base, applies prod patches|
|  |   `-- replica-patch.yaml    # e.g., replicas: 10                        |
+---------------------------------------------------------------------------+

  kubectl apply -k overlays/prod/
             |
             v
  1. Kustomize reads base/ manifests AS-IS (complete, valid YAML)
             |
             v
  2. Applies declared patches (strategic merge patch or JSON patch)
             |
             v
  3. Produces final merged YAML - NO templating language was ever used
             |
             v
  4. Applies to cluster
```

**Key conceptual difference:** Helm uses **templating** (Go template syntax), meaning raw files aren't valid YAML until rendered. Kustomize uses **patching/overlaying** on top of plain, always-valid YAML.

## 5. How do we create and manage them?

```yaml
apiVersion: v2
name: mychart
description: A sample Helm chart
version: 1.0.0            # CHART version
appVersion: "2.3.1"        # APPLICATION version
```

```yaml
replicaCount: 3
image:
  repository: my-app
  tag: "2.3.1"
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
service:
  type: ClusterIP
  port: 80
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

**Essential Helm commands:**
```bash
helm create mychart
helm lint mychart/
helm install my-release mychart/ -f prod-values.yaml
helm install my-release mychart/ --set replicaCount=5
helm list
helm status my-release
helm get values my-release
helm upgrade my-release mychart/ -f prod-values.yaml --set image.tag=2.4.0
helm rollback my-release 1
helm history my-release
helm uninstall my-release
helm template mychart/ -f prod-values.yaml
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo postgresql
helm install my-postgres bitnami/postgresql
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
- ../../base
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: my-app
images:
- name: my-app
  newTag: "2.4.0"
commonLabels:
  env: production
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10
```

**Essential Kustomize commands:**
```bash
kubectl apply -k overlays/prod/
kubectl kustomize overlays/prod/
kustomize build overlays/prod/
kubectl delete -k overlays/prod/
```

**Best practices:**
- Use **Helm** for reusable, shareable packages, complex conditional logic, versioned releases
- Use **Kustomize** for simple environment overlays without templating language
- Store `values-<env>.yaml`/overlay directories in Git for GitOps
- Always run `helm template`/`kubectl kustomize` locally to review before applying

**Common mistakes:**
- Overusing Helm's templating for logic that doesn't need it
- Forgetting `helm upgrade --install` vs plain `helm install`
- Kustomize patches targeting the wrong resource — silently doesn't apply
- Not pinning chart versions

**Debugging:**
```bash
helm template mychart/ -f prod-values.yaml | less
helm get manifest my-release
kubectl kustomize overlays/prod/ | less
```

## 6. When should we use each?

- **Helm:** installing/managing third-party software, or apps needing packaging/versioning/rollback
- **Kustomize:** simple environment overlays on manifests you own, staying close to plain kubectl

## 7. When should we NOT use them?

- Don't reach for Helm for a single, simple app with no environment variation
- Don't use Kustomize for genuinely complex conditional logic

## 8. Comparison Table: Helm vs Kustomize

| Aspect | Helm | Kustomize |
|---|---|---|
| Approach | Templating (Go templates) | Patching/overlaying plain YAML |
| Learning curve | Higher | Lower |
| Built into kubectl | No | **Yes** |
| Versioned releases + rollback | **Yes**, built-in | No native concept |
| Best for | Packaging/distributing, complex logic | Environment-specific overlays |
| Third-party ecosystem | Massive | Minimal |

## 9. Interview Questions

**Beginner**
- What is a Helm Chart, and what problem does it solve?
- How does Kustomize differ fundamentally from Helm?

**Intermediate**
- Difference between `helm install` and `helm upgrade --install`?
- How does Kustomize apply environment-specific changes without templating syntax?

**Advanced / Scenario**
- Distributing an internal platform tool to 50 teams needing slightly different config but sharing 90% of manifests — Helm or Kustomize?
- A `helm upgrade` unexpectedly changed far more resources than intended. Debugging commands and recovery path?

## 10. Mini Quiz

1. True/False: Kustomize requires learning a custom templating syntax similar to Helm's. -> **False**
2. Which tool has native versioned "releases" with rollback? -> **Helm**
3. Command to preview Helm's rendered output without installing? -> `helm template`

## Hands-on Lab

```bash
helm create demo-chart
helm template demo-chart/ | less
helm install demo-release demo-chart/ --set replicaCount=2
helm list
helm upgrade demo-release demo-chart/ --set replicaCount=5
helm history demo-release
helm rollback demo-release 1
helm uninstall demo-release

mkdir -p kdemo/base kdemo/overlays/prod
kubectl kustomize kdemo/overlays/prod/
kubectl apply -k kdemo/overlays/prod/
kubectl get deployments -n production
kubectl delete -k kdemo/overlays/prod/
```

## Summary

Helm (templating, packaging, versioned releases) and Kustomize (patch-based, template-free overlays) solve the same problem with different philosophies.


# Lesson 25: Operators & CRDs (Custom Resource Definitions)

## 1. What is it?

**CRD (Custom Resource Definition)** — extends the Kubernetes API with **new object types** you define (e.g., `PostgresCluster`, `KafkaTopic`, `Certificate`) that behave exactly like built-in objects.

**Operator** — a **custom controller** following the exact reconciliation loop pattern from ReplicaSets, that watches custom resources and takes real action to make the cluster's actual state match what's declared.

## 2. Why do we need them?

**Without CRDs**, you'd be limited to built-in object types forever. **Without Operators**, even with a CRD, nothing would actually *act* on it.

**Industry motivation:** A human SRE knows how to run Postgres in production — an Operator encodes that expertise as software, running continuously.

## 3. Where is it used?

- **Databases:** Postgres Operator, MongoDB Community Operator, MySQL Operator
- **Message queues:** Strimzi (Kafka Operator)
- **Certificates:** cert-manager
- **Service meshes:** Istio's control plane
- **Cloud provider integrations:** AWS Controllers for Kubernetes (ACK), Crossplane

## 4. How does it work internally?

```
+------------------------- CRD + Operator Architecture -------------------------+
|                                                                                  |
|  1. CRD registers a new API type with the API server:                          |
|     "PostgresCluster" objects are now valid, just like "Pod" or "Deployment"    |
|                                                                                  |
|  2. User creates a custom resource (an INSTANCE of the CRD):                    |
|     kind: PostgresCluster                                                        |
|     spec: { replicas: 3, version: "16", storageSize: "100Gi" }                   |
|                        |                                                          |
|                        v                                                          |
|  3. Operator Pod is CONTINUOUSLY WATCHING the API server for PostgresCluster    |
|     objects (using the exact same watch/reconcile pattern as built-in           |
|      controllers)                                                                |
|                        |                                                          |
|                        v                                                          |
|  4. Operator's reconcile loop compares DESIRED (the CR spec) to ACTUAL           |
|     and takes action:                                                            |
|     - Creates/updates a StatefulSet with 3 replicas                              |
|     - Creates a headless Service for peer discovery                              |
|     - Generates Secrets for credentials                                           |
|     - Configures replication between Postgres instances                           |
|     - Continuously monitors: if the primary fails, PROMOTES a replica            |
|                        |                                                          |
|                        v                                                          |
|  5. Operator updates the CR's `status` subresource to reflect real-world state    |
+------------------------------------------------------------------------------+
```

**Key internal facts:**
- CRDs are registered via a `CustomResourceDefinition` object — teaches the API server the schema but adds **zero behavior** on its own
- The Operator is just a regular Pod/Deployment running controller code (commonly built with Kubebuilder or Operator SDK)
- Custom Resources support a `status` subresource — `spec` = desired state (user-authored), `status` = observed state (Operator-authored)
- Operators commonly use **owner references** so deleting the custom resource cascades cleanup

## 5. How do we create and manage them?

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.db.example.com
spec:
  group: db.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
              version:
                type: string
              storageSize:
                type: string
          status:
            type: object
            properties:
              readyReplicas:
                type: integer
              primaryPod:
                type: string
    subresources:
      status: {}
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
    shortNames: ["pgc"]
```

```yaml
apiVersion: db.example.com/v1
kind: PostgresCluster
metadata:
  name: my-postgres
spec:
  replicas: 3
  version: "16"
  storageSize: "100Gi"
```

**Essential kubectl commands:**
```bash
kubectl apply -f crd.yaml
kubectl get crd
kubectl explain postgrescluster
kubectl apply -f my-postgres-cluster.yaml
kubectl get postgresclusters
kubectl get pgc my-postgres -o yaml
kubectl describe pgc my-postgres
kubectl get statefulsets,services,secrets -l app=my-postgres
kubectl get pods -n operators-namespace -l app=postgres-operator
kubectl logs -n operators-namespace -l app=postgres-operator
```

Installing a real-world Operator:
```bash
helm repo add strimzi https://strimzi.io/charts/
helm install kafka-operator strimzi/strimzi-kafka-operator

kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
EOF
```

**Best practices:**
- Prefer a well-maintained, widely-adopted Operator over building your own
- Always check an Operator's `status` conditions/fields when debugging
- Understand what RBAC permissions an Operator requests before installing it
- Version your CRDs carefully

**Common mistakes:**
- Treating a custom resource like "fire and forget"
- Deleting a CRD without understanding it cascades to delete every instance
- Installing multiple Operators managing the same underlying resource type
- Underestimating an Operator's RBAC footprint

**Debugging a stuck Custom Resource:**
```bash
kubectl describe pgc my-postgres
kubectl get pgc my-postgres -o jsonpath='{.status}'
kubectl logs -n operators-namespace -l app=postgres-operator --tail=100
kubectl get events --field-selector involvedObject.name=my-postgres
```

## 6. When should we use it?

When you need Kubernetes to manage a stateful, operationally complex system with domain-specific logic that generic Deployments/StatefulSets can't express.

## 7. When should we NOT use it?

- Don't build a custom Operator for simple use cases a Helm chart + Deployment can handle
- If a managed cloud service meets your needs, it's often operationally simpler

## 8. Comparison Table: Built-in Controllers vs Operators

| Aspect | Built-in Controller | Operator |
|---|---|---|
| Object types managed | Pods, ReplicaSets (generic) | Domain-specific CRDs |
| Shipped with Kubernetes | Yes | No — installed separately |
| Encodes domain expertise | No (generic mechanics only) | **Yes** (failover logic, backup strategies) |
| Reconciliation pattern | Same watch/compare/act loop | **Same watch/compare/act loop** |

## 9. Interview Questions

**Beginner**
- What's the difference between a CRD and an Operator?
- Does creating a CRD alone cause any behavior/action in the cluster?

**Intermediate**
- Explain the `spec` vs `status` convention for custom resources.
- Why might you choose an existing, widely-adopted Operator over building a custom one?

**Advanced / Scenario**
- Design an Operator for managing "TenantWorkspace" custom resources representing a customer's isolated namespace + quota + NetworkPolicy + RBAC.
- A Postgres Operator's CR shows `status.phase: Reconciling` for over an hour with no progress. Full diagnostic path?

## 10. Mini Quiz

1. True/False: Registering a CRD automatically gives Kubernetes the ability to act on instances of that type. -> **False**
2. What subresource convention lets a custom resource expose "observed state"? -> `status` subresource
3. What happens to all instances of a custom resource if you delete its CRD? -> **All are deleted too (cascading)**

## Hands-on Lab

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager --set installCRDs=true -n cert-manager --create-namespace
kubectl get crd | grep cert-manager
kubectl explain certificate.spec

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-cert
  namespace: default
spec:
  secretName: demo-cert-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
  - demo.example.com
EOF

kubectl get certificate demo-cert -w
kubectl get secret demo-cert-tls
```

## Summary

CRDs extend the Kubernetes API's vocabulary; Operators are the controllers that give that vocabulary real meaning, applying the exact same reconciliation pattern that powers every built-in controller.


# Lesson 26: Admission Controllers

## 1. What is it?

**Admission Controllers** are the final stage of API request processing — after Authentication and Authorization — where requests are **inspected, potentially modified, and potentially rejected** before being persisted to etcd.

- **Mutating Admission Controllers** — can **modify** the request object
- **Validating Admission Controllers** — can only **accept or reject** — always run **after** all mutating controllers

## 2. Why do we need them?

**Without admission control**, RBAC alone can only answer "is this user allowed to create A Pod?" — not nuanced policy questions like "does this Pod's image come from an approved registry?"

**Industry motivation:** Enterprise governance requires enforcing organization-wide policies beyond RBAC.

## 3. Where is it used?

- **ResourceQuota and LimitRange enforcement** — both implemented as built-in admission controllers
- **Istio/Linkerd sidecar injection**
- **Pod Security Admission**
- **Policy engines**: OPA Gatekeeper, Kyverno
- **cert-manager's own webhook**

## 4. How does it work internally?

```
+---------------------- Full Request Processing Pipeline ----------------------+
|                                                                                  |
|  Request -> AUTHENTICATION -> AUTHORIZATION (RBAC) ->                          |
|                                                                                  |
|         +-----------------------------------------------------+               |
|         |           MUTATING ADMISSION CONTROLLERS               |               |
|         |  (run first, in a defined order)                        |               |
|         |  - Built-in: DefaultStorageClass, ServiceAccount        |               |
|         |    NamespaceLifecycle                                    |               |
|         |  - Custom: MutatingWebhookConfiguration                 |               |
|         |    (e.g., Istio sidecar injector, Kyverno)               |               |
|         |  -> Each can MODIFY the object being created/updated    |               |
|         +-----------------------+-------------------------------+               |
|                                    |                                            |
|                                    v                                            |
|         +-----------------------------------------------------+               |
|         |          VALIDATING ADMISSION CONTROLLERS              |               |
|         |  (run AFTER all mutations are applied - sees the       |               |
|         |   FINAL version of the object)                          |               |
|         |  - Built-in: ResourceQuota, LimitRanger, PodSecurity     |               |
|         |  - Custom: ValidatingWebhookConfiguration                |               |
|         |    (e.g., OPA Gatekeeper policy checks)                  |               |
|         |  -> Each can only ACCEPT or REJECT (no modification)    |               |
|         +-----------------------+-------------------------------+               |
|                                    | (if ALL pass)                              |
|                                    v                                            |
|                        Object written to etcd                                  |
+--------------------------------------------------------------------------------+
```

**Custom webhook mechanics:**
1. Register a `MutatingWebhookConfiguration`/`ValidatingWebhookConfiguration`, specifying which resources/operations trigger it and a webhook Service URL
2. When a matching request comes in, the API server pauses and sends the object (`AdmissionReview` JSON) to that webhook
3. The webhook returns `allowed: true/false`, and (mutating) an optional JSON Patch
4. The API server applies the patch or accepts/rejects

**Failure policy:** `failurePolicy` (`Fail` or `Ignore`) — if the webhook is unreachable, `Fail` blocks the request entirely; `Ignore` lets it through unchecked.

## 5. How do we create and manage them?

```bash
kubectl get pods -n kube-system kube-apiserver-<node> -o yaml | grep enable-admission-plugins
```

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: image-registry-policy
webhooks:
- name: check-registry.example.com
  clientConfig:
    service:
      name: image-policy-webhook
      namespace: policy-system
      path: "/validate"
    caBundle: <base64-encoded-CA-cert>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
  failurePolicy: Fail
  sideEffects: None
  admissionReviewVersions: ["v1"]
```

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: auto-label-injector
webhooks:
- name: inject-labels.example.com
  clientConfig:
    service:
      name: label-injector-webhook
      namespace: policy-system
      path: "/mutate"
    caBundle: <base64-encoded-CA-cert>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
  failurePolicy: Ignore
  sideEffects: None
  admissionReviewVersions: ["v1"]
```

OPA Gatekeeper example:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-owner-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    labels: ["owner"]
```

**Essential kubectl commands:**
```bash
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
kubectl describe validatingwebhookconfiguration image-registry-policy
kubectl get pods -n policy-system
kubectl logs -n policy-system <webhook-pod-name>
```

**Best practices:**
- Set `failurePolicy: Fail` only for critical security policies where the webhook has HA
- Always set timeouts on webhooks
- Scope webhook `rules` as narrowly as possible
- Run webhook server Pods with high availability
- Prefer established policy engines (OPA Gatekeeper, Kyverno) over hand-rolled webhooks

**Common mistakes:**
- Setting `failurePolicy: Fail` without HA -> a bad webhook deploy blocks all Pod creation cluster-wide
- Forgetting webhooks add latency to every matching request
- Not excluding `kube-system` from broad policies
- Confusing mutating vs validating order

**Debugging a rejected request:**
```bash
kubectl get validatingwebhookconfigurations -o yaml
kubectl logs -n policy-system <webhook-pod>
kubectl get events -A | grep -i admission
```

**Debugging a webhook causing cluster-wide issues:**
```bash
kubectl get pods -n policy-system
kubectl describe validatingwebhookconfiguration <name> | grep failurePolicy
kubectl delete validatingwebhookconfiguration <name>    # emergency unblock
```

## 6. When should we use it?

Enforcing organization-wide policies beyond RBAC's scope: required labels, approved image registries, automatic mutation needs.

## 7. When should we NOT use it?

- Don't build custom webhook servers for needs already covered by OPA Gatekeeper/Kyverno
- Don't add admission webhooks for things RBAC already handles cleanly
- Be cautious with `failurePolicy: Fail` in less mature clusters without HA

## 8. Comparison Table: RBAC vs Admission Control

| Aspect | RBAC (AuthZ) | Admission Control |
|---|---|---|
| Question answered | "Is this identity allowed to perform this verb on this resource type?" | "Given this specific object's content, should it be allowed/modified?" |
| Can modify the request? | No | Yes (mutating webhooks) |
| Granularity | Identity + verb + resource type | Arbitrary logic on the object's fields/content |
| Runs relative to RBAC | Before admission | **After** authentication + authorization |

## 9. Interview Questions

**Beginner**
- What's the difference between a mutating and validating admission controller?
- Where does admission control fit relative to AuthN/AuthZ?

**Intermediate**
- Why do mutating webhooks always run before validating webhooks?
- Risk of `failurePolicy: Fail` on a webhook without HA?

**Advanced / Scenario**
- Design an admission control strategy for: (1) required owner/cost-center labels, (2) approved image registry only, (3) no root Pods.
- No new Pods can be created cluster-wide, `kubectl apply` hangs 10s then fails with webhook timeout. Diagnose and mitigate.

## 10. Mini Quiz

1. True/False: Validating admission webhooks can modify the object. -> **False**
2. Field controlling behavior when webhook is unreachable? -> `failurePolicy`
3. Two built-in admission controllers covered earlier? -> **ResourceQuota, LimitRanger**

## Hands-on Lab

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
kubectl get pods -n gatekeeper-system -w

kubectl apply -f - <<EOF
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := input.review.object.metadata.labels
        required := input.parameters.labels
        missing := required[_]
        not provided[missing]
        msg := sprintf("Missing required label: %v", [missing])
      }
EOF

kubectl apply -f - <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-owner-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    labels: ["owner"]
EOF

kubectl run test-pod --image=nginx
kubectl run test-pod-2 --image=nginx --labels="owner=platform-team"
```

## Summary

Admission control enables policy enforcement on object content, going beyond RBAC's identity-based model.


# Lesson 27: Monitoring & Logging (Prometheus, Grafana, Metrics Server, EFK/Loki)

## 1. What is it?

**Monitoring** — collecting numerical time-series data to understand system behavior and trigger alerts. **Prometheus** is the dominant Kubernetes-native metrics system; **Grafana** provides visualization. **Logging** — collecting the actual text output containers produce, aggregated centrally. **EFK** and **Loki** are the two dominant log stacks. **Metrics Server** — lightweight component providing basic CPU/memory metrics for `kubectl top`/HPA.

## 2. Why do we need them?

**Without metrics/monitoring**, you're flying blind on capacity planning and performance regressions. **Without centralized logging**, debugging a crashed Pod is nearly impossible once it's gone.

**Industry motivation:** The three pillars of observability — metrics, logs, and traces.

## 3. Where is it used?

- Every production cluster: Metrics Server (HPA), monitoring stack, centralized logging
- SRE/on-call workflows — Grafana dashboards + Prometheus alerts
- Capacity planning
- Debugging distributed systems

## 4. How does it work internally?

```
+----------------------------- Prometheus Architecture -----------------------------+
|                                                                                       |
|  Prometheus operates on a PULL model:                                               |
|                                                                                       |
|  +-------------+     scrapes /metrics endpoint       +-------------------+          |
|  | Prometheus   |---------every 15-30s (default)----->|  App Pod           |          |
|  | Server        |                                     |  exposes /metrics  |          |
|  | (Deployment)  |<-----------------------------------|  (Prometheus       |          |
|  |                |                                     |   client library)  |          |
|  +------+-------+                                     +-------------------+          |
|          |                                                                            |
|          |  ALSO scrapes:                                                             |
|          |  - node-exporter DaemonSet -> node-level CPU/mem/disk                       |
|          |  - kube-state-metrics Deployment -> K8s OBJECT state as metrics             |
|          |  - kubelet's own /metrics/cadvisor endpoint -> container resource usage     |
|          v                                                                            |
|  Stores time-series data locally (or remote-writes to Thanos/Cortex/Mimir)           |
|          |                                                                            |
|          v                                                                            |
|  +-------------+         +-------------+        +--------------------+              |
|  |   Grafana     |--------->|  Alertmanager |        |  HPA / Prometheus  |              |
|  |  (dashboards) |  query  |  (routes fired |        |  Adapter (custom    |              |
|  |                |        |   alerts)      |        |  metrics API)       |              |
|  +-------------+         +-------------+        +--------------------+              |
+---------------------------------------------------------------------------------------+
```

```
+---------------------------- Logging Architecture (EFK/Loki) ----------------------------+
|                                                                                             |
|  Every node runs a log-shipping DaemonSet:                                                |
|                                                                                             |
|  +----------------+   reads container stdout/stderr    +----------------------+          |
|  |  Fluent Bit /    |<------ from node's filesystem ------|  Container runtime      |          |
|  |  Fluentd /        |       (/var/log/containers/*.log)  |  writes stdout/stderr    |          |
|  |  Promtail          |                                    |  to a log file per        |          |
|  |  (DaemonSet)       |                                    |  container                |          |
|  +--------+--------+                                    +----------------------+          |
|           |  ships parsed/enriched logs                                                    |
|           v                                                                                 |
|  +-----------------+          OR          +-----------------+                            |
|  |  Elasticsearch     |                     |  Loki              |                            |
|  |  (stores + indexes  |                     |  (INDEXES ONLY       |                            |
|  |   full log content)  |                     |   LABELS, not full   |                            |
|  +--------+---------+                     |   text - cheaper)   |                            |
|           |                                 +--------+---------+                            |
|           v                                          |                                       |
|  +-----------------+                                 v                                       |
|  |     Kibana         |                       +-----------------+                            |
|  |  (search/visualize) |                       |     Grafana        |                            |
|  +-----------------+                       |  (LogQL queries)    |                            |
|                                              +-----------------+                            |
+--------------------------------------------------------------------------------------------+
```

**Key architectural insight:** both node-exporter and Fluent Bit reuse the DaemonSet pattern — both need node-local collection before centralizing.

**Elasticsearch vs Loki:** Elasticsearch indexes the full text of every log line (powerful, expensive); Loki indexes only labels/metadata, storing log content compressed and unindexed (cheaper, less powerful free-text search).

## 5. How do we create and manage them?

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: app
    image: my-app:1.0
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: high-error-rate
  namespace: monitoring
spec:
  groups:
  - name: app-alerts
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Error rate above 5% for {{ $labels.service }}"
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n logging --create-namespace \
  --set promtail.enabled=true --set grafana.enabled=false
```

**Essential kubectl commands:**
```bash
kubectl top nodes
kubectl top pods -A
kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
kubectl get servicemonitors -A
kubectl get prometheusrules -A
kubectl logs -f deployment/my-app
kubectl logs -f deployment/my-app --all-containers=true
kubectl logs -l app=my-app --prefix=true
kubectl get pods -n logging -l app=fluent-bit -o wide
```

**Best practices:**
- Use structured logging (JSON)
- Set log retention policies deliberately
- Use Prometheus Operator + ServiceMonitor/PrometheusRule CRDs rather than raw config
- Set up Alertmanager routing thoughtfully
- Consider remote-write/long-term storage (Thanos, Cortex, Mimir)

**Common mistakes:**
- Confusing Metrics Server (minimal, no history, HPA-only) with a full monitoring solution
- Not setting resource limits on Fluent Bit/node-exporter DaemonSets
- Excessive cardinality in Prometheus labels
- Alerting on symptoms that don't matter

**Debugging monitoring/logging gaps:**
```bash
kubectl get servicemonitor my-app-monitor -n monitoring -o yaml
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090
# visit http://localhost:9090/targets

kubectl get pods -n logging -l app=fluent-bit -o wide
kubectl logs -n logging <fluent-bit-pod-on-that-node>
kubectl exec -it <fluent-bit-pod> -- cat /var/log/containers/<your-pod>*.log
```

## 6. When should we use each?

- **Metrics Server:** always install
- **Prometheus + Grafana:** standard for cluster-wide metrics/alerting
- **EFK (Elasticsearch):** powerful full-text log search
- **Loki:** cost-efficient, Prometheus-philosophy-aligned logging

## 7. When should we NOT use certain approaches?

- Don't rely on `kubectl logs`/`kubectl top` alone for production observability
- Don't default to Elasticsearch if cost is a major constraint
- Don't over-instrument with excessive custom metrics/high-cardinality labels

## 8. Comparison Table: Elasticsearch vs Loki

| Aspect | Elasticsearch (EFK) | Loki |
|---|---|---|
| Indexing | Full-text, every field | Labels/metadata only |
| Storage cost | Higher | Lower |
| Search power | Very powerful full-text | Good for label-based filtering |
| Philosophy | General-purpose search repurposed | "Prometheus, but for logs" |
| Typical pairing | Kibana | Grafana |

## 9. Interview Questions

**Beginner**
- What's the difference between Metrics Server and Prometheus?
- Why can't you rely on `kubectl logs` alone for production log management?

**Intermediate**
- Explain Prometheus's pull-based scraping model.
- Why does Loki index only labels instead of full log text?

**Advanced / Scenario**
- Design a complete observability stack for a 200-microservice platform.
- Prometheus's memory usage has grown dramatically, correlating with a new service deployment. Hypothesis and PromQL query to confirm?

## 10. Mini Quiz

1. True/False: Metrics Server provides historical, long-term-retained metrics data. -> **False**
2. What pattern do node-exporter and Fluent Bit share architecturally? -> **DaemonSet**
3. What does Loki index, compared to Elasticsearch? -> **Labels/metadata only**

## Hands-on Lab

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get pods -n monitoring -w
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090
kubectl top nodes
kubectl top pods -A
```

## Summary

Monitoring and logging turn Kubernetes' opaque, ephemeral runtime behavior into observable, queryable, alertable data.


# Lesson 28: Production Best Practices — High Availability, Disaster Recovery, Security Hardening & Upgrade Strategies

## 1. What is it?

This capstone lesson ties together every prior lesson into the operational discipline of running Kubernetes reliably at production scale: ensuring the cluster survives failures (**HA**), can recover from catastrophic loss (**DR**), resists compromise (**security hardening**), and can be upgraded safely over time (**upgrade strategy**).

## 2. Why do we need it?

Every mechanism covered in this course exists to solve a piece of reliability/security — but production readiness requires combining them deliberately.

## 3. Where is it used?

Every real production cluster, especially regulated industries with formal DR/security audit requirements.

## 4. High Availability (HA)

**Control Plane HA:**
```
+------------------- HA Control Plane (3 nodes minimum) -------------------+
|  etcd cluster: 3 or 5 members (ODD number - quorum math)                  |
|  Quorum = floor(n/2)+1 members must be alive to keep functioning          |
|    3 members -> tolerates 1 failure                                       |
|    5 members -> tolerates 2 failures                                      |
|    (EVEN numbers waste a node without adding fault tolerance)             |
|                                                                              |
|  kube-apiserver: multiple replicas behind a load balancer                  |
|                                                                              |
|  scheduler + controller-manager: run multiple replicas with LEADER         |
|  ELECTION (only one is "active" at a time)                                 |
+------------------------------------------------------------------------------+
```

**Application-layer HA:**
- **Multiple replicas** (Deployment) — baseline redundancy
- **Pod Anti-Affinity / Topology Spread Constraints** — spread across nodes AND zones
- **PodDisruptionBudgets (PDB)**:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
```

**What this does:** guarantees that **voluntary disruptions** (node drains, Cluster Autoscaler scale-downs) never take more Pods offline than the budget allows — `kubectl drain` refuses to evict a Pod if it would violate the PDB. This does **not** protect against involuntary disruptions (node crashes) — only planned/voluntary ones.

- **Multi-zone/multi-region node pools** combined with topology spread constraints

## 5. Disaster Recovery (DR)

**etcd backup — the single most critical DR practice:**
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20260707.db

ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-20260707.db \
  --data-dir=/var/lib/etcd-restored
```

**PersistentVolume backup:** use CSI VolumeSnapshots for application data, automate via CronJob or backup operator.

**Velero — de-facto standard backup/DR tool:**
```bash
velero backup create full-cluster-backup --include-cluster-resources=true
velero backup create namespace-backup --include-namespaces=production
velero schedule create daily-backup --schedule="0 2 * * *" --include-namespaces=production
velero restore create --from-backup full-cluster-backup
```

Velero backs up both Kubernetes object manifests AND can trigger CSI volume snapshots.

**Multi-cluster/multi-region DR strategies:**

| Strategy | RTO/RPO | Cost | Complexity |
|---|---|---|---|
| Backup & Restore (Velero + etcd snapshots) | Hours (RTO) | Low | Low |
| Pilot Light | Tens of minutes | Medium | Medium |
| Active-Passive | Minutes | High | Medium-High |
| Active-Active | Near-zero | Highest | Highest |

## 6. Security Hardening

**Layered security checklist:**

| Layer | Mechanism | Reference |
|---|---|---|
| API access | RBAC least-privilege, no `cluster-admin` for workloads | Lesson 19 |
| Pod identity | Dedicated ServiceAccounts, disable automount where unneeded | Lesson 19 |
| Network | Default-deny NetworkPolicies, explicit allow-lists | Lesson 20 |
| Secrets | etcd encryption at rest, external secret managers | Lesson 14 |
| Admission | Pod Security Admission, OPA Gatekeeper/Kyverno | Lesson 26 |
| Images | Scan for CVEs (Trivy, Grype), enforce signed/approved-registry images | New |
| Runtime | Read-only root filesystem, non-root user, dropped Linux capabilities | New |

**Pod-level hardening — `securityContext`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

**Pod Security Admission:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**etcd encryption at rest:**
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - kms:
      name: myKmsPlugin
      endpoint: unix:///var/run/kmsplugin/socket.sock
  - identity: {}
```

## 7. Upgrade Strategies

**Kubernetes version skew policy:**
- `kube-apiserver` must be the **newest** component
- `kube-controller-manager`, `kube-scheduler` can be up to **1 minor version older**
- `kubelet`, `kube-proxy` can be up to **2 minor versions older**
- **Never skip minor versions**

**Standard upgrade sequence (kubeadm):**
```bash
kubeadm upgrade plan
kubeadm upgrade apply v1.31.0
apt-get update && apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
systemctl restart kubelet

kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# on the node: upgrade kubeadm, kubelet, kubectl
kubeadm upgrade node
systemctl restart kubelet
kubectl uncordon <node>
```

**Managed cluster upgrades (EKS/AKS/GKE):** cloud provider handles control plane; you control worker node pool upgrades, typically via rolling node replacement, reusing the cordon/drain pattern.

**Best practices:**
- Test upgrades in a non-production cluster matching production's config first
- Read the changelog/deprecation notices for every version
- Ensure PDBs are correctly configured before any upgrade involving node drains
- Upgrade during low-traffic windows even with HA

## 8. Comprehensive Comparison: The Four Pillars

| Pillar | Primary Risk Addressed | Key Mechanisms |
|---|---|---|
| **HA** | Component/node/zone failure | Multi-replica control plane, PDBs, anti-affinity, topology spread |
| **DR** | Catastrophic/total data loss | etcd snapshots, Velero, VolumeSnapshots, multi-region strategy |
| **Security** | Compromise, data exposure, policy violation | RBAC, NetworkPolicy, Secrets encryption, admission control, securityContext |
| **Upgrades** | Downtime/breakage from version changes | Version skew policy, sequential upgrades, drain/cordon, PDB-aware rollouts |

## 9. Interview Questions

**Beginner**
- Why must etcd clusters use an odd number of members?
- What's the difference between a voluntary and involuntary disruption?

**Intermediate**
- Explain the Kubernetes version skew policy and why upgrades must be sequential.
- Why is etcd backup considered the single most critical DR practice?

**Advanced / Scenario**
- Design a full DR strategy for a financial services platform requiring RPO < 5 minutes and RTO < 30 minutes across two cloud regions.
- Plan a control-plane upgrade for a 3-node HA control plane serving a zero-downtime-required application, including rollback plan.
- A security audit flags no etcd encryption, several `cluster-admin` ServiceAccounts, and no NetworkPolicies anywhere. Prioritize remediation.

## 10. Mini Quiz

1. True/False: A PodDisruptionBudget protects against a node hardware failure. -> **False**
2. What must always be the newest component in version skew policy? -> **kube-apiserver**
3. What tool is the de-facto standard for full Kubernetes cluster backup/restore? -> **Velero**

## Hands-on Lab

```bash
kubectl create deployment critical-app --image=nginx --replicas=3
kubectl apply -f - <<EOF
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical-app
EOF

kubectl drain <node-with-critical-app-pods> --ignore-daemonsets

ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db --endpoints=https://127.0.0.1:2379 ...
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db
```

## Summary

Production readiness isn't a single feature — it's the disciplined combination of everything in this curriculum: multi-replica control planes and PDBs for HA, etcd/Velero backups for DR, layered RBAC/NetworkPolicy/admission control for security, and sequential version-skew-aware processes for upgrades.


# Curriculum Complete

This document covers the full, unabridged 28-lesson Kubernetes curriculum spanning:

- **Core architecture & workloads** (Pods -> StatefulSets -> Jobs)
- **Networking** (Services -> Ingress -> Gateway API -> CNI -> kube-proxy)
- **Configuration & storage** (ConfigMaps/Secrets -> PV/PVC -> CSI)
- **Security** (RBAC -> NetworkPolicy -> Admission Control)
- **Scaling & scheduling** (Taints/Affinity -> HPA/VPA/CA)
- **Extensibility & operations** (Helm/Kustomize -> Operators/CRDs -> Monitoring/Logging -> Production hardening)

*Generated as a complete personal study reference — CKA/CKAD interview-ready.*
