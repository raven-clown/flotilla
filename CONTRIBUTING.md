# Contributing to flotilla

Thanks for helping. flotilla is a Kubernetes-compatible orchestrator with a Rust control plane, and it is early: the design is settled and the first code is being written. Issues, pull requests and discussion are all welcome.

## Before you start

- Read [docs/design.md](docs/design.md). It explains what flotilla is, what is in scope and which phase each feature belongs to.
- The current work is in [docs/plans/](docs/plans/). Each plan task has its own issue in the P1 milestone.
- For anything larger than a small fix, open an issue or a discussion first so we can agree on the approach before you write code.
- Issues labelled `good first issue` are small and well scoped. `help wanted` means outside help is especially welcome.

## Development setup

Nodes run Linux, so anything that starts a cluster needs Linux. On Windows, use WSL2 (Ubuntu 24.04 with systemd on). Crates that do not touch Linux APIs must build and pass their tests on Windows too.

You need the Rust toolchain pinned in `rust-toolchain.toml` (rustup installs it on first use).

```shell
cargo build --workspace
cargo test --workspace
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo deny check
```

## Making a change

1. Fork the repository and create a branch named `feat/<topic>`, `fix/<topic>`, `docs/<topic>`, `ci/<topic>` or `chore/<topic>`.
2. Write the test first, then the code. Table-driven tests where there is a table. No network in unit tests.
3. Keep comments short and about why, not what.
4. Every error a client sees must be a Kubernetes `Status` object with the right code and reason.
5. Never log keys, certificates, tokens or Secret values.
6. Add a line under `Unreleased` in `CHANGELOG.md` for anything users will notice.
7. Use [Conventional Commits](https://www.conventionalcommits.org/) for commit messages, for example `feat(store): add compaction`.
8. Open a pull request against `main` and fill in the template.

CI must pass before a pull request is merged. Pull requests from forks need a maintainer to approve the CI run first, so expect a short wait on your first one. Pull requests are merged with a squash, so one pull request becomes one commit on `main`.

## Reporting bugs and asking for features

Use the issue templates. For questions and ideas that are not yet a concrete task, use [Discussions](https://github.com/raven-clown/flotilla/discussions).

Security problems go through [SECURITY.md](SECURITY.md), never a public issue.

## License

By contributing you agree that your contribution is licensed under the [Apache License 2.0](LICENSE), the same as the rest of the project.
