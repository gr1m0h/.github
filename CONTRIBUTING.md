# Contributing

Thanks for your interest in contributing to a project under `gr1m0h`. This
guide is inherited org-wide from the `.github` repository; individual repos may
add their own `CONTRIBUTING.md` with project-specific details.

## Before you start

- Search existing issues and pull requests first — your topic may already be tracked.
- For anything non-trivial, open an issue to align on direction before sending a PR. It saves everyone time.
- By contributing, you agree your contributions are licensed under the repository's license (MIT unless stated otherwise).

## Pull requests

1. Fork and create a topic branch from the default branch.
2. Keep changes focused; one logical change per PR.
3. Match the surrounding code style and add tests where the repo has them.
4. Make sure the repo's CI passes locally where possible (lint, tests, build).
5. Write a clear PR description: what changed and why.

## CI and security expectations

These repos enforce a shared baseline (see [`gr1m0h/.github`](https://github.com/gr1m0h/.github)):

- All GitHub Actions `uses:` refs must be pinned to a full-length commit SHA.
- Security workflows (CodeQL, govulncheck/OSV, gitleaks, dependency review) run on PRs.

If CI flags an unpinned action or a vulnerability, fix it in the same PR.

## Reporting bugs and security issues

- **Bugs / features** — open an issue using the provided templates.
- **Security vulnerabilities** — do not open a public issue; follow [`SECURITY.md`](./SECURITY.md).

## Code of Conduct

By participating, you agree to abide by the [Code of Conduct](./CODE_OF_CONDUCT.md).
