# reusable-workflows

Centrally-maintained, versioned GitHub Actions reusable workflows for every
Sheth-Family venture. Built to solve a specific problem: `template-fullstack-app`
(the golden image) used to get *copied* into each new venture at scaffold
time, and after that every copy drifted independently — a fix or new check
had to be hand-applied to every venture that already existed. LiveSimpli/simpli
spent a full day (2026-08-31/09-01) paying for exactly that kind of drift:
months of divergence between its `main` and `staging` branches, a deploy
pipeline with a hardcoded Supabase pooler host that was wrong for the actual
project, and a missing production secret nobody had ever hit because the
pipeline had been broken for so long nothing reached that step. This repo
exists so that class of problem gets fixed once, centrally, not once per
venture.

## How it works

Each workflow here is triggered by `workflow_call`, not `push`/`pull_request`
directly. A venture doesn't copy the logic — it has a thin wrapper file that
calls this repo's version:

```yaml
# a venture's .github/workflows/ci-checks.yml
name: CI — Type check, Lint, Audit
on:
  push: { branches: [main] }
  pull_request: {}
jobs:
  quality:
    uses: Sheth-Family/reusable-workflows/.github/workflows/ci-checks.yml@v1
```

## Versioning

Pin to a version tag (`@v1`, `@v1.2.0`), never `@main` — same discipline
already used for every pinned action SHA in these workflows. A fix or new
check here becomes a new tag; a venture doesn't get it until it bumps its
ref.

That bump doesn't have to be manual. **Dependabot already tracks reusable
workflow references** the same way it tracks `actions/checkout` version
bumps — it's the same `github-actions` package ecosystem. A venture with
`package-ecosystem: "github-actions"` in its `dependabot.yml` (the golden
image already has one) will get an automatic, reviewable PR whenever this
repo cuts a new tag. That's the whole point: central source of truth,
deliberate per-venture adoption, no silent action-at-a-distance breakage.

Cutting a release:

```bash
git tag v1.1.0
git push origin v1.1.0
# and move/create the major tag so @v1 always points at the latest v1.x.y
git tag -f v1
git push origin v1 --force
```

## Cross-repo `vars`/`secrets` — read this before adding an input

A workflow called via `uses: ./.github/workflows/x.yml` (same repo) can
freely reference the caller's `vars.*`/`secrets.*` context directly, because
it's really just the same workflow run split into files. A workflow called
**cross-repo** (`uses: Org/OtherRepo/.github/workflows/x.yml@ref`, which is
everything in this repo) cannot rely on that — GitHub's docs confirm
`secrets: inherit` works cross-repo within an org, but don't document `vars`
behavior at all, and it's not safe to assume. **Every value a called
workflow here needs must be an explicit `workflow_call` input.** Don't add a
bare `vars.X` reference inside any workflow in this repo — pass it in from
the caller, which resolves its own `vars.X` at the call site (unambiguously
same-repo, always correct).

**One narrow, deliberate exception:** `build-deploy.yml`'s `service_account`
input falls back to `${{ vars.GCP_DEPLOYER_SA }}` read from *inside* its own
jobs, each of which has `environment: ${{ inputs.environment }}` set. This
exists because the natural fix — the *caller's* job resolving its own
`vars.GCP_DEPLOYER_SA` and passing it in, exactly like every other input
here — doesn't work for this specific variable: it's Environment-scoped
(set per staging/prod by `gcp-bootstrap`'s `wire-github.sh`), and a caller
job that itself calls a reusable workflow via `uses:` isn't allowed to also
declare `environment:` (actionlint-enforced), so it has no Environment
context to resolve an Environment-scoped variable from at all. Reading it
inside this workflow's own Environment-scoped jobs instead is the only way
to get a correct value — status as of 2026-09-03: reasoned through and
code-reviewed, not yet confirmed by a real deploy. This note will be
updated once one has. Don't treat this as license to add other bare
`vars.X` reads here — this is a workaround for one specific, unusual
constraint, not a general pattern.

## What's here

- `ci-checks.yml` — typecheck, lint, `npm audit --audit-level=high`. Takes a
  `node-version` input (default `22`) — pins Node explicitly rather than
  relying on whatever the runner image happens to preinstall.
- `ci-lint-workflows.yml` — actionlint + shellcheck on the calling repo's
  own workflow files.
- `secret-scan.yml` — gitleaks, pinned binary install (not the marketplace
  action — license-gated for org repos and has a startup-failure history).
  Expects a `.gitleaks.toml` at the caller's root.
- `opengrep-scan.yml` — SAST via Opengrep, same pinned-binary-install
  pattern. Takes an `opengrep-version` input.
- `verify-staging-gate.yml` — blocks a prod deploy unless the exact commit
  has already been through staging. Optional; wire in as a `needs:` ahead of
  a deploy job once a venture has a staging environment.
- `promote-to-prod.yml` — fast-forwards `main` to a specific commit
  (deliberately NOT a merge commit — see the file's own header for why a
  `--no-ff` merge always fails `verify-staging-gate.yml`'s ancestry check).
  Call it from a venture's own thin wrapper, triggered by `workflow_run`
  once that venture's staging deploy succeeds, passing
  `commit-sha: ${{ github.event.workflow_run.head_sha }}` and
  `triggering-conclusion: ${{ github.event.workflow_run.conclusion }}` —
  promoting the exact tested commit rather than "whatever staging's tip
  is right now" (which could have moved past the tested commit by the
  time this job runs), and refusing to run at all unless the triggering
  conclusion was `"success"`, as a fail-closed default a caller can't
  forget to add. Needs a `PROMOTE_PAT` secret — a real PAT, not
  `GITHUB_TOKEN`, since pushes made with the default token don't trigger
  other workflow runs — with **both** `Contents: Read and write` and
  `Workflows: Read and write` (any staging commit touching a
  `.github/workflows/*` file needs the latter, and that's routine, not
  exotic). Provision one PAT per venture, fine-grained and scoped to that
  repo only — not a credential shared across ventures.
- `build-deploy.yml` — build, push to Artifact Registry, run Supabase
  migrations, deploy to Cloud Run, smoke-test, and auto-rollback on smoke
  failure. See the file's own header comment for the full input list and
  required GCP/Supabase setup. Optional capabilities (all off/empty by
  default, so adopting them is opt-in per venture): `test-command` (run
  before the Docker build), `build-secrets` (Secret-Manager-backed
  `--build-arg`s, e.g. for `NEXT_PUBLIC_*` values that must be baked in),
  `extra-env-vars` (plain `--set-env-vars` beyond `NEXT_PUBLIC_ENV`/`DD_ENV`),
  a Datadog deploy-event ping, a Cloudflare cache purge, a post-deploy
  Cloud Run log audit for DB errors, and Mission Control evidence emission
  (POSTs the smoke-test result as JSON if `MISSION_CONTROL_URL` is set, and
  always uploads it as a build artifact). These were generalized out of
  LiveSimpli/simpli's own build-deploy.yml when simpli migrated onto this
  workflow (2026-09-01) — simpli had grown real capability (a 220-test
  pre-deploy gate, build-time `NEXT_PUBLIC_*` secrets, Datadog/Cloudflare/log-audit
  steps, Mission Control) that a naive migration would have silently dropped.

## The Supabase migration step, specifically

`build-deploy.yml`'s migration step resolves the Supabase connection
pooler's host by calling the Management API
(`GET /v1/projects/{ref}/config/database/pooler`) at deploy time — it never
hardcodes or guesses a region or pooler-instance number. That's not
incidental caution: a hardcoded guess (`aws-0-us-east-1`, matching a region
that turned out to be wrong, then `aws-0-us-east-2`, matching the right
region but the wrong instance number) is exactly what caused LiveSimpli/simpli's
2026-08-31/09-01 incident, and the failure mode — `FATAL: tenant/user
postgres.<ref> not found` — reads like a bad password, not a routing
problem, so it's genuinely easy to chase the wrong thing. Needs a
`SUPABASE_ACCESS_TOKEN` (Management API personal access token) passed as a
secret, plus `supabase-project-ref` and `supabase-db-url-secret` inputs.
