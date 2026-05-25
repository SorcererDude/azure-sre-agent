# Copy/export an Azure SRE Agent

Use this guide to capture a live Azure SRE Agent configuration into files that can be reviewed, version-controlled, redeployed, or used as the starting point for another environment.

Validated against Microsoft Learn: [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac), last updated 2026-05-14.

## What export is for

Export is useful when you need to:

- Back up a live Azure SRE Agent configuration.
- Compare the live agent against repo configuration.
- Clone an agent from one environment to another.
- Capture existing skills, hooks, scheduled tasks, connectors, subagents, prompts, and persistent knowledge where supported by the Microsoft tooling.
- Convert portal-created configuration into repo-managed configuration.

## Expected exported structure

The Microsoft templates use a config directory like this:

```text
exported-agent/
  agent.json
  connectors.json
  connectors.secrets.env
  roles.yaml
  config/
    skills/
    subagents/
    hooks/
    common-prompts/
    repos/
  automations/
    scheduled-tasks/
    incident-filters/
    incident-platforms/
  data/
    knowledge/
    synthesized-knowledge/
```

Treat `data/knowledge/overview.md` as persistent knowledge. This is where repo-owned persistent memory should live before it is uploaded by the data-plane apply step.

## Export from a live agent

From the Microsoft `sreagent-templates` directory, use the export tooling.

Bash:

```bash
./bin/export-agent.sh \
  -s <subscription-id> \
  -g <source-resource-group> \
  -n <source-agent-name> \
  -o exported-agent/
```

PowerShell:

```powershell
./bin/ps/Export-Agent.ps1 `
  -SubscriptionId <subscription-id> `
  -ResourceGroupName <source-resource-group> `
  -AgentName <source-agent-name> `
  -OutputPath exported-agent/
```

Use PowerShell when working locally on Windows. The resulting files should be reviewed before they are committed.

## Clone to a new agent

The Microsoft templates also include clone tooling. Clone performs export plus name/resource-group changes for a new target.

Bash:

```bash
./bin/clone-agent.sh \
  --from-agent <source-agent-name> \
  --from-rg <source-resource-group> \
  --set agentName=sre-<workload>-<env>-<region>-<###> \
  --set resourceGroup=rg-<workload>-<env>-<region>-<###> \
  --set location=<azure-region> \
  -o cloned-agent/
```

PowerShell equivalent depends on the current Microsoft template script version. If a dedicated clone script is unavailable, run `Export-Agent.ps1`, edit `agent.json`, then deploy the copied config.

## Review before committing

Do not commit secrets. Review these areas carefully:

| Area | Review action |
|---|---|
| `connectors.secrets.env` | Keep local or store values in a secret manager. Do not commit real secrets. |
| `connectors.json` | Confirm target resource IDs and connector names are correct for the new environment. |
| `roles.yaml` | Confirm target resource groups and scopes are correct. |
| `config/skills/` | Confirm skill names, descriptions, and markdown instructions are environment-neutral. |
| `config/hooks/` | Confirm hooks match the target environment's safety policies. |
| `automations/scheduled-tasks/` | Confirm schedule, timezone expectation, prompt, and enabled state. |
| `data/knowledge/` | Confirm persistent knowledge is intentional and does not contain sensitive incident details. |
| `data/synthesized-knowledge/` | Treat learned/synthesized context as sensitive. Decide case-by-case whether to keep, redact, or omit. |

## Diff against live

After editing exported config, compare it with the live agent.

```bash
./bin/diff-agent.sh <subscription-id> <resource-group> <agent-name> exported-agent/
```

PowerShell:

```powershell
./bin/ps/Diff-Agent.ps1 `
  -SubscriptionId <subscription-id> `
  -ResourceGroupName <resource-group> `
  -AgentName <agent-name> `
  -InputPath exported-agent/
```

## Verify after deploy

Run verification after deploying or cloning.

```bash
./bin/verify-agent.sh <subscription-id> <resource-group> <agent-name> --expected exported-agent/
```

PowerShell:

```powershell
./bin/ps/Verify-Agent.ps1 `
  -SubscriptionId <subscription-id> `
  -ResourceGroupName <resource-group> `
  -AgentName <agent-name> `
  -ExpectedPath exported-agent/
```

Microsoft describes this as a multi-point check for live agent configuration such as connectors, skills, subagents, and hooks.

## Recommended repo import flow

1. Export the live agent to a temporary folder.
2. Review and redact secrets or environment-specific values.
3. Copy stable config into this repo under `agent/`.
4. Place `overview.md` under `agent/data/knowledge/overview.md`.
5. Keep environment-specific values in Terraform `tfvars`, CI variables, or secret stores.
6. Run diff against the live agent.
7. Deploy from the repo to a test agent.
8. Verify the test agent.
9. Promote the config to higher environments.

## Known limits

The Microsoft Learn page currently says these areas are data-plane operations: code repositories, hooks, HTTP triggers, knowledge files, and plugin configurations. Exports should be verified because data-plane features can have auth, token, or API-version constraints.

Secrets should not be expected to export as usable plaintext. Plan to rehydrate secrets from Key Vault or a CI/CD secret store.

## Sources

- [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac)
- [microsoft/sre-agent templates README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/README.md)
- [minimal recipe README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/recipes/minimal/README.md)
