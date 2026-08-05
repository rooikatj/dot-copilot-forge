# Changelog

All notable changes to this repository are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
as interpreted in [Versioning](README.md#versioning) — this repo ships prompts,
not code, so "breaking" means *behaviour a copied artifact no longer performs*.

## [Unreleased]

Nothing yet.

## [0.0.1] — 2026-08-05

First versioned state. Everything below already existed; this release is the
point at which it became addressable by a version number.

### Added

- **Versioning scheme.** `VERSION` at the repo root, a `version:` key in every
  artifact's frontmatter, this changelog, and a `v*` git tag per release. Every
  artifact starts at `0.0.1` alongside the repo.
- **`skills/iac-preflight`** — proves a Terraform change against the dev
  environment's real state before merge. Refreshed, locked plan against the dev
  backend, gated on no in-flight release run; reconciled against Azure DevOps
  variable groups, pipeline variables and the resolved pipeline YAML; then swept
  for the failures a green plan hides. Reports one of four evidence tiers, so a
  degraded run cannot be mistaken for a clean one.
- **`skills/health-check`** — cursory `/health-check` sanity ping: what is
  loaded this session, verdict only.
- **`skills/smoke-test`** — `/smoke-test` capability directory by default;
  `/smoke-test full` runs the rigorous verification pass.
- **`.claude/skills/publish`** — the sanitize-and-double-confirm gate for this
  repo. `master` is public with no pull request step, so this skill is the
  missing review gate.
- **Repository scaffolding** — `README.md`, and `agents/`, `prompts/` and
  `skills/` each with their own README.

### Known limitations

- `iac-preflight` has never been run against a real Terraform repository. Its
  logic is untested against real plan JSON, a real backend, or a real Azure
  DevOps variable group.
- `agents/` and `prompts/` are documented but empty.

[Unreleased]: https://github.com/rooikatj/dot-copilot-forge/compare/v0.0.1...HEAD
[0.0.1]: https://github.com/rooikatj/dot-copilot-forge/releases/tag/v0.0.1
