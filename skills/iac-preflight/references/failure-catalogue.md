# Failure catalogue

Loaded at Phase 3. Organised by **when the failure surfaces**, because that is
what determines blast radius. A plan-time failure costs a developer ten minutes.
A silent runtime failure costs an incident.

Work from `plan.json`, not the `.tf` source. Replacement, unknown-after-apply
and sensitive-masking are only visible in the JSON.

Decide per section whether the change could have touched it, and say which
sections you skipped.

---

## A. Destructive, and visible in the plan — if you read it properly

The plan says so. Reviewers still miss it, because the destructive line is one
row in a long output and the summary line only counts resources.

**A1 — Replacement of a stateful resource.**
*Signal:* `plan.json` → `resource_changes[].change.actions` containing
`["delete","create"]` or `["create","delete"]`, on a resource that holds data:
databases, storage accounts, Key Vaults, managed disks, Service Bus namespaces,
Container Registries, any resource with a data plane.
*Why:* Terraform reports this as one line among many. `replace_paths` names the
attribute that forced it — an immutable field somebody edited without realising
it was immutable.
*Fix:* Name the attribute, name the data at risk, and offer the two real
options: revert the attribute, or plan an explicit migration. Where the resource
should never be replaceable, recommend `lifecycle { prevent_destroy = true }` —
and note that this converts a silent disaster into a loud apply failure, which
is the trade being made.
**Severity: highest. Rank above everything else in the report.**

**A2 — `count` index shift.**
*Signal:* Several resources of the same type all replacing at once, with
addresses like `[1]`, `[2]`, `[3]`, when the diff merely inserted or removed an
element in a list variable.
*Why:* `count` keys state by position. Inserting at the front re-indexes
everything after it, so Terraform destroys and recreates the lot.
*Fix:* `for_each` over a map or set keys state by identity, not position, and is
immune. For the change in hand, `moved` blocks can re-key without destruction.
This one is worth stating explicitly even when the resources are stateless,
because the churn hits dependents.

**A3 — Deleting a `module` block, or renaming a resource.**
*Signal:* A large `delete` set that the developer describes as "just tidying up"
or "just a rename".
*Why:* Terraform has no notion of rename. A changed address is a destroy plus a
create.
*Fix:* A `moved` block in the same change. If the intent is to stop managing the
resource rather than delete it, that is a `removed` block with
`lifecycle { destroy = false }` on Terraform 1.7+, or `state rm` on older —
never a silent deletion of the config.

**A4 — `prevent_destroy` colliding with a required replacement.**
*Signal:* A resource carrying `prevent_destroy = true` appears in the plan with
a replace action.
*Why:* Terraform errors, but only when it reaches that resource, which can be
well into a long apply. The result is a half-applied environment.
*Fix:* Resolve it before merge. Report the exact address.

---

## B. Real, and invisible in the plan

The dangerous half. A clean plan here is a plan that has been told not to look.

**B1 — `ignore_changes` masking live drift.**
*Signal:* Any `lifecycle { ignore_changes = [...] }` on a resource the diff
touches, and every occurrence of `ignore_changes = all`.
*Why:* The plan is clean **by instruction**. The configuration and the running
resource can differ arbitrarily and nothing will ever say so.
*Fix:* For each ignored attribute, compare the configuration value against the
refreshed state value directly — `terraform state show <address>` — and report
the divergence as a finding in its own right. `ignore_changes = all` means the
resource is effectively unmanaged; say that plainly.

**B2 — `known after apply` on anything that decides behaviour.**
*Signal:* `after_unknown` true in `plan.json` for an attribute feeding a
connection string, hostname, endpoint, principal ID, or a conditional.
*Why:* The value does not exist at plan time, so no review can catch a wrong
one. It becomes real only during apply.
*Fix:* Where the value comes from a `data` source that resolves at apply time,
recommend an explicit `precondition` or `check` block asserting the shape it
must have. Failing loudly at apply beats a malformed endpoint reaching runtime.

**B3 — `sensitive` hiding the diff.**
*Signal:* `after_sensitive` in `plan.json`, or `(sensitive value)` in the human
output.
*Why:* Sensitivity is a display property. The value still changes; the reviewer
just cannot see that it did.
*Fix:* Report *that* a sensitive attribute changed and on which resource — never
the value. Ask the developer to confirm the change was intended.

**B4 — Drift the pipeline will not reconcile.**
*Signal:* Your refreshed plan shows changes nobody wrote, and the applying stage
runs `plan`/`apply` with `-refresh=false`.
*Why:* Someone changed the resource in the portal. Your plan sees it; the
pipeline is configured not to.
*Fix:* Report the drifted attributes. This is a finding about the *environment*
rather than the change, and it is usually the most valuable thing a preflight
produces.

**B5 — Removed implicit dependency.**
*Signal:* A reference replaced by a hardcoded value or a variable — for example
`azurerm_subnet.this.id` becoming a literal ID string.
*Why:* Terraform's graph is built from references. Remove the reference and the
ordering guarantee goes with it. Apply then races, and succeeds most of the
time, which is the worst possible failure rate.
*Fix:* Restore the reference, or add explicit `depends_on`. Note that a change
that "works locally" proves nothing here; ordering bugs are load-dependent.

---

## C. What the pipeline will do differently from your shell

The reason a local plan is not proof. Reconcile each of these against what Phase
2 read from Azure DevOps.

**C1 — Variables supplied by a variable group, not by a file.**
*Signal:* A root variable with a default that the pipeline overrides, or with no
value in any committed `.tfvars`.
*Why:* Your plan used the default. The pipeline will use the group's value. You
planned a configuration that will never be applied.
*Fix:* List every variable whose local value and pipeline value differ or cannot
be compared, and mark each as a gap. A new variable with no matching pipeline
value is a **hard blocker** — the apply will fail, or worse, take a default
nobody chose.

**C2 — The applied plan is not the reviewed plan.**
*Signal:* The release stage runs `terraform apply -auto-approve` without
consuming a plan file produced by the build stage.
*Why:* Apply recomputes its own plan against whatever state exists at that
moment. Whatever the PR reviewer approved was advisory.
*Fix:* Recommend publishing the plan as a pipeline artifact and applying that
exact file. This is the single highest-leverage pipeline change available and is
worth reporting even when the change in hand is safe.

**C3 — Build plans a different commit than release applies.**
*Signal:* The PR validation build plans the merge candidate; the release runs
from `main` after merge, potentially with other merges in between.
*Why:* Two changes that are individually safe can conflict. Neither plan saw the
other.
*Fix:* Note it as a standing property of the pipeline. Where the repository is
busy, recommend the release stage re-plan and gate on a human, rather than
auto-apply.

**C4 — Terraform version skew.**
*Signal:* `required_version`, `.terraform-version` and the pipeline task's
pinned version disagree — or the pipeline pins a floating `latest`.
*Why:* State written by a newer Terraform cannot be read by an older one, and
provider behaviour shifts across minors.
*Fix:* Pin all three to the same version. A floating version in a pipeline is a
future outage with no commit to blame.

---

## D. Pinning

Cheap to check, and the cause of "it worked yesterday" with an empty diff.

**D1 — Provider constraints too loose, or the lockfile not committed.**
*Signal:* `~>` or no constraint in `required_providers`; `.terraform.lock.hcl`
absent from the repository or absent from the diff when providers changed.
*Why:* Without a committed lockfile the pipeline resolves providers afresh and
may take a newer minor than you did. Provider minors change defaults.
*Fix:* Commit `.terraform.lock.hcl`, including the platform hashes the build
agent needs (`terraform providers lock -platform=linux_amd64 ...` for a Linux
agent when you are planning on Windows — a missing platform hash fails `init` on
the agent, which is at least loud).

**D2 — Unpinned module source.**
*Signal:* A `source` pointing at a git ref with no `?ref=<tag-or-sha>`, or a
registry module with no `version`.
*Why:* The pipeline pulls whatever the branch holds at apply time. Your plan and
the apply can use different module code.
*Fix:* Pin to a tag or commit SHA.

---

## E. Azure provider traps that pass plan and fail at apply

Not exhaustive. Check the ones the diff actually touches; add to this list as
the estate teaches you more.

| Trap | What happens |
|---|---|
| Key Vault soft-delete / purge protection | A replaced vault cannot reuse its name until purged, and purge protection makes that impossible for the retention period. Apply fails after the delete |
| Globally unique names | Storage accounts, ACR, and public DNS labels are unique across all of Azure. A generated name that collides fails only at apply |
| Immutable-after-create fields | Subnet delegation, VM zone, AKS network profile, Service Bus tier. Editing one silently converts an update into a replacement — cross-check against section A1 |
| NSG rule priority collisions | Two rules at the same priority; plan is happy, the API rejects it |
| Role assignments with a computed `principal_id` | Resolves at apply. If the principal does not exist yet, apply fails partway |
| `features {}` block on the azurerm provider | Controls destroy semantics such as purging soft-deleted vaults. A change here alters what destroy *means*, with no resource diff at all |
| Private endpoint DNS zone group | Frequently created outside Terraform. The resource applies clean and name resolution fails at runtime |
| Diagnostic settings and locks | Often added by Azure Policy after creation. Terraform then fights the policy on every apply, producing perpetual diffs |

---

## F. Identity and permissions

The house standard names identity mismatch as a standing hazard because it fails
at runtime, on first use, sometimes with a fallback credential masking it.

- Trace every identity name introduced by the change to both its declaration in
  Terraform and its use in application configuration or the pipeline.
- Check the dev service connection's identity actually holds the permission the
  new resource type requires. A new resource type is the usual trigger for a
  first-time permission failure, and it fails mid-apply.
- Where workload identity federation is used, the federated credential's subject
  must match the namespace and service account name exactly. A mismatch
  authenticates as nothing and fails at first call, not at startup.

Anything here that cannot be checked from the repository or the Azure DevOps API
is a gap, phrased as the condition that would close it.
