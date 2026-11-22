# Research notes

Background for `docs/design.md`. Figures come from vendor docs where possible and are marked when they come from blogs.

## Where Kubernetes hurts

**Memory and CPU.** A stock control plane is five or more processes (kube-apiserver, etcd, scheduler, controller-manager, kubelet, kube-proxy, plus a CNI and CoreDNS). Even k3s, which packs all of that into one binary, lists 2 cores and 2 GB RAM as the server minimum. Its own profiling page measures about 768 MB for the server on SQLite and about 896 MB on embedded etcd, and about 1.6 GB for one server plus one agent with no workload. Usage grows with nodes and objects. ([k3s requirements](https://docs.k3s.io/installation/requirements), [k3s resource profiling](https://docs.k3s.io/reference/resource-profiling))

**etcd.** etcd wants fast disks (fsync latency is the usual cause of leader loss), regular defrag and compaction, and a separate backup story. The API server keeps a watch cache of every object in memory, so the cost scales with object count, not with workload size.

**Config volume.** A single web app usually means a Deployment, a Service, an Ingress, a ConfigMap, maybe a Secret, an HPA and a PodDisruptionBudget. That is 100 to 200 lines of YAML that repeat labels and selectors by hand. Helm and Kustomize exist mostly to manage that volume, and add their own languages on top.

**Setup and day two.** kubeadm needs a CNI chosen and installed, certificates rotated, version skew rules followed and upgrades done one minor version at a time. Most of the "hard" in Kubernetes is the number of moving parts, not any single part.

**Concepts.** Pods, ReplicaSets, Deployments, Services, Endpoints/EndpointSlices, Ingress or Gateway, namespaces, labels and selectors, RBAC, service accounts. A newcomer meets all of them before the first app is reachable.

## What the lighter options do

| | Shape | What it keeps | What it drops or changes |
|---|---|---|---|
| **k3s** | One Go binary, server and agent modes | Full Kubernetes API, conformance | Swaps etcd for SQLite via kine by default, bundles flannel, CoreDNS, Traefik. Still around 1 GB for the server. |
| **k0s** | One Go binary, control plane runs as plain processes | Full Kubernetes API | No host OS deps, control plane isolated from workers. Similar weight to k3s. |
| **Nomad** | One Go binary, server and client agents | Its own API and HCL job files | Schedules containers, VMs and plain binaries. Needs Consul for service mesh and older service discovery, Vault for secrets. Moved to the Business Source License in 2023. |
| **Docker Swarm** | Built into the Docker engine (swarmkit) | Compose file format | Smallest learning curve: `docker swarm init`, then `docker stack deploy` with a Compose file. Raft built in, routing mesh, overlay network. Kept alive in maintenance mode, few new features. |

Blog comparisons (unsourced, treat loosely) put an idle Swarm manager near 50 MB and k3s near 500 MB, and agree that Swarm is the simplest and k3s the heaviest of the three. ([dev.to comparison](https://dev.to/selfhostingsh/k3s-vs-docker-swarm-4kog))

The lesson from the table: k3s and k0s prove you can package Kubernetes into one binary but not make it light, because the API surface itself is the weight. Swarm and Nomad are light because they chose a smaller model, and both lost ground because they also lost the Kubernetes ecosystem.

## Rust work that exists

- **kube-rs** (CNCF Sandbox): client, watcher and controller runtime in the style of client-go and controller-runtime. Current release on docs.rs is 4.2.0, tested against Kubernetes 1.32 and later. It is a client library. It does not serve the Kubernetes API. ([docs.rs/kube](https://docs.rs/crate/kube/latest))
- **k8s-openapi**: generated Rust types for every Kubernetes resource. Useful if we ever import or export Kubernetes manifests.
- **Krustlet** (Deis Labs, Microsoft): a kubelet in Rust that ran WebAssembly instead of containers. Archived on 1 Feb 2025 with a call for maintainers and no formal post-mortem we could find. The practical reasons: the Wasm-on-Kubernetes work moved to containerd shims (runwasi, SpinKube), so a whole second kubelet stopped being needed, and keeping a kubelet in step with upstream kubelet behaviour was a large ongoing cost for a small team. Lesson for us: do not chase a moving upstream component you do not control. ([InfoQ](https://www.infoq.com/news/2020/06/krustlet-kubelet-rust/), [archived fork](https://github.com/stackabletech/krustlet))
- **youki**: an OCI runtime (runc replacement) in Rust. A possible low-level runtime under containerd.
- **openraft**: a Raft library in Rust, used in production by Databend and others. The obvious basis for a built-in state store.
- **CRI from Rust**: CRI is a gRPC API defined in protobuf, so `tonic` plus the upstream `.proto` files give a client. Crates such as `containerd-client` and `k8s-cri` already do this.

## Where the work is huge

Said plainly so the scope can be cut early:

1. **Kubernetes API compatibility is years of work.** Serving the API means watch with resource versions, server-side apply and its field managers, strategic merge patch, OpenAPI discovery, CRDs with schema validation and conversion webhooks, admission, RBAC, and dozens of built-in kinds with exact controller behaviour. The conformance suite has hundreds of tests. k3s and k0s get this by reusing upstream Go code. A Rust reimplementation has no such shortcut.
2. **Networking.** Pod-to-pod across nodes needs an overlay (VXLAN or WireGuard), IP address management, service virtual IPs (nftables or eBPF) and cluster DNS. Each is its own project in the Kubernetes world.
3. **A replicated state store.** Raft is available as a library, but snapshots, membership changes, disk faults and upgrades still need careful testing.
4. **Node agent.** Image pulls, health checks, restarts, log streaming and exec all go through the runtime. CRI keeps this bounded, but the edge cases (OOM, disk pressure, zombie containers) are many.
5. **Linux only.** Everything above needs Linux kernel features. On Windows 11, development runs in WSL2 and tests run on Linux CI.
