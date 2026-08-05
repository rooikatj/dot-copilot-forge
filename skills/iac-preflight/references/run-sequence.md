# Run sequence

Loaded at Phase 1. The exact order of operations, the commands, and the two
places this can do harm if run carelessly.

## Two hard constraints

**Never take the state lock while a release is applying.** Confirm this via
Azure DevOps first (see `azure-devops.md`). If a run is in flight, stop and
report it. Do not substitute `-lock=false` — that silently downgrades the
evidence tier while producing output that looks identical.

**Never run `terraform force-unlock`, `apply`, `destroy`, `import`, `taint`,
`state rm` or `state mv`.** This skill is read-only against dev. A stale lock is
reported to the developer with the lock ID; releasing it is their call, because
the lock may belong to a run that is still alive.

## 1. Discovery

Local and free. Prefer reading configuration over guessing from directory names.

| Question | How to answer |
|---|---|
| Which Terraform roots exist? | Directories containing a `backend` block or a `provider` block with no parent root above them. A directory of `.tf` files with no backend is usually a module, not a root |
| Which root does the diff touch? | Map each changed `.tf` path up to its nearest root |
| Environments by directory or by workspace? | A `envs/dev`-style directory tree, versus `terraform workspace list`. Both patterns exist; do not assume |
| Which var files apply to dev? | The pipeline YAML is authoritative, not the filename. Find the `-var-file` the applying stage passes |
| Which backend and which state key? | The `backend "azurerm"` block plus any `-backend-config` file or pipeline-supplied values. **A backend whose key comes from a pipeline variable cannot be resolved locally — that is a Tier B gap, report it** |
| Which Terraform version applies? | `required_version`, `.terraform-version`, and the version the pipeline task pins. Record all three; a mismatch is a finding in its own right |

## 2. Authentication and target confirmation

```
az account show -o json
```

Confirm the subscription and tenant are the **dev** ones the applying stage
uses. Planning a dev change while pointed at another subscription produces a
plan full of phantom creations and is the most common way this exercise wastes
an afternoon. If the subscription cannot be matched to the pipeline's service
connection with confidence, say so and treat every resource-existence claim as a
gap.

Terraform authenticates through the same `az login` context by default. If the
root expects `ARM_*` environment variables instead, check they are present — but
never print their values.

## 3. Static checks, cheapest first

```
terraform fmt -check -recursive -diff
terraform init -input=false -backend=true
terraform validate -json
```

Notes that matter:

- `init` may need `-backend-config=<file>` or `-reconfigure`. Prefer
  `-reconfigure` over `-migrate-state`; migration writes.
- `init` upgrades nothing unless asked. **Do not pass `-upgrade`** — it rewrites
  `.terraform.lock.hcl` and turns a read-only preflight into a source change.
- A `validate` failure ends the run. Report it plainly and stop; everything
  downstream would be noise.

## 4. The plan

```
terraform plan -input=false -lock=true -refresh=true -detailed-exitcode \
  -var-file=<the file the pipeline uses> -out="$PLANDIR/tfplan"
terraform show -json "$PLANDIR/tfplan" > "$PLANDIR/plan.json"
```

- `-detailed-exitcode` gives **0** = no changes, **2** = changes present, **1** =
  error. Treat 1 as a full stop. Treat 0 as interesting rather than reassuring —
  see the drift-masking entries in the catalogue.
- `-refresh=true` is what makes this different from reading state. It is the only
  way out-of-band portal changes become visible. If the pipeline applies with
  `-refresh=false`, note the asymmetry: your plan sees drift the pipeline will
  not reconcile.
- Pass the var file the *pipeline* passes, not the one that is convenient. If the
  pipeline supplies variables from a variable group rather than a file, the local
  plan used defaults and every variable-dependent finding is provisional — this
  is the whole reason Phase 2 exists.

### The plan file is a secret

`tfplan` and `plan.json` contain every variable and attribute in **cleartext**,
including values marked `sensitive`. Treat them accordingly:

- Write them to a temporary directory outside the repository. Never inside the
  working tree, where a stray `git add -A` would commit them.
- Never paste raw plan JSON into the report.
- Delete both files when the run ends, including when it ends in an error.

## 5. Optional tools — detect, then run or skip out loud

Each of these is standard in a secops-managed enterprise estate and each is
optional here. Detect by invoking `--version`; if absent, **record the skip as a
gap in the report** rather than staying silent about it.

| Tool | Command | Catches what the plan does not |
|---|---|---|
| `tflint` | `tflint --init` then `tflint --recursive --format=json` | Deprecated arguments, invalid SKUs, invalid regions, unpinned providers. These pass `validate` and fail against the Azure API at **apply** time |
| `checkov` | `checkov -f "$PLANDIR/plan.json" --framework terraform_plan --compact --quiet -o json` | Policy and posture against the resolved plan, so it sees computed values a source scan cannot |
| `trivy` | `trivy config --format json "$PLANDIR/plan.json"` | Same ground as checkov from a second engine; `tfsec` is its predecessor and may still be what the estate has |

**Report only findings the diff introduced.** Run the scanner against the merge
base as well and difference the results, or filter to resources the plan
touches. A pre-existing backlog dumped into a PR review teaches reviewers to
skip the tool, which costs more than it ever saved.

## 6. Cleanup

Delete the temporary plan directory. Confirm the state lock was released — a
completed `plan` releases it, but a killed process does not. If a lock remains,
report the lock ID and who holds it, and stop there.
