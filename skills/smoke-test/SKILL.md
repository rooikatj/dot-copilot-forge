---
name: smoke-test
description: >-
  Two-speed check of what's loaded in this Copilot CLI session. Default
  (light) mode: a table of every skill loaded, grouped by source level
  (user vs. repo-scoped) with a short "best for" line each -- a fast
  capability directory, not a pass/fail check. Full mode ("/smoke-test
  full" or "full smoke test"): rigorous verification -- explicit
  invocation of every skill and agent, checked against the
  self-announcement convention. Trigger on "smoke test", "what skills are
  loaded", "skill directory", "what do I have available", or via
  /smoke-test.
---

# Smoke test

Open your response with `**Skill: smoke-test** — <why it fired>`, per the
pack-wide self-announcement convention (see
[skills/README.md](../README.md)).

Two modes. Default to light unless the request explicitly asks for the
full/rigorous pass.

## Light mode (default)

Answer only from what's already loaded into this session — do not read,
open, grep or search any files to answer, per the same rule
[health-check](../health-check/SKILL.md) follows.

List every skill visible in this session as a table, grouped by source
level:

| Skill | Level | Best for |
|---|---|---|
| *(name)* | user / repo | *(one-line: when to reach for it)* |

- **Level** is `user` (loaded from `~/.copilot/skills/`) or `repo` (loaded
  from `<repo>/.github/skills/`).
- **Best for** is a one-line compression of the skill's own `description`
  frontmatter — not a made-up summary. If a skill's description doesn't
  say when to use it, say so rather than guessing.
- This mode does not fire anything and does not verify self-announcement —
  it's a directory, not a check. For that, use full mode below.

Agents aren't covered in light mode — list skills only. If agents are also
loaded, note the count at the bottom of the table as a nudge toward full
mode, rather than listing them here.

## Full mode

Triggered by `/smoke-test full`, "full smoke test", or a request that
explicitly asks for the rigorous pass.

1. **List what's discoverable.** Answer only from what's already loaded
   into this session — don't read any files. List every skill and every
   custom agent visible, with the source (user vs. repo level) shown
   against each.
2. **Fire each one, deliberately.** For *every* skill and *every* agent in
   that list, invoke it explicitly in turn — name it directly, or use
   `/agent <name>` for agents — with a prompt that plausibly fits its
   stated purpose. Explicit invocation, not trigger words: this checks the
   announcement convention, not whether automatic inference fires.
3. **Record, for each:** did the response's first line match the
   convention — `**Skill: <name>** — <reason>` or
   `**Agent: <name>** — <reason>`? Was the stated reason specific to why
   it fired this time, not vague or copy-pasted?

Report as a short table: name, type (skill/agent), level, self-announced
(yes/no), reason given (specific/vague). **A skill or agent that fires
without announcing itself is the failure this mode exists to catch** — the
case that looks exactly like everything working, right up until you need
the transcript to prove what ran.
