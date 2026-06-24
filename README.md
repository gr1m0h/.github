# gr1m0h/.github

Single source of truth for the configuration distributed across repositories
under `gr1m0h`.

- **Rulesets** — `rulesets-apply.yml` applies `security/rulesets/*.json` via the GitHub API
- **Reusable workflows** — `.github/workflows/*.yml` (`workflow_call`)
- **Caller templates** — `files-apply.yml` PRs `templates/caller-*.yml.template` into target repos
- **CODEOWNERS** — not inherited from the org `.github`, so `files-apply.yml` distributes `templates/CODEOWNERS.template` as a real file into each repo (centralizes reviewer assignment, no GitHub Actions required)
- **Community health files** — root `SECURITY.md` / `FUNDING.yml` etc. are inherited automatically

```
.
├── .github/workflows/        # reusable workflows + apply workflows
├── security/
│   ├── baseline.yaml         # the single definition of what is applied where
│   └── rulesets/*.json       # ruleset payloads
├── templates/                # caller workflow templates
└── (community health files)
```

## Editing the configuration

Everything is driven by editing `security/baseline.yaml` and pushing to `main`.

- `scope` / `exclude` — coverage of rulesets-apply
- `files_apply.repos` — allowlist for files-apply (run narrower than rulesets)
- `defaults` / `repo_overrides` — override the rulesets and files that are applied

## Apply workflows

| Workflow | Trigger | Behavior |
|----------|---------|----------|
| `rulesets-apply.yml` | push to `security/baseline.yaml`, `security/rulesets/**` / manual | PUT/POST rulesets with a GitHub App token |
| `files-apply.yml` | push to `security/baseline.yaml`, `templates/**` / manual | append a commit to the `chore/baseline-sync` branch (no force-push) and open a PR |

Permissions required on the App:
- rulesets-apply: `Administration: write`, `Contents: read`, `Metadata: read`
- files-apply: `Contents: write`, `Pull requests: write`, `Workflows: write`

Manual runs:
```bash
gh workflow run rulesets-apply.yml -R gr1m0h/.github
gh workflow run files-apply.yml -R gr1m0h/.github
```

## Reusable workflows

| Workflow | Purpose |
|----------|---------|
| `codeql-go.yml` / `codeql-typescript.yml` | CodeQL |
| `dependency-review.yml` | PR dependency-diff review |
| `gitleaks.yml` | secret scan |
| `govulncheck.yml` | Go vulnerability scan |
| `npm-audit.yml` | npm audit |
| `osv-scanner.yml` | OSV scan (SARIF upload) |
| `actions-pin-lint.yml` | detect unpinned `uses:` refs |

Every third-party action is SHA-pinned and runs the `cicd-sensor` step first.

## Release

[tagpr](https://github.com/Songmu/tagpr) (`tagpr.yml`) maintains a release PR;
merging it pushes a tag automatically.

The bump kind is selected by PR labels:

| Label | Effect |
|-------|--------|
| `tagpr:patch` (default) | v1.1.0 → v1.1.1 |
| `tagpr:minor` | v1.1.0 → v1.2.0 |
| `tagpr:major` | v1.1.0 → v2.0.0 |

A floating major tag (`v1`) is not maintained; consumers pin to a concrete tag
or SHA.

## Adding a new reusable workflow

1. Create `.github/workflows/<name>.yml` with `on: workflow_call`
2. SHA-pin third-party `uses:` (`actions-pin-lint` fails otherwise)
3. Put `cicd-sensor/cicd-sensor-action` first
4. If needed, add the job to `templates/caller-{go,node}.yml.template`
5. Merge the tagpr release PR to cut a tag
