---
name: iac-preflight
description: >-
  Prove a Terraform change is safe to merge by testing it against the dev
  environment's real state, not just reading it. Runs a locked, refreshed plan
  against the dev backend, reconciles the plan against pipeline variables and
  variable groups, and sweeps for the failure modes a green plan hides —
  replacement of stateful resources, count/for_each index shifts, ignore_changes
  masking real drift, unpinned providers and modules, and values the pipeline
  will supply differently from your shell. Use before raising or merging a pull
  request that touches Terraform, tfvars, backend configuration, provider
  versions, module sources, or the build/release pipeline that applies them.
  Trigger on "is this terraform safe", "check against dev", "will this break
  dev", "preflight", "check my infra change", "what will this actually do",
  "did I miss anything before merging", and on any change under a Terraform
  root or to a pipeline that runs terraform.
---

# IaC preflight

**Skill: iac-preflight** — say this on the first line of the response, with a
one-line reason for firing. Copilot CLI does not reliably announce automatic
invocation, so the skill announces itself.

A Terraform plan that exits clean tells you the configuration parses and the
provider accepted it. It does not tell you what will happen when the release
pipeline applies it — a different commit, a different variable set, possibly a
different provider version, certainly a different identity. This skill closes
that gap by grounding every claim in the **dev environment's actual state**, and
by refusing to claim more than the evidence it managed to gather.

Three rules govern everything below:

1. **Report findings; don't fix them.** Recommend the change, show the exact
   edit where it is short, and leave the decision with the developer.
2. **"Not verifiable" is a finding, not a pass.** If the state, a variable
   group or a pipeline definition could not be read, that is reported as a gap
   with the specific thing that would need to be true — never rounded up to
   "looks fine".
3. **Honour the repository's own rules.** Read them first (below) and let them
   override anything here. A house convention beats a general best practice
   every time.

## Cost discipline

Infrastructure changes slowly. This skill is invoked rarely and must earn every
token it spends.

- Load `references/*.md` **only when the phase that needs it begins**. Never
  read all three up front.
- Phase 0 can end the run. If the diff touches no Terraform, tfvars, lockfile,
  module source or applying pipeline, say so in one line and stop.
- Read files the diff actually implicates. Do not walk the whole Terraform root
  to build a mental model of infrastructure that did not change.

## Phase 0 — Scope and house rules

Cheap, local, no network. Do this before anything else.

1. **House rules.** Read whichever exist: `.github/copilot-instructions.md`,
   `AGENTS.md`, `CONTRIBUTING.md`, `.github/instructions/*.instructions.md`,
   and any `README.md` inside the Terraform root. Note any rule that contradicts
   this skill and follow the repository's.
2. **The diff.** `git diff --name-status origin/<default-branch>...HEAD`.
   Classify what changed: Terraform config, variable values, backend, provider
   versions, `.terraform.lock.hcl`, module sources, pipeline YAML, or nothing
   relevant.
3. **Decide whether to continue.** If nothing relevant changed, report that in
   one line and stop — a preflight on an unrelated change is pure cost.
4. **State the plan of record.** Before spending anything, tell the developer
   which Terraform root you intend to plan, which environment it targets, and
   what evidence you will try to gather. A wrong root caught here is free.

## Phase 1 — Evidence, in tiers

Load `references/run-sequence.md` now. It carries the exact commands, the
discovery order for roots and backends, and the lock-safety gate.

Evidence comes in tiers and **the achieved tier is reported in the verdict**. A
degraded run must never read like a clean one.

| Tier | Evidence | What it licenses you to claim |
|---|---|---|
| **A** | Refreshed, locked `plan -json` against the dev backend, plus pipeline variables read from Azure DevOps | Everything in this skill |
| **B** | Plan against dev, but pipeline variables unread | Plan-visible findings only; every variable-dependent claim becomes a gap |
| **C** | `state pull` / `show -json`, no refresh | Config-versus-state divergence only. Blind to out-of-band drift |
| **D** | Static configuration analysis, no state | Cannot answer "is this safe against dev". Say so first, not last |

**The lock gate is not optional.** Before any operation that takes the state
lock, confirm no dev release is mid-apply. A plan taken during someone else's
apply is misleading, and a lock taken during theirs is an incident. If a run is
in flight, stop and say so — do not fall back to `-lock=false` and quietly
downgrade the evidence.

## Phase 2 — Azure DevOps reconciliation

Load `references/azure-devops.md` when this phase begins. It carries
authentication (prefer the Entra token from your existing `az login`; PAT is the
documented fallback and is never stored in the repository), the REST calls, and
what to compare.

This phase exists because of one fact: **the values your local plan used are
usually not the values the pipeline will apply.** Variable groups and pipeline
variables are invisible from the repository, so a local plan that looks perfect
can be a plan of a configuration that will never exist.

Read — never write, never print secret values:

- **Variable groups and pipeline variables** linked to the applying stage —
  existence, scope and whether each is secret. Compare against every variable
  the root requires.
- **Pipeline and environment definitions** as Azure DevOps resolved them,
  including templates and approval gates — so you check the real pipeline, not
  the YAML fragment in the diff.
- **Recent runs of the dev pipeline** — a partially failed apply leaves state
  that no longer matches either the old or the new configuration.
- **The service connection and identity** the dev stage runs as, for the
  identity-mismatch checks.

## Phase 3 — The sweep

Load `references/failure-catalogue.md` now. Work the catalogue against the
plan JSON and the evidence gathered, not against the raw `.tf` files — the
JSON is where replacement, unknown-after-apply and sensitive-masking are
actually visible.

For each catalogue section decide first whether this change *could* have touched
it. Most changes touch a handful. **Say which sections you skipped and why**, so
the developer can disagree with the skip rather than discover it.

Where the repository also carries the `silent-failure-check` skill, this one
covers the Terraform and pipeline surface; defer to that one for the
application-side hazards (unreplaced tokens in app config, generated client
drift, duplicated code across deployables) rather than duplicating them here.

## Output

One consolidated report to the terminal. Nothing written to disk — no artifact
to gitignore, nothing to leak.

Write for a reviewer who did not run the tool. Plain professional English, no
Terraform jargon left unexplained, no wall of raw plan output.

**1. Verdict, first, in one line.** Safe to merge / not safe to merge / safe
only if — followed by the evidence tier and, when the tier is below A, the
specific thing that was not verifiable.

**2. What this change actually does.** Two or three sentences of plain
language: resources added, changed, replaced, destroyed, and which of those
hold data.

**3. Findings, ranked by blast radius**, not by ease of fix. Each one:

> **Finding — <short title>** · *Severity* · *Where*
> What will happen, and when it will surface — plan time, apply time, or
> silently at runtime.
> **Why it happens** — one or two sentences of mechanism.
> **Recommended fix** — the concrete change, with the exact snippet or command
> where it is short enough to be useful.

**4. Gaps.** Everything that could not be verified, each phrased as the
condition that would close it: "not verifiable — needs X". Never omit this
section; when there are no gaps, say there are none.

**5. Checked and clear.** A compact list, one line each, of the catalogue
sections that were examined and found sound, and the ones deliberately skipped
with the reason. This is what makes a clean verdict trustworthy rather than
merely brief.

Close with the single most important thing to do next. If the verdict is "safe
only if", that sentence is the *if*.
