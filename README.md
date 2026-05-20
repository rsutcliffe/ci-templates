# ci-templates

Shared CI/CD workflows and cost-control policy for the `rsutcliffe` account.

Edit the standard once here, version it with a tag, and every consuming repo
inherits it on its own schedule.

## Reusable workflows

| Workflow | For | Caller pins |
|---|---|---|
| `ci-shared.yml` | Python backend + Next.js frontend (e.g. hoci/ScoreView) | `@v3` |
| `ci-next-basic.yml` | Generic Node / Next.js (npm or pnpm), optional Playwright | `@v3` |

Pin callers to a **tag** (`@v3`), never `@main` — template changes then
propagate only when a repo deliberately bumps its pin.

### Caller example — `ci-next-basic.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
    paths-ignore: ['docs/**', '**/*.md', '.claude/**']
  pull_request:
    branches: [main]
    paths-ignore: ['docs/**', '**/*.md', '.claude/**']

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: rsutcliffe/ci-templates/.github/workflows/ci-next-basic.yml@v3
    with:
      package-manager: npm
      node-version: "22"
      type-check-command: "npx tsc --noEmit"
      lint-command: "npm run lint"
      test-command: "npm test"
      run-playwright: false
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Cost-control policy

The May 2026 budget overrun (hoci: 241 CI runs in 7 days) traced to
per-commit triggers fanning out across GitHub Actions and Vercel. Every repo
follows these rules:

1. **Concurrency cancel-in-progress** — superseded runs are killed. In the
   caller's `concurrency:` block (triggers can't be shared).
2. **Draft PRs skip CI** — built into the reusable workflows (`v3+`). Bespoke
   workflows add `if: github.event_name != 'pull_request' ||
   github.event.pull_request.draft == false` to each job.
3. **Path filters** — `paths-ignore` for `docs/**`, `**/*.md`, `.claude/**` in
   the caller's `on:` block.
4. **Heavy jobs run post-merge only** — Playwright / e2e / Lighthouse / bundle
   analysis gate on `push` to `main`, not every PR commit. UAT on Vercel
   previews is the real merge gate.
5. **Merge queue on high-traffic repos** — batches N merges into one CI run.
6. **Dependabot grouped** — see template below; never let bumps fan out into
   individual PRs.
7. **One PR = one logical change, squash-merge** — fewer `main` deploys.

## Vercel preview-deploy control

Add to each Vercel-hosted repo's `vercel.json` (root of the Vercel project) to
skip preview builds when only non-deployable files changed:

```json
{
  "ignoreCommand": "git diff --quiet HEAD^ HEAD -- ':!docs' ':!Documentation' ':!*.md' ':!.github' ':!.claude'"
}
```

`git diff --quiet` exits 0 (skip build) when no deployable file changed,
1 (build) otherwise — matching Vercel's ignoreCommand contract.

Also, in each Vercel project dashboard: **Settings → Git** — keep preview
deploys to PR branches; confirm the "Ignored Build Step" honours
`ignoreCommand`.

## Dependabot template

Copy to `.github/dependabot.yml`. Grouping collapses many bumps into ≤2
PRs/week per ecosystem.

```yaml
version: 2
updates:
  - package-ecosystem: npm        # or pip / cargo
    directory: /
    schedule:
      interval: weekly
      day: monday
    open-pull-requests-limit: 3
    groups:
      production-dependencies:
        dependency-type: production
        update-types: [minor, patch]
      development-dependencies:
        dependency-type: development
        update-types: [minor, patch]
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
      day: monday
    open-pull-requests-limit: 3
```

## Monitoring

Set the account Actions **budget alert at 50%** (GitHub → Settings → Billing →
Budgets), not the default 90% — 50% gives time to react before a launch-week
spike blows the cap.

## Maintenance

Bump the template tag quarterly. Repos re-pin (`@v3` → `@v4`) when convenient.
