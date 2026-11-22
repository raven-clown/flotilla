# flotilla

A Kubernetes-compatible orchestrator with its control plane written in Rust. The goal is a drop-in replacement: kubectl, Helm, kustomize, existing manifests, CRDs and operators should work unchanged, on a control plane that is lighter, faster to set up and easier to run.

**Status: design in progress.** There is no code yet and nothing below works today. This README describes what is planned. The full design is in [docs/design.md](docs/design.md) and the background research is in [docs/research.md](docs/research.md).

## What is planned

- **The Kubernetes API itself**, served by a Rust control plane: API server, scheduler and controllers in one binary.
- **No etcd.** State lives in a built-in Raft store, starting with one server and growing to three for high availability.
- **Upstream kubelet and containerd on nodes** at first, joined with one command. A lighter Rust node agent is a later option.
- **Encrypted pod networking by default** with a WireGuard mesh between nodes.
- **Secrets encrypted at rest by default.**
- **A short config format** in TOML or YAML that compiles to normal Kubernetes objects, as an extra next to plain manifests:

  ```toml
  [app.web]
  image    = "ghcr.io/acme/web:1.4"
  replicas = 3
  port     = 8080
  health   = "/healthz"
  expose   = "web.acme.com"
  scale    = { min = 3, max = 10, cpu = 70 }
  ```

- **A migration command** that copies workloads from an existing Kubernetes cluster.

## How compatibility will be measured

Compatibility is measured, not claimed. CI will run the CNCF conformance suite (sonobuoy) and publish the pass count with every release, and claims about memory, speed and setup time will only be made once the benchmarks in `bench/` back them. Full conformance is a long way off. The design lists, phase by phase, what is expected to work and what is not.

## Phases

1. **kubectl works:** core kinds, one server, many nodes.
2. **Helm and production:** high availability, StatefulSets, volumes through CSI, autoscaling, full RBAC.
3. **Operators:** CRDs, admission webhooks, network policy.
4. **Conformance:** the full CNCF suite.
5. **Beyond:** a Rust node agent, scale benchmarks.

## Platform

Nodes run Linux. Development works on Linux or on Windows through WSL2.

## Contributing

Contributions are welcome. Start with [CONTRIBUTING.md](CONTRIBUTING.md), pick an issue from the [P1 milestone](https://github.com/raven-clown/flotilla/milestone/1), or ask in [Discussions](https://github.com/raven-clown/flotilla/discussions). Security problems go through [SECURITY.md](SECURITY.md).

## License

Apache-2.0. See [LICENSE](LICENSE).
