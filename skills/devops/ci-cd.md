# CI/CD Pipeline

| Field | Value |
|---|---|
| **Version** | 1.1.0 |
| **Last Updated** | 2026-09-21 |
| **Applicability** | GitHub-hosted repositories, GitHub Actions, Node.js/TypeScript projects |
| **Dependencies** | GitHub Actions, Node.js 18+, package manager (npm/pnpm/yarn) |

---

## GitHub Actions Workflow Design

### File Structure

```
.github/
  workflows/
    ci.yml              ← Runs on every PR
    deploy-staging.yml  ← Deploys to staging on merge to main
    deploy-prod.yml     ← Deploys to production (manual or tag-triggered)
    scheduled.yml       ← Nightly tasks (dependency audit, cleanup)
  actions/
    setup/action.yml    ← Composite action for shared setup steps
```

### Core Principles

- **One responsibility per workflow**: Separate CI checks from deployment.
- **Fail fast**: Run cheap checks (lint, types) before expensive ones (tests, builds).
- **Cache aggressively**: Dependencies, build artifacts, Docker layers.
- **Pin action versions**: Use SHA, not tags (`actions/checkout@<sha>`).
- **Minimize secrets exposure**: Scope secrets to the jobs that need them.

### Composite Action for Shared Setup

```yaml
# .github/actions/setup/action.yml
name: 'Project Setup'
description: 'Install dependencies with caching'
runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version-file: '.nvmrc'
        cache: 'pnpm'
    - run: pnpm install --frozen-lockfile
      shell: bash
```

---

## CI Workflow — PR Checks

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm lint
      - run: pnpm format:check

  typecheck:
    name: Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm typecheck

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: [lint, typecheck]
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm test --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: coverage/

  build:
    name: Build Verification
    runs-on: ubuntu-latest
    needs: [lint, typecheck]
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: .next/
          retention-days: 1

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm audit --audit-level=high
      - uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified
```

### Job Dependency Graph

```
PR opened
  |
  +-- lint (fast, ~30s)
  +-- typecheck (fast, ~30s)
  +-- security (independent)
  |
  +-- test (needs: lint, typecheck)
  +-- build (needs: lint, typecheck)
```

---

## Automated Testing on PR

### Test Strategy by Layer

| Layer | Tool | Trigger | Scope |
|---|---|---|---|
| Unit | Vitest / Jest | Every PR | Functions, utilities, hooks |
| Component | Testing Library | Every PR | UI components in isolation |
| Integration | Vitest + MSW | Every PR | Feature flows with mocked APIs |
| E2E | Playwright | Pre-deploy / nightly | Critical user paths |

### Test Configuration

```yaml
# E2E on PR (optional, for critical paths only)
e2e:
  name: E2E Tests
  runs-on: ubuntu-latest
  needs: [build]
  steps:
    - uses: actions/checkout@v4
    - uses: ./.github/actions/setup
    - uses: actions/download-artifact@v4
      with:
        name: build-output
        path: .next/
    - run: pnpm exec playwright install --with-deps chromium
    - run: pnpm e2e
    - uses: actions/upload-artifact@v4
      if: failure()
      with:
        name: playwright-report
        path: playwright-report/
```

### Test Optimization

- **Shard large test suites**: `vitest --shard=1/3` across parallel jobs.
- **Run affected tests only**: `vitest --changed HEAD~1` for fast feedback.
- **Playwright**: Run only smoke tests on PR; full suite nightly.
- **Cache Playwright browsers**: Use `actions/cache` with browser version key.

---

## Linting & Formatting Checks

### Required Tools

| Tool | Purpose | Config File |
|---|---|---|
| ESLint | Code quality, bug prevention | `eslint.config.js` |
| Prettier | Consistent formatting | `.prettierrc` |
| `prettier --check` | Verify formatting without modifying | N/A |

### Enforcement Strategy

- **Pre-commit** (local): `lint-staged` + `husky` for fast feedback.
- **CI** (authoritative): Full lint and format check on every PR.
- **Auto-fix**: Never auto-fix in CI. CI only validates; developers fix locally.

### `package.json` Scripts

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format:check": "prettier --check .",
    "format": "prettier --write .",
    "typecheck": "tsc --noEmit"
  }
}
```

---

## Type Checking

### CI Type Check

```bash
tsc --noEmit --pretty
```

- Runs against the full project, catches cross-file type errors.
- Separate from build — ensures types are valid even if build succeeds with warnings.
- For monorepos: `turbo typecheck` to check all packages in dependency order.

### Strict Mode

Enforce in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

---

## Dependency Vulnerability Scanning

### Approaches

| Method | When | Coverage |
|---|---|---|
| `pnpm audit` | Every PR | Known CVEs in dependencies |
| GitHub Dependabot | Continuous | Auto-PRs for vulnerable deps |
| Snyk / Socket | Continuous | Supply chain analysis |

### Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
    open-pull-requests-limit: 10
    groups:
      minor-and-patch:
        update-types: [minor, patch]
    reviewers:
      - team-name
```

### Audit in CI

```yaml
- name: Audit Dependencies
  run: pnpm audit --audit-level=high
  continue-on-error: false  # fail the build on high/critical
```

---

## Secrets Scanning

### Prevention

- **Pre-commit hook**: Use `trufflehog` or `gitleaks` locally.
- **CI scan**: Run on every PR against the diff.
- **`.gitignore`**: Always exclude `.env*` (except `.env.example`).

### CI Integration

```yaml
- name: Secrets Scan
  uses: trufflesecurity/trufflehog@main
  with:
    base: ${{ github.event.pull_request.base.sha }}
    head: ${{ github.event.pull_request.head.sha }}
    extra_args: --only-verified
```

### If a Secret is Leaked

1. **Rotate immediately** — assume compromised.
2. Revoke the old credential in the provider's dashboard.
3. Remove from git history: `git filter-repo` (rewrite history, force push).
4. Update all environments with new credentials.
5. Audit access logs for unauthorized use.

---

## Content Denylist Guard (Names and Identifiers)

Secrets scanners find credentials. They do not find a person's name, a customer's
name, a handle inside a URL, or a private email in a fixture, and some
repositories must never contain those. A denylist check, run as an ordinary
test, makes that rule enforceable instead of aspirational.

### Shape

- **The list is held by the owner, outside version control.** A gitignored file
  at the repository root, one entry per line. The list is itself sensitive (it
  is exactly the strings you are protecting), so it is never committed and an
  agent working in the repository never reads it.
- **Scan what git tracks** (`git ls-files`), case-insensitively, whole-word,
  skipping binaries. Staged-but-uncommitted files count; untracked files do not.
- **Output is `path:line` and nothing else.** Never print the matched text or
  the entry: the checker's own output must not leak what the list protects.
  Reports, commit messages, and decision logs follow the same rule and cite
  locations only.
- **The same logic runs as a script and as a test**, so the normal test run is
  the gate and there is no separate step to forget.

### Two honesty rules

- **An absent list is "not enforced", printed, never a silent pass.** On a
  machine without the list the test SKIPS with a printed line. A green check
  that checked nothing is worse than no check.
- **Exemptions are explicit, reasoned, and printed on every run.** Some paths
  cannot be edited (frozen history, an already-applied migration). List each
  with its reason in the checker, do not scan them, and print the list on every
  run, including when the denylist is absent. An exemption nobody can see is a
  hole. Adding one should take a recorded decision.

### Testing the guard without a real name

Build a throwaway git repository in a temp directory, with a denylist holding a
nonsense token: a hit in a normal path fails; the same token in an exempt path
does not; the exempt list is printed both with and without a list present; and
the printed output never contains the token.

### Seeding the list

Git metadata is a reasonable first source (author name, the email's local part
and any non-generic domain word, the remote's account handle), but it often
collapses to a single handle. It will not contain a full name that only appears
in prose, so the owner adds those by hand. Until they do, the gate misses most
of what a scrub was for.

---

## Build Verification

### What to Verify

- **Build succeeds**: `pnpm build` exits 0.
- **No build warnings treated as errors**: Configure `eslint` and `tsc` to fail on warnings in CI.
- **Bundle size budget**: Fail if bundle exceeds threshold.

### Bundle Size Check

```yaml
- name: Check Bundle Size
  run: |
    pnpm build
    npx @next/bundle-analyzer || true
    # Or use size-limit
    npx size-limit
```

### `size-limit` Configuration

```json
// package.json
{
  "size-limit": [
    { "path": ".next/static/**/*.js", "limit": "300 kB", "gzip": true }
  ]
}
```

---

## Automated Deployment

### Deployment Pipeline

```
PR merged to main
  |
  v
CI checks pass
  |
  v
Deploy to staging (automatic)
  |
  v
Smoke tests pass
  |
  v
Deploy to production (manual approval or automatic)
```

### Staging Deployment Workflow

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm build
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
      - name: Deploy
        run: vercel deploy --prod --token=${{ secrets.VERCEL_TOKEN }}
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
```

### Production Deployment Workflow

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Production

on:
  workflow_dispatch:       # manual trigger
  release:
    types: [published]     # or on release

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm build
      - name: Deploy
        run: vercel deploy --prod --token=${{ secrets.VERCEL_TOKEN }}
```

---

## Staging Promotion

### Promotion Strategy

| Approach | When to Use |
|---|---|
| Git-flow (merge to prod branch) | When staging/prod have different branches |
| Vercel promote | When staging deployment artifact is verified |
| Tag-based (`v1.2.3`) | When releases are versioned |
| Manual approval | When human review required before prod |

### GitHub Environment Protection

```yaml
# In repo Settings > Environments > production:
# - Required reviewers: 1+
# - Wait timer: optional
# - Deployment branches: main only
```

---

## Smoke Testing

### Post-Deploy Smoke Tests

```yaml
smoke-test:
  name: Smoke Test
  needs: [deploy]
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: ./.github/actions/setup
    - name: Health Check
      run: |
        for i in 1 2 3 4 5; do
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$DEPLOY_URL/api/health")
          if [ "$STATUS" = "200" ]; then exit 0; fi
          sleep 5
        done
        exit 1
      env:
        DEPLOY_URL: ${{ needs.deploy.outputs.url }}
    - name: Critical Path Tests
      run: pnpm playwright test --project=smoke
      env:
        BASE_URL: ${{ needs.deploy.outputs.url }}
```

### What to Smoke Test

- **Health endpoint**: `/api/health` returns 200.
- **Homepage loads**: Returns 200, contains expected content.
- **Auth flow**: Login page renders, OAuth redirects work.
- **Critical API**: One key API endpoint responds correctly.
- **Database connectivity**: Health check verifies DB connection.

---

## Workflow Optimization

### Caching

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.pnpm-store
      .next/cache
    key: ${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-
```

### Concurrency Control

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true   # cancel previous deploys on same branch
```

### Matrix Builds (Multi-Platform)

```yaml
strategy:
  matrix:
    node: [18, 20]
    os: [ubuntu-latest]
  fail-fast: true
```

---

## Notification & Monitoring

### Slack Notification on Failure

```yaml
- name: Notify Failure
  if: failure()
  uses: slackapi/slack-github-action@v2
  with:
    webhook: ${{ secrets.SLACK_WEBHOOK }}
    payload: |
      {"text": "CI failed on ${{ github.ref }} by ${{ github.actor }}"}
```

### Status Badges

```markdown
![CI](https://github.com/org/repo/actions/workflows/ci.yml/badge.svg)
```
