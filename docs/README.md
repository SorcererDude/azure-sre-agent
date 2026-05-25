# Documentation

Start here when working with this repo.

## Implementation runbooks

| Guide | Use it for |
|---|---|
| [Deployment guide](deployment-guide.md) | Building a new Azure SRE Agent environment with Terraform-first IaC and data-plane fallback steps. |
| [Copy/export an Azure SRE Agent](export-copy-agent.md) | Capturing an existing live agent's config, reviewing it, and importing it into repo-managed files. |
| [Local prerequisite setup](local-prerequisites.md) | Installing and validating local tools on Windows with PowerShell. |
| [Azure Monitor alert fatigue scheduled tasks](azure-monitor-alert-fatigue-scheduled-tasks.md) | Configuring weekly alert hygiene and monthly threshold audit scheduled tasks from Microsoft's Azure Monitor alert-fatigue solution. |

## Background

| Guide | Use it for |
|---|---|
| [Azure SRE Agent IaC options](azure-sre-agent-iac-options.md) | Comparing Terraform, Bicep, PowerShell, and Azure Developer CLI for this use case. |

## Current repo decisions

- Terraform is the primary IaC engine.
- PowerShell is used for local setup and data-plane fallback orchestration, not as the Terraform abstraction layer.
- The Microsoft `minimal` recipe is the base.
- `overview.md` is persistent knowledge and should live at `agent/data/knowledge/overview.md`.
- Log Analytics Workspace is attached to Application Insights; the SRE Agent deployment references Application Insights for agent telemetry.
- Azure SRE Agent resources use the repo-local naming prefix `sre` until CAF publishes an official abbreviation.
- Azure Monitor alert-fatigue incident response plans are a future TODO; scheduled tasks are documented and represented as config files now.
