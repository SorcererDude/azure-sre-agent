# Azure SRE Agent

This repo captures deployment and operational runbooks for Azure SRE Agent using infrastructure as code and the Microsoft SRE Agent template tooling.

## Solutions

1. [Deployment guide](docs/deployment-guide.md)
   - Uses the requested workflow.
   - Terraform owns the Azure foundation resources wherever possible.
   - PowerShell is reserved for local orchestration or data-plane gaps that Terraform cannot safely own yet.

2. [Copy/export an existing Azure SRE Agent](docs/export-copy-agent.md)
   - Exports a live agent into a version-controlled config directory.
   - Captures existing configuration such as skills, hooks, scheduled tasks, persistent knowledge, connectors, subagents, and prompts where supported by the Microsoft tooling.

3. [Local prerequisite setup](docs/local-prerequisites.md)
   - Windows/PowerShell-first setup guide.
   - Installs or validates tools such as Git, Azure CLI, Terraform, PowerShell 7, jq, and access to the Microsoft SRE Agent templates.

4. [Azure Monitor alert fatigue scheduled tasks](docs/azure-monitor-alert-fatigue-scheduled-tasks.md)
   - Documents the scheduled-task portion of Microsoft's Azure Monitor alert-fatigue solution.
   - Adds weekly alert hygiene and monthly threshold audit task definitions under `agent/automations/scheduled-tasks/`.
   - Tracks incident response plan configuration as a future TODO.

## Design decisions

- Use the Microsoft `minimal` recipe as the base and add repo-owned skills, hooks, scheduled tasks, and persistent knowledge explicitly.
- Treat `overview.md` as persistent knowledge under `data/knowledge/overview.md`.
- Attach Log Analytics Workspace to Application Insights. The SRE Agent deployment should reference Application Insights for agent telemetry, not the workspace directly unless the current API requires it.
- Assign the deployer/current user `Azure SRE Agent Administrator` on the SRE Agent resource group so data-plane controls can apply successfully.
- Use `sre-<workload>-<env>-<###>` as the default repo-local Azure SRE Agent naming convention until CAF publishes an official abbreviation for `Microsoft.App/agents`.
- Use a region token only when it adds value, such as multi-region deployments, globally unique names, or resources where CAF examples include region.

## Reference

See [Azure SRE Agent IaC options](docs/azure-sre-agent-iac-options.md) for the evaluation that led to these solution guides.
