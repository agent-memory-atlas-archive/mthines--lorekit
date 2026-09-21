# LoreKit — Agent Context

LoreKit is a Supabase-backed MCP server for shared, persistent agent memory.
Agents read and write *lore* (lessons) via MCP tool calls. A Next.js dashboard
lets humans browse, search, and manage those lessons.

→ For architecture, MCP tools, scope format, tokens, OTel, and deployment:
  **read [docs/](./docs/README.md) on demand — do NOT load all docs upfront.**

---

## Package map

| Package | Path | Role |
|---------|------|------|
| `@lorekit/core` | `packages/mcp-core/` | Scope validator, DB client, 10 tool handlers, OTel tracer/meter |
| `@lorekit/feature-flags` | `packages/feature-flags/` | OpenFeature-standard flag evaluation. `registry.ts` is the single hand-authored source of truth (zod-validated, all four OpenFeature value types); `nx run feature-flags:generate` projects it into typed TS bindings + a language-neutral `flags.manifest.json`. `LoreKitFlagProvider` resolves: session override → A/B(/n) experiment (deterministic FNV-1a bucketing on a stable `targetingKey`) → static default. Telemetry is two mechanisms: `featureFlagOtelHook` stamps `feature_flag.*` span attributes per evaluation (server-side), while `packages/web`'s `FeatureFlagsProvider` tags RUM signals with `feature_flag.<key>` (browser-side). UI-affecting experiments use a copy-and-suffix component convention (a resolver + one whole component per variant, never inline branches). `/settings/developer` (override UI) is open to any signed-in user outside production; in production it's gated by a server-side email allowlist (`developer-users.ts`, `notFound()` on the page). Package rules + file map: [`packages/feature-flags/CLAUDE.md`](./packages/feature-flags/CLAUDE.md). Full guide: [`docs/feature-flags.md`](./docs/feature-flags.md). Add/update/remove a flag via the `feature-flags` skill |
| `@lorekit/web` | `packages/web/` | Next.js 15 dashboard (Vercel) |
| `@lorekit/cli` | `packages/cli/` | Zero-dep Node CLI. **Setup:** `install`/`uninstall`/`doctor`/`update` (scaffold skills + MCP + lifecycle hooks into `.claude`; connectivity/token/scope health checks; offline skill refresh). **Reads:** `list`/`search`/`show`/`stats`/`scopes`/`diff`/`tree`/`lint`/`dedupe`/`link` (Offline + Remote split, `--json`/`--scope`). **Recurrence tooling:** `obligations` (a changed-file set vs a declarative Surface-Partner Map; per-entry `state` `advisory`/`gating`/`retired`) + `invariants candidates` (candidate scan reusing `dedupe`'s clustering; never auto-compiles or gates). **Maintenance:** `purge`/`purge-expired` (remote-only, account-wide, irreversible, confirm-or-`--yes`). Plus `hook` (shared hook engine behind the plugins), `mcp` (local stdio MCP server), `migrate`, `completion`. Self-contained OTLP telemetry (`service.name=cli`). Full command reference: [`docs/cli.md`](./docs/cli.md) |

| `plugins/` | `plugins/` | Per-framework deterministic bundles: `lorekit-claude` (marketplace plugin: skill + hooks + MCP), `lorekit-cursor` (rule + `stop` hook), `lorekit-codex` (feature-flagged hooks + `AGENTS.md` fallback, experimental). Root `.claude-plugin/marketplace.json` lists the Claude plugin. |
| `supabase` | `supabase/` | Edge Functions (production MCP server), migrations, NX targets |
| `@lorekit/smoke-tests` | `packages/smoke-tests/` | Live-endpoint integration/smoke tests against the deployed Edge Functions (memories, orgs, MCP, BYOD) — no application code, self-skips when its env vars are absent |

The **production MCP server** is `supabase/functions/mcp/index.ts` (Deno, self-contained). There
is no other MCP server implementation — a prior Node.js/Fly.io variant (`packages/mcp-server/`)
was never deployed and has been removed.

**Shared hook engine:** `lorekit hook --adapter <claude|cursor|codex> --event <name>` reads the host's
JSON on stdin and injects lessons / a retrospective nudge on stdout, always exiting 0. Logic lives once
in `packages/cli/src/{core,adapters}/`; each adapter reshapes I/O to its host. On a tool failure it
additionally does a best-effort lesson lookup (`failureQuery` distils terms from the tool name + error
text → `relevantLessonsFromStore` QUERIES the store — a SINGLE `store.search` carrying ALL the terms in
one call (OR semantics) across the scope hierarchy, so the offline store is walked once, not once per
term → the pure `dedupeRelevant` de-dupes the hits by `scope::key` and caps at 3, keeping the store's own
ordering) and injects any relevant prior lessons BEFORE the write-nudge — an unusable store, a throwing
search, or no match silently falls back to the nudge alone, and any error is swallowed (exit 0). This
deliberately QUERIES rather than post-filtering the SessionStart-injected set: post-filtering could only
ever resurface an already-shown lesson, so a paraphrased match or one past the per-scope read cap was
unreachable. Matching is the store's job — server-side FTS (with stemming) for
remote, full-scope substring for local — but ORDERING is not relevance: the remote handler orders by
`updated_at desc` (`supabase/functions/memories/handlers/search.ts`), and the local two-tier store puts
project-tier hits ahead of home-tier ones, so scope precedence holds only within a tier;
`store.search`'s `q` accepts a term LIST for exactly this
one-pass multi-term query (the remote joins it into one `websearch` `OR` query, a single round-trip). The cross-scope precedence merge (the SessionStart read) still
uses the SAME `resolvePrecedence` the read commands use, in the dependency-free
`packages/cli/src/shared/lessons-pure.mjs` (re-exported by `lessons-view.mjs`), so the hot path shares it without
dragging in the `util`/render stack. The end-of-turn retrospective nudge
is **friction-gated** by default (`hooks.stop`, resolved in `control.mjs`: `friction` | `always` | `off`,
default `friction`): in `friction` mode the Stop handler reads the session transcript via the pure
`packages/cli/src/core/friction.mjs` (`detectFriction` = errored tool results OR a tool+input repeated
≥ `STUCK_LOOP_THRESHOLD`; `readSessionFriction` is the IO wrapper; `shouldRetrospect` is the gating
matrix) and stays SILENT on a clean session — so a trivial turn no longer nudges. The friction read
happens BEFORE the once-per-session throttle is consumed, so a clean early turn doesn't burn the marker
and a later friction turn can still fire once. `friction: null` (no transcript, e.g. Cursor/Codex) falls
back to firing so no lesson is lost where friction can't be measured. The nudge itself is a terse
one-liner naming the detected reasons — the lore deep-link lives on the write CONFIRMATION, not here. The
Claude plugin's skill copy is vendored from `packages/cli/skill/` — keep in sync via
`node scripts/codegen/sync-plugin-skill.mjs` (a `--check` mode guards drift).

**Cross-framework validation:** `packages/cli/test/frameworks.test.mjs` replays payload fixtures
(`test/fixtures/<adapter>-<event>.json`) through the binary and asserts each host's output contract, runs
`claude plugin validate` (skips if the CLI is absent), and structurally checks the Cursor/Codex configs.
Harvest real fixtures with `LOREKIT_HOOK_RECORD=<dir>` set on the hook command (one run per framework).

---

## NX commands

### Never run whole-repo Nx fan-outs in a cloud sandbox

**Agents must NOT run `pnpm nx run-many -t … --all` (or `npx nx run-many --all`,
or any other whole-repo fan-out) in a cloud or container environment.** It
saturates the box — every project's target starts at once, the Nx daemon and the
spawned workers contend for the small CPU/memory allowance — and the session
freezes or stalls indefinitely rather than failing cleanly. Recovering costs a
whole session.

Run the narrow equivalent instead:

```bash
# What CI actually runs on a PR — only the projects your change affects
pnpm nx affected -t typecheck,test,lint

# Or name the projects explicitly, one target at a time
pnpm nx typecheck mcp-core
pnpm nx test cli
```

If you genuinely need whole-repo coverage, cap the fan-out and scope the targets
(`pnpm nx run-many -t typecheck --all --parallel=1`) and run one target per
invocation — never `typecheck,test,lint` together across every project. The
unqualified `--all` form below is documented as the CI gate; **CI is where it
belongs**, not a sandbox.

### Sandbox baseline — read before trusting a red gate

Six traps in a fresh sandbox, each costing a session when rediscovered. Full runbook
+ error signatures: [`docs/sandbox.md`](./docs/sandbox.md).

1. **Run `pnpm install --frozen-lockfile` before the first `pnpm nx`** — a fresh container's install is absent; the tell is `Command "nx" not found` or a `Cannot find module 'zod'` cascade.
2. **`cli:test` is red on a clean tree** (loopback-HTTP mock never arrives) and the failing set GROWS — never pattern-match a remembered count. Web lint has pre-existing `no-non-null-assertion` **warnings, 0 errors**.
3. **Prove a failure pre-existing with `git stash -u`, not from memory** — stash, re-run, compare; "N before == N after".
4. **`supabase start` is impossible (no Docker socket)** but `migrations.test.sql` runs against a throwaway PG16 cluster + `bare-postgres-bootstrap.sql`. A pass is NOT a substitute for CI's `Integration smoke`.
5. **`deno check` runs locally — install via `npm i -g deno`** (the official installer's host is egress-blocked); run `node scripts/ci/deno-check-functions.mjs` with `--node-modules-dir=none`.
6. **Node's `fetch` ignores `HTTPS_PROXY` — run with `NODE_USE_ENV_PROXY=1`**, else a `403 Host not in allowlist` even for allowlisted hosts. Never "fix" by unsetting `HTTPS_PROXY` or disabling TLS.

```bash
# CI gate — CI ONLY. Do not run this in a cloud/sandbox session; it stalls the
# box (see "Never run whole-repo Nx fan-outs in a cloud sandbox" above).
pnpm nx run-many -t typecheck,test,lint --all

# The sandbox-safe equivalent
pnpm nx affected -t typecheck,test,lint

# Individual packages
pnpm nx typecheck mcp-core
pnpm nx typecheck web
pnpm nx test mcp-core          # needs supabase start
pnpm nx serve web              # Next.js dev server

# Supabase (needs SUPABASE_PROJECT_REF in .env.local)
# NOTE: these are for local/first-time setup. Merging to main runs the
# staging-first CI/CD pipeline (.github/workflows/deploy.yml) automatically.
# See docs/deployment.md → "Automated deployment (CI/CD)".
pnpm nx deploy supabase        # typecheck + test → db push → fn:deploy
pnpm nx db:push supabase       # push migrations
pnpm nx fn:deploy supabase     # deploy mcp + health Edge Functions
pnpm nx db:types supabase      # generate TypeScript types from DB
pnpm nx health supabase        # curl /health endpoint
pnpm nx start supabase         # start local Supabase
pnpm nx fn:dev supabase        # run Edge Functions locally
```

### Web Storybook tests (interaction + visual regression)

The `@lorekit/web` dashboard has Storybook 10 (`@storybook/nextjs-vite`) wired to
Vitest **browser mode** via `@storybook/addon-vitest`. Two story files per
component, two suites, one browser run — driven by `packages/web/vitest.storybook.config.ts`
(kept **separate** from `vitest.config.ts` so the node/jsdom `nx test` target
never boots a browser):

- **Interaction tests** — `*.test.stories.tsx` (the `/Tests` namespace, `tags: ['test']`,
  `chromatic.disableSnapshot`); their `play` functions run as browser tests.
- **Visual regression** — every OTHER story (`*.stories.tsx` `Default`/`Playground`)
  is screenshotted by a Storybook-level `afterEach` in `.storybook/vitest.setup.ts`
  using Vitest 4's `toMatchScreenshot`. Baselines are committed under
  `src/**/__screenshots__/**/*-chromium-linux.png`.

```bash
cd packages/web
npx vitest run --config vitest.storybook.config.ts                 # both suites
npx vitest run --config vitest.storybook.config.ts --changed=main  # only changed stories
npx vitest run --config vitest.storybook.config.ts -u              # update baselines
```

- Invoke with **`npx`**, not `pnpm exec` / `pnpm run` / `nx run` — those wrap the
  process and keep the Playwright browser child's stdio open, so the run never
  returns. An `nx test-storybook` target exists for graph awareness but is not
  used by any CI gate for this reason.
- **Playwright is pinned to `1.56.0`** (Chromium build 1194) via a root pnpm
  override, so local runs and CI render on the same browser build and pixel
  baselines compare like-for-like. Bumping it requires regenerating baselines
  (`-u`) on Linux/Chromium.
- CI runs these in the `web-test` job of `ci.yml`, gated by the `changes.web`
  path filter and diff-optimized with `vitest --changed <base>` (a
  one-component edit re-tests one component). It is a browser job, so it is NOT
  part of the `check` job's `nx affected -t test`.

**MSW-mocked full-page stories.** Page/subtree stories mock the Supabase REST
(PostgREST) responses with [MSW](https://mswjs.io) so the app's real
`@tanstack/react-query` hooks resolve against a stable dataset (no SWR — React
Query is the one data layer). Pieces under `packages/web`: `public/mockServiceWorker.js`
(committed via `msw init`, served in the deployed build via `staticDirs: ['../public']`
so hosted stories mock too), `src/mocks/memories.ts` (`memoryHandlers()` + fixtures +
`FROZEN_NOW`), `src/mocks/decorators.tsx` (`withQueryClient` — retries off/no refetch;
`withFrozenClock` — pins `Date` so time-relative renders are deterministic;
`withMemorySidebar` — the `/lore` tree's context), and `.storybook/preview.tsx`
(`initialize()` + `mswLoader`, injects the public Supabase URL, and collapses `motion`
animations for stable snapshots — inert for the existing component stories). A story opts
in via `parameters.msw.handlers`. **Mixed rendering model:** `'use client'` pages story as
true full pages (`app/(dashboard)/lore/LorePage.stories.tsx` — needs
`parameters.nextjs.appDirectory: true` for `useRouter`/`useSearchParams`); server-component
pages can't render in the browser, so story their largest client subtree instead
(`components/dashboard/DashboardStats.stories.tsx` for the RSC `/overview`), never refactor
an RSC page to client just to story it. The `/lore` lesson list reads the `listMemories`
**server action** (gated on `getUser()` → empty in the mocked context), so its results panel
shows the empty state while the browser-fetched scope tree + heatmap populate.
- **Storybook deploys as its own Vercel project** via native Git integration (no CI job, no
  `VERCEL_TOKEN`). `packages/web/vercel.json` is the **dashboard** project (Next.js → `.next`),
  so the Storybook project uses Root Directory = repo root, Framework Preset = Other, Build
  Command `pnpm --filter @lorekit/web build-storybook`, Output Directory
  `packages/web/storybook-static`. Full runbook in [docs/storybook.md](./docs/storybook.md).

---

## User-facing docs (mandatory on every change)

**Any change that alters what a user can do, see, or configure MUST update the user-facing
docs AND `packages/web/public/llms.txt` in the SAME PR.** Docs are not a follow-up — a shipped
capability nobody can find is unshipped, and a documented capability that no longer behaves that
way is worse than no documentation.

This applies to a new or changed MCP tool / REST route / CLI command or flag, a new config key or
env var, a changed limit, token prefix, scope rule, or error contract, and any new dashboard
surface. It does NOT apply to a pure refactor, a test-only change, or an internal rename with no
observable effect.

| Surface | Path | Update when |
|---------|------|-------------|
| **`llms.txt`** | **GENERATED** — never edit `packages/web/public/llms.txt`. Edit `packages/schemas/src/llms/template.md` (editorial prose) or `packages/schemas/src/shared/tool-catalog.ts` (tool reference), then `pnpm nx generate:llms schemas`. | **Always.** The MCP tool reference, permission matrix and docs index derive from the catalog and the MDX frontmatter; the quickstart and scope explanation are editorial. `render.spec.ts` fails when the committed file is not what the generator produces. |
| Public docs | `packages/web/src/content/docs/*.mdx` | The change affects setup, config, offline/remote mode, orgs, labels, or a use case. Adding a page = drop the `.mdx` **and** add its `lib/docs/sections.ts` entry (`sections.spec.ts` fails on drift). |
| Dashboard copy | `packages/web/src/**` | The change alters an in-product flow the copy describes. |
| Contributor docs | `docs/*.md` + the index table in `docs/README.md` | The change affects architecture, deployment, limits, tokens, OTel, or a runbook. |
| `README.md` | repo root | The change alters the pitch, the install path, or the package map. |

Writing rules for all of the above: always the concrete MCP endpoint, never a `<ref>` placeholder
(see Endpoints and Key decisions); tag every fenced code block with its language; keep `llms.txt`
consistent with the MDX docs — when the two disagree, agents read `llms.txt` and get it wrong.

Definition of done: the diff either touches the surfaces above, or the PR description says in one
line why none applied.

---

## PR workflow (mandatory — always follow this order)

Every PR in this repository goes through a fixed five-step sequence.
Do NOT skip steps or change the order, whether the PR is a draft or ready for review.

Before Step 1, settle the docs: apply
[User-facing docs](#user-facing-docs-mandatory-on-every-change) and commit those edits with the
change they document, so `/polish` and `review-loop`'s `pr-reviewer` pass see the finished diff.

### Prerequisites — install agent-skills (once per sandbox)

Before running any PR workflow steps, ensure `agent-skills` is cloned and all skills and agents are
wired into `~/.claude/`. This is idempotent — safe to re-run, no-op if already set up.

```bash
# Clone if not already present, then wire every skill and agent into ~/.claude/
git clone https://github.com/mthines/agent-skills.git /tmp/workspace/agent-skills 2>/dev/null || true
bash /tmp/workspace/agent-skills/scripts/sync-symlinks.sh
```

The script discovers all skills (any directory under `skills/` containing a `SKILL.md`) and all
agents (`agents/*.md`), and creates a two-tier symlink chain so they are available as native Claude
skills and sub-agents. It repairs broken links and skips already-correct ones.

### Step 1 — Run `/polish` and auto-fix all findings

Before opening the PR, run the `polish` skill against the branch. This is a **local-only** pass —
it never writes to GitHub. All auto-fixable findings must be committed before opening the PR.

Dispatch a sub-agent (subagent_type: general):

> Read /tmp/workspace/agent-skills/skills/quality/polish/SKILL.md and follow it exactly.
> Run in full mode (default). Auto-fix all simple findings. Apply all Class M mechanical refactors
> that pass the confidence gate. Commit each pass separately. Do NOT write to GitHub.

Wait for the polish run to finish before proceeding. Planned-complex items (Class J, judgment-required)
are surfaced for awareness but do not block the workflow — they require a human decision.

Skip this step only if the branch diff is non-code only (docs, lockfiles, generated artefacts).

### Step 2 — Open the PR with `/create-pr` (as a draft)

Open the PR with `/create-pr` (it opens as a **draft** — the draft state is what lets `review-loop`'s
`pr-reviewer` pass post inline comments while the branch converges).

**Do not pass `--no-review`.** This repo has no external review bot, so the review pass is
`/create-pr`'s own built-in `review-loop` — dropping it would leave the PR unreviewed. By default
(no quality flags) `/create-pr`'s Step 6.5 runs `Skill("review-loop", "<pr-url> --no-ci")` for you
immediately after opening the PR (Step 3 below covers what that does). Do **not** pass `--no-feedback`
(that skips Step 4) or `--no-quality` (that skips Step 3 and Step 4 both). Undrafting is the last,
human step, after the flow reaches ready-to-review.

### Step 3 — `review-loop` converges the PR (the ONE review agent on the PR)

`review-loop` is the sole reviewer here — there is no external bot to defer to. `/create-pr`'s Step 6.5
invokes it automatically as `Skill("review-loop", "<pr-url> --no-ci")`: up to 5 iterations of
`pr-reviewer` (dispatched fresh each round, read-only, posts one `COMMENT` review, and **re-reviews on
every new push**, marking its own addressed findings resolved) → `implement-suggestion --resolve-all`
(applies actionable findings, and replies-to-and-resolves the non-fix threads it can honestly close) →
`polish simplify` (Class M mechanical refactors) — converging until **every review thread is resolved**,
either by a fix or an honest reply. The only threads left open at exit are genuine human-judgment flags
the loop will neither auto-apply nor honestly decline. On convergence it also refreshes the PR
description to match the shipped diff — do not re-edit the body yourself afterward.

You do NOT need to trigger this manually if you opened the PR with `/create-pr` (Step 2) — it is
already running. If you push a follow-up commit by hand afterward, re-run it yourself:

```
Skill("review-loop", "<pr-url> --no-ci")
```

(`--no-ci` because Step 5 below owns driving CI green; `review-loop` would otherwise also try.)

### Step 4 — Absorb genuine external feedback with `/implement-suggestion --watch`

This step exists for **real** external parties — CodeRabbit, a human reviewer — not a self-review
duplicate. `/create-pr` dispatches it automatically once `review-loop` converges (its Step 6.7,
external-bot feedback step) as a **background** sub-agent — so if you opened the PR with `/create-pr`
(Step 2) it is already running. It is scoped to comments posted **after** `review-loop`'s last push, so
it never re-applies `review-loop`'s own findings. If you need to drive it yourself, dispatch a
background sub-agent (`run_in_background: true`, subagent_type: general):

> Invoke: Skill('implement-suggestion', '<pr-url> --watch')
> Absorb any CodeRabbit / human review feedback to completion. It never opens a new PR and never
> undrafts this one. Return its per-iteration watch report.

`--watch` waits for each external review, applies the actionable comments (**one commit per comment**,
each gated by `/critical` then `/confidence`), pushes, and repeats — **bounded to 5 iterations**
(`--max-iters` default; hard cap 10), processing only comments newer than the last round so it never
re-applies one. It **never undrafts**.

### Step 5 — Drive CI green; ready-to-review = `review-loop` converged + green CI

`/create-pr` watches CI and delegates mechanical failures to `/ci-auto-fix` for you. For any red check
on a hand-pushed commit, run it yourself:

```
/ci-auto-fix
```

This uses the `ci-auto-fix` skill (wired in during Prerequisites), diagnoses any failing GitHub
Actions checks, applies a minimal targeted fix, and iterates until all checks are green. The skill
is confidence-gated (>=90 auto-apply, 80-89 ask, <80 escalate) and will never disable or weaken a
check. Skip only when CI is already fully green. A `/ci-auto-fix` push is itself a new commit, so if
you're driving this by hand, re-run Step 3's `Skill("review-loop", "<pr-url> --no-ci")` afterward to
re-verify against the new head — `/create-pr`'s own run of Steps 6.5/6.7/7–9 already sequences this for
you end to end.

**Definition of ready-to-review:** `review-loop`'s final `pr-reviewer` verdict is PASS with zero open
threads (only genuine human-judgment flags may remain, and those must be surfaced to the user) **and**
every CI check is green. That is the *content* state this flow drives to; the agent does **not** flip
the draft flag — undrafting stays a human/explicit decision.

### Summary table

| Step | Action | Who triggers |
|------|--------|--------------|
| 0 | Clone agent-skills + run sync-symlinks.sh (once per sandbox) | Agent |
| 0.5 | Update user-facing docs + regenerate `llms.txt` (or state why none applied) | Agent |
| 1 | Run `/polish` — review + simplify, auto-fix all findings, commit each pass | Agent |
| 2 | `/create-pr` — open draft PR; no `--no-review` (there is no external bot to defer to) | Agent |
| 3 | `review-loop` converges automatically (`pr-reviewer` → `implement-suggestion --resolve-all` → `polish simplify`, ≤5 iters) — the ONE review agent, refreshes the PR description on convergence | Agent (auto-dispatched by `/create-pr`) |
| 4 | `/implement-suggestion --watch` (background) — absorb genuine CodeRabbit/human feedback posted after `review-loop`'s last push, one commit per comment, ≤**5** iters, never undrafts | Agent (background) |
| 5 | `/ci-auto-fix` until green. Ready-to-review = `review-loop` PASS with zero open threads AND green CI (agent does not undraft) | Agent |

## Scope format (canonical — `::` separator only)

```
global
project::{name}                           project::agent-skills
repo::{owner}/{repo}                      repo::mthines/gw-tools
branch::{owner}/{repo}::{branch}          branch::mthines/gw-tools::feat/x
```

Single `:` → 400 error. All segments lowercased. See [docs/scope-format.md](./docs/scope-format.md).

---

## Auth tiers (MCP server)

1. `SUPABASE_SERVICE_ROLE_KEY` → full access, bypasses RLS (CI only)
2. `lk_rw_*` / `lk_ro_*` / `lk_wo_*` API token → service-role client + **mandatory `user_id` filter** on every query
3. Supabase JWT → user-scoped client, RLS enforced automatically

**Critical:** `api_key` auth uses service-role. ALL queries must `.eq('user_id', userId)`.
Write tools require write permission (`lk_rw_*` / `lk_wo_*`); read tools require read
permission (`lk_rw_*` / `lk_ro_*`). `lk_ro_*` is denied on write tools; `lk_wo_*` is denied
on read tools — both with the standard `-32001` permission-denied error. The gating logic
(`READ_TOOLS`/`WRITE_TOOLS`/`toolRequires`/`tokenPrefixFor`) is a shared pure module,
`packages/mcp-core/src/auth/permissions.ts`, mirrored self-contained into
`supabase/functions/mcp/permissions.ts` (the `limits.ts` pattern).

**Entitlement must gate the response BODY, not only the write.** On any
`supabase/functions/mcp/**` (or REST) handler authenticated by a user JWT but keyed on a
**caller-supplied resource id**, the same `entitled` / `verdict.kind === 'linked'` computation
that gates the RPC/upsert MUST also gate every field of the success response. The recurring
bug: the check guards the DB write while the 200 body still returns third-party/unentitled
metadata sourced from the fetched object — an information leak even though nothing was written.
The tell when reviewing: an `entitled` value that gates the mutation but is never referenced
when constructing the response literal; grep the response literal for fields sourced from the
fetched third-party object. (Related, still-open residue not covered by this rule: a
pending-vs-linked `status` field / not-found-vs-ok split can remain an existence oracle —
`status` can't simply be dropped because `packages/web/src/lib/github-installations.ts`
branches on it.)

---

## Limits & rate limiting

Two abuse guardrails, both free-tier defaults, config-driven, per-user
overridable (no billing built yet — see [docs/limits.md](./docs/limits.md)):

- **Memory cap** (default 5000 active memories/user, raised from 1000 by migration 00032_plans.sql) — enforced authoritatively
  by a `BEFORE INSERT` trigger on `memories` (`enforce_memory_cap()`,
  `supabase/migrations/00004_limits.sql`). Rejections are translated into an
  actionable `LimitError` (code `memory_cap`) by the app layer.
- **Rate limit** (default 120 req/min/user, all MCP methods) — a Postgres-backed
  fixed-window RPC (`lorekit_check_rate_limit()`), called by the transport layer
  right after auth resolves. Blocked requests get HTTP `429` + `Retry-After`.
- Both read their limits through `lorekit_get_limit(user_id, key)` =
  `COALESCE(user_limits override, lorekit_default_limit(key))` — no numeric
  limit is hardcoded in app code. Raising a user's limit is a `user_limits` row
  upsert (SQL) for now.
- Service-role (CI, `user_id IS NULL`) is exempt from both guardrails.

---

## Key files

The full annotated index (172 files, grouped by subsystem) lives in
[`docs/key-files.md`](./docs/key-files.md) — read it when you need to locate a
specific handler, migration, or pure module. The load-bearing "start here" files:

| File | Purpose |
|------|---------|
| `packages/schemas/src/shared/tool-catalog.ts` | The single origin of the operation SURFACE — every tool's schema, `permission`, `auth`, and its `surfaces` binding (which of MCP/CLI/REST, under what name, backed by which handler, with a declared reason for each absence). Zero-import by construction; `gen-surfaces.mjs` projects the two consumers that cannot import it |
| `packages/cli/src/commands.mjs` | The ONE CLI command registry — `bin/lorekit.mjs` derives dispatch, aliases, flag strictness, help and `traceCommand` wrapping from it |
| `packages/mcp-core/src/scope/scope.ts` | Canonical scope validation + wildcard expansion |
| `packages/mcp-core/src/scope/scope-precedence.ts` | Which row wins when a read named NO scope — `readOrder` as a total order (mirrored to edge; cross-language twin `packages/cli/src/shared/scope-precedence.mjs`) |
| `packages/mcp-core/src/auth/permissions.ts` | `READ_TOOLS`/`WRITE_TOOLS`, `toolRequires`, `tokenPrefixFor` — the `lk_rw_`/`lk_ro_`/`lk_wo_` prefix derivation + tool gating (mirrored to edge + web) |
| `packages/mcp-core/src/limits/limits.ts` | `LimitError`, `translateCapError`, `checkRateLimit` — the origin of the "pure module mirrored self-contained into the edge function" pattern |
| `packages/mcp-core/src/auth/tenant-scope.ts` | `applyTenantScope` — the single widened tenant-visibility predicate (RLS side is `lorekit_member_org_ids()`) |
| `supabase/functions/mcp/index.ts` | Self-contained Deno MCP server (production) |
| `supabase/functions/_shared/telemetry/otel.ts` | Reusable OTel for Edge Functions: `traceRequest()`, `createTracedClient()`, and the ONE source of the OTLP resource attributes / endpoint / attribute encoding that both the span and metric exporters share |
| `packages/mcp-core/src/telemetry/io-ledger.ts` | `mergeBusyMs`/`attributeIoTime` — the self-time split behind `lorekit.self_time_ms` (mirrored to `_shared/`). Merged intervals, never summed |
| `supabase/functions/_shared/audit/audit.ts` (← `packages/mcp-core/src/audit/audit.ts`) | THE single edge audit writer (MCP tools **and** REST handlers) |
| `supabase/functions/_shared/telemetry/usage.ts` | `recordUsageEvent` + `getUserPlanName` — the single edge usage-event writer |
| `supabase/migrations/00001_memories.sql` | `memories` table, FTS, RLS |
| `supabase/migrations/00004_limits.sql` | Memory-cap trigger (`enforce_memory_cap`) + rate-limit RPC (`lorekit_check_rate_limit`) + `user_limits`/`lorekit_get_limit` config source |
| `packages/web/src/lib/api/` | The dashboard's client for LoreKit's OWN REST API (`restFetch`, typed wrappers from `@lorekit/schemas`) |
| `packages/web/src/lib/filters.ts` | Pure model for the Lore Explorer filter bar (OR within a dimension, AND across; `filtersToBody` is the wire seam the Explorer uses, `filtersToQueryParams` the GET encoding kept for query-string callers) |
| `packages/web/src/lib/dash0-rum.ts` | The SINGLE browser RUM init path for `@dash0/sdk-web` (init guard, endpoint validator, identity) |

See [`docs/key-files.md`](./docs/key-files.md) for the remaining ~137 files:
all migrations, the `_shared`/`mcp-core` pure modules and their edge mirrors,
the auth/org/invite/scope-binding surfaces, and the Explorer/Settings UI.

---

## OTel attributes (custom)

All `lorekit.*` spans carry:
- `lorekit.tool.name` — bounded: `memory.write|read|list|delete|search`
- `lorekit.scope` — canonical scope string
- `lorekit.scope.type` — bounded: `global|project|repo|branch|mixed|invalid`, and OMITTED when the operation carries no scope. Resolved by the shared `scope-type-attribute.ts` (mirrored into `_shared/`), never by an inline `split('::')` in a transport
- `lorekit.key` — lesson key
- `service.namespace` — always `lorekit`
- `deployment.environment.name` — `production|preview|development|local` (on `web`, from `VERCEL_ENV` **cross-checked against `NODE_ENV`**, never `VERCEL_ENV` alone — see Key decisions), plus the synthetic `test` stamped on smoke/CI runs (the pipelines set `DEPLOYMENT_ENVIRONMENT=test`; the edge also honours it per-request via the `X-LoreKit-Deployment-Environment` header, allowlisted to `test`) — see [docs/otel.md](./docs/otel.md) → "Smoke / test runs are tagged"

Metric: `lorekit.tool.duration` histogram (unit `s`) with `lorekit.tool.name` + `lorekit.scope.type`.

Every edge ROOT request span additionally carries the self-time split, stamped by
`traceRequest` and fed by span KIND (any `SPAN_KIND_CLIENT` span counts as an outbound call):
- `lorekit.io.wait_ms` — wall-clock ms with ≥1 outbound call in flight. Concurrent calls count ONCE
- `lorekit.io.calls` — how many outbound calls (summed, not merged — an N+1 vs one slow query)
- `lorekit.self_time_ms` — the residue no child span explains: our own code

Numeric measures, not dimensions, so they add no cardinality. The merge lives in the pure
`io-ledger.ts` (mirrored to `_shared/`) — never simplify it back to a SUM, which double-counts
concurrent queries and drives self time negative.

**Profiles are NOT a signal LoreKit can emit** — Dash0 collects them with a host-level eBPF agent and
every runtime here is managed serverless. Query-level profiling (`pg_stat_statements` → the three
`lorekit.db.query.*` cumulative sums, via the service-role-only `profiling` function, OFF until two
Vault secrets exist) is the substitute. Read
[docs/otel.md](./docs/otel.md) → "Query-level profiling" and
[docs/decisions.md](./docs/decisions.md#profiling-is-sql-level-because-there-is-no-host-to-profile)
before proposing a profiler.

Trace-context propagation (W3C `traceparent` — who sends/receives, the origin allow-list, the parser, span kinds, and the recorded-not-acted-on sampled flag) and the `service.name` inventory (edge = one `api` service told apart by `faas.name`; `mcp`/`web`/`cli`; never a per-function `SERVICE_NAME` secret) live in [`docs/otel.md`](./docs/otel.md) → "Custom span attributes — propagation & service.name".

---

## Endpoints

The production Supabase project ref is **`pqokxlhvnosogizsjztg`** (static). Always
write the concrete endpoint below in any user-facing surface — dashboard copy,
Learn pages, config examples, docs — **NEVER** a `<ref>` / `<project-ref>`
placeholder for the MCP server URL.

| URL | Auth | Purpose |
|-----|------|---------|
| `https://pqokxlhvnosogizsjztg.supabase.co/functions/v1/mcp` | Bearer token required | MCP server for agents |
| `https://pqokxlhvnosogizsjztg.supabase.co/functions/v1/health` | None (public) | Uptime monitoring |
| `https://lorekit.io` | GitHub OAuth, email + password, or magic link | Web dashboard |

---

## Key decisions (do not relitigate)

Each decision's full rationale lives in [`docs/decisions.md`](./docs/decisions.md) —
the headline here is the rule; follow the link for the "why". Short entries carry
their rationale inline. **Do not relitigate these.**

- **Dashboard is a CLIENT of LoreKit's REST API** — memory reads/writes go through the `memories` edge function (user JWT), never a direct PostgREST/supabase-js query; every new data surface becomes part of the PUBLIC contract (schema + handler + OpenAPI + `migrations.test.sql`). [rationale](./docs/decisions.md#dashboard-is-a-client-of-lorekits-rest-api)
- **MCP server endpoint is a static production URL** — always write `https://pqokxlhvnosogizsjztg.supabase.co/functions/v1/mcp` in user-facing content, never a `<ref>` / `<project-ref>` placeholder. [rationale](./docs/decisions.md#mcp-server-endpoint-is-a-static-production-url)
- **Lore Explorer filters through ONE two-level command menu + pills** — never one picker per dimension, never client-side narrowing; OR within a dimension, AND across; all dimensions filtered server-side; facets are their own drill-down query. 00110: under `tags_mode='all'` a candidate label's count is WITHIN-GROUP CO-OCCURRENCE (rows satisfying every other filter + all selected tags + the candidate); `tags_mode='any'` and scalar dimensions stay self-exclusion. Facet enumeration is STABLE (cross-join-then-filter, so a zero-match value reports `count: 0` and stays listed). [rationale](./docs/decisions.md#lore-explorer-filters-through-one-two-level-command-menu)
- **The Explorer's Activity panel has a DISPLAY default (24h), separate from the list's (all time)** — substituted for an absent `?range=`, never written back; `RangePicker` emits `{preset:'all'}` (not `null`) so "chose All" and "has not chosen" stay two values. [rationale](./docs/decisions.md#the-explorers-activity-panel-has-a-display-default-separate-from-the-lists)
- **The Explorer's Activity panel shows ONE body at a time and remembers your disclosure** — a `SegmentedControl` (Stat charts / Heatmap); the expanded heatmap view is the calendar ALONE (no stat grid), collapsed keeps the four numbers. Opens EXPANDED; disclosure + view persist to `localStorage` via `useSyncExternalStore` (`null` = "not yet consulted", render COLLAPSED while unresolved). Never move these to the URL; never re-seed from a `useState` initializer. [rationale](./docs/decisions.md#the-activity-panel-shows-one-body-at-a-time-and-remembers-your-disclosure)
- **Chart bucket readouts are PORTALED, one per chart** — `AnchoredTooltip` reuses `Tooltip`'s pure `computeTooltipPosition` and takes an `anchor: Element`, because an in-flow panel is clipped by `CollapsibleStatCard`'s `overflow:hidden` reveal region and one `Tooltip` per bucket would be 364 portals. The heatmap's native `title` is gone; its `aria-label` is not. [rationale](./docs/decisions.md#chart-bucket-readouts-are-portaled-and-there-is-one-per-chart)
- **Dashboard figures COUNT to a new value** (`AnimatedNumber`) — a change indicator, not decoration; two nodes (visible + `sr-only`), so read the `.sr-only` half, never `textContent`; honours `MotionConfig reducedMotion="always"` on top of the device preference, which is what makes the visual baselines deterministic. The `TrendChip` delta uses the same two-node pattern, but ONLY when it abbreviates a large percentage (`+8.8K%`). [rationale](./docs/decisions.md#dashboard-figures-count-to-a-new-value)
- **Mobile transient selection surfaces use the `BottomSheet` primitive** — never an anchored popover on the phone breakpoint; share ONE body between desktop popover and sheet (`FilterMenu` is the reference). [rationale](./docs/decisions.md#mobile-transient-selection-surfaces-use-the-bottomsheet-primitive)
- `::` separator avoids collision with `/` in repo paths and `:` in branch names
- `lk_rw_` prefix encodes permission visibly in config files
- **Write-only tokens (`lk_wo_*`)** store `permissions: ['write']` in the existing `text[]` column (zero migration); gating logic in the shared pure `permissions.ts`. [rationale](./docs/decisions.md#write-only-tokens-lk_wo_)
- Token SHA-256 hash in DB — shown once, never stored in plain text
- **API token scoping (scopes + orgs)** — `api_tokens.scopes` (empty = unrestricted, owner wildcards reuse `expandScopeForSearch`'s grammar, with the trailing `*` legal only after a `/` or a `::`) + a tri-state `org_access`/`org_ids`; the key restriction is authoritative over `org_scope_bindings` auto-routing; scoping is set through an owner-only SECURITY DEFINER RPC, never an UPDATE policy. **Live end to end — 00068 ships the columns and the two predicates, 00069 makes them binding in three layers (transport refusal, query narrowing, and the SQL functions the transports cannot stand in front of), `TokenManager.tsx` sets them, and 00070 audits every change.** [rationale](./docs/decisions.md#api-token-scoping-scopes--orgs)
- `AlwaysOn` OTel sampler — sampling deferred to Dash0 pipeline, never SDK-side
- `instrumentation.ts` must be `async function register()` with `NEXT_RUNTIME === 'nodejs'` guard
- **Browser RUM initialises in `lib/dash0-rum.ts`, identity set at INIT** — every event carries a `user.id` (`anon:<uuid>` until login); never simplify back to a login-only `identify()`. [rationale](./docs/decisions.md#browser-rum-init--identity-at-init)
- **`OTEL_SERVICE_NAME` must never decide a component's name** — `register()` overwrites it with the code-declared name and warns on conflict. [rationale](./docs/decisions.md#otel_service_name-must-never-decide-a-components-name)
- **`VERCEL_ENV` must never decide `deployment.environment.name` alone** — always cross-checked against `NODE_ENV` in one shared pure module (`otel-deployment-env.ts`), so a dev server can never report `production`/`preview`; `VERCEL` is deliberately not also gated on. [rationale](./docs/decisions.md#vercel_env-must-never-decide-the-deployment-environment-alone)
- **Caller identity belongs on the ROOT request span** — `createRouter` sets `auth.type`/`auth.user_id` on the REST root span (as MCP does); enables web↔CLI↔MCP correlation by account, no fingerprinting. [rationale](./docs/decisions.md#caller-identity-belongs-on-the-root-request-span)
- **CLI telemetry is attributable via a minted install id + a LEARNED account id** — `service.instance.id` (opaque random, persisted to `$LOREKIT_HOME/telemetry-id.json`) plus `user.id` (the account once known, else `install:<id>`, learned from the `X-LoreKit-User-Id` response header and cached, which is what lets an OFFLINE run join to server-side `auth.user_id`). Four invariants are load-bearing: nothing minted and no file created while export is disabled; the account cache only UPDATES an existing file, never creates one; the id is random so deleting the file resets it; an unpersistable id reports as NO identity, never a fresh one per run. Never add an in-memory fallback. [rationale](./docs/decisions.md#cli-telemetry-is-attributable-by-a-locally-minted-install-id-plus-a-learned-account-id)
- **`hook` and `mcp` are untraced but METERED** — `meterCommand` emits the counter alone (no span) on a 400ms budget, started before the command and awaited after so it overlaps its work; `surface-parity.test.mjs` asserts both that they ARE metered and that they never reach `traceCommand`. [rationale](./docs/decisions.md#hook-and-mcp-are-untraced-but-metered)
- **The edge's `deployment.environment.name` is set by `deploy.yml`, not inferred** — a Supabase project has no `VERCEL_ENV`, so with the secret unset BOTH projects reported `local` and preview/production traffic was indistinguishable; each deploy job now `supabase secrets set DEPLOYMENT_ENVIRONMENT` (`preview`/`production`) so it self-heals. `preview`, never `staging`. The per-request `test` header still wins for smoke runs. [rationale](./docs/decisions.md#the-edges-deploymentenvironmentname-is-set-by-the-deploy-pipeline-not-inferred)
- **Edge Function is self-contained Deno** — no cross-package/bare imports; schemas mirrored into `_shared/schemas/`, `npm:` specifiers only; never re-add an import map. [rationale](./docs/decisions.md#edge-function-is-self-contained-deno-no-import-map)
- NX 22.4.0 — matches `gw-tools` exactly; bump both together
- **Memory cap enforced by a DB trigger** (`NEW.user_id`-keyed, auth-agnostic, unbypassable) — not app-side counting. [rationale](./docs/decisions.md#memory-cap-enforced-by-a-db-trigger)
- Rate limiting is a Postgres-backed fixed-window counter (not in-memory/Redis) — edge isolates are stateless; no new infra
- Limits config lives in one DB function (`lorekit_default_limit`) + `user_limits` override table — no numeric limit hardcoded; raising a ceiling is one row upsert
- **Webhook secrets are repo-scoped** — matched by `repository.full_name` against `webhook_secrets.repo`; `selectWebhookSecrets` pure + mirrored. [rationale](./docs/decisions.md#webhook-secrets-are-repo-scoped)
- **Audit logging is captured at the app layer** — explicit `recordAudit` after each mutation; actor via `auditUserId`; ONE edge writer (`_shared/audit/audit.ts`); one action vocabulary in `@lorekit/schemas`. [rationale](./docs/decisions.md#audit-logging-is-captured-at-the-app-layer)
- **Usage events recorded once per surface, in the dispatcher** — never per handler; a REST route reports the equivalent MCP tool name via `rest-tool-name.ts`. [rationale](./docs/decisions.md#usage-events-recorded-once-per-surface-in-the-dispatcher)
- **Org/scope sharing is ORG-FIRST (Phase 1)** — single authoritative shared row; tenant visibility in ONE place (`lorekit_member_org_ids` / `applyTenantScope`). [rationale](./docs/decisions.md#orgscope-sharing-is-org-first-phase-1)
- **Org-sharing Phase 2 (org-owned writes)** — `memory_write` gains `p_org_slug`, ownership authorization-derived inside the RPC; cap becomes tenant-keyed; `LK002` denial. [rationale](./docs/decisions.md#orgscope-sharing-phase-2-org-owned-writes)
- **Audit Logs pagination is keyset (cursor), not OFFSET** — opaque `nextCursor`; own `user_id` filter so a forged cursor can't widen visibility. [rationale](./docs/decisions.md#audit-logs-pagination-is-keyset-cursor-not-offset)
- **Org-sharing Phase 3 (org management backend)** — every state transition is a SECURITY DEFINER RPC; no insert/update/delete RLS; anti-TOCTOU invite accept. [rationale](./docs/decisions.md#orgscope-sharing-phase-3-org-management-backend)
- **Org-sharing Phase 4 (dashboard UX)** — Settings→Organization page, pure `org-ui.ts` affordances, `ConfirmDialog`/`ToastProvider`; `lorekit_org_members_list` for real identities. [rationale](./docs/decisions.md#orgscope-sharing-phase-4-dashboard-ux)
- **Safe org deletion** — soft-delete (`deleted_at`) + owner-only `lorekit_org_purge`; hidden from reads via `lorekit_member_org_ids`. [rationale](./docs/decisions.md#safe-org-deletion)
- **Scope→org binding** — admin binds a scope; a write auto-routes to the org for write-capable members, falls back to personal (never rejected) otherwise. [rationale](./docs/decisions.md#scopeorg-binding) **A binding can also be a WILDCARD prefix** (`repo::owner/*`, `branch::owner/repo::*`, reusing the API-token scope grammar verbatim) — when several bindings match, resolution is most-specific-wins (exact beats any wildcard; longer wildcard prefix beats shorter), gated in SQL by `lorekit_api_token_scopes_valid` on `lorekit_scope_bind`. [rationale](./docs/decisions.md#wildcard-scopeorg-bindings-and-most-specific-wins-precedence)
- **GitHub App single-secret model** — all App events HMAC-verified against ONE `GITHUB_APP_WEBHOOK_SECRET`; dashboard visibility via `installations/sync`, not the webhook. [rationale](./docs/decisions.md#github-app-single-secret-model)
- **Comment-relevance classification is server-side, config-driven, and refuses to guess** — the App's webhook path classifies review-thread outcomes (`pull_request_review_thread.resolved` + the merged-PR sweep), not a per-repo GitHub Actions workflow; a directional record requires corroborated evidence and every undecidable branch writes NOTHING (an incomplete commit walk is never "untouched"); the consumer's vocabulary lives in `github_relevance_configs` (literal marker delimiters, never a regex from the DB) and records are written as the installation's OWNER, never `user_id = null`. [rationale](./docs/decisions.md#comment-relevance-classification-is-server-side-config-driven-and-refuses-to-guess)
- **Hook scope ordering unified, project scope IS injected** — `readOrder` = `[project, branch, repo, global]`, matching the read commands' `scopeList`. [rationale](./docs/decisions.md#hook-scope-ordering-unified-project-scope-injected)
- **Hook precedence + match is single source of truth with read commands** — `resolvePrecedence`/`matchesQuery` in dependency-free `lessons-pure.mjs`. [rationale](./docs/decisions.md#hook-precedence--match-is-single-source-of-truth-with-read-commands)
- **CI/CD is split** — `ci.yml` verifies before merge, `deploy.yml` promotes the verified commit (preview→prod); don't re-merge or re-add a deploy-time test job. [rationale](./docs/decisions.md#cicd-is-split-ciyml-verifies-deployyml-promotes)
- **The deploy SCOPE is measured against what is deployed** — each half is diffed against the SHA it last reached production at (`deployed/api-production`/`deployed/web-production`), never the previous commit; rollbacks repoint the tag, tags fail open, and the decision is the unit-tested `scripts/ci/resolve-deploy-scope.mjs` (called by `deploy.yml`'s `changes` job). Never reinstate the single-push baseline: it let web promote ahead of an API that had never deployed. [rationale](./docs/decisions.md#cicd-is-split-ciyml-verifies-deployyml-promotes)
- **Smoke tests clean up after themselves + a sweeper** — hard-delete/purge; name-pattern sweep behind four guards; never revert to soft delete / id tracking / a permissive pattern. [rationale](./docs/decisions.md#smoke-tests-clean-up-after-themselves)
- **Invite-details modal** — SECURITY DEFINER `lorekit_invite_org_details` gated on `lorekit_invite_addressed_to_caller`; Tier-A fields only, never leaks existence. [rationale](./docs/decisions.md#invite-details-modal)
- **Docs are a PUBLIC MDX section at `/docs`** — single source `DOCS_SECTIONS`; full-text search derived from the same MDX files. [rationale](./docs/decisions.md#docs-are-a-public-mdx-section-at-docs)
- **Settings sections named for the user's goal** — `/settings/integrations`; sub-nav only when >1 card; manual webhook UI removed (ingest path untouched). [rationale](./docs/decisions.md#settings-sections-named-for-the-users-goal)
- **Org REST routes open to `lk_*` tokens, gated by token permission not auth tier** — actor via `p_actor_user_id` (service-role only) + explicit tenant reads; CLI dropped MCP entirely. [rationale](./docs/decisions.md#org-rest-routes-open-to-lk_-tokens-gated-by-token-permission)
- **Org-owned lore archive/hard-delete over REST** — `DELETE /memories?…&org=` routes to the role-gated `memory_delete`; no `/memories/:id`+`org` form; restore has no org branch on either surface. [rationale](./docs/decisions.md#org-owned-lore-archivehard-delete-over-rest)
- **Usage analytics answer record-level questions** — `GET /memories/usage` reports call AND record counts; two fail-safe headers; expiry event-sourced through the purge. [rationale](./docs/decisions.md#usage-analytics-answer-record-level-questions)
- **Profiling is SQL-level, because there is no host to profile** — Dash0 profiling needs a host-level eBPF agent and every runtime here is managed serverless; the substitutes are per-request self-time attribution (merged intervals, never summed) and `pg_stat_statements` → cumulative sums through the service-role-only `profiling` function (off until two Vault secrets exist). Don't re-open this as "add the profiler". [rationale](./docs/decisions.md#profiling-is-sql-level-because-there-is-no-host-to-profile)
- **The tool catalog is the single origin of the operation SURFACE** — `packages/schemas/src/shared/tool-catalog.ts` declares which of MCP/CLI/REST exposes each op, under what name, and the reason for each absence. Adding an operation? Follow [`docs/adding-an-operation.md`](./docs/adding-an-operation.md) (or `/add-operation`) — it is the step-by-step checklist behind this decision. Consumers *derive* (can import it), *generate* (cannot — `gen-surfaces.mjs`, committed artifacts, `--check`), or *assert* (deriving would be wrong). Never hand-edit a `*.generated.*` file; CLI **behaviour** stays hand-written in `packages/cli/src/commands.mjs`. [tiers + gates](./docs/architecture.md#surface-generation)
- **`READ_TOOLS`/`WRITE_TOOLS` stay HAND-WRITTEN, not derived from the catalog** — the duplication *is* the authorization control: deriving the gate from the thing it gates means one careless catalog edit silently opens a tool. Held to the catalog by assertion instead (`tool-catalog-parity.spec.ts`), the same way the audit vocabulary is. Do not "simplify" this.
- **A tool-originated MCP failure is an `isError` result, not a protocol error** — a failure thrown from inside a tool call returns a SUCCESSFUL result with `isError: true` so the model can see it and self-correct; a failure to DISPATCH stays a JSON-RPC error. The line is dispatch, and it maps onto the handler's own try/catch. Auth-family errors stay in-band protocol errors (a 401 or `id: null` hangs mcp-remote) and `-32603` still covers server faults. Wire-contract change: a client testing only `response.error` reads a cap hit as success. [rationale](./docs/decisions.md#a-tool-originated-mcp-failure-is-an-iserror-result-not-a-protocol-error)
- **MCP `org.*` tools serve `lk_*` tokens, gated by permission not auth tier** — matching REST; `org.list` reads, the three mutations write, and token permission is orthogonal to org ROLE (`lorekit_org_can` in the RPCs is still the only role gate). Actor via `p_actor_user_id`; every raw org read carries its own tenant predicate because the api_key path is service-role. [rationale](./docs/decisions.md#mcp-org-tools-serve-lk_-tokens-on-the-same-actor-override-rest-uses)
- **Dashboard analytics reads stay REST-only** — `/usage`, `/usage/runs`, `/tags`, `/facets`, `/pivot`, `/activity`, `/read-activity`, `/read-ranking`, `/utility`, `/clusters` get no MCP tool and no CLI command (charts, not agent primitives; most are name-bearing scope-leak surface). Guarded as `restOnly` in `telemetry-vocabulary.ts` (spec pins the TEN by name). `/clusters`/`/utility` have agent-side equivalents that are BETTER (`lorekit dedupe`, `memory.list max_opened_count => 0`); `/relevant` is NOT one of them (`memory.list order=rank` covers it). [rationale](./docs/decisions.md#dashboard-analytics-reads-stay-rest-only)
- **The Explorer's Duplicate Clusters panel is a PANEL, not an instrument** — near-duplicate grouping is not a `?filters=` dimension (never in `explorer-instruments.ts`/`filters.ts`) and is READ-ONLY (never merges/edits/deletes). Two gates in order: the `lore-explorer-duplicate-clusters` flag (default `off`, copy-and-suffix resolver per render site so `LoreExplorer.tsx` carries no `&&`) decides the surface EXISTS, then the `open` preference (`PREFERENCE_KEYS.explorerClustersOpen`) decides whether it FETCHES. Non-modal LEFT sidebar (flex sibling, never `position:fixed`); selecting a cluster SWAPS the Explorer's own list through the same `LessonCard`, bridged by optimistic cache seeding (`seedOptimisticLesson`). `GET /memories/clusters` ships unflagged. [rationale](./docs/decisions.md#dashboard-analytics-reads-stay-rest-only)
- **Dashboard analytics live on one dedicated `/insights` route** (unconditional — the gating flag is gone) composing `HealthSummary`/`UsageHealth`/`AgentBreakdown`/`ScopeConsumption`/`LoreUtilityGrid`/`RunsList` led by `LoreCostHeadline`. Explorer (`/lore`) stays the home slot for find/edit and keeps onboarding (`PendingInvitesBanner`/`OnboardingChecklist`/first-token mint); `/insights` is analytics-only. Load-bearing invariants: TWO independent range controls (never one page-wide picker — the shared one is bounded-only `24h`/`7d`/`30d`/`90d`, `ScopeConsumption` keeps its own window); `HealthSummary`/`UsageHealth` EXCLUDE dashboard-originated reads (`excludeDashboardReads`), `AgentBreakdown` keeps them; the verdict is TWO-DIMENSIONAL (`healthVerdict` reliability AND `readCoverage` records-per-read, reporting the WORSE, over record-bearing tools only). [rationale](./docs/decisions.md#dashboard-analytics-live-on-one-dedicated-insights-route)
- **A COUNT must describe the rows its LIST returns — retention thresholds reach all four readers** — the five conditions (`min_age_days`, `unseen_days`, `max_seen_count`, `max_read_count`, `max_opened_count`) narrow `list`/`_facets`/`_activity`/`_pivot` alike through ONE shared `lorekit_match_retention` (00108; cutoff INSTANTS, applied in the `base` CTE WHERE — never an `ok_*` flag, so it narrows even the self-excluded dimension). `?? null` is never a truthiness check (`max_opened_count => 0` is the point of 00105); all-null params equal omitting them; the Read stat card stays scope-level (`usage_events` can't answer a per-lesson threshold). [rationale](./docs/decisions.md#a-count-must-describe-the-rows-its-list-returns)
- **A read must never restamp `updated_at`** — `lorekit_record_memory_reads` bumping counters with a plain UPDATE fired the `updated_at` trigger, so a `memory.list` page restamped every row it returned; since `updated_at` is the Explorer/`order=recency` default sort, read activity was driving the order agents received lore in. 00103 suppresses the trigger inside the read-recording function. Read counters move on reads; `updated_at` moves on writes; no column does both. [rationale](./docs/decisions.md#a-read-must-never-restamp-updated_at)
- **Lore value is the RATIO `opened_count / read_count`, and `/insights` leads with the bill** — absolute counters are supply-side (`seen_count`≈1 for 88% of rows; `read_count` ranks scope breadth), so dividing cancels the confound. 00104 adds `opened_count`; 00106 adds `GET /memories/utility`. FIVE states not four — `load-bearing`/`specialist`/`noise-tax`/`dormant` from two booleans behind an evidence floor (age ≥ 7d AND ≥ 10 deliveries), else `unproven`; a `0` is captioned `counting_since`, never "never". ONE threshold origin (`LESSON_UTILITY_THRESHOLDS` in `@lorekit/schemas`, read by the TS chip AND passed into SQL as params). Two windows: lifetime census on `memories` vs windowed cost on `memory_read_daily`. `HotColdLore` is replaced, but `/read-ranking` and `max_read_count` are NOT deprecated (they answer context COST). [rationale](./docs/decisions.md#lore-value-is-a-ratio-and-insights-leads-with-the-bill)
- **A citation is the agent's word, and it is a fact about a RUN** — pull-through under-counts SessionStart-injected lore by construction, so 00107 adds `cited: string[]` (`scope::key`) to `memory.write`/`POST /memories`, a `memory_citations` ledger, and `memories.cited_count`/`last_cited_at`. A FIELD, not a `memory.cite` verb (no extra `tools/list` cost). EVIDENCE, never a denominator — voluntary, so `0` = "nothing said so"; no rate/share. The run comes from `X-LoreKit-Correlation-Id`, never the body. Ref grammar is self-contained (`isReferenceScope`, deliberately NOT `validateScope`); failures SILENT, lists truncate at 32, idempotency `(cited, citing, coalesce(correlation_id,''))`. [rationale](./docs/decisions.md#a-citation-is-the-agents-word-and-it-is-a-fact-about-a-run)
- **`memory.read` batching (`refs: string[]`) is conditional, verbatim-scoped, and inflates a signal it doesn't count** — MCP, `POST /memories/read` and variadic `lorekit show` fetch several `scope::key` in one call. Response shape is CONDITIONAL: singular `scope`+`key` keeps its exact pre-existing shape, `entries`/`missing` appears only under `refs` (every caller needs a shape-dispatch). Batch resolution matches the STORED scope VERBATIM (`groupRefsByScope`, NO `validateScope` normalisation — transports differ in casing), and `parseMemoryRefs` won't fold case-only-different refs. Batching degrades `opened_count` pull-through; the 32-ref cap (shared with `cited`) bounds but doesn't remove it. `readCoverage` isn't comparable across the rollout window — accepted gap. [rationale](./docs/decisions.md#memoryread-batching-is-conditional-verbatim-scoped-and-inflates-a-signal-it-doesnt-count)
- **An omitted scope on a read means EVERYWHERE, not `global`** — `memory.read`/`memory.list`/`memory.list_archived` and `lorekit show` no longer require a scope; a `global` default would turn a recoverable error into a silent `null` for repo-scoped lore. Read resolves across every visible scope: `memory.read` returns ONE winner by scope-TYPE precedence (`project→branch→repo→global`, the `readOrder` in import-free `scope/scope-precedence.ts`; ties by `updated_at` desc then scope asc — a TOTAL order), the list tools widen. Three copies, guarded by `edge-parity.spec.ts` (byte) + `scope-precedence-parity.spec.ts` (behaviour). A singular read always reports the answering `scope` (+ `other_scopes` when ambiguous); only the winner hits `recordMemoryReads`. REST list routes stay account-wide (unchanged). [rationale](./docs/decisions.md#an-omitted-scope-means-everywhere-not-global)
- **A tool's `inputSchema` carries NO top-level `oneOf`/`anyOf`/`allOf`** — Amazon Bedrock refuses any such tool and fails the ENTIRE `tools/list` request (it took Agent0 down post-#654). Mutually-exclusive argument shapes go in the tool `description` + handler enforcement; the schema stays permissive. Two gates: `JsonSchemaObject` omits the field (compile error under `satisfies`) and `tool-catalog-parity.spec.ts` asserts no WIRE projection carries any of the three. [rationale](./docs/decisions.md#a-tools-inputschema-carries-no-top-level-oneofanyofallof)
- **A skill's `metadata.version` must be bumped on any content change, and CI enforces it** — `doctor`/`lorekit update`/the SessionStart drift nudge compare installed vs shipped `SKILL.md` version (a no-op if the stamp never moves — all three skills sat at `1.0.0` through four content PRs). Version-based by design (a human decides whether to nudge; not content-hash, not tied to CLI version). `scripts/ci/skill-version-guard.mjs` is a real `ci.yml` gate over SOURCE `packages/cli/skill/*` only (the plugin mirror is guarded by `sync-plugin-skill.mjs --check`). [rationale](./docs/decisions.md#a-skills-metadataversion-must-be-bumped-on-any-content-change-and-ci-enforces-it)
