# Deployment guide

This guide implements the requested Azure SRE Agent deployment workflow.

Primary approach: **Terraform for Azure infrastructure and SRE Agent deployment**, with **PowerShell only as a fallback** for local orchestration or data-plane actions that Terraform/ARM cannot currently perform.

Validated against Microsoft Learn: [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac), last updated 2026-05-14.

## Target workflow

1. Create a new resource group using CAF naming.
2. Create a new Key Vault using CAF naming.
3. Create a new user-assigned managed identity using CAF naming, then assign it `Key Vault Secrets User` on the Key Vault.
4. Create Application Insights using CAF naming.
5. Create a Log Analytics Workspace for Application Insights using CAF naming.
6. Create Azure SRE Agent using CAF naming, in the new resource group, with the user-assigned managed identity and Application Insights.
7. Upload and configure skills.
8. Upload `overview.md` as persistent knowledge.
9. Configure agent hooks.
10. Configure scheduled tasks.
11. Start one thread with a specific prompt.

## Deployment boundary

Terraform should own these resources:

| Step | Terraform ownership |
|---|---|
| Resource group | Yes |
| Key Vault | Yes |
| User-assigned managed identity | Yes |
| Key Vault Secrets User role assignment | Yes |
| Log Analytics Workspace | Yes |
| Application Insights | Yes, connected to the workspace |
| Azure SRE Agent | Yes, using the Microsoft Terraform backend or `azapi` resource support |
| Skills/config exposed as ARM sub-resources | Prefer Terraform/Microsoft template backend |
| Scheduled tasks exposed through config deployment | Prefer Microsoft template backend |
| Persistent knowledge upload | Data-plane fallback |
| Hooks | Data-plane fallback per current Microsoft Learn guidance |
| Initial thread | Post-deploy action |

Microsoft's IaC page says deployment currently happens in two phases:

1. ARM infrastructure phase: resource group, managed identity, Log Analytics, Application Insights, SRE Agent, RBAC, connectors, skills, subagents, and tools.
2. Data-plane phase: code repositories, hooks, HTTP triggers, knowledge files, and plugin configurations.

Because of that split, avoid forcing everything into Terraform with brittle `local-exec` blocks. Use Terraform for the foundation and use the official `apply-extras` flow or a small PowerShell wrapper for data-plane work.

## Naming convention

Use CAF-style abbreviations and a consistent token order. CAF lists region as a useful naming component, but not a universal requirement. The default convention in this repo omits region to keep names shorter; add a region token when it improves clarity or uniqueness, especially for multi-region deployments.

Default patterns:

| Resource | Pattern |
|---|---|
| Resource group | `rg-<workload>-<env>-<###>` |
| Key Vault | `kv-<workload>-<env>-<###>` |
| User-assigned managed identity | `id-<workload>-<env>-<###>` |
| Log Analytics workspace | `log-<workload>-<env>-<###>` |
| Application Insights | `appi-<workload>-<env>-<###>` |
| Azure SRE Agent | `sre-<workload>-<env>-<###>` |

Optional region-aware patterns:

| Resource | Pattern |
|---|---|
| Resource group | `rg-<workload>-<env>-<region>-<###>` |
| Key Vault | `kv-<workload>-<env>-<region>-<###>` |
| User-assigned managed identity | `id-<workload>-<env>-<region>-<###>` |
| Log Analytics workspace | `log-<workload>-<env>-<region>-<###>` |
| Application Insights | `appi-<workload>-<env>-<region>-<###>` |
| Azure SRE Agent | `sre-<workload>-<env>-<region>-<###>` |

Key Vault names have stricter global naming limits. The Terraform naming module should shorten or omit tokens for Key Vault if required.

## Recommended repo layout

```text
infra/
  terraform/
    main.tf
    providers.tf
    variables.tf
    locals.tf
    naming.tf
    outputs.tf
    environments/
      dev.tfvars
      prod.tfvars
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
```

## Terraform variable model

Do not put environment values directly into `variables.tf`.

Use this split:

| File | Purpose |
|---|---|
| `variables.tf` | Declares variable names, types, defaults, validations, and descriptions. |
| `environments/dev.tfvars` | Stores non-secret dev values. |
| `environments/prod.tfvars` | Stores non-secret prod values. |
| `locals.tf` / `naming.tf` | Derives CAF names from workload, environment, optional region abbreviation, and instance. |
| CI/CD secret variables | Stores secrets and sensitive runtime values. |
| `outputs.tf` | Exposes resource IDs needed by agent config and post-deploy scripts. |

Example `dev.tfvars`:

```hcl
workload     = "azsre"
environment  = "dev"
location     = "australiaeast"
instance     = "001"
include_region_in_name = false
# region_abbr = "aue" # enable only when include_region_in_name is true
target_resource_group_names = ["rg-example-app-dev-001"]
```

## Deployment commands

Run Terraform directly. PowerShell may be the shell you type into, but Terraform is still the deployment engine.

```powershell
terraform -chdir=infra/terraform init
terraform -chdir=infra/terraform plan -var-file=environments/dev.tfvars
terraform -chdir=infra/terraform apply -var-file=environments/dev.tfvars
```

The same commands can run in GitHub Actions, Azure DevOps, or Bash.

## Agent configuration flow

Start from the Microsoft `minimal` recipe because this repo should own the operational behavior explicitly.

Recommended flow:

1. Generate a minimal agent config directory from the Microsoft templates.
2. Commit repo-owned skills under `agent/config/skills/`.
3. Commit repo-owned hooks under `agent/config/hooks/`.
4. Commit scheduled tasks under `agent/automations/scheduled-tasks/`.
5. Commit persistent knowledge, including `overview.md`, under `agent/data/knowledge/`.
6. Deploy infrastructure and ARM-supported agent config with Terraform/Microsoft template backend.
7. Apply data-plane extras using the Microsoft `apply-extras` flow or `scripts/apply-agent-config.ps1`.
8. Verify the live agent.
9. Start the initial thread using `scripts/start-initial-thread.ps1` or the current SRE Agent API/CLI endpoint.

## Fallback PowerShell responsibilities

PowerShell should be used only where it adds value:

- Install or validate local prerequisites.
- Run Microsoft template scripts on Windows.
- Upload persistent knowledge if Terraform/ARM cannot do it.
- Apply hooks if they remain data-plane-only.
- Start the first thread after deployment.

PowerShell should not hide Terraform behind a custom deployment framework. Keep Terraform commands plain and CI-friendly.

## Validation checklist

- `terraform plan` shows expected RG, Key Vault, UAMI, LAW, App Insights, RBAC, and SRE Agent changes.
- Application Insights is connected to the Log Analytics Workspace.
- SRE Agent uses the user-assigned managed identity.
- Managed identity has `Key Vault Secrets User` on the Key Vault.
- `overview.md` exists in persistent knowledge and is visible to the agent after apply-extras.
- Skills are present and callable.
- Hooks are present.
- Scheduled tasks are present.
- Initial thread starts with the expected prompt.
- Microsoft verify tooling passes for the live agent.

## Sources

- [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac)
- [microsoft/sre-agent templates README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/README.md)
- [minimal recipe README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/recipes/minimal/README.md)
- [Azure resource naming guidance](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure resource abbreviation recommendations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
