# Security policy

## Supported versions

flotilla has no release yet. Until 1.0, only the latest release gets security fixes.

## Reporting a vulnerability

Please do not open a public issue for a security problem.

Report it privately through GitHub: open the [Security tab](https://github.com/raven-clown/flotilla/security) and choose "Report a vulnerability". Include what you found, how to reproduce it, and the version or commit you tested.

You can expect a first reply within 7 days. Once the problem is confirmed, a fix and an advisory are prepared together, and you are credited unless you ask not to be.

## Scope

In scope: the `flotilla` binary and everything in this repository, including the API server, the store, authentication, TLS and certificate handling, Secret encryption, and the node install scripts.

Upstream components that flotilla bundles on nodes (kubelet, containerd, kube-proxy, CoreDNS, CNI plugins) should be reported to their own projects. If the problem is in how flotilla configures or ships them, report it here.
