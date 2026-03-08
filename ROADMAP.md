# nWave IAC Agent Roadmap

## Vision

Build a specialized nWave agent that reverse-engineers existing Azure infrastructure into Terraform IaC — using a TDD loop where `terraform plan` showing **no changes** is the green bar.

---

## The Core Insight

TDD's loop is not specific to code:

1. **Define what "done" looks like before you start** (the test)
2. **Work until that definition is satisfied**
3. **Verify nothing else broke**

For IaC reverse-engineering:

| TDD Concept | IaC Equivalent |
|-------------|----------------|
| Failing test | `terraform plan` shows drift (changes pending) |
| Green bar | `terraform plan` exits with no changes |
| Regression | `terraform plan` re-introduces drift after refactor |
| Commit | Clean plan locked into source control |

---

## Problem Statement

Azure resources exist without IaC. No Terraform state, no HCL. The resource group was created manually (via portal, CLI, or ad-hoc script). Goal: produce Terraform that **fully describes** the existing resource group and all its resources, verified by a clean `terraform plan`.

---

## Agent Design

### Name: `nw-iac-crafter`

### Authentication

All Azure API calls use the ARM REST API directly (`management.azure.com`), authenticated via:
```
POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token
scope: https://management.azure.com/.default
```
Bearer token injected into all subsequent ARM calls. No `az cli` dependency.

### ARM REST API vs az CLI

`az cli` is a wrapper around ARM REST API — same data, same limitations. Using ARM directly gives:
- No CLI installation dependency
- Full control over pagination, retries, and response parsing
- Consistent auth model across all resource types

### Secret Retrieval Strategy

| Secret type | ARM endpoint | Retrievable? |
|-------------|-------------|--------------|
| Storage account keys | `POST .../storageAccounts/{name}/listKeys` | ✅ Yes |
| Redis cache keys | `POST .../redis/{name}/listKeys` | ✅ Yes |
| Service Bus connection strings | `POST .../authorizationRules/{name}/listKeys` | ✅ Yes |
| Event Hub connection strings | `POST .../authorizationRules/{name}/listKeys` | ✅ Yes |
| Key Vault secret values | Key Vault data plane (`vault.azure.net`) | ✅ Yes (with access) |
| SQL admin password | Not stored by Azure after creation | ❌ No |
| VM admin password | Not stored by Azure after creation | ❌ No |
| AKS service principal secret | Not stored by Azure after creation | ❌ No |

For retrievable secrets: inject value into `terraform.tfvars` (gitignored), reference via variable in HCL.
For non-retrievable secrets: use `lifecycle { ignore_changes = [...] }` + document as manual input.

### Workflow

```
INPUT: Azure resource group name + subscription ID + tenant ID
  |
  1. AUTH — obtain ARM bearer token
     POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
     Store token for all subsequent ARM calls
  |
  2. DISCOVER — inventory all resources via ARM
     GET https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}/resources
     For each resource: GET .../providers/{type}/{name}?api-version=...
     Build manifest: resource type, name, ID, location, tags, properties
     Retrieve retrievable secrets via listKeys/listSecrets endpoints
  |
  3. PLAN — build dependency graph
     Parse ARM properties to extract inter-resource references (subnet → VNet, NIC → subnet, etc.)
     Produce a topologically sorted import order
     Flag non-importable resources and non-retrievable secrets upfront
     Output: ordered manifest with dependency edges + secret handling strategy per resource

     Common dependency chains:
       VNet → Subnet → NIC → VM
       VNet → Subnet → NSG (association)
       Key Vault → Key Vault Secret/Key/Certificate
       App Service Plan → App Service / Function App
       Storage Account → Blob Container / Queue / Table

  |
  4. PREPARE — scaffold Terraform project
     provider.tf      → azurerm provider + backend config
     variables.tf     → subscription_id, resource_group_name, location + one variable per secret
     terraform.tfvars → secret values retrieved from ARM (gitignored)
     main.tf          → empty, filled resource by resource in GREEN loop
     terraform init
  |
  5. RED — establish baseline drift
     terraform plan -detailed-exitcode → expect exit 2 (changes pending, nothing declared yet)
     Record: this is the failing state we must eliminate
  |
  6. GREEN loop — one resource at a time, in dependency order
     For each resource in topologically sorted manifest:
       a. terraform import {resource_type}.{name} {azure_resource_id}
       b. terraform show -json → inspect imported state
       c. Write HCL block into main.tf:
            - Use resource references (azurerm_vnet.x.id) not hardcoded ARM IDs
            - Inject variable refs for secrets (var.storage_primary_key)
            - Add lifecycle { ignore_changes = [...] } for non-retrievable secrets
       d. terraform plan -detailed-exitcode → progress check
            exit 0 → resource clean, continue to next
            exit 2 → drift remains, adjust HCL → repeat from (d)
            exit 1 → surface error, stop

       Loop-break controls (three layers):
         Layer 1 — per-resource drift iteration cap (application-level)
           max_drift_iterations = rigor.iac_max_drift_iterations (default: 5)
           On cap reached → log STUCK_{resource} to execution-log.json
                          → surface last plan output to user
                          → stop entire GREEN loop (do not proceed to next resource)

         Layer 2 — TurnCounter (DES domain)
           Agent increments TurnCounter each iteration
           TurnCounter.is_limit_exceeded(phase="GREEN", max_turns=N) → clean exit
           Allows graceful stop vs hard cutoff

         Layer 3 — max_turns on Task invocation (platform-level hard ceiling)
           Task(max_turns=30) → Claude Code enforces unconditionally
           subagent-stop hook fires → orchestrator sees incomplete GREEN phase
           → orchestrator re-dispatches or surfaces to user

     True green bar: exit 0 after ALL resources processed
  |
  7. COMMIT — full clean plan is the gate
     terraform plan -detailed-exitcode → exit 0 = PASS
     git commit -m "feat(iac): import {resource-group} into Terraform"
```

### DES Integration

The "test" is a CLI command with a deterministic exit condition:

```bash
terraform plan -detailed-exitcode
# exit 0 = no changes (GREEN)
# exit 1 = error
# exit 2 = changes pending (RED)
```

DES tracks phase progression the same way it does for code TDD — the agent logs each phase, the `subagent-stop` hook verifies the GREEN phase produced exit code 0 before marking the step complete.

---

## Scope (v1)

**In scope:**
- Azure resource groups (single RG, single subscription)
- Terraform (HCL) only
- `azurerm` provider
- Resources: resource group, common types (VNet, NSG, Storage Account, App Service, Key Vault)

**Out of scope (future):**
- Multi-subscription / management group scope
- Pulumi, Bicep, CDK
- Modules and remote state migration
- Drift detection on already-managed resources

---

## Open Questions

- [ ] How to handle resources that `terraform import` does not support? (some Azure types have no import support)
- [ ] Backend config — local state for discovery, remote for final output?
- [ ] Non-retrievable secrets (VM/SQL passwords) — require manual input; how to prompt user and wire into tfvars?
- [ ] Dependency graph — how to detect cross-resource references from ARM properties reliably? (ARM responses use IDs, not logical names — need to map IDs back to manifest entries)
- [ ] Circular dependencies — rare in Azure but possible (e.g. NSG ↔ Subnet association); how to break cycles?
- [ ] Tags — should tags be variables or literals in HCL?
- [ ] API versions — ARM requires explicit api-version per resource type; need a lookup map or discovery mechanism
- [ ] Key Vault data plane auth — separate endpoint (`{vault}.vault.azure.net`), needs additional scope in token request
- [ ] What should happen when a resource hits `max_drift_iterations`? Stop everything, skip and continue, or flag for manual fix and proceed?
- [ ] Should `iac_max_drift_iterations` live in `des-config.json` rigor profile or as a separate IAC-specific config?

---

## Success Criteria

- [ ] Agent can take a resource group name and produce valid Terraform
- [ ] `terraform plan` exits with no changes after agent completes
- [ ] Output is committed, reviewable HCL (not generated noise)
- [ ] Works without manual intervention for common resource types
- [ ] DES verifies the clean plan before marking delivery complete
