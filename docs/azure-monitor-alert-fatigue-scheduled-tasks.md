# Azure Monitor alert fatigue scheduled tasks

This guide documents the Azure SRE Agent alert-fatigue pattern from the Microsoft Tech Community article [Azure SRE Agent for Azure Monitor Alerts: Reduce Alert Fatigue, Investigate What Matters](https://techcommunity.microsoft.com/blog/appsonazureblog/azure-sre-agent-for-azure-monitor-alerts-reduce-alert-fatigue-investigate-what-m/4513458).

The article has two setup areas:

1. Intelligent alert handling with Azure Monitor incident response plans, reinvestigation cooldowns, and tiered response plans.
2. Scheduled tasks for recurring alert hygiene and threshold review.

For this repo, **step 1 is a future TODO** because it depends on incident response plans that are not part of the current repo solution yet. This guide implements step 2 only.

## Scope

Create two scheduled tasks:

| Task | Purpose | Recommended cadence |
|---|---|---|
| Weekly alert hygiene report | Find noisy Azure Monitor alert rules, especially rules that fire repeatedly or auto-resolve frequently. | Weekly |
| Monthly threshold audit | Review Azure Monitor alert rule thresholds and recommend safe threshold changes based on observed behavior. | Monthly, after several weekly reports exist |

Terraform is preferred. In practice, scheduled tasks are part of the Azure SRE Agent config directory and are deployed by the Microsoft Terraform backend (`deploy-tf.sh`) or equivalent config deployment. If the backend cannot apply a scheduled task in the current API version, use the SRE Agent data-plane apply step or portal as fallback.

## Prerequisites

- Azure SRE Agent deployed from this repo's `minimal`-based config.
- Azure Monitor connected as an incident source or connector where available.
- Application Insights and Log Analytics connected so the agent can query alert and telemetry history.
- The deployer/current user has `Azure SRE Agent Administrator` on the SRE Agent resource group.
- SRE Agent data-plane token works. If extras are skipped, run `az login --scope "https://azuresre.dev/.default"` and retry.

## Files to add

This repo includes ready-to-edit task definitions:

```text
agent/
  automations/
    scheduled-tasks/
      weekly-alert-hygiene-report.yaml
      monthly-threshold-audit.yaml
```

These follow the Microsoft template shape used by the `azmon-lawappinsights` recipe.

## Task 1: Weekly alert hygiene report

File: `agent/automations/scheduled-tasks/weekly-alert-hygiene-report.yaml`

Default schedule:

```cron
0 8 * * 1
```

This runs Mondays at 08:00 UTC. Change it to your team's preferred reporting time.

Recommended behavior:

- Review Azure Monitor alert activity from the previous 7 days.
- Rank alert rules that fired more than 3 times.
- Identify rules with high auto-resolution rates.
- Identify repeated fires that likely belong in a cooldown/merge strategy.
- Identify alerts with weak signal, missing context, or noisy thresholds.
- Recommend whether each rule should be tuned, suppressed, split into a separate response plan, or left unchanged.
- Produce a report that can be pasted into an operational review.

## Task 2: Monthly threshold audit

File: `agent/automations/scheduled-tasks/monthly-threshold-audit.yaml`

Default schedule:

```cron
0 9 1 * *
```

This runs on the first day of each month at 09:00 UTC. Run this only after weekly hygiene reports have accumulated enough history.

Recommended behavior:

- Audit Azure Monitor alert rules for the agent's monitored subscriptions/resource groups.
- Use the previous 30 days of alert fires and telemetry where available.
- Review current thresholds against observed baselines.
- Identify rules where thresholds are too sensitive, too loose, stale, or missing severity alignment.
- Recommend safe threshold changes, including the evidence and expected reduction in alert volume.
- Avoid recommending threshold changes that would hide critical incidents.
- Output proposed Azure CLI or Terraform follow-up commands only when confidence is high; otherwise mark the recommendation for human review.

## Terraform-preferred deployment

The scheduled tasks live in the agent config directory. Deploy them through the Microsoft Terraform backend so the same config is version-controlled.

From `sreagent-templates`:

```bash
./bin/deploy-tf.sh <path-to-this-repo>/agent/
```

On Windows, if using PowerShell as the shell:

```powershell
bash ./bin/deploy-tf.sh <path-to-this-repo>/agent/
```

If Bash is unavailable locally, run the deployment from Cloud Shell or CI. Keep the Terraform deployment path as the preferred path rather than manually creating tasks in the portal.

## Data-plane fallback

Microsoft's IaC guidance says deployment has an ARM phase and a data-plane phase. Scheduled tasks are represented in the config directory, but some related SRE Agent extras can require data-plane access.

If the Terraform backend reports that the scheduled tasks were skipped:

1. Confirm the deployer/current user has `Azure SRE Agent Administrator` on the SRE Agent resource group.
2. Refresh data-plane auth:

   ```powershell
   az login --scope "https://azuresre.dev/.default"
   ```

3. Rerun the Microsoft apply/deploy flow.
4. If still blocked, create the tasks manually in the SRE Agent portal using the YAML prompt content as the source of truth, then export the agent config back into the repo and compare.

## Portal fallback steps

Use only when IaC/data-plane deployment cannot apply the task.

1. Open the SRE Agent portal.
2. Open the target agent.
3. Select **Scheduled tasks**.
4. Select **Create task**.
5. Enter the task name from the YAML file.
6. Paste the task prompt from the YAML file into task details/instructions.
7. Set the frequency:
   - Weekly alert hygiene report: weekly, Monday, 08:00 UTC or team-preferred time.
   - Monthly threshold audit: monthly, day 1, 09:00 UTC or team-preferred time.
8. Leave response subagent empty unless a dedicated Azure Monitor subagent exists.
9. Use Review mode initially.
10. Create the task.
11. After the first run, open the generated conversation thread and verify the report quality.
12. Export the live agent config and diff it against the repo.

## Future TODO: incident response plans

Do not document this as an implemented repo solution yet.

Future work:

- Connect Azure Monitor as an incident source.
- Define response plans for at least:
  - Critical alerts: cooldown disabled.
  - Operational alerts: cooldown around 6 hours.
  - Low-priority alerts: cooldown around 1 hour.
- Route by severity and alert title keywords.
- Add a PostToolUse hook that nudges Log Analytics queries toward bounded time ranges.
- Add verification steps for cooldown behavior.

## Sources

- [Azure SRE Agent for Azure Monitor Alerts: Reduce Alert Fatigue, Investigate What Matters](https://techcommunity.microsoft.com/blog/appsonazureblog/azure-sre-agent-for-azure-monitor-alerts-reduce-alert-fatigue-investigate-what-m/4513458)
- [Microsoft Tech Community Azure SRE Agent tag page](https://techcommunity.microsoft.com/tag/azure%20sre%20agent)
- [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac)
- [Tutorial: Create and edit scheduled tasks in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/create-scheduled-task)
- [microsoft/sre-agent azmon-lawappinsights scheduled task example](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/recipes/azmon-lawappinsights/automations/scheduled-tasks/daily-health-check.yaml)
