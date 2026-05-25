# Azure SRE Agent IaC options

Date reviewed: 2026-05-26

## Summary

For the requested workflow, use **Terraform as the primary IaC layer** and reuse the Microsoft Azure SRE Agent template config directory for agent-specific configuration. Terraform is the strongest fit because the workflow is not only "create an SRE Agent"; it also requires CAF naming, a Key Vault, a user-assigned managed identity, RBAC, Application Insights, Log Analytics, and repeatable day-2 updates.

The Microsoft templates support four backends from the same generated config directory:

| Option | Fit for this workflow | Notes |
|---|---:|---|
| Terraform | Strongest | Best fit for CAF naming, state, resource dependencies, Key Vault, RBAC, and enterprise CI/CD. Uses `azapi` for `Microsoft.App/agents`; still needs Microsoft data-plane scripts or equivalent API calls for extras. |
| Bicep | Strong | Best Azure-native option and the official default. Good if the team wants ARM-native deployments without Terraform state. Add a wrapper/module for Key Vault and CAF naming. |
| PowerShell | Moderate | Useful on Windows and for bootstrap/prototyping. Not ideal as the long-term source of truth for named infrastructure and drift control. |
| Azure Developer CLI (`azd`) | Moderate | Great for quick app-like environments and demos. Less suitable as the authoritative production deployment path when CAF naming, existing platform conventions, and separate operational resources matter. |

## Workflow fit

| Step | Terraform | Bicep | PowerShell | azd |
|---|---|---|---|---|
| 1. Create resource group with CAF name | Native | Native | Scripted | Via env/template |
| 2. Create Key Vault with CAF name | Native | Native | Scripted | Needs template extension |
| 3. Create user-assigned managed identity and assign Key Vault Secrets User | Native, strong dependency handling | Native, strong dependency handling | Scripted | Needs template extension |
| 4. Create Application Insights with CAF name | Native | Native | Scripted | Template/env-driven |
| 5. Create Log Analytics Workspace with CAF name | Native | Native | Scripted | Template/env-driven |
| 6. Create Azure SRE Agent with UAMI and App Insights | Supported through official Terraform backend / `azapi` | Official default | Supported PowerShell wrapper | Supported through `azd up` |
| 7. Upload/configure skills | Config directory; apply via template/data-plane | Config directory; apply via template/data-plane | Config directory | Config directory |
| 8. Update `overview.md` persistent memory | Data-plane knowledge upload | Data-plane knowledge upload | Data-plane script/API | Data-plane script/API |
| 9. Configure agent hooks | Data-plane today | Data-plane today | Script/API | Script/API |
| 10. Configure scheduled tasks | Config directory; verify after deploy | Config directory; verify after deploy | Config directory | Config directory |
| 11. Start one thread with a prompt | Post-deploy API/CLI action | Post-deploy API/CLI action | Natural fit as script | Post-deploy hook/script |

## Recommendation

Use a two-layer deployment:

1. **Foundation IaC in Terraform**
   - Generate CAF-compliant names for:
     - `rg-...` resource group
     - `kv-...` Key Vault
     - `id-...` user-assigned managed identity
     - `appi-...` Application Insights
     - `log-...` Log Analytics workspace
     - `sre-...` Azure SRE Agent
   - Create the Key Vault and RBAC assignment for `Key Vault Secrets User`.
   - Create or pass through the managed identity, Log Analytics workspace, and Application Insights resources.
   - Deploy the SRE Agent using the Microsoft Terraform backend or a thin Terraform module around the generated config.

2. **Agent config/data-plane layer**
   - Store skills under `config/skills/`.
   - Store hooks under `config/hooks/`.
   - Store scheduled tasks under `automations/scheduled-tasks/`.
   - Store persistent knowledge such as `overview.md` under `data/knowledge/` or the repo's chosen source folder, then upload it during the post-deploy phase.
   - Start the initial thread as an explicit post-deploy step, after the agent endpoint is available.

This keeps infrastructure declarative while acknowledging that several SRE Agent capabilities are still data-plane operations. Terraform should be the deployment engine for the foundation layer; PowerShell is optional as a convenience wrapper for Windows users or post-deploy data-plane calls, not a requirement for running Terraform.

## Important findings from the Microsoft templates

- The official IaC page says deployments happen in two phases:
  - ARM infrastructure phase: resource group, UAMI, Log Analytics, Application Insights, SRE Agent, RBAC, connectors, skills, subagents, and tools.
  - Data-plane phase: code repositories, hooks, HTTP triggers, knowledge files, and plugin configurations.
- The same generated config directory can be deployed through Bicep, Terraform, PowerShell, or `azd`.
- The Azure Monitor recipe `azmon-lawappinsights` is the closest starting point because it already targets Application Insights and Log Analytics and includes skills, hooks, and a scheduled task.
- The `minimal` recipe is a better starting point if the repo should define everything from first principles and avoid sample operational behavior.
- The templates include export, clone, diff, and verify scripts. The verify step matters for this workflow because data-plane pieces can be skipped when the deployment identity lacks the SRE Agent data-plane token.

## CAF naming considerations

Microsoft CAF naming guidance uses resource-type abbreviations plus workload, environment, region, and instance tokens. The relevant CAF abbreviations are:

| Resource | CAF abbreviation |
|---|---|
| Resource group | `rg` |
| Key Vault | `kv` |
| User-assigned managed identity | `id` |
| Application Insights | `appi` |
| Log Analytics workspace | `log` |

Azure SRE Agent is new enough that the CAF abbreviation table might not yet include a dedicated abbreviation for `Microsoft.App/agents`. Until there is an official abbreviation, use the repo-local convention `sre-<workload>-<env>-<region>-<###>` and document that exception.

## Implementation shape

Recommended repo layout:

```text
infra/
  terraform/
    main.tf
    variables.tf
    outputs.tf
    naming.tf
agent/
  agent.json
  connectors.json
  roles.yaml
  config/
    skills/
    hooks/
    common-prompts/
    subagents/
  automations/
    scheduled-tasks/
  data/
    knowledge/
      overview.md
scripts/
  apply-agent-config.ps1
  start-initial-thread.ps1
docs/
  azure-sre-agent-iac-options.md
```

Suggested deployment order:

1. Run Terraform directly from local development or CI/CD to create the resource group, Key Vault, identity, RBAC, Log Analytics, Application Insights, and SRE Agent.
2. Run the Microsoft template `apply-extras` flow or equivalent API calls for knowledge files, hooks, and any data-plane-only configuration.
3. Verify the live agent configuration.
4. Start the first thread with the required prompt as a final post-deploy action.

PowerShell can host steps 2-4 if that is the most ergonomic local experience, but the Terraform layer should remain plain Terraform so it can also run cleanly in GitHub Actions, Azure DevOps, or any other CI runner.

## Open items to validate before implementation

- Confirm whether the current SRE Agent ARM API supports supplying an existing Log Analytics workspace for agent telemetry in the same way it supports existing managed identity and existing agent Application Insights.
- Confirm the current data-plane endpoint/API for creating the initial thread and whether it requires `Microsoft.App/agents/threads/write`.
- Decide whether `overview.md` should be treated as persistent knowledge (`data/knowledge/overview.md`) or as a common prompt/system instruction. For "persistent memory", knowledge upload is the safer initial assumption.
- Decide whether the repo should start from `azmon-lawappinsights` or `minimal`. For this workflow, `minimal` plus explicit repo-owned skills/hooks/tasks is cleaner; `azmon-lawappinsights` is faster if its defaults are acceptable.

## Sources

- Microsoft Learn: [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac), last updated 2026-05-14.
- Microsoft GitHub: [microsoft/sre-agent `sreagent-templates` README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/README.md).
- Microsoft GitHub: [azmon-lawappinsights recipe README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/recipes/azmon-lawappinsights/README.md).
- Microsoft GitHub: [minimal recipe README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/recipes/minimal/README.md).
- Microsoft Learn: [Azure SRE Agent overview](https://learn.microsoft.com/en-us/azure/sre-agent/overview).
- Microsoft Learn: [Agent hooks in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/agent-hooks).
- Microsoft Learn: [Azure resource naming guidance](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming).
- Microsoft Learn: [Azure resource abbreviation recommendations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations).
