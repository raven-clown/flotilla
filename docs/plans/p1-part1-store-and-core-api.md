# P1 part 1: store and core API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `flotilla server` starts a one-voter Raft store and serves enough of the Kubernetes API over mutual TLS that kubectl and kube-rs can create, get, list, watch, patch and delete Namespaces, ConfigMaps and Secrets.

**Architecture:** A Cargo workspace. `store` is an MVCC key-value store on redb, wrapped in openraft with a single voter. `registry` turns stored bytes into Kubernetes objects (metadata, defaults, validation, patch, Secret encryption). `apiserver` is a thin axum layer that maps Kubernetes REST paths onto the registry. `controllers` holds the namespace lifecycle loop. The `flotilla` binary wires them together.

**Tech Stack:** Rust (edition 2024), tokio, axum, hyper, rustls, tokio-rustls, rcgen, openraft 0.9, redb, serde, serde_json, k8s-openapi, json-patch, aes-gcm, clap, tracing, prometheus-client, async-trait, thiserror, tokio-util, uuid, base64, dirs, anyhow (binary only). Dev: kube (client and runtime), tempfile, tower.

**Spec:** `docs/design.md` (sections 3.1, 3.2, 3.4, 4.1 row P1, 8, 9, 11).

**Where this sits in P1:** P1 is split into plans that each ship something testable.
1. **This plan:** store and core API (Namespace, ConfigMap, Secret).
2. Pods, Nodes, Events and ServiceAccounts, plus the scheduler and the kubelet join with `flotilla node`.
3. Deployment, ReplicaSet, Service, EndpointSlice and the GC controllers, plus server-side apply.
4. flotilla CNI with WireGuard, with kube-proxy and CoreDNS bundled.
5. Short config (`apply`, `render`), `migrate --dry-run`, `dev cluster` and the P1 benchmarks.

## Global Constraints

- Rust edition 2024. `rust-toolchain.toml` pins an exact stable version, 1.89 or newer (the kube 4.x MSRV).
- `KUBERNETES_VERSION` at the repo root holds the target minor version on one line (the newest minor that the latest `k8s-openapi` supports). The `k8s-openapi` feature flag matches it.
- Every crate in this plan builds and passes `cargo test` on both Linux and Windows. Nothing here uses Linux-only APIs.
- No network in unit tests. Integration tests bind `127.0.0.1:0` only.
- Library crates never print and never touch stdout. They log with `tracing`.
- Every error a client sees is a Kubernetes `Status` JSON object with the right `code` and `reason`.
- Keys, certificates, tokens and Secret values never appear in logs, errors or test fixtures that get printed.
- Store keys follow the upstream etcd layout: `/registry/<resource>/<namespace>/<name>` for the core group, `/registry/namespaces/<name>` for cluster-scoped kinds.
- `resourceVersion` is the store's `mod_revision` as a decimal string. It is set on read and never stored inside the object bytes.
- Work on branch `feat/p1-store-api`. Conventional Commits. No trailers of any kind. No em-dashes. Comments only for why.
- Never mention AI, assistants or code generation anywhere in the repo.

## Review Focus

1. **Stale writes.** An update or delete with an old `resourceVersion` must return 409 Conflict and leave the object unchanged. Two clients editing at once is the normal case. Tests: Task 6 `update_with_stale_rv_conflicts`, Task 8 `put_with_stale_rv_returns_409`.
2. **Watchers that fall behind.** A watch or a `continue` token from before compaction must get 410 with reason `Expired`, so the client relists, and never a silent gap. Tests: Task 4 `watch_before_compaction_is_compacted`, Task 9 `watch_from_compacted_rv_sends_expired_error`.
3. **Restart in the middle.** After a restart, revisions keep rising from where they were, and a watch from a revision before the restart replays what came after it. Test: Task 5 `restart_keeps_revisions_and_history`.
4. **Patches we do not understand.** A strategic merge patch directive that is not supported must fail loudly with 422 and must not apply part of the patch. Test: Task 7 `unknown_smp_directive_is_rejected`.
5. **Writes into a namespace that is missing or terminating.** These must fail with 404 NotFound or 403 Forbidden ("is being terminated"), never create orphans. Tests: Task 6 `create_in_missing_namespace_is_not_found`, Task 11 `create_in_terminating_namespace_is_forbidden`.

---

### Task 1: Workspace, CI and repo rules

**Files:**
- Create: `Cargo.toml` (workspace), `rust-toolchain.toml`, `KUBERNETES_VERSION`, `deny.toml`, `rustfmt.toml`, `.gitignore`, `.gitattributes` (`* text=auto eol=lf`)
- Create: `crates/flotilla/Cargo.toml`, `crates/flotilla/src/main.rs`
- Create: `crates/{api,store,registry,apiserver,controllers}/Cargo.toml` and `src/lib.rs` (empty libraries)
- Create: `.github/workflows/ci.yml`
- Create: `AGENTS.md`, `CLAUDE.md` (contains only `@AGENTS.md`), `CHANGELOG.md`
- Test: `crates/flotilla/tests/cli.rs`

**Interfaces:**
- Produces: workspace members `flotilla`, `flotilla-api`, `flotilla-store`, `flotilla-registry`, `flotilla-apiserver`, `flotilla-controllers`. Shared dependency versions live in `[workspace.dependencies]`.

- [ ] **Step 1: Write the failing test**

```rust
#[test]
fn version_prints_package_version() {
    let out = std::process::Command::new(env!("CARGO_BIN_EXE_flotilla"))
        .arg("--version").output().unwrap();
    assert!(out.status.success());
    assert_eq!(String::from_utf8(out.stdout).unwrap().trim(),
               format!("flotilla {}", env!("CARGO_PKG_VERSION")));
}
```

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla --test cli`
Expected: FAIL, because the binary has no clap parser yet.

- [ ] **Step 3: Add a clap `Cli` with `#[command(version)]` and an empty `Command` enum in `crates/flotilla/src/main.rs`**

- [ ] **Step 4: Write CI and the repo docs**

`ci.yml` runs on `push` to main and on `pull_request`, with these jobs:
- `lint` on ubuntu-latest: `cargo fmt --check` and `cargo clippy --workspace --all-targets -- -D warnings`.
- `test` on ubuntu-latest and windows-latest: `cargo test --workspace`.
- `deny` on ubuntu-latest: `cargo deny check`.
- `coverage` on ubuntu-latest: `cargo llvm-cov --workspace`, reporting only. Task 12 adds `--fail-under-lines 80` once there is real code.

`AGENTS.md` follows the shape of bridge's file (Project, Commands, Layout, Rules). It carries the Global Constraints above as rules, plus "tests first" and "update CHANGELOG under Unreleased".

`CHANGELOG.md` starts with an empty `## Unreleased` section.

- [ ] **Step 5: Run everything**

Run: `cargo fmt --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace`
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml rust-toolchain.toml KUBERNETES_VERSION deny.toml rustfmt.toml .gitignore .gitattributes crates .github AGENTS.md CLAUDE.md CHANGELOG.md
git commit -m "chore: set up cargo workspace, ci and repo rules"
```

### Task 2: Resource registry and Status errors

**Files:**
- Create: `crates/api/src/resource.rs`, `crates/api/src/error.rs`
- Modify: `crates/api/src/lib.rs`
- Test: inline `#[cfg(test)]` modules in both files

**Interfaces:**
- Produces:
  - `pub struct ResourceInfo { pub group: &'static str, pub version: &'static str, pub resource: &'static str, pub singular: &'static str, pub kind: &'static str, pub list_kind: &'static str, pub short_names: &'static [&'static str], pub namespaced: bool, pub verbs: &'static [&'static str] }`
  - `pub fn served() -> &'static [ResourceInfo]`, which returns namespaces, configmaps and secrets, all in group `""` version `v1`.
  - `pub fn lookup(group: &str, version: &str, resource: &str) -> Option<&'static ResourceInfo>`
  - `impl ResourceInfo { pub fn key(&self, namespace: Option<&str>, name: &str) -> String; pub fn prefix(&self, namespace: Option<&str>) -> String; pub fn api_version(&self) -> String }`
  - `pub enum Reason { BadRequest, Unauthorized, Forbidden, NotFound, AlreadyExists, Conflict, Invalid, Expired, MethodNotAllowed, UnsupportedMediaType, NotAcceptable, InternalError, ServiceUnavailable }`
  - `pub struct ApiError { pub code: u16, pub reason: Reason, pub message: String, pub details: Option<Details> }` and `pub struct Details { pub name: String, pub group: String, pub kind: String }`
  - Constructors: `ApiError::not_found(info: &ResourceInfo, name: &str)`, `already_exists(info, name)`, `conflict(info, name, msg: &str)`, `invalid(info, name, causes: Vec<(String, String)>)`, `expired(msg: &str)`, `forbidden(msg: &str)`, `unauthorized()`, `bad_request(msg: &str)`, `unsupported_media_type(ct: &str)`, `internal(msg: &str)`, `unavailable(msg: &str)`.
  - `impl ApiError { pub fn to_status(&self) -> k8s_openapi::apimachinery::pkg::apis::meta::v1::Status }`

- [ ] **Step 1: Write the failing tests**

Write a table-driven test `status_json_matches_upstream`. Each row has a constructor call, then the expected `code`, `reason` string and `message`. Copy the messages from upstream:
- `not_found(configmaps, "a")` gives 404, `"NotFound"`, `configmaps "a" not found`, with details `{name: "a", kind: "configmaps"}`.
- `already_exists` gives 409, `"AlreadyExists"`, `configmaps "a" already exists`.
- `conflict` gives 409, `"Conflict"`, `Operation cannot be fulfilled on configmaps "a": <msg>`.
- `invalid` gives 422, `"Invalid"`, `ConfigMap "a" is invalid: <field>: <msg>`.
- `expired("too old resource version: 5 (9)")` gives 410, `"Expired"`.

Every row also checks `status == "Failure"`, `kind == "Status"` and `apiVersion == "v1"` in the serialized JSON.

Also write:
- `keys_follow_etcd_layout`: `configmaps.key(Some("ns"), "a") == "/registry/configmaps/ns/a"`, `namespaces.key(None, "ns") == "/registry/namespaces/ns"`, and `configmaps.prefix(Some("ns")) == "/registry/configmaps/ns/"`.
- `lookup_finds_served_and_rejects_unknown`.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-api`
Expected: FAIL to compile, because the items are missing.

- [ ] **Step 3: Implement `resource.rs` and `error.rs` to the signatures above**

The verbs for all three kinds are `create, delete, deletecollection, get, list, patch, update, watch`. The short names are `ns` and `cm`. Secrets has none.

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-api`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/api
git commit -m "feat(api): add served resource table and Status errors"
```

### Task 3: MVCC store on redb

**Files:**
- Create: `crates/store/src/types.rs`, `crates/store/src/mvcc.rs`
- Modify: `crates/store/src/lib.rs`
- Test: `crates/store/tests/mvcc.rs`

**Interfaces:**
- Produces:
  - `pub type Revision = u64;`
  - `pub struct KeyValue { pub key: String, pub value: Vec<u8>, pub create_revision: Revision, pub mod_revision: Revision }`
  - `pub struct ListOptions { pub limit: Option<usize>, pub start_after: Option<String>, pub at_revision: Option<Revision> }`
  - `pub struct ListResult { pub items: Vec<KeyValue>, pub revision: Revision, pub more: bool }`
  - `pub enum WatchEvent { Put { kv: KeyValue, prev: Option<KeyValue> }, Delete { prev: KeyValue, revision: Revision } }`
  - `pub type WatchStream = Pin<Box<dyn Stream<Item = Result<WatchEvent, StoreError>> + Send>>;`
  - `#[derive(thiserror::Error)] pub enum StoreError { NotFound, AlreadyExists, Conflict { current: Revision }, Compacted { compact_revision: Revision }, Unavailable(String), Internal(String) }`
  - The `#[async_trait] pub trait Store: Send + Sync` methods:
    - `create(&self, key: &str, value: Vec<u8>) -> Result<KeyValue, StoreError>`
    - `update(&self, key: &str, value: Vec<u8>, expected: Option<Revision>) -> Result<KeyValue, StoreError>`
    - `delete(&self, key: &str, expected: Option<Revision>) -> Result<KeyValue, StoreError>`
    - `get(&self, key: &str) -> Result<Option<KeyValue>, StoreError>`
    - `list(&self, prefix: &str, opts: ListOptions) -> Result<ListResult, StoreError>`
    - `watch(&self, prefix: &str, after: Revision) -> Result<WatchStream, StoreError>`
    - `compact(&self, upto: Revision) -> Result<(), StoreError>`
    - `current_revision(&self) -> Revision`
  - `pub struct MvccStore`, with `MvccStore::open(path: &Path) -> Result<Self, StoreError>`, implementing `Store`.

`MvccStore` keeps these redb tables:
- `latest: key -> (create_rev, mod_rev)`
- `versions: (key, rev) -> bytes or tombstone`
- `history: rev -> (key, op)`
- `meta: "current" and "compacted" -> u64`

Each write is one redb transaction that bumps `current` by one. `list` with `at_revision` reads `versions` at the newest rev that is not above it. `list` sorts by key, honours `limit` and `start_after`, and sets `more`.

In this task, `watch` may return `StoreError::Internal("not implemented")`. Task 4 fills it in.

- [ ] **Step 1: Write the failing tests**

Write these table-driven where it fits:
- `create_then_get_returns_value_and_revisions`: the first create on an empty store has `create_revision == mod_revision == 1`.
- `create_existing_is_already_exists`
- `update_with_matching_revision_bumps_mod_revision`
- `update_with_stale_revision_is_conflict`: the error carries the current revision.
- `delete_missing_is_not_found`
- `list_is_sorted_paged_and_flags_more`: with 5 keys and `limit = 2`, the pages are `[a,b] more`, `[c,d] more`, `[e] no more`.
- `list_at_old_revision_sees_old_values`
- `prefix_does_not_match_sibling`: the prefix `/registry/configmaps/ns/` must not return `/registry/configmaps/ns2/x`.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-store --test mvcc`
Expected: FAIL to compile.

- [ ] **Step 3: Implement `types.rs` and `mvcc.rs`**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-store --test mvcc`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/store
git commit -m "feat(store): add MVCC key-value store on redb"
```

### Task 4: Watch and compaction

**Files:**
- Modify: `crates/store/src/mvcc.rs`
- Test: `crates/store/tests/watch.rs`

**Interfaces:**
- Consumes: `MvccStore`, `WatchEvent`, `StoreError::Compacted` from Task 3.
- Produces: `MvccStore::watch` and `MvccStore::compact` fully working. Every committed write is announced on a `tokio::sync::broadcast` channel.

A watch first replays `history` entries with revision `> after` whose key matches the prefix, then follows the broadcast. It never skips or duplicates a revision across the switch. If the broadcast lags, the stream ends with `StoreError::Compacted` so the client relists.

`compact(upto)` drops old `versions` entries and `history` entries at or below `upto`, but keeps the newest version of each live key. After that, a `watch` or `list` with a revision below `upto` returns `Compacted`.

- [ ] **Step 1: Write the failing tests**

- `watch_from_zero_after_writes_replays_all_then_follows`: three writes, then watch from 0, then a fourth write. The stream yields revisions 1, 2, 3, 4 in order.
- `watch_only_matches_prefix`
- `delete_event_carries_previous_value`
- `watch_before_compaction_is_compacted`: write 5 values, `compact(3)`, then `watch(after = 2)` gives `Compacted { compact_revision: 3 }`.
- `list_at_compacted_revision_is_compacted`
- `no_gap_between_replay_and_live`: a writer task makes 1,000 writes while the watch starts partway through. The test collects 1,000 events and checks that the revisions are strictly increasing with no gaps from the start revision.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-store --test watch`
Expected: FAIL.

- [ ] **Step 3: Implement watch and compact**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-store`
Expected: PASS, including the Task 3 tests.

- [ ] **Step 5: Commit**

```bash
git add crates/store
git commit -m "feat(store): add watch with history replay and compaction"
```

### Task 5: Raft wrapper with one voter

**Files:**
- Create: `crates/store/src/raft/mod.rs`, `crates/store/src/raft/log.rs`, `crates/store/src/raft/state_machine.rs`, `crates/store/src/raft/command.rs`
- Modify: `crates/store/src/lib.rs`
- Test: `crates/store/tests/raft.rs`

**Interfaces:**
- Consumes: `Store`, `MvccStore` from Tasks 3 and 4.
- Produces:
  - `pub enum Command { Create { key: String, value: Vec<u8> }, Update { key: String, value: Vec<u8>, expected: Option<Revision> }, Delete { key: String, expected: Option<Revision> }, Compact { upto: Revision } }`, serialized with serde.
  - `pub struct RaftStore`, with `RaftStore::open(dir: &Path, node_id: u64) -> Result<Self, StoreError>`. This initializes a one-member cluster on first open and reuses it on later opens. `RaftStore` implements `Store`.
  - `RaftStore::shutdown(self) -> Result<(), StoreError>`

Use the openraft 0.9 storage v2 traits (`RaftLogStorage`, `RaftStateMachine`). The log and vote live in `dir/raft.redb`, and the state machine is an `MvccStore` at `dir/state.redb`. Writes go through `client_write`. Reads (`get`, `list`) call `ensure_linearizable` first and then read the local `MvccStore`. `watch` reads the local `MvccStore` directly. Store errors that the state machine returns (Conflict, NotFound, AlreadyExists) come back to the caller unchanged and are not counted as Raft failures. Raft snapshots are a P2 task. Here the state machine's snapshot builder returns the redb file bytes, and one test covers it.

- [ ] **Step 1: Write the failing tests**

- `writes_go_through_raft_and_read_back`
- `conflict_from_state_machine_reaches_caller`
- `restart_keeps_revisions_and_history`: open, write 3 values, shutdown, reopen, write 1 more. The test checks the last revision is 4 and that `watch(after = 1)` replays revisions 2, 3, 4.
- `snapshot_round_trips_state`: build a snapshot, install it into a fresh state machine, and check a list matches.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-store --test raft`
Expected: FAIL.

- [ ] **Step 3: Implement the raft module**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-store`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/store
git commit -m "feat(store): run the store through openraft with one voter"
```

### Task 6: Object registry with metadata, validation and Secret encryption

**Files:**
- Create: `crates/registry/src/lib.rs`, `crates/registry/src/meta.rs`, `crates/registry/src/validate.rs`, `crates/registry/src/kinds.rs`, `crates/registry/src/encrypt.rs`
- Test: `crates/registry/tests/registry.rs`

**Interfaces:**
- Consumes: `ResourceInfo`, `ApiError` (Task 2), and `Store`, `StoreError`, `Revision`, `WatchStream` (Tasks 3 to 5).
- Produces:
  - `pub struct Registry`, with `Registry::new(store: Arc<dyn Store>, cipher: Option<SecretCipher>) -> Self`
  - Methods. Objects are `serde_json::Value` with `apiVersion`, `kind` and `metadata`:
    - `create(&self, info: &ResourceInfo, ns: Option<&str>, obj: Value) -> Result<Value, ApiError>`
    - `get(&self, info, ns, name: &str) -> Result<Value, ApiError>`
    - `list(&self, info, ns: Option<&str>, opts: ListParams) -> Result<ObjectList, ApiError>`
    - `update(&self, info, ns, name, obj: Value) -> Result<Value, ApiError>`. It uses `metadata.resourceVersion` as the precondition when set.
    - `delete(&self, info, ns, name, precondition_rv: Option<Revision>) -> Result<Value, ApiError>`
    - `watch(&self, info, ns, after: Revision) -> Result<ObjectWatch, ApiError>`
  - `pub struct ListParams { pub limit: Option<usize>, pub continue_token: Option<String>, pub label_selector: Option<String>, pub field_selector: Option<String> }`
  - `pub struct ObjectList { pub items: Vec<Value>, pub resource_version: String, pub continue_token: Option<String> }`
  - `pub enum ObjectEvent { Added(Value), Modified(Value), Deleted(Value) }` and `pub type ObjectWatch = Pin<Box<dyn Stream<Item = Result<ObjectEvent, ApiError>> + Send>>`
  - `pub struct SecretCipher`, with `SecretCipher::from_key(key: [u8; 32], key_name: &str) -> Self`, `encrypt(&self, plain: &[u8]) -> Vec<u8>` and `decrypt(&self, stored: &[u8]) -> Result<Vec<u8>, ApiError>`. The stored prefix is `k8s:enc:aesgcm:v1:<key_name>:` followed by a 12-byte nonce and the ciphertext, the same as upstream's aesgcm provider.

What the registry does to objects:
- **On create:** sets `uid` (uuid v4), `creationTimestamp` (RFC 3339, seconds) and `generation = 1`. Fills `name` from `generateName` plus 5 random lowercase alphanumerics. Removes any client-sent `resourceVersion`.
- **On read:** sets `metadata.resourceVersion`.
- **On update:** keeps `uid` and `creationTimestamp` from the stored object.
- **Defaults:** a Secret's `type` is `Opaque`. A Secret's `stringData` merges into `data` as base64 and is then removed. A Namespace gets `spec.finalizers = ["kubernetes"]` and `status.phase = "Active"`.
- **Validation:**
  - Names: DNS-1123 subdomain for ConfigMap and Secret, DNS-1123 label for Namespace.
  - ConfigMap and Secret keys match `[-._a-zA-Z0-9]+` and are at most 253 characters.
  - Secret `data` values are valid base64.
  - The total size of a ConfigMap or Secret is at most 1 MiB.
  - `metadata.namespace` must match the path namespace.
- **Namespaced create** checks that the namespace exists (404 if not).
- **Label selectors:** support `=`, `==`, `!=`, `in`, `notin`, `key` and `!key`.
- **Field selectors:** support `metadata.name` and `metadata.namespace` with `=` and `!=`.
- **Continue tokens** are base64 JSON `{"rv": <list revision>, "start": "<last key>"}`. They pin the list revision.

- [ ] **Step 1: Write the failing tests**

Each test runs on a `MvccStore` in a tempdir:
- `create_sets_server_metadata`
- `generate_name_produces_unique_names`
- `create_in_missing_namespace_is_not_found`
- `update_with_stale_rv_conflicts`
- `update_keeps_uid_and_creation_timestamp`
- `invalid_names_and_keys_are_rejected`: a table of bad inputs. Each must give 422 with a cause on the right field.
- `secret_string_data_moves_to_data`
- `secret_is_encrypted_at_rest`: create a Secret with value `"hunter2"`. Read the raw bytes straight from the store, and check that they start with `k8s:enc:aesgcm:v1:key1:` and do not contain `aHVudGVyMg==` or `hunter2`. A registry `get` returns the original data.
- `list_pages_with_continue_token_at_fixed_revision`: list page 1, write a new object, then list page 2. Page 2 must not include the new object.
- `label_selector_table`

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-registry`
Expected: FAIL.

- [ ] **Step 3: Implement the registry modules to the signatures above**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-registry`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/registry
git commit -m "feat(registry): store Kubernetes objects with metadata, validation and secret encryption"
```

### Task 7: Patch

**Files:**
- Create: `crates/registry/src/patch.rs`
- Modify: `crates/registry/src/lib.rs`
- Test: `crates/registry/tests/patch.rs`

**Interfaces:**
- Consumes: `Registry`, `ApiError`.
- Produces:
  - `pub enum PatchType { Json, Merge, StrategicMerge }`, with `PatchType::from_content_type(ct: &str) -> Result<Self, ApiError>` for `application/json-patch+json`, `application/merge-patch+json` and `application/strategic-merge-patch+json`. `application/apply-patch+yaml` and anything else return 415.
  - `Registry::patch(&self, info, ns, name, patch_type: PatchType, body: &[u8]) -> Result<Value, ApiError>`. It reads, patches, validates and writes with the read revision as the precondition, and retries up to 5 times on Conflict when the patch did not carry its own `resourceVersion`.
  - `pub fn strategic_merge(original: &Value, patch: &Value) -> Result<Value, ApiError>`

The strategic merge patch in this plan has map merge semantics (a `null` value deletes the key), `$patch: replace` and `$patch: delete` on maps, and replaces lists whole. Any other key starting with `$` (such as `$setElementOrder/...`, `$retainKeys` or `$deleteFromPrimitiveList/...`) returns 422 Invalid with the directive named in the message, and nothing is written. The kinds in this plan have no lists with merge keys, so this subset is enough for `kubectl apply`. Merge keys come in plan 3 with Deployments.

- [ ] **Step 1: Write the failing tests**

- `json_patch_ops_table`: add, remove, replace, test (a failing `test` gives 422), move and copy, on a ConfigMap.
- `merge_patch_null_deletes_key`
- `smp_merges_maps_and_replaces_lists`
- `smp_patch_replace_and_delete_directives`
- `unknown_smp_directive_is_rejected`: the patch `{"data": {"$retainKeys": ["a"]}}` gives 422, and a get afterwards shows the object unchanged.
- `patch_with_rv_in_body_conflicts_when_stale`
- `kubectl_apply_style_patch_updates_last_applied_annotation`: an SMP body that sets `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]` and changes `data`. Both are applied.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-registry --test patch`
Expected: FAIL.

- [ ] **Step 3: Implement `patch.rs`**

Use the `json-patch` crate for JSON patch and merge patch.

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-registry`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/registry
git commit -m "feat(registry): add json, merge and strategic merge patch"
```

### Task 8: HTTP API: discovery, OpenAPI and CRUD

**Files:**
- Create: `crates/apiserver/src/lib.rs`, `crates/apiserver/src/discovery.rs`, `crates/apiserver/src/openapi.rs`, `crates/apiserver/src/handlers.rs`, `crates/apiserver/src/response.rs`
- Create: `crates/apiserver/openapi/v2.json`, `crates/apiserver/openapi/v3-api-v1.json` (vendored)
- Create: `scripts/vendor-openapi.sh`, which downloads `api/openapi-spec/swagger.json` and `api/openapi-spec/v3/api__v1_openapi.json` from the kubernetes repo at the tag in `KUBERNETES_VERSION`, and keeps only the paths and definitions for the served kinds plus `io.k8s.apimachinery.*`
- Test: `crates/apiserver/tests/http.rs`

**Interfaces:**
- Consumes: `Registry`, `PatchType`, `ApiError`, `served()`, `lookup()`.
- Produces:
  - `pub struct ApiState { pub registry: Arc<Registry>, pub version: VersionInfo }`
  - `pub fn router(state: ApiState) -> axum::Router`. It has no TLS and no auth. Task 10 wraps it.
  - `impl axum::response::IntoResponse for ApiErrorResponse`, a newtype over `ApiError`.

Routes:
- `GET /version`
- `GET /api` (`APIVersions` with `versions: ["v1"]`)
- `GET /api/v1` (`APIResourceList` from `served()`, including the `namespaces/status` and `namespaces/finalize` subresources)
- `GET /apis` (an empty `APIGroupList`)
- `GET /openapi/v2`
- `GET /openapi/v3` and `GET /openapi/v3/api/v1`
- `GET|POST /api/v1/{resource}`
- `GET|POST|DELETE /api/v1/namespaces/{ns}/{resource}`
- `GET|PUT|PATCH|DELETE /api/v1/namespaces/{ns}/{resource}/{name}`
- `GET|PUT|PATCH|DELETE /api/v1/{resource}/{name}` for namespaces

Behaviour:
- `DELETE` returns the deleted object, or a `Status` with `status: Success` for a collection delete.
- Query parameters: `limit`, `continue`, `labelSelector`, `fieldSelector`, `resourceVersion` (accepted on list), `dryRun=All` (validate without writing), `fieldManager` (accepted and ignored until server-side apply) and `fieldValidation`. For `fieldValidation`, `Ignore` and `Warn` drop unknown fields, and `Warn` also sends a `Warning` header per field. `Strict` returns 400 BadRequest listing the unknown fields, which are found by comparing the body with the kind's `k8s-openapi` type after a serde round trip.
- A request whose `Accept` asks for aggregated discovery (`as=APIGroupDiscoveryList`) gets the legacy response with `Content-Type: application/json`, which kubectl accepts as a fallback.
- A `Content-Type` of protobuf gives 415.

- [ ] **Step 1: Write the failing tests**

Each test calls the router through `tower::ServiceExt::oneshot`:
- `discovery_lists_served_resources`: checks the kinds, verbs, `namespaced` flags and short names.
- `openapi_v2_contains_configmap_definition`
- `crud_round_trip_configmap`: POST, then GET, PUT, PATCH (merge), then DELETE, then GET gives 404 with a Status body.
- `put_with_stale_rv_returns_409`
- `cluster_list_across_namespaces`: `GET /api/v1/configmaps` returns items from two namespaces.
- `dry_run_does_not_write`
- `protobuf_content_type_is_415`
- `unknown_resource_is_404_status`
- `strict_field_validation_rejects_unknown_field`: a ConfigMap with `"datta": {}` and `fieldValidation=Strict` gives 400 naming `.datta`. With `Ignore`, it is created without the field.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-apiserver --test http`
Expected: FAIL.

- [ ] **Step 3: Run `scripts/vendor-openapi.sh`, then implement the modules**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-apiserver`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/apiserver scripts/vendor-openapi.sh
git commit -m "feat(apiserver): serve discovery, openapi and crud for core kinds"
```

### Task 9: Watch over HTTP and Table output

**Files:**
- Create: `crates/apiserver/src/watch.rs`, `crates/apiserver/src/table.rs`
- Modify: `crates/apiserver/src/handlers.rs`
- Test: `crates/apiserver/tests/watch.rs`, `crates/apiserver/tests/table.rs`

**Interfaces:**
- Consumes: `Registry::watch`, `ObjectEvent`, `ApiError::expired`.
- Produces: watch handling on the list routes when `watch=true` (or `watch=1`), and Table responses for get and list.

Watch:
- The response is chunked `application/json`, one `{"type": "ADDED"|"MODIFIED"|"DELETED"|"BOOKMARK"|"ERROR", "object": ...}` per line.
- `resourceVersion` sets where the watch starts. Empty or `"0"` means "send the current state as ADDED, then follow".
- With `allowWatchBookmarks=true`, a BOOKMARK carrying only `metadata.resourceVersion` goes out every 60 seconds (this interval can be set for tests).
- `timeoutSeconds` ends the stream.
- A `Compacted` error becomes one `ERROR` event with the 410 Expired Status, and then the stream ends.

Table:
- An `Accept` header containing `as=Table;v=v1;g=meta.k8s.io` returns a `meta.k8s.io/v1 Table`.
- The columns match upstream: namespaces have `NAME, STATUS, AGE`, configmaps have `NAME, DATA, AGE`, and secrets have `NAME, TYPE, DATA, AGE`.
- `includeObject=Metadata` (kubectl's default) puts a `PartialObjectMetadata` in each row.
- AGE uses upstream's short duration format (`5s`, `3m`, `2d`).

- [ ] **Step 1: Write the failing tests**

- `watch_streams_added_modified_deleted`
- `watch_from_zero_sends_current_state_first`
- `watch_from_compacted_rv_sends_expired_error`
- `bookmarks_are_sent_when_allowed`, with the interval set to 50 ms through `ApiState`.
- `timeout_seconds_ends_stream`
- `table_columns_match_upstream`: a table of kind against the expected column names and one example row.
- `short_duration_format_table`: 0s, 59s, 61s (`61s`), 10m, 3h, 2d and 400d (`400d`), matching upstream `duration.HumanDuration`.

- [ ] **Step 2: Run them and see them fail**

Run: `cargo test -p flotilla-apiserver --test watch --test table`
Expected: FAIL.

- [ ] **Step 3: Implement `watch.rs` and `table.rs`**

Add `bookmark_interval: Duration` to `ApiState`, with a default of 60 seconds.

- [ ] **Step 4: Run them and see them pass**

Run: `cargo test -p flotilla-apiserver`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/apiserver
git commit -m "feat(apiserver): add watch streams and table output"
```

### Task 10: TLS, cluster CA and client certificate auth

**Files:**
- Create: `crates/apiserver/src/pki.rs`, `crates/apiserver/src/auth.rs`, `crates/apiserver/src/serve.rs`
- Test: `crates/apiserver/tests/tls.rs`

**Interfaces:**
- Consumes: `router(ApiState)`.
- Produces:
  - `pub struct Pki { pub ca_cert_pem: String, pub server_cert_pem: String, pub server_key_pem: String, pub admin_cert_pem: String, pub admin_key_pem: String }`
  - `Pki::load_or_create(dir: &Path, sans: &[String]) -> Result<Pki, ApiError>`. On first run it creates a CA, a server cert valid for `sans` plus `localhost` and `127.0.0.1`, and an admin client cert with `CN=flotilla-admin` and `O=system:masters`. It writes them with file mode 0600 where the OS supports it, and loads them on later runs. The CA lasts 10 years and leaf certs 1 year.
  - `Pki::kubeconfig(&self, server_url: &str) -> String`, an admin kubeconfig YAML with the cluster, user and context all named `flotilla`.
  - `pub struct User { pub name: String, pub groups: Vec<String> }`, taken from the client certificate's CN and O fields.
  - `pub async fn serve(listener: TcpListener, pki: &Pki, state: ApiState, shutdown: impl Future<Output = ()>) -> io::Result<()>`

Auth rules in this plan:
- No client cert gives 401 Unauthorized as a Status.
- A user without the group `system:masters` gives 403 Forbidden as a Status.
- `/healthz`, `/livez`, `/readyz` and `/version` allow anyone.
- Full RBAC comes in plan 2.

- [ ] **Step 1: Write the failing tests**

- `pki_is_created_once_and_reloaded`
- `server_cert_covers_sans`
- `no_client_cert_is_401`
- `admin_cert_can_list_namespaces`
- `cert_without_masters_group_is_403`: sign an extra cert from the test CA.
- `health_endpoints_are_open`
- `kubeconfig_parses_with_kube_rs`: `kube::config::Kubeconfig::from_yaml`.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-apiserver --test tls`
Expected: FAIL.

- [ ] **Step 3: Implement `pki.rs` with rcgen, and `auth.rs` and `serve.rs` with tokio-rustls**

Pass the verified peer certificate into the axum request extensions.

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p flotilla-apiserver`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/apiserver
git commit -m "feat(apiserver): add cluster pki, mutual tls and admin auth"
```

### Task 11: Namespace lifecycle controller

**Files:**
- Create: `crates/controllers/src/lib.rs`, `crates/controllers/src/namespace.rs`
- Modify: `crates/registry/src/kinds.rs` (namespace delete marks it Terminating)
- Test: `crates/controllers/tests/namespace.rs`

**Interfaces:**
- Consumes: `Registry`, `served()`, `ObjectEvent`.
- Produces:
  - `pub async fn run_namespace_controller(registry: Arc<Registry>, shutdown: CancellationToken)`
  - Changes in the registry:
    - Deleting a Namespace sets `metadata.deletionTimestamp` and `status.phase = "Terminating"` instead of removing it.
    - Creates into a Terminating namespace give 403 Forbidden with the message `namespace <ns> is being terminated` (upstream wording, reason Forbidden).
    - A `PUT .../namespaces/{name}/finalize` with empty `spec.finalizers` removes the Namespace if `deletionTimestamp` is set.

The controller watches Namespaces. For each one with a `deletionTimestamp`, it deletes every namespaced object of every served kind in that namespace, then finalizes the namespace. It requeues with backoff on errors.

- [ ] **Step 1: Write the failing tests**

- `deleting_namespace_removes_contents_then_namespace`
- `create_in_terminating_namespace_is_forbidden`
- `controller_resumes_after_restart`: mark the namespace Terminating while the controller is stopped, then start the controller. It finishes the job.

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla-controllers`
Expected: FAIL.

- [ ] **Step 3: Implement it**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test --workspace`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/controllers crates/registry
git commit -m "feat(controllers): add namespace lifecycle controller"
```

### Task 12: `flotilla server` and end-to-end checks

**Files:**
- Create: `crates/flotilla/src/server.rs`, `crates/flotilla/src/metrics.rs`
- Modify: `crates/flotilla/src/main.rs`
- Create: `scripts/kubectl-smoke.sh`
- Modify: `.github/workflows/ci.yml` (add a `kubectl-smoke` job on ubuntu-latest, and `--fail-under-lines 80` on the coverage job)
- Modify: `README.md` (status line and a "try it" section), `CHANGELOG.md`, `AGENTS.md` (layout)
- Test: `crates/flotilla/tests/e2e_kube.rs`

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `flotilla server [--data-dir <path>] [--listen <addr>] [--advertise <host>]`. The defaults are a `flotilla` data directory under the OS data dir from the `dirs` crate, and `127.0.0.1:6443`.
  - On start the server opens `RaftStore` at `<data>/store`, loads or creates the PKI at `<data>/pki`, and loads or creates the Secret key at `<data>/secret.key` (32 random bytes, mode 0600). It writes `<data>/admin.kubeconfig`. It creates the `default`, `kube-system`, `kube-public` and `kube-node-lease` namespaces if they are missing. It starts the namespace controller and a compaction loop that keeps 5 minutes of history.
  - On SIGINT or Ctrl-C it shuts down cleanly. It logs the kubeconfig path once, and never its contents.
  - `/metrics` (admin only) shows Prometheus text: `flotilla_store_revision`, `flotilla_api_requests_total{verb,resource,code}` and `flotilla_api_request_duration_seconds`.
  - `pub async fn run_server(opts: ServerOptions, shutdown: impl Future<Output = ()>) -> anyhow::Result<()>` in `server.rs`, so tests can run the server in-process.

- [ ] **Step 1: Write the failing tests**

`e2e_kube.rs` starts `run_server` on `127.0.0.1:0` in a tempdir and uses a `kube::Client` built from the written kubeconfig:
- `kube_rs_crud_on_configmaps`: `Api::<ConfigMap>` create, get, list and delete.
- `kube_rs_watcher_sees_changes`: `kube::runtime::watcher` gets the Applied events for a create and an update, and the Deleted event for a delete.
- `default_namespaces_exist`
- `restart_keeps_objects`

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p flotilla --test e2e_kube`
Expected: FAIL.

- [ ] **Step 3: Implement `server.rs`, `metrics.rs` and the `server` subcommand**

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test --workspace`
Expected: PASS on Windows and in WSL.

- [ ] **Step 5: Write `scripts/kubectl-smoke.sh` and the CI job**

The script builds the release binary, starts `flotilla server` in the background with a temp data dir, waits for `/readyz`, and exports `KUBECONFIG`. Then it runs these checks and fails on the first error:
- `kubectl version`
- `kubectl api-resources` (must list configmaps, namespaces and secrets)
- `kubectl create namespace smoke`
- `kubectl -n smoke apply -f` a ConfigMap, then `kubectl apply` again with a changed value (client-side apply through the strategic merge patch)
- `kubectl -n smoke get cm -o json` (checks the value)
- `kubectl -n smoke get cm` (table output)
- `kubectl -n smoke create secret generic s --from-literal=a=b`
- `kubectl -n smoke get secret s -o jsonpath='{.data.a}'` (prints `Yg==`)
- `kubectl -n smoke delete cm --all`
- `kubectl delete namespace smoke --wait --timeout=30s`
- `timeout 5 kubectl get ns -w` (exit code 124 is success)

The CI job installs the kubectl version from `KUBERNETES_VERSION`.

- [ ] **Step 6: Run the smoke script in WSL**

Run: `bash scripts/kubectl-smoke.sh`
Expected: it ends with `smoke ok`.

- [ ] **Step 7: Update the README, CHANGELOG and AGENTS layout, then commit**

The README status changes to "early development: the server stores Namespaces, ConfigMaps and Secrets and kubectl works with them. No workloads run yet." It must not claim anything more.

```bash
git add crates/flotilla scripts/kubectl-smoke.sh .github/workflows/ci.yml README.md CHANGELOG.md AGENTS.md
git commit -m "feat: add flotilla server with kubectl smoke test"
```

### Task 13: First benchmark

**Files:**
- Create: `bench/README.md` (method), `bench/server-rss.sh`, `bench/results/.gitkeep`
- Create: `crates/flotilla/examples/load_configmaps.rs`

**Interfaces:**
- Consumes: `flotilla server`, kube-rs.
- Produces: `bench/server-rss.sh` writes `bench/results/<date>-server-rss.json` with these fields: `idle_rss_mb`, `rss_after_10k_configmaps_mb`, `list_10k_ms_p50`, `watch_event_latency_ms_p50`, `commit`, `kubernetes_version` and `machine`.

The script runs on Linux only (WSL counts). It reads RSS from `/proc/<pid>/status` `VmRSS`. The load example creates 10,000 ConfigMaps of about 1 KiB each, 64 at a time, then times 20 full lists and 200 watch round trips.

- [ ] **Step 1: Write `load_configmaps.rs` and `server-rss.sh`**

- [ ] **Step 2: Run it in WSL**

Run: `bash bench/server-rss.sh`
Expected: it writes a JSON file with every field filled in.

- [ ] **Step 3: Write the method and the first numbers into `bench/README.md`**

Write them as measured, with no comparison claims. The comparison against k3s comes in plan 5.

- [ ] **Step 4: Commit**

```bash
git add bench crates/flotilla/examples/load_configmaps.rs
git commit -m "chore(bench): add server memory and list latency benchmark"
```
