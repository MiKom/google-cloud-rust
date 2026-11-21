# Plan: Data Portability API client for Rust (fork-only)

Goal: a generated `google-dataportability-v1` crate for the
[Data Portability API](https://developers.google.com/data-portability/user-guide/overview),
built entirely in personal forks with no upstreaming:

- librarian fork: <https://github.com/MiKom/librarian>
- Rust fork: <https://github.com/MiKom/google-cloud-rust>

## Facts

- **No protos.** googleapis has no `google/dataportability`. The client is
  generated from the discovery document (as is the C# client).
- **Discovery document available.** `discoveries/dataportability.v1.json`
  exists at the discovery commit pinned in `librarian.yaml` (`066be1f6`,
  revision `20260719`). No change to the pinned sources is needed.
- **Template crate:** `google-cloud-dns-v1` (`src/generated/cloud/dns/v1`,
  added in `52ed78888`, tests in `d8a069e7a`).
- **Generator doesn't know the API.** librarian's
  `internal/serviceconfig/sdk.yaml` only has discovery entries for Compute and
  DNS, and that file is compiled into the binary, hence the librarian fork.
- **Small surface:** 6 methods, 11 schemas, 73 OAuth scopes, no long-running
  operations (callers poll `getPortabilityArchiveState`).

| Resource             | Method                       | HTTP                                    |
| -------------------- | ---------------------------- | --------------------------------------- |
| `portabilityArchive` | `initiate`                   | `POST v1/portabilityArchive:initiate`   |
| `archiveJobs`        | `getPortabilityArchiveState` | `GET v1/{+name}`                        |
| `archiveJobs`        | `cancel` / `retry`           | `POST v1/{+name}:cancel` / `:retry`     |
| `accessType`         | `check`                      | `POST v1/accessType:check`              |
| `authorization`      | `reset`                      | `POST v1/authorization:reset` → `Empty` |

## Decisions

- **Crate name:** `google-dataportability-v1`, from librarian path
  `google/dataportability/v1`, output in `src/generated/dataportability/v1`.
  This is the name upstream would most likely choose, so upstreaming later would
  not need a rename.
- **Version:** `0.1.0`. Never published, so it only matters to `Cargo.lock`.
- **Branches:** `dataportability` in both forks; `main` in each mirrors
  upstream.

## Phase 1: librarian fork

1. Clone the fork and branch from the commit this repo pins
   (`version: v0.53.1-0.20261120161107-40c5096357bb`, i.e. commit
   `40c5096357bb`). Keep the change small.
1. Add to `internal/serviceconfig/sdk.yaml`:
   ```yaml
   - path: google/dataportability/v1
     discovery: discoveries/dataportability.v1.json
     short_name: dataportability
     service_name: dataportability.googleapis.com
     title: Data Portability API
     documentation_uri: https://developers.google.com/data-portability
   ```
1. Build a local binary. `go run github.com/MiKom/librarian/...@ref` does not
   work because the fork's `go.mod` still declares
   `module github.com/googleapis/librarian`.
   ```sh
   cd <librarian-checkout> && go build -o <bin>/librarian-dp ./cmd/librarian
   ```
1. Fix templates only if Phase 2 fails. Watch for:
   - Discovery resources named like single objects (`accessType`,
     `authorization`, `portabilityArchive`) becoming services.
   - `{+name}` path templates and action-verb methods (`:check`, `:reset`,
     `:initiate`).
   - `Empty` mapping to `wkt::Empty`.
   - The open string enum `state`; unknown values must round-trip.
1. If librarian rejects the binary because it doesn't match `version:` in
   `librarian.yaml` (unverified whether it checks), set `version:` in the Rust
   fork to the fork's commit.

## Phase 2: generate the crate (Rust fork, `dataportability` branch)

1. From the Rust repo root:
   ```sh
   <bin>/librarian-dp add google/dataportability/v1
   <bin>/librarian-dp generate google-dataportability-v1
   <bin>/librarian-dp tidy
   ```
1. The `librarian.yaml` entry should match DNS without the LRO parts (no
   `keep: src/operation.rs`, `pollers`, or `google-cloud-lro`):
   ```yaml
     - name: google-dataportability-v1
       version: 0.1.0
       copyright_year: "2026"
       roots: [discovery, googleapis]
       specification_format: discovery
       rust:
         generate_rpc_samples: "false"
   ```
1. Expected changes: root `Cargo.toml` (workspace member plus a
   `default-features = false` workspace dependency), `Cargo.lock`,
   `librarian.yaml`, `src/generated/dataportability/v1/**`. Never edit generated
   files by hand; fix librarian and regenerate.
1. Run checks locally (the project's Cloud Build CI doesn't run on forks):
   ```sh
   cargo fmt -p google-dataportability-v1 -- --check
   RUSTFLAGS="-D warnings" cargo check -p google-dataportability-v1 --lib
   RUSTDOCFLAGS="-D warnings" cargo doc -p google-dataportability-v1 --no-deps
   cargo clippy -p google-dataportability-v1 --all-targets -- -D warnings
   cargo test -p google-dataportability-v1
   ```
1. Commit as a few isolated commits (librarian config, generated code, tests)
   so rebases stay easy.

## Phase 3: authentication (docs and example)

Data Portability accepts **only end-user OAuth tokens** with `dataportability.*`
scopes. Service accounts and the default `cloud-platform` scope fail, so the
generated quickstart (`Client::builder().build()`) won't work as written.

- Use `google_cloud_auth::credentials::user_account::Builder::new(authorized_user_json).with_scopes([...]).build()`
  (see `src/auth/src/credentials/user_account.rs`).
- The auth crate doesn't run the browser consent flow. The app obtains and
  stores the refresh token itself.
- The `resources` passed to `initiate` correspond 1:1 to the granted scopes.
- Put the example in `src/generated/dataportability/v1/examples/` only if
  `keep:` protects it from regeneration; otherwise in `tests/dataportability`
  or the app.

## Phase 4: tests (`tests/dataportability`)

- Workspace member with
  `google-dataportability-v1 = { workspace = true, default-features = true }`.
  Without `default-features = true`, no rustls crypto provider is configured and
  `reqwest` panics at runtime.
- **Mock tests** (pattern: `tests/discovery`, `tests/dns`): mock HTTP server
  checking paths, verbs and bodies for all 6 methods, plus `Empty` and unknown
  `state` values.
- **Live tests** behind a `run-integration-tests` feature, reading user
  credentials from an environment variable: `accessType.check`, then
  `initiate` → poll → `cancel` on one small resource. Leave
  `authorization.reset` out unless deliberately tested; it revokes the token's
  granted access.
- Optional: a GitHub Actions workflow in the fork running fmt, clippy and mock
  tests.

## Phase 5: use it in the app

```toml
[dependencies]
google-dataportability-v1 = { git = "https://github.com/MiKom/google-cloud-rust", branch = "dataportability", features = ["default-rustls-provider"] }
google-cloud-auth = { git = "https://github.com/MiKom/google-cloud-rust", branch = "dataportability" }
```

- Take **every** Google crate (gax, auth, wkt) from the same fork. Mixing in
  crates.io versions gives duplicate, mismatched types (e.g. credentials). If
  other Google crates come from crates.io, redirect them with
  `[patch.crates-io]`.
- Pin with `rev = "<sha>"` once it works.
- Don't publish to crates.io under a `google-*` name.

## Phase 6 (optional): helpers

In the app, or a separate non-generated crate in the fork:

- poll-until-finished helper built on `getPortabilityArchiveState`
- archive download from the signed `urls` with `reqwest`
- typed constants for the 73 scopes

## Maintenance

- **Upstream sync:** update `main` from upstream in both forks, rebase
  `dataportability`, rebuild the librarian binary, regenerate. Conflicts are
  mostly in `Cargo.toml`, `Cargo.lock`, `librarian.yaml`; regenerate the crate
  rather than merging it by hand.
- **Upstream librarian bumps:** when upstream changes `version:` in
  `librarian.yaml`, rebase the librarian fork onto that commit before
  regenerating.
- **Discovery document updates** arrive via upstream bumps of
  `sources.discovery`; regenerate and rerun the mock tests.

## Risks

1. **Generator gaps:** Compute and DNS are the only discovery clients so far;
   Phase 1 step 4 may need template fixes. This is the main effort unknown.
1. **Fork drift:** rebase about monthly.
1. **Auth:** the main source of runtime problems; test with real user
   credentials early.
