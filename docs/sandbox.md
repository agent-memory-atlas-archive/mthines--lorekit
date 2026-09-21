# Sandbox / container baseline — read before trusting a red gate

Six facts about a fresh sandbox/container, each confirmed by direct observation
rather than inferred. They cost time every time they are rediscovered. The
compressed summary lives in [`CLAUDE.md`](../CLAUDE.md) → "NX commands"; this is
the full runbook.

See also the hard rule that pairs with these: **never run whole-repo Nx
fan-outs in a cloud sandbox** (`CLAUDE.md` → "NX commands"). It saturates the box
and stalls the session; run `pnpm nx affected -t typecheck,test,lint` instead.

1. **Run `pnpm install` before the first `pnpm nx` command.** A fresh container's
   install is incomplete or absent. The signature is a cryptic
   `ERR_PNPM_RECURSIVE_EXEC_FIRST_FAIL Command "nx" not found` (no
   `node_modules/.bin/nx`) or a `Cannot find module 'zod'` cascade through
   `packages/schemas/**`. Both are the same missing install, not a broken nx
   config — `pnpm install --frozen-lockfile` at the root fixes both in ~15s.
   `pnpm` itself may or may not be on PATH; probe, and `corepack enable` only if
   it is missing.
2. **`cli:test` is red on a clean tree, so `run-many -t … --all` exits non-zero
   even with no changes.** The failures are all loopback-HTTP-shaped: the tests
   stand up a mock server on `127.0.0.1:0` and the spawned CLI child's `fetch`
   never arrives. The failing SET GROWS as new tests land on that surface, so
   **never pattern-match on a remembered count** — see point 3. (Writing a new
   test for this surface? Target the local on-disk store via `LOREKIT_HOME` +
   `LOREKIT_MODE=local` instead of a mock REST server.) Web lint separately
   reports dozens of pre-existing `no-non-null-assertion` **warnings, 0 errors** —
   not yours. The count drifts upward as tests land (it was recorded as ~47 and
   measured 59 later), so establish it with `git stash -u` like any other red;
   what CI gates on is the **0 errors**.
3. **Prove a failure pre-existing with `git stash -u`, not from memory.** Stash,
   re-run the same command, compare. The assertion that holds is "N failures
   before == N failures after"; any specific N goes stale. This takes ~40s and is
   the difference between reporting an inherited red and "fixing" something that
   was never broken.
4. **`supabase start` is impossible (no Docker socket) but the SQL tests still
   run.** `migrations.test.sql` — the only thing that exercises raw migration
   logic — needs a database, not the Supabase stack. PostgreSQL 16 is installed
   locally, so `initdb` a throwaway cluster, apply
   `supabase/tests/bare-postgres-bootstrap.sql` (it supplies the `auth.*` claim
   readers, `auth.users`/`auth.identities` and the three roles), apply
   `supabase/migrations/*.sql` in order, then run the test file. Its runbook is
   in its own header. Needs `apt-get install postgresql-16-pgvector` for
   00060/00062. **A failure is strong evidence; a pass is not a substitute for
   CI's `Integration smoke` job** — the bootstrap is a stand-in, not Supabase.
   The test file is one transaction ending in `rollback`, so it is re-runnable
   against the same database, which makes guard-biting an assertion cheap.
5. **`deno check` DOES run here — install it with `npm i -g deno`.** The
   official installer fetches from `deno.land`, which the egress policy blocks,
   and that made the ratchet look CI-only. npm is reachable, the `deno` package
   ships the same binary, and the version it lands (2.9.5) satisfies the `v2.x`
   CI pins and reproduces the committed baseline exactly. So
   `node scripts/ci/deno-check-functions.mjs` is a local gate, not a remote one —
   which is how the 83 baselined errors were driven to 0 rather than guessed at.
   Two traps if you script around it: `deno check` writes errors to stderr with
   ANSI colour, so a `grep -cE '^TS[0-9]+'` counts **zero** on real failures
   (strip ANSI first, and self-test the counter against a known-bad state); and
   pass `--node-modules-dir=none` so `npm:` specifiers resolve from Deno's cache
   the way production does, instead of the repo's pnpm `node_modules`.
6. **Node's built-in `fetch` ignores `HTTPS_PROXY` — run it with
   `NODE_USE_ENV_PROXY=1`.** This bites anything in this repo that exports
   telemetry over `fetch`: `scripts/migrations/sweep-rows.mjs`, the CLI's
   `packages/cli/src/telemetry/telemetry.mjs`, and the `_shared/telemetry/otel.ts` exporters when
   exercised locally. The symptom is a **`403 Host not in allowlist: <host>`**
   *even for a host that IS allowlisted*, because without the variable undici
   goes DIRECT and meets the network gateway's own, narrower allowlist instead
   of the session proxy's. The tell is a discrepancy between clients: `curl`
   reaches the host (it uses a `CONNECT` tunnel and so goes through the proxy)
   while `node -e 'fetch(...)'` returns that 403. Confirmed on Node 22.22 —
   `NODE_USE_ENV_PROXY=1 node …` turned the Dash0 sweep export from a 403 into
   a real 2xx, and `curl -sS "$HTTPS_PROXY/__agentproxy/status"` lists the
   recent denials with their reason. `/root/.ccr/README.md` documents this and
   the other per-tool proxy accommodations. Never "fix" it by unsetting
   `HTTPS_PROXY` or disabling TLS verification.
