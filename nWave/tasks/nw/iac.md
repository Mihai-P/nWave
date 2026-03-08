---
description: "Reverse-engineers existing Azure infrastructure into Terraform IaC using ARM REST API. TDD loop: terraform plan no-changes is the green bar."
argument-hint: '"resource-group-name" --subscription {subscription-id} --tenant {tenant-id}'
---

# NW-IAC: Azure IaC Reverse-Engineering

**Wave**: IAC | **Agent**: Main Instance (orchestrator) | **Command**: `/nw:iac "{resource-group}" --subscription {sub} --tenant {tenant}`

## Overview

Orchestrates full IaC reverse-engineering: existing Azure resource group → clean Terraform with verified `terraform plan` showing no changes. Delegates all implementation to `@nw-iac-crafter` via Task tool.

## CRITICAL BOUNDARY RULES

1. **NEVER implement steps directly.** All ARM calls, Terraform operations, and HCL writing MUST be delegated to `@nw-iac-crafter`.
2. **You are ORCHESTRATOR** — parse input, dispatch agent, verify output artifacts, report completion.
3. **Verify the green bar yourself** after the agent completes: run `terraform plan -detailed-exitcode` and confirm exit 0.

## Orchestration Flow

```
INPUT: "{resource-group}" --subscription {sub} --tenant {tenant}
  |
  1. Parse and validate input
     resource-group: required
     subscription:   required (UUID format)
     tenant:         required (UUID format)
     Derive output-dir: docs/iac/{resource-group}/
  |
  2. Read rigor profile from .nwave/des-config.json (key: rigor)
     If absent: use defaults
     Relevant keys: agent_model | iac_max_drift_iterations (default: 5) | max_turns (default: 60)
  |
  3. Create output directory
     mkdir -p docs/iac/{resource-group}/
  |
  4. Dispatch @nw-iac-crafter
     Task(
       subagent_type="nw-iac-crafter",
       model=rigor.agent_model,   # omit if "inherit"
       prompt=<IaC Task Template below>,
     )
  |
  5. Verify output artifacts exist after agent completes
     docs/iac/{resource-group}/manifest.json       — resource inventory
     docs/iac/{resource-group}/dependency-graph.json — import order
     docs/iac/{resource-group}/terraform/           — HCL files
     docs/iac/{resource-group}/import-log.json      — per-resource outcome
  |
  6. Verify green bar
     cd docs/iac/{resource-group}/terraform
     terraform plan -detailed-exitcode
     exit 0 → SUCCESS
     exit 2 → FAIL — surface last plan output, do not mark complete
     exit 1 → ERROR — surface error output
  |
  7. Report completion
     Summary: resources imported | resources stuck | secrets injected | manual inputs required
     List any STUCK_{resource} entries from import-log.json for manual follow-up
```

## IaC Task Template

Pass this verbatim to the agent. Fill `{placeholders}` from parsed input.

```
# IAC_TASK

## Target
Resource Group: {resource-group}
Subscription:   {subscription-id}
Tenant:         {tenant-id}
Output Dir:     docs/iac/{resource-group}/

## Rigor
max_drift_iterations: {iac_max_drift_iterations}

## Instructions
Execute the full IaC reverse-engineering workflow as defined in your agent specification.
Work autonomously. Do not ask clarifying questions — surface blockers in import-log.json.

## Boundary Rules
- Write all Terraform files under docs/iac/{resource-group}/terraform/
- Write manifest.json, dependency-graph.json, import-log.json to docs/iac/{resource-group}/
- Do not modify any existing project files outside docs/iac/{resource-group}/
- terraform.tfvars is gitignored — add to .gitignore if not already present
```

## Resume

If agent times out or fails mid-workflow:
- Check `docs/iac/{resource-group}/import-log.json` for last completed resource
- Re-dispatch agent — it reads import-log.json on start and resumes from first non-PASS resource

## Output Artifacts

```
docs/iac/{resource-group}/
  manifest.json           — ARM inventory (resource type, name, ID, dependencies)
  dependency-graph.json   — topologically sorted import order
  import-log.json         — per-resource outcome (PASS | STUCK | SKIPPED)
  terraform/
    provider.tf           — azurerm provider + backend config
    variables.tf          — input variables including secret references
    terraform.tfvars      — secret values (gitignored)
    main.tf               — all imported HCL resource blocks
    .terraform/           — provider cache (gitignored)
    terraform.tfstate     — local state (gitignored for sensitive envs)
```

## Success Criteria

- [ ] manifest.json created with full resource inventory
- [ ] dependency-graph.json produced with topological order
- [ ] All resources attempted (PASS, STUCK, or SKIPPED — none missing)
- [ ] `terraform plan -detailed-exitcode` exits 0
- [ ] STUCK resources documented with last plan output for manual follow-up
- [ ] terraform.tfvars and .terraform/ excluded from git
