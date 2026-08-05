# dot-copilot-forge

A friendly bunch of GitHub Copilot CLI artifacts — skills, custom agents,
and standalone paste-in prompts. Public and sanitized: no employer names,
no internal hostnames, no real ticket IDs — just structure, ready to be
copied wherever Copilot CLI needs it.

**Never trust these blindly.** Read a skill, agent, or prompt before
relying on it in a real session — nothing here is pre-vetted for your
environment, and a description that sounds right isn't the same as
behavior you've checked.

## Layout

| Path | Holds |
|---|---|
| `agents/` | Custom Copilot CLI agents (`*.agent.md`) |
| `skills/` | Skills (`SKILL.md` + `references/`) |
| `prompts/` | Standalone paste-in prompts — on-demand, not skills, zero standing context cost |
| `VERSION` | The repo's current version, one line |
| `CHANGELOG.md` | What changed, per version |

See each folder's own README for what it actually contains.

## Versioning

Two levels, because the two questions are different. *"Which forge is this?"* is
answered by `VERSION` and the matching `v*` git tag. *"Which version of this one
skill did I copy onto a work machine?"* is answered by the `version:` key in the
artifact's own frontmatter — the only one of the two that survives being copied
out of the repo.

| Carries it | Answers |
|---|---|
| `VERSION` + git tag `v0.0.1` | The repository as a whole. The tag also gives GitHub a Releases entry, and therefore a stable no-login ZIP URL per version |
| `version:` in each `SKILL.md` / `*.agent.md` | That artifact alone, and travels with the file |
| `CHANGELOG.md` | The narrative — what changed since you last copied something |

`version:` is not a key Copilot CLI reads. Unknown frontmatter keys are ignored,
so it is inert metadata for humans, deliberately.

### What the numbers mean here

This repo ships prompts, not code, so semantic versioning needs translating.
Read it as *what a person who copied this artifact would notice*:

- **MAJOR** — an artifact stops doing something someone relied on, or is renamed
  or removed. Their existing invocation now behaves differently.
- **MINOR** — a new artifact, or a new capability in an existing one. Nothing
  they had stops working.
- **PATCH** — clarifications, wording, corrected commands. Same behaviour, said
  better.

### The rule that keeps this honest

**Bump the artifact's `version:` in the same commit that changes the artifact.**
A version number updated later, or in a separate tidy-up pass, is worse than no
version number at all — it tells people the file is unchanged when it is not.
The repo version and `CHANGELOG.md` are updated at release time, when the tag is
cut; the per-artifact version is updated as the work happens.

Artifact versions move independently of each other and of the repo version. An
artifact at `0.3.0` inside a repo tagged `v0.1.0` is normal, not a mistake.
