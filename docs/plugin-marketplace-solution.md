# Plugin marketplace solution

Use the Azure SRE Agent Plugin Marketplace to publish **optional, reusable configuration packages** that SRE Agent Administrators can choose to install based on their own agent's requirements.

This is different from the baseline `agent/` config in this repo. Baseline config is automatically deployed because every instance of this repo's Azure SRE Agent should have it. Marketplace plugins are installable add-ons.

## When to use marketplace plugins

Use the marketplace for optional capabilities that are useful, reusable, and not required by every agent.

Good examples:

| Plugin | Why it belongs in the marketplace |
|---|---|
| `sap-hana-incident-triage` | Valuable for SAP teams, unnecessary for agents that do not monitor SAP workloads. |
| `aks-crashloop-triage` | Useful when an agent monitors AKS, but irrelevant for non-Kubernetes environments. |
| `cosmosdb-ru-throttling` | Useful for Cosmos DB workloads only. |
| `sql-deadlock-investigation` | Useful for SQL-backed apps, optional elsewhere. |
| `finops-cost-spike-review` | Useful for cost-focused agents or teams, not mandatory for incident agents. |
| MCP connector bundles | Useful when a team wants an optional tool integration such as CMDB, deployment tracker, or internal runbook API. |

## When not to use marketplace plugins

Keep these in the automatically deployed baseline instead:

| Config | Why it should stay baseline |
|---|---|
| Core Azure Monitor investigation skills | This repo is oriented around Azure SRE Agent and Azure Monitor operations. The agent should always know the common investigation paths. |
| `overview.md` persistent knowledge | It is agent-specific memory, not a portable plugin. |
| Required safety hooks | Guardrails should not depend on optional install decisions. |
| Required scheduled tasks | Recurring operational hygiene for this agent should deploy with the agent. |
| Environment-specific values | Marketplace plugins should not include resource IDs, tenant-specific secrets, or hard-coded production scopes. |
| Secrets | Use Key Vault, connector settings, or CI/CD secrets instead. |

## Decision rule

Ask this question:

> Should every deployment of this repo's Azure SRE Agent always have this capability?

If yes, put it in `agent/` and deploy it automatically.

If no, package it as a marketplace plugin so any SRE Agent Administrator can install it when it matches their environment.

## Recommended repo layout

```text
agent/
  config/
    skills/                  # baseline skills deployed automatically
    hooks/                   # baseline guardrails deployed automatically
  automations/
    scheduled-tasks/         # baseline recurring tasks deployed automatically
  data/
    knowledge/
      overview.md            # baseline persistent knowledge for this agent

plugins/
  sap-hana-incident-triage/
    plugin.json
    skills/
      SKILL.md
  aks-crashloop-triage/
    plugin.json
    skills/
      SKILL.md
  cosmosdb-ru-throttling/
    plugin.json
    skills/
      SKILL.md

.github/
  plugin/
    marketplace.json
```

## Marketplace manifest

Azure SRE Agent supports marketplace manifests in these locations:

- `.github/plugin/marketplace.json`
- `.claude-plugin/marketplace.json`

This repo should use `.github/plugin/marketplace.json` because it aligns with GitHub-hosted distribution.

Example manifest shape:

```json
{
  "name": "Azure SRE Agent Optional Plugins",
  "description": "Optional Azure SRE Agent skills and connector packages for workload-specific operations.",
  "plugins": [
    {
      "name": "sap-hana-incident-triage",
      "description": "Triage SAP HANA incidents using workload-specific investigation steps.",
      "version": "0.1.0",
      "author": "SorcererDude",
      "path": "plugins/sap-hana-incident-triage"
    },
    {
      "name": "aks-crashloop-triage",
      "description": "Investigate AKS CrashLoopBackOff and restart-loop incidents.",
      "version": "0.1.0",
      "author": "SorcererDude",
      "path": "plugins/aks-crashloop-triage"
    }
  ]
}
```

The exact marketplace schema should be validated against the current Azure SRE Agent Plugin Marketplace docs before implementation.

## Plugin package contents

A plugin should be self-contained and portable.

Recommended minimum:

```text
plugins/<plugin-name>/
  plugin.json
  skills/
    SKILL.md
```

For MCP-backed plugins:

```text
plugins/<plugin-name>/
  plugin.json
  skills/
    SKILL.md
  .mcp.json
```

Do not include secrets. If an MCP connector needs credentials, use placeholder environment variables or connector instructions.

## Administrator install flow

Any SRE Agent Administrator can choose to install optional plugins based on their agent's own requirements.

Portal flow:

1. Open `sre.azure.com`.
2. Select the target agent space.
3. Expand **Builder**.
4. Select **Plugins**.
5. Add this repo as a marketplace source using its GitHub `owner/repo` or URL.
6. Browse available plugins.
7. Preview plugin skills before installing.
8. Select only the skills that apply to that agent.
9. Import selected skills.
10. If the plugin includes MCP configuration, use **Add as connector** and complete the connector setup.
11. Verify the installed plugin appears under the Installed tab.
12. Use **Check for updates** later to compare installed content against the repo source.

## Public repository requirement

Based on current Microsoft documentation, the Plugin Marketplace resolves plugins from public GitHub repositories. The SRE Agent GitHub connector is not required for marketplace install.

Use the GitHub connector only when the agent needs to inspect or reason over GitHub repository content during investigations, especially private repositories.

## Versioning guidance

Use semantic versions in plugin metadata:

- Patch: wording fixes, safer prompts, minor clarifications.
- Minor: new investigation steps, new optional skill files, new supported tools.
- Major: behavior changes that could materially alter investigation output or safety posture.

After a plugin changes, SRE Agent Administrators can use marketplace update checks to compare hashes and decide whether to update.

## Future TODO

- Add `.github/plugin/marketplace.json` once the first optional plugin is ready.
- Add a sample `sap-hana-incident-triage` plugin.
- Add a validation checklist for plugin portability.
- Decide whether optional scheduled tasks should be packaged as plugins or remain repo baseline config.

## Sources

- [Plugin Marketplace for Azure SRE Agent: Build Once, Install Anywhere](https://techcommunity.microsoft.com/blog/appsonazureblog/plugin-marketplace-for-azure-sre-agent-build-once-install-anywhere/4513501)
- [Plugin marketplace in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/plugin-marketplace)
- [Tutorial: Install a marketplace plugin in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/install-plugin-from-marketplace)
