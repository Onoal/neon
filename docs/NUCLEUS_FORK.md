# Nucleus Fork Strategy — Onoal/neon

This document defines how the Neon engine fork is maintained for Nucleus PostgreSQL.

## Repositories

- Official upstream: `https://github.com/neondatabase/neon.git`
- Onoal fork: `https://github.com/Onoal/neon.git` (`origin`)
- Upstream remote name in local checkouts: `upstream`

## Branches

| Branch | Purpose |
|---|---|
| `main` | Clean mirror of `neondatabase/neon/main`. No Nucleus-specific patches, ever. |
| `nucleus` | Permanent integration branch. All Nucleus-specific engine changes land here. |

There is one worktree for engine development:

```text
~/development/worktrees/neon/nucleus   → branch `nucleus`
```

The old live checkout (`worktrees/onoal/neon`, branch `onoal-branch-01`) was
removed in October 2026 together with its `.neon` runtime data. A SQL dump of
the acceptance table is kept outside the repositories
(`~/neon-live-backup-20261028/`).

## Synchronization procedure

### `main` (upstream mirror)

Only fast-forward from a clean checkout that is not used by any runtime:

```bash
git fetch upstream
git merge --ff-only upstream/main
git push origin main
```

If `origin/main` has diverged from `upstream/main`, report the conflict.
Never reset or force-push.

### `nucleus` (integration branch)

Bring upstream changes in with an ordinary merge:

```bash
git switch nucleus
git merge upstream/main
```

Resolve conflicts while preserving the Nucleus patches listed below, rebuild,
re-run the socket-isolation verification, then push `origin/nucleus`.

Synchronization is deliberate and reviewable. It is not part of routine
Nucleus development.

### Current base (October 2026)

- Upstream base: `fa504217c` (identical on `origin/main` and `upstream/main`)
- `origin/nucleus` fast-forwards from that base plus the patches below.

## Nucleus patch policy

- Patches on `nucleus` must be minimal, generic Neon capabilities — never
  Nucleus-product logic inside the engine.
- Every patch preserves default behavior when its new options are not used.
- No force pushes, no history rewrites of published commits.

### Patch 1 — Configurable `compute_ctl` HTTP bind addresses (P06.6.1)

Upstream `compute_tools` binds both HTTP listeners to the IPv6 wildcard
`[::]` with no way to restrict them.

- `compute_tools/src/http/server.rs`: `Server::Internal` and
  `Server::External` gained `listen_addr: Option<IpAddr>`. `None` keeps the
  historical `[::]` default.
- `compute_tools/src/compute.rs`: `ComputeNodeParams` gained
  `external_http_listen_addr` / `internal_http_listen_addr`, passed through
  to both servers.
- `compute_tools/src/bin/compute_ctl.rs`: new optional arguments
  `--external-http-listen-address` and `--internal-http-listen-address`
  (typed `IpAddr`). Unknown arguments still fail via clap, so an old binary
  can never silently start with wildcard listeners when new arguments are
  passed.

### Patch 2 — `neon_local endpoint start --compute-ctl-loopback-only` (P06.6.1)

- `control_plane/src/endpoint.rs`: `EndpointStartArgs` gained
  `compute_ctl_loopback_only: bool`. When set, `neon_local` starts
  `compute_ctl` with both HTTP listen addresses pinned to `127.0.0.1`.
- `control_plane/src/bin/neon_local.rs`: `endpoint start` gained the
  `--compute-ctl-loopback-only` flag, forwarded to the above.

## Build instructions

Postgres vendor sources are git submodules and are initialized on demand
(only the needed versions, to save disk):

```bash
git submodule update --init --depth 2 vendor/postgres-v17
make postgres-install-v17 -j16
make neon-pg-ext-v17 -j16
```

Engine binaries:

```bash
cargo build --bin neon_local --bin compute_ctl
cargo build -p storage_controller --bin storage_controller
```

`neon_local` resolves its component directory (`neon_distrib_dir`) from its
own executable location, and `pg_install` from the current directory
(`POSTGRES_DISTRIB_DIR` overrides). A runnable distribution therefore
consists of the built binaries plus `pg_install/` side by side.

Only PostgreSQL 17 is currently built in this worktree. If other versions
are needed later, initialize their submodules and run the same make targets
for `v16`/`v15`/`v14`.

## Runtime version compatibility

The secure-start capability is detectable at runtime: `compute_ctl --help`
lists `--external-http-listen-address` and
`--internal-http-listen-address` if and only if the patch is present.
Nucleus integration (later milestone) must check for these arguments before
relying on loopback-only startup, and refuse activation otherwise.

## Secure listener capability

With both addresses set (directly or via
`neon_local endpoint start --compute-ctl-loopback-only`), socket inspection
must show:

```text
127.0.0.1:<pg-port>          PostgreSQL
127.0.0.1:<external-port>    compute_ctl external HTTP, no 0.0.0.0 / [::]
127.0.0.1:<internal-port>    compute_ctl internal HTTP, no 0.0.0.0 / [::]
```

Without the new arguments the historical `[::]` wildcard behavior is
preserved for existing Neon consumers.

## Verification performed (P06.6.1 scratch environment)

Disposable environment under `~/scratch-nucleus-p0661` (separate ports,
separate state; no live data involved):

- `tenant` + `timeline` created through the storage controller API.
- `mappings map --branch-name main`, `endpoint create test-compute`
  (pg 17, ports 55441/17881/18881).
- `endpoint start test-compute --compute-ctl-loopback-only` succeeded.
- `ss -tln` showed all three listeners on `127.0.0.1` only.
- `SELECT 1` via `psql` against `127.0.0.1:55441` succeeded.
- `compute_ctl --nonexistent-flag` exits non-zero (old binaries fail
  instead of silently using wildcards).
- `endpoint stop` brings all listeners down cleanly.

Known scratch-environment lessons (not engine defects): the storage
controller auto-seeds a default node 1 on first start (delete + re-register
with the real addresses incl. grpc), and `pageserver.toml` /
`control_plane_api` URLs plus `broker_endpoint` must point at the scratch
ports, not the historical defaults.
