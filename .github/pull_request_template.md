## What this changes

<!-- One or two sentences. Link the issue: Closes #123 -->

## How it was tested

<!-- Tests added or changed, and any manual check (kubectl, kube-rs, WSL). -->

## Checklist

- [ ] Tests written first and passing (`cargo test --workspace`)
- [ ] `cargo fmt --check` and `cargo clippy --workspace --all-targets -- -D warnings` are clean
- [ ] Client-facing errors are Kubernetes `Status` objects
- [ ] No keys, tokens or Secret values in logs, errors or fixtures
- [ ] `CHANGELOG.md` updated under Unreleased if users will notice
- [ ] Docs updated if behaviour or config changed
