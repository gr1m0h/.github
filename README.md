# gr1m0h/.github

Central configuration repository for `gr1m0h`'s GitHub organization-style policy
distribution. Holds:

- **Rulesets** (branch / tag / push protection) — applied automatically to target repos
- **Reusable workflows** (CodeQL, OSV, gitleaks, dependency-review, govulncheck, npm-audit, actions-pin-lint) — consumed by target repos via `workflow_call`
- **Caller workflow templates** — drop-in starters for each language
- **Community health files** (`SECURITY.md`, `SUPPORT.md`, `CODEOWNERS`, `pull_request_template.md`, `FUNDING.yml`, ISSUE templates) — inherited by every repo in the owner that does not define its own

```
.
├── .github/
│   ├── workflows/              # reusable workflows (workflow_call) + rulesets-apply
│   ├── ISSUE_TEMPLATE/
│   └── dependabot.yml
├── security/
│   ├── baseline.yaml           # single source of truth for what applies where
│   └── rulesets/*.json         # ruleset payloads consumed by the GitHub API
├── templates/
│   └── caller-*.yml.template   # copy these into each target repo
├── CODEOWNERS  SECURITY.md  SUPPORT.md  pull_request_template.md  FUNDING.yml
```

---

## 1. Rulesets (fully automated)

`rulesets-apply.yml` runs on every push to `main` that touches
`security/baseline.yaml` or `security/rulesets/**`, and can also be triggered
manually via `workflow_dispatch`.

### Flow

1. Mint an installation token using the `BASELINE_APP_ID` / `BASELINE_APP_PRIVATE_KEY` GitHub App
2. Enumerate target repos:
   - If `repos:` is present in `baseline.yaml` → opt-in list
   - Otherwise → all repos in `gr1m0h` matching `scope` (visibility / archived / forks)
3. For each repo, resolve which rulesets apply:
   - `repo_overrides[<repo>].rulesets` (full replace), or
   - `defaults.rulesets + repo_overrides[<repo>].rulesets_add`
4. Compare against existing rulesets by `name`, then `PUT` (update) or `POST` (create)

### One-time setup

| Item | Where | Notes |
|------|-------|-------|
| `BASELINE_APP_ID` | Variables (repo or org) | GitHub App **Client ID** (not numeric App ID) |
| `BASELINE_APP_PRIVATE_KEY` | Secrets | PEM private key of the same App |
| App permissions | App settings | `Administration: Read & write`, `Contents: Read`, `Metadata: Read` |
| App installation | Org / user | Installed on every repo that should receive rulesets |
| Branch protection bypass | `bypass_actors` in ruleset JSON | Include the App's installation actor ID so `rulesets-apply` itself is not blocked |

### Triggering manually

```bash
gh workflow run rulesets-apply.yml -R gr1m0h/.github
gh run watch -R gr1m0h/.github
```

### Editing what gets applied

Open `security/baseline.yaml` and edit `defaults.rulesets`, `repo_overrides`, or
`exclude`. Push to `main` — the workflow runs automatically.

---

## 2. Reusable workflows (consumed by target repos)

The workflows under `.github/workflows/` that declare `on: workflow_call` are
**libraries**, not policies. Each target repo opts in by adding a caller
workflow that points at them.

| Workflow | Purpose | Inputs (defaults) |
|----------|---------|--------------------|
| `codeql-go.yml` | CodeQL `security-and-quality` for Go | `go-version: "1.24"` |
| `codeql-typescript.yml` | CodeQL for JS/TS | — |
| `dependency-review.yml` | GitHub dependency review on PRs | `fail-on-severity: high`, `deny-licenses: ""` |
| `gitleaks.yml` | Secret scanning | — |
| `govulncheck.yml` | Go vulnerability scan | — |
| `npm-audit.yml` | npm audit | — |
| `osv-scanner.yml` | OSV scan, uploads SARIF | — |
| `actions-pin-lint.yml` | Block unpinned `uses:` references | — |

All reusable workflows pin third-party actions to commit SHAs and emit the
`cicd-sensor` runtime sensor step (see commits `22122ee`, `7b8d3f8`).

### Applying to a target repo (automated)

`files-apply.yml` runs on every push to `main` that touches
`security/baseline.yaml` or `templates/**`, and can also be triggered manually
via `workflow_dispatch`. Target selection is **intentionally narrower** than
`rulesets-apply.yml`:

1. If `files_apply.repos` is defined in `baseline.yaml`, only those repos are covered (preferred)
2. Else if top-level `repos:` is defined, that list is used
3. Else falls back to scope-based enumeration

Top-level `exclude:` is always honored. For each covered repo it:

1. Detects the primary language via the GitHub API
2. Resolves the effective file list from `defaults.files` + `repo_overrides`,
   filtered by `languages`
3. Renders each template (substituting `@<tag>` with the current upstream commit SHA)
4. Force-pushes a `chore/baseline-sync` branch onto the target repo
5. Opens a PR if none exists, or refreshes the existing PR

The GitHub App needs `contents: write`, `pull-requests: write`, and
`workflows: write` on every target repo. Without `workflows: write`, push will
be rejected when any rendered file lives under `.github/workflows/`.

### Applying to a target repo (manual fallback)

Useful when the App is not yet installed on the target, or when bootstrapping a
new repo before the automation runs.

```bash
cd <target-repo>
mkdir -p .github/workflows
curl -sSL https://raw.githubusercontent.com/gr1m0h/.github/main/templates/caller-go.yml.template \
  -o .github/workflows/security.yml

# Pin the version. Pick a release tag of gr1m0h/.github (recommended) or `main`.
sed -i '' 's|@<tag>|@v1.0.0|g' .github/workflows/security.yml   # macOS
# sed -i 's|@<tag>|@v1.0.0|g' .github/workflows/security.yml    # Linux

git add .github/workflows/security.yml
git commit -m "ci: adopt gr1m0h/.github reusable security workflows"
```

**Node / TypeScript repo:** identical, but use `templates/caller-node.yml.template`.

**What the caller looks like (Go example, after substitution):**

```yaml
# <target-repo>/.github/workflows/security.yml
name: security
on:
  pull_request:
  workflow_dispatch:
permissions:
  contents: read

jobs:
  govulncheck:
    uses: gr1m0h/.github/.github/workflows/govulncheck.yml@v1.0.0
  osv-scanner:
    uses: gr1m0h/.github/.github/workflows/osv-scanner.yml@v1.0.0
    permissions:
      contents: read
      security-events: write
  gitleaks:
    uses: gr1m0h/.github/.github/workflows/gitleaks.yml@v1.0.0
  dependency-review:
    if: github.event_name == 'pull_request'
    uses: gr1m0h/.github/.github/workflows/dependency-review.yml@v1.0.0
    permissions:
      contents: read
      pull-requests: write
  codeql:
    uses: gr1m0h/.github/.github/workflows/codeql-go.yml@v1.0.0
    permissions:
      contents: read
      security-events: write
      actions: read
  actions-pin-lint:
    uses: gr1m0h/.github/.github/workflows/actions-pin-lint.yml@v1.0.0
```

### Pinning strategy

- **Production repos** → pin to a tag (`@v1.0.0`) or a commit SHA. Tags are
  preferred for readability; SHAs are mandatory if the target repo enforces
  `actions-pin-lint` against `gr1m0h/.github` itself.
- **`@main`** is acceptable only for experimentation. Renovate / Dependabot
  cannot bump it.
- Renovate's `github-actions` manager updates `uses:` references automatically
  when pinned to a tag or SHA.

### Required permissions on the calling job

The caller `permissions:` block must grant at least what each reusable
workflow requires (the table above and the templates list the minimum). Calling
workflows cannot _expand_ permissions beyond what the caller declares, so a
top-level `permissions: contents: read` plus per-job grants is the safe shape.

---

## 3. Community health files

Files at the root of this repo (`SECURITY.md`, `SUPPORT.md`, `CODEOWNERS`,
`pull_request_template.md`, `FUNDING.yml`, `.github/ISSUE_TEMPLATE/*`,
`.github/dependabot.yml`) are picked up by GitHub automatically for any repo in
`gr1m0h` that doesn't define its own copy. **Nothing to do per-repo** — just
edit the file here.

Caveat: `dependabot.yml` is **not** inherited (Dependabot only reads it from
the target repo). To roll out Dependabot, copy the file into each target repo.

---

## 4. Releasing a new version

Consumers pin to tags (or commit SHAs), so cutting a release is how changes
propagate. Releases are **fully automated by [tagpr](https://github.com/Songmu/tagpr)**
(`.github/workflows/tagpr.yml`).

### How it works

1. Every push to `main` triggers `tagpr.yml`
2. tagpr maintains a single open **release PR** (e.g. `Release for v1.2.0`)
   containing the proposed next version + accumulated changelog
3. Merging that PR causes tagpr to **automatically push the corresponding tag**
   (e.g. `v1.2.0`) to `origin`
4. Renovate / Dependabot then bumps `uses:` references in downstream callers

### Controlling the bump kind

Default bump is **patch**. To change it, add a label to a merged PR (or to the
release PR itself) before merging:

| Label | Effect |
|-------|--------|
| `tagpr:patch` (default) | v1.1.0 → v1.1.1 |
| `tagpr:minor` | v1.1.0 → v1.2.0 |
| `tagpr:major` | v1.1.0 → v2.0.0 |

### Manual fallback

If tagpr is unavailable (e.g. App outage), cut a tag manually:

```bash
git tag -a v1.2.0 -m "v1.2.0: <summary>"
git push origin v1.2.0
```

### Pinning policy

**No floating major tag (`v1`) is maintained.** Consumers must pin to either
a concrete release tag (`@v1.2.0`) or a commit SHA. This keeps Renovate's
update graph deterministic and avoids silent breakage on tag movement.

---

## 5. Adding a new reusable workflow

1. Drop the file in `.github/workflows/<name>.yml` with `on: workflow_call`
2. Pin every third-party `uses:` to a commit SHA (`actions-pin-lint` runs on PRs here too)
3. Add the `cicd-sensor/cicd-sensor-action` step as the first step (matches existing workflows)
4. Add a row to the table in §2 above
5. Add the job to `templates/caller-go.yml.template` and/or `templates/caller-node.yml.template`
6. Tag a new release (§4)

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `rulesets-apply` fails with `Resource not accessible by integration` | App missing `Administration: write`, or not installed on the repo | Re-install the App and re-check permissions |
| `rulesets-apply` blocked by its own ruleset | App's actor ID is not in `bypass_actors` | Add it to `security/rulesets/*.json` |
| Caller workflow fails with `workflow was not found` | Tag in `@<tag>` doesn't exist, or repo is private and caller has no access | Push the tag, or make caller use `@main` while bootstrapping |
| `actions-pin-lint` fails in target repo | Unpinned `uses:` line | Replace `@v3` with `@<sha> # v3` |
| Ruleset not appearing on a repo | Repo matched by `exclude:` or not matched by `scope:` | Inspect `baseline.yaml`; trigger `workflow_dispatch` to re-run |
| `files-apply` fails with `refusing to allow a GitHub App to create or update workflow` | App missing `workflows: write` | Add `Workflows: write` to the App permissions and re-install |
| `files-apply` opens no PR for a repo | Primary language not in `files[].languages`, or branch already up-to-date | Inspect `gh repo view <repo> --json primaryLanguage`; check `chore/baseline-sync` branch state |
