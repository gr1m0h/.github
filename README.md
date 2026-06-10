# gr1m0h/.github

`gr1m0h` 配下のリポジトリへ配布する設定の単一ソース。

- **Rulesets** — `security/rulesets/*.json` を `rulesets-apply.yml` が GitHub API で適用
- **Reusable workflows** — `.github/workflows/*.yml`（`workflow_call`）
- **Caller templates** — `templates/caller-*.yml.template` を `files-apply.yml` が対象 repo に PR
- **Community health files** — root の `SECURITY.md` / `CODEOWNERS` / `FUNDING.yml` 等は自動継承

```
.
├── .github/workflows/        # reusable workflows + apply workflows
├── security/
│   ├── baseline.yaml         # 何を どこに 適用するか の唯一の定義
│   └── rulesets/*.json       # ruleset payload
├── templates/                # caller workflow テンプレ
└── (community health files)
```

## 設定の編集

すべて `security/baseline.yaml` を編集して `main` に push するだけ。

- `scope` / `exclude` — rulesets-apply の対象範囲
- `files_apply.repos` — files-apply の allowlist（rulesets より狭く運用）
- `defaults` / `repo_overrides` — 適用する ruleset・file の上書き

## Apply workflows

| Workflow | Trigger | 動作 |
|----------|---------|------|
| `rulesets-apply.yml` | `security/baseline.yaml`, `security/rulesets/**` への push / 手動 | GitHub App トークンで ruleset を PUT/POST |
| `files-apply.yml` | `security/baseline.yaml`, `templates/**` への push / 手動 | `chore/baseline-sync` ブランチに force-push して PR |

App に必要な権限:
- rulesets-apply: `Administration: write`, `Contents: read`, `Metadata: read`
- files-apply: `Contents: write`, `Pull requests: write`, `Workflows: write`

手動実行:
```bash
gh workflow run rulesets-apply.yml -R gr1m0h/.github
gh workflow run files-apply.yml -R gr1m0h/.github
```

## Reusable workflows

| Workflow | 用途 |
|----------|------|
| `codeql-go.yml` / `codeql-typescript.yml` | CodeQL |
| `dependency-review.yml` | PR 依存差分レビュー |
| `gitleaks.yml` | secret scan |
| `govulncheck.yml` | Go 脆弱性 |
| `npm-audit.yml` | npm audit |
| `osv-scanner.yml` | OSV scan（SARIF アップロード）|
| `actions-pin-lint.yml` | `uses:` 未 pin 検出 |

すべての third-party action は SHA pin、`cicd-sensor` ステップを先頭に持つ。

## Release

[tagpr](https://github.com/Songmu/tagpr) (`tagpr.yml`) が release PR を維持し、merge で自動タグ push。

bump 種別は PR ラベルで指定:

| Label | Effect |
|-------|--------|
| `tagpr:patch`（default） | v1.1.0 → v1.1.1 |
| `tagpr:minor` | v1.1.0 → v1.2.0 |
| `tagpr:major` | v1.1.0 → v2.0.0 |

floating major tag (`v1`) は維持しない。consumer は具体的なタグまたは SHA に pin する。

## 新しい reusable workflow を追加するとき

1. `.github/workflows/<name>.yml` に `on: workflow_call` で作成
2. third-party `uses:` は SHA pin（`actions-pin-lint` が落とす）
3. `cicd-sensor/cicd-sensor-action` を先頭に
4. 必要なら `templates/caller-{go,node}.yml.template` に job を追加
5. tagpr の release PR を merge してタグ発行
