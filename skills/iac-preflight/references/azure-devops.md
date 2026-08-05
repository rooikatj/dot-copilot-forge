# Azure DevOps reconciliation

Loaded at Phase 2. How to authenticate, what to read, and what to compare it
against.

This phase exists because the values a local plan uses are usually not the
values the pipeline applies. Variable groups and pipeline variables are
invisible from the repository, so without this phase a perfect-looking plan can
be a plan of a configuration that will never exist.

## Credential rules — these are not negotiable

- **Never ask for a PAT to be pasted into the chat, and never echo one.** A
  token in a transcript is a token in a log.
- **Never write a token into the repository**, into a `.tfvars`, or into any
  file the working tree can see.
- **Never request a token with more scope than reading.** This phase reads.
- If no credential is available, that is a **Tier B run**. Report the gap and
  carry on with plan-only evidence. Do not stall the whole preflight on it.

## Authentication — prefer the token you already have

**First choice — Entra token from the existing `az login`.** No PAT to create,
store, rotate or leak. `499b84ac-1321-427f-aa17-267ca6975798` is the fixed
resource ID for Azure DevOps.

```
az account get-access-token \
  --resource 499b84ac-1321-427f-aa17-267ca6975798 \
  --query accessToken -o tsv
```

Send it as `Authorization: Bearer <token>`. Capture it into a shell variable;
never into a file.

**Fallback — PAT, from the operating system's credential store.** Use this only
where Entra auth is blocked. The PAT needs *Read* on Build, Release, Variable
Groups and Service Connections, and nothing else.

- Windows: store with `cmdkey` or the Credential Manager and read it into the
  environment at the start of the session.
- Linux/macOS: `secret-tool` / Keychain, or an environment variable exported by
  the shell profile.
- `az devops` CLI reads `AZURE_DEVOPS_EXT_PAT` from the environment, which keeps
  it out of the command line and therefore out of shell history.

For raw REST with a PAT, the header is Basic auth over `:<pat>` — an empty
username and the PAT as password. Build it in a variable; do not inline it.

If the credential is missing or rejected, say which and stop this phase
cleanly. A 203 response with an HTML sign-in page rather than JSON means the
token was not accepted — Azure DevOps does not always return 401.

## What to read

Organisation and project come from the repository's remote URL or the pipeline
YAML. API version `7.1` throughout.

| Purpose | Endpoint |
|---|---|
| Variable groups | `GET https://dev.azure.com/{org}/{project}/_apis/distributedtask/variablegroups?api-version=7.1` |
| Pipeline list | `GET https://dev.azure.com/{org}/{project}/_apis/pipelines?api-version=7.1` |
| **Resolved YAML**, templates expanded | `POST https://dev.azure.com/{org}/{project}/_apis/pipelines/{pipelineId}/preview?api-version=7.1` — returns `finalYaml` |
| Runs of a pipeline | `GET https://dev.azure.com/{org}/{project}/_apis/pipelines/{pipelineId}/runs?api-version=7.1` |
| **In-flight builds** (the lock gate) | `GET https://dev.azure.com/{org}/{project}/_apis/build/builds?statusFilter=inProgress&api-version=7.1` |
| Classic release deployments | `GET https://vsrm.dev.azure.com/{org}/{project}/_apis/release/deployments?api-version=7.1` |
| Environments and approval gates | `GET https://dev.azure.com/{org}/{project}/_apis/pipelines/environments?api-version=7.1` |
| Service connections | `GET https://dev.azure.com/{org}/{project}/_apis/serviceendpoint/endpoints?api-version=7.1` |

The `preview` endpoint is the one worth the round trip: it returns the pipeline
**as Azure DevOps resolves it**, with every template expanded. Checking the YAML
fragment in the diff instead of the resolved output is how a template-injected
`-refresh=false` or a substituted Terraform version goes unnoticed.

Secret variables come back with `isSecret: true` and a null value. That is the
API protecting you, not a failure — you need existence and scope, never the
value. Say so in the report rather than treating the null as unreadable.

## The lock gate — do this before any state operation

Query in-flight builds and, on classic pipelines, in-progress deployments.

- **Anything applying to dev is in progress** → stop. Report which run, who
  queued it, and how long it has been going. Do not plan; do not use
  `-lock=false` as a workaround.
- **A run failed partway through an apply** → this is a finding on its own.
  State no longer matches either the old or the new configuration, so your plan
  is against a half-migrated environment and every result needs that caveat.
- **Nothing in flight** → proceed to the plan in `run-sequence.md`.

## What to compare

1. **Every variable the root requires, against every value the applying stage
   supplies.** Build the union of `variable` blocks in the root, then match each
   against pipeline variables, variable-group variables and committed `.tfvars`.
   - Required with no pipeline value and no default → **hard blocker**.
   - Value differs from the one the local plan used → every finding that depends
     on it is provisional. Say which findings those are.
   - New variable introduced by this change → confirm the group was updated in
     the same change, and for *every* environment, not just dev.
2. **Key Vault-linked variable groups.** A group of type `AzureKeyVault` only
   exposes the secret names selected in it. A secret that exists in the vault
   but was not selected resolves to nothing at run time, and the substitution
   often warns rather than fails.
3. **The applying stage's Terraform version and flags**, from the resolved YAML,
   against what you ran locally. Feed the result into catalogue sections C2 and
   C4.
4. **The service connection identity** for the dev stage, against the identity
   names the Terraform declares. Feed into catalogue section F.
5. **Approval gates on the dev environment.** Their absence is not a finding for
   dev, but it tells you how quickly a mistake reaches the environment, which
   belongs in the verdict's framing.

Anything that could not be read becomes a gap in the report, phrased as the
condition that would close it — never rounded up to a pass.
