# skills/

Each skill is a folder: `skills/<name>/SKILL.md`, optionally with a
`references/` subfolder for supporting docs the skill pulls in.

**Every skill must self-announce when invoked** — open the response with a
one-line self-identification, e.g. `**Skill: <name>** — invoked because
<reason>`. This is because whether Copilot CLI surfaces automatic
invocation in the transcript on its own isn't confirmable, so the skill
has to say so itself.

**Discovery paths:** skills load from `~/.copilot/skills/<name>/SKILL.md`
(user-level) or `<repo>/.github/skills/<name>/SKILL.md` (repo-level), and
appear as slash commands either way.

- [iac-preflight](iac-preflight/SKILL.md) — `/iac-preflight` proves a Terraform
  change against the **dev** environment's real state before merge: locked,
  refreshed plan, reconciled against Azure DevOps variable groups and the
  resolved pipeline, then swept for the failures a green plan hides. Tiered
  evidence — a degraded run never reads as a clean one.
- [health-check](health-check/SKILL.md) — cursory `/health-check` sanity
  ping: what's loaded this session, verdict only.
- [smoke-test](smoke-test/SKILL.md) — `/smoke-test` capability directory
  by default (skills loaded, grouped user vs. repo level, one-line "best
  for" each); `/smoke-test full` runs the full rigorous verification pass.

Otherwise empty for now — add skills as they're built and sanitized.
