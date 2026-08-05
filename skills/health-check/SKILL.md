---
name: health-check
version: 0.0.1
description: >-
  Quick, cursory sanity check of this Copilot home — confirms skills and
  agents are actually loaded in the current session, without the full
  verbatim-quote rigor of the smoke-test skill's full mode. Use when you
  want a fast "is everything here" ping rather than a full verification
  pass. Trigger on "health check", "sanity check", "is everything loaded",
  "quick check", "ping", or via /health-check.
---

# Health check

Open your response with `**Skill: health-check** — <why it fired>`, per
the pack-wide self-announcement convention (see
[skills/README.md](../README.md)).

**This is a cursory check** — fast, low-rigor, meant to be run often. It
only looks at what's already loaded into *this session*.

For a directory of what's loaded with a one-line "best for" each, see
[smoke-test](../smoke-test/SKILL.md) (`/smoke-test`). For full rigorous
verification (verbatim quotes, file-reads forbidden), use
`/smoke-test full` instead.

## What to report

Answer only from what is already loaded into this session — do not read,
open, grep or search any files to answer. If something is not loaded, say
so plainly rather than going to disk to check.

1. **Skills.** List every skill visible in this session, with the source
   shown against each (`(copilot)` vs. vendor).
2. **Agents.** List every custom agent visible in this session, with
   source.
3. **One line per category, verdict only** — OK, or a specific gap (e.g.
   "3 skills, not 4 — `smoke-test` missing").

Keep the whole report short — this is a ping, not an audit. If everything
looks right, say so in one line per category and stop; don't pad it out.

## If something's missing

Don't guess why. Point at `/smoke-test full` as the next step for a
rigorous check.
