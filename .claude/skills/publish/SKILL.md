---
name: publish
description: Sanitize-and-push workflow for this repo. Use before pushing any change to master — this repo is public with no PR review gate, so this skill is the review gate. Fires on "publish", "push to forge", "ship this", or before any git push in this repo.
---

# Publish (double-confirm, sanitize-then-push)

This repo (`dot-copilot-forge`) is public. `master` is the only branch and
it is what the public sees directly — there is no pull request step to
catch a leaked internal detail before it's visible. This skill is that
missing gate. Content bugs and typos are NOT what this gate is for — only
genericness/sanitization is. Do not use this skill to judge code quality.

## Steps

1. **Show the change.** Run `git status` and `git diff` (staged + unstaged)
   for everything about to be committed. Summarize what changed in plain
   language.

2. **Scan.**
   - Run `gitleaks detect --source . --no-banner -v` over the repo and
     surface any findings verbatim.
   - Grep the changed files for smells gitleaks won't catch: a real personal
     name, an employer name, internal hostnames/domains (anything not
     `example.com`/`localhost`/generic), IP addresses, or `<PLACEHOLDER>`
     markers that have been filled in with what looks like real data rather
     than a template value.
   - Report findings plainly. If nothing is found, say so — don't invent
     findings to seem thorough.

3. **First confirm — genericness.** Ask the user directly: does this change
   read as fully generic/sanitized? Wait for an explicit yes. If they say
   no, or want to fix something, stop here — do not proceed to commit or
   push.

4. **Second confirm — the push itself.** Separately, ask: confirm push to
   `origin master` on the public `dot-copilot-forge` repo? This is a
   distinct confirmation from step 3 — "this content is generic" and "yes,
   actually publish it" are different questions, and both need an explicit
   yes.

5. **Commit and push.** Only after both confirms:
   - `git add` the reviewed files specifically (not a blanket `git add -A`
     — add what was actually reviewed).
   - Commit with a message describing the change.
   - `git push origin master`.
   - Report the result, including if the local pre-push gitleaks hook or
     GitHub's push protection rejects the push — that is the safety net
     working as intended, not a bug to work around.

## What NOT to do

- Never skip step 2 (scan) because a change "looks small."
- Never treat a single yes as covering both confirms in steps 3 and 4 — ask
  them separately.
- Never use `--no-verify` to bypass the pre-push hook.
