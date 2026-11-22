# flotilla design

flotilla is a Kubernetes-compatible orchestrator with its control plane written in Rust. It serves the Kubernetes API, so kubectl, Helm, kustomize, existing manifests, CRDs and operators work unchanged. On top of that it adds a short config format, a light control plane with no etcd, encrypted networking by default and a one-command install.

Background research, with sources, is in `docs/research.md`.

## 1. Goal and audience

flotilla is for teams that run Kubernetes in production, or would if it were lighter and simpler. They should be able to switch without rewriting anything and end up with something cheaper to run and easier to operate.

Success means:

1. **Drop-in.** Workloads and tools that run on Kubernetes run on flotilla unchanged. Measured by the CNCF conformance suite (sonobuoy) pass rate, published in CI and the README for every release.
2. **Lighter.** The control plane uses well under the memory of k3s on the same cluster. Target: under 150 MB RSS for one server with 10 nodes and 500 pods.
3. **Faster to set up.** A fresh Linux machine runs its first app in under 5 minutes with no more than 3 commands.
4. **Shorter config.** A typical web app (deployment, service, ingress, autoscaler) is under 20 lines in flotilla's short config, against about 90 lines of Kubernetes YAML.

No claim of "better" goes into the README until a benchmark in `bench/` backs it (section 10).

## 2. Decisions

| Decision | Choice | Why |
|---|---|---|
| API | The Kubernetes API, with flotilla's short config as an extra on top | Switching must work right away. A new API would lock out kubectl, Helm and operators. |
| Control plane language | Rust, written from scratch | The weight of k3s and k0s is the Go API server, its watch cache and etcd. Reusing upstream Go code would keep that weight. |
| State store | openraft over redb, with MVCC revisions | Removes etcd and its operational cost. Raft from the first release, so high availability later needs no data migration. |
| MVP topology | One server, many nodes | HA with 3 servers comes in P2 on the same store. |
| Node agent | Upstream kubelet and containerd first, a Rust agent later as an option | Reimplementing kubelet means chasing upstream, which is what stopped Krustlet. kubelet makes conformance reachable. |
| Container runtime | Through CRI only | The sibling Rust container runtime can replace containerd later with no orchestrator change. |
| Networking | flotilla CNI with a WireGuard mesh by default | Encrypted pod traffic across nodes and clouds with no setup. |
| Service proxy and DNS | Upstream kube-proxy (nftables mode) and CoreDNS in P1, Rust replacements in P3 | Conformance needs ClusterIP and cluster DNS from the start. Replacing them is an optimisation, not a blocker. |
| Short config | TOML and YAML, one schema, compiled on the client into Kubernetes objects | Works against flotilla and against any real Kubernetes cluster. `flotilla render` shows the exact objects. |

## 3. Architecture

```
            kubectl / helm / flotilla CLI
                        |
                 HTTPS (Kubernetes API)
                        |
  +---------------- flotilla server ----------------+
  |  apiserver  |  scheduler  |  controllers        |
  |---------------------------------------------- --|
  |  store: openraft + redb (MVCC, watch)           |
  +-------------------------------------------------+
                        |
        HTTPS (kubelet uses the normal API)
                        |
  +------------------ node (each) ------------------+
  |  flotilla node: installs and supervises         |
  |    containerd  <-- CRI --  kubelet              |
  |    flotilla CNI (bridge, IPAM, WireGuard)       |
  |    kube-proxy (nftables)        [Rust in P3]    |
  +-------------------------------------------------+
```

One binary, `flotilla`, has these modes:

- `flotilla server`: the whole control plane in one process.
- `flotilla node`: installs, configures and supervises containerd, kubelet, the CNI plugin and kube-proxy on a node, and joins it to a server with a join token.
- `flotilla server --node`: both on one machine, for a single-node cluster.
- CLI commands: `apply`, `render`, `migrate`, `token`, `backup`, `restore`, `dev cluster`. Everything else is done with kubectl.

The server side is one binary. The node side is not: it bundles upstream kubelet, kube-proxy, containerd and the CNI reference plugins, the same way k3s bundles containerd. The docs say so plainly.

### 3.1 Store

- Key space mirrors the Kubernetes registry layout: `/registry/<group>/<resource>/<namespace>/<name>`.
- Every write gets a cluster-wide revision. `resourceVersion` is that revision. Objects are stored as protobuf or JSON bytes with the revision.
- Watch starts from any revision still held, and returns `410 Gone` after compaction, matching etcd semantics that clients depend on.
- Compaction runs on a schedule and keeps a configurable window (default 5 minutes of history).
- Writes go through Raft. Reads are served from the local state machine after a read index check, so lists are consistent.
- P1 runs one voter. P2 adds `flotilla server join`, membership changes and snapshots.
- Backup is a consistent snapshot to a file (`flotilla backup`). Restore rebuilds a one-voter cluster from it.

### 3.2 API server

Built on axum and hyper with rustls. Uses `k8s-openapi` types for built-in kinds.

- Discovery (`/api`, `/apis`, aggregated discovery) and OpenAPI v2 and v3.
- List with pagination (`limit`, `continue`), label and field selectors, watch with bookmarks.
- Create, update, delete with preconditions, `generateName`, finalizers and graceful deletion.
- Patch: JSON patch, JSON merge patch, strategic merge patch and server-side apply with managed fields.
- Subresources: `status`, `scale`, and `log`, `exec`, `attach`, `port-forward` proxied to the kubelet.
- Authentication: x509 client certificates, bearer tokens, bound service account tokens (JWT with OIDC discovery).
- Authorization: RBAC. P1 ships the default cluster roles so kubectl and Helm work.
- Admission: built-in plugins first (namespace lifecycle, service account, defaulting, limit ranges in P2), validating and mutating webhooks in P3, plus ValidatingAdmissionPolicy (CEL) in P3.
- CRDs in P3: structural schemas, defaulting, pruning, CEL validation rules, conversion webhooks, status and scale subresources.
- Aggregation layer (APIService) in P2 so metrics-server works.
- Secrets are encrypted at rest by default with AES-GCM, key held by the server and rotatable.

Memory is kept low by serving lists and watches straight from the store's revision index instead of a second full watch cache.

### 3.3 Scheduler

Runs in the server process, watching pods with no node. Implements the core Kubernetes plugins in order:

- P1: resource fit, node name, node selector, taints and tolerations, node unschedulable, spread across nodes by default.
- P2: node affinity, pod affinity and anti-affinity, topology spread constraints, volume binding.
- P4: priority and preemption, scheduling gates.

### 3.4 Controllers

Run in the server process with a shared informer layer over the local store.

- P1: Deployment, ReplicaSet, Namespace lifecycle, garbage collector (owner references), EndpointSlice, ServiceAccount and token, Node lifecycle (heartbeats, taints on not-ready), Event recording.
- P2: StatefulSet, DaemonSet, Job, CronJob, PV and PVC binding, attach and detach for CSI, HorizontalPodAutoscaler, PodDisruptionBudget, ResourceQuota, certificate signing for kubelet serving certs.
- P3: TTL after finished, remaining controllers that conformance needs.

### 3.5 Node side

`flotilla node` does the work that makes a node join in one command:

1. Downloads or unpacks the bundled containerd, kubelet, kube-proxy and CNI plugins for the server's Kubernetes version, and checks their hashes.
2. Writes their config and systemd units, then starts them.
3. Bootstraps kubelet with a join token over TLS bootstrap, as kubeadm does.
4. Runs the flotilla CNI plugin.

The flotilla CNI plugin:

- The server hands each node a pod subnet (default /24 from `10.42.0.0/16`) through the Node object's `podCIDR`.
- On the node: a Linux bridge plus host-local style IPAM for pods.
- Between nodes: a WireGuard interface. Each node publishes its public key and endpoint in a Node annotation. The plugin watches nodes and keeps peers and routes current through netlink.
- P3 adds NetworkPolicy enforcement with nftables.

### 3.6 Short config

The short config compiles into normal Kubernetes objects on the client. It never needs special server support, so it also works against a real Kubernetes cluster.

```toml
# app.toml
[app.web]
image    = "ghcr.io/acme/web:1.4"
replicas = 3
port     = 8080
health   = "/healthz"
expose   = "web.acme.com"
cpu      = "500m"
memory   = "256Mi"
scale    = { min = 3, max = 10, cpu = 70 }

[app.web.env]
LOG_LEVEL    = "info"
DATABASE_URL = { secret = "db-url" }

[app.worker]
image    = "ghcr.io/acme/worker:1.4"
replicas = 2
```

`flotilla apply app.toml` produces and applies, with server-side apply:

- `web`: Deployment (rolling update, readiness and liveness from `health`), Service, Ingress for `web.acme.com`, HorizontalPodAutoscaler.
- `worker`: Deployment.

The same file in YAML is accepted (`app.yaml`). Rules:

- Unknown keys are an error with file, line and column.
- `kind` defaults to `service`. Other kinds: `job`, `cron`, `daemon`, `stateful`.
- Every default goes into `docs/config.md` when the short config ships, and `flotilla render` shows them.
- An `extra` table per app takes a raw Kubernetes patch for anything the short form does not cover, so nobody is stuck.

The YAML parser is picked in the plan. `serde_yaml` is archived, so the candidates are maintained forks or `saphyr`.

## 4. Compatibility

### 4.1 What is drop-in, by phase

| Phase | Works unchanged | Not yet |
|---|---|---|
| P1 | kubectl `apply`, `get`, `describe`, `logs`, `exec`, `port-forward`, `rollout`. Pod, Deployment, ReplicaSet, Service (ClusterIP, NodePort), ConfigMap, Secret, Namespace, Event, ServiceAccount, Node. Helm charts that use only those kinds. Any ingress controller that runs as a Deployment. | HA, StatefulSet, PV/PVC, CRDs, webhooks, HPA |
| P2 | HA control plane, StatefulSet, DaemonSet, Job, CronJob, PV/PVC with existing CSI drivers, full RBAC, HPA with metrics-server, PDB, quotas, LoadBalancer services through an external controller | Operators that need CRDs |
| P3 | CRDs, admission and conversion webhooks, ValidatingAdmissionPolicy, NetworkPolicy, operators such as cert-manager and Velero | Full conformance |
| P4 | Full conformance, CNCF certification, Gateway API | |

### 4.2 How it is measured

- Every PR runs sonobuoy in quick mode against a CI cluster.
- Nightly CI runs the full conformance suite and records the pass count.
- The README shows the latest pass count and the Kubernetes version it targets.
- A compatibility test set of real Helm charts (ingress-nginx, cert-manager, redis, postgres, prometheus) runs in CI from the phase that should support it.

### 4.3 Honest limits

Full conformance is far off. The API machinery (watch semantics, server-side apply, CRD schemas with CEL, admission) is the largest part of Kubernetes, and there is no upstream code to reuse in Rust. P1 is a matter of months. P4 is likely years of work for one maintainer, and depends on outside contributors.

flotilla tracks one Kubernetes minor version at a time and follows upstream's skew policy for kubelet.

## 5. Feature matrix

"Phase" is when flotilla ships the row. "Better" is what flotilla aims to beat, and how it will be measured.

| Area | Kubernetes, k3s, k0s | Nomad | Swarm | flotilla | Better at |
|---|---|---|---|---|---|
| Workloads | Deployment, StatefulSet, DaemonSet, Job, CronJob | service, batch, system, sysbatch | replicated, global, jobs | Same kinds as Kubernetes (P1 to P2), plus the short config's single `[app]` form | Config length (measured: lines for reference apps) |
| Scheduling | Full plugin set | Constraints, affinity, spread, preemption | Constraints, placement prefs | Kubernetes plugins (P1 to P4) | Scheduling latency and throughput (benchmark) |
| Networking | Choose and install a CNI. k3s ships flannel, unencrypted by default | Bridge and CNI | Overlay, IPsec optional | WireGuard mesh by default (P1), NetworkPolicy (P3) | Encrypted by default, no setup |
| Service discovery | ClusterIP, CoreDNS | Built-in or Consul | DNS and VIP | ClusterIP and DNS (P1), Rust proxy and DNS (P3) | Memory per node (benchmark) |
| Ingress and TLS | Install a controller and cert-manager. k3s ships Traefik | External | Routing mesh, manual TLS | Any controller (P1), built-in Rust ingress with ACME (P3) | One line in short config: `expose` |
| Storage | CSI, PV/PVC | CSI, host volumes | Volume drivers | Same CSI drivers (P2) | Parity only. The driver ecosystem is shared, not beaten |
| Config and secrets | Secrets base64, encryption optional | Variables, Vault | Encrypted in Raft | Encrypted at rest by default (P1) | Safe by default |
| RBAC and tenancy | RBAC, namespaces | ACL, namespaces | None | RBAC (P1 defaults, P2 full), OIDC (P2) | Parity |
| Autoscaling | HPA (needs metrics-server), VPA, cluster autoscaler | Separate autoscaler | None | HPA (P2), metrics built into the server (P3), cluster autoscaler works through the API (P2) | No add-on for basic metrics |
| Rolling deploys | Rolling and recreate. Auto rollback needs Argo Rollouts | Canary, auto revert | Rolling, rollback on failure | Kubernetes semantics (P1). Short config adds health-gated auto rollback in `flotilla apply` (P2) | Rollback by default |
| Observability | Logs, events. Metrics need add-ons | UI and metrics | Logs | Logs, exec, events (P1). Prometheus `/metrics` on server and CNI (P1). OpenTelemetry traces (P4) | Metrics with no add-on |
| HA control plane | etcd with 3 or more members | Raft | Raft | Raft with 3 servers (P2) | Failover time (benchmark) |
| Install and upgrade | kubeadm steps. k3s and k0s: one script | One binary | Built into Docker | One script per machine (P1), rolling upgrade (P2) | Time to first app (benchmark) |
| Extensibility | CRDs and operators | Driver plugins | None | CRDs and operators (P3) | Parity only. Same ecosystem |
| Control plane memory | k3s about 800 to 900 MB for the server | Light | About 50 MB (blog figure) | Target under 150 MB at 10 nodes, 500 pods | RSS (benchmark) |

Where flotilla cannot honestly be better: maturity and years of production use, the breadth of the operator and CSI ecosystem (shared, so equal at best), and anything Swarm wins by being part of Docker.

## 6. Migration from an existing cluster

1. Stand up a flotilla cluster next to the old one.
2. `flotilla migrate --from <kubecontext> --namespaces a,b` reads the old cluster and applies to flotilla:
   - Namespaces, workloads, Services, ConfigMaps, RBAC, Ingresses.
   - Secrets only with `--include-secrets`, and the command says so before it copies them.
   - It drops server-owned fields (status, managed fields, UIDs, cluster IPs unless asked to keep them).
   - It reports every object it skipped and why, for example a CRD kind flotilla does not serve yet.
3. Move traffic one app at a time with DNS or the load balancer (blue and green clusters).
4. Data in persistent volumes is the hard part. P2 supports moving it with CSI snapshots where the driver allows, or with a restic copy job. From P3, Velero works unchanged.

`flotilla migrate --dry-run` prints the plan without writing.

## 7. Phases

Each phase ends with a release, a benchmark run and a published conformance count.

**P1: kubectl works (MVP).** One server, many nodes. Store, API server with list, watch, patch and server-side apply, default RBAC, P1 scheduler plugins and controllers, flotilla CNI with WireGuard, kube-proxy and CoreDNS bundled, `flotilla node` join, short config with `apply` and `render`, `flotilla migrate --dry-run`, `flotilla dev cluster`, first benchmarks. Exit: the P1 row of section 4.1 passes in CI on a 3-node cluster.

**P2: Helm and production.** HA with 3 servers, backup and restore, StatefulSet, DaemonSet, Job, CronJob, PV/PVC and CSI, full RBAC, OIDC login, aggregation layer, HPA, PDB, quotas, rolling upgrade of servers and nodes, health-gated auto rollback in `flotilla apply`, `flotilla migrate` with secrets and volumes. Exit: the P2 Helm chart set installs and the HA failover test passes.

**P3: operators.** CRDs with CEL, webhooks, ValidatingAdmissionPolicy, NetworkPolicy, Rust service proxy and DNS replacing kube-proxy and CoreDNS, built-in ingress with ACME. Exit: cert-manager and Velero pass their own smoke tests.

**P4: conformance.** Remaining API and controller behaviour, Gateway API, priority and preemption, OpenTelemetry. Exit: full sonobuoy conformance passes, submission to CNCF.

**P5: beyond.** Optional Rust node agent replacing kubelet, the sibling Rust runtime through CRI, benchmarks at 5,000 nodes, store tuning. Exit: published benchmark report against k3s, k0s, Nomad and Swarm.

## 8. Error handling and security

- Every API error is a Kubernetes `Status` object with the right code and reason, because clients branch on them.
- The store refuses writes when it loses quorum and reports it on `/readyz`. Reads keep working from the last applied state only with an explicit stale-read flag.
- All traffic is TLS. The server generates a cluster CA on first start. Join tokens expire (default 24 hours).
- Secrets are encrypted at rest. Keys and tokens never appear in logs, errors or test fixtures.
- Node binaries are pinned by hash in the flotilla release and checked before they run.

## 9. Development and testing

The user develops on Windows 11, and everything that runs a cluster needs Linux.

- **Local:** WSL2 with Ubuntu 24.04 and systemd on. Build with cargo inside WSL. `flotilla dev cluster --nodes 3` starts each node as a privileged container running containerd and kubelet, the way kind does, so a multi-node cluster runs on one laptop.
- **Crates that do not touch Linux** (api types, store, apiserver logic, scheduler, short config) must pass `cargo test` on Windows too. Linux-only code sits behind `cfg(target_os = "linux")`.
- **Tests first**, table-driven where there is a table, no network in unit tests. The store gets property tests and a Jepsen-style fault test in P2.
- **CI on GitHub Actions, Linux runners:**
  - `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`, `cargo deny check`, coverage with `cargo llvm-cov`.
  - Integration: start a 3-node dev cluster, run kubectl and Helm smoke tests.
  - Conformance: sonobuoy quick on every PR, full nightly.
  - Windows job for the portable crates.

## 10. Benchmarks

`bench/` holds scripts that run flotilla, k3s, k0s, Nomad and Swarm on identical VMs and record:

- Idle RSS and CPU of the control plane and of a node, at 1, 10 and 100 nodes.
- Time from a fresh machine to the first app answering HTTP.
- Time to roll out 100 replicas until all are ready.
- Scheduling throughput in pods per second.
- Control plane failover time (P2 and later).
- Config lines for the same reference apps.
- List and watch latency at 10,000 objects.

The methodology and raw results are published with each release. A row in section 5 says "better" only when these numbers show it.

## 11. Repository layout

```
crates/
  flotilla/      the binary and CLI
  api/           Kubernetes types, codecs, Status errors
  store/         openraft + redb, MVCC, watch, compaction
  apiserver/     HTTP handlers, discovery, patch, apply, authn, RBAC, admission
  scheduler/     scheduling framework and plugins
  controllers/   informers and controllers
  cni/           CNI plugin, bridge, IPAM, WireGuard
  node/          node install, supervision, join
  shortcfg/      TOML and YAML short config, compile to objects
  migrate/       read from a cluster, plan, apply
bench/           benchmark scripts and reference apps
docs/            design, config reference, plans
```

## 12. Related projects

- The sibling Rust container platform can replace containerd under kubelet through CRI. flotilla only talks CRI, so no shared code is needed.
- The sibling Rust registry is a normal OCI registry to flotilla. Nodes pull from it like any other.
- The sibling multi-cluster manager manages flotilla clusters through the Kubernetes API. Multi-cluster UI and fleet management belong there, not here.

## 13. Out of scope

- A web UI for one cluster beyond what the multi-cluster manager provides.
- Windows nodes.
- Running VMs or plain binaries as workloads (Nomad-style drivers).
- Any API that is not Kubernetes, apart from the client-side short config.
