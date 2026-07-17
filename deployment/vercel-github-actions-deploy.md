# Deploying a frontend to Vercel via GitHub Actions

How the `food-delivery-platform` GitHub org wires a Next.js frontend repo to deploy to Vercel
production on every push to `main` — gated on lint + typecheck passing first. This is a deliberate
alternative to Vercel's native Git integration (which would deploy directly on push, with no
explicit CI gate in between).

**Reference implementations:**
- [`customer-frontend`](https://github.com/food-delivery-platform/customer-frontend) — the original;
  built across branches `adding-deploy-to-vercel` and `fix-vercel-deploy-token-command`, both merged
  to `main`.
- [`courier-frontend`](https://github.com/food-delivery-platform/courier-frontend) — set up
  2026-07-17 on branch `adding-deploy-to-vercel`, mirroring the above.

---

## How it works

Two workflows, chained via `workflow_run` rather than combined into one job:

1. **`.github/workflows/checks.yml`** — runs on every push, to every branch: `npm ci`, `npm run
   lint`, `npm run typecheck`.
2. **`.github/workflows/deploy-vercel.yml`** — triggered when the `Checks` workflow *completes* on
   `main` (`workflow_run`, not `push` — this is what lets deploy wait on checks without duplicating
   the lint/typecheck steps). Guarded by
   `if: github.event.workflow_run.conclusion == 'success'`, so a failing lint/typecheck run never
   deploys. Builds and deploys via the Vercel CLI (`vercel pull` → `vercel build` → `vercel deploy
   --prebuilt --prod`), not Vercel's own Git integration.

Splitting these into two workflows (rather than one `push`-triggered workflow that lints then
deploys) means the deploy job always runs against the exact commit `Checks` validated
(`github.event.workflow_run.head_sha`), and a slow/flaky deploy step never blocks the fast
lint/typecheck feedback on non-`main` branches (`Checks` runs everywhere; `deploy-vercel` only ever
fires for `main`).

No `vercel.json` or `.vercel/` directory is committed to the repo — a plain Next.js app needs
neither. The Vercel *project* itself (which org/project this repo maps to) is created once by
importing the repo in Vercel's dashboard, or by running `vercel link` locally as any authenticated
team member; the three secrets below are what let the CI runner act as that same linked project
non-interactively.

---

## Procedure for a new repo

1. **Confirm `package.json` has the three scripts `checks.yml` calls:** `lint`, `typecheck` (`tsc
   --noEmit`), and the standard `build`. Add `typecheck` if it's missing — `courier-frontend` didn't
   have one before this was set up.
2. **Create a branch** named `adding-deploy-to-vercel` (the convention both repos in this org use —
   keep it even though the branch name says nothing about which app; it's the recognizable marker
   for "this PR wires up Vercel deploy").
3. **Add `.github/workflows/checks.yml`** and **`.github/workflows/deploy-vercel.yml`** — copy
   verbatim from `customer-frontend`'s `main` branch (not its `adding-deploy-to-vercel` branch —
   that one has since been superseded by a small fix, see the gotcha below; `main` always has the
   current, correct version). Node version and package manager should match whatever the repo
   already uses elsewhere in its own CI (both repos here use Node 22 + npm).
4. **Create the three required GitHub Actions secrets** on the new repo (Settings → Secrets and
   variables → Actions) — this is a repo admin action, not something committed in code:
   - `VERCEL_TOKEN` — a personal or team token from Vercel's Account Settings → Tokens.
   - `VERCEL_ORG_ID` and `VERCEL_PROJECT_ID` — from that repo's already-linked Vercel project; the
     easiest way to get both at once is running `vercel link` locally against the project and
     reading `.vercel/project.json` (then discard that file — it's local linkage metadata, not
     something to commit).
5. **Push the branch and merge to `main`** (via PR, per this org's normal workflow). The first push
   to `main` after merging will run `Checks`, then `deploy-vercel.yml` fires automatically on that
   `workflow_run` completion.
6. **Set the app's own runtime env vars** (e.g. `courier-frontend`'s `NEXT_PUBLIC_API_BASE_URL`) in
   Vercel's own **Project Settings → Environment Variables**, scoped per Vercel environment — never
   in these workflow files or any tracked file. See that repo's own `docs/PLAN.md` Phase 11c /
   `.env.local.example` for the local-dev equivalent.

---

## Known gotcha (already fixed in what you're copying from)

The first version of `deploy-vercel.yml` passed `--token="$VERCEL_TOKEN"` explicitly to every
`vercel` CLI subcommand inside a multi-line `run: >` block. This broke the deploy (branch
`fix-vercel-deploy-token-command` in `customer-frontend`). The fix — already applied on `main`, and
what both repos' current workflow files use — drops the explicit `--token` flag entirely and lets
the Vercel CLI pick up `VERCEL_TOKEN` from the job's `env:` block on its own, and adds
`VERCEL_TELEMETRY_DISABLED: 1` to the same `env:` block. If you ever find yourself copying an older
revision of this file (e.g. from a stale branch instead of `main`), you'll reintroduce this bug —
always copy from `main`.
