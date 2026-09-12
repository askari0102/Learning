# Kubernetes: The Complete Operator's Guide

## How to use this guide

This guide is written to stand alone: every technical term is explained where it is first used, so no external lookup should be necessary. A consolidated lookup list is in [Part 36 — Glossary](#part-36--glossary), and the larger parts begin with a short list of the vocabulary they introduce, so a reader can jump straight to a term's definition or find where a concept is treated in depth.

It can be read straight through as a course — Parts 1–5 are foundations and are best read in order — or used as a reference, since each part from 6 onward is largely self-contained.

**A note on starting point.** The build-out instructions assume an *empty* cluster: one that is running and reachable, with no workloads, no Gateway API, no reverse proxy, no service mesh, no GitOps controller, no certificate manager, and no deliberately chosen storage class. That is the most common situation after a cluster is first provisioned — whether by a cloud provider, an installer, or a colleague — and it is the cleanest order to build in. Readers who have inherited a cluster with some of this already installed should treat the relevant sections as verification and consolidation steps rather than fresh installs; the reasoning, the trade-offs, and the failure modes all still apply. Part 3 covers how clusters are provisioned and how to make that reproducible, and Parts 26–27 cover the lifecycle concerns that only appear over time.

**Vocabulary used immediately, defined here so nothing is assumed:**

| Term | Meaning |
|---|---|
| **Cluster** | A set of machines managed as one unit by Kubernetes, plus the control layer that decides what runs on them. |
| **Node** | One machine in a cluster: a VM, a bare-metal server, or something smaller. Nodes are either *control plane* nodes (they decide and record) or *worker* nodes (they run workloads). [Part 2.2](#22-node-roles-control-plane-vs-worker) |
| **Control plane** | The components that store desired state, make scheduling decisions, and run the reconciliation loops. Not where applications run (normally). [Part 2.3](#23-the-control-plane-components-that-decide-and-record) |
| **kubeconfig** | The credential file that tells client tools which cluster to talk to and how to authenticate. [Part 4.1](#41-the-kubeconfig) |
| **Workload** | A generic word for an application running on the cluster — a web API, a batch job, a database. Not a Kubernetes object type. |
| **Manifest** | A YAML or JSON file declaring Kubernetes objects. [Part 5.1](#51-objects-kinds-and-the-shape-of-everything) |
| **Namespace** | A scope for names and a unit of policy — the primary way work is partitioned inside a cluster. [Part 23](#part-23--namespaces-and-multi-tenancy) |
| **Ingress** | The legacy Kubernetes API for routing external HTTP traffic to services. Superseded by Gateway API. [Part 12.3](#123-gateway-api-vs-ingress-the-honest-comparison) |
| **Gateway API** | The modern, portable set of objects describing how traffic enters the cluster, replacing Ingress. [Part 12](#part-12--the-gateway-api-in-full) |
| **Reverse proxy** | Software that sits in front of servers, accepting requests and routing them to backends. Every cluster has at least one. [Part 11](#part-11--reverse-proxies-in-full) |
| **HPA** | Horizontal Pod Autoscaler: adds or removes pod replicas based on metrics such as CPU. [Part 25.1](#251-hpa) |
| **GitOps** | Keeping the cluster's desired state in git and having a controller in the cluster continuously reconcile reality to it. [Part 21](#part-21--gitops) |
| **cert-manager** | A controller that obtains and renews TLS certificates automatically. [Part 14](#part-14--tls-and-certificates) |
| **eBPF** | A Linux kernel technology allowing programs to run safely inside the kernel — used for high-performance networking and security. [Part 9.2](#92-the-cni-layer) |
| **KEDA** | An autoscaler that scales workloads based on external events such as queue depth, and can scale to zero. [Part 25](#part-25--autoscaling-and-capacity) |
| **SPIFFE** | A standard for giving each workload a cryptographic identity, so services can authenticate each other rather than trusting network location. [Part 13.1](#131-what-a-mesh-actually-provides) |

**What this guide covers, and what it deliberately does not.** It is a complete guide to *operating Kubernetes itself*: the platform, its primitives, and the practices around them. It is deliberately not a guide to any particular vendor or domain. Specifically out of scope, so you know where to look next:

- **Cloud-provider specifics.** EKS, GKE, and AKS are named where a behavior differs (managed control planes, autoscaling, load-balancer integration), but provider consoles, IAM design, and per-provider pricing are not covered in depth — each has its own documentation.
- **Alternative orchestrators.** Nomad, Docker Swarm, ECS, and Mesos are mentioned only for comparison. Nothing here helps you choose between them.
- **Application architecture.** This guide explains what Kubernetes requires *of* your application (graceful shutdown, statelessness, readiness semantics) but not how to design a distributed system.
- **Domain-specific platform stacks.** Machine learning, data engineering, and edge computing appear where Kubernetes changes (Parts 30 and 46), not as subjects in themselves.
- **Organizational concerns.** Team topology, on-call rotation design, compliance certification, and vendor procurement are decisions the platform must support, not topics this document can make for you.
- **Windows and non-Linux control planes.** Windows worker nodes are covered (Part 44.3); the control plane is always Linux.

Where a topic is genuinely contested or the ecosystem is split, the guide says so rather than presenting one option as settled.

**Version context (verified as of writing):**

| Thing | Current | Notes |
|---|---|---|
| **Kubernetes** | **1.37** ("Garhwal", 2026-08-26) | 1.36 and 1.35 also in support (N-2 policy, ~14 months each). **1.34 EOL 2026-10-27.** |
| **Gateway API** | **v1.6** (2026-08-03) | v1.5 (released 2026-02-27) moved features to Stable; **v1.6 graduated TCPRoute + UDPRoute to Standard**. |
| **Envoy Gateway** | v1.9.x | Reference neutral Envoy-based implementation. Check the compatibility matrix before pinning. |
| **Cilium** | 1.20.x | eBPF dataplane, Gateway API, LB-IPAM, multi-pool IPAM stable. Cilium supports the three most recent minors, so 1.19 is still supported; 1.19.7 is its latest patch (1.19 EOL comes with 1.21). |
| **Istio** | 1.31 | Ambient mode (sidecar-less mesh) has been GA since **1.24** (Nov 2024). |
| **Argo CD** | 3.5.x | |
| **cert-manager** | 1.21.x | 1.20 also supported; 1.19 went EOL Jul 2026. |
| **etcd** | 3.7 | The cluster's internal database. [Part 2.3](#23-the-control-plane-components-that-decide-and-record) |
| **Ingress NGINX** | ❌ **retired** | The `Ingress` API still exists; the community reference implementation was retired (announced Nov 2025, wind-down completed during 2026). A third-party fork (Chainguard) continues it, and other NGINX-based controllers remain maintained. |

**Recent changes that affect what is worth building today:**

- **Ingress NGINX retirement** — the de-facto default ingress controller is gone. Gateway API is the direction, and there is an official **Ingress2Gateway** migration tool (1.0 released Mar 2026). [Part 12.4](#124-migrating-from-ingress)
- **`Service` `externalIPs` deprecated in 1.36.** Upstream's guidance is that all users should migrate away from it, toward a load-balancer controller or a Gateway API implementation. It is deprecated rather than removed, so existing manifests still work — but it is on the way out and cannot support dual-stack.
- **In-place Pod resize is Stable** (1.35) and **HPA scale-to-zero is Beta** (1.37) — changing a pod's CPU/memory without restarting it, and scaling a workload to zero replicas, are now supported.
- **DRA (Dynamic Resource Allocation) reached GA in 1.34** and continues to expand. It is the modern way to request GPUs, NICs, and FPGAs, replacing the older integer-resource approach. [Part 30.3](#303-gpu-resource-management--dra-is-the-modern-answer)
- **AI workloads are now first-class**: the **AI Gateway Working Group** (Mar 2026) and the **Gateway API Inference Extension** exist specifically for routing LLM inference traffic (KV-cache-aware, prefix-aware, model-aware routing). [Part 30.1](#301-inference-serving-models)
- **Pod Certificates and Cluster Trust Bundles** (1.37) give workloads API-managed cryptographic identities, reducing the need to distribute long-lived secrets for service-to-service authentication. [Part 8.2](#82-secrets-and-why-the-default-is-not-secure)

---

## Table of Contents

**Foundations**
1. [What Kubernetes Actually Is](#part-1--what-kubernetes-actually-is)
2. [Cluster Anatomy](#part-2--cluster-anatomy)
3. [How the Cluster Itself Is Built and Managed](#part-3--how-the-cluster-itself-is-built-and-managed)
4. [Accessing the Cluster](#part-4--accessing-the-cluster)
5. [The API and the Resource Model](#part-5--the-api-and-the-resource-model)

**Workloads and Configuration**
6. [Workloads](#part-6--workloads)
7. [Templating and Packaging: Helm, Kustomize, Jsonnet](#part-7--templating-and-packaging-helm-kustomize-jsonnet)
8. [Configuration and Secrets](#part-8--configuration-and-secrets)

**Networking — the long section**
9. [Networking Fundamentals, From Zero](#part-9--networking-fundamentals-from-zero)
10. [Load Balancing: The Four Layers](#part-10--load-balancing-the-four-layers)
11. [Reverse Proxies, In Full](#part-11--reverse-proxies-in-full)
12. [The Gateway API, In Full](#part-12--the-gateway-api-in-full)
13. [Service Mesh](#part-13--service-mesh)
14. [TLS and Certificates](#part-14--tls-and-certificates)
15. [DNS, Egress, and Multi-Cluster Networking](#part-15--dns-egress-and-multi-cluster-networking)

**State**
16. [Storage](#part-16--storage)
17. [Databases and Stateful Data](#part-17--databases-and-stateful-data)

**Security**
18. [Security](#part-18--security)

**Operating**
19. [Observability](#part-19--observability)
20. [Debugging](#part-20--debugging)
21. [GitOps](#part-21--gitops)
22. [CI/CD: The Actual Workflow](#part-22--cicd-the-actual-workflow)
23. [Namespaces and Multi-Tenancy](#part-23--namespaces-and-multi-tenancy)
24. [Environments: One Cluster or Many?](#part-24--environments-one-cluster-or-many)
25. [Autoscaling and Capacity](#part-25--autoscaling-and-capacity)
26. [Cluster Lifecycle Operations](#part-26--cluster-lifecycle-operations)
27. [Backup, Disaster Recovery, and Business Continuity](#part-27--backup-disaster-recovery-and-business-continuity)
28. [Multi-Cluster and Multi-Region](#part-28--multi-cluster-and-multi-region)
29. [Cost](#part-29--cost)
30. [AI/ML and GPU Workloads](#part-30--aiml-and-gpu-workloads)
31. [Platform Engineering: Building the Golden Path](#part-31--platform-engineering-building-the-golden-path)

**Reference**
32. [The Build-Out Order](#part-32--the-build-out-order)
33. [Command Reference](#part-33--command-reference)
34. [Tooling Landscape](#part-34--tooling-landscape)
35. [Anti-Patterns](#part-35--anti-patterns)
36. [Glossary](#part-36--glossary)

**Advanced and specialized topics**

37. [Advanced Scheduling](#part-37--advanced-scheduling)
38. [Ephemeral Storage, Node Pressure, and Resource Exhaustion](#part-38--ephemeral-storage-node-pressure-and-resource-exhaustion)
39. [Node Lifecycle and Maintenance](#part-39--node-lifecycle-and-maintenance)
40. [etcd and Control Plane Health](#part-40--etcd-and-control-plane-health)
41. [API Priority and Fairness](#part-41--api-priority-and-fairness)
42. [Advanced Networking: CIDR Planning, Dual-Stack, Session Affinity, and MTU](#part-42--advanced-networking-cidr-planning-dual-stack-session-affinity-and-mtu)
43. [Server-Side Apply, Field Ownership, and Generated Manifests](#part-43--server-side-apply-field-ownership-and-generated-manifests)
44. [Nodes: Runtimes, Isolation, and Heterogeneous Clusters](#part-44--nodes-runtimes-isolation-and-heterogeneous-clusters)
45. [Platform Security Features](#part-45--platform-security-features)
46. [Batch, Event-Driven, and Job Workloads](#part-46--batch-event-driven-and-job-workloads)
47. [The Optional Serverless Layer](#part-47--the-optional-serverless-layer)
48. [The Local Development Inner Loop](#part-48--the-local-development-inner-loop)

---

# Part 1 — What Kubernetes Actually Is

## 1.1 Who this guide is for, and how to read it

This guide is written for anyone who has to run Kubernetes: a developer deploying their first service, an ops engineer inheriting a cluster, an architect deciding whether to adopt it, or a student learning the ecosystem. It assumes **no prior Kubernetes knowledge** and explains every term where it is first used rather than assuming it. A consolidated lookup list lives in [Part 36 — Glossary](#part-36--glossary) for readers who hit an unfamiliar word and want the short version.

It is organized so that it can be read straight through, top to bottom, as a course — and also used as a reference. Parts 1–5 are foundations and are worth reading in order. Parts 6 onward are largely independent and can be jumped to. Each part that builds on an earlier one says so explicitly.

Two useful conventions used throughout:

- **"Desired state"** means what a manifest declares should exist. **"Observed state"** means what is actually running. Nearly every Kubernetes problem is a gap between the two.
- **Bold terms** are the vocabulary of Kubernetes. They are defined at first use, and again in the glossary.

## 1.2 Containers and images, briefly, for readers who have not used Docker

Kubernetes orchestrates containers, so the container model has to come first. Readers who already know Docker can skip to 1.3.

**A container is a process on a Linux host that has been fenced off so it cannot see or disturb most of the rest of the machine.** It is not a virtual machine. There is no second kernel and no hardware emulation: the container runs the host's kernel directly, and the isolation comes from three Linux kernel features working together:

- **Namespaces** control what a process can *see*. A process in its own PID namespace sees only its own processes; in its own network namespace it has its own network interfaces, IP addresses, and ports; a mount namespace gives it its own view of the filesystem; there are also user, UTS (hostname), and IPC namespaces. This is why two containers can both listen on port 8080 without conflict — they are in different network namespaces.
- **cgroups (control groups)** control what a process can *use* — CPU time, memory, disk I/O, and how many processes it may create. cgroups are where a container's CPU and memory limits live, and how they are enforced. Two consequences matter later: exceeding **CPU** gets a process *throttled* (slowed, not killed), whereas exceeding **memory** triggers the kernel's out-of-memory killer to terminate it. The kill is performed by the kernel's OOM killer, not by cgroups themselves — cgroups meter usage and apply the limit, and the OOM killer is what acts when memory cannot be reclaimed. (Linux has two generations of this interface, cgroup v1 and v2. Kubernetes has supported only v2 for several releases now, and all supported container runtimes default to it, so on any modern cluster you will only encounter v2.)
- **Capabilities and seccomp** control what a process is *allowed to do* even as root — dropping `CAP_NET_ADMIN`, for example, prevents it from reconfiguring the network, and seccomp filters which system calls it may make at all.

Because these are kernel features rather than emulation, containers start in milliseconds (a VM takes seconds to minutes) and are cheap enough to run hundreds per host. That cheapness is what makes the orchestration problem in 1.4 worth solving — and also why containers are a *weaker* isolation boundary than VMs, which matters in [Part 18](#part-18--security) and [Part 23](#part-23--namespaces-and-multi-tenancy).

**An image is the packaged filesystem and metadata a container starts from.** It is built as a stack of **layers**, where each layer is a filesystem diff produced by one build instruction. Layers are content-addressed and shared: if ten images all start from the same base layer, that layer is stored and downloaded once. This is why image builds are fast when only the last layer changed, and why "put your dependency installation before your source copy" is standard advice in a Dockerfile — anything that changes invalidates every layer after it.

The standard for images is the **OCI (Open Container Initiative) image specification**, and the runtime interface is the **CRI (Container Runtime Interface)**. Kubernetes does not build or run images itself; it talks to a container runtime — **containerd** or **CRI-O** in practice — through CRI. Historically Kubernetes used Docker as its runtime, via an adapter called `dockershim`; that adapter was removed in Kubernetes 1.24, which is what older migration notes mean by "Docker is not supported anymore." Images are still routinely built with Docker or BuildKit — that part was never in question — but the *runtime on the node* is now containerd or CRI-O. The practical consequence for you: `docker ps` on a node shows nothing about Kubernetes workloads, so you inspect them with `kubectl` or `crictl` instead.

**A registry** (Docker Hub, GitHub Container Registry, Amazon ECR, Google Artifact Registry, Harbor) stores and distributes images. An image reference has three parts — `registry.example.com/team/api:1.14.2` is registry + repository + tag. Tags are mutable pointers and therefore unreliable for production (someone can repoint `1.14.2` at different content tomorrow). A **digest** — `registry.example.com/team/api@sha256:abc123...` — is a cryptographic hash of the image content and is immutable. Pinning by digest is the reproducible choice; see [Part 22](#part-22--cicd-the-actual-workflow).

## 1.3 What a container does not solve

Running a container by hand on one machine is genuinely useful and also completely insufficient for running software that matters, because every hard problem is still unsolved:

- The container process died. Who restarts it?
- The machine died. Who moves the container somewhere else?
- Traffic tripled. Who runs more copies, and how do they get discovered?
- A new version must ship without dropping requests. Who sequences the replacement, health-checks it, and rolls back if it is bad?
- Ten services must find each other, and their addresses change every time they restart. Who tracks that?
- Configuration, credentials, TLS certificates, storage volumes, and resource limits must be handled across forty machines. Who schedules all of it, and who decides which container goes on which machine?

These could be answered with shell scripts, systemd units, and cron. Organizations did exactly that for years. The scripts rot, the failure modes multiply, and eventually the organization has written a bad, undocumented, unportable orchestrator. Kubernetes is that orchestrator, written once, by many people, with an API — and with an ecosystem built on top of it.

## 1.4 The orchestration problem, stated precisely

Kubernetes is a **declarative, level-triggered, distributed state machine for containers**. Each phrase carries weight, and together they are the entire mental model.

**Declarative — you describe the destination, not the route.** A manifest says "three copies of this image should exist, reachable on port 8080, with these resource limits." It never says "start a container, then wait, then register it, then add it to a load balancer." Kubernetes works out the steps, in whatever order current reality demands, and re-works them out later if reality changes. The practical difference from an imperative script is enormous: a script is correct only for the situation its author imagined, while a declaration is a specification that can be re-satisfied after any amount of chaos.

**Level-triggered — the logic compares states rather than reacting to the change that happened.** This is the subtle phrase and the reason Kubernetes is reliable, but the usual one-line version of it ("Kubernetes does not use events") is wrong in a way that causes real confusion later, so it is worth getting exactly right.

The distinction upstream draws is between **level-based** and **edge-based** *logic*. Kubernetes is level-based: a controller's decision depends on the current state it can observe, not on the specific transition that occurred. When a container dies, the controller does not reason about "a death happened" — it compares desired three against observed two and acts to close the gap.

But the *transport* is absolutely an event stream. Controllers, the scheduler, and the kubelet learn that something changed by **watching** the API server (`list` to get the current state, then `watch` for change notifications). So Kubernetes does respond to events — it simply does not depend on any individual event being received. That is the property that makes it robust: if a controller is down for ten minutes, or misses a notification, it re-lists, compares, and reconciles. **Correctness is independent of the notification channel; only latency depends on it.**

One terminology trap: the watch notifications above are not the same thing as the **`Event` API objects** you see in `kubectl describe`. Those are informational records written to the cluster for humans to read. Losing them changes nothing about correctness, which is why they are allowed to expire.

**Distributed state machine — many small loops sharing one source of truth.** Dozens of independent **controllers** each own a slice of the world: one manages Deployments, another ReplicaSets, another nodes, another routes. They do not call each other. They read from and write to one central API server, communicating only by changing shared state. This is architecturally similar to a database with triggers, and it is why Kubernetes can be extended to manage things its authors never imagined — a controller for PostgreSQL, for certificates, for DNS records, for cloud load balancers — without modifying Kubernetes itself. [Part 5.4](#54-extending-the-api-crds-controllers-operators-webhooks) shows how to add one.

The practical payoff of all three properties together: **deployment logic becomes versionable, reviewable, diffable, and revertable.** A `git revert` becomes a deployment mechanism, because the declaration lives in git and the controller will re-satisfy it. That single property is what makes GitOps ([Part 21](#part-21--gitops)) possible, and it is the strongest reason to prefer Kubernetes over hand-rolled automation even at small scale.

## 1.5 How it differs from plain Docker / Compose

"I know Docker" is the most common source of wrong intuitions in Kubernetes, so the differences are worth laying out explicitly.

| Concern | Docker / Compose | Kubernetes |
|---|---|---|
| Unit of work | Container | **Pod** (one or more containers sharing network + IPC + lifecycle) |
| Scope | One host | A fleet of machines (**nodes**) |
| Placement | You choose the host (or Compose picks locally) | The **scheduler** chooses, based on resource requests, affinity, taints, and current load |
| Desired state | `docker run` is imperative; Compose is declarative but single-host with no reconciliation | Declarative **and continuously reconciled** |
| Restart on failure | `--restart=always` (host-level, crude) | Controllers recreate pods; `restartPolicy` governs one pod's containers |
| Service discovery | Container names, manual links, external LB | **Service** objects plus cluster DNS: a stable virtual IP and DNS name in front of changing pods |
| Configuration | `-e`, `--env-file`, bind mounts | **ConfigMap** and **Secret** objects, injected as env vars or mounted files |
| Storage | Volumes and bind mounts, tied to the host | **PersistentVolume** / **PersistentVolumeClaim** with CSI drivers — storage is decoupled from any machine |
| Scaling | Manual, per host | `replicas: N`, plus **HPA**/KEDA and node autoscaling |
| Rolling updates | Scripted by hand | **Deployment** does it natively with `maxSurge`/`maxUnavailable`, including rollback |
| Health checking | Manual, or a weak image-level `HEALTHCHECK` | **Liveness, readiness, and startup probes** as first-class objects, and readiness actually controls traffic |
| Networking | Bridge networks, published ports | A **CNI** plugin provides a flat pod network; **NetworkPolicy** provides a firewall |
| Secrets | Env vars and files, in plaintext | `Secret` objects — base64-encoded, and **not encrypted by default** ([Part 8.2](#82-secrets-and-why-the-default-is-not-secure)) |
| Multi-tenancy | None | Namespaces, RBAC, quotas, and policy |
| Extensibility | Plugins | **CRDs and controllers** — an entire ecosystem |

Three conceptual jumps cause most of the confusion:

**Jump 1 — a Pod is not a container.** A Pod is a group of containers that are always placed on the *same* machine, share a network namespace (so they share one IP address and one port space — two containers in the same pod cannot both bind port 8080), and can share storage volumes. Most pods contain exactly one container. Extra containers in a pod are **sidecars** — an auxiliary process that supports the main one, such as a log forwarder or a proxy — or **init containers**, which run to completion *before* the main containers start and are used for setup tasks like database migrations or fixing file permissions. A database and the application that uses it should *not* share a pod: they have different lifecycles, different scaling needs, and different failure characteristics.

**Jump 2 — pods are mortal, and their identities are not stable.** A pod is not a durable object. It receives a new IP address every time it starts. It is destroyed and recreated on every deploy, on node maintenance, and whenever the node it lives on runs short of resources. Nothing may depend on a pod's IP or name persisting. This is precisely why **Services** exist: a Service is a stable virtual IP and DNS name that always points at whatever pods are currently healthy and match its label selector. Anything that treats pods as long-lived — writing state to the container filesystem, hardcoding an IP, expecting a specific pod to exist tomorrow — will break, and it will break at the least convenient time.

**Jump 3 — the operational model is "observe and reconcile," not "log in and fix."** There is no supported workflow of SSHing into a machine and editing configuration. Instead, an operator inspects declared state, observed state, and the event stream, determines which layer disagrees, and changes the *declaration* (or fixes the underlying cause). This is a different skill from traditional server administration, and it is the one skill that most repays investment; [Part 20](#part-20--debugging) is devoted to it.

## 1.6 What Kubernetes is not

Expectations matter as much as capabilities, because most Kubernetes disappointments come from expecting something it never claimed to provide.

- **Not a platform-as-a-service.** Kubernetes supplies primitives, not a finished platform. There is no built-in CI system, no built-in image registry, no built-in logging or dashboard product, no built-in database service. Everything beyond the primitives is assembled by the operator — which is exactly why [Part 31](#part-31--platform-engineering-building-the-golden-path) exists.
- **Not a database.** Kubernetes schedules processes; it does not provide data guarantees. Databases can run on it via **StatefulSets** and **operators**, and that is a legitimate choice in specific circumstances, but "can" and "should" differ. See [Part 17](#part-17--databases-and-stateful-data).
- **Not a fix for application design.** If an application takes four minutes to become ready, ignores `SIGTERM` (the signal Kubernetes sends to ask a process to shut down cleanly), is not **idempotent** (meaning that performing the same operation twice has the same effect as performing it once), and cannot run two copies at once, Kubernetes will faithfully amplify those problems. Applications must be *designed to be orchestrated*: stateless where possible, fast to start, graceful on shutdown, tolerant of dependencies disappearing.
- **Not a security boundary by default.** A namespace is not a virtual machine. Isolation is assembled from RBAC, NetworkPolicy, Pod Security Admission, admission policy, runtime security, and the kernel features from 1.2. Multi-tenant clusters in particular require deliberate work ([Part 23](#part-23--namespaces-and-multi-tenancy)).
- **Not a reverse proxy, load balancer, or firewall.** Kubernetes standardizes how to *ask* for those things — through Service, Gateway API, and NetworkPolicy objects — but the actual proxying, balancing, and filtering is performed by software that must be installed and operated ([Parts 10–13](#part-10--load-balancing-the-four-layers)). This distinction surprises people more than any other, and it is why "I created a Service, why is nothing reachable from the internet?" is such a common early question.

---

# Part 2 — Cluster Anatomy

This part answers the question "what is actually running where?" Understanding the physical and logical layout of a cluster makes every later topic — scheduling, security, upgrades, debugging — far easier to reason about. It also defines the vocabulary (node, taint, QoS, eviction) that the rest of the guide depends on.

## 2.1 The physical view: what a cluster is made of

A **cluster** is a set of machines, called **nodes**, plus a control layer that decides what runs on them. In a cloud environment the machines are usually virtual machines; on-premises they may be bare metal; on a laptop or in a homelab they may be tiny VMs or Raspberry Pis. Kubernetes does not care. To Kubernetes, each node is a machine with some CPU, memory, disk, and a network address.

There is one more piece of shared infrastructure that is not a node: the **network fabric** that lets all nodes and pods talk to each other, and (in cloud environments) the APIs that let Kubernetes create load balancers, disks, and DNS records. Kubernetes delegates nearly all of that to plugins.

Two terms appear constantly and are worth fixing now:

- **Workload** — an application running on the cluster. It is a generic word covering "a web API," "a batch job," "a database," and so on. It is not a Kubernetes object type.
- **Component** — a Kubernetes program (as opposed to a workload). The API server, the scheduler, and the kubelet are components.

## 2.2 Node roles: control plane vs worker

Not every node does the same job. Kubernetes, and the tool that installed it, assign each node a **role**. In modern usage the roles are:

| Role | Purpose | Typical count |
|---|---|---|
| **Control plane** | Runs the components that *decide* and *record* what should happen. Does not (by default) run application workloads. | 1 in development, 3 or 5 in production, because control-plane components must survive a machine failure. |
| **Worker** | Runs application workloads — the pods. | 2 to thousands, scaled with demand. |

The older vocabulary, still visible in older documentation, blog posts, and CLI output, is **master node** for control plane and **minion** or **slave node** for worker. The two words have different histories. **"Minion" was replaced by "node" in 2016**, very early in the project's life. **"Master" persisted far longer** and was only retired in the 1.20–1.24 era, when the label `node-role.kubernetes.io/master` became `node-role.kubernetes.io/control-plane`. When reading older material, "master" always means "control plane."

Whether workloads *can* run on a control-plane node is a policy decision, and the default differs by distribution — this is a real source of confusion when moving between clusters:

- **kubeadm** (the reference self-managed installer) applies a **taint** to control-plane nodes that prevents ordinary workloads from landing there. This is the behavior most documentation assumes.
- **Managed cloud clusters** (EKS, GKE, AKS and similar) hide the control plane entirely: those machines are not nodes in the cluster at all, and there is nothing to taint or inspect. The cluster consists only of worker nodes.
- **Lightweight and single-node distributions** often allow workloads on the control-plane node deliberately, because the common deployment is a single machine. Examples include **k3s**, where the server node is schedulable by default, and **Canonical Kubernetes**, which documents that it lets workloads run on control-plane nodes, explicitly noting this differs from kubeadm and recommending isolation for production multi-node clusters ([Canonical Kubernetes node roles](https://documentation.ubuntu.com/canonical-kubernetes/release-1.32/snap/explanation/roles/)).

The practical consequence: **"why is my pod scheduled onto the control-plane node?" is answered by inspecting the node's taints, not by assuming a default.** The command is in 2.7.

Production guidance, regardless of distribution: **keep the control plane isolated from workloads.** Control-plane components contend for the same CPU, memory, and disk as workloads; a workload that saturates a control-plane node can make the *entire cluster* unresponsive, because that node may be hosting the scheduler or a control-plane data store. Isolation also reduces the blast radius of a compromised workload, since node-level access on a control-plane node is closer to cluster-admin than node-level access on a worker.

## 2.3 The control plane: components that decide and record

The **control plane** is the decision-making layer. It stores desired state, watches reality, decides what should happen, and records everything. It does not run application containers. Its components can run as normal processes on machines (self-managed clusters) or as pods in a dedicated namespace (many self-managed distributions) — or they may not be visible at all (managed clouds).

| Component | What it does | Why it matters to an operator |
|---|---|---|
| **kube-apiserver** | The single front door to the cluster. Every read and write goes through it. It authenticates the caller, authorizes the request, runs admission checks, validates the object, and persists it. It is the only component that talks to the data store. | If the API server is unhealthy, the cluster is effectively down even if workloads keep running. Everything — `kubectl`, controllers, kubelets — depends on it. It is stateless and can be scaled horizontally. |
| **etcd** | The cluster's database: a distributed, consistent key-value store. All desired and observed state lives here. | **The most important thing to back up.** Losing etcd means losing the cluster's memory of itself. It requires a **quorum** — a majority of its members must agree before accepting a write — which is why it is run with an odd number of members (3 tolerates 1 failure, 5 tolerates 2) and why 2 members is strictly worse than 1 or 3. |
| **kube-scheduler** | Decides which node each new pod runs on (detail in 2.8). | When pods sit in `Pending`, the scheduler's decisions — or its refusal to decide — are the explanation, and the reason appears in pod events. |
| **kube-controller-manager** | Runs the built-in control loops: the Deployment controller, ReplicaSet controller, Node controller, Job controller, EndpointSlice controller, and dozens more. | This is where "reconciliation" physically happens. It is one binary containing many independent loops. |
| **cloud-controller-manager** | Translates Kubernetes objects into cloud-provider actions: provisioning a load balancer for a `Service` of type `LoadBalancer`, attaching disks, managing node lifecycle and routes. | Present only in cloud clusters. It is the component that makes `type: LoadBalancer` do anything, and its absence is why that Service type hangs with no address on bare metal ([Part 12.2](#step-2--check-that-a-loadbalancer-can-actually-get-an-ip)). |
| **DNS provider** (usually CoreDNS) | Serves names like `api.prod.svc.cluster.local` to every pod. | Runs as an ordinary workload rather than as part of the control plane. Be careful with the implication: a query goes to the `kube-dns` Service and is forwarded to a **CoreDNS pod**, so name resolution *is* pod-to-pod traffic. The accurate relationship is that **DNS depends on pod networking, not the reverse** — a pod can reach another pod by IP even when DNS is broken, which is why "does it work by IP?" is such a useful debugging question ([Part 9.3](#93-dns-coredns)). |

**High availability.** With three control-plane nodes, each runs its own API server, scheduler, and controller-manager plus one etcd member. The API servers sit behind a load balancer; the scheduler and controller-manager use leader election so only one instance is active at a time (the others are hot standbys). This is why a control plane can lose one machine without downtime, and why a single-control-plane cluster is a development-only configuration: losing that machine means no scheduling, no reconciliation, and no API access until it returns.

## 2.4 Worker nodes: components that run workloads

Every node, worker or control plane, runs these two components. They are why a node is a node.

**kubelet** — the node's agent. It is a system service, not a container. It receives from the API server the set of pods assigned to its node, and it is responsible for making them real: pulling images, mounting volumes, creating the pod's network namespace, starting and stopping containers, running the health probes, reporting status and resource usage back, and enforcing the node's eviction policy (2.6). If the kubelet stops, the node stops being healthy within a minute or so and is marked `NotReady`.

**Container runtime** — the program that actually creates and runs containers, reached through the **CRI** (Container Runtime Interface). In practice this is **containerd** or **CRI-O**. The kubelet does not implement containerization; it delegates. ([Part 1.2](#12-containers-and-images-briefly-for-readers-who-have-not-used-docker) covers images and runtimes.)

**kube-proxy** (somewhat optional now) — the component that historically implemented Service networking on each node by programming packet-forwarding rules so that traffic to a Service's virtual IP reaches a backing pod. It runs in one of several modes (iptables, nftables, or **IPVS** — IP Virtual Server, the kernel's L4 load balancer). In many modern clusters it is **removed entirely** and replaced by eBPF-based service handling in the CNI — Cilium calls this "kube-proxy replacement." Which one is in use matters for debugging, because the place to look for Service routing problems differs ([Part 20.3](#203-service-and-networking-debugging)).

**CNI plugin** — a program on each node that assigns pod IP addresses and wires up connectivity. The CNI is what makes the flat pod network real ([Part 9.2](#92-the-cni-layer)). It is usually deployed as a DaemonSet, meaning one pod per node.

**CSI node plugin** — the per-node half of a storage driver, responsible for attaching and mounting volumes on that machine ([Part 16](#part-16--storage)). Also typically a DaemonSet.

So a typical worker node is running: the kubelet and container runtime as native services, plus several DaemonSet pods (CNI, CSI node plugin, log collector, metrics agent, and any security agent). This is worth remembering when budgeting node capacity: **DaemonSets consume resources on every node**, and they are usually excluded from the accounting that "how much room do I have?" calculations show.

## 2.5 Labels, selectors, and node roles in practice

A node's role is not a special field. It is a **label** — a key/value pair attached to the object for the purpose of selection. Control-plane nodes carry `node-role.kubernetes.io/control-plane: ""`, some distributions also label workers with `node-role.kubernetes.io/worker: ""` (this is not universal — do not depend on it), and all nodes carry standard labels describing hardware and topology:

```
kubernetes.io/hostname: worker-1
kubernetes.io/os: linux
kubernetes.io/arch: amd64
topology.kubernetes.io/region: eu-west-1
topology.kubernetes.io/zone: eu-west-1a
node.kubernetes.io/instance-type: m6i.2xlarge
```

These labels are how pods express *where* they are willing to run, and they are the foundation of scheduling ([Part 6.6](#66-resources-qos-and-scheduling)). The zone and region labels in particular are used automatically by Kubernetes to spread pods across failure domains.

## 2.6 Taints and tolerations: how a node refuses work

This is worth explaining properly, because "taint" is often described in one sentence and then relied upon for the rest of a guide.

**A taint is a property of a node that says "do not schedule pods here unless they explicitly opt in."** It is the node side of the decision. A **toleration** is the matching opt-in, declared on the pod. Both are needed for a pod to land on a tainted node: the node must be marked, and the pod must tolerate the mark.

The mental model that makes this click: **taints repel, tolerations permit — and tolerations do not attract.** A pod that tolerates `workload=database:NoSchedule` is *allowed* on database nodes, not *sent* there. Attraction is a different mechanism (`nodeAffinity` or `nodeSelector`, [Part 6.6](#66-resources-qos-and-scheduling)). This distinction explains the very common mistake of adding a toleration and finding the pod scheduled elsewhere: tolerating a taint is permission, not a destination.

A taint has three parts: **key**, optional **value**, and **effect**. The effect is the interesting part, because it determines *what* is prevented:

| Effect | Behavior |
|---|---|
| `NoSchedule` | New pods that do not tolerate it will not be scheduled onto the node. Pods already running there are **left alone**. |
| `PreferNoSchedule` | A soft preference: the scheduler avoids the node if it can, but will use it if there is nowhere better. Useful for gradual consolidation. |
| `NoExecute` | The strict one: new pods will not be scheduled, *and* existing pods that do not tolerate it are **evicted** (removed) from the node. |

The `NoExecute` effect is how Kubernetes automatically recovers from node failures: when a node stops reporting, the node controller applies taints such as `node.kubernetes.io/not-ready:NoExecute`, and pods that do not tolerate them are eventually evicted and rescheduled elsewhere. **Toleration duration matters here** — a toleration may specify `tolerationSeconds`, meaning "tolerate this for N seconds, then evict me," which is how Kubernetes avoids a thundering herd when a node has a brief network blip.

Declaring a toleration on a pod looks like this:

```yaml
spec:
  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "database"
      effect: "NoSchedule"
    # or the blunt version — tolerate ANY taint (used by DaemonSets):
    - operator: "Exists"
```

The common taints an operator will actually meet:

| Taint | Applied by | What it means |
|---|---|---|
| `node-role.kubernetes.io/control-plane:NoSchedule` | kubeadm and similar installers | Keep ordinary workloads off control-plane nodes |
| `node.kubernetes.io/not-ready:NoExecute` | Node controller (automatically) | The node is not reporting; its pods are being evicted |
| `node.kubernetes.io/unreachable:NoExecute` | Node controller (automatically) | The node cannot be contacted |
| `node.kubernetes.io/memory-pressure:NoSchedule` | The **control plane** (node controller), on the kubelet's report | The node is low on memory; stop adding pods |
| `node.kubernetes.io/disk-pressure:NoSchedule` | The **control plane** (node controller), on the kubelet's report | The node is low on disk |
| `node.kubernetes.io/unschedulable:NoSchedule` | The **control plane**, when `spec.unschedulable` is set (by `kubectl cordon` or draining) | An operator is deliberately preventing new pods (2.9) |
| `workload=database:NoSchedule`, `nvidia.com/gpu=present:NoSchedule` | An operator, deliberately | Reserve specific machines for specific workload types |

**Why taints are used deliberately.** The three legitimate uses are:

1. **Isolation** — keep general workloads off control-plane nodes, or off nodes running something fragile.
2. **Dedication** — reserve expensive hardware (GPUs, high-memory machines) so that only workloads that need it consume it, and so it is not wasted on a random web server.
3. **Draining** — take a machine out of service for maintenance without killing running work immediately (2.9).

**A caution about dedicated nodes:** the standard pattern is to taint a node *and* give the intended workload both a toleration and a node affinity or node selector. With a toleration alone, other workloads that happen to have a broad toleration (many system components tolerate everything) can still land there, and the dedicated capacity is not actually dedicated.

## 2.7 Schedulability: the practical checklist

When a pod is not being scheduled, the answer is one of a small number of conditions. Each can be inspected directly.

**Is the node willing to accept pods?**

```bash
kubectl get nodes -o wide
kubectl describe node <node> | grep -A3 Taints
```

Three things can make a node refuse work: a **taint** the pod does not tolerate, being **cordoned** (`spec.unschedulable: true`), or being **NotReady** (which itself applies taints automatically as shown above).

**Does the pod fit?** The scheduler only places a pod on a node whose **allocatable** resources cover the pod's **requests** (requests and allocatable are explained in [Part 6.6](#66-resources-qos-and-scheduling); the short version is that requests are what the scheduler reserves, and allocatable is what is left on the node after the operating system and Kubernetes system daemons take their share). `kubectl describe node <node>` prints an "Allocated resources" section showing the sum of existing requests against capacity — the quickest way to see that a node is full.

**Does the pod have somewhere it is allowed to go?** `nodeSelector`, `nodeAffinity`, `podAffinity`/`podAntiAffinity`, and topology spread constraints can all make a pod unschedulable if the required nodes do not exist or do not have room.

**Is there a volume problem?** A pod with a PersistentVolumeClaim bound to a volume in one zone cannot run in another. This looks like a scheduling failure and is one of the more confusing ones ([Part 16.1](#161-the-mental-model)).

**Is there a limit on how many pods a node may host?** Kubernetes defaults to a maximum number of pods per node (commonly 110), and cloud CNIs impose their own limits — notably the AWS VPC CNI, where each pod consumes an IP address from the node's elastic network interfaces. Hitting that limit produces `Pending` pods even on an apparently empty node ([Part 20.4](#204-node-and-resource-debugging)).

The definitive tool is the event stream, which states the scheduler's reasoning verbatim:

```bash
kubectl describe pod <pod> | sed -n '/Events/,$p'
# Typical messages:
#   0/5 nodes are available: 2 node(s) had untolerated taint {workload: database},
#   3 Insufficient memory.
#   0/5 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

## 2.8 How the scheduler actually decides

The scheduler's job is to answer "which node?" for each newly created pod. It does this in two phases per candidate node.

**Phase 1 — Filtering (predicates).** Every node is checked against hard requirements, and nodes that fail any check are eliminated. The checks include: does the node have enough allocatable CPU and memory for the pod's requests; do the node's taints all have matching tolerations; does the pod's node affinity/selector match the node's labels; are the pod's anti-affinity rules satisfiable; would placing the pod violate a topology spread constraint; is a required volume available in this node's zone; is the node cordoned; has the node reached its pod-count limit. Whatever remains is the candidate set. **If nothing remains, the pod stays `Pending` and the reasons are listed in its events** — which is why "read the events" is the correct first move for an unschedulable pod.

**Phase 2 — Scoring.** Surviving nodes are ranked by soft preferences, and the default set is worth knowing because it is not what most people assume. It includes: preferring nodes that **already have the container image cached** (`ImageLocality`, faster start), respecting taint tolerations and node affinity preferences, **spreading** pods rather than packing them (`PodTopologySpread`), preferring nodes with more free resources (`NodeResourcesFit` with the default `LeastAllocated` strategy), and balancing resource types so one dimension is not exhausted.

Notice what is **not** in that list: there is no default preference for placing pods of the same service in the same zone. If anything the defaults spread them. Co-location only happens if the pod explicitly asks for it with a preferred `podAffinity`, which is why "the scheduler packs my service into one zone to save latency" is not a behaviour you can assume. (If you *want* same-zone traffic, the mechanism is [`trafficDistribution: PreferSameZone`](#425-service-topology-and-cross-zone-traffic) on the Service, not the scheduler.)

The highest-scoring node wins, with ties broken randomly. The assignment is then written to the pod's `spec.nodeName`, which is the signal the kubelet on that node acts on.

Two consequences worth internalizing:

- **Requests, not limits, drive placement.** A pod declaring `requests: {cpu: 100m}` occupies 100 millicores of the node's budget regardless of how much CPU it actually uses. Under-requesting leads to overcommitted nodes and evictions (2.9); over-requesting leads to wasted capacity and unschedulable pods. This is why requests are the single most consequential field in a pod spec.
- **Scheduling happens once.** Once a pod is assigned to a node, the scheduler does not revisit the decision. If conditions change — the node fills up with other pods, or the pod's usage grows — nothing moves it. Rebalancing is the job of a separate component (the descheduler) or of the pod being recreated.

## 2.9 Node lifecycle, cordoning, draining, and eviction

Nodes have a lifecycle, and several of its states look like failures but are deliberate.

**Registration.** A node joins the cluster either by an administrator explicitly creating a `Node` object or by the kubelet registering itself (which is the modern default, with approval handled by the node authorizer and, in stricter setups, by manual approval of CertificateSigningRequests). On registration, the node reports its capacity: how much CPU, memory, and disk it has, how many pods it can host, and its labels and addresses. The Node controller monitors each node's heartbeats; missing heartbeats eventually mark it `NotReady`.

**Cordon** — mark a node unschedulable so no *new* pods land on it, while existing pods keep running.

```bash
kubectl cordon <node>        # adds unschedulable: true and the unschedulable taint
```

**Drain** — cordon plus actively remove the pods so maintenance can be performed. For each pod, draining asks the API server to evict it (which respects PodDisruptionBudgets, see below), and the pod is normally rescheduled elsewhere by its controller.

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

The two flags exist because draining is not always possible: **DaemonSet pods cannot be moved** (their controller would simply recreate them on the same node), so they must be skipped with `--ignore-daemonsets`; and pods using `emptyDir` volumes would lose their data, so Kubernetes refuses to proceed until `--delete-emptydir-data` acknowledges that loss. A pod with no owning controller will also block a drain until `--force` is given — and forcing it means that pod will not come back.

**A drain that hangs is usually a PodDisruptionBudget doing its job.** A **PDB** declares a minimum availability for a set of pods (for example, "at least 2 replicas of the API must always be running"), and eviction requests that would violate it are refused. If a workload has 2 replicas and a PDB requiring `minAvailable: 2`, then *zero* pods may ever be evicted, and `kubectl drain` waits forever. This is not a bug; it is the cluster reporting that the workload cannot tolerate any voluntary disruption. The fix is to add replicas or relax the PDB ([Part 6.6](#66-resources-qos-and-scheduling)).

The distinction between **voluntary** and **involuntary** disruption matters throughout Kubernetes:

- **Voluntary** — an operator (or an autoscaler) is deliberately moving pods: draining a node, scaling down, upgrading. **PodDisruptionBudgets apply**, and they are the mechanism that makes maintenance safe.
- **Involuntary** — hardware fails, memory runs out, a node disappears. PDBs do not apply; nothing can negotiate with a dead machine. Only replication and anti-affinity save you here, which is why running multiple replicas *spread across zones* is the only real protection.

## 2.10 Node pressure, QoS, and eviction

A node can run out of resources. When it does, the kubelet must remove pods to protect the machine, and the rules it follows are precise. This section defines the terms that appear throughout the rest of the guide.

### Node conditions

A **condition** is a named status flag on an object, with a value (`True`/`False`/`Unknown`), a reason, and a message. Nodes report a standard set, and each is a self-describing statement about the machine's health:

| Condition | Meaning when `True` |
|---|---|
| `Ready` | The node is healthy and able to accept pods (the others are all problems) |
| `MemoryPressure` | The node is low on available memory |
| `DiskPressure` | The node is low on available disk space, or on inode count |
| `PIDPressure` | The node is running out of process IDs — usually a fork bomb or a runaway process count |
| `NetworkUnavailable` | The node's network is not correctly configured (often the CNI is not ready) |

**Who does what here matters.** The **kubelet** detects and *reports* the conditions; the **node controller in the control plane** is what maps conditions into taints — it "automatically creates taints with a `NoSchedule` effect for node conditions." So the taint you see on a pressured node was applied by the control plane on the kubelet's report, not by the kubelet itself.

All three pressure conditions — memory, disk, and **PID** — produce a taint and can drive eviction. The memory and disk conditions have **two distinct effects**, which are easy to fuse into one: the taint stops *new* pods from arriving, and eviction *relieves* pressure from pods already running. The taint does the preventing; eviction does the relieving. **`PIDPressure` deserves more respect than it usually gets:** it is easy to dismiss as "a bug," but `pid.available` is a first-class eviction signal, and a container that spawns processes without limit will exhaust the node's PID table and take unrelated pods down with it. Because there is no "PID request" to compare against, eviction under PID starvation is ranked by pod priority rather than by request usage.

```bash
kubectl describe node <node> | sed -n '/Conditions/,/Addresses/p'
```

Under the hood, the kubelet decides based on configurable **eviction thresholds** (expressed either as absolute values or as percentages of capacity, with separate soft and hard thresholds). A **soft** threshold triggers eviction with a grace period; a **hard** threshold triggers immediate eviction. These are configurable per node, which is why the exact moment a node starts evicting can differ between a managed cluster and a self-managed one.

### What "QoS" means

**QoS (Quality of Service) here does not mean network prioritization** — the term is unfortunately overloaded across computing. In Kubernetes, **QoS class is a label Kubernetes assigns automatically to every pod, based on how the pod's containers declare their CPU and memory requests and limits. It determines the order in which pods are evicted when a node is under pressure, and how the kernel schedules them relative to each other.**

Two definitions are needed first:

- A **request** is the amount of CPU or memory the pod is guaranteed. It is what the scheduler reserves when placing the pod, and what the node's capacity accounting counts. The kernel uses it as a relative weight when CPU is contended — a pod requesting 200m gets twice the CPU share of a pod requesting 100m under pressure.
- A **limit** is the maximum the pod may consume. Exceeding a CPU limit results in **throttling** (the process is slowed, not killed). Exceeding a memory limit results in the container being **OOM-killed** (terminated by the kernel's out-of-memory killer) and restarted according to its restart policy. Memory is not compressible; CPU is. That asymmetry drives most resource-tuning decisions.

Kubernetes assigns one of three classes:

| QoS class | Assignment rule | Kernel CPU treatment | Eviction under pressure |
|---|---|---|---|
| **Guaranteed** | Every container in the pod has requests equal to limits, for both CPU and memory | Weighted by its **request** — like every pod; not by class | **Last** |
| **Burstable** | At least one container declares a **request or a limit**, and requests do not equal limits | Weighted by request | **Middle** — unless usage stays inside its request, in which case it groups with `Guaranteed` |
| **BestEffort** | **No** container declares any request **or limit** | Minimum share; starved under contention, not throttled | **First** — but only because a zero request is exceeded by any usage at all |

Read that last column as a *consequence*, not a rule. The reality, and the reason the conceptual section below exists: **the kubelet does not consult the QoS class when choosing eviction victims.** The class happens to correlate with the real ranking keys, and correlation is all it is.

### How eviction actually chooses victims

The kubelet ranks victim pods to minimize damage to the *node* while removing the least-committed work first. **It does not rank by QoS class** — a widespread misconception, which is why the table above is labelled as a consequence rather than a rule. The documented ranking keys, in order, are:

1. **Whether the pod's usage exceeds its request.** This is the dominant factor. A pod consuming more than it asked for is a candidate for eviction; a pod living within its request is not.
2. **Pod priority** (`priorityClassName`).
3. **How far usage exceeds the request** — the magnitudes, used to sort within the above groups.

The practical consequence is the part worth remembering: **a `Burstable` pod that stays within its request is evicted in the same group as a `Guaranteed` pod**, and a `BestEffort` pod that somehow stays within its (zero) request is still evicted early only because it has nothing reserved. What actually protects a pod is **declaring a request and staying inside it**, not the label the cluster derives from that declaration.

Two further caveats. **Priority can override everything above** — a higher-priority pod is spared in favour of a lower-priority one. And QoS ordering does not apply at all to **inode or PID starvation** (there are no requests to compare) or to `ephemeral-storage`; those cases are ranked by priority alone.

### The practical conclusions

The chain of reasoning is what matters, not the ordering table:

- **Setting requests properly is not bookkeeping.** Requests determine where pods can be scheduled, how much CPU they get relative to neighbours under contention, and whether they are evicted first or last. A pod with no requests is `BestEffort`: it will be throttled first, evicted first, and it cannot participate meaningfully in autoscaling, because autoscaling targets are expressed as a percentage of requests ([Part 25.1](#251-hpa)).
- **Setting memory limits is mandatory; setting CPU limits is a judgement call.** Because memory is incompressible, a missing memory limit means a runaway pod can consume the node and trigger evictions of *other* pods. Because CPU is compressible, a CPU limit converts contention into throttling, which often causes latency problems worse than the contention itself — hence the common practice of setting CPU requests without CPU limits.
- **`Guaranteed` is the right class for things you cannot afford to lose** — databases, gateways, and control-plane-adjacent workloads. Set requests equal to limits, for CPU and memory.
- **`BestEffort` is almost always a mistake**, even for trivial workloads, because the failure mode (silent eviction under pressure, with the pod rescheduled or lost) is disproportionate to the small saving in configuration effort. If a workload is genuinely unimportant, say so with a low priority class rather than by omitting its requirements.

Priority classes, PodDisruptionBudgets, and the full resource model are covered in [Part 6.6](#66-resources-qos-and-scheduling).

## 2.11 Distributions: which software built this cluster, and why it matters

"Kubernetes" is a specification plus a set of components; a **distribution** is a particular way of assembling, installing, and operating them. Which one is in use changes what is visible, what is configurable, and who is responsible for upgrades.

| Distribution | Model | Control plane runs workloads by default? | Who patches the control plane? | Typical use |
|---|---|---|---|---|
| **Managed cloud (EKS, GKE, AKS, OKE)** | Provider operates the control plane; the operator gets only worker nodes | Not applicable — no control-plane nodes exist in the cluster | The provider | Most production clusters |
| **Managed with managed nodes** (EKS Auto Mode, GKE Autopilot, AKS Automatic) | Provider also provisions and patches nodes | Not applicable | The provider | Teams that want the least operational surface |
| **kubeadm** | Reference self-managed installer | No — control plane is tainted | The operator | Self-managed clusters, the basis of many others |
| **k3s / RKE2** | Lightweight single-binary; SQLite by default in k3s (etcd supported) | **Yes** by default in k3s | The operator | Edge, IoT, homelab, small on-prem |
| **Talos** | Immutable, API-managed OS whose only purpose is running Kubernetes | Configurable; isolated by default in multi-node setups | The operator | Security-conscious on-prem and edge |
| **Canonical Kubernetes** | Snap or charm based | **Yes**, documented as differing from kubeadm | The operator | Ubuntu-centric on-prem |
| **kind / minikube / k3d** | Local clusters in containers or VMs | Usually yes (single node) | The operator (recreate) | Development and testing only — never production |

**Why the distribution matters to an operator:**

- **What you can see and change.** On a managed cluster, "how do I tune the scheduler?" has the answer "you mostly cannot" — the provider exposes a subset of configuration, and the rest is hidden. On a self-managed cluster, you own every parameter and every failure.
- **Upgrades.** On managed clusters, the control plane is patched for you and you choose when to roll worker nodes. On self-managed clusters, you upgrade the control plane, etcd, node OS, and kubelet yourself, in the order dictated by the version skew policy ([Part 26.2](#262-cluster-upgrades)).
- **Defaults.** As established in 2.2, whether workloads land on the control plane, whether a CNI and default StorageClass are already installed, and whether a load-balancer implementation exists all vary. **A guide that says "just apply this manifest" is often silently assuming a specific distribution.**
- **The presence of a cloud controller.** This is the single most common source of "it works in the docs but not here" in networking and storage: `type: LoadBalancer` Services and dynamically provisioned volumes require a controller that knows how to talk to a cloud API, and a bare-metal cluster has none until one is installed.

## 2.12 The reconciliation model, concretely

With the vocabulary established, a single Deployment can be traced through every layer. This is the mental model the rest of the guide assumes.

```
An operator or CI system applies a Deployment manifest
  │
  ├─ kube-apiserver: authenticates the caller → authorizes (RBAC) → runs admission
  │   checks (Pod Security, policy, webhooks) → validates → persists to etcd
  │
  ├─ deployment-controller (in kube-controller-manager): desired 3 replicas,
  │   observed 0 → creates a ReplicaSet object
  │
  ├─ replicaset-controller: desired 3 pods, observed 0 → creates 3 Pod objects
  │   (the pods have no node yet)
  │
  ├─ kube-scheduler: for each pending pod → filter nodes (requests, taints,
  │   affinity, volumes) → score the survivors → write spec.nodeName
  │   (requests only — limits play no part in placement)
  │
  ├─ kubelet (on the chosen node): sees a pod assigned to it → pulls the image
  │                          → creates the pod sandbox (a pause container that
  │                            holds the network namespace and IP)
  │                          → runs init containers in order
  │                          → starts the main containers
  │                          → runs readiness/liveness probes, reports status
  │
  ├─ endpointslice-controller: the pod matches a Service selector → its IP is
  │   listed in the Service's EndpointSlice, carrying a `ready` condition flag.
  │   Readiness does not add the endpoint; it sets that flag, and consumers
  │   (kube-proxy, the CNI datapath, reverse proxies) route only to endpoints
  │   marked ready and serving.
  │
  └─ HPA controller: metrics show CPU above target → patches the Deployment's
      replica count → the loop begins again
```

Every arrow is an independent loop reading and writing shared state. Controllers coordinate **only** through that shared state — controller-to-controller, there are no direct calls, and that is the property that makes the architecture extensible.

It is worth being precise, though, because the sweeping version of this claim ("nothing ever calls anything") is false and will mislead you during an incident. There are real synchronous dependencies in the system:

- The **API server makes synchronous HTTP calls to admission webhooks**. This is why a `ValidatingWebhookConfiguration` with `failurePolicy: Fail` whose backend is down blocks creates cluster-wide — a hard, direct dependency on an external service, and a genuine single point of failure ([Part 5.4](#54-extending-the-api-crds-controllers-operators-webhooks)).
- The **kubelet calls out** to the CRI runtime, the CNI plugin, and the CSI driver on its node.
- **Controllers call cloud APIs** to create load balancers, attach disks, and manage DNS.

The safe formulation: **controller-to-controller coordination is purely through shared state; components still have direct, often synchronous dependencies on plugins, webhooks, and external APIs.** When something is broken, the productive question is "at which arrow did the chain stop?" — that question, asked systematically, resolves the large majority of Kubernetes problems, and it is the method [Part 20](#part-20--debugging) formalizes.

---

# Part 3 — How the Cluster Itself Is Built and Managed

Eventually every cluster must be rebuilt: a region fails, a version reaches end of life, a provider deprecates its platform, or a new environment is needed. This part explains how clusters are created and how to make that reproducible. The reason is blunt: **a cluster that cannot be rebuilt from code is a cluster that cannot be recovered**, and the difference between a multi-day outage and a 40-minute restore is usually whether this work was done in advance.

Readers who already have a cluster and are not responsible for provisioning it can skim this part, but should return to 3.3 — the checklist of what must live in code applies to anyone operating a cluster, not only to whoever created it.

The layers, in order, are these:

## 3.1 The layers of the stack

```
Cloud accounts / projects / VPC / subnets / IAM          ← Terraform, Pulumi, Crossplane
        │
Cluster (control plane, node pools, CNI, add-ons)        ← Terraform + Helm, Cluster API, Crossplane
        │
Platform add-ons (cert-manager, gateway, Argo CD, ...)   ← Argo CD (app-of-apps)
        │
Namespaces, quotas, policies, RBAC                       ← Argo CD
        │
Applications                                             ← Argo CD
```

Each layer has a tool that owns it. The important rule: **pick one owner per layer and don't fight yourself.** The anti-pattern is Terraform managing Kubernetes objects *and* Argo CD managing them too — they will overwrite each other forever.

## 3.2 Tools for provisioning the cluster

| Tool | Language | Best for | Notes |
|---|---|---|---|
| **Terraform / OpenTofu** | HCL | The default. Cloud infra + managed clusters (EKS/AKS/GKE modules) + Helm releases. | Massive ecosystem. State management is the hard part (use remote state + locking). |
| **Pulumi** | TypeScript/Python/Go | Teams that want real programming languages and testing. | Great DX; smaller ecosystem than Terraform. |
| **Crossplane** | Kubernetes CRDs | Teams that want *everything* through the Kubernetes API — cluster and cloud resources reconciled like any other object, with no separate state file. | Powerful "control plane of control planes". Steeper learning curve; a bad Composition can delete cloud resources. |
| **Cluster API (CAPI)** | Kubernetes CRDs | Declarative *self-managed* cluster lifecycle: create, upgrade, scale clusters as Kubernetes objects. **v1.12 (Jan 2026) added in-place updates and chained upgrades.** | The standard for fleets of self-managed clusters. Uses a "management cluster" to control workload clusters. |
| **kubeadm / Talos / k3s** | CLI/config | The actual node-level bootstrap underneath CAPI or by hand. | Talos is machine-config-declarative by design. |
| **Managed console / CLI** | — | Managed clusters, if you're not automating yet. | Fine for one cluster; untenable for five. |

**A sane, boring recommendation for most teams:** Terraform (or OpenTofu) for cloud infra and the managed cluster, **Argo CD for everything inside the cluster**, and Cluster API only if you must run self-managed clusters at scale. Crossplane if your organization has already decided Kubernetes is the universal control plane.

## 3.3 What "reproducible cluster" means concretely

For your cluster to be reproducible, these must all be in code:

1. **Cloud infra** — VPC, subnets, IAM roles, the cluster, node pools. (Terraform)
2. **Cluster bootstrap** — CNI, storage class defaults, the Gateway implementation, cert-manager, external-secrets, metrics-server, Argo CD itself. (A `bootstrap` directory, applied once, then self-managing.)
3. **Everything else** — Argo CD's app-of-apps points at git, and the cluster converges on its own (Part 21).
4. **Secrets** — references to an external secret manager, never values (Part 8).
5. **Data** — the one thing that is *not* reproducible from code. Databases live outside the cluster (or are backed up to object storage), and their restore procedure is documented and *tested* (Part 27).

**Test it.** Once a year, build a new cluster from scratch in a scratch account and restore a real environment into it. Whatever you forgot to codify will become obvious in the first 20 minutes. This drill is the difference between "we have infrastructure as code" and "we can actually recover."

## 3.4 Day-2 reality

- **Node images** need patching monthly (OS CVEs), independently of Kubernetes minor upgrades.
- **Add-on versions** need tracking. Keep a table (Part 26.5) of every add-on, its version, and its supported Kubernetes range. This table is your platform's inventory and your upgrade plan.
- **Terraform drift** is real: someone changes a security group in the console at 2am and the next plan wants to revert it. Run `terraform plan` on a schedule and alert on unexpected diffs.
- **Cluster-level changes need the same review as application changes.** A change to the CNI or a GatewayClass affects every tenant.

---

# Part 4 — Accessing the Cluster

## 4.1 The kubeconfig

Everything goes through the API server, and the API server is authenticated. Your `kubeconfig` is a YAML file (default `~/.kube/config`, or `$KUBECONFIG`) describing:

- **clusters** — API server URL + the CA certificate to verify it.
- **users** — *how you authenticate* (client cert, token, exec plugin, OIDC).
- **contexts** — a named pairing of (cluster, user, namespace).
- **current-context** — which one is active.

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: prod
    cluster:
      server: https://api.prod.example.com:6443
      certificate-authority-data: <base64 CA>
users:
  - name: alice
    user:
      exec:                      # exec plugins = short-lived creds (OIDC, cloud IAM)
        apiVersion: client.authentication.k8s.io/v1
        command: aws
        args: ["eks", "get-token", "--cluster-name", "prod"]
contexts:
  - name: prod-admin
    context: { cluster: prod, user: alice, namespace: default }
current-context: prod-admin
```

**Rules to live by:**

- **Make the current context visible in your shell prompt.** A green/red prompt segment showing `kube-context` and `kube-namespace` prevents the classic disaster of running a destructive command against production because you forgot to switch. Tools: `kube-ps1`, `kubectx`/`kubens`, `starship`.
- **Never commit a kubeconfig to git.** It is a credential.
- **Never hand out the cluster-admin kubeconfig.** It is root on the entire cluster.
- Use `kubectx` and `kubens` (or `kubectl config use-context`) constantly.
- Set `KUBECONFIG=/path/a:/path/b` to merge files — useful for keeping a read-only prod config and a write dev config separate.

```bash
kubectl config get-contexts          # list
kubectl config use-context dev       # switch
kubectl config set-context --current --namespace=mystuff   # default ns for this context
kubectl cluster-info
kubectl version                       # client + server, spot version skew
```

## 4.2 How humans should authenticate: OIDC, not client certs

The default kubeadm cluster gives you client-certificate auth (an `admin.conf` with an embedded cert). That is fine for bootstrap and **bad practice for humans**, but the reason is narrower than it is often stated. Certificates *can* carry group membership — the `O` (organization) fields in the subject are mapped to RBAC groups, and the `CN` becomes the username. The real defects are that **Kubernetes has no certificate revocation mechanism**, so a certificate cannot be withdrawn before it expires, and that certificates are typically long-lived. Together those mean that a departing employee's credential remains valid until expiry, and that you cannot respond to a leak by revoking access. That is what centralized identity fixes.

**Modern pattern — OIDC / identity-provider integration:**

1. Run an IdP (Okta, Entra ID, Keycloak, Google Workspace, Auth0).
2. Configure the apiserver with `--oidc-issuer-url`, `--oidc-client-id`, `--oidc-username-claim`, `--oidc-groups-claim`.
3. Users run `kubectl` with an **exec credential plugin** (e.g. `kubelogin`, `aws eks get-token`, `gcloud`) that performs the browser/device login and hands kubectl a short-lived ID token.
4. You bind **RBAC** to the groups the IdP asserts (`oidc:platform-admins`, `oidc:developers`).
5. Deprovisioning a person = disabling them in the IdP. Their access dies with the token, typically within an hour.

This is the single highest-leverage access-control decision you will make, and it should be made in week one, not year two. Retrofitting identity onto 200 static kubeconfigs is miserable.

**Security note on exec plugins:** `kubectl` will run whatever binary your kubeconfig names, which makes a malicious or tampered kubeconfig a code-execution vector. `kubectl` gained a user-preferences file (`kuberc`) in 1.33 (alpha) and 1.34 (beta), and 1.35 added the **credential-plugin allowlist** so you can restrict which binaries a kubeconfig is permitted to execute. Enable it once you are past bootstrap.

**Service accounts** are the machine counterpart: a workload needs to call the API? Give it a ServiceAccount, and on modern clusters bind it to a cloud IAM role via **IRSA** (EKS), **Workload Identity** (GKE/AKS) — so the pod gets cloud credentials without any long-lived key ever existing. This is how pods should read S3 buckets or call cloud APIs.

## 4.3 Authorization: RBAC

RBAC is allow-only; there are no deny rules. If no rule allows an action, it is denied. Four objects:

- **Role** — permissions, namespaced.
- **ClusterRole** — permissions, cluster-scoped (or reusable across namespaces).
- **RoleBinding** — grants a Role/ClusterRole *within a namespace*.
- **ClusterRoleBinding** — grants a ClusterRole *cluster-wide*.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: team-a, name: deployer }
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { namespace: team-a, name: team-a-deployers }
subjects:
  - kind: Group
    name: oidc:team-a-developers      # bind to IdP groups, not individuals
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**RBAC hygiene that matters:**

- **Bind to groups, never to users.** Individuals leave; groups are managed in the IdP.
- **Never** create a binding to `cluster-admin` for a human long-term. If you must, make it time-boxed and audited.
- **Watch for privilege escalation via `create pods`.** Anyone who can create a pod can usually mount any ServiceAccount in that namespace, mount hostPath volumes, or run a privileged container — i.e. they can become root on the node. `create pods` is effectively node-admin in a namespace that lacks Pod Security enforcement.
- **Beware `escalate`/`bind`/`impersonate` verbs** and `secrets` read access — reading a Secret often yields cloud credentials.
- Audit it: `kubectl auth can-i --list`, `kubectl auth can-i create pods --as=system:serviceaccount:ns:sa`, and tools like **rbac-police**, **kubescape**, or **krane**.

## 4.4 Other ways in

- **`kubectl port-forward`** — a tunnel from your machine through the apiserver to a pod port. Excellent for debugging a service you haven't exposed. Not for production traffic. `kubectl port-forward svc/foo 8080:80`.
  - **Security caveat:** port-forward bypasses your Gateway, WAF, auth proxy, and rate limits entirely, and it's often allowed by RBAC to developers who shouldn't reach production databases. Audit who has `pods/portforward` in prod namespaces. Upstream guidance (notably the "Securing Production Debugging in Kubernetes" post, published March 2026) pushes toward time-boxed, audited, policy-gated debugging access rather than blanket port-forward rights.
- **ServiceAccount tokens are mounted into every pod by default.** Turn that off unless a pod genuinely needs API access: [Part 45.2](#452-serviceaccount-tokens-stop-using-long-lived-credentials).
- **`kubectl exec -it pod -- sh`** — a shell in a container. Great for debugging, terrible as a management strategy (and often *impossible* if you run **distroless** images — minimal images containing only the application and its runtime libraries, with no shell, package manager, or debugging tools, which is precisely what makes them harder to attack).
- **`kubectl debug`** — attach a *separate* ephemeral container with debugging tools to a running pod, or copy a pod with a modified spec. This is how you debug distroless containers:
  ```bash
  kubectl debug -it pod/api-xyz --image=nicolaka/netshoot --target=api
  kubectl debug node/worker-1 -it --image=busybox   # debug a whole node
  ```
- **API via proxy / client libraries** — controllers and CI systems talk to the API directly. Use `client-go` (Go), `kubernetes` (Python), `@kubernetes/client-node` (Node).
- **Dashboards** — **Headlamp** (the **CNCF** — Cloud Native Computing Foundation — dashboard, and where the ecosystem has consolidated — plugins exist for Karpenter, Knative, Kubeflow, Cluster API, Volcano), **k9s** (terminal UI, highly recommended), **Lens/OpenLens**, **Rancher**. Read convenience only; they need their own auth story (OIDC + RBAC, not a static token) and must never be your only access path.
- **Direct API access** — `kubectl proxy` exposes the API on localhost with your credentials; `kubectl get --raw /metrics`.

---

# Part 5 — The API and the Resource Model

## 5.1 Objects, kinds, and the shape of everything

Most Kubernetes objects declare a `spec` and report a `status`, and the split between them is the central convention of the API. It is a convention rather than a universal law — objects whose state cannot diverge from what you asked for may have only `spec`, and some rename it (a ConfigMap and a Secret carry `data`; an Event has neither). The four top-level fields you will see on nearly every object are:

```yaml
apiVersion: apps/v1        # which API group+version
kind: Deployment           # what type of object
metadata:                  # identity: name, namespace, labels, annotations, uid
  name: api
  namespace: prod
  labels: { app: api, tier: backend }
  annotations: { owner: team-a }
spec:                      # DESIRED state — yours to write
  replicas: 3
status:                    # OBSERVED state — written by controllers, never by you
  readyReplicas: 3
```

**The spec/status split is the whole game.** You write `spec`. Controllers write `status`. Reconciliation is the process of dragging `status` toward `spec`. If you find yourself editing `status`, you are doing something wrong.

One nuance worth knowing: `status` is not *exclusively* written by controllers. The **kubelet** reports status for Nodes and Pods — which is why a pod's phase, conditions, and container states come from the node it runs on — and writing to the `/status` subresource from a controller you have written is a supported pattern rather than a violation. The rule of thumb is "do not hand-edit status," not "only controllers ever touch it.

**API groups** — the API is versioned and grouped:
- Core (`v1`): Pod, Service, ConfigMap, Secret, Namespace, Node, PersistentVolumeClaim.
- `apps/v1`: Deployment, StatefulSet, DaemonSet, ReplicaSet.
- `batch/v1`: Job, CronJob.
- `networking.k8s.io/v1`: NetworkPolicy, Ingress.
- `gateway.networking.k8s.io/v1`: GatewayClass, Gateway, HTTPRoute, GRPCRoute, TCPRoute, UDPRoute.
- `rbac.authorization.k8s.io/v1`: Role, RoleBinding, ClusterRole, ClusterRoleBinding.
- `apiextensions.k8s.io/v1`: CustomResourceDefinition.
- Plus everything installed by operators.

Version suffixes are meaningful: `v1alpha1` (may change or vanish — never use in prod), `v1beta1` (broadly stable, may change), `v1` (stable, GA). When you upgrade clusters, **deprecated API removal is the #1 cause of "my apply broke"** — the `v1beta1` you used gets deleted and your manifests 404. Check with `kubectl api-resources`, `kubectl explain`, and tools like **pluto** or **kubent**.

**Key commands to learn the API without leaving the terminal:**

```bash
kubectl api-resources                       # every kind: short name, API group, namespaced?
kubectl api-versions
kubectl explain deployment.spec.strategy     # built-in docs for ANY field, including CRDs
kubectl explain pod.spec.containers.resources --recursive
kubectl get deployment api -o yaml           # the full truth about an object
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP'
```

## 5.2 Labels, selectors, and annotations

**Labels** are key/value pairs used for *selection*. They are the connective tissue of Kubernetes: Services find pods by label, Deployments own ReplicaSets by label, NetworkPolicies apply by label, `kubectl get -l` filters by label, Prometheus discovers targets by label, Argo CD tracks resources by label.

```yaml
labels:
  app.kubernetes.io/name: api
  app.kubernetes.io/instance: api-prod
  app.kubernetes.io/version: "1.14.2"
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: shop
  app.kubernetes.io/managed-by: argocd
```

Use the [recommended common labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) — they are what dashboards, service meshes, and tooling expect. **Set them consistently from day one**; retrofitting labels across 300 manifests is awful, and label inconsistency makes NetworkPolicy and metrics silently useless.

**Selectors** come in two flavors:
- Equality-based: `-l app=api,env=prod`, `-l 'env!=dev'`.
- Set-based: `-l 'env in (prod,staging)'`, `-l 'tier'` (exists), `-l '!canary'`.

**Annotations** are also key/value but are *not* selectable. They hold non-identifying metadata for tools: `cert-manager.io/cluster-issuer`, `argocd.argoproj.io/sync-wave`, `prometheus.io/scrape`, cost-allocation tags, owner info, change-ticket links. Rule of thumb: **if you need to select or group by it, it's a label; if it's metadata for a tool or a human, it's an annotation.**

## 5.3 Owned objects, finalizers, and garbage collection

Objects reference each other:
- **ownerReferences** create parent/child links, and they are what drives **garbage collection**. Delete a Deployment → the delete cascades to its ReplicaSets → to their Pods. This is why `kubectl delete deployment api` actually removes everything.

  Keep this distinct from **selection**, which is a different mechanism: a label selector is how a Deployment *finds and adopts* matching ReplicaSets, and how a Service finds pods. Ownership (what dies with what) is `ownerReferences`; selection (what matches what) is labels. The two usually agree, but conflating them makes cascading deletion and adoption behaviour hard to predict.
- **Finalizers** are strings in `metadata.finalizers` that *block deletion* until a controller removes them. They exist so external resources get cleaned up (delete a `LoadBalancer` Service → the cloud LB must be destroyed first). **A stuck `Terminating` object almost always means a finalizer whose controller is gone.** Fix:
  ```bash
  kubectl get ns stuck -o json | jq '.spec.finalizers'
  # Namespace finalizers live in spec.finalizers — patching metadata.finalizers does nothing:
  kubectl replace --raw "/api/v1/namespaces/stuck/finalize" -f finalize.json   # last resort
  # For ordinary objects (not namespaces) the field really is metadata.finalizers:
  kubectl patch svc stuck -p '{"metadata":{"finalizers":null}}' --type=merge
  ```
  Note the asymmetry: a **namespace's** deletion-blocking finalizers are in `spec.finalizers`, while every other object uses `metadata.finalizers`. Patching the wrong one appears to succeed and changes nothing, which is why "I patched it and it is still Terminating" is such a common dead end. Also note that force-finalizing a namespace **deletes its contents without waiting** for their own cleanup — only do it when you accept that the underlying external resources will leak (you will have to delete the cloud load balancer by hand).

## 5.4 Extending the API: CRDs, controllers, operators, webhooks

**This is why Kubernetes won.** You can add your own types to the API and write a controller that reconciles them. The entire ecosystem — Gateway API, cert-manager, Argo CD, every database operator — is built this way.

- **CRD (CustomResourceDefinition)** — declares a new kind. Its schema is OpenAPI-validated by the apiserver, so `kubectl explain` and `kubectl apply --dry-run=server` work on it automatically.
  ```yaml
  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata: { name: widgets.example.com }
  spec:
    group: example.com
    scope: Namespaced
    names: { plural: widgets, singular: widget, kind: Widget, shortNames: [wdg] }
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
                  size: { type: integer, minimum: 1, maximum: 10 }
              status:
                type: object
                properties:
                  ready: { type: boolean }
        subresources: { status: {} }      # separate /status endpoint
        additionalPrinterColumns:
          - { name: Size, type: integer, jsonPath: .spec.size }
  ```
- **Controller** — a loop with a watch on your kind. Standard libraries: `controller-runtime` (Go, the default), `kopf` (Python), `kubernetes-client` (Java/JS).
- **Operator** — a controller that encodes *operational knowledge* for an application: install, upgrade, failover, backup, restore. "Operator" is a pattern, not a technology.
- **Admission webhooks** — HTTP callbacks the apiserver invokes on create/update:
  - **Mutating** — change the object (inject sidecars, add defaults, add labels).
  - **Validating** — allow or deny.
  - **The critical operational risk: `failurePolicy: Fail`.** If a webhook is `Fail` and its backing service is down, **no matching object can be created anywhere in the cluster** — including the pods that would restore the webhook. This is a real, common, total-outage mode. Mitigations: `failurePolicy: Ignore` for non-critical webhooks, tight `timeoutSeconds` (2–5s, never 30), `objectSelector`/`namespaceSelector` to narrow scope, never match your own namespace, and run ≥2 replicas behind a PDB.
  ```yaml
  apiVersion: admissionregistration.k8s.io/v1
  kind: ValidatingWebhookConfiguration
  metadata: { name: policy.example.com }
  webhooks:
    - name: policy.example.com
      admissionReviewVersions: ["v1"]
      sideEffects: None
      failurePolicy: Ignore
      timeoutSeconds: 3
      namespaceSelector:
        matchLabels: { policy.example.com/enforce: "true" }
      rules:
        - apiGroups: [""]
          apiVersions: ["v1"]
          operations: ["CREATE", "UPDATE"]
          resources: ["pods"]
      clientConfig:
        service: { name: policy-webhook, namespace: policy-system, path: /validate }
        caBundle: <base64 CA>
  ```
- **Where policy belongs now.** For common cases, prefer **`ValidatingAdmissionPolicy`** (CEL — Common Expression Language — based, **built into Kubernetes**, no webhook to keep alive) over a self-hosted webhook. Kubernetes 1.36 added **manifest-based admission control** so these policies themselves can't be casually deleted, and **declarative validation** graduated to GA in 1.36. Use Kyverno/Gatekeeper when you need their richer feature sets (mutation, generation, external data), and webhooks only when nothing else fits.

---

# Part 6 — Workloads

**Vocabulary introduced in this part:** Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob, probe (liveness, readiness, startup), init container, native sidecar, resource request and limit, QoS class, priority class, PodDisruptionBudget, scheduling constraint. The core objects are defined here; the scheduling and resource-allocation rules were introduced in [Part 2.6–2.10](#26-taints-and-tolerations-how-a-node-refuses-work) and are applied in [6.6](#66-resources-qos-and-scheduling).

## 6.1 Pods

A **pod** is the smallest deployable unit in Kubernetes: one or more containers that are always placed on the same node, share a network namespace (and therefore an IP address and a port space), and can share storage volumes. [Part 1.5](#15-how-it-differs-from-plain-docker--compose) explains why the pod — rather than the container — is the unit.

Pods are almost never created directly. Instead an operator declares a **controller** — an object whose job is to create and maintain pods — and the controller creates them. The reason is the reconciliation model from [Part 1.4](#14-the-orchestration-problem-stated-precisely): a bare pod that dies is simply gone, because nothing owns it and nothing will recreate it. A pod created by a controller is continuously maintained. The four workload controllers are a Deployment (6.2), a StatefulSet (6.3), a DaemonSet (6.4), and a Job or CronJob (6.5).

A pod spec contains:

- `containers[]`: `image`, `command`/`args`, `ports`, `env`, `envFrom`, `resources`, `volumeMounts`, `securityContext`, `livenessProbe`/`readinessProbe`/`startupProbe`, `lifecycle`.
- `initContainers[]`: run to completion, in order, before app containers. Use for migrations, waiting on dependencies, fetching config, `chown`ing volumes.
- `volumes[]`: emptyDir, configMap, secret, projected, persistentVolumeClaim, hostPath (avoid), CSI.
- `restartPolicy`: `Always` (default), `OnFailure`, `Never`.
- `serviceAccountName`, `nodeSelector`, `affinity`, `tolerations`, `topologySpreadConstraints`.
- `securityContext` (pod-level) and per-container `securityContext`.

**Init containers vs sidecars.** Init containers run *before* the app and must exit. **Native sidecar containers** (a `restartPolicy: Always` init container; beta and enabled by default since 1.29, **stable since 1.33**) run *alongside* the app, start before it, and are shut down *after* it. This is the correct way to run a proxy or log forwarder in-pod, and it replaced most hand-rolled sidecar patterns. There's a subtlety worth knowing: because native sidecars are init containers, they start *before* your app container — which is usually what you want (the proxy is listening before the app tries to reach it), but it changes startup ordering assumptions from the old "app first, sidecar whenever" behavior.

Kubernetes 1.34 also added **per-container restart policy** (restart just one container in a pod) and the ability to **define app environment variables from an init container**, and 1.35 added **in-place pod restart** and **in-place pod resize** as Stable — meaning you can grow a container's resources without recreating the pod.

## 6.2 Deployment — stateless apps

A **Deployment** is the standard controller for a stateless application. It manages a **ReplicaSet**, which is a simpler controller whose only job is to keep a specified number of identical pods running. The extra layer exists so that updates are possible: to deploy a new version, the Deployment creates a *new* ReplicaSet for the new pod template, scales it up, and scales the old one down. Keeping the old ReplicaSet around (subject to `revisionHistoryLimit`) is what makes `kubectl rollout undo` possible — it is not re-running your CI pipeline in reverse, it is scaling the previous ReplicaSet back up.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: { app.kubernetes.io/name: api }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app.kubernetes.io/name: api }   # IMMUTABLE after creation
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # how many extra pods above replicas during update
      maxUnavailable: 0        # never dip below desired capacity
  minReadySeconds: 10          # pod must stay ready 10s before counting as available
  progressDeadlineSeconds: 600
  template:
    metadata:
      labels: { app.kubernetes.io/name: api }
    spec:
      serviceAccountName: api
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app.kubernetes.io/name: api }
      containers:
        - name: api
          image: registry.example.com/api:1.14.2     # NEVER :latest
          imagePullPolicy: IfNotPresent
          ports: [{ name: http, containerPort: 8080 }]
          envFrom:
            - configMapRef: { name: api-config }
            - secretRef: { name: api-secrets }
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits:   { memory: 512Mi }              # CPU limit usually omitted
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /ready, port: http }
            periodSeconds: 5
          startupProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 5
            failureThreshold: 60          # up to 5 min to start, before liveness kicks in
          lifecycle:
            preStop:
              exec: { command: ["sleep", "5"] }   # let endpoints drain before SIGTERM
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
      volumes:
        - name: tmp
          emptyDir: {}
      terminationGracePeriodSeconds: 30
```

**The probes are not decoration.** Getting them right is the difference between a self-healing service and an outage:

- **Startup probe** — asks "has the application finished starting?" While it is failing, the liveness and readiness probes are **disabled**, so a slow boot cannot be mistaken for a hang. Once it succeeds it stops running and the other two take over. Use it for anything with a long or unpredictable warm-up (a JVM loading classes, a service building caches). The startup budget is `periodSeconds × failureThreshold` — the example in [Part 6.2](#62-deployment--stateless-apps) allows 60 × 5s = 5 minutes — and note that when the startup probe *does* exhaust its budget, the container is killed under the restart policy, so this is a real boundary rather than a diagnostic. It exists because the alternative, a liveness `initialDelaySeconds`, is a guess: too short kills the app during boot, too long delays detecting a genuine hang later.

- **Readiness probe** — asks "should this pod receive traffic right now?" Failing it does **not** restart anything. What actually happens is more specific than "the pod is removed from endpoints," and the detail matters when you debug it:

  1. The kubelet runs the probe and updates the pod's `Ready` **condition**.
  2. The pod's entry in its Service's **EndpointSlice** carries `ready`, `serving`, and `terminating` condition flags. **Membership in the EndpointSlice comes from matching the Service's label selector, not from readiness** — a not-ready pod is still listed, with its flags set false.
  3. Consumers — `kube-proxy` (or the CNI's eBPF datapath), and any reverse proxy or load balancer watching EndpointSlices — read those flags and **route only to endpoints marked ready and serving**.

  So the mechanism is *flag-based filtering at the consumer*, not removal from a list. That is why a readiness change takes effect at slightly different moments in different consumers, and why endpoint propagation is asynchronous ([Part 39.2](#392-graceful-termination-in-the-correct-order)). Use readiness for "I am alive but cannot serve properly right now" — a saturated connection pool, a dependency outage, a warm-up phase. The pod keeps running and rejoins the rotation automatically when the probe passes again.

- **Liveness probe** — asks "is this process still functioning at all?" Failing it makes the kubelet **kill the container**, which then restarts under the pod's restart policy. This is the only one of the three that destroys work in progress, which is why it is the one most often misconfigured. Use it narrowly, for states a restart genuinely fixes: a deadlock, a wedged event loop, an unrecoverable internal error. **A liveness probe that tests an external dependency is a self-inflicted outage**: the database blips, every replica fails liveness at the same moment, every replica restarts together, and the resulting thundering herd is worse than the original blip. Point liveness at the process itself; never wire it to a dependency. If you cannot name the wedged state a liveness probe is meant to catch, you probably do not need one — a readiness probe is the safer default.

**Rollout mechanics:**

```bash
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api --to-revision=3
kubectl set image deployment/api api=registry.example.com/api:1.14.3
kubectl rollout restart deployment/api        # careful: this is an imperative deploy
```

**Graceful shutdown ordering** matters, and the sequence is usually taught wrong in a way that hides why deploys drop requests. What actually happens, with two independent tracks running at once:

**Track 1 — the object's deletion.** (1) The API server records a `deletionTimestamp` and starts the grace clock. (2) The kubelet begins local shutdown: it runs `preStop`, (3) then sends `SIGTERM` to PID 1, (4) waits out whatever remains of `terminationGracePeriodSeconds`, (5) then `SIGKILL`. Note that **the grace clock starts at deletion, not at `SIGTERM`** — `preStop` consumes the same budget, which is why a long `preStop` plus a long shutdown can exceed the grace period and end in `SIGKILL` mid-request.

**Track 2 — the endpoint.** Separately and *concurrently*, the control plane decides whether to stop routing traffic here. It does not "remove the pod from Endpoints": the endpoint **stays in the EndpointSlice**, with its `terminating` condition set and `ready: false`. Consumers then stop selecting it — on their own schedule.

**There is no ordering guarantee between the two tracks**, and that is the entire problem. The pod can be shutting down, or already gone, while a load balancer or proxy still holds it in rotation. Your app must (a) handle SIGTERM as "stop accepting, finish in-flight, exit", and (b) have a `preStop` sleep or a readiness that flips to false, so the **load balancer and reverse proxy** have time to notice. Without this you get 502s on every deploy — the classic "we deploy and lose 0.3% of requests" bug. Note that the proxy layer's endpoint propagation is usually the slowest part, which is why the `preStop` sleep is measured in seconds, not milliseconds.

## 6.3 StatefulSet

For workloads needing **stable identity**: stable pod names (`db-0`, `db-1`, `db-2`), stable per-pod DNS (`db-0.db.ns.svc.cluster.local`), stable per-pod storage, and ordered start/stop/scale operations.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db }
spec:
  serviceName: db-headless        # a headless Service (clusterIP: None) for identity
  replicas: 3
  podManagementPolicy: OrderedReady   # or Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate: { partition: 0 }    # partition>0 = canary: only ordinals >= partition update
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
        - name: db
          image: postgres:17
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumeClaimTemplates:                # one PVC PER POD, stable across reschedules
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources: { requests: { storage: 100Gi } }
```

What StatefulSet does **not** do: initialize a cluster, promote a replica, reconfigure membership, back up, or restore. That is operator territory (Part 17).

Gotchas: **deleting a StatefulSet does not delete its PVCs** (deliberate — your data survives; but it also means orphaned volumes accumulate and cost money). Scaling down does not delete PVCs either.

## 6.4 DaemonSet

Runs one pod per node (or per matching node). For node-level agents: log shippers, CNI, CSI node plugins, node exporters, security agents, GPU device plugins.

```yaml
spec:
  selector: { matchLabels: { app: node-exporter } }
  updateStrategy: { type: RollingUpdate, rollingUpdate: { maxUnavailable: 1 } }
  template:
    spec:
      tolerations:                      # DaemonSets usually need to run everywhere
        - { operator: Exists, effect: NoSchedule }
      containers: [...]
```

## 6.5 Job and CronJob

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: migrate-1-14-2 }
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400       # auto-delete finished jobs (keep the cluster clean!)
  template:
    spec:
      restartPolicy: Never             # or OnFailure
      containers:
        - { name: migrate, image: registry.example.com/api:1.14.2, args: ["migrate"] }
---
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-report }
spec:
  schedule: "0 2 * * *"                # UTC! set timeZone: "Europe/Berlin" to be explicit
  timeZone: "Europe/Berlin"
  concurrencyPolicy: Forbid            # don't overlap runs
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate: { spec: { template: { spec: { ... } } } }
```

CronJobs are evaluated by the **controller-manager's clock**, and in an HA control plane only the leader runs them — so a CronJob will not double-fire. But `concurrencyPolicy: Forbid` plus a job that runs longer than its interval means silent skips. Always set `ttlSecondsAfterFinished` and history limits, or you'll find 40,000 finished Job objects slowing down every `kubectl get all`.

**Newer Job features worth knowing:** `podReplacementPolicy` (GA in 1.34) controls whether failed pods are replaced immediately or only after the Job terminates; `managedBy` (GA in 1.35) lets an external controller (like a queue-based autoscaler) own Job lifecycle; and **mutable pod resources for suspended Jobs** went beta in 1.36 so you can right-size a suspended Job's resources before resuming it.

For migration jobs in a GitOps world, use **Argo CD sync hooks/waves** so migrations run *before* the new Deployment rolls (`argocd.argoproj.io/hook: PreSync`, `sync-wave`). Never rely on a human remembering to run the migration.

## 6.6 Resources, QoS, and scheduling

This section is the practical counterpart to [Part 2.10](#210-node-pressure-qos-and-eviction), which explains *why* nodes evict pods and defines the QoS classes. Advanced placement — spreading across failure domains, correcting drift over time, and reserving capacity — is covered in [Part 37](#part-37--advanced-scheduling); the storage resource that causes the most node evictions is covered in [Part 38](#part-38--ephemeral-storage-node-pressure-and-resource-exhaustion). Here is how to declare resources correctly and how to influence where pods land.

**Requests vs limits** — the two numbers every container should have, and the distinction that causes the most confusion:

- A **request** is the amount the scheduler *reserves* for the container. It is a promise: the node must have this much free (as measured against **allocatable** capacity, which is the machine's total minus what the operating system and Kubernetes system daemons need) or the pod will not be placed there. Requests also set the container's relative CPU weight when the node is contended, and they are what autoscalers target.
- A **limit** is the ceiling the container may not exceed at runtime. Exceeding a CPU limit produces **throttling** — the container is slowed down, not killed. Exceeding a memory limit produces an **OOM kill** (the kernel's out-of-memory killer terminates the process), and the container restarts according to its restart policy. That asymmetry — CPU is compressible, memory is not — explains the guidance below.

Guidance that holds in most cases:

- **Always set requests.** A container with no requests cannot be scheduled predictably, gets the lowest CPU share under contention, is the first evicted, and cannot be autoscaled meaningfully.
- **Always set a memory limit.** Without one, a leaking or misbehaving container can consume the node's memory and trigger evictions of *unrelated* pods. A memory limit converts a node-wide incident into a single-container restart.
- **Set CPU limits deliberately, often by omitting them.** A CPU limit converts contention into throttling, and throttling frequently shows up as latency spikes that are hard to diagnose. Many production setups set CPU requests without CPU limits and rely on requests for fair sharing, while setting memory requests equal to memory limits for predictability.
- **Size for the steady state, not the peak.** Requests based on peak usage waste capacity and can make pods unschedulable; requests based on nothing at all cause evictions. Use observed usage (see [Part 25.2](#252-vpa) on the Vertical Pod Autoscaler, which produces recommendations) rather than intuition.

**QoS classes** — Kubernetes derives a pod's **QoS (Quality of Service) class** automatically from its requests and limits. It is not a setting to choose; it is a consequence of what is declared, and it determines eviction order when a node runs short of resources ([Part 2.10](#210-node-pressure-qos-and-eviction) explains the reasoning):

| Class | Assigned when | CPU weight under contention | Eviction under node pressure |
|---|---|---|---|
| `Guaranteed` | Every container sets requests **equal to** limits, for both CPU and memory | Proportional to its **request** — like every other pod | Evicted last |
| `Burstable` | Any container declares a CPU or memory **request or limit**, and requests do not equal limits | Proportional to its **request** | Middle, *unless* its usage is within its request — then it groups with `Guaranteed` |
| `BestEffort` | **No** container declares any CPU or memory request **or limit** | Minimum share — starved rather than throttled | Evicted first |

**Two corrections to how this is usually explained.** First, **the kernel's CPU weight comes from the container's CPU *request*, not from its QoS class.** A `Guaranteed` pod requesting 100m gets a *smaller* share than a `Burstable` pod requesting 500m. QoS class affects `oom_score_adj` (the OOM killer's preference), not CPU scheduling. Second, the common claim that Guaranteed pods are "least likely to be throttled" is backwards: because a Guaranteed pod has request *equal to* limit, it is throttled at that ceiling, while a Burstable pod with a CPU request and **no** CPU limit is never throttled at all. BestEffort pods are not "throttled first" either — with no requests they receive the minimum share and are simply starved when the node is busy.

**On eviction order, be careful:** the kubelet does **not** rank by QoS class. Its documented order is (1) whether the pod's usage exceeds its request, (2) pod priority, (3) how far usage exceeds the request. The practical consequence is the useful part: **a `Burstable` pod whose usage stays inside its request is evicted in the same group as a `Guaranteed` pod**, not "in the middle." QoS class is a rough predictor at best, and it does not apply at all to inode or PID starvation, or to `ephemeral-storage` requests.

One practical note, which generalises beyond any single class: the kubelet prefers to evict pods whose *actual usage exceeds their request*, and **that rule is independent of QoS class** — it applies to every pod. So staying within your declared request is what protects you, not merely declaring one. And `priorityClassName` can override this ordering entirely (see below). The class is visible on any running pod:

```bash
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```

**Scheduling controls** — the mechanisms for influencing which node a pod lands on. The first four are *attraction* (pull toward a node); taints are *repulsion* (the node pushing pods away, [Part 2.6](#26-taints-and-tolerations-how-a-node-refuses-work)):

- `nodeSelector` — the simplest form: a set of labels the node must have, e.g. `{ disktype: ssd }`. Hard requirement, no scoring.
- `nodeAffinity` — the expressive version, with `requiredDuringSchedulingIgnoredDuringExecution` (a hard requirement: "only nodes in these zones") and `preferredDuringSchedulingIgnoredDuringExecution` (a soft preference: "prefer those nodes, but use others if necessary"), plus operators like `In`, `NotIn`, `Exists`, `Gt`, `Lt`. The phrase "IgnoredDuringExecution" means the rule is evaluated at scheduling time only, so a pod is not evicted if its node's labels later change.
- `podAffinity` / `podAntiAffinity` — express placement *relative to other pods* rather than to node labels: "run me on the same node as a pod labeled `app=cache`," or "never run me on the same node as another replica of myself." The `topologyKey` selects the domain (for example `kubernetes.io/hostname` for per-node, or `topology.kubernetes.io/zone` for per-zone). Anti-affinity with `topologyKey: kubernetes.io/hostname` is the standard way to guarantee that replicas of a service are on different machines.
- `topologySpreadConstraints` — the modern replacement for most anti-affinity use cases. Instead of "never co-locate," it says "distribute evenly across these domains, allowing at most this much imbalance (`maxSkew`)." This is the correct tool for spreading replicas across availability zones, and it degrades more gracefully than hard anti-affinity when the cluster cannot satisfy the ideal distribution.
- **taints + tolerations** — nodes repel pods; pods opt in. Used for dedicated hardware (GPU pools, database nodes) and for isolating workloads from each other. Remember from Part 2.6 that a toleration is permission, not attraction: dedicated nodes need both a taint on the node and a toleration *plus* an affinity or selector on the pod.
- `priorityClassName` — assigns the pod a **priority** (see [Part 37.5](#375-priority-preemption-and-capacity-reservation)). Priority has two effects: higher-priority pods are scheduled first when resources are scarce, and **preemption** allows a high-priority pending pod to cause lower-priority *running* pods to be evicted so it can be placed. Define tiers deliberately and early, and apply them consistently, for example: `platform-critical` (1000000), `platform` (100000), `app` (10000), `batch` (1000). A cluster where everything has the default priority cannot protect anything.
- `podDisruptionBudget` — protects availability during *voluntary* disruptions (node drain, upgrades):
  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata: { name: api }
  spec:
    minAvailable: 2          # or maxUnavailable: 1
    selector: { matchLabels: { app.kubernetes.io/name: api } }
  ```
  **A PDB can block node drains forever.** If `minAvailable` equals your replica count, `kubectl drain` hangs. This is intentional — it is telling you your workload can't tolerate any disruption.
- **In-place pod resize** (Stable in 1.35) — change CPU/memory without a restart. Combined with VPA this finally makes right-sizing non-disruptive.
- **Workload-aware scheduling** (introduced 1.35, advanced in 1.36/1.37) — Kubernetes is gaining the ability to schedule groups of related pods (gang scheduling) atomically, which matters enormously for distributed training and batch jobs where "all 8 workers or none" is the requirement.

---

# Part 7 — Templating and Packaging: Helm, Kustomize, Jsonnet

You will not hand-write 40 near-identical Deployment manifests. You need a templating strategy, and this is a decision you make in week one because migrating later is painful.

## 7.1 The three approaches

| Tool | Model | Strengths | Weaknesses |
|---|---|---|---|
| **Helm** | Go templates + values files; packages are *charts*; releases tracked in-cluster | The package manager of Kubernetes. Huge ecosystem (Artifact Hub). Versioned, installable/upgradeable third-party software. | Go templating is awkward, whitespace-sensitive, and hard to debug. Values files become sprawling. Release state in Secrets can drift. |
| **Kustomize** | Overlay/patch model; no templating, pure YAML | Reached via `kubectl apply -k` or `kubectl kustomize` (there is no bare `kubectl -k`). You can always read the input and the output. Great for "same app, N environments". | Not a package manager. Overlay chains get confusing at depth. Patching arbitrary fields can be fiddly. |
| **Jsonnet / CUE / dhall** | Real programming languages for config | Maximum expressive power, real abstraction and testing | A language to learn; smaller talent pool; tooling less standard |

**The pragmatic industry answer:** use **Helm for third-party software** (cert-manager, Argo CD, Cilium, kube-prometheus-stack — you are not going to rewrite those), and **Kustomize for your own applications** (or Helm if your team already knows it). Many teams use both: Helm charts rendered by Argo CD's Helm support, with Kustomize for the per-environment overlay on top.

## 7.2 Helm, concretely

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgresql
helm show values bitnami/postgresql > my-values.yaml     # ALWAYS read the values file
helm template my-pg bitnami/postgresql -f my-values.yaml  # render locally, inspect
helm upgrade --install my-pg bitnami/postgresql \
  -n data --create-namespace -f my-values.yaml --version 16.x.x --atomic --wait
helm list -A
helm history my-pg -n data
helm rollback my-pg 2 -n data
helm uninstall my-pg -n data
```

**Helm practice that prevents pain:**
- **Pin chart versions.** `--version` always. an unpinned `helm upgrade` after a `helm repo update` silently moving you from chart 1.2 to 2.0 is a classic incident.
- **`helm template` before applying.** Render and read the YAML. Never blindly `upgrade --install` something you haven't seen.
- **`--atomic --wait`** so a failed upgrade rolls back instead of leaving you half-deployed.
- **Don't `helm upgrade` third-party charts by hand in a GitOps world** — declare them as Argo CD `Application`s (Part 21) so git remains the source of truth.
- **Helm release secrets are cluster state**, not git state. If someone `helm upgrade`s manually, git and the cluster diverge and Argo CD will fight them. Pick one owner.
- **Values files should be per-environment and in git**, never `--set` strings in a CI script (invisible, unreviewable, and easy to typo).

**Do you need to write your own Helm chart?** Only if you're distributing software. For your own apps, a Kustomize base + overlays is usually simpler and more legible.

## 7.3 Kustomize, concretely

```
apps/api/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── httproute.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    └── prod/
        ├── kustomization.yaml
        ├── patch-replicas.yaml
        └── patch-resources.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: [deployment.yaml, service.yaml, httproute.yaml]
labels:
  - pairs: { app.kubernetes.io/name: api }
    includeSelectors: true      # `commonLabels` is deprecated in Kustomize v5
---
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: api-prod
resources: [../../base]
images:
  - name: registry.example.com/api
    newTag: abc1234                     # ← CI/GitOps bump this line to deploy
patches:
  - path: patch-replicas.yaml
  - target: { kind: Deployment, name: api }
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: "1"
configMapGenerator:
  - name: api-config
    literals: ["LOG_LEVEL=info", "ENV=production"]
```

**The `images:` field is the deployment mechanism** for most GitOps setups. Promotion = changing one tag in one file, in a PR. That's the whole "cd" step, and it's reviewable.

Kustomize also has `configMapGenerator`/`secretGenerator` (with automatic hash suffixes so config changes actually trigger rollouts — solving the "ConfigMap changed but pods didn't restart" problem), `replacements` for cross-field wiring, and `components` for reusable optional chunks.

## 7.4 Which one, and the trap

**The trap is mixing three tools and ending up with YAML nobody can trace.** Pick Helm for third-party, Kustomize for yours, and be consistent. If you find yourself writing a Helm template that generates a Kustomize overlay that is consumed by a Helm chart, stop.

Two more honest notes:
- **`kubectl apply -f` on a directory of raw YAML** is fine for learning and for tiny setups, and completely unsatisfactory past about five services per environment.
- **Never store rendered output in git alongside the source.** Render in CI, or let Argo CD render (it supports Helm and Kustomize natively). Two copies of the truth is one copy too many.

---

# Part 8 — Configuration and Secrets

## 8.1 ConfigMaps

Non-secret configuration. Two consumption patterns:

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: api-config }
data:
  LOG_LEVEL: info
  config.yaml: |
    feature_flags:
      new_checkout: true
```

- **As env vars** (`envFrom.configMapRef`) — simple, but **changes do not propagate to running pods.** You must restart the deployment. This trips people constantly.
- **As a mounted volume** — the file in the container updates within ~60s (kubelet sync period) when the ConfigMap changes. This is the right choice for config files you want hot-reloadable, and it's why apps should watch their config file rather than reading it once at boot.

**Use `immutable: true`** for ConfigMaps/Secrets that shouldn't change — it prevents accidental modification and improves apiserver performance. Version your ConfigMaps by name (`api-config-v14`) and have the Deployment reference the version, so a config change is an explicit rolling update that GitOps can track (Part 21). This is far better than mutating a ConfigMap in place and hoping pods restart. (Kustomize's `configMapGenerator` hash suffix automates exactly this.)

## 8.2 Secrets, and why the default is not secure

```bash
kubectl create secret generic api-secrets --from-literal=DB_PASSWORD=hunter2
kubectl get secret api-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

That `base64 -d` is the point: **Kubernetes Secrets are base64-encoded, not encrypted.** Out of the box:

- They're stored in etcd **in plaintext** unless you enable **encryption at rest** (`EncryptionConfiguration` with `aescbc`/`aesgcm`/KMS provider — a real KMS is strongly preferred over a local key).
- Anyone with `get secret` in a namespace can read them. Anyone who can create a pod in a namespace can mount any Secret in it. Anyone with node access can read the kubelet's copy.
- They're visible in `kubectl get secret -o yaml`, in CI logs if you're careless, and in Helm release state.

**Etcd encryption at rest is table stakes:**

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets", "configmaps"]
    providers:
      - kms: { name: kms-provider, endpoint: unix:///var/run/kmsplugin/socket.sock, cachesize: 1000 }
      - identity: {}          # fallback for reading old unencrypted data
```

**A 1.37-era note:** **Cluster Trust Bundles** and **Pod Certificates** landed as a way to give workloads real, API-managed certificates for identity — reducing the pressure to distribute long-lived secrets for service-to-service auth. It's young, but it's where workload identity is heading (and it pairs with SPIFFE-style identity from a mesh).

## 8.3 The modern answer: External Secrets Operator + a real secret manager

Storing secrets as encrypted blobs **in git** (Sealed Secrets, SOPS) was the previous generation and still works, but the modern pattern is: **the secret lives in a dedicated secret manager; Kubernetes syncs a reference to it.**

- **External Secrets Operator (ESO)** — `SecretStore`/`ClusterSecretStore` + `ExternalSecret` CRDs pull from AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, 1Password, Doppler.
- **Secrets Store CSI Driver** — mounts secrets as volumes from the same backends without creating a Secret object at all (better: nothing to read out of etcd).
- **Vault + injector / Vault Secrets Operator** — dynamic, short-lived database credentials. The strongest option, and the most operational work.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: { name: api-secrets, namespace: prod }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secrets-manager, kind: ClusterSecretStore }
  target: { name: api-secrets, creationPolicy: Owner }
  data:
    - secretKey: DB_PASSWORD
      remoteRef: { key: prod/api/db, property: password }
```

**Why this is better:** rotation happens in one place (the secret manager) and propagates; there is no encrypted blob to leak from git; access is controlled by cloud IAM with audit logs; and *no human ever pastes a production credential into a YAML file.*

**Other rules:**
- **Never** put secrets in a ConfigMap, a container image, a Helm values file, or a CI log.
- **Never** `--from-literal` in a script that lands in shell history for prod.
- Prefer **workload identity** (IRSA/Workload Identity) over any static credential — then there is no secret to manage at all for cloud API access.
- **Rotate.** A secret that has never been rotated is a secret that has leaked; you just don't know yet.
- Scan for leaks: `gitleaks`, `trufflehog` in CI, and in git history.

---

# Part 9 — Networking Fundamentals, From Zero

## 9.1 The problems to solve

Kubernetes mandates a specific network model and delegates the implementation to a **CNI plugin**:

1. **Pod-to-pod** — every pod gets a unique, routable IP; any pod can reach any pod *without NAT*. This is a flat network, and it's the model, not a suggestion.
2. **Pod-to-service** — stable virtual IPs (ClusterIPs) in front of ephemeral pods.
3. **External-to-service** — **north-south** traffic, meaning traffic entering or leaving the cluster from outside it. This is where load balancers, reverse proxies, and the Gateway API live (Parts 10–12). The opposite direction — service-to-service traffic *inside* the cluster — is called **east-west**.
4. **Service-to-external** — egress, including NAT and egress policy.

Plus the ones people forget: **DNS**, **NetworkPolicy**, **service mesh**, and **cross-cluster/multi-cluster**.

## 9.2 The CNI layer

On an empty cluster, this is your first real decision. The CNI provides pod IPs and connectivity.

| CNI | Best at | Notes |
|---|---|---|
| **Cilium** (eBPF) | Modern default for self-managed. Policy based on workload identity, kube-proxy replacement, Hubble observability, Gateway API, and transparent encryption (WireGuard or IPsec). | The strongest all-round modern choice. Removes kube-proxy. Multi-pool IPAM went stable in 1.19. |
| **Calico** | Policy depth, mature, BGP for on-prem, Windows support. | Also a solid default; strong NetworkPolicy implementation. |
| **Flannel** | Simplicity | No NetworkPolicy enforcement. Avoid for anything serious. |
| **Cloud CNIs** (AWS VPC CNI, Azure CNI, GKE Dataplane V2) | Native cloud integration, IAM-aware, security groups per pod | Managed clusters come with one; you generally keep it. AWS VPC CNI has the IP-exhaustion footgun (Part 20.4). |
| **Antrea / OVN-Kubernetes** | VMware/OpenShift ecosystems | Fine, but ecosystem-coupled. |

**Installing Cilium on a fresh self-managed cluster (Helm):**

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update cilium

helm install cilium cilium/cilium --version 1.20.1 \
  --namespace kube-system \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443 \
  --set kubeProxyReplacement=true \
  --set routingMode=tunnel \
  --set ipam.mode=kubernetes \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2}"
```

`kubeProxyReplacement=true` means Cilium programs service load-balancing itself via eBPF — you then remove kube-proxy (or never install it). This is a meaningful performance and observability win, and it's also the first place to look when Service routing behaves oddly (Part 20.3). **Ordering matters:** the CNI must exist before nodes are `Ready` and before any pod with a dependency on networking can schedule. On kubeadm: install the CNI *before* joining worker nodes.

**Verify networking is real before building anything on top:**

```bash
kubectl get pods -n kube-system -o wide            # all CNI pods Running
kubectl run nettest --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl exec -it nettest -- ping -c2 <another-pod-ip>
kubectl exec -it nettest -- dig kubernetes.default.svc.cluster.local
kubectl exec -it nettest -- nslookup google.com     # egress + DNS test
```

## 9.3 DNS: CoreDNS

CoreDNS is a Deployment in `kube-system` (usually 2 replicas) that serves cluster DNS. Every pod's `/etc/resolv.conf` points at the **kube-dns ClusterIP**.

Resolution patterns you must memorize, because half of all "why can't it connect" incidents are DNS:

```
<service>.<namespace>.svc.cluster.local      → ClusterIP of the Service
<service>.<namespace>                        → same, from another namespace
<service>                                    → same namespace only
<pod-ip-dashes>.<namespace>.pod.cluster.local → individual pod (rare)
<statefulset-pod>.<headless-svc>.<ns>.svc.cluster.local → stable pod identity
```

**Search domains** are what make short names work — most resolv.conf has `search <ns>.svc.cluster.local svc.cluster.local cluster.local`. The classic trap: **`ndots:5`**. A name with fewer than 5 dots (e.g. `api.stripe.com` has 2) gets the search domains appended *first*, so it is tried as `api.stripe.com.prod.svc.cluster.local` and times out before the real lookup. This causes real latency and puzzling failures. Fixes: use fully qualified domain names (FQDNs) with a trailing dot (`api.stripe.com.`), or tune `dnsConfig` per pod:

```yaml
spec:
  dnsConfig:
    options:
      - { name: ndots, value: "2" }
```

**CoreDNS scale/health:** `kubectl -n kube-system get cm coredns -o yaml`, `kubectl -n kube-system logs deploy/coredns`. Enable **NodeLocal DNSCache** on busy clusters to cut latency and take load off CoreDNS.

## 9.4 Services

A Service is a stable virtual IP + DNS name in front of a dynamic set of pods, selected by labels.

| Type | What it gives you | Use when |
|---|---|---|
| `ClusterIP` | Internal-only virtual IP | Default. Internal service-to-service. |
| `NodePort` | Opens a port (30000-32767) on *every* node | Legacy/dev. Ugly in prod. Sometimes a building block for an external LB. |
| `LoadBalancer` | Cloud controller provisions an L4 LB | Cheap L4 exposure, or when a Gateway implementation needs one. |
| `ExternalName` | CNAME to an external DNS name | Aliasing an external dependency. |
| **Headless** (`clusterIP: None`) | No VIP; DNS returns all pod IPs | StatefulSets, clients that need to talk to specific pods, and **client-side load balancing**. |

```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  type: ClusterIP
  selector: { app.kubernetes.io/name: api }   # must match pod labels exactly
  ports:
    - { name: http, port: 80, targetPort: http }   # targetPort can be a NAME
  # sessionAffinity: ClientIP    # rarely what you want; usually a smell
```

**How it works** — and this differs enough between datapaths that one sentence cannot cover both:

- **kube-proxy in iptables or IPVS mode** writes per-packet rules that **DNAT** the ClusterIP to a chosen pod IP, and records the translation in **conntrack**. That conntrack entry is what pins a whole connection to one backend, and it is also what determines source-IP behaviour and how `externalTrafficPolicy` works — so it is the thing to inspect when connections behave oddly.
- **Cilium's eBPF datapath** instead resolves the Service at **socket level, at `connect()` time**. For in-cluster traffic the packet is sent straight to the chosen backend with no netfilter hop and no conntrack entry (and DSR mode avoids rewriting the return path entirely).

Either way the observable contract is the same to a client: the ClusterIP and DNS name are stable, and the endpoints behind them are not. **Endpoints** / **EndpointSlices** are the actual list of (pod IP, port) that are *ready*. This is why the readiness probe is load-bearing for traffic:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=api
kubectl get endpoints api            # empty endpoints = selector mismatch or pods not Ready
```

**Minimal-diff note:** a *named* `targetPort` (`targetPort: http`) resolves against a named `containerPort`, so it survives renumbering — but only if a `containerPort` with that name exists. A numeric `targetPort` needs no matching `containerPort` declaration at all (that field is informational); it must simply be a port the process actually listens on. Named ports are more maintainable.

**`externalIPs` is deprecated.** Kubernetes 1.36 deprecated `Service.spec.externalIPs`, and upstream's advice is that all users migrate away from it — toward an external load-balancer controller or a Gateway API implementation. It is *deprecated, not removed*, so manifests using it continue to work for now. Two reasons to migrate anyway: the field cannot support dual-stack, and address allocation was never managed by Kubernetes (the cluster administrator owns those IPs).

**Traffic distribution / topology:** by default, Service traffic can cross zones (costly and slower). `spec.trafficDistribution` lets you express a *preference* for topologically closer endpoints. The current values are **`PreferSameZone`** (prefer endpoints in the client's zone) and **`PreferSameNode`** (prefer endpoints on the client's node) — note that **`PreferClose` is now deprecated as an older, less precise alias for `PreferSameZone`**, so use the newer name in anything you write today. These are preferences, not guarantees: if no local endpoint is healthy, traffic crosses zones rather than failing. Use them on chatty internal services to cut cross-AZ latency and egress bills.

## 9.5 NetworkPolicy

By default in Kubernetes, **all pods can reach all pods**. Every CNI that implements flat networking gives you zero isolation out of the box. NetworkPolicy is the fix — and it is *default-deny-friendly, additive-only*:

- **Isolation is per direction, and "additive" describes how allowances combine.** Two separate ideas are usually compressed into one sentence here, so let us take them in turn. (1) A policy applies only to the directions listed in its `policyTypes` (or inferred from the rules it contains) — so a pod selected by an **Ingress-only** policy still has completely **unrestricted egress**. Allowing one direction does not restrict the other; you need a policy for each. (2) Within a direction, allowances are a **union**: if several policies select the same pod, traffic is permitted if *any* of them allows it. There are no deny rules and no precedence, which is why the order policies appear in never matters.
- Policies are **namespaced** and select pods by label.
- You need **ingress and egress** rules — allowing one doesn't allow the other. Egress to DNS must be explicitly allowed or your pods can't resolve anything.
- **Your CNI must enforce it.** Flannel does not. Verify with a real test.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: prod }
spec:
  podSelector: {}                        # all pods in this namespace
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-allow, namespace: prod }
spec:
  podSelector: { matchLabels: { app.kubernetes.io/name: api } }
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: prod } }
          podSelector: { matchLabels: { app.kubernetes.io/name: web } }
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: gateway-system } }
      ports: [{ protocol: TCP, port: 8080 }]
  egress:
    - to:                                              # DNS
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
    - to:                                              # database
        - ipBlock: { cidr: 10.20.0.0/24 }
      ports: [{ protocol: TCP, port: 5432 }]
    - to:                                              # external HTTPS
        - ipBlock: { cidr: 0.0.0.0/0, except: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16] }
      ports: [{ protocol: TCP, port: 443 }]
```

**Realistic adoption path:** start with default-deny *ingress* per app namespace, then tighten egress — egress default-deny breaks things loudly and needs observability to do safely. Cilium can run in **policy audit mode** to log what *would* be blocked before you enforce, which is the correct way to roll this out. Cilium 1.19 added **NetworkPolicy enhancements** (richer selectors, better policy semantics) worth reading if you are on it.

**Before you plan your address space:** pod and Service CIDR ranges cannot be changed later, and dual-stack adds a second family to plan. See [Part 42.1](#421-cidr-planning-the-decision-you-cannot-easily-reverse).

**The egress gap:** NetworkPolicy controls L3/L4 by pod identity. It does not inspect HTTP, does not do FQDN allowlisting by default (Cilium's `toFQDNs` and Calico's DNS policy do), and it's enforced at the CNI, not at the app. For "this service may only call api.stripe.com over HTTPS," you want an **egress gateway / forward proxy** — the same reverse-proxy technology, pointed outbound. That's Part 11.9.

**A crucial interaction with reverse proxies:** your proxy/gateway pods must be *allowed* to reach backend pods, and backend pods must allow ingress from the gateway namespace. Teams routinely write default-deny policies that silently break their own ingress, then spend an afternoon debugging the wrong layer. Add the gateway namespace to your allow rules **before** you enforce default-deny.

---

# Part 10 — Load Balancing: The Four Layers

Before talking about reverse proxies, you need the layering straight, because "load balancing in Kubernetes" means at least four different things.

## 10.1 L4 vs L7: the distinction that explains everything

- **Layer 4 (L4) load balancing** operates on TCP/UDP. It sees *connections*, not requests. It picks a backend when the connection is established and then forwards bytes. It cannot read HTTP headers, cannot route by path or host, cannot retry a failed request, and cannot tell a `200` from a `500`.
- **Layer 7 (L7) load balancing** terminates the connection, parses the protocol (HTTP/1.1, HTTP/2, gRPC), and makes routing decisions *per request*. It can route by host, path, header, query, or method; retry a failed request; enforce timeouts; rewrite headers; do auth; and emit per-request metrics.

**The consequence that surprises everyone:** a Kubernetes `Service` is **L4**. kube-proxy (or Cilium's eBPF datapath) load-balances **per connection**, not per request. If your HTTP client uses keep-alive connections (it does), then **one client connection is pinned to one pod for its lifetime.** Ten clients hammering a Service with persistent connections = ten pods handling everything, and the rest idle. This is not a Kubernetes bug; it's what L4 means. The fixes are:

1. **Put an L7 proxy in front** (a reverse proxy — Part 11) that load-balances per request.
2. **Use short-lived connections or client-side load balancing** (gRPC's recommended approach: resolve a headless Service and round-robin across pod IPs yourself, e.g. with `grpc-go`'s `round_robin` resolver).
3. **Use `trafficDistribution`/session affinity carefully.**

Most production north-south traffic goes through an L7 proxy precisely for this reason.

## 10.2 The four layers of load balancing in a cluster

```
④ Global / DNS load balancing          ← Route53/Cloudflare latency or geo routing, GSLB, multi-region
        │
③ External L4 (cloud LB / VIP)         ← AWS NLB, GCP TCP LB, MetalLB, kube-vip  (Service type=LoadBalancer)
        │
② L7 proxy / gateway                   ← Envoy, NGINX, HAProxy, Traefik, Gateway API data plane  (Parts 11–12)
        │
① Internal L4 (Service VIP)            ← kube-proxy (iptables/IPVS) or Cilium eBPF; EndpointSlices
```

Every layer exists for a reason, and each has its own failure modes:

| Layer | What it balances | Health model | Where it goes wrong |
|---|---|---|---|
| ① Service VIP | Connections across ready pod IPs | Readiness probes → EndpointSlices | Empty endpoints, stale endpoint propagation, per-connection pinning |
| ② L7 proxy | Requests across backends | Active health checks + endpoint watching | Timeouts, retries storms, header/body buffering, stale config |
| ③ External LB | Connections across nodes | Node health / target group health | Cross-zone charges, target group misregistration, health check path wrong |
| ④ DNS / global server load balancing (GSLB) | Users across regions/clusters | Health-checked DNS records | TTL caching, failover lag, DNS-based failover is *slow* (minutes) |

**Design principle:** each layer should be *stateless* and *independent*. The cloud LB does not know your pods; the L7 proxy does. Keep health checking correct at each layer or you get "the LB thinks it's fine but users see 503s."

## 10.3 Internal L4: how a Service actually balances

Two implementations:

- **kube-proxy in iptables mode** (historic default): DNAT rules per Service/endpoint. O(n) rule churn as endpoints change; fine for small clusters, painful at thousands of Services. Uses **random** selection per connection.
- **kube-proxy in IPVS mode**: hash-table based, better scaling, more algorithms available (rr, lc, sh, mh).
- **eBPF (Cilium, kube-proxy replacement)**: service lookup in eBPF maps at the socket layer. Fastest, and it avoids the DNAT and conntrack overhead of the iptables path; also enables per-request visibility (Hubble) and socket-level load balancing. Be precise about what socket LB skips: it bypasses the **Service-VIP translation hop** by choosing the backend inside the socket at `connect()` time, so no iptables or conntrack entry is created — but the packets still traverse the network stack and the pod network to reach the chosen backend. It does not make pod-to-pod traffic bypass the stack, and plain pod-to-pod traffic that never targets a Service VIP is not involved at all.

**Algorithms available at L4 depend entirely on which datapath you are running**, and the differences are large enough to invalidate advice you may have read elsewhere:

- **kube-proxy in iptables mode** offers no choice at all — it selects randomly per connection.
- **kube-proxy in IPVS mode** exposes scheduler algorithms: round robin, least connection, source hashing, and maglev.
- **Cilium (eBPF)** defaults to `random`, and offers Maglev **only for external (north-south) traffic**. This is the counter-intuitive part: internal pod-to-Service connections are resolved by **socket-level load balancing at `connect()` time**, which assigns a backend without hashing. So "use consistent hashing so a given key always reaches the same pod" **does not work for in-cluster Service calls on Cilium** — the connection is pinned to whichever endpoint was chosen when the socket connected.

**Session affinity (`sessionAffinity: ClientIP`)** exists but is usually a smell. If you need it, ask *why*: usually the real answer is a shared session store, or sticky routing implemented at the L7 layer where you can use cookies instead of IPs (IPs break behind NAT and mobile networks).

## 10.4 L7 algorithms — what a real proxy offers

Once you're at L7, you have choices L4 can't express:

| Algorithm | Behavior | Use when |
|---|---|---|
| **Round robin** | Even rotation | Uniform, fast backends |
| **Weighted round robin** | Proportional | Canary, heterogeneous capacity |
| **Least request** | Fewest outstanding requests (often power-of-two-choices) | Variable request cost — the best default for HTTP |
| **Random** | Uniform random | Large backend sets, avoids herding |
| **Ring hash / Maglev (consistent hashing)** | Same key → same backend | Cache locality, session affinity, sharded state |
| **Latency-aware (least-request with slow-start)** | Avoids cold backends | New pods warming up |

**Slow start** deserves a mention: when a new pod joins, sending it a full share of traffic immediately is how you get a latency spike on every deploy (the pod is warming JITs, connection pools, caches). Envoy supports this natively, as does NGINX **Plus** (open-source NGINX does not have `slow_start`). This one setting removes a whole class of "deploys are slow for 30 seconds" complaints.

---

# Part 11 — Reverse Proxies, In Full

**This is the part the first draft of this guide was missing.** A reverse proxy is not an optional add-on in Kubernetes — it is the component that turns a flat pod network into a usable service topology. There are reverse proxies at *every* layer of a Kubernetes cluster, and if you don't understand what they are and where they sit, half of the rest of this document is magic.

## 11.1 What a reverse proxy is

A **proxy** is an intermediary that accepts a connection and makes another connection on someone's behalf. The direction of the "someone" defines the two kinds:

- **Forward proxy** — sits in front of *clients*. The client is configured to send its requests to the proxy, which fetches them from the internet on the client's behalf. Used for egress control, caching, and anonymity. In Kubernetes: your **egress gateway** (11.9).
- **Reverse proxy** — sits in front of *servers*. Clients think they are talking to the real server; the proxy accepts the request, decides which backend should handle it, forwards it, and returns the response. Clients never know how many backends exist or where they are. Used for ingress, routing, TLS termination, load balancing, caching, auth, and rate limiting. In Kubernetes: your **Gateway**, **ingress controller**, and **service mesh sidecar**.

The name "reverse" is historical — it's a proxy that runs *backwards* from the client's perspective.

### Why you need one at all

Five reasons, and each is independently sufficient:

1. **Decoupling.** Clients address a stable name; backends come and go constantly. Pods die and are reborn with new IPs dozens of times a day. Without a proxy, every client would have to track that. This is the *primary* reason Kubernetes needs proxies.
2. **Load balancing at the right layer.** As established in 10.1, an L4 Service pins connections; only an L7 proxy balances *requests*.
3. **Cross-cutting concerns in one place.** TLS termination, auth, rate limiting, CORS, compression, request/response header manipulation, access logging, and metrics. Implemented once at the proxy instead of in 40 microservices in 6 languages.
4. **Protocol translation and normalization.** HTTP/1.1 ↔ HTTP/2 ↔ gRPC, WebSocket upgrades, SSE, gzip/br, header casing rules.
5. **Protection.** Timeouts, retries with budgets, circuit breaking, connection limits, body size limits, and WAF rules that stop a misbehaving client or a bad deploy from taking down backends.

### What a reverse proxy actually does to a request

Take `GET https://api.example.com/v1/orders/42` hitting an Envoy-based gateway:

1. **Accept** the TCP connection (often from a cloud L4 LB in front — that's layer ③ passing to layer ②).
2. **TLS terminate** using the certificate whose SAN matches SNI `api.example.com`. (Or **TLS passthrough** — forward encrypted bytes to the backend, never seeing plaintext.)
3. **Parse** the HTTP request: method, path, headers, body (subject to size limits and buffering policy).
4. **Normalize** — HTTP/2 header names lowercased, hop-by-hop headers stripped, `X-Forwarded-For` / `X-Forwarded-Proto` / `X-Request-Id` added or rewritten.
5. **Match** against route rules: host `api.example.com`, path prefix `/v1`, maybe a header or method condition. First (most specific) match wins.
6. **Apply route policy** — timeouts, retry policy and budget, rate limit, auth (JWT/API key/OAuth), CORS, URL rewrite, request/response header mutation, request mirroring.
7. **Resolve the backend** — evaluate the route's weighted cluster list, pick an endpoint using the configured LB algorithm (respecting health status and locality preferences).
8. **Connect** to the backend pod IP (reusing a pooled connection if possible — connection pooling to the backend is a major performance feature).
9. **Forward** the request, applying the outbound timeout and, if configured, re-encrypting with mTLS to the backend (`BackendTLSPolicy` / `DestinationRule`).
10. **Observe the response.** Code 5xx? Retry per policy (idempotent methods only, within a retry budget, to a *different* endpoint). Too slow? Enforce the timeout and return 504.
11. **Transform the response** — strip internal headers, add security headers, compress, maybe cache.
12. **Log and emit metrics** — one access log line with latency, status, upstream host, and trace ID; counters and histograms per route/backend/status.
13. **Return** to the client, keeping the client connection alive.

**Every one of the 13 steps is a place where something can go wrong, and every one is configurable.** That's why "the reverse proxy is the most common source of confusing production behavior" is a true statement — a 502, a 504, a truncated response, a lost header, a doubled request, or a stale backend all come from a specific step here.

## 11.2 The taxonomy: where proxies live in a Kubernetes cluster

```
                    Internet / users
                          │
                ┌─────────▼──────────┐
                │  Cloud L4 LB / VIP │  ← Service type=LoadBalancer / MetalLB   (L4)
                └─────────┬──────────┘
                          │
        ┌─────────────────▼──────────────────┐
        │  EDGE / GATEWAY reverse proxy      │  ← Gateway API data plane:
        │  TLS, routing, WAF, rate limit     │    Envoy, NGINX, Traefik, HAProxy,
        └─────────┬──────────────────────────┘    cloud LB controller, Cilium
                  │  (north-south, L7)
   ┌──────────────┼───────────────┬────────────────────┐
   │              │               │                    │
┌──▼───┐     ┌────▼───┐      ┌────▼───┐          ┌─────▼─────┐
│ pod  │     │  pod   │      │  pod   │          │ MESH      │ ← ztunnel/waypoint
│ app  │◄───►│  app   │◄────►│  app   │          │ SIDECARS  │   (east-west, mTLS)
└──┬───┘     └────────┘      └────────┘          └───────────┘
   │
   │  ┌────────────────┐        ┌──────────────────┐
   └─►│ IN-POD sidecar │        │ EGRESS gateway   │──► external APIs / DB
      │ proxy (local)  │        │ (forward proxy)  │
      └────────────────┘        └──────────────────┘
```

**Six distinct places a reverse proxy appears:**

| # | Position | Purpose | Typical software |
|---|---|---|---|
| 1 | **Edge / north-south** | Public entry, TLS, routing, WAF, rate limits | Envoy (Envoy Gateway, Istio ingress), NGINX, Traefik, HAProxy, Cilium, cloud LB controllers |
| 2 | **Internal north-south** | Same, for internal clients only | Same software, separate Gateway, internal IP |
| 3 | **Southbound to backends** | Re-encrypt, apply per-backend policy | Proxy's upstream TLS config, `BackendTLSPolicy` |
| 4 | **In-pod sidecar** | Local proxy for one pod: auth, protocol translation, egress interception | Envoy sidecar, linkerd2-proxy, NGINX sidecar, oauth2-proxy |
| 5 | **Node-level mesh proxy** | mTLS + L4 policy without per-pod proxies | Istio ambient **ztunnel**, Cilium (eBPF, no proxy at all) |
| 6 | **Egress / forward proxy** | Control and audit outbound traffic, FQDN allowlists | Envoy egress gateway, Squid, Cilium FQDN policy, cloud NAT gateway |

**The key realization:** when people say "Kubernetes networking is complicated," a lot of what they mean is "there are six proxies in the path and I don't know which one is dropping my header."

## 11.3 The proxy implementations you can actually run

### Envoy

The dominant modern data plane. A C++ L7 proxy originally built at Lyft, now CNCF-graduated, and the engine under **Istio, Envoy Gateway, Ambassador/Emissary, Contour, Gloo**, and many cloud LBs (note that **AWS App Mesh reaches end of support on 2026-09-30**, so it is not a choice for new work). Configured via **xDS** — a dynamic API by which a control plane pushes listeners, routes, clusters, and endpoints without restarts. That dynamic-config design is why it fits Kubernetes so well: the control plane watches EndpointSlices and pushes updates continuously.

- **Strengths:** HTTP/2 and gRPC first-class, excellent observability (stats, tracing, access logs), rich routing and resilience features, extremely dynamic.
- **Weaknesses:** raw Envoy config is huge and unfriendly; you essentially always use it through a control plane (Istio, Envoy Gateway, Contour) rather than writing YAML by hand. Debugging means learning `config_dump`, `clusters`, and `stats` admin endpoints.

### NGINX and NGINX Gateway Fabric

The most widely deployed web server/reverse proxy in existence. Battle-tested, excellent static file serving and caching, extremely well-understood by ops teams, and enormous documentation.

- **NGINX Ingress Controller** (the Kubernetes ingress controller) — note that **the community `ingress-nginx` project retired in March 2026**, but there are other NGINX-based controllers (F5/NGINX Inc.'s NGINX Ingress Controller, and others) still maintained.
- **NGINX Gateway Fabric** — the Gateway API implementation from NGINX Inc. Use this if you want NGINX semantics under the Gateway API model.
- **Strengths:** familiar, predictable, great docs, strong caching/static serving, lots of extension modules.
- **Weaknesses:** the open-source version's dynamic reconfiguration historically means reloads (a reload storm under rapid endpoint churn is a real scaling issue); the Gateway API implementation is newer than Envoy's; annotation-driven Ingress config is not portable (which is the whole problem Gateway API solves).

### HAProxy

The original high-performance L4/L7 load balancer. Exceptional connection handling, extremely mature, superb at raw TCP and at very high connection counts.

- **HAProxy Ingress** / **HAProxy Kubernetes Ingress Controller** / Gateway API support in newer versions.
- **Strengths:** performance, reliability, TLS, fine-grained L4 control, the `runtime API` for dynamic changes.
- **Weaknesses:** config-file-centric; the Kubernetes integration story is less cohesive than Envoy's; smaller cloud-native ecosystem.

### Traefik

Go-based, Kubernetes-native by design, with automatic service discovery, Let's Encrypt integration built in, and a strong middleware concept (rate limit, auth, headers, circuit breaker as composable middlewares).

- **Strengths:** fastest path from zero to working ingress with automatic TLS; excellent CRDs (IngressRoute/Middleware) and Gateway API support; great for small-to-mid clusters.
- **Weaknesses:** historically less suited to very high-throughput, deeply customized L7 scenarios than Envoy; middleware model is Traefik-specific (portability caveat).

### Cilium (no proxy at all)

Cilium's distinctive bet: do it in **eBPF in the kernel** instead of in a userspace proxy. Service load balancing, network policy, and (increasingly) L7 policy are enforced in the datapath, and Cilium's Gateway API implementation programs Envoy only where L7 features genuinely require it.

- **Strengths:** no per-pod proxy overhead, best-in-class observability (Hubble), identity-based policy, kube-proxy replacement, transparent encryption.
- **Weaknesses:** eBPF means a newer kernel and a different debugging mental model; some L7 features still require the Envoy integration; "the proxy" is less of a distinct thing you can point at.

### Istio (Envoy, distributed)

Istio is a *control plane* for Envoy proxies — it runs Envoy as gateways (north-south) and as sidecars or ztunnels/waypoints (east-west). See Part 13.

### Cloud load balancers as proxies

AWS ALB (L7), GCP HTTP(S) LB, Azure Application Gateway, Azure Front Door, Cloudflare. These are managed reverse proxies outside your cluster. They're excellent (global anycast, WAF, DDoS protection, managed certs) and they're *outside your GitOps*, which is both a feature (they don't break when your cluster does) and a drawback (their config is in Terraform/console, not in your Kubernetes manifests).

**A very common production topology:** Cloudflare/WAF → cloud L7 LB → Kubernetes Gateway (Envoy/NGINX) → Services → pods. Four proxies deep. It works, but when something breaks you must know which of the four is responsible — which is why trace IDs and consistent access logs across all layers matter enormously (Part 19).

### In-pod proxies

Sometimes you want a proxy inside the pod itself:
- **oauth2-proxy / Envoy with ext_authz** — add authentication to an app that doesn't have it, without changing the app.
- **Database connection proxies** — but note the two examples are different species. `cloud-sql-proxy` is a genuine **sidecar**: it runs in the pod and brokers the connection. **AWS RDS Proxy is not** — it is a managed service living in your VPC that pods connect *to* over the network; it cannot run as a sidecar and is not scaled by your Deployment. Both solve the same problem (secure, IAM-authenticated pooled connections), by different means.
- **mTLS sidecars** — encrypt pod-to-database or pod-to-legacy-service traffic.
- **Protocol adapters** — expose an HTTP health endpoint for a non-HTTP service.
- **Native sidecar containers** (6.1) make these much cleaner than they used to be, with correct startup/shutdown ordering.

## 11.4 What a reverse proxy gives you — feature by feature

Each of these is a real feature you configure, plus the failure mode if you get it wrong.

**Routing.** Host, path (prefix/exact/regex), header, query param, method, and combinations. Port-based and SNI-based routing for TLS. *Failure mode:* precedence confusion. This differs by implementation and is worth checking rather than assuming: **Gateway API ranks matches by specificity regardless of the order they appear in the file** (a more specific match wins even if listed last), whereas NGINX prefix locations use longest-prefix-wins, and some proxies are strictly first-match-in-order. The practical hazard is assuming your file order determines precedence when it does not.

**TLS termination and origins.** Terminate at the edge (simplest, most common), **re-encrypt** to the backend (defense in depth; requires certificates your pods trust), or **passthrough** (proxy can't read or route on L7; useful for end-to-end mTLS to a backend that terminates itself). Plus: SNI-based cert selection, multiple certs on one listener, HTTP→HTTPS redirect, HSTS, minimum TLS version, and cipher suites. *Failure mode:* a certificate in the wrong namespace or with mismatched SANs produces a handshake error that looks like a total outage.

**Load balancing.** Covered in 10.4: round robin, least request, weighted, ring hash/maglev, random, locality-aware, plus **slow start** and **outlier detection**.

**Outlier detection (passive health checking).** The proxy watches backend responses. If a backend returns 5 consecutive 5xx responses, it is ejected from the pool. The ejection duration is not a fixed 30s with exponential backoff — it is `base_ejection_time` **multiplied by the number of times that host has been ejected in a row**, capped by `max_ejection_time`. The host is then re-admitted and re-checked on the interval, and (with active health checking) a single successful check restores it. The practical effect is that a host failing repeatedly is kept out for progressively longer, which is the behaviour you want but not the mechanism the shorthand suggests. This is *enormous* for resilience: it means a single broken pod (bad node, corrupted cache, deadlock) stops receiving traffic within seconds, **without any human intervention and without your readiness probe needing to know.** Active health checking (periodic probe requests) is the complement. Configure both. *Failure mode:* aggressive ejection with low replica counts ejects your entire backend set at once and turns a partial outage into a complete one. Set `maxEjectionPercent`.

**Circuit breaking.** Cap concurrent connections and pending requests per backend. When the cap is hit, fail fast (`503`) instead of queueing unboundedly and collapsing under backpressure. This is the difference between "one endpoint is slow" and "the whole service is down." *Failure mode:* circuit breakers set too low cause spurious 503s under normal bursts.

**Timeouts — three of them, and they are not the same.**
1. **Client/request timeout** — total time the proxy will wait for a response before returning `504`. 
2. **Backend/upstream idle timeout** — how long an idle backend connection is kept.
3. **Connection timeout** — how long to wait for the TCP handshake with a backend.
*Failure modes:* a request timeout shorter than your slowest legitimate request → random 504s on the 99th percentile. A timeout longer than your client's own timeout → your client gives up first and retries, doubling load. **Timeouts must increase as you move outward, never the reverse**: app < gateway < cloud LB < client. If an inner layer times out *after* an outer one, the outer layer returns an error for work the inner layer is still doing, and its retry multiplies the load.

**Retries — and retry budgets.** Retry on 5xx/connect-failure/timeouts, N times, ideally only for idempotent methods, with exponential backoff and jitter. **A retry budget** (e.g. "retries may not exceed 20% of total requests") is the safety valve: without it, a broad outage causes every request to be retried 3×, tripling load on already-struggling backends and turning degradation into collapse. This is the classic **retry storm**, and Gateway API v1.3 added retry budgets to the spec specifically because of it. *Failure mode:* retries without jitter ⇒ synchronized retry waves.

**Header manipulation.** Add, set, or remove request and response headers. Add `X-Request-Id` if absent, strip client-supplied `X-Forwarded-*` (spoofing risk), remove `Server`/`X-Powered-By` (information disclosure), add security headers (`Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options`). *Failure mode:* header name **casing** — HTTP/2 lowercases all header names, so an app doing exact-match on `X-My-Header` breaks when traffic moves to HTTP/2. This is a real and very confusing migration bug.

**Body and buffering control.** Buffering the request/response body at the proxy is usually good (frees the backend), but it breaks **streaming**: Server-Sent Events, chunked responses, long-polling, and gRPC streaming will appear to hang or arrive all at once if buffering is on or if compression is applied to a streaming response. *Failure mode:* "our SSE endpoint works locally and buffers in production" is almost always proxy buffering or a compression filter.

**WebSocket and upgrade handling.** Requires the proxy to *not* apply normal request/response buffering and timeouts to the upgraded connection, and to allow the `Upgrade`/`Connection` headers through. *Failure mode:* WebSocket connections dropping after 60s = the proxy's idle timeout, not your app.

**Compression.** gzip/brotli for text responses. *Failure mode:* compressing already-compressed content wastes CPU; compressing streaming responses breaks them; and BREACH-style attacks exist against compression + secrets. Be deliberate.

**Caching.** Some proxies cache responses (NGINX and Varnish especially). In Kubernetes this is rarer than you'd think — most teams cache at the CDN or in the app — because cache invalidation across ephemeral pods is hard. If you do it, do it deliberately with explicit `Cache-Control` handling.

**Rate limiting.** Per-IP, per-header, per-route, per-JWT-claim. Usually **distributed** (a shared Redis or an Envoy rate-limit service) because each proxy replica enforcing its own limit means your effective limit is `limit × replicas`. *Failure mode:* local-only rate limiting that multiplies by replica count; also rate limiting on a proxy that's behind a cloud LB without `X-Forwarded-For` handling rates the *LB's* IP, not the client's — which rate-limits everyone at once.

**Authentication and authorization.** JWT validation, OIDC/OAuth2 flows, API keys, mTLS client certs, Basic auth, and **external authorization** (ext_authz: the proxy calls your authz service per request). Centralizing auth at the proxy is one of the highest-value uses of a reverse proxy — it means every service behind it gets auth without implementing it. *Failure mode:* auth at the proxy but the backend still reachable directly (Service is ClusterIP but another pod can hit it) ⇒ the auth is decorative. Pair proxy auth with NetworkPolicy so only the gateway can reach the backend.

**WAF and DDoS protection.** OWASP rule sets, bot detection, IP reputation. Available in cloud LB/WAF products, ModSecurity with NGINX, Coraza, or commercial Envoy filters. *Failure mode:* WAF in blocking mode on day one breaks legitimate traffic; run in detection mode first.

**Observability.** Access logs with latency breakdowns, request/response sizes, upstream host, and status; metrics (`requests_total`, `request_duration_seconds`, `upstream_rq_time`, connection pool stats); distributed trace context propagation. This is where the proxy earns its keep operationally: **the proxy's metrics tell you whether a problem is at the edge or in the backend**, and proxy access logs are often the only place you can see the requests that never reached your app.

**mTLS.** Origin mTLS (proxy ↔ backend) and client mTLS (client ↔ proxy). Plus **SPIFFE/SPIRE**-style identity for workload-to-workload auth.

**PROXY protocol and real client IP.** Cloud L4 LBs often can't preserve the client IP. Enabling **PROXY protocol** makes the LB prepend the original source address; the proxy must be configured to expect it. The alternative is `X-Forwarded-For`. *Failure mode:* enabling PROXY protocol on one side only ⇒ all connections fail or all clients appear as the LB's IP. This is a classic 2am incident.

**Request mirroring (shadowing/traffic shadow).** Send a copy of live traffic to a new version without affecting responses. Enormously useful for testing a rewrite against real production traffic. Gateway API v1.3 improved mirroring support. *Failure mode:* the mirrored backend writes to production data.

**Traffic splitting and canary.** Weighted routing across backends — `weight: 95/5`. The single most useful deployment feature in the whole stack (Part 22.3).

## 11.5 The modes: sidecar vs node-level vs library vs eBPF

A proxy can be deployed four ways, with real trade-offs:

| Mode | Example | Pros | Cons |
|---|---|---|---|
| **Sidecar (per-pod proxy)** | Istio (classic), Linkerd | Per-pod identity and policy; works for any protocol; strong mTLS | Resource overhead per pod (CPU/mem × hundreds of pods), added latency hop, iptables redirection complexity, upgrade churn restarts pods |
| **Node-level proxy (ambient)** | Istio ambient **ztunnel**, Cilium per-node | No per-pod overhead, no pod restarts on mesh upgrade, L4 mTLS everywhere | L7 features require an additional **waypoint** proxy per namespace/service |
| **eBPF in the kernel** | Cilium | Lowest overhead, no userspace hop, superb visibility | L7 features need Envoy anyway; kernel/version sensitivity; different debugging model |
| **Library / client-side** | gRPC `round_robin` resolver, finagle-style | No extra process, no extra hop | Language-specific, security logic in app code, inconsistent across services |

**The industry has clearly moved toward ambient/node-level and eBPF** for east-west (because sidecar overhead and upgrade pain became unacceptable at scale), while the edge remains a userspace L7 proxy (because you need L7 features and you have only a few of them). Istio's ambient mode reaching GA in **1.24** is the clearest signal of this shift.

## 11.6 North-south vs east-west, and the proxy's role in each

- **North-south** (outside → in): the edge/gateway proxy. TLS termination, WAF, rate limits, routing by hostname, auth, canary. Configured via **Gateway API** (Part 12). Usually tens of pods, not thousands. L7 features are the point.
- **East-west** (pod → pod): either nothing (plain Services + NetworkPolicy — perfectly fine for most teams), or a mesh (Part 13) when you need mTLS everywhere and per-request policy between services. Usually thousands of flows; overhead matters enormously here.

**Do not use your edge proxy for east-west traffic** by routing service-to-service calls back through the public hostname. It adds latency, couples internal availability to the edge, breaks NetworkPolicy assumptions, and makes incidents harder to reason about. Internal calls should use `Service` DNS names directly.

## 11.7 Proxies and NetworkPolicy: the interaction that breaks people

If you adopt default-deny NetworkPolicy (which you should, Part 18.3), you must explicitly allow:

1. **Gateway → backend pods** on the app port.
2. **Backend pods → DNS** (kube-dns) on 53.
3. **Proxy/gateway pods → their own control plane** for Envoy Gateway the proxy is the xDS **client**: the data plane **dials out** to the control plane's xDS server on port **18000** and subscribes, and configuration updates come back over that same stream. The direction matters when you write policy — you must allow the **gateway pod's egress to the controller on 18000**, not the reverse, plus health checks and the proxy's admin port.
4. **Backend pods → the mesh/waypoint proxy** if you run one.
5. **Health checkers** (cloud LB health checks come from *outside* the cluster and may need pod CIDR allowances or a dedicated allow rule).

**The symptom of getting this wrong is a timeout, not a refusal** — a dropped packet by NetworkPolicy typically looks like "the service hangs," which sends you debugging the app, the DNS, and the gateway before you think to check policy. When you roll out default-deny, have `hubble observe --verdict DROPPED` (Cilium) or a policy audit mode ready.

## 11.8 Debugging the proxy layer

Proxy bugs present as specific, recognizable symptoms. Learn this table:

| Symptom | Most likely cause | Where to look |
|---|---|---|
| **502 Bad Gateway** | Backend refused/reset the connection: app not listening on the expected port, app crashed, backend closed the connection early (e.g. `preStop`/SIGTERM race) | Proxy error logs (they usually say `connection refused` or `upstream reset`), then pod logs |
| **503 Service Unavailable** | No healthy endpoints: readiness failing, endpoints not propagated, circuit breaker open, rate limit rejected | `kubectl get endpointslices`, proxy cluster/upstream health, readiness probe results |
| **504 Gateway Timeout** | Backend exceeded the proxy's request timeout | Compare proxy timeout vs app latency metrics at p99; check for a slow dependency |
| **Random 502s during deploys** | Endpoint propagation lag + no graceful drain | `preStop` sleep, `terminationGracePeriodSeconds`, readiness flip, and `maxUnavailable: 0` |
| **Requests hang, no error** | NetworkPolicy drop (looks like a timeout), or backend not responding and timeout is very long | `hubble observe --verdict DROPPED`, tcpdump in the pod |
| **All requests appear from one IP** | PROXY protocol or `X-Forwarded-For` not configured end to end | LB config + proxy real-IP module |
| **Headers missing at the app** | Header stripped by a proxy hop; or HTTP/2 lowercase mismatch | `curl -v` through each hop, proxy access logs with headers enabled |
| **Streaming responses buffer** | Proxy buffering or compression enabled | Disable buffering on that route; disable compression for SSE/gRPC |
| **WebSocket drops after ~60s** | Proxy idle timeout on the upgraded connection | Raise idle timeout for that route |
| **Traffic unevenly distributed** | L4 per-connection pinning (10.1), or keep-alive pooling, or a hash-based algorithm | Check whether an L7 proxy is actually in the path; check LB algorithm |
| **TLS handshake errors** | Cert mismatch/SAN/hostname, missing Secret, wrong namespace, SNI not matching listener | `openssl s_client -connect host:443 -servername host` |
| **Some routes 404 but others work** | Route match precedence; hostname case; exact vs prefix match | Route status conditions, proxy route config dump |

**The single most useful debugging technique:** check the proxy's own config dump and status. Every implementation exposes it:

```bash
# Envoy-based (Envoy Gateway, Istio): which clusters/routes does the proxy actually have?
# The data-plane Deployment is named envoy-<gateway-namespace>-<gateway-name>-<hash> and runs
# in the Envoy Gateway install namespace. Find it by label instead of guessing:
kubectl get deploy -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=main-gateway
# Then use its real name, e.g.:
kubectl exec -n envoy-gateway-system deploy/envoy-gateway-system-main-gateway-abc123 \
  -- curl -s localhost:19000/config_dump | jq '.configs[] | .["@type"]'
kubectl exec -n envoy-gateway-system deploy/<that-name> -- curl -s localhost:19000/clusters | head -50
kubectl exec -n envoy-gateway-system deploy/<that-name> -- curl -s localhost:19000/stats | grep -E 'upstream_cx|upstream_rq_5xx|ejections'

# NGINX
kubectl exec deploy/nginx-gateway -- nginx -T          # full effective config
kubectl exec deploy/nginx-gateway -- cat /var/log/nginx/error.log

# Traefik
kubectl exec deploy/traefik -- traefik healthcheck
# Traefik dashboard (port-forward only, never publicly)
```

**Remember the layers rule:** if `kubectl port-forward` directly to the pod works but the hostname doesn't, the problem is **above** the pod (Service endpoints, proxy, LB, DNS, TLS) — not in your application. That single test splits the entire search space in half. Part 20.3 continues this method.

## 11.9 Egress proxies (the forward-proxy case)

Egress deserves its own note because it's the mirror image and is usually neglected.

**Why you need egress control:** data exfiltration prevention, compliance ("what external endpoints does this service talk to?"), FQDN allowlisting so a compromised pod can't call `attacker.example.com`, consistent TLS inspection policy, and auditing outbound traffic.

**Your options, cheapest to strictest:**

1. **NetworkPolicy egress with `ipBlock`** — L3/L4 only. Cannot express "only api.stripe.com" (IPs change).
2. **DNS-aware policy** (Cilium `toFQDNs`, Calico DNS policy) — allow by FQDN, resolved at the CNI. Practical and popular.
3. **Egress gateway proxy** (Istio egress gateway, Envoy Gateway as a forward proxy, Squid) — all egress traffic is forced through a proxy that enforces FQDN allowlists, does TLS origination or inspection, and logs everything. The strictest practical option.
4. **Cloud NAT + VPC endpoints** — for cloud services (S3, Secrets Manager), use **VPC endpoints / Private Service Connect** so traffic never traverses the internet at all. This is both a security and a cost win (NAT gateway data processing charges are real).

**The trap:** an egress proxy becomes a **single point of failure for every outbound call in your cluster**. Run it with multiple replicas, a PDB, and — critically — a documented **fail-open vs fail-closed** decision. Failing closed means a proxy outage takes down everything that talks to the internet; failing open means your policy is advisory during outages. Decide deliberately, write it down.

## 11.10 Choosing and sizing a proxy

**How to choose:**

| If you... | Choose |
|---|---|
| Want the modern default, portability, and Gateway API conformance | **Envoy Gateway** |
| Already run Cilium | **Cilium Gateway API** (fewer moving parts, eBPF, LB-IPAM) |
| Want mesh + gateway unified, or need advanced traffic management | **Istio** (Gateway API + ambient) |
| Are on a managed cloud cluster and want least effort | **Cloud-native** (GKE Gateway, AWS LBC/Lattice, AKS App Gateway) |
| Know NGINX and want familiar semantics | **NGINX Gateway Fabric** |
| Want fast setup with automatic TLS on a small cluster | **Traefik** |
| Need raw TCP performance or already use HAProxy | **HAProxy** |

**Sizing and capacity notes:**
- **Proxy resource requests matter enormously.** A gateway handling 50k RPS needs real CPU; under-provisioned proxies are a top cause of unexplained latency and 5xx spikes. Set CPU requests generously (proxies are CPU-bound, and CPU throttling shows up as tail latency) and scale on CPU **and** connection/request rate.
- **Connection pooling** to backends reduces handshake overhead dramatically. Tune max connections per host to match your backends' capacity.
- **TLS is expensive.** Terminating thousands of TLS connections per second costs real CPU; consider session resumption and, at scale, hardware or cloud LB offload.
- **Run multiple replicas** across zones, always with a PDB. A single gateway replica is a single point of failure for your entire product.
- **Keep the proxy stateless.** If the proxy holds state (sessions, rate-limit counters), either externalize it (Redis) or accept that replica loss loses that state.

## 11.11 The honest summary

- **You will run a reverse proxy.** Every production Kubernetes cluster has at least one, usually three (cloud LB, in-cluster gateway, and often a mesh or egress proxy).
- **Gateway API is how you configure the north-south one** (Part 12), and it's implementation-neutral — which is why the choice of *which* proxy matters less than it used to.
- **The customer-facing proxy is where most "weird production behavior" originates**: buffering, timeouts, retries, header handling, and per-connection load balancing. When something is inexplicable, look at the proxy's config dump and access logs before you blame the app.
- **Layer your timeouts, budget your retries, eject outliers, and slow-start new backends.** Those four things prevent the majority of self-inflicted traffic-related outages.

---

# Part 12 — The Gateway API, In Full

## 12.1 What the hell is the Gateway API, actually

**The Gateway API is a set of Kubernetes API objects (CRDs) that describe how traffic enters and moves through your cluster.** It is the successor to `Ingress`, and it fixes Ingress's two structural problems: vendor lock-in through annotations, and an inability to express anything beyond simple HTTP.

But that's the *what*. The important part is *why it's shaped the way it is*.

### The core insight: it separates roles

In the `Ingress` model, one object did everything, so one team owned everything. In reality, at least three different teams touch ingress:

| Role | Cares about |
|---|---|
| **Infrastructure provider** | Which load balancer implementation exists, what IPs/ports it binds, how it's deployed. |
| **Cluster operator** | Shared gateways, TLS certificates, which namespaces may attach to which gateway, policy. |
| **Application developer** | "My service's `/api` path goes to my pods." Nothing else. |

The Gateway API models this explicitly with **three layers of objects owned by three different roles**:

```
GatewayClass          ← "which implementation"     owned by infra provider
    │
    └── Gateway       ← "a specific load balancer: ports, TLS, listeners"   owned by cluster operator
            │
            └── HTTPRoute / GRPCRoute / TLSRoute / TCPRoute / UDPRoute
                          ← "which host/path/header goes to which Service"  owned by app developer
```

Why the separation matters concretely:

- A developer writes an `HTTPRoute` mentioning only their Service. They do **not** need to know whether it's Envoy, Cilium, Istio, HAProxy, or an AWS ALB underneath — and they do not need permission to touch the load balancer.
- An operator owns the `Gateway` and can enforce TLS, allowed hostnames, and which namespaces are permitted to attach routes. Two teams can share one gateway (one IP, one TLS cert, one cloud LB bill) without seeing each other's routes.
- Swapping the implementation means changing the `GatewayClass` reference — **routes are portable across implementations**. That is the entire point, and it's what Ingress's annotation soup destroyed.

**Gateway API is not installed in your cluster.** Like any API extension, the CRDs must be installed by you (or by the implementation's Helm chart). On your empty cluster, `kubectl get gatewayclass` returns "the server doesn't have a resource type" until you do. That's normal, and it's step one.

### The resource family

| Object | Role | Purpose |
|---|---|---|
| **GatewayClass** | Infra | Names a controller (`controllerName: gateway.envoyproxy.io/gatewayclass-controller`). Cluster-scoped. Like StorageClass/IngressClass, but per-implementation. |
| **Gateway** | Operator | A concrete data plane: listeners (port, protocol, hostname), TLS config, address/IP. **This is what actually provisions a cloud load balancer.** |
| **HTTPRoute** | Developer | Host/path/header/query/method matching → backend Services, with filters (rewrite, redirect, header mod, request mirror), weights, timeouts, retries. |
| **GRPCRoute** | Developer | Same, for gRPC (method/service matching, gRPC-specific status handling). |
| **TLSRoute** | Developer | TLS passthrough/termination by SNI without HTTP parsing. |
| **TCPRoute / UDPRoute** | Developer | Raw L4 forwarding. **Graduated to Standard in Gateway API v1.6 (Aug 2026)** — this is the last piece that used to force people back to vendor-specific config for non-HTTP protocols. |
| **ReferenceGrant** | Operator | **Cross-namespace permission.** A route in namespace A wanting to reference a Service in namespace B requires a ReferenceGrant *in namespace B*. This is the security model that stops "any namespace can route to my backend." |
| **BackendTLSPolicy** | Developer | Configure TLS *from* the gateway *to* the backend (re-encrypt to your pods). |
| **Backend / ServiceImport** | Developer | Extensions for non-Service backends (external hosts, S3, multi-cluster services). |
| **Policy CRDs** | Operator/Dev | `ClientTrafficPolicy`, `BackendTrafficPolicy`, `SecurityPolicy` (implementation-specific) for timeouts, rate limits, CORS, auth, retries. |

**Policy attachment** is the extensible part: the core spec defines the routing skeleton, and implementations add policy CRDs that *attach* to a Gateway, a Route, or a backend, with precise precedence rules (more specific wins; direct policy beats inherited). It's how you get rate limiting and JWT validation without another annotation soup.

### Version history that matters

| Version | When | What |
|---|---|---|
| v1.0 | Oct 2023 | GA. GatewayClass/Gateway/HTTPRoute stable. |
| v1.1 | 2024 | GRPCRoute stable, service mesh support. |
| v1.2 | 2025 | Better listener/route semantics. |
| **v1.3** | Jun 2025 | **Request mirroring, CORS, gateway merging, retry budgets** — retry budgets in particular are an operational safety feature (Part 11.4). |
| **v1.4** | Nov 2025 | Continued stabilization, more fields to Standard. |
| **v1.5** | Apr 2026 | Moving features to Stable. |
| **v1.6** | Aug 2026 | **TCPRoute and UDPRoute graduate to Standard.** |
| **Inference Extension** | 2025–2026 | LLM-specific routing (Part 30). |
| **Ingress2Gateway 1.0** | Mar 2026 | Official Ingress→Gateway migration tooling. |

### What Gateway API gives you that Ingress never did

- **Native traffic splitting** — `weight: 90/10` for canaries, first-class in the spec.
- **Native header/query/method matching** — no annotations.
- **Timeouts and retries (with budgets)** — in the object.
- **Cross-namespace routing with explicit grants** — safe delegation.
- **Non-HTTP protocols** — TCP, UDP, TLS passthrough, gRPC as first-class (and now Standard, not experimental).
- **Multiple listeners on one gateway** — different hostnames, different TLS certs, different ports, one IP.
- **Portability** — the same `HTTPRoute` works on Envoy Gateway, Cilium, Istio, Kong, Traefik, NGINX Gateway Fabric, HAProxy, and the cloud implementations, modulo implementation-specific policies.

### Which implementation should you pick?

| Implementation | Good choice when |
|---|---|
| **Envoy Gateway** | You want a neutral, CNCF-backed, Envoy-based data plane, strong conformance, easy standalone install. Excellent default for self-managed. |
| **Cilium Gateway API** | You already run Cilium. Fewer moving parts, eBPF dataplane, LB-IPAM for on-prem addresses. |
| **Istio Gateway API** | You want the mesh and the gateway unified, or need advanced traffic management/ambient mode. |
| **Cloud-native** (GKE Gateway, AWS Load Balancer Controller / VPC Lattice, AKS Application Gateway, Azure Front Door) | Managed clusters. These integrate with cloud LBs, WAFs, and IAM, and are usually the least-effort path. |
| **NGINX Gateway Fabric, Kong, Traefik, HAProxy** | Existing familiarity/licensing, specific feature needs. |
| **Contour, Gloo, Emissary** | Envoy-based, mature, historically Gateway-API-forward. |

**Do not run two Gateway API implementations for the same purpose.** They are not additive; multiple GatewayClasses mean multiple data planes to secure, monitor, and pay for.

## 12.2 Installing Gateway API on your empty cluster, end to end

The canonical quickstart is **Envoy Gateway**, because it's a single Helm chart and a neutral Envoy data plane. The same shape applies to any implementation.

### Step 0 — prerequisites that people forget

You need:
1. A CNI that works (Part 9.2). Gateway pods are just pods; if pod networking is broken, nothing here works.
2. **A way to get traffic to the gateway.** On a cloud cluster, `type: LoadBalancer` on the generated Service uses the cloud controller. **On bare metal / homelab there is no cloud LB**, so you need one of:
   - **MetalLB** (ARP/BGP-based LB IP allocation) — the standard answer.
   - **Cilium LB-IPAM** if you run Cilium.
   - **kube-vip**.
   Without this, your Gateway will sit `Programmed: False, Address: <pending>` forever. This is the #1 "Gateway API doesn't work" cause on-prem.
3. **DNS** that can point a hostname at the gateway's external IP.
4. **cert-manager** (or your own certs) if you want TLS — Part 14.

### Step 1 — install the CRDs and the controller

```bash
# Envoy Gateway publishes only an OCI chart, so there is no `helm repo add` step
# (adding a repo would fail and `helm search repo` would find nothing).
helm show chart oci://docker.io/envoyproxy/gateway-helm --version v1.9.0   # confirm the version

helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.9.0 \
  --namespace envoy-gateway-system --create-namespace
```

**Read the release notes before upgrading any Gateway implementation.** Envoy Gateway v1.5.0, for example, documented **breaking changes** — Gateway API projects move fast and deprecations happen. Pin the version, read the notes, upgrade dev first.

Verify the controller is alive *before* writing any Gateway:

```bash
kubectl get pods -n envoy-gateway-system
kubectl get gatewayclass            # you should now see 'eg' with ACCEPTED=True
kubectl get crd | grep gateway.networking.k8s.io
```

**What is a GatewayClass, conceptually?** It's the bridge between "a Gateway object" and "some piece of software that knows how to make it real." Its essential field is `controllerName`, which must match the string the controller watches for. Multiple GatewayClasses can exist (e.g. one for internal, one for internet-facing); each points at a different controller configuration. A Gateway that references a GatewayClass nobody implements stays `Accepted: False` forever — which is exactly how you detect a typo.

### Step 2 — check that a LoadBalancer can actually get an IP

```bash
kubectl create deployment whoami --image=traefik/whoami --replicas=1
kubectl expose deployment whoami --port=80 --type=LoadBalancer
kubectl get svc whoami -w        # EXTERNAL-IP must become a real IP, not <pending>
```

If it stays `<pending>` on bare metal, install MetalLB now:

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace

# Give MetalLB a pool of addresses from your LAN
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: default-pool, namespace: metallb-system }
spec:
  addresses: ["192.168.1.240-192.168.1.250"]
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: { name: default, namespace: metallb-system }
spec: { ipAddressPools: ["default-pool"] }
EOF
```

### Step 3 — write the Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: gateway-system
spec:
  gatewayClassName: eg                 # which implementation fulfils this
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: "*.example.com"        # optional: restrict hostnames here
      allowedRoutes:
        namespaces:
          from: Selector               # security: who may attach routes?
          selector:
            matchLabels:
              shared-gateway-access: "true"
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: wildcard-example-com-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              shared-gateway-access: "true"
    - name: https-internal
      protocol: HTTPS
      port: 8443
      hostname: "internal.example.com"
      tls: { mode: Terminate, certificateRefs: [{ name: internal-tls }] }
      allowedRoutes: { namespaces: { from: Same } }
```

```bash
kubectl apply -f gateway.yaml
kubectl get gateway -n gateway-system -w
# NAME           CLASS   ADDRESS        PROGRAMMED   AGE
# main-gateway   eg      192.168.1.240  True         30s
```

**Every listener is a port/protocol/hostname tuple, and `allowedRoutes` is your multi-tenancy control.** `from: Same` means only routes in the Gateway's own namespace may attach. `from: All` is convenient and dangerous — any namespace can claim your hostnames. `from: Selector` is the sane default: namespaces opt in with a label, and only your platform team can set labels.

### Step 4 — the developer writes a route

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: prod                      # must be permitted by allowedRoutes
  labels: { shared-gateway-access: "true" }
spec:
  parentRefs:
    - name: main-gateway
      namespace: gateway-system
      sectionName: https               # attach to a SPECIFIC listener
  hostnames: ["api.example.com"]       # must be within the listener's hostname
  rules:
    - matches:
        - path: { type: PathPrefix, value: /v1 }
      backendRefs:
        - name: api                      # a Service in THIS namespace
          port: 80
          weight: 100
    - matches:
        - path: { type: PathPrefix, value: /v2 }
          headers:
            - { name: x-canary, value: "true" }
      backendRefs:
        - { name: api, port: 80, weight: 90 }       # 90% to v1 Service
        - { name: api-v2, port: 80, weight: 10 }    # 10% canary
      timeouts: { request: 30s, backendRequest: 10s }
      # NOTE: `retry` is a single object (not `retries`), and `backoff` is a
      # Duration STRING, not a map. `retry` and `sessionPersistence` are
      # Experimental — they need the Experimental CRD channel (see 12.1).
      retry:
        attempts: 3
        codes: [502, 503, 504]
        backoff: 100ms
```

**Two schema details here catch people out.** The field is **`retry`**, singular — `retries` is rejected or silently ignored depending on the implementation, so a route that "looks configured" for retries simply is not. And **`backoff` is one duration string** (`100ms`), not an object with `base`/`max`; implementations may add their own exponential growth and jitter above the minimum you set, which is why no maximum is expressible. Both come directly from the `HTTPRouteRetry` type, and both are **Experimental**, so a Standard-channel install will not accept them.

```bash
kubectl get httproute -A
kubectl describe httproute api-route -n prod    # check Status.parents for Accepted/ResolvedRefs
kubectl get svc -n gateway-system               # the generated Envoy proxy service + its IP
```

**Read the route's status.** Gateway API objects report their health in `status.parents[]`, but the condition sets differ by kind and conflating them costs you real time. **Gateways** report `Accepted` and `Programmed` — which is what the `PROGRAMMED` column in `kubectl get gateway` shows. **Routes** report `Accepted`, `ResolvedRefs`, and `PartiallyInvalid` (when some rules were dropped but others are serving). There is no `Programmed` condition on a Route, so `kubectl wait --for=condition=Programmed httproute/...` never completes — a mistake worth avoiding because it looks like a hang rather than an error. When traffic doesn't flow, this is the first place to look, and each condition has a `reason` and `message` that usually states the problem outright (`NoMatchingParent`, `InvalidKind`, `RefNotPermitted`, `BackendNotFound`).

### Step 5 — DNS

Point `api.example.com` at the Gateway's address. Options, in increasing order of automation:

1. **Manual A record** — fine to start.
2. **external-dns** — watches Gateway/HTTPRoute objects and creates DNS records automatically in Route53/Cloudflare/Google DNS:
   ```yaml
   metadata:
     annotations:
       external-dns.alpha.kubernetes.io/hostname: api.example.com
       external-dns.alpha.kubernetes.io/target: 192.168.1.240
   ```
   With `--source=gateway-httproute`, external-dns derives records from your routes. Use a **wildcard DNS record** (`*.example.com` → gateway IP) to avoid per-service DNS toil.
3. **Cloud-native DNS** (GKE's Gateway + Cloud DNS integration) — fully managed.

## 12.3 Gateway API vs Ingress: the honest comparison

| Capability | Ingress | Gateway API |
|---|---|---|
| HTTP routing | ✅ | ✅ |
| Config beyond basic routing | ❌ annotations (non-portable) | ✅ typed fields + policy CRDs |
| Traffic weighting | ❌ annotations | ✅ `weight` |
| Header/query/method match | ❌ annotations | ✅ |
| gRPC / TCP / UDP / TLS-passthrough | ❌ (TCP/UDP via annotation/ConfigMap) | ✅ first-class, **Standard as of v1.6** |
| Cross-namespace routing | ❌ unsafe or impossible | ✅ `ReferenceGrant` |
| Role separation | ❌ one object | ✅ GatewayClass / Gateway / Route |
| CORS | ❌ | ✅ **Standard as of v1.5** |
| Retry budgets, request mirroring | ❌ | ✅ mirroring Standard; `retry` is Experimental |
| Portability across vendors | ❌ effectively none | ✅ core spec is portable |
| Ecosystem direction | ❄️ frozen, reference impl retired 2026 | ✅ active |

**Verdict for a new, empty cluster in 2026: use Gateway API. Do not build a new Ingress-based setup.** If you inherit Ingress objects, use **Ingress2Gateway** (1.0, Mar 2026) to migrate, and read the "Five Surprising Ingress-NGINX Behaviors" guidance first — the migration is not purely mechanical.

## 12.4 Migrating from Ingress

If you inherit an Ingress-based cluster:

1. **Inventory** controllers in use and the annotations you depend on. Annotations are the migration work; core routing is mechanical.
2. **Run `ingress2gateway`** to generate equivalent Gateway API resources from existing Ingress objects, then review the output by hand.
3. **Stand up the Gateway API implementation alongside** the old controller — both can coexist (different IngressClasses/GatewayClasses).
4. **Migrate DNS one hostname at a time** by pointing records at the new Gateway's IP. This gives you instant rollback (repoint DNS) instead of a big bang.
5. **Watch for behavioral differences**: path matching semantics (regex support, trailing slash handling), header casing, `rewrite-target` equivalents, default backend behavior, and client IP handling. These are where "it worked on Ingress" turns into a bug report.
6. **Decommission** the old controller only after every hostname has been moved and observed through a full traffic cycle.

---

# Part 13 — Service Mesh

A service mesh is **a distributed reverse proxy with a control plane** — the same technology as Part 11, pointed at east-west traffic, with identity and policy layered on. Part 11.5 covered the deployment modes; this part covers when and why.

## 13.1 What a mesh actually provides

1. **mTLS everywhere, automatically** — every pod-to-pod connection encrypted and mutually authenticated, with certificates issued and rotated by the control plane. This is the #1 reason companies adopt a mesh (compliance, zero-trust).
2. **Workload identity** — each workload gets a cryptographic identity (SPIFFE ID like `spiffe://cluster.local/ns/prod/sa/api`), and policy is written against identity, not IP.
3. **L7 authorization** — "service A may only `POST /orders` on service B," enforced cryptographically, not by network location.
4. **Traffic management** — per-route retries, timeouts, circuit breaking, weighted traffic shifting, fault injection, mirroring, locality-aware routing.
5. **Observability** — uniform L7 metrics and traces for every service regardless of language or instrumentation.

## 13.2 The three main options

| | **Istio** | **Linkerd** | **Cilium Service Mesh** |
|---|---|---|---|
| Data plane | Envoy (sidecar or ambient ztunnel+waypoint) | linkerd2-proxy (Rust micro-proxy, sidecar) | eBPF + Envoy where needed |
| Complexity | Highest | Lowest | Medium |
| Feature depth | Deepest | Core features done well | Growing |
| Overhead | Sidecar: notable; Ambient: much lower | Lowest of the sidecar options | Lowest overall |
| Best for | Large orgs needing everything, including multi-cluster | Teams wanting mTLS + reliability with minimal ops | Cilium users wanting L4 policy/observability without sidecars. **Important:** Cilium **deprecated its Mutual Auth (mTLS) feature in 1.20** and plans to remove it; its direction for in-cluster mTLS is ztunnel-based support (beta). Verify status before choosing Cilium *for mTLS*. |

**Istio ambient mode** (GA since **Istio 1.24**, November 2024) is the significant recent change: instead of a proxy in every pod, a **per-node ztunnel** handles L4 mTLS and identity, and **waypoint proxies** are deployed per namespace/service only where L7 features are needed. This removes the two worst sidecar problems — per-pod resource overhead and pod restarts on every mesh upgrade — while keeping mTLS ubiquitous.

## 13.3 Should you run a mesh?

**Adopt a mesh when you can name the problem it solves:**

- ✅ "We need mTLS between all services for compliance." — Clear, legitimate.
- ✅ "We need per-request authorization between services, and we have 40 services in 6 languages." — Legitimate.
- ✅ "We need uniform L7 metrics/traces without instrumenting every service." — Legitimate.
- ✅ "We need cross-cluster service discovery and failover." — Legitimate.
- ❌ "It's best practice." — Not a reason.
- ❌ "We have 3 services." — A mesh is more infrastructure than your application.
- ❌ "We want retries." — Your gateway or your client library can do that with zero new infrastructure.

**The costs are real:** a new control plane to operate and upgrade, a new debugging surface (Part 20.3 covers mesh-specific diagnosis), added latency and resource consumption, and a new way to cause a cluster-wide outage (a bad mesh config push can break all service-to-service traffic).

**A defensible path:** Gateway API at the edge from day one; NetworkPolicy for east-west isolation; a mesh only when mTLS or L7 inter-service authz becomes a requirement. Because Istio and Cilium both implement Gateway API, adding a mesh later doesn't force you to rewrite your routes — which is another argument for Gateway API over Ingress.

## 13.4 Mesh and Gateway API together

In Istio, the mesh *is* a Gateway API implementation: `Gateway` resources with `gatewayClassName: istio` are the ingress layer, and `HTTPRoute`s attach to them. In ambient mode, routes can also be attached to **waypoints** for internal L7 policy. So the mental model is unified: **Gateway API describes routing; the mesh decides how much proxy is in the path.**

---

# Part 14 — TLS and Certificates

**cert-manager** is effectively mandatory. It watches `Certificate` objects and Ingress/Gateway annotations, obtains certs from ACME (Let's Encrypt), a private CA, Vault, or a cloud ACM, and writes them into `Secret`s that the Gateway consumes. It also handles renewal — which is the real reason you use it, because hand-renewed certificates is exactly the toil that causes outages.

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true \
  --set config.gatewayAPI.enabled=true \
  --version v1.21.0        # check cert-manager.io/docs/releases before pinning

# NOTE: Gateway API support is NOT enabled by default. Without
# config.gatewayAPI.enabled=true, cert-manager ignores Gateway/HTTPRoute
# resources entirely, and an `http01.gatewayHTTPRoute` solver never runs —
# the Certificate just sits there without being issued.
```

**Two issuer kinds:**

- **ClusterIssuer** — cluster-scoped, usable by any namespace (typical for public Let's Encrypt).
- **Issuer** — namespaced (typical for a private internal CA per team).

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt-prod }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@example.com
    privateKeySecretRef: { name: letsencrypt-prod-account-key }
    solvers:
      - http01:                                  # simplest: serves /.well-known/...
          gatewayHTTPRoute:
            parentRefs:
              - { name: main-gateway, namespace: gateway-system, kind: Gateway }
      # or dns01 with your provider — REQUIRED for wildcards
      # - dns01:
      #     route53: { region: eu-west-1 }
```

Then a certificate, either explicitly or via annotation on the Gateway:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: { name: wildcard-example-com, namespace: gateway-system }
spec:
  secretName: wildcard-example-com-tls
  dnsNames: ["example.com", "*.example.com"]
  issuerRef: { name: letsencrypt-prod, kind: ClusterIssuer }
```

**Details that bite:**

- **HTTP-01 cannot issue wildcards.** Wildcards require DNS-01, which requires your DNS provider's API credentials in the cluster (secure them — see Part 8).
- **Rate limits are real.** Let's Encrypt production has strict limits. Test against the **staging** endpoint first (`acme-staging-v02.api.letsencrypt.org`), then switch.
- **Renewal** defaults to 2/3 of lifetime. Monitor `certmanager_certificate_expiration_timestamp_seconds` and alert on <21 days. Silent certificate expiry is one of the most embarrassing and most common outages.
- **Certificates live in Secrets** — which means RBAC on Secrets and encryption at rest for Secrets matter (Part 8).
- **Internal TLS** — use a private CA issuer and distribute the CA bundle via **trust-manager**, so in-cluster services can talk TLS without public certs. Kubernetes 1.37's **Cluster Trust Bundles** are the API-native version of this idea.
- **Certificate transparency and revocation** — know how you'd revoke a leaked cert (short lifetimes make revocation less critical; that's the modern argument for 90-day certs).

**TLS in the proxy chain:** decide explicitly where TLS terminates and whether you re-encrypt. Common patterns:
- Terminate at cloud LB, plaintext to the cluster — fast, simple, weaker.
- Terminate at cloud LB, re-encrypt to the Gateway — better.
- **Terminate at cloud LB → re-encrypt to Gateway → re-encrypt to backend (`BackendTLSPolicy`)** — strongest, most certificates to manage, most places for a cert to expire.
- **Passthrough to the backend** — the Gateway can't route on L7, so this only works for single-backend listeners (or SNI routing).

Pick deliberately; the "everything encrypted everywhere" option is genuinely more operational work, and you should know you're signing up for it.

---

# Part 15 — DNS, Egress, and Multi-Cluster Networking

## 15.1 External DNS and service discovery

- **external-dns** keeps public/private DNS in sync with Kubernetes objects (Service, Ingress, HTTPRoute, Gateway). Sources configured per record type; providers for Route53, Cloudflare, Azure DNS, Google Cloud DNS. Use **wildcard records** to reduce toil, and scope external-dns to a specific zone to prevent it from deleting records it doesn't own.
- **Internal DNS zones** for private services (split-horizon DNS) — a private zone that resolves `*.internal.example.com` to internal IPs only.
- **`ExternalName` Services** for aliasing external dependencies into cluster DNS — useful for migration (point at the external DB now, switch to in-cluster later without changing app config).

## 15.2 Egress, revisited

Part 11.9 covered egress proxies. The infrastructure-level complements:

- **Cloud NAT gateways** — outbound internet access for private subnets. Watch the **data processing charges**; they're often a surprise line item, and routing S3/cloud-API traffic through a NAT gateway instead of a VPC endpoint is both slower and more expensive.
- **VPC endpoints / Private Service Connect / PrivateLink** — reach cloud services without traversing the internet. Security *and* cost win. Do this for S3, ECR, Secrets Manager, and any managed database.
- **Static egress IPs** — if a third party allowlists your IPs, you need a NAT gateway with an elastic IP or a dedicated egress proxy. This is a common enterprise requirement that's easy to forget until a vendor says "we need your IP."

## 15.3 Multi-cluster networking (preview — see Part 28)

Options, briefly, because this is a deep topic:

- **Gateway API multi-cluster** — `HTTPRoute` with `ServiceImport` backends, via implementations like Cilium ClusterMesh, Istio multi-cluster, or cloud-specific multi-cluster gateways.
- **Cilium ClusterMesh** — flat pod networking across clusters, cross-cluster service discovery, and global services with local failover. Genuinely excellent for multi-cluster Kubernetes on the same network fabric.
- **Submariner** — similar goals, different implementation (IPsec tunnels, broker-based).
- **Service mesh multi-cluster** — Istio's multi-primary/multi-network models; powerful and complex.
- **DNS-based failover** — the crude but universal fallback (health-checked DNS records pointing at different clusters' gateways). Slow to fail over (DNS TTLs), but works across clouds and providers with no shared control plane.

**The honest advice:** most teams should start with **independent clusters per environment/region and DNS-level failover**, then adopt cross-cluster service discovery only when they can articulate why a request must cross a cluster boundary rather than being served locally.

---

# Part 16 — Storage

## 16.1 The mental model

Kubernetes storage has a deliberate indirection, and understanding why saves hours:

```
Pod → volumeMount
        │
      Volume (in pod spec)
        │
      PersistentVolumeClaim (PVC)     ← the developer's request: "10Gi, ReadWriteOnce, fast"
        │
   StorageClass                       ← the operator's offer: "here's how to make one"
        │
      CSI driver → cloud disk / NFS / Ceph / local NVMe
        │
      PersistentVolume (PV)            ← the actual thing, usually created automatically
```

**Why the indirection?** Because the developer shouldn't choose a disk from a specific provider. They declare *requirements*; the platform declares *capabilities*; and something binds the two — which controller does the binding turns out to matter when a pod will not start. With immediate binding, the **PV controller** in kube-controller-manager matches a PVC against a PV on storage class, access modes, and capacity (≥ the request), and binds them one-to-one. With `volumeBindingMode: WaitForFirstConsumer`, the PV controller steps aside: the **scheduler's volume-binding plugin** drives provisioning *during* scheduling, so the volume is created in the zone the pod actually landed in. That is the whole point of the setting — with immediate binding and a multi-zone cluster, the volume may be created in a zone where no node can run the pod, and the pod then waits forever on a volume-node affinity error. **Dynamic provisioning** means creating a PVC with a `storageClassName` auto-creates the backing volume — no admin action needed.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: api-data }
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: fast-ssd
  resources: { requests: { storage: 20Gi } }
```

**Access modes — the single most misunderstood part of Kubernetes storage:**

| Mode | Meaning | Reality |
|---|---|---|
| `ReadWriteOnce` (RWO) | Read-write by **a single node** | Multiple pods on the *same* node can mount it; the volume plugin enforces one node at attach time. This is the common cloud-disk mode, and it is **not** a barrier to running multiple replicas: a StatefulSet gives each replica its **own** PVC from `volumeClaimTemplates`, so an RWO multi-replica datastore spread across nodes is entirely normal. What RWO *does* prevent is **sharing one PVC across pods on different nodes** — which fails at attach with a multi-attach error. `ReadWriteOncePod` exists to make single-pod restriction explicit. |
| `ReadOnlyMany` (ROX) | Read-only, many nodes | Needs shared storage (NFS, CephFS). |
| `ReadWriteMany` (RWX) | Read-write, many nodes | **Needs a network filesystem** (NFS, EFS, CephFS, Azure Files, Filestore). Slower, more expensive, but required for "many pods share a directory." |
| `ReadWriteOncePod` (RWOP) | Single *pod* only | The strict version of RWO; good for guarding against accidental double-mount. |

**Set a default StorageClass.** Otherwise every PVC must name one, and `kubectl get pvc` sits `Pending` with `no default StorageClass` — a top-5 beginner incident:

```bash
kubectl get storageclass
kubectl patch storageclass <name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Common StorageClasses people define: `fast-ssd` (gp3/premium SSD, RWO), `standard` (cheaper, RWO), `shared` (EFS/NFS, RWX), plus `volumeBindingMode: WaitForFirstConsumer` so volumes are created in the same zone as the pod (essential for multi-AZ clusters — with `Immediate`, your volume can be provisioned in a zone where no node can schedule the pod, and the pod hangs `Pending` forever with a volume-node affinity error).

**Expansion:** PVCs can grow if the StorageClass has `allowVolumeExpansion: true`. **They can never shrink.** Plan sizes accordingly; you'll be deleting and migrating to shrink. Kubernetes 1.34 made **recovery from a failed expansion** a first-class flow, and **VolumeAttributesClass** (GA in 1.34) lets you change volume *performance* attributes (IOPS/throughput tier) without recreating the volume — genuinely useful for databases that need more IOPS under load.

**Reclaim policy:** `Delete` (the default for dynamically provisioned volumes, inherited from the StorageClass) removes the backing disk when the PVC is deleted — **data loss**. `Retain` keeps it, and the PV moves to `Released`.

**`Released` is a dead end, and that surprises people.** The PV still holds a `claimRef` to a PVC that no longer exists, so it will **not** bind to any new PVC until an administrator intervenes — clearing the `claimRef` or deleting and recreating the PV. And note that `Retain` protects the *PV object*, not necessarily the *data*: the behaviour of the underlying disk depends on the storage backend, so "the PV is still there" and "the data is still reachable" are not the same statement. For anything precious, `Retain` plus a real backup process.

**Snapshots:** `VolumeSnapshotClass` + `VolumeSnapshot` (CSI snapshot API), with **volume group snapshots** GA in 1.36 for consistent multi-volume snapshots. Useful, but **not** a backup strategy by itself — see Part 17.

**What not to do:** don't use `hostPath` (pins your pod to one node, breaks on rescheduling, is a security hole), don't use `emptyDir` for anything you need to survive a restart (it's deleted with the pod), and don't assume a PVC is a backup.

**Ephemeral storage is a schedulable resource too.** Everything in this part concerns persistent volumes; the *other* kind of storage — container writable layers, logs, and `emptyDir` — is covered in [Part 38](#part-38--ephemeral-storage-node-pressure-and-resource-exhaustion).

**One more thing about rescheduling:** a pod with an RWO cloud disk is bound to the zone of that disk. Node loss in that zone means the pod cannot start elsewhere until the volume is available. This is a real availability constraint for stateful workloads, and it's why database replicas must be spread across zones with their own volumes (Part 17).

---

# Part 17 — Databases and Stateful Data

This is the question everyone asks, and the honest answer is: **it depends, but the default has moved decisively toward managed external databases.**

## 17.1 The decision table

| Option | Use when | Costs |
|---|---|---|
| **Managed cloud DB** (RDS/Aurora, Cloud SQL, Azure SQL, MongoDB Atlas, Neon, Supabase) | **Default for most teams and almost all production OLTP.** | Money; some vendor lock-in; network egress; latency if cross-region. |
| **Kubernetes operator** (CloudNativePG, Crunchy PGO, Percona, Zalando, Vitess, MongoDB Operator) | You must self-host (on-prem, data residency, cost at scale, Dev/Test clusters). | **You now operate a database.** Upgrades, failover, backups, PITR, tuning all become your job. |
| **External VM/bare-metal DB** | Legacy migration paths, licensing (Oracle), very large instances. | Not cloud-native; separate ops. |
| **In-cluster StatefulSet, hand-rolled** | **Almost never.** Learning only. | You will get failover, backup, and upgrades wrong. |
| **SQLite / embedded** | Edge, tiny apps, read-mostly, a single replica | No HA. Fine for genuinely small things. |

**Why managed is the default:** your managed database gets automated backups with point-in-time recovery, patching, multi-AZ failover with a tested promotion path, read replicas, parameter tuning, and monitoring — provided by people whose full-time job is that database. Running Postgres in Kubernetes *well* is a real skill; running it badly is how companies lose data. Kubernetes gives you a pod scheduler, not a database.

**Why you might self-host anyway:** cost at very large scale (managed DB pricing scales hard), strict data-residency or air-gapped requirements, on-prem where there is no managed option, or dev/test where you want cheap ephemeral databases.

## 17.2 If you self-host: use an operator, not raw manifests

Operators encode the operational knowledge (failover, replica promotion, backup, version upgrades) as controllers. For PostgreSQL, **CloudNativePG** is the modern CNCF choice; **Crunchy PGO** and **Percona** are also solid. Similar patterns exist for MySQL (Vitess, Percona XtraDB), MongoDB, Redis (though for Redis, measure whether you need persistence at all), Kafka (Strimzi), and ClickHouse.

What an operator actually does that you would otherwise do by hand:
- Bootstraps the cluster (initdb, replication setup, users).
- Watches primary health and **promotes a replica** on failure.
- Manages the headless Services for read/write split and replica discovery.
- Coordinates **rolling minor-version upgrades** with zero or minimal downtime.
- Schedules **base backups** and streams **WAL** for PITR.
- Exposes metrics for lag, connections, replication status.

```yaml
# CloudNativePG shape (illustrative)
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: pg, namespace: data }
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.2
  storage:
    size: 100Gi
    storageClass: fast-ssd
  resources:
    requests: { cpu: "2", memory: 4Gi }
    limits:   { memory: 4Gi }
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      isWALArchiver: true
  backup:
    target: primary
    pluginConfiguration:
      name: barman-cloud.cloudnative-pg.io
      # The object store itself is now an ObjectStore CR managed by the plugin.
      # In-tree `barmanObjectStore` is deprecated as of CloudNativePG 1.26.
  monitoring: { enablePodMonitor: true }
  affinity:
    topologyKey: topology.kubernetes.io/zone
    enablePodAntiAffinity: true
```

## 17.3 The rules if you self-host (non-negotiable)

1. **Anti-affinity across zones.** Three replicas in one zone is not HA; it's three failures with one cause.
2. **Storage that survives pod rescheduling**, with `WaitForFirstConsumer` binding and enough IOPS. Database performance is 90% storage.
3. **Backups to object storage, off-cluster.** A backup in the same cluster (or same cloud account) is not a backup — a cluster deletion or account compromise takes both. **Test restores on a schedule.** An untested backup is a hope.
4. **PITR, not just nightly dumps.** A nightly dump loses up to 24h of data. WAL archiving gives you point-in-time recovery.
5. **Resource limits set high enough not to OOM, and requests == limits** (`Guaranteed` QoS) so the DB is last to be evicted.
6. **PodDisruptionBudget** that doesn't block failover, and a **maintenance window** strategy for node upgrades.
7. **Connection pooling.** Postgres forks a process per connection; 500 app pods × 10 connections = dead database. Use **PgBouncer** (transaction mode) as a sidecar or a Deployment, or a pooler built into your operator. This is *the* most common self-hosted-DB outage.
8. **Migrations as explicit, versioned steps**, run via Jobs/Argo hooks — never as an implicit side effect of app startup in a multi-replica Deployment (N replicas racing to migrate = corruption).
9. **Secrets in an external store** (Part 8), not committed to git.
10. **Know your failover time.** Measure it. "It should fail over" and "it fails over in 40 seconds, and our clients time out at 30" are different facts.

## 17.4 Caching and other stateful systems

- **Redis/Valkey** — often used as a cache and a session store. If it's a *cache*, run it as a Deployment with no persistence and accept that a restart empties it (and make sure your app handles that). If it's a *datastore* (queues, sessions you can't lose), it needs persistence, replication, and failover — i.e. an operator or a managed service. **Many "Redis outages" are actually "someone treated a cache as a database."**
- **Message queues** (Kafka, RabbitMQ, NATS) — stateful, need stable identity, and Kafka especially needs careful storage and anti-affinity. Strimzi is the mature Kafka operator.
- **Search** (Elasticsearch/OpenSearch) — stateful, memory-hungry, and easy to run badly. Managed services are usually worth it.
- **Object storage** — **do not run S3 in your cluster.** Use a bucket (Part 17.5).

## 17.5 Object storage: do not run S3 in your cluster

**Use a bucket, not a pod.** Object storage (S3, GCS, Azure Blob, MinIO on separate hardware) is the correct home for: user uploads, static assets, backups, logs, model artifacts, data-lake files, and anything large/binary.

Reasons not to run the object store *inside* the cluster: it's a distributed system needing its own hardware and failure domain; it becomes a circular dependency (your cluster needs storage to store its own backups); and object storage is cheap and effectively infinite when managed. Give pods access via **IRSA / Workload Identity** so no static keys exist (Part 18), and presign URLs so clients upload directly rather than proxying through your app.

**The standard pattern:** app pods are stateless and hold no user data; a managed SQL database holds relational state; a bucket holds files; the cluster holds only compute and config. This is what makes your cluster disposable — and a disposable cluster is a cluster you can upgrade, rebuild, and recover confidently.

---

# Part 18 — Security

An empty cluster is insecure by default in specific, enumerable ways. Here's the hardening path, in priority order.

## 18.1 Admission control and Pod Security Admission

**Admission controllers** run *after* authentication/authorization and can mutate or reject objects. **Pod Security Admission (PSA)** is the built-in implementation of the Pod Security Standards, applied per namespace via labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Three levels:
- **privileged** — no restrictions. Effectively "anything goes."
- **baseline** — blocks known privilege escalations: no `hostNetwork`/`hostPID`/`hostIPC`, no privileged containers, no `hostPath` volumes, restricted capabilities, and controls on host ports. Two precisions worth having: baseline's HostPorts rule allows port `0` and "a known list" rather than forbidding host ports outright, and the built-in PSA controller does not support the known-list form — so non-zero `hostPort`s are permitted at baseline. Baseline is also broader than "privilege escalation" suggests: it constrains AppArmor profile overrides, SELinux type/user/role, sysctls, `procMount`, and host-probe/lifecycle-hook settings.
- **restricted** — baseline + must run as non-root, `allowPrivilegeEscalation: false`, all capabilities dropped, `seccompProfile: RuntimeDefault`, and a non-root `runAsUser`.

**Roll out with `enforce: baseline` + `audit/warn: restricted` first.** Enforcement without warning mode will break workloads in production at an unpredictable time. `restricted` requires your images to actually run as non-root — which is an image-build change, not a manifest change.

**Validating admission with policy engines** — PSA covers pod security; for everything else ("no `:latest` tags", "every Deployment must have resource limits", "no LoadBalancer Services in this namespace", "images must come from our registry"), you need a policy engine:

| Engine | Language | Notes |
|---|---|---|
| **ValidatingAdmissionPolicy** | CEL, **built into Kubernetes** | No extra controller to keep alive. The modern first choice for common cases. 1.36 added manifest-based admission control so policies can't be casually deleted. |
| **Kyverno** | Kubernetes-native YAML | Easiest to adopt, can also mutate and generate resources. Great default when you need more than CEL. |
| **OPA Gatekeeper** | Rego | Most powerful/expressive, steeper learning curve, large policy library. |

```yaml
# Kyverno: require resource limits, run in Audit first, then Enforce
# Kyverno is moving from ClusterPolicy to CEL-based ValidatingPolicy.
# `spec.validationFailureAction` is deprecated in favour of per-rule failureAction.
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata: { name: require-resources }
spec:
  validationActions: [Audit]            # flip to [Deny] once the cluster is clean
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >-
        object.spec.containers.all(c,
          has(c.resources) && has(c.resources.requests) &&
          has(c.resources.requests.cpu) && has(c.resources.requests.memory) &&
          has(c.resources.limits) && has(c.resources.limits.memory))
      message: "CPU and memory requests, and a memory limit, are required on every container."
      # An equivalent ClusterPolicy form exists but is deprecated — see Kyverno docs.
```

## 18.2 Runtime security and image supply chain

- **Scan images in CI** (trivy, grype) and **in-cluster** (Trivy Operator, which continuously scans running workloads and reports vulnerabilities as CRDs).
- **Sign and verify images** — **Sigstore cosign** + **Kyverno/Policy Controller** to *enforce* that only signed images from your registry run. This blocks the "attacker pushes a malicious `latest`" path.
- **Software bill of materials (SBOM)** with syft, attached to the image. Increasingly a compliance requirement.
- **Runtime threat detection** — **Falco** (eBPF-based syscall monitoring: shell spawned in a container, unexpected outbound connection, write to `/etc`), **Tetragon** (Cilium's eBPF security observability, can also *enforce*).
- **Distroless or minimal images** (no shell, no package manager) — shrinks attack surface dramatically and makes `kubectl exec` attacks useless.
- **Read-only root filesystem** + explicit `emptyDir` mounts for `/tmp` — makes container compromise non-persistent.
- **Keep the cluster patched.** Most real-world Kubernetes compromises exploit a *known, patched* vulnerability in an unpatched cluster (or an exposed, unauthenticated component). Patch cadence belongs in your calendar, not your intentions.

## 18.3 Network security

- **NetworkPolicy default-deny** in every app namespace (Part 9.5). This is the single biggest reduction in blast radius you can get — and remember the proxy interaction in Part 11.7.
- **Ingress with TLS only**, HSTS, and no plaintext listeners for production hostnames.
- **WAF / rate limiting** at the edge (cloud WAF, or Gateway API policy CRDs for rate limits and JWT validation). Gateway API implementations expose `SecurityPolicy`/`ClientTrafficPolicy`-style CRDs for CORS, JWT, basic auth, and IP allowlists — this is where the "typed policy" advantage over Ingress annotations pays off.
- **Egress control** — default-deny egress, plus FQDN allowlisting or an egress gateway, to stop exfiltration and C2 callbacks (Part 11.9).
- **Separate internal and external gateways** — never expose an admin UI through the same Gateway as public traffic. `allowedRoutes` and hostname scoping enforce this.
- **Lock down the debugging surface.** `pods/exec`, `pods/portforward`, and `pods/attach` are effectively production access that bypasses every control you built. Use PSA, RBAC scoping, and admission policies to restrict them, and audit their use. This is an area the community has explicitly flagged as under-secured.

## 18.4 Cluster-level and infra security

- **API server exposure.** In production the apiserver should not be directly internet-reachable. Put it behind a private endpoint (managed clusters: "private cluster" mode), a VPN, or a bastion with strong auth. Public apiservers get scanned and attacked continuously.
- **Audit logging** — enable apiserver audit logs and ship them off-cluster. When something goes wrong, "who did what, when" is the question you need answered.
- **Node access** — treat nodes as compromised-adjacent. No SSH keys for everyone, immutable OS (Talos/Bottlerocket/Flatcar) where possible, no secrets on node filesystems.
- **Cloud IAM mapping** — one IAM role per ServiceAccount via IRSA/Workload Identity, least privilege, and **never** a wildcard `*` policy. An over-permissive pod identity is a full cloud-account breach.
- **etcd** — encrypted at rest, TLS between members, backups encrypted, access strictly limited. Etcd access is cluster-admin access.
- **Compliance baselines** — run **kube-bench** (the CIS — Center for Internet Security — Benchmark) and **Kubescape** regularly; they will find things (anonymous auth, permissive RBAC, missing audit policy) that you did not think about.
- **Secrets encryption + external secret management** (Part 8). Not optional in production.
- **Supply chain for manifests too** — Argo CD 3.5 added internal mTLS and source integrity verification. If someone can commit to your GitOps repo, they can deploy to production; treat that repo's permissions accordingly (branch protection, signed commits, required reviews).

**Platform-level security mechanisms** — seccomp/AppArmor/SELinux configuration, ServiceAccount token handling, Pod Certificates, audit log policy, and the runtime-security baseline — are covered in [Part 45](#part-45--platform-security-features).

## 18.5 A realistic sequencing

Don't try to do all of this at once. A defensible order for an empty but real cluster:

1. **Identity + RBAC** (OIDC, no shared admin kubeconfig, groups not users).
2. **Secrets**: encryption at rest + external secret manager.
3. **PSA `baseline` enforced, `restricted` in warn mode**, plus 2–3 policies in `Audit`.
4. **NetworkPolicy default-deny ingress** per app namespace (with gateway allowances pre-added).
5. **Image scanning + signing enforcement**, and pin all images by digest.
6. **Audit logs shipped**, plus basic anomaly alerting.
7. **WAF + TLS-only at the edge**, internal/external gateway split, debugging verbs restricted.
8. **CIS benchmark pass**, then tighten to `restricted` enforcement and default-deny egress.

---

# Part 19 — Observability

You cannot operate what you cannot see. Observability in Kubernetes has three legs — **metrics, logs, traces** — plus the thing that actually makes them useful: **labels that correlate them**.

## 19.1 Metrics

**kube-state-metrics** (object state: replicas available, PVC phase, deployment generation mismatch) + **node-exporter** (host CPU/memory/disk/network) + **cAdvisor/kubelet metrics** (per-container usage, plus **PSI metrics** — pressure stall information — which are GA as of 1.36 and are the best signal for "is this node actually under resource pressure") + **your app's metrics**.

**Prometheus** is the de-facto standard, usually deployed via **kube-prometheus-stack** (Helm chart bundling Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, and a sane default rule set). For scale, use **Prometheus Operator** with `ServiceMonitor`/`PodMonitor` CRDs, plus **Thanos**, **Mimir**, or **VictoriaMetrics** for long-term storage and multi-cluster querying. **OpenTelemetry Collector** is increasingly used as the ingestion layer for all three signal types.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: api, namespace: prod, labels: { release: kube-prometheus-stack } }
spec:
  selector: { matchLabels: { app.kubernetes.io/name: api } }
  endpoints:
    - { port: http, path: /metrics, interval: 30s }
```

**The USE and RED method, mapped to Kubernetes:**

- **RED** (per service): **R**ate of requests, **E**rror rate, **D**uration (latency percentiles — p50/p95/p99, never averages).
- **USE** (per resource): **U**tilization, **S**aturation, **E**rrors — for nodes, CPU, memory, disk I/O, network.

**Crucially, measure at the reverse proxy too.** Proxy metrics (`upstream_rq_time`, `upstream_cx_connect_fail`, 4xx/5xx by route, request rate per route) tell you whether latency or errors originate at the edge or in the backend — answering in seconds what would otherwise take an hour. If your proxy isn't emitting metrics, that's a gap to close immediately.

**The alerts that actually matter on day one:**

| Alert | Why |
|---|---|
| Pod in `CrashLoopBackOff` (a *waiting reason*, not a restart count) | The app is crash-looping. Alert on `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}` — a raw restart-rate alert fires on healthy rolling updates and `rollout restart`, and *misses* a pod that crashed once and stayed down. |
| Deployment `availableReplicas < desired` for >5m | A rollout is stuck or capacity is short. |
| Node `NotReady` | Capacity loss, possibly imminent eviction storm. |
| PVC usage > 85% | Disk full is a hard, fast, total outage. |
| Certificate expires < 21 days | Silent expiry is avoidable embarrassment. |
| Proxy 5xx rate or upstream connect failures rising | The edge is telling you something the app can't. |
| Proxy retry rate elevated | Retry storms precede outages. |
| Rising `container_memory_working_set_bytes` against the limit | The actual early warning of an OOM leak. The restart counter is *reason-agnostic* and only tells you after the kill — pair it with `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` to confirm the cause. Note `kubectl top` samples point-in-time and will miss the pre-OOM peak. |
| API server latency / etcd fsync duration | The control plane is the thing that must not degrade. |
| Job/CronJob failed | Silent data-pipeline failure. |
| cert-manager / Argo CD / gateway controller unavailable | The platform layer is down. |
| HPA at max replicas | You are out of headroom *before* the incident. |
| Certificate/secret sync failures (ESO) | Rotation is broken; expiry is coming. |

**SLOs, briefly:** define availability and latency SLOs per user-facing service, alert on **error-budget burn rate** rather than raw thresholds, and you stop paging on noise. This is the difference between monitoring and operating.

## 19.2 Logs

The cardinal rule: **containers should log to stdout/stderr, not to files.** The kubelet captures those two streams, which is what makes `kubectl logs` and every log collector work. What is written to a file inside the container is *outside that pipeline*: the kubelet does not read it, so it never reaches `kubectl logs` or your aggregator, and it disappears with the container unless it sits on a persistent volume.

To be precise rather than absolute, file logs are not unreachable while the container lives — `kubectl exec` or `kubectl cp` can read them, and a sidecar tailing a shared `emptyDir` is a legitimate pattern for legacy applications that cannot be changed. But both are workarounds. The rule is about *which pipeline sees the data*, and the default pipeline only sees stdout/stderr.

The standard stack: **Fluent Bit** (or **Vector**, or **Grafana Alloy**) as a DaemonSet → an aggregator/buffer → **Loki** (cheap, label-indexed, Grafana-native), **Elasticsearch/OpenSearch**, or a SaaS (software-as-a-service, e.g. Datadog, Grafana Cloud, Better Stack).

Practical concerns:
- **Structured JSON logs** with request IDs and trace IDs, so you can correlate across services. **Include the request ID the proxy generated** — that's what lets you follow a request from the edge to the app.
- **A log-level strategy** — `info` in prod, `debug` toggleable without redeploy (via ConfigMap + reload).
- **Cardinality control.** Loki indexes labels; putting a user ID or request ID in a *label* (rather than in the log body) will destroy the index. Labels: namespace, pod, container, app. Everything else belongs in the line.
- **Retention and cost** — logs are the fastest-growing and least-read signal. Set retention deliberately (e.g. 30d hot, 1y cold for audit).
- **Audit logs and application logs are different streams** with different retention and access requirements.
- **Proxy access logs are application logs.** Ship them the same way and keep the fields consistent (method, path, status, latency, upstream, request ID) so you can join them.

## 19.3 Traces

**OpenTelemetry** is the vendor-neutral standard: instrument once, export anywhere (Jaeger, Tempo, Datadog, Honeycomb). Distributed tracing answers the question the other two signals can't: *where in the call chain did this request spend 2 seconds?* Adopt it once you have more than a handful of services; below that, good metrics plus correlated logs are enough.

**Instrument with the OTel SDK or auto-instrumentation (eBPF/agent-based), send to an OTel Collector (as a sidecar, Deployment, or DaemonSet), and propagate `traceparent` headers** through every hop, including your gateway. A trace that stops at the gateway tells you nothing — configure the proxy to propagate trace context, and ideally to generate spans for upstream calls.

## 19.4 Profiling and continuous profiling

**Pyroscope / Grafana Profiles / Parca** — continuous CPU/memory profiling with labels, letting you answer "which function got slower after last Tuesday's deploy?" This is now mainstream and finds things metrics never will.

## 19.5 Golden rule of observability hygiene

**Every signal must carry consistent labels** — `namespace`, `app.kubernetes.io/name`, `app.kubernetes.io/version`, `cluster`, `region`. If labels don't line up between metrics, logs, and traces, you cannot pivot between them, and you end up grepping. This is why Part 5.2 insists on the common-label convention from day one.

---

# Part 20 — Debugging

Debugging Kubernetes is a *method*, not a pile of commands. The method is: **find the layer where desired state and observed state diverge, then look at that layer's events and status.**

## 20.1 The universal first three commands

```bash
kubectl get pods -n <ns> -o wide        # what state are things in?
kubectl describe pod <pod> -n <ns>      # events + spec + conditions (the money command)
kubectl logs <pod> -n <ns> [--previous] [-c <container>]
```

Then, depending on what you see:

```bash
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -30
kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp | tail -30
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl get deploy,rs,pod,svc,endpoints,ingress,httproute -n <ns>
kubectl get pod <pod> -o yaml | less
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState}'
```

`kubectl describe` is underrated. It shows the **Events** section (image pull failures, scheduling failures, probe failures, OOMKills, volume mount errors) and the **Conditions**, which usually names the problem outright.

## 20.2 Pod state → cause map

| Symptom | Meaning | Go look at |
|---|---|---|
| `Pending` | Not scheduled | `describe pod` → Events. Insufficient CPU/memory, node selector/affinity unsatisfiable, taints without toleration, unbound PVC, too many pods per node. |
| `ContainerCreating` (stuck) | Sandbox/volume/network setup | Events: volume mount failure, CNI IP exhaustion, image pull slow, secret/configmap missing. |
| `ImagePullBackOff` / `ErrImagePull` | Can't fetch image | Wrong tag, private registry without `imagePullSecrets`, registry rate limit, node can't reach registry. |
| `CrashLoopBackOff` | Container starts and exits repeatedly | `logs --previous` (the *crashed* instance), wrong command, missing config, app bug, failed migration, missing secret env var. |
| `OOMKilled` (in lastState) | Exceeded memory limit | Raise the limit, or fix the leak. Look at actual usage, not the limit. |
| `Error` / non-zero exit | App exited with failure | Logs, exit code. |
| `Running` but not `Ready` | Readiness probe failing | `describe` → probe events; check the probe path/port, and whether the app actually serves it. |
| `Terminating` (stuck) | Finalizer or a node that's gone | Finalizers, `kubectl get pod -o yaml \| grep finalizers`, node status. |
| `Evicted` | Node pressure | Node conditions, requests/limits, QoS class. |
| `Completed` then gone | Job finished; `ttlSecondsAfterFinished` deleted it | Expected. |
| `Init:Error` / `Init:CrashLoopBackOff` | Init container (or native sidecar) failing | `logs -c <init-container>`. |
| `CreateContainerConfigError` | Missing ConfigMap/Secret key | `describe`: "secret not found" / "couldn't find key". |

## 20.3 Service and networking debugging

**The dominant rule: connectivity problems are usually endpoint problems.** If a Service has no endpoints, nothing downstream can work, no matter what the gateway or proxy says.

```bash
# 1. Does the Service exist and select anything?
kubectl get svc api -n prod -o yaml | grep -A5 selector
kubectl get endpointslices -n prod -l kubernetes.io/service-name=api
kubectl get endpointslices -n prod -l kubernetes.io/service-name=api   # no addresses = the problem

# 2. Do pod labels actually match the selector?
kubectl get pods -n prod --show-labels
kubectl get pods -n prod -l app.kubernetes.io/name=api     # zero results = mismatch

# 3. Is the pod Ready? (not Ready = not in endpoints)
kubectl get pods -n prod -o wide

# 4. Is the port right? (containerPort / targetPort / Service port)
kubectl get svc api -o jsonpath='{.spec.ports}'

# 5. Test from inside the cluster
kubectl run tmp --rm -it --image=nicolaka/netshoot --restart=Never -- \
  curl -sv http://api.prod.svc.cluster.local:80/healthz

# 6. DNS specifically
kubectl run tmp --rm -it --image=nicolaka/netshoot --restart=Never -- \
  nslookup api.prod.svc.cluster.local

# 7. Port-forward to bypass Service routing AND the proxy entirely
kubectl port-forward -n prod pod/api-xyz 8080:8080
#    works via port-forward but not via Service → Service/endpoint/selector problem,
#      OR a NetworkPolicy drop (port-forward bypasses policy — see the note above)
#    works via Service but not via hostname  → proxy/gateway/LB/DNS/TLS problem
#    fails both ways → the app, the pod, or your portforward RBAC/kubelet health
```

**That three-way split is the single most valuable debugging test in Kubernetes.** It divides the entire search space into thirds in about 15 seconds.

**Gateway / reverse-proxy routing checklist, in order:**

```bash
kubectl get gateway -A                     # ACCEPTED + PROGRAMMED True? ADDRESS assigned?
kubectl describe gateway main-gateway -n gateway-system
kubectl get httproute -A                   # status.parents: Accepted / ResolvedRefs
kubectl describe httproute api-route -n prod
kubectl get svc -n gateway-system          # data plane service has an external IP?
kubectl get pods -n <gateway-controller-ns>  # controller healthy?
# proxy's own view (Envoy-based):
kubectl get deploy -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=main-gateway   # real name
kubectl exec -n envoy-gateway-system deploy/<name> -- curl -s localhost:19000/clusters | head
# then, from OUTSIDE:
curl -sv https://api.example.com/v1 --resolve api.example.com:443:<GATEWAY_IP>
```

Common Gateway API failures and their tells:
- **`ResolvedRefs: False, reason: BackendNotFound`** — the Service name/port in `backendRefs` is wrong or missing.
- **`Accepted: False, reason: NotAllowedByListeners`** — the route's namespace isn't allowed by `allowedRoutes`, or the hostname doesn't overlap the listener's.
- **`Accepted: False, reason: NoMatchingParent`** — `parentRefs` name/namespace wrong.
- **`RefNotPermitted`** — cross-namespace backend without a `ReferenceGrant`.
- **Gateway `Programmed: False`, no address** — no LoadBalancer implementation (bare metal without MetalLB) or the controller isn't running.
- **Everything green but 404** — your `hostnames` don't match the requested Host header, or path match type (`Exact` vs `PathPrefix`) is stricter than you think.
- **TLS handshake failure** — the certificate Secret is missing, in the wrong namespace, or the listener's `hostname` doesn't match the cert's SANs.

**NetworkPolicy debugging** is special because a blocked packet usually produces a *timeout*, not a refusal — which looks like "the service is down."

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy -n prod
# Cilium:
cilium status
hubble observe --namespace prod --verdict DROPPED --last 50
# Generic: temporarily add an allow-all policy to confirm the policy is the cause
```

## 20.4 Node and resource debugging

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -20
kubectl describe node <node>                 # Conditions, Allocated resources, Events
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
journalctl -u kubelet -n 200 --no-pager      # on the node
```

- **`Allocated resources` in `describe node`** shows the sum of pod requests vs capacity. If it's at 95%, you are one deploy away from Pending pods — and this is the #1 cause of "the scheduler won't place my pod."
- **Disk pressure → evictions.** Check `df -h`, image garbage collection, and log volume. Container images accumulate; nodes that never prune fill up eventually.
- **IP exhaustion in the CNI** (especially AWS VPC CNI, where each pod consumes an ENI IP): pods go `Pending` with `failed to assign an IP`. Fix by enabling prefix delegation or raising ENI limits.

## 20.5 Debugging tools that earn their place

- **`k9s`** — terminal UI; the fastest way to browse a cluster, read logs, and exec. Learn it.
- **`stern`** — multi-pod log tailing with label selectors and color: `stern -n prod -l app=api --since 10m`.
- **`kubectl-neat`** — strips managedFields/status noise from YAML output.
- **`netshoot`** image — a container with `dig`, `curl`, `tcpdump`, `nmap`, `traceroute`, `mtr`. The Swiss army knife for network debugging.
- **`kubectl debug`** — ephemeral containers for distroless pods, and `node/` debugging.
- **`bpftrace` / Tetragon** — kernel-level tracing when you need to see syscalls. (`kubectl trace` is an older iovisor project, not part of the Cilium/Tetragon ecosystem.)
- **`kubectl diff`** — see exactly what an apply would change before it changes it.
- **`popeye`** — cluster hygiene/sanity scanning (dangling services, missing probes, unused configmaps, empty endpoints). Cheap CI quality gate.
- **Proxy admin endpoints** — Envoy `config_dump`/`clusters`/`stats`, NGINX `nginx -T`, Traefik healthcheck. Learn these; they answer proxy questions definitively.

**When the cluster itself feels slow rather than a workload being broken**, the problem is likely control-plane overload — see [Part 40.4](#404-control-plane-capacity-and-overload) and [Part 41](#part-41--api-priority-and-fairness).

## 20.6 The mental discipline

When something is broken, work **outward from the API server**, and at each step ask *"is the object what I think it is, and is its status what I think it is?"*:

```
Is the object accepted?        → kubectl get/describe, Events, admission webhook rejections
Is it scheduled?               → pod.spec.nodeName, scheduler Events
Is it running?                 → containerStatuses, logs, probe results
Is it reachable?               → EndpointSlices, DNS, NetworkPolicy
Is it routing?                 → Service/Route status, gateway status, proxy config dump, external curl
Is it performing?              → metrics (app AND proxy), traces, HPA state, node pressure
```

Two habits make this dramatically faster:
1. **Read `status` and `Events` before touching anything.** 90% of incidents announce their cause there.
2. **Change one thing at a time and observe.** Kubernetes is a distributed system; a shotgun of fixes produces a state you can no longer reason about.

---

# Part 21 — GitOps

## 21.1 What GitOps is, and why it's the right default

**GitOps is: the desired state of the cluster lives in a git repository, and a controller in the cluster continuously makes reality match the repository.** Git is the single source of truth. Everyone reads state from git. Nobody `kubectl apply`s to production by hand.

Why this is not just "CI that deploys":

- **Declarative + reconciled** — you get drift detection for free. If someone `kubectl edit`s a Deployment at 2am, the controller notices and reverts it (or reports it, depending on config). Hand-edited clusters are impossible to reason about; eventually nobody knows what's actually deployed.
- **Auditable** — `git log` is your deployment history with authors, reviewers, and diffs. "What changed before the incident?" has a real answer.
- **Reviewable** — infrastructure changes go through pull requests, exactly like code. A platform team can require review on prod changes.
- **Recoverable** — disaster recovery becomes "point a new cluster at the repo." This is the single strongest argument, and it only works if *everything* is in git.
- **Rollback = git revert** — and the controller converges to it. No imperative "undo" scripts.

There are two flavors, and you should know the difference:

- **Push-based GitOps** — CI runs `kubectl apply`/`helm upgrade` with cluster credentials. Simple, but the credentials live in CI, there's no drift detection, and the cluster's true state is unknown.
- **Pull-based GitOps** — an agent *in the cluster* pulls from git and applies. No cluster credentials leave the cluster, drift is continuously corrected, and the cluster always reports its own state. **This is what "GitOps" means today.**

## 21.2 The two main implementations, honestly compared

| | **Argo CD** | **Flux CD** |
|---|---|---|
| Model | Application CRD (a bundle of resources) | Toolkit of controllers (source, kustomize, helm, notification) |
| UI | Rich web UI (genuinely useful for "what's deployed where, is it healthy") | CLI-first (Weave GitOps provides a UI) |
| Multi-tenancy | Projects + RBAC | Strong native multi-tenancy, per-namespace |
| Templating | Native Helm, Kustomize, Jsonnet, plugins | Native Helm, Kustomize; no plugins |
| Progressive delivery | Argo Rollouts (tight integration) | Flagger |
| Secrets | External Secrets integration | SOPS built in as a first-class citizen |
| Learning curve | Lower (the UI teaches you) | Lower once you know the model; more composable |
| Ecosystem | Very large, CNCF graduated | Very large, CNCF graduated |

**Which to pick:** both are excellent and this is genuinely close. Argo CD is the more common first choice because the UI shortens onboarding and `Application` bundles map well to "an app in an environment." Flux appeals to teams that want composability, no UI, and SOPS-native secrets. **Pick one and don't run both for the same resources** — two reconcilers fighting over the same object is a uniquely miserable failure mode.

**Other options:** **Jenkins X** (opinionated, less common now), **Rancher Fleet** (good for very large cluster fleets), **Argo CD + ApplicationSet** (which generates Applications from a template) for fan-out, and **Kubernetes-native CI** tools like **Tekton** if you want your pipelines in-cluster too.

## 21.3 Installing Argo CD on the empty cluster

```bash
kubectl create namespace argocd
# Pin the version — `stable` is a moving branch. Match it to a supported release.
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.0/manifests/install.yaml

# The default admin password (CHANGE IT / replace with SSO immediately)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d

kubectl -n argocd port-forward svc/argocd-server 8080:443
```

**In production, use the Helm chart** with:
- **HA mode** (`redis-ha`, 3 replicas of each component).
- **Ingress/Gateway** exposure — and put it **behind SSO and not on the public internet**, or restrict by IP/VPN. An Argo CD with cluster-admin and a weak password is a total cluster compromise.
- **SSO via OIDC** and **RBAC** mapping IdP groups to Argo CD roles (`policy.csv`).
- **Secrets via External Secrets**, not the chart's default secret handling.
- **Source integrity verification** (Argo CD 3.5) — but understand precisely what it does, because it is easy to over-trust. It performs **GnuPG verification of Git commit and tag signatures** (`AppProject.spec.sourceIntegrity.git.policies`), confirming *who signed the commits* your target revision resolves to. It does **not** verify the manifest content those commits produce: a commit signed by a trusted key that happens to contain malicious YAML passes verification. It is also **Git-only** — it cannot be used with Helm or OCI application sources. So it authenticates the *revision*, not the *rendered output*, and it complements branch protection and required review rather than replacing them.

## 21.4 The repository structure that actually works

Two repos (or two directories), and this separation is important:

**1. The application repo** — code, Dockerfile, CI pipeline. CI builds and pushes an image tagged with the commit SHA.

**2. The deployment/config repo (the "GitOps repo")** — all Kubernetes manifests/Helm values/kustomize overlays, describing what should run in each environment.

```
k8s-config/
├── bootstrap/                 # Argo CD itself + app-of-apps root (chicken-and-egg)
├── infrastructure/            # platform add-ons, each an Argo Application
│   ├── cilium/
│   ├── cert-manager/
│   ├── external-secrets/
│   ├── envoy-gateway/
│   ├── kube-prometheus-stack/
│   └── metallb/
├── apps/
│   ├── base/api/              # environment-agnostic manifests
│   └── overlays/
│       ├── dev/api/
│       ├── staging/api/
│       └── prod/api/          # image tag, replicas, resources, hostnames
└── clusters/
    ├── dev/
    ├── staging/
    └── prod/
```

**The app-of-apps pattern** is how you avoid manually creating an Argo Application per service forever: one root `Application` points at a directory of `Application` manifests, and Argo CD creates them all. The root app is the only thing you apply by hand, once, at bootstrap.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/k8s-config
    targetRevision: main
    path: clusters/prod          # a directory of Application manifests, NOT the app overlays
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

**Note the self-reference caveat:** Argo CD must not manage its own namespace in a way that lets a bad commit brick the controller. Keep Argo CD's own manifests in a separately-bootstrapped repo/path, and be careful with `prune` on anything that contains the thing doing the pruning.

**`ApplicationSet`** is the scaling mechanism: generate Applications from a matrix (clusters × apps), a git directory generator, or a **pull-request generator** (which is how you get per-PR preview environments for free — see Part 22.5).

**`prune: true` and `selfHeal: true` deserve respect:** `prune` deletes resources removed from git (great, but it will delete a resource someone added manually — often the intent). `selfHeal` reverts manual changes, which ends the era of hotfixing prod by hand. **Be deliberate about enabling these in production**, ideally after your team has adapted and after you have alerting on sync failures.

**Ignore-differences matter.** HPA-managed `replicas`, admission-webhook-injected fields, and defaulted fields cause permanent "OutOfSync" noise. Configure `ignoreDifferences` for those fields explicitly (see Part 25.1 for the HPA case, which bites everyone).

## 21.5 How a deployment actually flows

```
dev pushes commit → CI (tests, build) → push image :sha-abc123
     → CI opens a PR against k8s-config, updating prod/api/kustomization.yaml image tag
     → PR reviewed & merged (or auto-merged for dev)
     → Argo CD detects the new commit (webhook or ~3min polling)
     → Argo CD diffs desired (git) vs live (cluster)
     → Argo CD syncs: applies manifests; PreSync hooks run migrations
     → Deployment rolls out; Argo CD reports Progressing → Healthy
     → if unhealthy, Argo CD alerts (and optionally auto-rolls-back)
```

**This is the whole workflow, and the property that makes it valuable is that the deploy step has no cluster credentials in CI and is fully reproducible from git history.**

**Sync waves and hooks** order the apply:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"     # lower = earlier (CRDs, namespaces first)
    argocd.argoproj.io/hook: PreSync       # PreSync | Sync | PostSync | SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

Typical ordering: CRDs and namespaces at wave `-5`, secrets/operators at `-3`, the database migration Job as a `PreSync` hook, then the app Deployment at wave `0`.

**What you must NOT put in the Argo-managed repo:** plaintext secrets (use External Secrets), TLS private keys, and anything environment-specific that differs per cluster but isn't parameterized.

**Managing multiple clusters with one Argo CD:** register clusters (`argocd cluster add`), then each Application's `destination.server` targets one. Two patterns: **hub-and-spoke** (one Argo CD controls all clusters — simple, but the hub is a single point of failure and needs network access to every cluster's API), or **Argo CD per cluster** (installed via Argo CD itself, better isolation, more setups to update). Most teams start hub-and-spoke for dev/staging and go per-cluster for production.

## 21.6 The uncomfortable parts of GitOps

- **It's slower to hotfix.** That's the point, but you need an explicit break-glass procedure (documented, audited, followed by a reconciliation to git) or people will route around the system.
- **Git repo structure becomes load-bearing.** A bad directory layout causes sync loops, duplicated resources, and ownership confusion (two Applications fighting over the same object). Decide ownership boundaries early — one object, one owner.
- **`prune` on a wide-open Application can delete things you meant to keep.** Scope Applications narrowly; use `syncOptions: [PruneLast=true]` where appropriate.
- **Secrets still need a story** (Part 8) — GitOps doesn't solve secrets, it just makes their absence from git mandatory.
- **Controller outages stall deploys.** Alert on Argo CD's own health, and know how to sync manually with the CLI in an emergency.
- **It does not replace testing.** GitOps deploys reliably; it does not make wrong changes right. Progressive delivery is the complement.
- **Repo access = production access.** Branch protection, required reviews, and signed commits on the GitOps repo are production security controls, not developer conveniences.

---

# Part 22 — CI/CD: The Actual Workflow

## 22.1 The end-to-end flow, with the boundaries marked

```
┌─────────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────────────────┐
│ Developer   │──▶│ CI pipeline  │──▶│ Image registry│──▶│ GitOps repo      │
│ git push/PR │   │ test+build   │   │ :sha-abc123   │   │ (manifest update)│
└─────────────┘   └──────────────┘   └───────────────┘   └────────┬─────────┘
                                                                  │ pull + diff
                                                                  ▼
                                                          ┌──────────────────┐
                                                          │ Argo CD (in-cluster)│
                                                          │ sync → rollout   │
                                                          └──────────────────┘
```

The critical boundary: **CI builds an artifact and updates a manifest. CD (Argo CD) applies the manifest.** CI never needs cluster credentials. That separation is the security and reliability win.

## 22.2 What CI must do

1. **Lint and test** the application.
2. **Build the image** and push it tagged with the **immutable git SHA** (`registry/app:abc1234`) — never `latest`. Immutability is what makes rollback and auditing possible.
3. **Scan the image** (trivy/grype) and fail on critical CVEs (published known vulnerabilities).
4. **Generate an SBOM** and **sign** the image (cosign).
5. **Update the GitOps repo** — bump the image tag in the target environment's overlay, and open a PR (or commit directly for dev).
6. Optionally: run integration tests against an **ephemeral preview environment** (Part 22.5).

## 22.3 Deployment strategies

| Strategy | How | When |
|---|---|---|
| **RollingUpdate** | Deployment default; surge/unavailable control | Most services. |
| **Recreate** | Kill all, then start | Single-replica, can't-run-twice apps; accept downtime. |
| **Blue/Green** | Two full environments, switch Service selector | Need instant rollback and full pre-cutover verification; 2× resources. |
| **Canary** | Route a % of traffic to the new version (Gateway API `weight`) | Risk-managed releases on high-traffic services. |
| **Progressive delivery + analysis** — releasing to a small slice of traffic, verifying against metrics, then promoting or rolling back automatically | Argo Rollouts / Flagger automate the canary with metric gates and auto-rollback | Mature teams with good metrics. |

**Gateway API canary, concretely** — this is where the traffic-splitting field earns its keep, and where the reverse proxy from Part 11 does the real work:

```yaml
spec:
  rules:
    - backendRefs:
        - { name: api-v1, port: 80, weight: 95 }
        - { name: api-v2, port: 80, weight: 5 }     # 5% to canary
```

Roll the weights 5 → 25 → 50 → 100 while watching error rate and p99 latency **at the proxy**, then delete the old Service. No DNS changes, no new load balancer. Combine with **request mirroring** to shadow production traffic at a new version before sending it real users.

**Argo Rollouts** adds automated analysis: define success criteria (error rate < 0.5%, p99 < 500ms) and the rollout promotes or rolls back automatically. This is the mature end state of progressive delivery.

**Database migrations in this flow:** always **backward-compatible, expand/contract:**
1. Deploy schema change that adds (new column nullable, new table) — old and new code both work.
2. Deploy code that writes both / reads new.
3. Backfill.
4. Remove the old column in a *later* release.

A migration that breaks the previous version makes rollback impossible — which is exactly when you'll need it. Run migrations as a `PreSync` hook or a Job, gated on success, and make them **idempotent**.

## 22.4 Environment promotion

Promotion should be **a git commit that changes a tag in a directory**, not a copy-paste of manifests:

```yaml
# apps/overlays/dev/api/kustomization.yaml
images: [{ name: registry/app, newTag: abc1234 }]
```
Promote to staging = PR that sets staging's tag to the same `abc1234`, after dev has been verified. Because the same image digest moves through environments, you are testing the *exact artifact* that will run in prod — a property that "rebuild per environment" destroys.

## 22.5 Ephemeral/preview environments

Per-PR namespaces (created by an `ApplicationSet` with a pull-request generator, deleted on merge) let reviewers click a real running version. They are cheap, they catch integration bugs early, and they force you to make your manifests truly parameterized (names, hostnames, resource sizes). This is one of the highest-value practices in modern Kubernetes work, and it is only feasible if environments are template-driven — another reason for Kustomize/Helm from the start.

**You need a wildcard DNS record and wildcard TLS cert** for preview hostnames (`pr-1234.preview.example.com`), which is exactly why DNS-01 cert issuance and external-dns matter.

---

# Part 23 — Namespaces and Multi-Tenancy

## 23.1 What namespaces are actually for

A **namespace** is a scope for names and a unit of policy. It is *not* a security boundary, and *not* a network boundary (until you add NetworkPolicy), and *not* a resource boundary (until you add ResourceQuota).

What namespaces actually do:

1. **Name scoping** — you can have a `postgres` Service in `dev` and one in `prod` without conflict. Most objects are namespaced; some (Node, PersistentVolume, StorageClass, ClusterRole, GatewayClass, CRD) are cluster-scoped.
2. **RBAC scoping** — RoleBindings grant permissions *within* a namespace. This is the primary mechanism for team isolation.
3. **Resource quotas** — `ResourceQuota` caps total CPU/memory/object counts per namespace. `LimitRange` sets default requests/limits for pods that don't specify them. Together they stop one team from eating the cluster.
4. **Policy scoping** — NetworkPolicy, Pod Security Admission labels, and most policy engines operate per namespace.
5. **Operational grouping** — a namespace is the natural blast-radius unit for `kubectl get all -n X`, for deleting a whole tenant's stuff, and for metrics/logs dimensions.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-a-quota, namespace: team-a }
spec:
  hard:
    requests.cpu: "40"
    requests.memory: 80Gi
    limits.memory: 160Gi
    persistentvolumeclaims: "20"
    requests.storage: 500Gi
    count/deployments.apps: "50"
    count/services.loadbalancers: "2"       # cap cloud LB spend!
---
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: team-a }
spec:
  limits:
    - type: Container
      default:        { cpu: 500m, memory: 512Mi }   # applied if no limit set
      defaultRequest: { cpu: 100m, memory: 128Mi }   # applied if no request set
      max:            { cpu: "8",  memory: 16Gi }
```

**Important caveat:** once a ResourceQuota defines `requests.cpu`, **every** pod in that namespace must declare a CPU request, or it is rejected. LimitRange defaults make that work. Set both together, and communicate it, or you'll break every existing manifest at once.

## 23.2 Namespace design patterns

**By environment:** `dev`, `staging`, `prod` — the minimum. Even if you use separate clusters, namespaces mirror them.

**By team or domain:** `team-payments`, `team-search`, `platform`, `observability`.

**By function within a cluster:**
- `kube-system` — Kubernetes' own components. **Never** deploy your apps here; messing it up takes down the cluster.
- `kube-public`, `kube-node-lease` — special, leave alone.
- `gateway-system` (or `envoy-gateway-system`, `istio-system`) — shared gateway/proxy.
- `cert-manager`, `external-secrets`, `monitoring`, `logging`, `velero` — platform controllers.
- `argocd` — GitOps.
- `data` — self-hosted databases.
- Application namespaces.

**A workable convention** for a 20–200 engineer org:

```
kube-system, gateway-system, cert-manager, monitoring, logging, argocd, external-secrets   ← platform
team-a-dev, team-a-staging, team-a-prod                                                     ← tenant × env
data-dev, data-prod                                                                         ← stateful
sandbox-<username>                                                                          ← scratch, quota'd, optionally auto-deleted
```

**Labels to set on every namespace** (they are the hooks for policy and selection):
```yaml
labels:
  kubernetes.io/metadata.name: team-a-prod     # set automatically
  environment: production
  team: team-a
  cost-center: "1234"
  pod-security.kubernetes.io/enforce: restricted
  shared-gateway-access: "true"                # if it may attach routes to the shared Gateway
```

## 23.3 Is one namespace enough isolation? No.

Namespaces alone give you naming and RBAC. To get *real* multi-tenant isolation you need, at minimum: RBAC per namespace, `ResourceQuota` + `LimitRange`, NetworkPolicy default-deny, Pod Security Admission `restricted`, and admission policies blocking the escape hatches:

- **hostPath / hostNetwork / hostPID** — node escape. PSA baseline blocks these.
- **privileged containers** — root on node. Blocked by baseline.
- **`create pods` is effectively node-admin in a namespace, via a specific chain.** It is not that pod creation itself grants impersonation — `impersonate` is a separate verb, and so is `create serviceaccounts/token`. The real chain is: create a pod whose `serviceAccountName` names a privileged ServiceAccount → the kubelet mounts a short-lived **bound token** for that SA into the pod → `exec` into the pod (or read the projected file) and you now hold that SA's API permissions *and* its cloud identity if the SA is bound to an IRSA/Workload Identity role. Two preconditions make this work, and both are worth closing: the target SA must actually be bound to something, and you must be able to create pods in that namespace. Mitigate by scoping IRSA roles tightly, avoiding powerful default ServiceAccounts, and treating namespace-level pod creation as a privileged operation.
- **`nodes/proxy`** — the most dangerous verb most people have never heard of. Upstream warns that it "grants access to all other kubelet APIs," including APIs that **execute commands in any container running on the node** — and that `get` on it "is not a read-only permission." Granting it is equivalent to granting node-level command execution, which defeats namespace isolation entirely.
- **`pods/exec`, `pods/attach`, `pods/portforward`, and the ephemeral-containers subresource** — a shell or tunnel into any pod in the namespace. Note that `portforward` in particular bypasses NetworkPolicy, because the tunnel enters the pod from its own node.
- **`kubectl debug node/<node>`** — creates a privileged pod in the host's namespaces, which is the canonical way to convert pod-creation rights into node-admin. If you lock down the three verbs above and leave this open, you have closed nothing.
- **`impersonate`** and **`create serviceaccounts/token`** — the two verbs that grant identity theft *directly*, without the pod-creation chain described above.
- **`create persistentvolumes` / PV reclaim** — data access across tenants.
- **`escalate` (granting yourself permissions you do not hold) and `bind` (binding roles you do not hold) **on RBAC**, and creating `ClusterRoleBinding`s.
- **Reading `secrets`** — often cloud credentials.
- **CRDs that create other resources** (a `Certificate` is fine; a CRD that can create pods is not) — with great power comes great ability to break tenancy.

**If tenants are mutually untrusted → use separate clusters.** A namespace is a fence, not a vault. For hostile multi-tenancy (running other people's code), you want separate clusters, or stronger isolation technologies — **vCluster** (a virtual cluster inside a namespace), **Kata Containers** (a lightweight VM per pod), or **gVisor** (a user-space kernel that intercepts syscalls), or a managed offering with explicit tenancy guarantees. Most companies do not need this and over-invest in it, while under-investing in RBAC hygiene — which is the actual risk.

## 23.4 Namespace lifecycle

- Creating: let GitOps do it (`CreateNamespace=true`), with labels and quotas, in git. Never hand-create prod namespaces — you'll forget the quotas and policies.
- Deleting: **watch out.** Deleting a namespace deletes everything in it, asynchronously, and it can hang on finalizers. It does **not** delete cluster-scoped resources (CRDs, ClusterRoles, PVs) that belonged to it — those leak.
- **Cost of leak:** orphaned PVs from deleted namespaces are a real, recurring cloud bill surprise. Automate detection (`kubectl get pv | grep Released`).

---

# Part 24 — Environments: One Cluster or Many?

**Short answer: separate clusters for production, at minimum. Same-cluster environments are a dev/staging convenience, not a production pattern.**

## 24.1 The trade-off, honestly

**One cluster with `dev`/`staging`/`prod` namespaces:**

- ✅ Cheap. One control plane, one set of platform add-ons, one bill.
- ✅ Fast — no cluster provisioning to spin up an environment.
- ✅ Easy cross-environment testing.
- ❌ **Shared blast radius.** A cluster-wide failure (etcd corruption, CNI bug, node pool exhaustion, a bad cluster-scoped CRD/ClusterRole/webhook, apiserver overload, a runaway namespace eating all node capacity) takes down *all* environments, including prod.
- ❌ **A cluster upgrade is a production event** even if you're only testing dev.
- ❌ **Cluster-scoped resources are shared** — CRDs, webhooks, ClusterRoles, GatewayClasses, StorageClasses, PVs, nodes. One team's broken CRD or misconfigured webhook (especially a `failurePolicy: Fail` validating webhook that goes down) blocks *every* namespace. This is the most under-appreciated risk.
- ❌ **Security boundary is weak.** A pod-creation permission in dev can reach node-level privilege, and cluster-wide resources are reachable. "Dev can't touch prod" is hard to guarantee with namespaces alone.
- ❌ **Noisy neighbours.** A dev load test consumes the same node capacity as prod unless you use taints, node pools, and quotas rigorously.
- ❌ **Compliance.** Most auditors and compliance regimes (SOC 2, PCI DSS, HIPAA) want production isolated from non-production. If you're in scope, this decides it for you.
- ❌ **Different requirements.** Prod needs HA control plane, private API endpoint, backups, audit logs, Multi-AZ. Dev doesn't. One cluster forces prod-grade config everywhere (wasteful) or dev-grade (dangerous).

**Separate clusters per environment:**

- ✅ Real blast-radius isolation — this is the whole point.
- ✅ Independent upgrade cadence (upgrade dev, wait a week, upgrade prod) — the correct way to de-risk Kubernetes upgrades.
- ✅ Independent security posture, network boundaries, and IAM.
- ✅ Cleaner DR story and per-environment capacity/cost tuning.
- ✅ Forces your manifests to be **truly parameterized** — which is a healthy forcing function that reveals hidden coupling.
- ❌ More clusters to manage (patching, upgrades, add-on versions, cost).
- ❌ Requires automation from day one (Terraform/Crossplane/Cluster API + GitOps). Hand-built clusters × 3 is a maintenance disaster.
- ❌ Slightly slower to provision new environments (mitigated by ephemeral preview envs).

## 24.2 Recommendations by scale

| Situation | Recommendation |
|---|---|
| Solo/small team, pre-product | **One cluster**, `dev` + `prod` namespaces, node pools separated by taint, strict quotas. Accept the risk knowingly. |
| Small team, real users | **Two clusters**: `nonprod` (dev+staging namespaces) and `prod`. The minimum defensible split. |
| Growing team, real revenue | **Three+**: dev, staging, prod — separate clusters. Staging should be as prod-like as budget allows. |
| Regulated / enterprise | **Per environment, per region, often per tenant**, with separate cloud accounts/projects per environment for IAM and billing isolation. |
| Hostile multi-tenancy | Separate clusters per tenant, or a stronger isolation technology. |

## 24.3 If you *do* share a cluster, do these things

1. **Node pools with taints**: `workload=prod:NoSchedule` and matching tolerations; dev pods physically cannot land on prod nodes. Also run system/daemon workloads on dedicated pools.
2. **ResourceQuota on every non-prod namespace**, and set them *low*. Cap `services.loadbalancers` and `persistentvolumeclaims` explicitly — those cost real money.
3. **PriorityClasses** so prod pods preempt dev pods under pressure. Without this, a dev batch job can evict prod.
4. **Strict RBAC**: no one gets cluster-admin in a shared cluster for "just a minute."
5. **Pod Security `restricted`** on prod namespaces, at minimum baseline elsewhere.
6. **Watch cluster-scoped resources like a hawk** — CRDs, admission webhooks, ClusterRoles, GatewayClasses. These are the shared failure domains. Set `failurePolicy: Ignore` on non-critical webhooks with tight timeouts.
7. **Upgrade discipline**: test minor upgrades in the dev namespace's workloads and a scratch cluster *before* touching the shared control plane, and treat control-plane upgrades as a full change-managed event.
8. **Per-namespace observability and cost allocation** — labels on namespaces feeding your cost tooling, or you'll never know which team is spending the money.
9. **Separate the gateway from the tenants.** Tenants attach `HTTPRoute`s to a platform-owned `Gateway`; they never own the Gateway or the certificates. This is exactly what Gateway API's role separation is designed for.

**The bottom line:** the cost of a second cluster is usually a few hundred dollars a month. The cost of a dev experiment taking down prod for an afternoon is usually much higher. Separate prod. If you can only afford one cluster today, do it deliberately, with taints, quotas, and priorities — and plan the split.

---

# Part 25 — Autoscaling and Capacity

There are **four independent autoscalers** in a modern cluster. Confusing them causes most capacity problems.

| Autoscaler | Scales | Trigger | Requirement |
|---|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Pod count | Metrics (CPU, memory, custom, external) | metrics-server (or a custom metrics adapter) |
| **VPA** (Vertical Pod Autoscaler) | Pod requests/limits | Observed usage history | VPA controller. Restart-free updates require `updateMode: InPlaceOrRecreate` (or the alpha `InPlace`); the default mode (`Recreate` — `Auto` is deprecated and equivalent) still evicts and recreates |
| **Cluster Autoscaler / Karpenter** | Node count | Scales **up** on pods that failed to schedule and would fit on a new node; scales **down** on nodes whose pod *requests* are well below allocatable. It is not a CPU-usage autoscaler. | Cloud provider integration |
| **KEDA** | Pod count, incl. scale-to-zero | Event sources (Kafka lag, queue depth, cron, Prometheus) | KEDA + ScaledObject |

**New in 1.37: HPA scale-to-zero went Beta.** HPA can now scale a workload to zero replicas (previously KEDA-only territory), which is a significant change for cost optimization of idle services — and a change you must design for, because a workload at zero needs a real cold-start path.

## 25.1 HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api, namespace: prod }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
    scaleDown:
      stabilizationWindowSeconds: 300        # don't thrash; wait 5 min of low load
      policies: [{ type: Percent, value: 25, periodSeconds: 60 }]
```

**HPA requirements and traps:**
- **`resources.requests` must be set**, or CPU utilization targets are meaningless (utilization is a % of *request*, not of the node). This is the #1 HPA failure.
- **metrics-server** must be installed and healthy (`kubectl top pods` works = it's fine).
- **HPA fights GitOps** if `replicas` in git and HPA both manage the same field. Fix: **do not set `replicas` in the GitOps manifest** (or configure Argo CD `ignoreDifferences` for `/spec/replicas`). Otherwise Argo CD will reset the replica count to the git value every sync. This bites everyone once.
- **Scale-up can't outrun a slow-starting app.** If a pod takes 3 minutes to be ready, HPA reacting to CPU/user load is *too late*. You need headroom (raise `minReplicas`) or scale on a leading indicator (queue depth via KEDA, RPS via a custom metric) rather than a lagging one. **Proxy request rate per pod is a great leading indicator** and is often available from your gateway's metrics.
- **Cluster capacity limits HPA.** If the cluster has no room for new nodes, HPA raises `replicas` and pods sit `Pending`. HPA and cluster autoscaler must be paired.
- **Scale-to-zero (beta in 1.37) needs a wake path.** Requests must be able to trigger scale-up; otherwise the first request after idle fails.

## 25.2 VPA

Useful for right-sizing: run VPA in `updateMode: Off` (recommendation-only) to get suggestions without restarts, then apply them to your manifests. **Do not confuse Kubernetes' in-place resize with VPA's behavior** — they are separate features. Kubernetes 1.35 made in-place pod resize Stable, but that does not change what VPA's default mode does. The documented default is **`Recreate`** (the older `Auto` value is deprecated and equivalent to it), and it still **evicts and recreates** pods to apply new requests. Restart-free adjustment requires `updateMode: InPlaceOrRecreate` (or the alpha `InPlace` mode), and even then VPA falls back to recreation when an in-place change is infeasible — for example if the change would alter a pod's QoS class, or if it cannot be applied within the resize window. **Do not run HPA and VPA on the same metric for the same workload** — they will fight.

## 25.3 Node autoscaling

- **Cluster Autoscaler** — classic. Adds nodes when pods are unschedulable *and would fit if a node were added*; removes nodes that are no longer needed. Two clarifications that explain most confusion about it: it reacts to **unschedulable pods**, not to CPU load (upstream's own FAQ answers "should I use a CPU-usage-based node autoscaler with Kubernetes?" with "No"), and its scale-down decision uses the ratio of pod **requests** to allocatable (`--scale-down-utilization-threshold`, default 0.5) plus whether every pod on the node can be moved. So a cluster that looks busy but is fully requested will not scale, and a pod whose request exceeds any available instance type will never trigger scale-up at all. Works via cloud node groups. Slower, group-based, less flexible.
- **Karpenter** — the modern approach. Provisions right-sized nodes directly from pending pod requirements, consolidates aggressively, supports spot with automatic fallback, and is dramatically faster than ASG-based (Auto Scaling Group) scaling. If you're on EKS, Karpenter is the current best practice.
- **GKE Autopilot / AKS automatic / EKS Auto Mode** — the provider handles node provisioning entirely.
- **Overprovisioning pattern** — run a low-priority "pause" Deployment with N replicas; when real pods arrive, the pause pods are preempted and the autoscaler immediately has capacity to fill. This removes the 2–5 minute node-provisioning delay from your critical path. Standard trick, worth using for latency-sensitive services.

Batch and event-driven workloads scale on different signals entirely (queue depth, not CPU), which is covered in [Part 46](#part-46--batch-event-driven-and-job-workloads).

## 25.4 Real capacity planning

- **Requests are your capacity budget.** Cluster usable capacity = sum of allocatable resources minus system reservations. Keep total requests at 60–70% of allocatable; above that, scheduling failures and eviction storms start.
- **Right-size continuously.** Under-requested pods cause OOMKills and evictions; over-requested pods waste money and block scheduling. VPA recommendations + `kubectl top` + a periodic review is how you keep this honest.
- **Know your ceilings**: max pods per node (110 default, ENI-limited on AWS), max nodes per cluster, IP space (a /16 pod CIDR is a real constraint at scale), and API server QPS limits.
- **Load test before launch**, and test *failure*: kill a node, kill a zone, kill the database, kill the gateway. Chaos engineering (Litmus, Chaos Mesh, AWS FIS — Fault Injection Simulator) turns "we think it fails over" into evidence. **Kill a gateway replica** specifically — proxy-layer failover is the least-tested and most-impactful path.

---

# Part 26 — Cluster Lifecycle Operations

## 26.1 The operational calendar

Things that need to happen on a schedule, not "when we remember":

| Frequency | Task |
|---|---|
| Continuous | Alerting on platform health; drift detection |
| Weekly | Review failed jobs, crashlooping pods, pending PVCs, cost anomalies |
| Monthly | Patch node OS images; review RBAC and unused resources; cost review |
| Quarterly | Kubernetes minor upgrade planning; restore drill; certificate inventory review |
| Every ~15 weeks | New Kubernetes minor released — plan the next hop |
| Annually | Full rebuild-from-scratch drill; disaster recovery test; CIS benchmark pass |

## 26.2 Cluster upgrades

**Cadence reality:** Kubernetes releases a minor every ~15 weeks, supports N-2 (≈14 months). If you skip upgrades, you eventually face a multi-version jump, expired certificates, unsupported add-ons, and a scary weekend. **Upgrade one minor at a time, on a schedule, with a staged rollout.** Never jump minors.

**Order of operations (managed clusters differ, but the shape holds):**

1. **Read the release notes and the deprecated-API removal list.** Check your manifests with `pluto detect-all-in-cluster` or `kubent`, and your add-ons' compatibility matrices (CNI, CSI (Container Storage Interface), Gateway API implementation, Argo CD, cert-manager — each has a supported-version table).
2. **Upgrade the control plane** (managed: a button/API call; self-managed kubeadm: `kubeadm upgrade plan` → `apply`).
3. **Upgrade the CNI/add-ons** to versions supporting the new Kubernetes minor.
4. **Upgrade nodes**, one at a time: `kubectl cordon node` → `kubectl drain node --ignore-daemonsets --delete-emptydir-data` → upgrade kubelet/runtime → `kubectl uncordon`. PDBs protect availability. For managed clusters, use node group rolling upgrades (surge settings matter — no surge means capacity dips).
5. **Verify** — workloads healthy, no `Pending` due to version skew, no crash-looping platform controllers, all CRDs still serving, and `kubectl api-versions` sane.
6. **Repeat minor by minor**, dev → staging → prod, with a bake period.

**Specific recent breakages to watch for** if you're crossing these versions:
- **`Service.spec.externalIPs` deprecated in 1.36** — not removed, but upstream advises migrating off it; it is incompatible with dual-stack.
- **SELinux volume label changes** (GA in 1.36, with implications in 1.37) — can change how volumes are labeled on SELinux-enforcing hosts.
- **etcd 3.6/3.7 upgrades** — there's a documented "zombie cluster member" footgun when upgrading to etcd 3.6. Read before you upgrade etcd.
- **Declarative validation GA (1.36)** and **storage version migration enabled by default (1.37)** change some upgrade behaviors.

**Node image patching is separate from Kubernetes version upgrades** and must happen more often (monthly is a reasonable baseline) — OS CVEs don't wait for a Kubernetes minor release.

## 26.3 Certificate rotation

- **kubeadm clusters**: control-plane certificates expire (typically 1 year). `kubeadm certs check-expiration`, `kubeadm certs renew all`, then restart the control plane. **Letting the apiserver cert expire takes down the cluster** — put a calendar entry or a monitoring check on it.
- **Managed clusters**: handled. Verify the provider's SLA and that you're on a supported version anyway.
- **Webhook/aggregation certs** and ServiceAccount token signing keys also rotate — some managed providers rotate SA keys automatically, some need action.
- **Your own TLS** — cert-manager (Part 14), with expiry alerts.
- **Etcd peer/client certs**, and the CA certificate itself (10-year default). Know when the CA expires; replacing it is a significant operation.

## 26.4 Node maintenance and disruption

```bash
kubectl cordon node-1
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --force   # --force only for unmanaged pods
# ... maintenance ...
kubectl uncordon node-1
```

This is the short version; the full anatomy of a safe drain, why drains hang, and permanent node removal are in [Part 39](#part-39--node-lifecycle-and-maintenance).

Common blockers: **a PDB preventing eviction** (check whether it's `minAvailable == replicas`), **emptyDir data loss** warnings, **`local-path` volumes / hostPath** pinning pods, and **StatefulSet pods**, where what actually blocks or breaks is either a PDB preventing eviction, or the application being unable to re-form quorum after a member moves.

**Spot instance interruptions** deserve their own handling: a 2-minute termination notice arrives on the instance metadata service. The **AWS Node Termination Handler** (or Karpenter's native handling) cordons and drains gracefully. Without it, spot reclaims look like random pod restarts.

## 26.5 Platform add-on inventory

An empty cluster becomes a platform by installing these. Track versions and upgrades for each — this inventory *is* your platform:

| Layer | Typical choice |
|---|---|
| CNI | Cilium / Calico / cloud CNI |
| kube-proxy replacement | Cilium eBPF |
| Load balancer (bare metal) | MetalLB / Cilium LB-IPAM / kube-vip |
| Gateway / reverse proxy | Envoy Gateway / Cilium / cloud Gateway |
| Certificates | cert-manager (+ trust-manager) |
| DNS records | external-dns |
| Secrets | External Secrets Operator / Vault / Secrets Store CSI |
| GitOps | Argo CD / Flux |
| Metrics | kube-prometheus-stack, metrics-server |
| Logs | Fluent Bit + Loki / Vector + OpenSearch |
| Traces | OpenTelemetry Collector + Tempo/Jaeger |
| Policy | ValidatingAdmissionPolicy / Kyverno / Gatekeeper |
| Runtime security | Falco / Tetragon |
| Image scanning | Trivy Operator |
| Backup | Velero + CSI snapshots |
| Autoscaling | HPA / VPA / KEDA / Karpenter |
| Progressive delivery | Argo Rollouts / Flagger |
| Cost | OpenCost / Kubecost |
| Node OS | Talos / Bottlerocket / Flatcar / managed AMI |

**Pin versions and upgrade deliberately.** Unpinned `stable` tags in platform tooling is how a Tuesday morning turns into a three-hour incident. Every add-on you install is a new component to monitor, patch, and eventually remove.

---

# Part 27 — Backup, Disaster Recovery, and Business Continuity

## 27.1 Define RTO and RPO before choosing tools

- **RPO (Recovery Point Objective)** — how much data you can afford to lose. 24h? 5 minutes? Zero?
- **RTO (Recovery Time Objective)** — how long you can be down. An hour? A day?

These numbers decide everything: backup frequency, whether you need a warm standby, whether you can use DNS failover or need active-active. Writing them down turns DR from a vague anxiety into an engineering plan with a budget. **If nobody has stated an RTO, you don't have a DR plan; you have a hope.**

## 27.2 Three things must be backed up

1. **Cluster state (etcd)** — for self-managed clusters, take snapshots regularly and **test restoring**.
   ```bash
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     snapshot save /backup/etcd-$(date +%F).db
   ```
   For GitOps-managed clusters, etcd is *mostly* reconstructible from git — except for things not in git: dynamically provisioned PVs, Secrets synced from external stores (recoverable), and anything created by hand. Which is an argument for keeping nothing out of git.
2. **Persistent volume data** — **Velero** (resources + volumes via CSI snapshots or file-level backup, restorable into a *different* cluster) plus **CSI VolumeSnapshots**. Kubernetes 1.36 added **volume group snapshots** GA for consistent multi-volume snapshots (important for a database spread across multiple volumes).
3. **Application data (logical backups)** — database dumps and PITR exported to object storage in a **different account/region**. This is the most important one and the one that actually restores correctly.

**A backup that has never been restored is not a backup.** Schedule restore drills (quarterly minimum) and record the RTO/RPO you *actually* achieved.

## 27.3 Failure scenarios and what handles them

| Failure | Handled by | Test it by |
|---|---|---|
| Pod dies | ReplicaSet/Deployment | `kubectl delete pod` |
| Container wedges | Liveness probe | Kill the process inside |
| Node dies | Scheduler + ReplicaSets (and cluster autoscaler) | Stop a node |
| Availability zone lost | Multi-AZ node groups + **topology spread** (not PDBs — see note) | Drain a zone |
| Reverse proxy/gateway dies | Multiple replicas + PDB + LB health checks | Kill all but one gateway pod, then all |
| Database primary fails | Operator failover or managed multi-AZ | Force a failover |
| Bad deploy | Rolling update + rollback (or canary analysis) | `kubectl rollout undo` |
| Bad *data* deploy (migration) | PITR | Restore to a point before it |
| Region lost | Second cluster + DNS/GSLB failover | Fail over for real, once |
| Cluster destroyed | Rebuild from infrastructure-as-code + GitOps + restore data | Full rebuild drill |
| Ransomware / compromised creds | Immutable/versioned backups, a separate account, and MFA (multi-factor authentication) delete protection | Restore from an isolated copy |
| Accidental namespace deletion | Velero backup **that includes volume data** + PV retention policy | Delete a namespace in dev |
| Certificate expiry | cert-manager + expiry alerts | Force-renew |

**Two corrections to the table above, both commonly misunderstood.**

**PDBs do not help with a lost zone.** A PodDisruptionBudget constrains *voluntary* disruptions — and only when the evictor uses the Eviction API. An availability zone failing is an **involuntary** disruption; upstream is explicit that PDBs "cannot be prevented by PDBs" in that case (the budget is merely debited). What saves you from a lost zone is **topology spread across zones** plus enough replicas. A PDB is a maintenance-safety tool, not an availability guarantee.

**A Velero backup without a volume path restores empty volumes and reports success.** Velero's primary backup captures *API objects* to object storage; volume data is a **separate, opt-in** path (CSI snapshots, or File System Backup). A backup configured without either will restore your Deployments and Services flawlessly and your databases as empty directories — which is the single most common Velero failure, and the reason to test a restore rather than trusting a green backup job. For CSI snapshots specifically: they are crash-consistent (the application is not quiesced, and in-memory state is absent), bound to the driver and region, and restoring into another cluster requires the same CSI driver.

**Notice how many of these are only tested by actually doing them.** The value is in the drill, not the document.

## 27.4 Backup hygiene

- **3-2-1 rule**: 3 copies, 2 media, 1 offsite (a different account/region is the modern version).
- **Immutable / object-lock backups** where available, so a compromised credential can't delete them.
- **Encrypt backups**, and manage the keys separately from the data.
- **Alert on backup job failure.** A silently failing backup is worse than no backup, because you believe you're covered. This is one of the most common and most damaging operational failures.
- **Document the restore runbook** and keep it somewhere you can read it when the cluster is gone (not only inside the cluster).
- **Test the *whole* restore**, not just the database: new cluster → infrastructure-as-code → GitOps sync → secrets → data restore → DNS cutover. The parts people forget are DNS and secrets.

---

# Part 28 — Multi-Cluster and Multi-Region

## 28.1 When you actually need multiple clusters

- **Environment isolation** (Part 24) — the most common reason, and usually the only one you need at first.
- **Regulatory/data residency** — data must stay in a country/region.
- **Blast-radius reduction** — separating business units or tiers so one failure doesn't take everything.
- **Scale beyond one cluster's limits** — very large clusters hit API server, etcd, and IP-space limits (thousands of nodes, tens of thousands of pods).
- **Latency** — serving users from multiple geographies.
- **Availability** — surviving the loss of a whole region or provider.
- **Different trust levels** — hostile multi-tenancy (Part 23.3).

**Do not go multi-cluster for its own sake.** Each cluster multiplies operational cost: upgrades, add-on versions, certificates, monitoring, and cognitive load. Multi-cluster is a tax you pay for a specific benefit — know which one.

## 28.2 The patterns

| Pattern | Description | Complexity |
|---|---|---|
| **Independent clusters + DNS failover** | Each cluster is standalone; health-checked DNS (or a global LB) sends users to a healthy one. Data replication is your problem. | Low |
| **Hub-and-spoke** | One management cluster runs Argo CD/CAPI for all others. | Low-medium |
| **Multi-cluster service discovery** | Cilium ClusterMesh / Submariner: flat networking and cross-cluster services. | Medium |
| **Multi-cluster service mesh** | Istio multi-primary/multi-network: unified identity and routing across clusters. | High |
| **Active-active with global LB** | Multiple regions serve traffic simultaneously; data layer must handle multi-region writes (hard). | High |
| **Federated control planes** | Karmada, Fleet, or cloud-specific fleet management. | Medium-high |
| **Cell-based architecture** | Many small identical clusters ("cells"), each serving a slice of tenants; failures are contained per cell. | High but very robust |

## 28.3 The data problem, which is the actual hard part

Multi-cluster compute is comparatively easy. **Multi-cluster data is where multi-region projects die.** Your options:

- **Single-region primary + read replicas** — simplest, but writes in a failed region are unavailable, and cross-region writes are slow.
- **Multi-primary async replication** — writes anywhere, replication lag means conflict resolution and potential data loss on failover. Most teams accept this and design for last-write-wins or tenant-pinning.
- **Consensus-based (CockroachDB, YugabyteDB, Spanner)** — synchronous multi-region consistency. Expensive, and write latency is bounded by the speed of light between regions.
- **Tenant/shard pinning** — each tenant's data lives in exactly one region; failover moves *tenants*, not individual rows. This is the pragmatic, widely-used answer, and it composes well with cell-based architecture.

**Be brutally honest here.** "We'll do active-active multi-region" is a two-year project for most systems, not a quarter. Most businesses are far better served by a single well-run region with tested backups and a documented DR procedure than by a half-built active-active setup that fails in a novel way during its first real incident.

## 28.4 Multi-cluster operational concerns

- **Deployment**: Argo CD `ApplicationSet` with a cluster generator, or Flux with per-cluster overlays. One source of truth, many destinations.
- **Observability**: a single query plane (Thanos/Mimir, or a SaaS) with a `cluster` label on every signal. Without this, "is this a global problem or one cluster?" is unanswerable.
- **Cost**: per-cluster control plane fees, cross-region data transfer, duplicated observability stacks.
- **GitOps topology**: hub-and-spoke vs per-cluster Argo CD (Part 21.5) — decide based on whether you can tolerate a hub outage stopping deploys.
- **Upgrades**: you now have N clusters to upgrade on a cadence. Automate it or drown. Upgrade nonprod → one prod region → all regions.
- **Identity**: cross-cluster identity for workloads (SPIFFE, mesh, or shared IdP) and for humans (one IdP, RBAC per cluster).

---

# Part 29 — Cost

Kubernetes makes it easy to spend money invisibly. Where it goes, and what to do:

- **Node overhead (often 30–60% of the bill).** Fix with Karpenter/Cluster Autoscaler consolidation, **spot/reserved instances** for stateless workloads (with `topologySpreadConstraints` and PDBs so spot interruptions don't hurt), and right-sizing requests. Track **requested vs used** — most clusters run at 15–30% actual utilization against requests.
- **Orphaned resources.** Released PVs, unattached cloud disks, unused LoadBalancers (**each is a monthly charge**), old snapshots, abandoned dev namespaces. Automate detection; this is free money. Remember that deleting a StatefulSet leaves PVCs behind by design.
- **NAT gateway data processing.** A quiet, large line item. Route cloud-service traffic through VPC endpoints (Part 15.2), not the NAT gateway.
- **Cross-AZ and cross-region traffic.** Chatty services paying egress. Use `trafficDistribution: PreferSameZone` (or `PreferSameNode` where appropriate) and zone-aware placement.
- **Log and metric volume.** Logs are the sneaky one — verbose debug logging in prod can cost more than the compute. **Proxy access logs are a major contributor at high request rates**; sample or trim fields if the bill is large. Sample traces, filter logs, set retention.
- **Overprovisioning for HA.** 3 replicas of everything, Multi-AZ, PDBs, and now multiple gateway/proxy replicas and possibly a mesh — necessary, but budget it. Autoscale down at night for non-prod (KEDA cron scaling, or scaled-to-zero dev environments — and note HPA can now scale to zero in 1.37).
- **Cost allocation.** You cannot optimize without showing teams their spend. Label namespaces with `team`/`cost-center`, run **OpenCost** or the cloud's own cost allocation, and review monthly. This single practice usually finds 20–40% savings.
- **GPU cost** (Part 30) is its own category — GPU nodes idle between jobs are the biggest single line item in many AI companies. **Multi-Instance GPU (MIG)** partitioning, time-slicing, and aggressive scale-to-zero are the levers.
- **Non-prod hygiene.** Dev clusters idle 70% of the time. Options: scheduled scale-down, ephemeral preview environments (delete on merge), or a shared non-prod cluster with quotas.

---

# Part 30 — AI/ML and GPU Workloads

If you're running anything AI-related, this is now a first-class part of "modern Kubernetes" rather than a niche. Two distinct workloads with different needs:

## 30.1 Inference (serving models)

**The new front door is the AI Gateway.** In March 2026 the community formed an **AI Gateway Working Group**, and the **Gateway API Inference Extension** extends Gateway API concepts to **LLM** (large language model) serving, because traditional HTTP load balancing is wrong for inference:

- **Requests have wildly variable cost** — a 10-token prompt and a 10,000-token prompt are not equal work. Least-request and round-robin both misbehave.
- **KV cache locality matters enormously.** As a model generates text it caches attention key/value state (the "KV cache") for the prompt it has seen. If a follow-up request lands on the pod that already holds that state, generation continues from cache; if it lands elsewhere, the prompt must be reprocessed from scratch. **Prefix-aware and KV-cache-aware routing** — sending requests with a shared prefix to the same backend — are the whole point of the inference extension.
- **Model-aware routing** — different backends may serve different models, or different **LoRA** (Low-Rank Adaptation) adapters over a shared base model; routes need to select on that.
- **Streaming (SSE) is the norm** — which means the proxy buffering pitfalls from Part 11.4 are not edge cases, they're the default path. Disable buffering on inference routes.
- **Long timeouts** — generation can take minutes; typical default proxy timeouts (Envoy's default route timeout is 15s) will cut it off. Set `request` timeouts explicitly per route.

**Serving stacks** — inference servers that load a model and expose an HTTP API — include vLLM, Hugging Face TGI (Text Generation Inference), SGLang, NVIDIA Triton, KServe, Ray Serve, and Ollama for small/local use. They are all just deployments with GPU requests, autoscaling on the right metric (queue depth, not CPU), and a Gateway API route in front.

## 30.2 Training and batch

- **Gang scheduling / workload-aware scheduling.** Distributed training needs *all* workers running simultaneously or none — a half-scheduled job wastes GPUs. Kubernetes is actively building this in (workload-aware scheduling in 1.35–1.37), and **Volcano** (an alternative batch scheduler) and **Kueue** (a job queueing system) provide it today. This is one of the biggest real gaps traditional Kubernetes scheduling has for AI.
- **Job queues and quotas** — **Kueue** provides fair-share queueing across teams so one training job doesn't consume every GPU. **Volcano** adds batch-scheduler semantics.
- **Checkpointing** — long jobs will be preempted (especially on spot). Use the **Checkpoint/Restore Working Group** work (checkpointing containers is an active area) plus framework-level checkpointing to object storage. Assume preemption.
- **Pipelines** — Kubeflow Pipelines, Argo Workflows, Flyte, or Metaflow for orchestration. Argo Workflows is the common Kubernetes-native choice.
- **Notebooks** — JupyterHub on Kubernetes for interactive work, with per-user namespaces and quotas (which is a nice practical application of the multi-tenancy rules in Part 23).

## 30.3 GPU resource management — DRA is the modern answer

The old way: `resources.limits: { nvidia.com/gpu: 1 }` — a bare integer count. It cannot express *which* GPU model you need, cannot require devices to be co-located (two GPUs on one NUMA node, or a GPU plus an RDMA NIC), and has no notion of device attributes. (Sharing and partitioning a GPU is solved separately, by the device plugin — see 30.4.)

**Dynamic Resource Allocation (DRA)** — GA in 1.34 and expanding rapidly (1.36 added more drivers and features, 1.37 continued) — replaces that with a proper API: `ResourceClaim`s and `DeviceClass`es describe what a pod needs (device type, attributes, capacity, topology constraints), and the scheduler matches them. This enables:

- **MIG partitioning** (split one physical GPU into smaller isolated slices)
- **GPU sharing / time-slicing** across pods
- **Topology-aware placement** (keeping work near its memory: same **NUMA** node — the memory attached to one CPU socket — or the same PCIe/NVLink interconnect)
- **Multi-device requests** (GPUs + RDMA NICs together, which is what real training jobs need)

**One correction to a widespread misconception:** MIG partitioning and GPU time-slicing/MPS are **not** DRA features. They are NVIDIA device-plugin and GPU Operator capabilities that predate DRA — MIG instances are advertised as ordinary extended resources (e.g. `nvidia.com/mig-1g.5gb`) and time-slicing is enabled in the device plugin's configuration. Both are *also* available through DRA, whose driver can allocate MIG devices. The accurate framing is that MIG and time-slicing work on **either** path, and DRA's distinctive contribution is **attribute- and topology-aware selection** among devices — pinning a specific GPU model, or requiring a GPU and an RDMA NIC on the same NUMA node — which a bare integer count genuinely cannot express.

**If you're building GPU infrastructure now, learn DRA, not the extended-resource syntax.** The **device plugin** — the kubelet-side agent that advertises GPUs to the scheduler — is the older mechanism that DRA is progressively replacing for *selection*, though the plugin remains the mechanism for sharing and partitioning.

## 30.4 Practical GPU cluster concerns

- **GPU nodes are expensive; idle GPUs are the biggest waste in AI infrastructure.** Autoscale aggressively, use MIG/time-slicing to pack more work per GPU, and scale training jobs to zero when idle.
- **Taints and node pools** — dedicate GPU nodes (`nvidia.com/gpu=present:NoSchedule`) so non-GPU workloads don't consume them, with a separate **node pool** (a provider-managed group of identically configured machines) per GPU model.
- **Drivers and the device plugin** — the **NVIDIA GPU Operator** manages drivers, container runtime config, and the device plugin as a DaemonSet. Getting this wrong is a multi-day debugging exercise; use the operator.
- **Images are huge** (multi-GB). Image pull time dominates cold start — use image pre-pulling, a local registry/cache, or SOCI-style lazy loading.
- **Network** — multi-node training needs high-bandwidth, low-latency fabric (InfiniBand/RoCE), which means **RDMA devices** requested via DRA and a CNI that handles them (Cilium and others support this).
- **Storage** — training reads datasets at very high throughput. Object storage + a caching layer, or a parallel filesystem.
- **Backups matter less, artifacts matter more** — model weights, checkpoints, and datasets in object storage; the cluster stays disposable.

## 30.5 Agentic workloads (the newest thing)

There's active work on running AI *agents* on Kubernetes — sandboxed code execution with strong isolation, short-lived workloads, and per-agent identity. The **Agent Sandbox** project (2026) is the community's answer to "run untrusted generated code safely." If you're building agent platforms, this is the area to watch, and the security model (Part 18) matters more than usual: you are running code you did not write, at scale. gVisor/Kata-style isolation, strict NetworkPolicy, tight RBAC, and aggressive resource quotas are the baseline.

---

# Part 31 — Platform Engineering: Building the Golden Path

You are not just running Kubernetes; **you are building an internal platform on top of it.** Whether you call it that or not, that's what happened the moment you installed Argo CD and wrote a namespace convention.

## 31.1 The goal

The goal is that a new service goes from "I have an idea" to "it's deployed, observable, secured, and routable" **without the developer needing to understand Kubernetes** — while still using real Kubernetes underneath, so you're not locked into an abstraction that breaks when you need to go deeper.

**Good platform abstractions are thin, leaky-on-purpose, and optional.** If your abstraction hides Kubernetes entirely, the first production incident becomes unfixable by the people on call.

## 31.2 What a golden path contains

A "golden path" is the supported, paved route for the common case — not a mandate.

1. **A service template / scaffold** — `cookiecutter`/`copier`/Backstage template that generates: Dockerfile, CI config, Kustomize base+overlays, a Gateway API route, ServiceMonitor, PDB, HPA, resource defaults, security context, and a GitOps Application.
2. **Sane defaults baked in** — `runAsNonRoot`, read-only root filesystem, seccomp, resource requests, probes, an SLO, and the common labels. **The secure and correct thing should be the default**, or it won't be done.
3. **A self-service namespace** — created by a PR, with quotas, PSA labels, NetworkPolicy baseline, and gateway access enabled.
4. **A paved deploy path** — push code, CI builds, PR to GitOps, Argo syncs. No kubectl.
5. **Observability by default** — metrics scraped, logs collected, a starter dashboard, and alerts that page the owning team.
6. **Documentation that answers the 10 questions** every developer asks: how do I deploy? how do I see logs? how do I get a hostname? how do I get a secret? how do I add a database? how do I scale? how do I debug? who do I ask? how do I roll back? how do I run a migration?

## 31.3 Tools

| Concern | Options |
|---|---|
| **Developer portal / catalog** | Backstage (the standard), Port, Cortex, Roadie |
| **Scaffolding** | Backstage Software Templates, Copier, Cookiecutter, custom CLI |
| **Abstraction CRDs** | Crossplane Compositions, custom "App" CRDs (KubeVela, Score, Humanitec), or plain Kustomize bases |
| **Self-service namespaces** | A git repo of namespace manifests + Argo CD; Capsule or Hierarchical Namespace Controller for delegated tenancy |
| **Policy as a product** | Kyverno/ValidatingAdmissionPolicy with clear error messages ("add resource limits — see docs/xyz") |
| **Docs** | Backstage TechDocs, MkDocs, Docusaurus, or a well-maintained repo |

**The `App` CRD temptation.** It's genuinely appealing to let developers write:

```yaml
apiVersion: platform.example.com/v1
kind: App
metadata: { name: checkout }
spec:
  image: registry/checkout:v1.2.3
  port: 8080
  host: checkout.example.com
  database: postgres-small
  replicas: { min: 2, max: 10 }
```

...and have a controller expand that into Deployments, Services, HTTPRoutes, HPAs, PDBs, Secrets, and a database claim. This is a real pattern (Crossplane, KubeVela, Score, and countless in-house controllers). **Do it if you can commit to maintaining it**, because you are now writing and operating a controller that is on the critical path for every deployment. If you can't maintain it, a well-documented Kustomize base gets you 80% of the benefit for 5% of the cost.

## 31.4 Measuring the platform

You can't improve what you don't measure:
- **Time from "new service" to "in production"** — should trend down.
- **Deploy frequency, lead time, change failure rate, MTTR** (mean time to recovery) — the **DORA** (DevOps Research and Assessment) metrics — but attribute them to *your platform's* friction, not the teams'.
- **Ticket volume for platform requests** — every recurring ticket is a missing self-service feature (this is the most actionable metric).
- **Adoption of the golden path vs off-path deployments** — off-path deployments aren't disobedience; they're feedback about a gap.
- **Developer satisfaction / friction surveys** — the qualitative signal that predicts attrition of goodwill.

## 31.5 The failure modes of platform teams

- **Building a platform nobody asked for.** Solve the top three recurring tickets first.
- **Too much abstraction.** If developers can't drop to raw Kubernetes when needed, you've built a cage.
- **Abstraction without documentation.** An undocumented internal platform is a distributed riddle.
- **Platform as gatekeeper rather than enabler.** If your team is the only one who can deploy, you are the bottleneck and the incident.
- **Ignoring the last mile.** Someone has to be on call for the platform too — including the gateway, Argo CD, and cert-manager. Unstaffed platforms rot after hours.

---

# Part 32 — The Build-Out Order

For an empty cluster, this is a sane order. Each step assumes the previous one works.

**Phase 0 — Foundations (day 1)**
1. Confirm CNI works; test pod-to-pod, DNS, egress.
2. Set a **default StorageClass**; test a PVC binding and a pod writing to it.
3. Install **metrics-server** (`kubectl top` works).
4. Configure **OIDC auth + RBAC groups**; stop using the admin kubeconfig for humans.
5. Set up **etcd/config backups** and a **restore test**.
6. Enable **audit logging** if self-managed.
7. Write down **RTO/RPO** and who owns the cluster.

**Phase 1 — Platform skeleton (week 1)**
8. **cert-manager** + a staging ClusterIssuer.
9. **LoadBalancer path** (MetalLB on bare metal, or the cloud controller) — verify a Service gets an external IP.
10. **Gateway API + a reverse proxy** (Envoy Gateway / Cilium / cloud), plus a controller designed for a shared Gateway. Get one public hostname serving TLS to a test pod — end to end, DNS included. **Understand which proxy is in the path and how to inspect its config** (Part 11.8).
11. **External Secrets Operator** wired to your secret manager; enable **encryption at rest** for Secrets.
12. **Namespaces** with labels, ResourceQuotas, LimitRanges, and **PSA `enforce: baseline`**.

**Phase 2 — Delivery (week 1–2)**
13. **Argo CD** installed, SSO'd, with the **app-of-apps** root.
14. Put everything from Phase 0–1 **into the GitOps repo** (the cluster must be reproducible from git).
15. **CI pipeline**: build → scan → sign → push immutable tag → PR the manifest.
16. Deploy a real service end to end through the Gateway.
17. **Preview environments** for PRs (optional but high value).

**Phase 3 — Operability (week 2–4)**
18. **kube-prometheus-stack** + Grafana + Alertmanager; wire alerts to a real channel.
19. **Proxy/gateway metrics and access logs** into the same dashboards. Alert on 5xx rate and upstream connection failures.
20. **Log aggregation**; structured logs from apps with request IDs.
21. **Dashboards** per service (RED) and per cluster (USE).
22. **Velero** + application-level backup with a **tested restore**.
23. **NetworkPolicy default-deny ingress** per app namespace (with gateway allowances first!); Cilium audit mode for egress.
24. **Set proxy timeouts, retries with budgets, outlier detection, and slow start** deliberately — do not leave defaults unexamined.

**Phase 4 — Hardening (week 4+)**
25. **Policy engine** in Audit → Enforce (require limits, no `:latest`, registry allowlist).
26. **Image signing enforcement**; Trivy Operator scanning.
27. **Runtime security** (Falco/Tetragon) with alerting on real signals.
28. **PSA `restricted`** in production namespaces.
29. **Egress control** and an internal/external gateway split.
30. **Restrict `exec`/`portforward`** in production namespaces.
31. **CIS benchmark** (kube-bench/Kubescape) and fix findings.

**Phase 5 — Scale and resilience (ongoing)**
32. **HPA/KEDA** with sane behavior configs; **Karpenter/Cluster Autoscaler**.
33. **Trace instrumentation** with OpenTelemetry, propagated through the proxy.
34. **Upgrade cadence** documented and practiced; add-on version tracking.
35. **Cost visibility** (OpenCost) and a monthly review.
36. **Chaos/failure drills**: kill a node, a zone, the database, the gateway; verify failover and alerts.
37. **SLOs and error budgets** for user-facing services.
38. **Platform layer**: service template, self-service namespaces, documentation.

**Do not skip Phase 0 and 4.** The steps people skip are identity, backups, policy, and upgrades — precisely the ones that turn a bad day into a catastrophe.

---

# Part 33 — Command Reference

**Context and access**
```bash
kubectl config get-contexts
kubectl config use-context prod
kubectl config set-context --current --namespace=prod
kubectl cluster-info
kubectl version
kubectl auth can-i --list
kubectl auth can-i create pods --as=system:serviceaccount:prod:api
kubectl api-resources
kubectl api-versions
kubectl explain deployment.spec.strategy
```

**Reading state**
```bash
kubectl get pods -A -o wide
kubectl get all -n prod
kubectl get pods -n prod --sort-by=.status.startTime
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp | tail -30
kubectl describe pod <pod> -n prod
kubectl get deploy api -o yaml
kubectl get pods -l app=api -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,RESTARTS:.status.containerStatuses[*].restartCount'
kubectl get pods --field-selector spec.nodeName=node-1
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl diff -f manifest.yaml
```

**Debugging**
```bash
kubectl logs <pod> -c <container> --previous --tail=200
kubectl logs -l app=api --all-containers --prefix --since=10m
stern -n prod -l app=api --since 10m
kubectl exec -it <pod> -- sh
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
kubectl debug node/<node> -it --image=busybox
kubectl port-forward svc/api 8080:80
kubectl run tmp --rm -it --image=nicolaka/netshoot --restart=Never -- bash
kubectl get endpointslices -l kubernetes.io/service-name=api
kubectl get svc api -o jsonpath='{.spec.selector}'
kubectl cp <pod>:/path/file ./local
```

**Proxy / gateway diagnosis**
```bash
kubectl get gatewayclass
kubectl get gateway -A
kubectl describe gateway main-gateway -n gateway-system
kubectl get httproute,grpcroute,tcproute,udproute -A
kubectl describe httproute api-route -n prod
kubectl get referencegrant -A
kubectl get backendtlspolicy -A
# Envoy-based proxies: inspect the actual generated config
kubectl get deploy -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=main-gateway
kubectl exec -n envoy-gateway-system deploy/<name> -- curl -s localhost:19000/clusters | head -50
kubectl exec -n envoy-gateway-system deploy/<name> -- curl -s localhost:19000/config_dump > /tmp/cd.json
kubectl exec -n envoy-gateway-system deploy/<name> -- curl -s localhost:19000/stats | grep -E 'upstream_cx|upstream_rq_5xx|ejections|retry'
# NGINX
kubectl exec deploy/nginx-gateway -- nginx -T
# End-to-end from outside
curl -sv https://api.example.com --resolve api.example.com:443:<GW_IP>
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
```

**Applying and changing**
```bash
kubectl apply -f manifest.yaml
kubectl apply -k ./overlays/prod
kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
kubectl delete -f manifest.yaml
kubectl scale deploy/api --replicas=5
kubectl patch deploy api -p '{"spec":{"replicas":5}}'
kubectl label node node-1 workload=prod
kubectl taint node node-1 workload=database:NoSchedule
kubectl set image deploy/api api=registry/api:1.2.3
kubectl delete pod <pod> --grace-period=0 --force    # last resort only
```

**Rollouts and jobs**
```bash
kubectl rollout status deploy/api
kubectl rollout history deploy/api
kubectl rollout undo deploy/api --to-revision=2   # then re-sync from git (Part 21)
kubectl rollout restart deploy/api
kubectl rollout pause deploy/api && kubectl rollout resume deploy/api
kubectl create job --from=cronjob/nightly-report manual-run
```

**Nodes**
```bash
kubectl get nodes -o wide
kubectl describe node <node>
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
kubectl get pdb -A
```

**Storage**
```bash
kubectl get storageclass
kubectl get pvc -A
kubectl get pv
kubectl describe pvc data-db-0 -n data
kubectl get volumesnapshot -A
kubectl patch pvc data-db-0 -n data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'   # needs allowVolumeExpansion on the StorageClass
```

**Secrets and config**
```bash
kubectl create secret generic api --from-literal=KEY=value --dry-run=client -o yaml
kubectl get secret api -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
kubectl get externalsecret -A
kubectl get secretstore,clustersecretstore -A
```

**GitOps**
```bash
argocd app list
argocd app get api-prod
argocd app diff api-prod
argocd app sync api-prod
argocd app history api-prod
argocd app rollback api-prod <id>
kubectl -n argocd get applications -o wide
kubectl -n argocd get app <name> -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}'
```

**Helm**
```bash
helm repo add <name> <url> && helm repo update
helm search repo <chart> --versions | head
helm show values <repo>/<chart> > values.yaml
helm template <release> <repo>/<chart> -f values.yaml | less
helm upgrade --install <release> <repo>/<chart> -n <ns> --create-namespace -f values.yaml --version <v> --atomic --wait
helm list -A
helm history <release> -n <ns>
helm rollback <release> <rev> -n <ns>
```

**Troubleshooting shortcuts**
```bash
# Every pod not Running/Succeeded, cluster-wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Why is this pod not ready?
kubectl get pod <pod> -o jsonpath='{.status.conditions}' | jq

# Restart counts and last termination reason
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.restartCount}{"\t"}{.lastState.terminated.reason}{"\n"}{end}'

# Pods with no CPU/memory requests (Burstable, or BestEffort if they also set no limits)
kubectl get pods -A -o json | jq -r '.items[] | select([.spec.containers[].resources.requests] | all(. == null)) | "\(.metadata.namespace)/\(.metadata.name)"' | sort -u

# Node capacity vs allocated
kubectl describe node <node> | sed -n '/Allocated resources/,/Events/p'

# Certificates expiring soon (cert-manager) — sorted by actual expiry
kubectl get certificate -A -o json | jq -r '.items[] | select(.status.notAfter != null) | "\(.status.notAfter)\t\(.metadata.namespace)/\(.metadata.name)"' | sort

# Leaked cloud disks: PVs whose PVC is gone (Released). 'Available' PVs are merely unbound.
kubectl get pv | awk '$5=="Released"'

# Services with no endpoints (broken selectors)
kubectl get endpoints -A | awk '$3=="<none>"'   # column 3: NAMESPACE is column 1 with -A
```

---

# Part 34 — Tooling Landscape

The tools you'll actually encounter, grouped by what they do. (This is a map, not a mandate — install what solves a problem you have.)

**Cluster provisioning**
`Terraform` / `OpenTofu`, `Pulumi`, `Crossplane`, `Cluster API`, `kubeadm`, `k3s`/`RKE2`, `Talos`, `kind`/`minikube`/`k3d` (local)

**Interactive/CLI**
`kubectl`, `k9s`, `kubectx`/`kubens`, `kube-ps1`, `stern`, `kubectl-neat`, `kubectl-tree`, `popeye`, `pluto`/`kubent`, `kubectl-view-allocations`, `kubectl-explore`, `kubecolor`, `fubectl`

**Templating/packaging**
`Helm`, `Kustomize`, `Jsonnet`/`tanka`, `CUE`, `cdk8s`, `helmfile`, `helm-diff`, `chart-testing`

**GitOps/CD**
`Argo CD`, `Argo Rollouts`, `Argo Workflows`, `ApplicationSet`, `Flux CD`, `Flagger`, `Rancher Fleet`, `KubeVela`, `Jenkins X`, `Tekton`

**Networking / proxies / gateways**
`Cilium`, `Calico`, `Antrea`, `Flannel` (avoid), `MetalLB`, `kube-vip`, `Envoy`, `Envoy Gateway`, `NGINX` / `NGINX Gateway Fabric`, `HAProxy`, `Traefik`, `Contour`, `Kong`, `Gloo`, `Istio`, `Linkerd`, `Submariner`, `external-dns`, `multus` (multi-network), `Cilium ClusterMesh`

**Storage**
CSI drivers (EBS, GCE PD, Azure Disk, Ceph, Longhorn, Rook, NFS), `OpenEBS`, `Longhorn`, `Rook`, `Velero`, `k8up`, `volume-snapshot` tooling

**Secrets/config**
`External Secrets Operator`, `Sealed Secrets`, `SOPS` + `age`, `Vault` + `vault-secrets-operator`, `Secrets Store CSI Driver`, `cert-manager`, `trust-manager`, `reflector`

**Observability**
`kube-prometheus-stack`, `Prometheus Operator`, `Thanos`, `Mimir`, `VictoriaMetrics`, `Grafana`, `Alertmanager`, `Loki`, `Fluent Bit`, `Vector`, `Grafana Alloy`, `OpenTelemetry Collector`, `Tempo`, `Jaeger`, `Pyroscope`, `Parca`, `kube-state-metrics`, `node-exporter`, `metrics-server`, `Pixie`, `Hubble`

**Security / policy**
`Kyverno`, `OPA Gatekeeper`, `ValidatingAdmissionPolicy` (built in), `Falco`, `Tetragon`, `Trivy` / `Trivy Operator`, `Grype`, `cosign` / `Sigstore`, `syft`, `kube-bench`, `Kubescape`, `rbac-police` (note: **kube-hunter is no longer maintained** — its authors recommend Trivy), `Polaris`, `gitleaks`, `trufflehog`, `SPIRE`, `Kyverno`/`vap` for admission

**Autoscaling / capacity**
`metrics-server`, `HPA`, `VPA`, `KEDA`, `Karpenter`, `Cluster Autoscaler`, `Goldilocks` (VPA recommendations as a dashboard), `Descheduler`, `Node Termination Handler`

**Databases on Kubernetes**
`CloudNativePG`, `Crunchy PGO`, `Percona` operators, `Vitess`, `Zalando postgres-operator`, `Strimzi` (Kafka), `Redis/Valkey operators`, `ClickHouse operator`, `PgBouncer`

**AI/ML**
`NVIDIA GPU Operator`, `KServe`, `vLLM`, `TGI`, `Triton`, `Ray`/`KubeRay`, `Kubeflow`, `Kueue`, `Volcano`, `Argo Workflows`, `JupyterHub`, `Gateway API Inference Extension`, `Agent Sandbox`

**Backup/DR**
`Velero`, `Kasten K10`, `CloudNativePG` backups, cloud-native snapshot tooling, `restic`/`kopia`

**Cost**
`OpenCost`, `Kubecost`, `kube-resource-report`, cloud cost tools

**Multi-tenancy / platform**
`vCluster`, `Capsule`, `Hierarchical Namespace Controller`, `Backstage`, `Port`, `Crossplane`, `KubeVela`, `Score`, `HNC`

**Developer experience**
`Tilt`, `Skaffold`, `DevSpace`, `Garden`, `Telepresence`, `Okteto`, `Gefyra`, `mirrord` (run local code against a remote cluster — genuinely excellent for debugging)

---

# Part 35 — Anti-Patterns

Each of these has caused real outages. Read them as a checklist of things to *not* do.

**Workloads**
- `image: myapp:latest` — no reproducibility, no rollback, and image pull policy surprises on nodes that cached an old `latest`. **Pin by tag (preferably digest).**
- Setting **limits but no requests** — Kubernetes copies the limit into the request, so the pod is `Burstable` or `Guaranteed`, not `BestEffort`, and the request you never chose becomes the one the scheduler reserves. Set requests explicitly rather than letting them be inferred.
- **No `resources` at all** — the pod is `BestEffort` by definition (that requires *no* requests and *no* limits on any container), it is the first thing evicted, scheduling is unpredictable, and CPU-based autoscaling becomes meaningless (queue- or request-rate-driven autoscaling can still work).
- Aggressive **liveness probes on external dependencies** — a DB blip restarts every pod simultaneously. Liveness = "is my process wedged," not "is the world healthy."
- No readiness probe — traffic hits pods that aren't ready; deploys cause 502s.
- No `preStop` / no SIGTERM handling — dropped requests on every deploy.
- Single replica for anything user-facing — every node upgrade is downtime. **This includes the gateway/proxy and Argo CD.**
- `replicas: 3` with **pod anti-affinity ignored** — all three land on one node/zone; HA is fiction.
- `hostPath` volumes — pod pinned to a node, security hole, data loss on rescheduling.
- Editing deployments with `kubectl edit` in production — drift from git, undocumented change, broken GitOps.
- Putting a database in the same Pod as the app — different lifecycles, impossible scaling.
- Long-running migrations inside app startup with multiple replicas — race conditions and corruption.

**Proxies / networking**
- **Using L4 `Service` load balancing where you need L7 request balancing** — keep-alive connections pin clients to single pods, and you wonder why one pod is hot.
- **Leaving proxy timeouts, retries, and buffering at defaults** — the defaults are wrong for streaming, for LLMs, and for your slowest legitimate request.
- **Unbounded retries** — retry storms triple load on already-failing backends. Use retry budgets.
- **Aggressive outlier ejection with low replica counts** — you eject the entire backend set and convert a partial problem into a total outage.
- **Timeout stacking in the wrong order** — if the proxy times out *after* the client does, users see errors your logs never record.
- **Rate limiting locally per proxy replica** without a shared counter — your real limit is `limit × replicas`.
- **Assuming client IP arrives intact** — behind a cloud L4 LB you need PROXY protocol or `X-Forwarded-For`, configured on *both* sides.
- **Running two Gateway API implementations** for the same purpose — two data planes, two bills, two failure modes.
- **`allowedRoutes.namespaces.from: All`** on a public Gateway — any namespace can claim any hostname.
- **Exposing admin interfaces** (Argo CD, dashboards, Prometheus, Traefik dashboard, Elasticsearch) on the public internet — these are actively scanned and exploited.
- **No NetworkPolicy** — a flat network with no isolation, so one compromised pod reaches every database in the cluster.
- **Default-deny NetworkPolicy without gateway allowances** — you break your own ingress and debug the wrong layer for an hour.
- **Egress proxy with no HA and no documented fail-open/fail-closed decision** — a single point of failure for every outbound call.
- `type: LoadBalancer` for internal services — each one costs money and widens the attack surface.
- No default StorageClass, or one without `WaitForFirstConsumer` in multi-AZ — `Pending` PVCs.
- Ignoring DNS `ndots` — mysterious latency and timeouts on external calls.
- **WebSocket/SSE through a proxy with buffering and a 60s idle timeout** — works locally, breaks in production.

**Config and secrets**
- Secrets committed to git in plaintext — treat any occurrence as a leak requiring rotation.
- Secrets in ConfigMaps (no encryption, no access-control distinction).
- `kubectl create secret --from-literal` for prod — the value ends up in your shell history (and in CI logs if scripted). Prefer an external secret manager, and remember that even the `--dry-run=client -o yaml` form still puts the literal in shell history.
- Mutating a ConfigMap in place and expecting **environment-injected** config to update in running pods — it will not. (Mounted ConfigMap *files* do refresh within about a minute, but the application still has to re-read them; see [Part 8.1](#81-configmaps).)
- Wildcard IAM roles attached to ServiceAccounts — one compromised pod = whole cloud account.
- Never rotating credentials.
- **`helm upgrade` by hand on a chart that Argo CD also manages** — two owners, eternal fight.

**Platform**
- Everything in `default` namespace — no quotas, no policy scoping, no RBAC boundaries.
- One cluster serving dev+staging+prod without taints, quotas, or priority classes.
- Running your applications in `kube-system`.
- No `PodDisruptionBudget` on critical multi-replica services — node drains cause outages. (Conversely: a PDB with `minAvailable: replicas` blocks drains forever.)
- A validating webhook with `failurePolicy: Fail`, no HA, and a 30s timeout — it becomes a cluster-wide single point of failure. Prefer `ValidatingAdmissionPolicy` for common cases.
- Installing a service mesh "because it's best practice" before you can name the problem it solves.
- CRDs and operators installed without upgrade plans — they break on the next Kubernetes minor.
- Unpinned platform chart versions (`stable`, `latest`) — silent, unannounced upgrades.
- Never upgrading Kubernetes, then facing a 4-version jump and expired certificates.
- No etcd/volume backups; or backups that have never been restored; or backup jobs that fail silently with no alert.
- No audit logs, so nobody can answer "who deleted that."
- **Guessing at RTO/RPO** instead of writing them down.
- **Multi-region active-active chosen for prestige** rather than for a requirement, and half-built.

**Process**
- Manually applying manifests to production outside GitOps — the cluster stops matching git, and nobody knows the true state.
- CI holding long-lived cluster-admin credentials.
- `prune: true` enabled on a wildly-scoped Argo Application without understanding it.
- HPA and `replicas` in git both managing the same field — Argo CD resets the HPA's scaling every sync.
- Deploying on Friday without canary or rollback plan. (Some things are universal.)
- No alerting on the platform layer itself — you find out Argo CD or cert-manager was down when a deploy fails or a cert expires.
- Treating Kubernetes like a VM: SSHing in, hand-configuring, "fixing it live."
- **Building a platform abstraction nobody asked for**, and becoming the bottleneck for every deployment.

---

# Part 36 — Glossary

| Term | Meaning |
|---|---|
| **Admission** | Interception point where objects are validated or mutated before being persisted. |
| **Admission controller** | Intercepts objects after authn/authz, before persistence. Used for policy, defaults, mutation. |
| **Affinity/Anti-affinity** | Rules for co-locating or separating pods relative to nodes or other pods. |
| **AI Gateway** | Gateway API work for routing LLM inference — KV-cache-aware, prefix-aware, model-aware routing. |
| **Allocatable** | The CPU/memory a node can actually give to pods: capacity minus what the OS and system daemons reserve. |
| **Ambient mesh** | Sidecar-less service mesh (Istio): per-node ztunnel for L4 mTLS plus per-namespace waypoints for L7. |
| **Annotation** | Non-selectable key/value metadata for tools and humans (contrast with label). |
| **API Priority and Fairness (APF)** | The API server's queueing system: classifies requests into priority levels so one heavy client cannot starve the rest. |
| **API server (kube-apiserver)** | The cluster's single entry point; authenticates, authorizes, validates, and persists every request. |
| **AppArmor** | A Linux Security Module applying per-program mandatory access control; configured per pod via securityContext. |
| **Argo CD** | A GitOps controller: keeps the cluster matching manifests stored in git. |
| **Availability zone (AZ)** | An isolated failure domain within a cloud region; spreading replicas across zones survives a zone outage. |
| **BackendTLSPolicy** | Gateway API object configuring TLS from the gateway to the backend. |
| **Blue/green** | Deploy two full environments, switch traffic atomically. |
| **Bound token** | A ServiceAccount token that is audience-scoped, time-limited, and rotated automatically — the modern replacement for non-expiring tokens. |
| **Canary** | Send a percentage of traffic to a new version, watch, then promote or roll back. |
| **CEL** | Common Expression Language — the expression language Kubernetes uses for built-in validation policies. |
| **Circuit breaking** | Capping concurrent/pending requests per backend so failure doesn't cascade. |
| **Cluster Autoscaler / Karpenter** | Components that add and remove nodes based on pending pods and utilization. |
| **Cluster Trust Bundle** | An API object distributing CA certificates to workloads so they can verify in-cluster TLS. |
| **ClusterIssuer / Issuer** | cert-manager objects describing where certificates come from (ACME, a private CA, Vault). |
| **CNI** | Container Network Interface — the plugin providing pod IPs and connectivity. |
| **ConfigMap** | Non-secret configuration object. |
| **Consistent hashing (ring hash/maglev)** | LB algorithm mapping a key to a stable backend — for cache locality or session affinity. |
| **Container runtime** | The program that actually starts containers (containerd, CRI-O), reached via the CRI. |
| **Controller** | A reconciliation loop that drives observed state toward desired state. |
| **Cordon** | Mark a node unschedulable so no new pods land on it (existing pods keep running). |
| **CoreDNS** | The cluster's DNS server; resolves Service names to IPs for every pod. |
| **CRD** | CustomResourceDefinition — extends the Kubernetes API with your own types. |
| **CRI** | Container Runtime Interface — the API between the kubelet and the container runtime. |
| **CSI** | Container Storage Interface — the plugin interface for storage. |
| **DaemonSet** | Runs one pod per node. |
| **Deployment** | Controller managing stateless pods via ReplicaSets, with rolling updates. |
| **Descheduler** | A component that evicts pods which no longer fit their placement, letting the scheduler re-place them. |
| **Device plugin** | A kubelet-side agent that advertises special hardware such as GPUs to the scheduler. |
| **DRA (Dynamic Resource Allocation)** | The modern API for requesting GPUs/devices with attributes and topology (GA in 1.34). |
| **Drain** | Cordon a node and evict its pods so maintenance can be performed. |
| **Dual-stack** | Both IPv4 and IPv6. Two distinct layers: the *cluster's* dual-stack support is fixed at creation time (control plane + CNI + kube-proxy), while a *Service* is configured with `ipFamilyPolicy` (`SingleStack`/`PreferDualStack`/`RequireDualStack`), plus `ipFamilies` and `clusterIPs`. |
| **Egress gateway** | A forward proxy controlling/auditing outbound traffic from the cluster. |
| **EndpointSlice** | The list of pod IPs behind a Service, each with a `ready` flag; consumers such as kube-proxy filter on readiness. |
| **Envoy** | The dominant L7 proxy/data plane; configured dynamically via xDS. |
| **Ephemeral storage** | Node-local disk a pod consumes: writable layer, logs, and emptyDir. A schedulable resource, enforced by eviction. |
| **Eviction** | The kubelet removing pods to relieve node pressure; ordered by QoS class and priority. |
| **Eviction threshold** | A kubelet setting (hard or soft, absolute or percentage) that triggers pod eviction when a node resource runs low. |
| **Finalizer** | A marker that blocks an object's deletion until cleanup completes. |
| **Gang scheduling** | Scheduling a group of pods all-or-nothing, so distributed jobs never start half-placed. |
| **Gateway** | A concrete data-plane instance (listeners, TLS, address) — provisions the LB/proxy. |
| **Gateway API** | The successor to Ingress: role-oriented, portable, extensible traffic-routing CRDs. |
| **GatewayClass** | Names which Gateway implementation (controller) fulfils a Gateway. |
| **GitOps** | Desired state in git; an in-cluster controller continuously reconciles reality to it. |
| **Golden path** | The supported, paved route for the common case on an internal platform. |
| **gRPC** | HTTP/2-based RPC; needs L7 proxying and careful timeouts/buffering. |
| **gVisor** | A sandboxing runtime providing a user-space kernel, so containers do not call the host kernel directly. |
| **Headless Service** | A Service with no ClusterIP; DNS returns pod IPs — for StatefulSets and client-side LB. |
| **Helm** | The Kubernetes package manager; packages are charts, installs are releases. |
| **Horizontal scaling (HPA)** | Adding/removing pod replicas based on metrics. |
| **HPA / VPA / KEDA** | Horizontal pods / vertical resources / event-driven autoscalers. |
| **HTTPRoute** | Routing rules (host/path/header) from a Gateway to backend Services. |
| **Huge pages** | Pre-allocated large memory pages requested as a schedulable resource; reduces TLB pressure for latency-sensitive workloads. |
| **Infrastructure as code (IaC)** | Managing cloud and cluster infrastructure through versioned definition files rather than manual changes. |
| **Ingress** | The legacy north-south HTTP API; reference implementation retired 2026. |
| **Init container** | Runs to completion before app containers; also the basis for native sidecars. |
| **IRSA / Workload Identity** | Mapping a Kubernetes ServiceAccount to a cloud IAM role with no static keys. |
| **Job / CronJob** | Controllers for run-to-completion work and for work repeated on a schedule. |
| **KEDA** | Kubernetes Event-driven Autoscaling: scales workloads on external signals such as queue depth, and can scale to zero. |
| **kube-proxy** | Programs Service virtual IPs (VIPs) on each node; often replaced by the CNI's eBPF datapath. |
| **Kubelet** | The per-node agent that starts containers, runs probes, mounts volumes, and enforces eviction. |
| **Kustomize** | A template-free manifest customization tool built into kubectl (`kubectl apply -k`). |
| **KV cache** | The per-token attention key/value state an LLM keeps for a prompt; reusing it avoids recomputation. |
| **L4 / L7** | Transport layer (connections) / application layer (requests). Determines what a balancer can do. |
| **Label** | A selectable key/value pair attached to objects; the basis of every selector in Kubernetes. |
| **LimitRange** | Sets default/min/max resources per container in a namespace. |
| **Local development inner loop** | The edit-build-verify cycle a developer experiences; shortening it is a platform responsibility. |
| **LoRA** | Low-Rank Adaptation — a small adapter that specializes a base model without retraining it. |
| **MIG (Multi-Instance GPU)** | Splitting one physical GPU into isolated slices. Available through *both* paths: the NVIDIA device plugin advertises MIG instances as extended resources, and the NVIDIA DRA driver can allocate them. |
| **mTLS** | Mutual TLS — both sides present certificates; the basis of zero-trust service identity. |
| **Namespace** | A scope for names and a unit of policy — how work is partitioned inside a cluster. |
| **Native sidecar** | An init container with `restartPolicy: Always` that runs alongside the app. |
| **NetworkPolicy** | Pod-level L3/L4 firewall, enforced by the CNI, additive and default-deny-friendly. |
| **Node condition** | A named status flag on a node: Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable. |
| **Node pool** | A provider-managed group of identically configured nodes; the main tool for shaping where workload classes run. |
| **NUMA** | Non-Uniform Memory Access — memory attached to a specific CPU socket; locating work near it is faster. |
| **Operator** | A controller encoding operational knowledge for an application (e.g. a database). |
| **Outlier detection** | Passively ejecting unhealthy backends based on observed errors. |
| **Overprovisioning (capacity buffer)** | Low-priority placeholder pods that hold capacity so real workloads start without waiting for node provisioning. |
| **PDB** | PodDisruptionBudget — limits voluntary disruption; protects drains and upgrades. |
| **Pod** | Smallest deployable unit: one or more containers sharing network and lifecycle. |
| **Pod Certificate** | An API letting a pod obtain a short-lived certificate: the kubelet generates the key, creates a `PodCertificateRequest`, and refreshes the projected bundle, while issuance is performed by the **signer** named in `signerName` — not by the API server, which brokers the request. |
| **PodDisruptionBudget** | Declares minimum availability during voluntary disruption; consulted by eviction and can block a node drain. |
| **PodDisruptionBudget (PDB)** | Declares minimum availability during voluntary disruption; can block a node drain. |
| **Preemption** | Evicting lower-priority running pods so a higher-priority pending pod can be scheduled. |
| **Priority class** | Assigns a pod a priority, affecting scheduling order and preemption. |
| **PriorityClass** | An object assigning scheduling priority and preemption behaviour to pods. |
| **Probe (liveness/readiness/startup)** | Periodic checks the kubelet runs against a container to decide restart vs. route traffic. |
| **PROXY protocol** | A header prepended by L4 LBs preserving the original client IP. |
| **PSA** | Pod Security Admission — built-in enforcement of privileged/baseline/restricted. |
| **PVC / PV** | PersistentVolumeClaim (request) / PersistentVolume (the actual storage). |
| **QoS class** | Kubernetes' automatic classification (Guaranteed/Burstable/BestEffort) deciding eviction order. |
| **RBAC** | Role/RoleBinding/ClusterRole/ClusterRoleBinding — allow-only authorization. |
| **Reconciliation** | The loop that closes the gap between desired and observed state. |
| **ReferenceGrant** | Permission for a cross-namespace reference in Gateway API. |
| **ReplicaSet** | A controller that keeps N identical pods running; managed by a Deployment. |
| **Request / limit** | What a container is guaranteed (scheduling input) vs. the ceiling it may not exceed. |
| **ResourceQuota** | Caps aggregate resource consumption per namespace. |
| **Retry budget** | A cap on retries as a fraction of total requests, preventing retry storms. |
| **Reverse proxy** | A proxy in front of servers: routes, terminates TLS, balances, protects. See Part 11. |
| **RPO / RTO** | Acceptable data loss / acceptable downtime. Define these before choosing tools. |
| **RuntimeClass** | Names an alternative container runtime (gVisor, Kata) and selects it per pod. |
| **Scheduler (kube-scheduler)** | The component that decides which node each new pod runs on. |
| **Seccomp** | A kernel facility restricting which system calls a process may make; `RuntimeDefault` is the standard baseline. |
| **Selector** | A label query used to choose objects (a Service selecting pods, a Deployment selecting ReplicaSets). |
| **SELinux** | A kernel-wide mandatory access control system; in Kubernetes it mainly affects how volumes are labelled. |
| **Server-side apply (SSA)** | Applying only the fields you own, with the API server merging and recording field ownership. |
| **Service** | Stable virtual IP + DNS name in front of selected pods (L4). |
| **Service mesh** | A distributed proxy layer giving all service-to-service traffic identity, mTLS, and policy. |
| **ServiceAccount** | Identity for pods to authenticate to the API and cloud. |
| **ServiceMonitor / PodMonitor** | Prometheus Operator objects declaring which Services/Pods to scrape. |
| **Session affinity** | Routing requests from the same client to the same backend; at L4 it is per client IP, which behaves badly behind NAT. |
| **Sidecar** | An auxiliary container in the same pod (log shipper, proxy). |
| **SLO / error budget** | A reliability target (e.g. 99.9% success) and the failure allowance it implies; burn rate is how fast it is spent. |
| **Slow start** | Ramping traffic gradually to a new backend so it can warm up. |
| **SOPS** | A tool that encrypts secret values in files so they can be stored in git. |
| **Spot instance** | Cheap spare cloud capacity that can be reclaimed with little notice. |
| **StatefulSet** | Controller for stable-identity workloads with per-pod storage. |
| **StorageClass** | Declares how volumes are dynamically provisioned. |
| **Taint/Toleration** | Node repels pods; pods with the toleration may land there. |
| **Topology spread** | Evenly distributing pods across zones/hosts. |
| **Topology spread constraint** | A rule distributing pods evenly across failure domains such as zones or nodes. |
| **ValidatingAdmissionPolicy** | CEL-based policy enforcement built into Kubernetes — no webhook to operate. |
| **WAF** | Web application firewall — L7 attack filtering, usually at the edge. |
| **Waypoint** | In ambient mesh, a proxy deployed for a namespace/service where L7 features are needed. |
| **Windows node** | A worker node running Windows containers; control plane remains Linux and security primitives differ. |
| **xDS** | The dynamic configuration API used by Envoy control planes. |
| **Zero trust** | No implicit trust from network location; every request authenticated and authorized. |
| **ztunnel** | The per-node L4 proxy in Istio's ambient mode. |

---

# Part 37 — Advanced Scheduling

Part 2.8 explained the two-phase filter-and-score model, and Part 6.6 listed the controls available. This part covers the parts that only matter once a cluster is real: spreading workloads across failure domains, letting the scheduler move pods it already placed, and reserving capacity before it is needed.

## 37.1 Why placement is an availability property

A Deployment with three replicas on one node is not highly available. It is one node failure away from a total outage, and it looks perfectly healthy in every dashboard until that failure happens. Placement is therefore not a tuning concern; it is the mechanism by which "replicas: 3" becomes an actual availability guarantee.

There are three independent dimensions to get right, and each protects against a different failure:

| Dimension | Protects against | Mechanism |
|---|---|---|
| **Across nodes** | A single machine failing | `topologySpreadConstraints` or pod anti-affinity on `kubernetes.io/hostname` |
| **Across zones** | A whole datacenter/availability zone failing | topology spread on `topology.kubernetes.io/zone` |
| **Across the failure domain of a volume** | A pod that cannot start because its disk lives elsewhere | Matching the pod's zone to its PVC's zone (implicit, but it constrains placement) |

**The default behavior is weaker than most people assume, but it is not nothing** — and stating it precisely explains why you still need explicit constraints. The scheduler ships **built-in default topology spread constraints**, delivered by the `PodTopologySpread` score plugin: `maxSkew: 3` on `kubernetes.io/hostname` and `maxSkew: 5` on `topology.kubernetes.io/zone`, both with `ScheduleAnyway` (a preference, never a block), with selectors derived from the controlling Service, ReplicaSet, StatefulSet, or ReplicationController.

Two consequences follow. First, those tolerances are extremely loose: with `maxSkew: 3` on hostname, three replicas are *permitted* to land on one node, so the defaults will not stop the packing you were worried about. Second, **a pod with no such controller owner gets no default spreading at all** — a bare pod, or one owned by a custom controller, is not covered. So the conclusion stands — declare explicit constraints — but for the right reason: the defaults exist, they are just too permissive to be an availability guarantee.

## 37.2 Topology spread constraints, properly

`topologySpreadConstraints` is the modern tool and has largely replaced hard pod anti-affinity for spreading. It expresses "distribute evenly across these domains, tolerating at most this much imbalance," rather than the binary "never co-locate."

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone     # the failure domain
      whenUnsatisfiable: DoNotSchedule             # hard: refuse to place rather than cluster
      labelSelector:
        matchLabels: { app.kubernetes.io/name: api }
      minDomains: 3                                # optional: require this many domains to exist
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname          # second constraint: also spread per node
      whenUnsatisfiable: ScheduleAnyway            # soft: prefer, but do not block a deploy
      labelSelector:
        matchLabels: { app.kubernetes.io/name: api }
```

Understanding the fields:

- **`maxSkew`** — the permitted difference between the domain with the most matching pods and the **global minimum**, which is the smallest matching-pod count across eligible domains. `maxSkew: 1` means no domain may hold more than one more than that minimum. Subtlety worth knowing: when the number of eligible domains is **less than `minDomains`**, the baseline is treated as **zero** rather than the actual minimum — which is exactly what makes `minDomains` able to block placement. Only pods in the same namespace matching the `labelSelector`, in domains eligible under `nodeAffinityPolicy`/`nodeTaintsPolicy`, are counted. For 3 replicas across 3 zones with the default `minDomains: 1`, `maxSkew: 1` gives one each.
- **`topologyKey`** — which node label defines a domain. `kubernetes.io/hostname` = per-node. `topology.kubernetes.io/zone` = per-zone. Any node label works, which is how you spread across custom domains (rack, power feed, GPU model).
- **`whenUnsatisfiable`** — `DoNotSchedule` makes the constraint hard: if it cannot be satisfied, the pod stays `Pending`. `ScheduleAnyway` makes it a scoring preference: the scheduler tries, but places the pod anyway. **This is the field people get wrong most often**, in both directions. A hard constraint does *not* automatically block anything in a single-zone cluster: `maxSkew` is measured against the *global minimum* across eligible domains, and `minDomains` defaults to `1`. With only one eligible domain its count *is* the minimum, skew is always zero, and every replica schedules there. To actually refuse to cram replicas into too few zones you must set `minDomains` explicitly (e.g. `minDomains: 3`), which is what forces the third replica of a three-replica service to wait rather than pile into a second zone. The usual production pattern is: **hard on zones only if you genuinely have multiple zones, soft on hosts.**
- **`labelSelector`** — which pods are counted. This must match the pods being spread, or the constraint counts nothing and has no effect. A constraint with a selector matching zero pods is silently inert.
- **`minDomains`** — a floor on how many domains must exist. With `minDomains: 3` and `maxSkew: 1` in a two-zone cluster, the third replica will not schedule at all, which is often the intent (it is better to be Pending and visible than to silently pack three replicas into two zones).

**The interaction with rolling updates** is worth knowing, because it causes unexplained rollout stalls — but the cause is narrower than it first appears. `maxSkew: 1` tolerates one extra pod in a domain (three zones at 1/1/1 may legally become 2/1/1), so a surge pod is *not* blocked merely because the cluster is balanced. Rollouts actually stall when (a) **`minDomains` exceeds the number of eligible domains** — the classic case being `minDomains: 3` on a two-zone cluster, where the third replica can never be placed; or (b) **two hard constraints intersect to an empty set**, for example a zone spread and a hostname spread that cannot both be satisfied at once. Diagnose by reading the `Pending` pod's scheduler event, which names the violated constraint. Remedies: reduce `minDomains`, relax the host-level constraint to `ScheduleAnyway`, or reduce `maxSurge`.

## 37.3 Node affinity vs pod affinity: choosing the right tool

These sound similar and do different jobs.

**Node affinity** matches *node labels* — properties of the machine: zone, region, instance type, CPU architecture, GPU model, whether it is spot capacity.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:      # hard
      nodeSelectorTerms:
        - matchExpressions:
            - { key: topology.kubernetes.io/zone, operator: In, values: [eu-west-1a, eu-west-1b] }
            - { key: kubernetes.io/arch, operator: In, values: [amd64] }
    preferredDuringSchedulingIgnoredDuringExecution:     # soft, weighted
      - weight: 100
        preference:
          matchExpressions:
            - { key: node.kubernetes.io/instance-type, operator: In, values: [m6i.2xlarge] }
```

**Pod affinity and anti-affinity** match *other pods* — relative placement. "Run me near the cache" (affinity) or "never run me near another replica of myself" (anti-affinity).

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:      # hard: strictly one per node
      - topologyKey: kubernetes.io/hostname
        labelSelector:
          matchLabels: { app.kubernetes.io/name: api }
```

**Why topology spread is usually better than hard anti-affinity:**

- **Anti-affinity is a boolean per node**, so with 3 replicas and 2 nodes, the third replica can never schedule — it stays `Pending` until you add capacity. Topology spread tolerates the imbalance and places it.
- **Anti-affinity scales badly.** The scheduler must evaluate the rule against every candidate node, and with many pods and many nodes this becomes a measurable scheduling cost. Topology spread is computed more efficiently.
- **Anti-affinity with `requiredDuringScheduling`** is a common cause of "we scaled to 5 and 2 pods are stuck Pending," and the fix is usually to convert it to topology spread or to soft anti-affinity.

**When to still use anti-affinity:** when co-location is genuinely forbidden rather than merely undesirable — for example, two replicas of a quorum-based datastore where co-location would defeat the purpose of replication. Even then, prefer `ScheduleAnyway`-equivalent semantics unless you truly want the Pending pod.

**One warning about `requiredDuringSchedulingIgnoredDuringExecution`:** the "IgnoredDuringExecution" half is doing real work. If node labels change after a pod is placed, the pod is *not* evicted. So if you relabel nodes to move a workload, nothing moves until the pods are recreated. That is usually the desired safety property, but it surprises people who expect affinity to be continuously enforced.

## 37.4 The descheduler: correcting drift over time

The scheduler places each pod **once**. It never revisits the decision. Over a cluster's lifetime this produces predictable drift:

- A node was restarted and came back empty; new pods filled it while the old nodes remain densely packed.
- A workload was scaled up during an incident and scaled back down, leaving pods unevenly distributed.
- A node was cordoned, drained, and uncordoned, and the workloads that moved never came back.
- Pods that violate topology spread (placed before the constraint was added) stay violative forever.

The **descheduler** is a separate component (a Job or CronJob) that finds pods which *would not be placed where they are* if scheduling happened now, evicts them, and lets the scheduler place them properly. It is the maintenance counterpart to the scheduler's one-shot decision.

**The critical caveat: the descheduler evicts pods, and eviction is disruptive.** It must be configured conservatively:

- It honors **PodDisruptionBudgets**, so a workload with a restrictive PDB will not be disrupted — which also means the descheduler may achieve nothing for your most important workloads.
- Run it during low-traffic windows (hence a CronJob rather than a continuous loop) for anything latency-sensitive.
- Enable only the strategies you need. Common ones: `LowNodeUtilization` (balance underused nodes), `RemoveDuplicates` (break up co-located replicas), `RemovePodsViolatingTopologySpreadConstraint` (fix spread violations), `PodLifeTime` (recycle long-lived pods). Each is separately enabled.
- Exclude namespaces and workload types that cannot tolerate eviction — databases, single-replica services, anything with local state.

**Is it necessary?** In a small, stable cluster, no. In a large or long-lived one, or one where node autoscaling consolidates aggressively, yes — and it is the only mechanism that fixes historical placement mistakes without a redeploy.

## 37.5 Priority, preemption, and capacity reservation

Part 6.6 introduced priority classes. Their operational use deserves more detail, because they are how a cluster behaves predictably under contention.

**Priority has two distinct effects, and they are often confused:**

1. **Scheduling order** — when several pods are pending, higher-priority pods are considered first. It does not reserve capacity; it only determines who gets served first.
2. **Preemption** — a high-priority pod that cannot be scheduled may cause **lower-priority running pods to be evicted** to make room. This is disruptive but it is what protects critical workloads when the cluster is full.

That second effect is powerful and dangerous. A misconfigured priority can cause a batch job to evict production. Guidance:

- **Define a small number of tiers and apply them consistently.** For example: `platform-critical` (1000000) for cluster infrastructure, `platform` (100000) for shared services, `app` (10000) for production applications, `batch` (1000) for jobs, and the default (0) for everything else. Two constraints on the numbers: user-defined priorities may not exceed **1000000000**, and the highest built-in values are reserved (`system-cluster-critical` = 2000000000, `system-node-critical` = 2000001000). Also note the naming rule — **a PriorityClass name may not begin with `system-`**, which is why the example above is `platform-critical` rather than `system-critical`; the API rejects the latter.
- **Never leave everything at the default priority.** If all pods are equal, preemption cannot help anything, and under pressure the cluster behaves arbitrarily.
- **Preemption should be rare.** If it happens routinely, the cluster is chronically overcommitted, and the correct fix is capacity — not a priority arms race where every team raises its own numbers.
- **Mark non-preempting critical pods with `preemptionPolicy: Never`** when a pod must have high scheduling priority but must never be the cause of evicting someone else.

## 37.6 Reserving capacity: PriorityClass and placeholder pods

Kubernetes has no "reserved capacity" primitive, so the standard technique is a **placeholder Deployment**: a set of low-priority pods whose only job is to occupy space so that the cluster is never completely full, and which are preempted the instant real work arrives.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: overprovisioning }
value: -10                 # lower than every real workload
globalDefault: false
description: "Placeholder pods that hold capacity for fast scale-up."
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: capacity-buffer, namespace: kube-system }
spec:
  replicas: 4                                    # tune to (expected burst) / (pod size)
  selector: { matchLabels: { app: capacity-buffer } }
  template:
    metadata: { labels: { app: capacity-buffer } }
    spec:
      priorityClassName: overprovisioning
      terminationGracePeriodSeconds: 0           # die immediately when preempted
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.10
          resources:
            requests: { cpu: "1", memory: 2Gi }   # a quarter of a typical node
```

**What this buys:** when a real workload scales up, it is scheduled immediately onto the node the placeholder occupied (the placeholder is preempted), instead of waiting two to five minutes for the node autoscaler to provision a new machine. That delay is otherwise unavoidable, and it is the dominant cause of slow scale-up in workloads that take seconds to start.

**What it costs:** you pay for idle capacity — the placeholder pods' requests occupy real nodes. The trade-off is explicit and worth making only for latency-sensitive services; for batch work, waiting for a node is free.

**A second use of the same mechanism:** placeholders in a specific zone or on a specific node pool keep capacity warm for workloads that need a particular hardware class (GPUs, high-memory instances) where provisioning is slowest.

---

# Part 38 — Ephemeral Storage, Node Pressure, and Resource Exhaustion

Part 2.10 explained what happens when a node runs short — conditions, taints, QoS-ordered eviction. This part covers the resource that causes the most surprising evictions and the practical mechanics of keeping nodes healthy.

## 38.1 The third resource

CPU and memory get all the attention. **Disk is the one that takes down nodes.** Kubernetes tracks two kinds of storage, and the distinction matters:

- **Persistent storage** — volumes backed by a PersistentVolume, which live independently of the pod. Covered in Part 16.
- **Ephemeral storage** — the node-local disk a pod consumes during its lifetime: its containers' writable layers, the log files written to stdout/stderr (which the kubelet stores on disk), and its `emptyDir` volumes. The accounting has two levels, and image layers sit differently in each. For the **container-level limit check**, the kubelet compares the container's writable layer plus its logs against that container's limit — images are not part of that comparison. But images are not simply ignored: when the starved filesystem is `imagefs`, the kubelet attributes image usage to pods when ranking eviction victims. So the accurate statement is that images are excluded from the *limit* comparison but still count against you in the *eviction ranking* when images are the resource under pressure.

Ephemeral storage is a **schedulable resource** in the same sense CPU and memory are — it is a built-in resource type, it appears in a node's allocatable capacity, and the scheduler accounts for a pod's **request** when deciding whether it fits. But almost nobody sets it, and that omission is why `DiskPressure` is a common and confusing node condition: a node with plenty of CPU and memory becomes unschedulable because a single pod wrote 200 GB of logs.

One precision worth stating, because it is easy to get wrong: the scheduler filters on **requests only**, for every resource — it does not check limits. (A limit influences scheduling indirectly only when a request is omitted, because Kubernetes then copies the limit and uses it as the request.)

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
    ephemeral-storage: 1Gi     # reserve this much local disk; the scheduler accounts for this
  limits:
    memory: 512Mi
    ephemeral-storage: 2Gi     # exceed this and the POD IS EVICTED, not throttled
```

**Ephemeral-storage limits are enforced by eviction, not by a cgroup.** CPU and memory limits are enforced by the kernel through cgroups; ephemeral storage has no equivalent. The documented behavior is explicit: if a container's writable layer and log usage exceeds its storage limit, *"the kubelet marks the Pod for eviction"*, and the whole pod is terminated. The comparison with memory is worth keeping straight, because the consequences differ:

| Exceeded | What happens | Scope of impact |
|---|---|---|
| **Memory limit** | The kernel terminates the process (reactively — the docs say the kernel "may" terminate it); the container restarts if its restart policy allows | One container |
| **Ephemeral-storage limit** | The kubelet marks the **whole pod** for eviction; it is terminated and replaced elsewhere | The entire pod |
| **Ephemeral-storage request** | No documented enforcement action | None — requests here affect scheduling only |

That last row needs care, because the obvious reading of it is wrong. Exceeding an ephemeral-storage request does not by itself evict you — but it **does** change your eviction prospects under disk pressure, and the reason is the ranking rule from [Part 2.10](#210-node-pressure-qos-and-eviction) applied to a different resource.

The eviction ranking's first key is "does this pod's usage exceed its request **for the resource that is under pressure**?" When the pressure is `DiskPressure`, the starved resource is node filesystem or image filesystem — and pod usage of that is compared against the **ephemeral-storage request**. So a pod that stays inside its ephemeral-storage request is in the protected tier, and a pod that exceeds it is in the exposed tier. What is true is the narrower statement: the *QoS class* mechanism (Guaranteed/Burstable/BestEffort) is CPU-and-memory based and does not apply here.

So the two numbers have distinct jobs: the **request** affects scheduling *and* determines which eviction tier you fall into under disk pressure; the **limit** is what outright marks the pod for eviction when crossed. Both are worth setting.

Practical guidance:

- **Set requests** so the scheduler reserves disk for the pod. A pod with no ephemeral-storage request is treated as requesting zero, so it can land on a node whose disk is nearly full and immediately trigger pressure.
- **Set a generous limit**, sized to the pod's genuine maximum plus headroom, because the failure mode is pod eviction rather than a single container restart. Under-sizing produces evictions that look like random restarts — which is exactly why they are so often misdiagnosed.
- **The usual culprit is logs.** A pod in a crash loop, or one logging at debug in production, can generate gigabytes per hour. Application-level rotation helps; the durable fix is the kubelet's `containerLogMaxSize` (default `10Mi`) and `containerLogMaxFiles` (default `5`), which are per-node settings, plus not logging at debug in production.

## 38.2 Node eviction thresholds, concretely

A node's "am I running out?" decision is configured on the kubelet with **eviction thresholds**, and understanding them explains why eviction begins at, say, 10% free disk rather than 0%.

```yaml
# kubelet configuration (per node — not a Kubernetes API object)
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
evictionSoft:
  memory.available: "500Mi"
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "1m30s"
evictionMaxPodGracePeriod: 60
```

The distinction between the two categories is the grace period:

- **Hard thresholds** trigger **immediate** eviction. They are the emergency brake and should be set conservatively — low enough that they only fire when the node is genuinely about to fail.
- **Soft thresholds** trigger eviction only after the resource has stayed below the threshold for the **grace period**. This avoids evicting pods during a transient spike (a big file being written and deleted, a cache flush). Soft thresholds are the right tool for "start shedding load before it becomes an emergency."

**The three filesystem signals** are worth knowing because they fail differently:

- **`nodefs`** — the node's root filesystem. Running out means the kubelet cannot write pod logs or container state, and the node becomes unusable.
- **`imagefs`** — the filesystem holding container images and the writable layers. Running out prevents new pods from starting on that node.
- **`nodefs.inodesFree`** — the *count* of files, not their size. A directory with millions of tiny files exhausts inodes while showing plenty of free bytes. This is a genuine failure mode for applications that create many small files (caches, session stores, mail spools) and it presents as "the disk is 40% free but nothing works."

Both `nodefs` and `imagefs` may be the same filesystem; the kubelet detects this and adjusts.

**Why percentage thresholds can be misleading on huge disks:** a node with a 2 TB disk and a 10% threshold will not begin evicting until 200 GB are free — by which point some other process may already have failed. On very large disks, absolute thresholds (e.g. `"50Gi"`) are often more appropriate than percentages. This is a real configuration decision that managed clusters make for you and self-managed clusters do not.

## 38.3 What the kubelet does when pressure hits

The response differs by resource, and the difference matters operationally.

**Under memory pressure:**

1. The node is tainted so no new pods are scheduled (`node.kubernetes.io/memory-pressure:NoSchedule`).
2. The kubelet ranks victim pods — `BestEffort` first, then `Burstable` pods exceeding their requests, and `Guaranteed` last (Part 2.10).
3. Victims are terminated and their controllers recreate them elsewhere. **Node-pressure eviction does not honour the pod's `terminationGracePeriodSeconds`** — a hard threshold terminates immediately (zero grace), and soft thresholds use the kubelet's `evictionMaxPodGracePeriod` instead. This is a genuine difference from the Eviction API and from scheduler preemption, both of which *do* respect the pod's grace period; applications that rely on a long grace period to drain connections will lose in-flight requests when the node evicts them.

**Under disk pressure:**

1. The node is tainted (`node.kubernetes.io/disk-pressure:NoSchedule`).
2. **The kubelet reclaims node-level resources before it touches end-user pods.** The documented order is: first garbage-collect dead pods and containers, then delete unused images (controlled by the image GC high/low thresholds). With a dedicated `imagefs`, `nodefs` pressure triggers dead pod/container GC while `imagefs` pressure triggers image deletion. This step alone frequently resolves the pressure without any eviction, and it is why "the node is full of old images" is so often the real answer.
3. Only if reclaiming node-level resources does not bring the signal back under the threshold does the kubelet evict pods. Its ranking is, in order: **whether the pod's usage exceeds its request**, then **pod priority**, then **how far usage exceeds the request** — with the magnitude of the resource consumed only as a final sort key within a class. So the intuitive "evict the biggest disk consumer first" is not the rule; a small pod well over its request outranks a large pod sitting inside its request. Note also that the QoS-class ordering described in [Part 2.10](#210-node-pressure-qos-and-eviction) is based on CPU and memory and **does not apply to ephemeral-storage requests**.

An important operational consequence: **evictions are not the same as OOM kills**, and confusing them sends you debugging the wrong thing.

| Symptom | Cause | Where to see it |
|---|---|---|
| `OOMKilled` in `lastState.terminated.reason` | Container exceeded its own **memory limit**; the kernel killed the process inside it | `kubectl describe pod`, container `lastState` |
| `Evicted` in pod `status.reason` | The **kubelet** removed the pod to protect the **node** (memory or disk pressure) | `kubectl get pods` shows `Evicted`; `kubectl describe node` shows pressure conditions |
| Pod disappears with no reason | Node was removed from the cluster, taking its pods with it | Node events, cluster autoscaler logs |

`OOMKilled` means "this container is too big for its limit." `Evicted` means "this node is out of room." The first is an application-sizing problem; the second is a capacity or noisy-neighbour problem.

## 38.4 Keeping nodes healthy

A short checklist that prevents most node-pressure incidents:

- **Set requests and limits for CPU, memory, *and* ephemeral storage** in every workload. A LimitRange (Part 23.1) applies defaults to anything that forgets — which is the only reliable way to enforce this across an organization.
- **Cap log growth per node** with kubelet `containerLogMaxSize`/`containerLogMaxFiles`, and ship logs off the node rather than letting them accumulate (Part 19.2).
- **Reserve resources for the system** with `systemReserved` and `kubeReserved`, so that application pods cannot starve the kubelet and the container runtime. A node whose kubelet cannot run is a node that stops reporting and eventually gets its pods evicted anyway.
- **Give DaemonSets their own accounting.** Log collectors, CNI, CSI, and monitoring agents consume resources on *every* node, and they are usually excluded from capacity calculations. A cluster running at 85% requested capacity may effectively be at 95% once DaemonSets are counted.
- **Alert on node conditions**, not just on pod failures: `MemoryPressure`, `DiskPressure`, `PIDPressure`, and `nodefs.inodesFree`. By the time pods are being evicted, the alert is late.
- **Watch for the "everything restarted at once" pattern.** If all pods on one node cycle together, look at node pressure and kubelet eviction — not at the applications, which are victims.
- **Treat `DiskPressure` as a capacity signal, not just a cleanup task.** Recurring disk pressure on the same node type means the workload-to-disk ratio is wrong, and the fix is different node storage sizing, not a bigger cleanup cron.

---

# Part 39 — Node Lifecycle and Maintenance

Part 2.9 introduced cordon, drain, and the voluntary-versus-involuntary distinction. Part 26 covered upgrade sequencing. This part covers the operations you perform on nodes routinely — and the ways they go wrong.

## 39.1 The anatomy of a safe drain

Draining is the fundamental maintenance operation: empty a node so it can be rebooted, patched, resized, or removed. The sequence Kubernetes performs, and the sequence you should expect:

```bash
kubectl cordon <node>                                    # 1. stop new pods landing here
kubectl drain <node> \
  --ignore-daemonsets \                                  # 2. DaemonSet pods cannot move; skip them
  --delete-emptydir-data \                                # 3. accept that emptyDir contents are lost
  --grace-period=120 \                                    # 4. give slow apps time to finish
  --timeout=600s                                          # 5. fail loudly instead of hanging forever
# ... perform maintenance ...
kubectl uncordon <node>                                  # 6. allow scheduling again
```

**What draining actually does:** for each pod on the node, it creates an **Eviction** object rather than deleting the pod. That distinction is the whole point — the eviction API consults **PodDisruptionBudgets** and refuses requests that would violate them. A `kubectl delete pod` ignores PDBs entirely; a drain respects them. This is why draining is safe and deleting pods is not.

**Cordon first, always.** Draining an uncordoned node is a race: the scheduler may place a new pod on the node between the drain emptying it and your maintenance starting. `kubectl drain` cordons automatically, but doing it explicitly makes the intent legible and prevents the mistake of forgetting it on manual workflows.

**`--ignore-daemonsets` is required, not optional.** DaemonSet pods are managed by a controller that wants exactly one per node. Evicting one simply makes the controller recreate it on the same node, so the drain can never finish. Kubernetes refuses to proceed unless you acknowledge this, and the DaemonSet pods keep running throughout the maintenance — which is usually fine (log collectors, CNI, and CSI drivers tolerate a brief interruption) but does mean a node is never truly empty.

**Why `--delete-emptydir-data` exists:** `emptyDir` volumes are stored on the node's disk and are destroyed with the pod. Kubernetes will not silently discard them, so it blocks the drain until you confirm you accept the loss. For pods using `emptyDir` as scratch space (which is the correct use), this is harmless. For a pod that mistakenly keeps state there, this flag is the moment you find out — check before you pass it.

**Drain hangs are usually one of four things**, and the error message names which:

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot evict pod as it would violate the pod's disruption budget` | A PDB with `minAvailable` equal to (or nearly equal to) the replica count | Add replicas, relax the PDB, or accept that this workload cannot tolerate maintenance |
| `cannot delete Pods that declare no controller (use --force to override)` | A bare pod — no Deployment/StatefulSet/Job owns it | Investigate why it exists; `--force` deletes it permanently |
| Waits indefinitely, no error | A pod with a very long `terminationGracePeriodSeconds`, or a `preStop` hook that never returns | Check `terminationGracePeriodSeconds` and the hook; use `--timeout` to fail fast |
| `cannot delete Pods with local storage (use --delete-emptydir-data to override)` | A pod with an **`emptyDir`** volume | Confirm the contents are disposable. Note that this filter tests `emptyDir` only — pods using `hostPath` or a `local` PersistentVolume are **not** blocked by drain and will be evicted normally, even though they are equally pinned to the node |

**The "PDB blocks maintenance" case is the one that matters most**, because it is common and it is a design signal rather than a bug. A three-replica service with `minAvailable: 3` can never lose a pod, so node upgrades are impossible without overriding the PDB. The correct answer is almost always to relax the PDB to `minAvailable: 2` (or `maxUnavailable: 1`) and accept brief reduced redundancy — the alternative is never being able to patch the node.

## 39.2 Graceful termination, in the correct order

Part 6.2 described the shutdown sequence. The subtlety worth restating here, because it causes 502s on every deploy, is that **the steps are concurrent, not sequential**:

- The pod is marked `Terminating` and, at the same time, begins being removed from Service endpoints (`EndpointSlice`).
- `preStop` runs, and the container keeps running.
- After `preStop` completes, `SIGTERM` is sent to the container's main process.
- After `terminationGracePeriodSeconds`, `SIGKILL` is sent regardless.

**Endpoint removal is asynchronous and propagates at different speeds to different consumers.** Every reverse proxy, kube-proxy, and load balancer learns about the removal on its own schedule — often seconds later. So a pod can receive `SIGTERM`, stop accepting connections, and exit *while a load balancer is still sending it traffic.* That window is the source of the classic "we deploy and lose a fraction of a percent of requests."

Three mitigations, and production setups usually need all three:

1. **A `preStop` sleep** long enough for endpoint propagation — commonly 5–15 seconds. It looks like a hack and is in fact the standard remedy, because it delays shutdown until the rest of the system has noticed.
2. **Readiness that flips to false on shutdown**, so the endpoint is actively withdrawn rather than passively expiring. This requires the app to react to `SIGTERM` by failing its readiness probe while continuing to serve in-flight requests.
3. **Applications that finish in-flight requests** rather than exiting immediately on `SIGTERM`.

`terminationGracePeriodSeconds` must exceed the slowest in-flight request plus the `preStop` duration, or the process is `SIGKILL`ed mid-request and the client sees a reset.

## 39.3 Node autoscaling and the maintenance interaction

Node autoscalers (cluster autoscaler, Karpenter) both add and **remove** nodes. Removal is where maintenance concerns appear:

- **Consolidation** — Karpenter and the cluster autoscaler actively move workloads off underutilized nodes to remove them. This is a continuous, automated drain. It respects PDBs, which means a workload with a restrictive PDB will keep an otherwise-empty node alive indefinitely (and cost money). If nodes are not being consolidated as expected, check PDBs first.
- **Spot interruption** — spot capacity is reclaimed with roughly a two-minute notice. Handling it requires the notice to be acted on: the **AWS Node Termination Handler**, or Karpenter's built-in handling, cordons and drains the node before it disappears. Without it, spot reclaims appear as pods vanishing, and anything without a controller is lost.
- **Scale-to-zero risks** — a scale-down that removes the last node of a given type can strand pods that require that type (`nodeSelector` for a GPU, say). Autoscalers handle this, but a pod with a hard node affinity and no matching node pool will sit `Pending` while the autoscaler cannot help it. The event message says so explicitly.

**Check the autoscaler's own logs when capacity behaves strangely.** "I scaled up but no nodes appeared" is answered there, not in the application: common reasons are an instance type unavailable in that zone, an account quota exhausted, or a pod whose resource requests no instance type can satisfy (a classic: a pod requesting 200 GB of memory in a cluster whose largest instance has 128 GB — it will never schedule, and no amount of autoscaling will fix it).

## 39.4 Removing a node permanently

The order matters, because getting it wrong leaves orphaned cloud resources that cost money:

1. **Cordon and drain** the node.
2. **Confirm the workloads landed elsewhere** — check that every controller reports its desired replica count as available, and that nothing is stuck `Pending` for lack of capacity.
3. **Remove the node from the cluster** — `kubectl delete node <name>` for a self-managed node, or terminate the instance through the provider so its cloud resources are cleaned up.
4. **Verify provider-side cleanup.** Deleting the `Node` object does *not* delete the underlying VM, and terminating the VM does *not* necessarily detach and delete its disks. Unattached cloud disks are one of the most common sources of quiet, recurring cloud spend.
5. **Check for leaked PersistentVolumes** whose pods are gone but whose disks remain — `kubectl get pv | awk '$5=="Released"'`.

**Nodes that re-register after deletion** indicate the kubelet is still running on the machine and its bootstrap credentials are still valid. It will rejoin as a fresh, empty node. For self-managed clusters, stop the kubelet and, if the node was removed because it is being decommissioned, revoke its certificate.

## 39.5 Node image and kernel management

Kubernetes version upgrades are only half of node maintenance, and often the less frequent half.

- **OS and kernel patching cadence** should be monthly at minimum. Container images are pinned and immutable, but the *node kernel* is shared by every container on the machine, and a kernel vulnerability is a cluster-wide exposure regardless of how well your images are pinned.
- **Immutable node images** (Talos, Bottlerocket, Flatcar, or a managed AMI) make patching a matter of replacing nodes rather than mutating them in place. This is strictly better operationally: a failed patch rolls back by not replacing the node, and there is no configuration drift between machines. It is the direction the industry has moved.
- **Kubelet configuration drift** is a real problem on hand-managed fleets. Use a configuration management tool or, better, immutable images so that every node is identical by construction. Kubernetes 1.35 promoted a kubelet configuration drop-in directory to GA, which makes layered kubelet config more manageable.
- **Node restarts are not free even when the cluster is healthy.** Draining a node reshuffles its pods, which can momentarily reduce a service's capacity below its comfortable margin. Do node maintenance in waves, not all at once, and watch error rates during the operation — that is the only way to know if your PDBs and replica counts are actually adequate.

---

# Part 40 — etcd and Control Plane Health

Part 2.3 introduced etcd as the cluster's database. For managed clusters this part is background reading, because the provider operates it. For self-managed clusters it is the difference between a cluster that survives a bad week and one that does not — **etcd is the only stateful component whose loss is unrecoverable.** Everything else can be rebuilt from git, manifests, and backups; etcd *is* the record.

## 40.1 Why etcd is different from every other component

Every other control-plane component is stateless and replaceable. etcd is a distributed consensus system, and consensus imposes rules that must be respected rather than worked around.

**Quorum is a majority, and it is a hard requirement.** With three members, two must agree before any write is accepted; one member can fail and the cluster continues. With two members, a majority is two — so losing either one stops all writes. This is why member counts are odd: **2 is strictly worse than 1 or 3.** Run 3 for ordinary production and 5 only if you need to survive two simultaneous failures, because each member adds write latency (every write must be acknowledged by a majority) and operational surface.

**Losing quorum is not data loss, it is an outage.** With quorum lost, the surviving members refuse writes and the API server cannot function. Recovery means restoring a member or restoring from snapshot. There is a documented procedure for recovering a cluster after quorum loss by restoring one member from a snapshot and forcing a new cluster — it works, but it is stressful and you should have read it *before* you need it.

**etcd is latency-sensitive, not throughput-hungry.** Its performance is dominated by **fsync** — the guarantee that a write has reached durable storage. This makes etcd exceptionally sensitive to slow or shared disks in a way that is disproportionate to its modest resource usage:

- **Use dedicated, low-latency SSDs** for etcd. Network-attached storage with variable latency is a common cause of mysterious apiserver slowness.
- **Do not co-locate etcd with other I/O-heavy workloads.** A busy container on the same disk can double etcd's write latency.
- **The key metric is `etcd_disk_wal_fsync_duration_seconds`.** If its 99th percentile climbs into the tens of milliseconds, the cluster's responsiveness degrades, and the fix is storage, not tuning. (etcd's own hardware guidance is expressed in IOPS and latency terms rather than a single numeric threshold, so treat any specific figure — including commonly cited ones — as a rule of thumb and baseline against your own cluster.)

**The database has a size limit.** etcd's default quota is 2 GB (configurable, but raising it is rarely the right answer). Events are the usual culprit: a cluster with flapping workloads generates enormous numbers of event objects. Kubernetes mitigates this with event aggregation and default TTLs, but on busy clusters you still must verify that events are being pruned and that the API server's event TTL matches your expectations. A full etcd refuses all writes, which takes the entire cluster down.

## 40.2 The maintenance tasks nobody schedules

**Defragmentation.** etcd never returns freed space to the filesystem on its own. Deleting objects frees logical space inside the database file, but the file stays the same size. Over months, an etcd database can grow to several times its logical content. **Defragmentation reclaims that space** and is a required periodic task:

```bash
# On each etcd member, one at a time — defrag blocks the member briefly
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag

# Check size and health afterwards
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 ... endpoint status --write-out=table
```

Defrag one member at a time and let the cluster return to health between members. Defragmentation blocks reads and writes on the member it runs on, so running it on every member at once produces a cluster-wide **outage or severe latency spike** — not a loss of Raft quorum (membership and consensus state are untouched), but from an operator's perspective in that moment the difference is academic.

**Compaction.** etcd keeps a revision history so that watchers can catch up. Old revisions accumulate and consume space. Kubernetes requests compaction automatically every five minutes by default (configurable), and compaction is what makes old revisions eligible for deletion — but only defragmentation actually reclaims the disk. The two go together, which is why "we compact, why is the file still huge?" has the answer "compaction makes the space free; defrag releases it."

**Watch for `etcd_server_quota_backend_bytes`** and the `NOSPACE` alarm. When etcd hits its quota it raises a **NOSPACE alarm and refuses all writes** until the alarm is cleared — which requires freeing space (usually by defragmenting after compaction). Recovering from it takes three steps, and the third is the one people miss: **compact**, then **defrag** (compaction frees logical space; only defrag returns it to the filesystem), then **`etcdctl alarm disarm`** — the alarm stays set and the cluster stays in read-only maintenance mode until it is explicitly cleared. This is a cluster-wide outage triggered purely by database housekeeping, and it is entirely preventable with monitoring.

## 40.3 Backups, and the honesty required around them

The backup command, the restore procedure, and the discipline of testing them are covered in Part 27. Two points bear repeating because they are where self-managed clusters actually fail:

**A snapshot must come from a healthy member, and the snapshot is only as good as its restoration.** The workflow is: snapshot, move the snapshot off the machine, restore it into a scratch cluster, and confirm the API responds and objects are present. A snapshot file that has never been restored is an untested assumption. On a GitOps-managed cluster this matters slightly less (most objects are in git), but it does not stop mattering: dynamically provisioned PVs, objects created outside git, and cluster-scoped configuration are all in etcd and nowhere else.

**Restoring etcd is not a partial operation.** Restoring an old snapshot reverts *everything* — including objects created after the snapshot. If the snapshot is six hours old, you lose six hours of every change made to the cluster, not just the thing you were trying to recover. This is the single most important reason to keep the time between snapshots short, and to prefer recovering individual objects (from git, or from a Velero backup) whenever the cluster itself is healthy.

## 40.4 Control plane capacity and overload

Two failure modes that look like "the cluster is broken" and are actually control-plane overload:

**The API server is being hammered.** Symptoms: `kubectl` commands time out, controllers fall behind, and everything feels slow. Causes: a controller with a list-watch bug, a client with no rate limiting, a CronJob that lists every pod in the cluster every minute, or a genuinely huge number of objects. Diagnosis starts with the API server's own metrics — request rate, latency, and the count of in-flight requests by verb and resource — which identify the caller. **Part 41 covers API Priority and Fairness**, the mechanism that prevents one client from starving everyone else.

**The cluster is too large for its control plane.** Very large clusters (thousands of nodes, hundreds of thousands of pods) hit real limits in etcd, the API server's watch caches, and the scheduler. Managed providers publish scaling targets and will throttle you beyond them. Self-managed clusters need deliberate tuning: **sharded list-and-watch** (server-side sharding, available in recent versions) and careful controller cache design are what make large clusters viable, and both are active areas of Kubernetes development.

**Practical monitoring for a self-managed control plane** — these are the signals that predict trouble:

| Signal | Why it matters |
|---|---|
| `etcd_disk_wal_fsync_duration_seconds` p99 | Storage latency; the leading indicator of etcd trouble |
| `etcd_server_leader_changes_seen_total` | Frequent leader elections mean network or disk instability |
| `etcd_mvcc_db_total_size_in_bytes` vs quota | Approaching the quota means an imminent write refusal |
| `etcd_server_has_leader` | No leader = cluster is down |
| `apiserver_request_duration_seconds` by verb/resource | Which operations are slow, and for whom |
| `apiserver_flowcontrol_*` | Whether API Priority and Fairness is shedding load (Part 41) |
| `scheduler_pending_pods` | Pods waiting to be scheduled — capacity or constraint problem |
| `controller_manager` workqueue depth | Whether controllers are falling behind events |
| Control-plane certificate expiry | Silent expiry takes the cluster down (Part 26.3) |

---

# Part 41 — API Priority and Fairness

Most guides omit this. It matters because it addresses a failure mode that is otherwise very hard to defend against: **one misbehaving client taking down the entire control plane.**

## 41.1 The problem: the API server has finite capacity

Every actor in a cluster goes through the API server: kubelets reporting status, controllers reconciling, the scheduler placing pods, autoscalers reading metrics, metrics scrapers scraping, and humans running `kubectl`. The API server has a bounded number of workers and a bounded amount of memory, so under enough load it queues, then slows down, then times out.

The dangerous property is **positive feedback**. When the API server slows down:

- Controllers fall behind and retry more aggressively, adding load.
- Watch connections drop on timeout and are re-established, and each reconnection triggers a fresh **list** — the most expensive operation the API server performs, because it serializes every matching object.
- More clients time out and retry.

The result is a stampede that does not self-correct: the load caused by the slowdown makes the slowdown worse. This is a **retry storm against the control plane**, and it is how clusters have total outages that begin with something as mundane as a misconfigured monitoring agent.

**Concrete causes that have taken down real clusters:**

- A monitoring system listing all pods cluster-wide every 15 seconds.
- A custom controller watching a high-churn resource with a full list on every reconnection.
- A CronJob that calls the API in a loop, for every namespace.
- An HPA using a custom metrics adapter that queries the API per pod per scaling decision.
- A `kubectl get --watch` left running in a shell script on a cron.

## 41.2 The mechanism: PriorityLevelConfigurations and FlowSchemas

**API Priority and Fairness (APF)** is the built-in defense. It is enabled by default in modern Kubernetes, and it works like a queueing system with several lanes:

- A **FlowSchema** classifies each incoming request into a **priority level**, based on who is calling (user, group, service account) and what they are doing (verb, resource, namespace). Matching is first-match-wins in `matchingPrecedence` order, so more specific rules come first.
- A **PriorityLevelConfiguration** defines the capacity of that lane: a proportional **share** of the server's total concurrency budget (`nominalConcurrencyShares` — not a fixed request count; the level's limit is derived from that share and adjusted dynamically as levels lend and borrow concurrency), how many may queue (`limitResponse.queuing.queues` and `queueLengthLimit`), and what happens when the lane overflows — **queue** or **reject immediately with HTTP 429**.

Kubernetes ships with sensible defaults, including a set of well-known levels:

| Default priority level | What lands there | Behavior |
|---|---|---|
| `system` | Non-health requests from the `system:nodes` group — i.e. **kubelets** | High concurrency |
| `node-high` | Kubelet status updates | High concurrency |
| `leader-election` | Controller leader election | Protected — otherwise the control plane loses leadership under load |
| `workload-high` | Requests from **built-in control-plane controllers** (kube-controller-manager, the scheduler) | Substantial but bounded |
| `workload-low` | Requests from **any other service account** — i.e. controllers running in your own pods | Lower |
| `global-default` | Anything unmatched (including most human `kubectl`) | Modest |
| `exempt` | The `system:masters` group and other explicitly exempted traffic | **The only level subject to no flow-control limit at all** |

**Where human `kubectl` traffic lands depends on who you are.** `global-default` is the level for interactive `kubectl` commands run by **non-privileged** users, and it is deliberately modest — so a developer or automation using a normal binding can be throttled during control-plane overload while system controllers keep functioning. That is by design: the alternative is that debugging makes the outage worse. But there is an important asymmetry: the mandatory `exempt` FlowSchema classifies **all** requests from the `system:masters` group, which is what the usual cluster-admin kubeconfig belongs to. An operator holding cluster-admin credentials is therefore **not** throttled, even during an overload that is starving everyone else — which is useful during an incident, and also means cluster-admin access bypasses a protection you may be relying on. Both facts are worth knowing before you need them.

## 41.3 Using APF in practice

For most clusters, the defaults are correct and the useful actions are *diagnostic* rather than configurative.

**Detect that APF is shedding load.** This is the signal that you are being protected rather than merely slow:

```
apiserver_flowcontrol_rejected_requests_total     # requests rejected with 429
apiserver_flowcontrol_current_inqueue_requests    # queue depth per priority level
apiserver_flowcontrol_dispatched_requests_total
apiserver_flowcontrol_nominal_limit_seats         # the lane's capacity
```

**Client-side symptom:** an error like `the server has received too many requests and has asked us to try again later` (HTTP 429). When controllers log this, the cluster is protecting itself from them — the fix is to fix the client's request pattern, not to raise limits.

**Add a dedicated lane for a known-heavy, low-priority client.** If a batch analytics job needs to list pods continuously, give it its own low-priority level and FlowSchema so it is throttled *independently* and cannot consume the queue that controllers depend on:

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata: { name: analytics-low }
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 5          # a small slice of total capacity
    limitResponse:
      type: Queue
      queuing:
        queues: 8
        queueLengthLimit: 16             # beyond this, reject rather than queue forever
        handSize: 4
---
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata: { name: analytics-namespace }
spec:
  matchingPrecedence: 800                # lower number = evaluated earlier
  priorityLevelConfiguration:
    name: analytics-low
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount: { name: analytics, namespace: batch }
      resourceRules:
        - verbs: ["get", "list", "watch"]
          apiGroups: ["*"]
          resources: ["*"]
          clusterScope: true
```

**Guidance, in order of importance:**

1. **Do not raise the limits of `global-default` as a first response to 429s.** That removes the protection and lets a single client starve everything. Find the client generating the load.
2. **Protect `leader-election` and system levels.** Never repurpose or narrow them; controllers losing leadership during overload is a cascading failure.
3. **Grade clients by design, not by request.** A dashboard or analytics tool should be in a lower lane than a controller responsible for availability.
4. **Recognize that APF is a backstop, not a capacity plan.** It limits the blast radius of a bad client; it does not make an undersized control plane adequate.
5. **Watch for the controllers whose reconnection causes full lists.** Where possible, use **watch-based** patterns with resource versions (or the newer watch-list / streaming-list features) rather than periodic full lists. The difference between `list` and `watch` under load is enormous.

## 41.4 Adjacent: how to not be the bad client

APF contains clients that misbehave, but the cheaper option is to not misbehave:

- **Use informers/caches** rather than polling. Client libraries provide shared caches that maintain a single watch per resource and serve reads from memory. Hand-rolled `client.Get`/`List` loops in a controller are the classic mistake.
- **Set client-side rate limits** (`QPS`/`Burst` in client-go) deliberately. Infinite retry at full speed turns a transient failure into a storm.
- **Use field and label selectors** so a list returns less. `kubectl get pods -A` for a human is fine; doing it from a loop is not.
- **Avoid `--watch` in scripts without a monitor.** A leaked watch holds a connection and may re-list repeatedly.
- **Respect retry-after responses** rather than retrying immediately.
- **Paginate large lists** (`--chunk-size`) so a single list does not hold a huge response in memory on both sides.

---

# Part 42 — Advanced Networking: CIDR Planning, Dual-Stack, Session Affinity, and MTU

Part 9 covered the fundamentals: the CNI, DNS, Services, and NetworkPolicy. This part covers the networking decisions that are painful to change later and the failure modes that produce the strangest symptoms.

## 42.1 CIDR planning: the decision you cannot easily reverse

Every pod and every Service needs an IP address from a range that you (or your provider) choose when the cluster is created. Changing these ranges later means rebuilding the cluster. Getting them wrong produces problems that only appear at scale.

There are three distinct address spaces:

| Space | Used for | Typical size | Who assigns it |
|---|---|---|---|
| **Node / cluster network** | The machines themselves | Provider-managed | Cloud VPC or your own network plan |
| **Pod CIDR** | Every pod IP | `/16` or larger is common | CNI, from the cluster configuration |
| **Service CIDR** | The virtual IPs of Services | `/16` or `/12` | Cluster configuration |

**Sizing the pod CIDR.** The question is not "how many pods do I have today" but "how many can this cluster ever have." A `/24` gives 256 addresses — exhausted by a single modest node. A `/16` gives ~65,000. Guidance:

- **A pod IP is consumed for the pod's entire life, not per connection.** Pods are cheap and numerous; scaling a Deployment to 200 replicas consumes 200 addresses.
- **Headroom must cover peak, not average.** A cluster that normally runs 2,000 pods can briefly run 6,000 during a deploy storm or an incident.
- **Count the DaemonSets.** Every node runs CNI, CSI, log, and monitoring pods — commonly 5–10 pods per node before any application runs.
- **Leave room for expansion.** A pod CIDR cannot be enlarged in place. `/16` on IPv4 is the usual safe default; on IPv6, plan a `/48` or larger delegation so you never think about it again.

**Sizing the Service CIDR.** Smaller needs, since Services are far fewer than pods — but the same irreversibility applies. It must not overlap the pod CIDR or the node network.

**Overlap is the classic self-inflicted wound.** If the pod CIDR overlaps your corporate VPN range, the on-premises network, or another cluster you intend to peer with, pods become unreachable from those networks in confusing and selective ways (some addresses work, some do not). **Draw the whole address plan on paper before creating the cluster**, including every other cluster, VPN, and peered network you will ever connect to.

**Per-node pod subnets.** Most CNIs carve the pod CIDR into a per-node slice (a `/24` per node is common), which caps pods per node at 254 and means a node gets its slice whether or not it needs it. Some CNIs support dynamic allocation instead; some cloud CNIs allocate pod IPs directly from VPC subnets, in which case **subnet size, not the pod CIDR, is your real limit** (this is the AWS VPC CNI's characteristic constraint, Part 20.4).

## 42.2 IPv6 and dual-stack

Kubernetes has supported dual-stack (IPv4 and IPv6 simultaneously) as a stable feature for several major versions, and dual-stack has been enabled by default since 1.21. IPv6-only clusters are supported by the major Linux CNIs and cloud providers, but the claim is not universal: **Windows nodes do not support single-stack IPv6-only networking**, and some CNIs (Flannel, for example) are IPv4 fabrics. Verify support for your specific CNI and node OS before planning an IPv6-only cluster. The practical considerations:

**Why dual-stack rather than IPv6-only:** during a long migration you need both; some dependencies and clients remain IPv4-only for years.

**What dual-stack changes:**

- **Every pod gets two addresses**, one per family — doubling address consumption in both spaces.
- **Services can be `SingleStack`, `PreferDualStack`, or `RequireDualStack`** (`spec.ipFamilyPolicy`), and a Service's `clusterIPs` list holds one address per family. A Service with a single-stack ClusterIP is **not** reachable from the other family — a pod with only an IPv6 address cannot reach an IPv4-only ClusterIP, and the failure looks like a timeout.
- **`kube-proxy` and the CNI must both be configured for dual-stack.** Mismatched configuration is a common cause of "IPv6 works between pods but not to Services."
- **DNS returns both families** (A and AAAA records). Clients choose by their own happy-eyeballs logic, so *which* address a connection uses is a client decision you do not fully control. Some clients handle a broken IPv6 path badly, producing intermittent slowness rather than a clean failure.
- **Node addressing matters as much as pod addressing.** A dual-stack cluster whose nodes only have IPv4 addresses will have IPv6 pods with no working egress path.

**The honest guidance:** if your cloud network and CNI support it cleanly, dual-stack costs little and avoids a future migration. If it requires contortions (custom node networking, an unsupported CNI combination), **IPv4-only with generous CIDR headroom is the pragmatic choice** — and note that overrunning a `/16` of pod addresses is a more distant problem than most teams imagine.

## 42.3 Session affinity, and why it is usually a design smell

Session affinity (`spec.sessionAffinity: ClientIP`) makes a Service send requests from the same client IP to the same backend pod. It exists, and it is frequently misapplied.

**The core problem, restated from Part 10.1:** a Service is L4. Affinity at L4 is therefore **per client IP**, which behaves badly behind NAT (thousands of users behind one corporate gateway all land on one pod) and badly on mobile networks (a phone's apparent IP changes as it moves between cells, scattering its requests). It also fights autoscaling: a newly created pod receives no affinity-matched traffic.

**When `ClientIP` affinity is defensible:**

- A legacy application that keeps session state in pod memory and cannot be changed quickly — as a temporary measure with an explicit end date.
- WebSocket or long-lived connections where reconnection cost dominates — though in that case the connection is naturally sticky already, because L4 pinning does it for you.
- Cache-warming scenarios where a client benefits from always reaching the same warmed backend — better solved with consistent hashing at L7.

**What to do instead**, in order of preference:

1. **Make the application stateless.** Move session state to a shared store (Redis/Valkey, a database) so any pod can serve any request. This is the correct answer in the overwhelming majority of cases and removes an entire class of scaling and reliability problems.
2. **Terminate the session at the reverse proxy** using a cookie rather than an IP, which survives NAT and IP changes. Every L7 proxy supports cookie-based affinity, and Gateway API intends to express it.
3. **Use consistent hashing at L7** if the goal is cache locality rather than "sessions" — routing a key to a stable backend deliberately.

`sessionAffinityConfig.clientIP.timeoutSeconds` controls how long the affinity lasts. Note that a long timeout combined with rolling updates produces a specific failure: clients stay pinned to a pod that is terminating, and the pod's removal from endpoints makes those clients fail until the timeout expires.

## 42.4 MTU: the cause of the strangest symptom in container networking

**MTU (maximum transmission unit)** is the largest packet a network path will carry. Inside a tunnel (VXLAN, Geneve, IPsec, WireGuard — all common CNI overlays), each packet carries extra encapsulation headers, so the usable MTU inside the tunnel is *smaller* than on the underlying network.

**The symptom when this is misconfigured is distinctive:** small requests work perfectly, and large ones hang or fail. `curl` to a health endpoint succeeds; a POST with a large body times out; a TLS handshake works (small packets) but the first real data transfer stalls. It is intermittent, it is size-dependent, and it is easy to mistake for an application bug.

Why it happens: when a packet exceeds the path MTU, it must be fragmented or the sender must be told to reduce size (ICMP "fragmentation needed"). Many clouds and networks **drop ICMP**, so the sender never learns and simply retries the oversized packet forever. Inside a tunnel, this is easy to trigger because the effective MTU is lower than everything else on the network believes.

**Diagnosis and fix:**

```bash
# Find the largest packet that gets through, from INSIDE a pod
kubectl run nettest --rm -it --image=nicolaka/netshoot --restart=Never -- \
  ping -M do -s 1472 <target>          # 1472 + 28 bytes of headers = 1500 MTU
# Reduce -s until it works; that reveals the real path MTU.
```

The fix is to configure the CNI's MTU to match the underlying network minus encapsulation overhead. Every CNI documents the formula (for example, a VXLAN overlay on a 1500-byte network typically uses 1450). **Getting this wrong is a configuration error, not a network fault** — and it is worth verifying immediately after installing a CNI, because it will otherwise surface weeks later as an unexplained intermittent failure.

## 42.5 Service topology and cross-zone traffic

Two settings influence where traffic goes and what it costs:

**`trafficDistribution`** (the modern successor to the topology-aware routing hints annotation) tells Kubernetes to prefer closer endpoints. The current values are **`PreferSameZone`** and **`PreferSameNode`**; `PreferClose` is a deprecated alias for `PreferSameZone`, and it is the one you will find in most older documentation and blog posts — prefer the newer names. `PreferSameZone` tells Kubernetes to prefer endpoints in the client's zone. This reduces cross-zone latency and, in cloud environments, avoids **cross-zone data transfer charges** — which are a genuine and often surprising line item for chatty services. It is a *preference*: if no local endpoint is healthy, traffic crosses zones rather than failing. Use it on internal services that call each other frequently.

**`internalTrafficPolicy: Local`** restricts a Service's endpoints to pods on the same node. This is rarely what you want (it breaks when a node has no local replica), but it is the correct tool when you specifically want to avoid any network hop for a node-local agent — and it makes a Service fail on nodes with no matching pod, which must be a deliberate choice.

**Where these matter most:** service meshes and per-request proxies amplify cross-zone traffic because every hop is a connection. A mesh that doubles the number of network hops also doubles the cross-zone charge for any traffic that leaves its zone.

---

# Part 43 — Server-Side Apply, Field Ownership, and Generated Manifests

Most guides treat `kubectl apply` as a single thing. It is actually two different mechanisms, and the difference explains a family of confusing failures: fields that keep reverting, fields that cannot be removed, and tools fighting over the same object.

## 43.1 The two apply mechanisms

**Client-side apply (the classic behavior).** The client (`kubectl`) reads the live object, merges your file into it locally, and sends the complete result as a `PATCH`. To do this it needs a record of what it previously sent, so it stores the last-applied configuration in an annotation (`kubectl.kubernetes.io/last-applied-configuration`).

Problems with this model:

- **The client must read before writing**, so it is subject to races: two clients applying simultaneously can clobber each other's changes.
- **Conflicts are resolved by whoever writes last**, silently. The precise rule matters, because it is widely misstated: client-side apply clears a field only when that field *is* present in your last-applied configuration and now missing from your file. A field that **is missing from both** your file and your last-applied annotation is left untouched — which is exactly why a controller-added field survives a naive `kubectl apply`. What breaks is the reverse case: you once applied a field, then removed it from your YAML, and now client-side apply deletes it from the live object even if a controller has since taken it over.
- **The bookkeeping annotation inflates objects** and is itself a field that can be lost.
- **Nothing tracks ownership.** There is no way to ask "who set this field?"

**Server-side apply (SSA).** You send *only the fields you care about*, and the **API server** performs the merge, tracking which manager owns which field. Ownership is recorded in `metadata.managedFields` — a list of managers, the fields each owns, and whether each applied or updated them.

Consequences that make SSA better for automation:

- **No read-before-write**, so no race.
- **Field-level ownership**: if two managers both set `spec.replicas`, the API server reports a **conflict** rather than silently letting one win. That is a feature — it surfaces a real problem instead of hiding it.
- **`managedFields` becomes the record of ownership**, and can be inspected to answer "who set this?" Note that `kubectl apply --server-side` *still maintains* the `last-applied-configuration` annotation (that is deliberate — it is what lets you drop back to client-side apply). Non-kubectl clients such as Argo CD and controllers do not write it.
- **Removal is explicit**: dropping a field from your applied file means you stop owning it, and if no one else owns it, it is removed. This is what makes SSA predictable for GitOps.

```yaml
# HPA owns spec.replicas; git owns the rest. With SSA, neither clobbers the other.
```

`kubectl apply --server-side` enables it, and most GitOps tools use SSA for good reasons: Argo CD and Flux both support it, and it is the recommended mode for large or heavily-controller-managed objects.

**When SSA causes trouble:** a conflict error (`Apply failed with 1 conflict: conflict with "kubectl" using apps/v1: .spec.replicas`) means another manager owns a field you are trying to set. The resolution options are, in order of preference: (1) stop managing that field from the losing side — this is usually the correct fix, and for the HPA case it means removing `replicas` from your Deployment manifests; (2) take ownership explicitly with `--force-conflicts`, which is correct when you genuinely are the new owner; (3) rename the field manager if a tool changed names.

**Inspecting ownership** is the skill that makes SSA debuggable:

```bash
# Who owns what on this object?
kubectl get deployment api -o jsonpath='{.metadata.managedFields}' | jq .
# Look at fields owned by a specific manager
kubectl get deployment api --show-managed-fields -o yaml | less
```

**Do not treat `managedFields` as noise.** It is the record of which tool is responsible for which field, and it is the only reliable way to answer "why does this keep reverting?"

## 43.2 Why fields keep reverting

The most common GitOps complaint is "I changed it, and it changed back." There are four distinct causes, and identifying which one you have determines the fix:

| Symptom | Cause | Fix |
|---|---|---|
| `spec.replicas` reverts to the git value | **HPA also manages it.** Two owners, one field. | Remove `replicas` from your manifests (or add `ignoreDifferences`) — Part 25.1 |
| A field you set disappears | A **controller or admission webhook** owns it (defaulted fields, injected sidecars) | Check `managedFields`; add the field to the tool's ignore list, or stop setting it |
| An object is permanently "OutOfSync" | The cluster defaults a field your manifest omits | Configure `ignoreDifferences` for it |
| Your change is rejected as a conflict | Another manager owns the field (SSA) | Resolve ownership deliberately, then optionally `--force-conflicts` |

For Argo CD specifically, an Application can be told which differences are not meaningful:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers: ["/spec/replicas"]        # HPA owns this
    - group: ""
      kind: Service
      jqPathExpressions: ['.spec.clusterIP']  # assigned by the cluster, immutable
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

`jsonPointers` and `jqPathExpressions` are two ways to name the same thing; `jqPathExpressions` is more expressive for nested or conditional paths. `RespectIgnoreDifferences=true` matters: without it, Argo CD still applies the ignored fields during a sync.

**The general principle:** a field should have exactly one owner. When two systems both want to write a field, decide which one owns it and remove it from the other's manifests. Fighting it with ignore rules is workable but leaves the ambiguity in place.

## 43.3 Declarative configuration hygiene

Small practices that prevent large amounts of confusion, all of them the direct consequence of the reconciliation model:

- **One owner per object.** Terraform *and* Argo CD managing the same Kubernetes resource means they will overwrite each other forever. Pick one owner per layer (Part 3.1).
- **Do not `kubectl edit` production objects.** It creates drift, is invisible to review, and the next GitOps sync will revert it — which then looks like a bug. If you must change something urgently, change it in git and let the controller apply it, or use a documented break-glass procedure (Part 21.6).
- **Declare every field you care about, and leave the rest to defaults.** Defaults are stable within a version but can change across versions, so critical fields (security contexts, resource limits, probes) should be explicit.
- **Do not commit generated output.** Render Helm/Kustomize in CI or let the GitOps controller render it. Two copies of the truth is one too many.
- **Do not assume the `last-applied-configuration` annotation disappears under SSA.** `kubectl apply --server-side` keeps it updated; tools that are not kubectl generally do not write it. Its presence tells you a kubectl client has been applying, not that client-side apply is in use.
- **Version your ConfigMaps rather than mutating them** (Part 8.1), so configuration changes appear as explicit rollouts in git history.

---

# Part 44 — Nodes: Runtimes, Isolation, and Heterogeneous Clusters

Part 2.4 introduced the container runtime. This part covers what you do when a single cluster must run workloads with different isolation requirements, different CPU architectures, or different operating systems.

## 44.1 RuntimeClass: selecting the isolation level per workload

By default every pod on a node uses the same container runtime. **RuntimeClass** lets you name alternative runtimes and select one per pod.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata: { name: gvisor }
handler: runsc                     # the runtime's handler name, configured in the node's CRI
overhead:                          # resources the runtime itself consumes — the scheduler accounts for these
  podFixed: { cpu: 250m, memory: 32Mi }
scheduling:
  nodeSelector: { sandbox: gvisor }   # only nodes that have this runtime
  tolerations: [{ key: sandbox, operator: Exists }]
---
spec:
  runtimeClassName: gvisor          # on the pod
```

**Why you would choose a different runtime**, and what each option actually is:

| Runtime | What it is | Isolation strength | When to use |
|---|---|---|---|
| **runc** (default) | The standard OCI runtime; containers share the host kernel | Weakest (kernel is shared) | Everything trusted |
| **gVisor** (`runsc`) | A **user-space kernel** that intercepts system calls and implements them in userspace, so the container never calls the host kernel directly | Strong, at a performance cost | Running untrusted or third-party code where a container escape must be prevented |
| **Kata Containers** | Each pod runs inside a **lightweight virtual machine**, so it has its own kernel | Strongest of these; highest overhead | Multi-tenant platforms, untrusted code, compliance-driven isolation |
| **Wasm** runtimes | Confined, fast-starting WebAssembly environments | Different model entirely (no Linux syscalls) | Specific function-style workloads |

**The decision rule:** container isolation is a shared-kernel boundary (Part 1.2), which is good enough when you trust the code — your own images, your own code — and not good enough when you do not. Running arbitrary user-supplied or AI-generated code on `runc` in a cluster alongside your production database is a real risk, and RuntimeClass plus gVisor/Kata is the standard mitigation.

**Practical notes:**

- **Runtimes are installed per node**, and RuntimeClass's `scheduling` stanza routes pods to nodes that have them. Without that, a pod requesting a runtime that a node lacks fails to start.
- **Overhead is declared**, so the scheduler accounts for the runtime's own resource consumption — otherwise you overcommit nodes running sandboxed workloads.
- **Performance costs are real but workload-dependent.** gVisor's syscall interception is expensive for syscall-heavy work (I/O, networking) and cheap for CPU-bound computation. Benchmark before adopting broadly.
- **Pod Security Admission does not make a runtime safe.** A privileged pod on gVisor is still privileged within its sandbox, but the sandbox is the point.

## 44.2 Multi-architecture clusters

`amd64` is no longer the only architecture that matters. Graviton (AWS), Ampere (Azure/GCP), Apple Silicon for development, and ARM edge devices mean most organizations eventually have images that must run on more than one architecture.

**How Kubernetes handles it:** node labels (`kubernetes.io/arch`, `kubernetes.io/os`) and the scheduler's node affinity. A pod without architecture constraints can land on any node — which is fine only if the image is multi-architecture.

**The failure mode is a crash loop, not a scheduling error.** If an image exists only for `amd64` and the pod lands on an `arm64` node, the container starts and immediately dies with an "exec format error." Nothing in the scheduler prevented it, because the scheduler does not read image manifests.

**The fix is at build time, not deploy time:**

```bash
# Build and push a multi-arch image in one command (BuildKit)
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.example.com/api:1.14.2 --push .
```

Then either let the scheduler choose freely (the correct default for a multi-arch image) or constrain deliberately:

```yaml
nodeSelector:
  kubernetes.io/arch: arm64        # only when you specifically want ARM (cost, availability)
```

**Cost is the usual motivation for ARM:** Graviton-class instances are meaningfully cheaper per unit of compute, and for compiled languages and modern runtimes the performance is comparable or better. The migration cost is verifying that every dependency (base images, native extensions, JVM builds) has an `arm64` variant.

**Taints still apply.** If you want ARM nodes reserved for specific workloads, taint them (`arch=arm64:NoSchedule`) and give the intended workloads a toleration plus a node selector — the pattern from Part 2.6.

## 44.3 Windows nodes

Kubernetes supports Windows worker nodes in the same cluster as Linux nodes, and for some organizations it is unavoidable (existing .NET Framework applications, Windows-specific dependencies). It is also a significantly different operational world, and worth understanding before committing.

**Control plane is Linux.** The API server, scheduler, and controller-manager always run on Linux. Windows is workers only.

**What is different:**

- **Containers are Windows containers, and the version match is strict.** Kubernetes supports **only process isolation** — it explicitly does not support running Windows containers with Hyper-V isolation. The result is a hard rule: the host OS version must match the container base image's OS version, so a Windows Server 2022 node cannot run a Windows Server 2019 image. Node operating systems are limited to the supported Windows Server releases. This is the single most common source of "the image pulls but won't start."
- **No `hostNetwork` in the Linux sense, no Linux capabilities, no seccomp.** The security primitives are different (AppArmor/SELinux/capabilities simply do not exist), so Pod Security Admission's `restricted` profile has different implications and many Linux-oriented security policies cannot be applied.
- **The CNI must support Windows.** Not every CNI does — upstream's Windows networking documentation lists win-bridge, win-overlay, Azure-CNI, Flannel, and OVN-Kubernetes. Calico documents Windows support; for others (including Cilium) verify current Windows support explicitly rather than assuming it, because the list changes.
- **Networking is often the hardest part.** Windows nodes historically required a different networking mode (overlay via VXLAN, or a cloud-native L2 mode), and per-node pod capacity behaves differently.
- **Node OS patching follows Windows Update cadence**, and in-place upgrades between Windows versions are not supported in the way they are sometimes assumed to be.
- **Persistent storage support is more limited** — CSI drivers must implement Windows support, and some do not.

**When to use it:** when you have Windows workloads that cannot be containerized on Linux, and you accept a separate, more constrained operational surface. **When not to:** "because we have some Windows servers." A mixed cluster is more work than two clusters, and most organizations isolate Windows into its own node pool (or its own cluster) with taints so the two worlds do not interfere.

## 44.4 Node pools as a design tool

A **node pool** is a provider-managed group of identically configured nodes. Node pools are the primary mechanism for shaping *where* different classes of workload run, and a sensible pool design removes the need for most per-pod placement constraints.

A workable default layout:

| Pool | Instance profile | Taint | Purpose |
|---|---|---|---|
| **system** | Small, stable, on-demand | `CriticalAddonsOnly` or none | CNI, CSI, DNS, monitoring agents, the CNI's DaemonSets |
| **general** | Balanced, mixed on-demand/spot | none | Ordinary stateless services |
| **memory** | High memory-per-CPU | `workload=memory:NoSchedule` | Caches, in-memory databases, JVMs |
| **compute** | High CPU | `workload=compute:NoSchedule` | Batch, encoding, compilation |
| **gpu** | GPU instances | `nvidia.com/gpu=present:NoSchedule` | Inference and training (Part 30) |
| **database** | Large local NVMe | `workload=database:NoSchedule` | Self-hosted stateful workloads (Part 17) |

**Why pools plus taints beat per-pod selectors alone:** the taint guarantees that *only* intended workloads land on expensive hardware, whereas a node selector alone merely says where one workload wants to go. Without the taint, a random pod with no constraints can occupy a GPU node and waste it.

**Give system DaemonSets their own space.** Running the CNI on the same nodes as memory-hungry workloads means an eviction storm can take out networking for every pod on the node — including the ones that would have restarted the CNI. A small, dedicated, on-demand system pool is cheap insurance.

**Spot capacity belongs in its own pool** (or in a Karpenter-provisioned node that declares it), with tolerations on the workloads that can tolerate interruption and PDBs that make the drain orderly. Mixing spot and on-demand in one pool means you cannot predict which workloads will be interrupted.

---

# Part 45 — Platform Security Features

Part 18 covered the security program — identity, RBAC, admission control, supply chain, network policy, hardening order. This part covers the specific platform mechanisms that a modern cluster is expected to have configured, several of which are commonly missed because they are node-level or API-level rather than pod-level.

## 45.1 Seccomp, AppArmor, and SELinux: three different layers

Container isolation (Part 1.2) rests on kernel mechanisms, and each of these restricts a different thing. They are complementary, not alternatives.

**seccomp (secure computing mode) filters system calls.** A container that never needs to call `ptrace`, `mount`, or `reboot` should not be able to. The `RuntimeDefault` profile — the container runtime's built-in allowlist — blocks the most dangerous syscalls with essentially no compatibility cost, which is why it is part of Pod Security Admission's `restricted` level.

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault        # strongly recommended; set it on every workload
      # type: Localhost          # a custom profile installed on the node
      # localhostProfile: profiles/my-app.json
```

**AppArmor applies per-program mandatory access control profiles** — restricting file paths, network access, and capabilities by executable path. It is a Linux Security Module, available by default on Debian/Ubuntu-derived nodes and SUSE, and **not** on Red Hat-derived systems (which use SELinux instead). As of Kubernetes 1.30+, AppArmor is configured through the standard `securityContext` rather than annotations:

```yaml
spec:
  securityContext:
    appArmorProfile:
      type: RuntimeDefault        # or Localhost with a profile name
```

**SELinux labels files and processes with a security context**, and enforcement is kernel-wide. In Kubernetes it matters mainly for **volume labeling**: the kubelet labels mounted volumes so the container's SELinux context can access them, and `securityContext.seLinuxOptions` lets you override that. Kubernetes 1.36 promoted a change to SELinux volume labeling to GA, with documented implications in 1.37 — worth reading the release notes if you run SELinux-enforcing nodes, because volumes that previously mounted successfully can be labeled differently.

**Practical configuration:** set `seccompProfile: RuntimeDefault` and `appArmorProfile: RuntimeDefault` on every workload via a policy or a LimitRange-equivalent default, rather than per-manifest — and let `restricted` Pod Security Admission enforce seccomp for you. For SELinux, verify after any node OS upgrade that your volumes still mount, because this is an area where the platform changed recently.

## 45.2 ServiceAccount tokens: stop using long-lived credentials

Every pod gets a ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. The evolution of that token is a security story worth understanding, because it affects how you should configure workloads.

**The old behavior:** a non-expiring JWT stored in a Secret, mounted into every pod, valid forever. Any leak — a log dump, a misconfigured backup, a compromised container — yielded a permanent credential.

**The current behavior (bound tokens):** tokens are **projected, audience-scoped, and time-limited**, and are rotated automatically by the kubelet. They are bound to the pod's identity and cannot be used elsewhere.

**What to do:**

- **Do not mount the token if the pod does not need API access.** Set `automountServiceAccountToken: false`, either on the ServiceAccount or the pod. Most application pods never call the Kubernetes API, and a token they cannot use is pure attack surface.
  ```yaml
  spec:
    automountServiceAccountToken: false
  ```
- **Create one ServiceAccount per workload** with the minimum RBAC, rather than sharing `default`.
- **Set a short expiration** where you need explicit control, and audience-scope tokens to the API server so a token stolen from one context cannot be replayed against another.
- **Never mount a token into a pod that runs untrusted code.** Combined with `automountServiceAccountToken: false` and a NetworkPolicy that blocks API server access, a compromised pod cannot touch the control plane at all. (Remember from Part 4.3 that `create pods` is already effectively node-admin — so a token with RBAC is a serious privilege.)

## 45.3 Pod certificates and Cluster Trust Bundles

Kubernetes 1.37 advanced two APIs that address workload identity directly in the platform:

- **Pod Certificates** — a pod can request a certificate for its own identity. The division of labour is worth knowing: the **kubelet** generates the private key, creates the `PodCertificateRequest`, and refreshes the projected certificate when it is due for rotation; the actual **signing** is done by a signer implementation (the in-tree one runs in kube-controller-manager), which populates the certificate chain in the request's status. This gives in-cluster workloads the thing service meshes were built to provide (verifiable, automatically rotated identity) **without a mesh**.
- **Cluster Trust Bundles** — an API object for distributing CA bundles to workloads, so pods can validate those certificates without baking a CA into an image or mounting a Secret.

**Why this matters:** the traditional ways to give a workload an identity have all been awkward — a long-lived Secret (rotates badly, leaks permanently), a service mesh (heavy, Part 13), or bespoke tooling (SPIFFE/SPIRE, which works but is another system to run). Pod certificates move identity into the platform. They are young, so treat them as the direction of travel rather than the default choice today, but they are the thing that may make "we need a mesh just for mTLS" obsolete for many clusters.

## 45.4 Runtime security: the layers you should actually enable

A realistic baseline, cheapest first:

| Control | What it catches | Cost |
|---|---|---|
| **seccomp `RuntimeDefault`** on every pod | Exploitation of syscalls the app never needs | None — set it everywhere |
| **`readOnlyRootFilesystem: true`** | Persistence after compromise | Low — needs `emptyDir`s for `/tmp` and caches |
| **`allowPrivilegeEscalation: false`**, all capabilities dropped | Privilege escalation via setuid binaries or added caps | Low |
| **`runAsNonRoot`** with a real UID | Container breakout via in-container root | Low, but requires image changes |
| **Pod Security Admission `restricted`** | All the above, enforced cluster-wide | Medium — breaks non-conforming workloads |
| **Runtime detection** (Falco or Tetragon) | Behavioural anomalies: a shell spawned in a container, an unexpected outbound connection, writes to `/etc` | Medium — needs tuning to avoid noise |
| **Image signing verification** | Running images that did not come from your pipeline | Medium — requires a signing workflow |
| **eBPF-based process/file observability** (Tetragon) | The same as above, with the ability to *enforce* rather than just alert | Medium-high |

**The most underrated item on that list is `readOnlyRootFilesystem`.** It converts "attacker got a shell" into "attacker has a shell in a filesystem they cannot modify," which blocks most persistence techniques — droppers, webshells, modified binaries — with a one-line change and an occasional `emptyDir` for the paths the app genuinely writes.

**The most overrated is runtime detection without tuning.** A detection tool emitting thousands of unactionable alerts per day trains your team to ignore it, which is worse than not having it. Start in alert-only mode, tune to a handful of high-signal rules, and only then consider enforcement.

## 45.5 Audit logging: what actually needs to be recorded

Audit logging is often described as "enable it and ship it," which leaves the important part unspecified. The policy decides *what* is recorded and therefore what you can answer later.

Four levels, and the trade-off is volume against detail:

| Level | Records | Use |
|---|---|---|
| `None` | Nothing | For health checks and other high-volume noise (this is the level that makes audit logging affordable) |
| `Metadata` | Who, what, when, and the result — but not request/response bodies | The right default for most resources |
| `Request` | Metadata plus the request body | Needed for writes to sensitive resources, so you can see what was changed |
| `RequestResponse` | Plus the **response** body | Very high volume, and dangerous: it writes response bodies into the audit log. **Do not use it for `secrets`** — that would copy secret contents into the log, which is the opposite of the goal. Upstream's reference policy logs `secrets` at `Metadata`. |

The things you will actually want to be able to answer, and which therefore deserve `Request` level:

- **Who read or modified a Secret?** (`secrets` — reading one is often equivalent to reading cloud credentials.)
- **Who created a pod, exec'd into one, or port-forwarded?** Note that **exec and port-forward are separate subresources** — `pods/exec` and `pods/portforward`. A rule covering the `pods` resource does **not** match requests to its subresources, so a policy that only audits `pods` will silently record no exec or port-forward activity at any level. Name the subresources explicitly. (Part 18.3 — these are production access.)
- **Who changed RBAC?** (`roles`, `rolebindings`, `clusterroles`, `clusterrolebindings`.)
- **Who deleted something?** (Delete operations across the board.)
- **Which admission webhook rejected this?** (Useful during incidents.)

**Practical guidance:**

- **Exclude the noise deliberately** — health probes, the kubelet's node status updates, and leader-election writes generate enormous volume. Without exclusions, the audit log becomes too expensive to keep and gets turned off.
- **Ship logs off-cluster**, to storage the cluster cannot delete. An audit log stored inside the cluster is destroyed by the incident it was meant to explain.
- **Set retention against your compliance requirement**, and remember that audit logs are usually subject to longer retention than application logs.
- **Alert on a few high-signal patterns**: unexpected `exec` into a production pod, changes to RBAC outside the GitOps pipeline, secret reads by an unexpected identity, and access from a new source IP.

---

# Part 46 — Batch, Event-Driven, and Job Workloads

Part 6.5 covered Job and CronJob mechanics. This part covers the workloads that are batch-first or event-driven, where the scheduling and scaling model differs from a long-running service.

## 46.1 Jobs and CronJobs at scale

The mechanics are in Part 6.5. The operational realities at scale:

- **Use a queue-based job pattern, not one Job per item.** Creating a Job per work item means the control plane processes tens of thousands of Job objects, and each is subject to scheduling overhead, retry limits, and garbage collection. Instead, run N worker pods that pull from a queue (a database table, Kafka, SQS, Redis) and exit when the queue is empty. This is the standard pattern for high-volume batch.
- **Set the parallelism and completion semantics deliberately.** `parallelism` (how many run at once), `completions` (how many must succeed), `completionMode: Indexed` (each pod gets a unique index — essential for partitioned work), and `backoffLimit` (retries before giving up). A Job with no `completions` set and `parallelism: 1` looks fine until you realize it runs once and stops.
- **Work queues need a failure story.** What happens to an item whose pod died mid-processing? Either the work must be idempotent (best) or the queue must track leases and time out. `activeDeadlineSeconds` on the Job bounds the total runtime, which prevents a stuck job from occupying capacity indefinitely.
- **Clean up aggressively.** `ttlSecondsAfterFinished` and the history limits on CronJobs, or the API server accumulates tens of thousands of finished Jobs and slows down for everyone (Part 41).
- **CronJobs are wall-clock and leader-elected.** Set `timeZone` explicitly rather than relying on the default — with no `spec.timeZone`, the schedule is interpreted in the time zone of the **kube-controller-manager process**, which in practice is usually but not necessarily UTC, and remember that `concurrencyPolicy: Forbid` plus a job that outlives its interval means silent skips.
- **Long-running Jobs are scheduled like anything else but tolerate interruption poorly.** Long jobs should checkpoint to durable storage so a preemption (spot reclaim, node drain, eviction) costs minutes rather than hours. Assume preemption; it will happen.

## 46.2 Offloading heavy batch: Spark, Flink, and friends

For genuinely large data processing, running the work as ordinary pods is the wrong shape. The established pattern is a **job controller that creates pods per work unit and then exits**:

- **Apache Spark on Kubernetes** — a Spark driver runs as a pod, requests executor pods, and the cluster scales them; the driver exits when the job finishes. Spark has native Kubernetes support and does not require a permanent cluster.
- **Apache Flink on Kubernetes** — typically a **long-running** JobManager Deployment with TaskManager pods, managed by the Flink Kubernetes Operator, which handles savepoints and upgrades.
- **Ray / KubeRay, Dask, Argo Workflows, Kubeflow Pipelines** — the same pattern for Python-native and pipeline-shaped work. KubeRay manages Ray clusters as CRDs; Argo Workflows models a DAG of steps, each a pod.

**What to get right when you add one of these:**

- **Resource sizing is done by the framework, not by you.** A Spark job asks for executors; if you set its requests too low, executors are OOM-killed mid-job and the job retries from the beginning. Size based on the framework's guidance and observed usage, and give batch work its own node pool so it cannot starve services (Part 44.4).
- **Do not run the driver as a Deployment.** A driver that restarts orphaned executors is a classic operational mess. Run it as a Job, and use the framework's own restart semantics.
- **Batch and latency-sensitive workloads should not share nodes without taints** and a low priority class. A Spark job is designed to consume everything available; that is desirable in a batch pool and catastrophic in a service pool.
- **Spot capacity is ideal for batch** — it is cheap and batch usually tolerates interruption — provided the framework can recover (Spark and Flink both can, with checkpoints/savepoints). This is the single largest cost lever for data workloads (Part 29).

## 46.3 Event-driven autoscaling with KEDA

Plain HPA scales on CPU or memory. For queue-driven work that is the wrong signal: a worker consuming from a queue may sit at 5% CPU while the backlog grows to a million messages, because the bottleneck is I/O wait, not compute. **KEDA** (Kubernetes Event-driven Autoscaling) scales on the *queue*, not the pod.

**What KEDA is:** a controller that installs a metrics adapter and a set of CRDs, so HPA can target external metrics. It supports dozens of sources (Kafka, RabbitMQ, SQS, Redis, Prometheus, cron, Azure Service Bus, and more) plus a `ScaledJob` type for scaling Jobs rather than Deployments.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: order-worker, namespace: prod }
spec:
  scaleTargetRef: { name: order-worker }
  minReplicaCount: 1
  maxReplicaCount: 50
  cooldownPeriod: 300
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: orders
        topic: orders
        lagThreshold: "100"     # target: 100 messages of lag per replica
    - type: cron               # keep a floor during business hours
      metadata:
        timezone: Europe/Berlin
        start: "0 7 * * *"
        end: "0 20 * * *"
        desiredReplicas: "3"
```

**Why this is usually the right answer for queue consumers:**

- **It scales on the actual backlog**, which is the signal that correlates with user-visible latency.
- **It can scale to zero**, which is the cost win for intermittent work — and with HPA scale-to-zero now beta in Kubernetes 1.37 (Part 25.1), the platform is converging on the same capability natively.
- **Multiple triggers combine**, so you can have a queue trigger plus a cron floor plus a Prometheus-based ceiling.

**The traps:**

- **Scaling to zero means cold starts.** A consumer that needs 30 seconds to boot will make the first request after idle slow. Keep a floor (`minReplicaCount: 1`) for anything user-facing.
- **`maxReplicaCount` above what the downstream can absorb makes things worse.** If a queue feeds a database that supports 50 connections, scaling to 200 consumers causes connection exhaustion and failures — the classic self-inflicted outage. Cap consumers by the *downstream's* capacity, not by the queue depth.
- **Scaling to zero interacts with PDBs and node autoscaling** — a workload at zero replicas cannot satisfy a PDB, and nodes may be removed while it is idle, so scale-up then waits for node provisioning. The capacity-buffer pattern (Part 37.6) addresses exactly this.
- **KEDA and HPA must not fight.** KEDA *uses* HPA under the hood; do not also define your own HPA for the same workload.

## 46.4 Event-driven architecture on Kubernetes

Kubernetes itself has no event bus. If you are building event-driven systems, the broker is a workload you run or a managed service you consume:

- **Managed services are usually the better default** (Kafka via MSK/Confluent, SQS/SNS, Pub/Sub, Event Hubs) because brokers are stateful distributed systems with their own operational burden (Part 17.4).
- **If you self-host**, **Strimzi** is the mature Kafka operator, and it handles the genuinely hard parts: rolling upgrades, rack awareness, and PVC management. **NATS** (with JetStream) is far lighter and often sufficient. **RabbitMQ** has a cluster operator but is the least pleasant to run at scale.
- **Knative Eventing** provides a broker/trigger abstraction over a source of your choosing, with CloudEvents as the envelope. It is a real solution when you want event routing decoupled from the broker, at the cost of another control plane.
- **KEDA is the scaling half** of event-driven architecture; a broker is the transport half. They compose: broker for delivery, KEDA for consumer scaling.

---

# Part 47 — The Optional Serverless Layer

Kubernetes is not serverless, but a **serverless layer on top of Kubernetes** is a legitimate architectural choice — and one you should adopt deliberately, knowing what you are buying and what it costs.

## 47.1 What "serverless on Kubernetes" actually means

Four different things get called this, and they are not the same:

| Approach | What it provides | Example |
|---|---|---|
| **Scale to zero** | A Deployment that drops to 0 replicas when idle and wakes on traffic | KEDA (Part 46.3), HPA scale-to-zero (beta in 1.37) |
| **Request-driven compute** | Containers started per request, with concurrency limits and a request queue | Knative Serving, OpenFaaS, Fission |
| **Event-driven functions** | Small handlers triggered by events, with the platform managing the deployment | Knative + Knative Eventing, Kubeless (unmaintained) |
| **Managed FaaS with Kubernetes compatibility** | A cloud-managed function service that can reference container images | AWS Lambda with container images, Google Cloud Run |

**The important distinction is scale-to-zero versus request-driven.** Scale-to-zero is a cost optimization on an ordinary Deployment, and it has a cold-start cost you must design around. Request-driven compute is a different execution model, where the platform queues requests, starts containers to serve them, and scales down aggressively — which means your application's assumptions change (no background threads, no in-memory caches that survive, no long-lived connections).

## 47.2 Knative, honestly

**Knative Serving** is the reference implementation of request-driven compute on Kubernetes. It introduces CRDs — `Service` (an unfortunate name collision with the core kind), `Revision`, `Route`, `Configuration` — and manages the full lifecycle: build/apply a Revision, route traffic to it, scale it from zero on incoming requests, and support traffic splitting between Revisions (which makes canarying effectively free).

**What you get:** automatic scale-to-zero, per-revision traffic splitting with fine-grained percentages, revisions as immutable deployable snapshots (rollback is instant), and a pluggable networking layer that now supports Gateway API.

**What it costs:**

- **A control plane of its own** — the Knative controllers, plus (in many setups) Kourier or Istio/Envoy Gateway for the data plane. That is another set of components to upgrade, monitor, and reason about.
- **Cold starts are the central design constraint.** A container that takes 30 seconds to start means a 30-second first request. Mitigations: keep a minimum of one replica for latency-sensitive services (which negates the cost saving), use faster runtimes, or pre-warm. The available strategies all trade money or complexity for latency.
- **The concurrency model changes your application.** A Knative service gets a concurrency target per replica; under load, requests queue and new replicas start. Applications with long-lived connections (WebSockets), background workers, or in-memory session state do not fit cleanly.
- **Debugging spans two systems.** A failed request may be a Knative routing/activation problem or an application problem, and you need to know which layer to inspect (the same discipline as Part 11.8).

**When it is right:** bursty, HTTP-shaped, stateless workloads where idle cost matters more than tail latency — internal tools, webhooks, low-traffic APIs, and preview environments. **When it is wrong:** high-throughput services (you will keep replicas warm anyway, so you have added a control plane for nothing), long-running connections, and anything with meaningful startup cost.

## 47.3 The architectural decision

The honest framing: **serverless on Kubernetes solves a cost problem (idle capacity) and creates an operations problem (another control plane, cold starts, a different execution model).** Decide based on which one dominates for you.

If your workloads are steady-state, **plain Deployments with HPA are simpler and better**. If you have many low-traffic services, ephemeral environments, or genuinely bursty traffic, the trade-off flips. And if what you actually want is a managed FaaS, using the cloud's own service (Cloud Run, Lambda with container images) gets you the same execution model without operating a platform for it — at the cost of leaving Kubernetes for that part of your system.

---

# Part 48 — The Local Development Inner Loop

Everything so far has been about running workloads in a cluster. This part is about the question that determines whether developers actually like your platform: **how do you iterate on code when the application runs in Kubernetes?**

If the answer is "build an image, push it, wait for a rollout, read logs" for every change, your inner loop is minutes long instead of seconds, and developers will avoid the cluster — which is how platforms fail (Part 31.5). Getting this right is a platform responsibility, not a personal preference.

## 48.1 The four approaches

| Approach | How it works | Best for | Cost |
|---|---|---|---|
| **Local cluster** (kind, k3d, minikube) | The whole cluster runs on the laptop | Testing manifests, CRDs, and cluster-level behavior; learning | Resource-hungry; not production-like for cloud-specific features |
| **Hot-reload in cluster** (Skaffold, Tilt, DevSpace, Garden) | Code changes are synced into a running pod; the process restarts or the app rebuilds | Day-to-day feature work on a service | Needs a shared or personal cluster; sync configuration |
| **Local process, cluster dependencies** (Telepresence, Gefyra, mirrord, `kubectl port-forward`) | The app runs on the developer's machine but sees cluster services, DNS, and secrets | Fast iteration on one service against real dependencies | Network complexity; interception can surprise teammates on shared clusters |
| **Preview environments** (Argo CD ApplicationSet, per-PR namespaces) | A full environment per pull request, torn down on merge | Review, integration testing, demos (Part 22.5) | Cost, plus the discipline to make manifests parameterized |

**Most effective setups use two of these**: hot-reload or connect-mode for the inner loop (seconds), and preview environments for the review loop (minutes but isolated). A local cluster is for manifest work, not for application development against real dependencies.

## 48.2 How the connect-mode tools work, and when they bite

Tools like Telepresence, Gefyra, and mirrord solve the same problem differently: they make a process running **on your laptop** appear to be running **in the cluster**, so it can resolve `api.prod.svc.cluster.local`, read mounted Secrets, and reach databases.

The mechanism is traffic interception. In the common mode:

- The tool installs a small agent or sidecar in the cluster.
- Your local process is given the cluster's DNS and network view (via a tunnel and a proxy, or via a namespace with a modified resolver).
- **Optionally**, all traffic to your service is redirected to your laptop, so other services in the cluster reach *your* local instance.

**That last option is powerful and dangerous on a shared cluster.** Redirecting a shared service's traffic to your laptop means:

- Teammates using that environment now hit your half-finished code.
- Your laptop sleeping or disconnecting breaks *their* testing.
- Debugging becomes confusing for everyone who is not you.

**Rules for using interception tools safely:** use a **personal namespace** or a **personal cluster**; if you must intercept on a shared environment, intercept a service nobody else depends on, announce it, and make sure the tool cleans up reliably when you disconnect. Most teams end up with per-developer namespaces for exactly this reason (Part 23).

## 48.3 What the platform team owes developers

A defensible inner-loop offering for a platform, in the order that removes the most pain:

1. **A documented way to run the app locally** with dependencies, without a cluster, for the common case. Fastest possible iteration, zero cluster involvement. Many services can test 90% of their logic this way, and it should be the default suggestion.
2. **Per-developer namespaces** (or a shared dev namespace with strict naming), created by a PR, with quotas, so there is somewhere safe to experiment (Part 23.1).
3. **One supported sync/reload tool**, configured and documented — not five tools and a shrug. Provide a working config in the service template so nobody has to learn it.
4. **Preview environments per PR**, automatically created and destroyed.
5. **A local cluster recipe** (kind/k3d config in the repo) for testing cluster-level changes such as CRDs and admission policy.

**Signals you have this wrong:** developers rebuilding images to change a log line; a documented `kubectl port-forward` sequence longer than three lines; no way to test a change without merging it; and a platform team that has never timed its own edit-to-verify cycle. That last one is worth doing literally — measure the loop, then shorten it. It is usually the highest-leverage platform improvement available.

---

*Written against an empty cluster and current versions as of late 2026. Version-sensitive claims (feature-gate stages, per-implementation Gateway API support, add-on compatibility) should be re-checked against the upstream release notes for whatever you actually run — that habit is itself part of operating Kubernetes well.*
