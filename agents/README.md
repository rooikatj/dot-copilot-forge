# agents/

Custom Copilot CLI agents live here as `*.agent.md` files — one agent per
file, front-matter + instructions, same convention as Copilot CLI's own
custom-agent format.

**Every agent must self-announce when invoked** — open the response with a
one-line self-identification, e.g. `**Agent: <name>** — invoked because
<reason>`. This is because whether Copilot CLI surfaces automatic
invocation in the transcript on its own isn't confirmable, so the agent
has to say so itself.

Empty for now — add agents as they're built and sanitized.
