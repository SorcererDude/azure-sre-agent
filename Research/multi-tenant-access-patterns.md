# Multi-tenant access patterns for Azure SRE Agent

Date reviewed: 2026-05-26

## Question

Many customers cite security concerns as the reason Azure Lighthouse is not approved in their environments. This note compares two ways for Azure SRE Agent to investigate resources in remote Azure tenants:

1. **Azure Lighthouse** delegated resource management.
2. **Client service principal plus SRE Agent skill/connector**, where the customer creates a service principal in their tenant and the agent uses a skill or MCP-style connector to query that tenant.

The second pattern is close to the AWS cross-cloud pattern documented by Microsoft: Azure SRE Agent connects to an external cloud through an MCP proxy, the proxy uses credentials supplied as environment variables, and the agent invokes external APIs through that constrained tool surface.

## Executive summary

Azure Lighthouse is the more Azure-native and operationally integrated model when a customer approves delegated resource management. It avoids customer-managed app credentials in the SRE Agent environment and gives a central Azure management plane for multi-tenant operations.

The service-principal-plus-skill model is often easier to approve in security-sensitive customer environments because it avoids Azure Lighthouse delegation, can be scoped to a narrower API/tool surface, and lets the customer own credential lifecycle and revocation. The tradeoff is that it becomes a custom integration pattern: the team must design, secure, audit, rotate, and support the connector/skill.

Recommended stance for this repo:

- Treat **Azure Lighthouse** as the preferred option only where the customer already permits Lighthouse and accepts its delegated-management model.
- Treat **client service principal plus skill/connector** as the preferred alternative when Lighthouse is not approved, especially for read-heavy investigations and narrowly scoped operational workflows.
- Do not present the SP-plus-skill pattern as a way to bypass customer governance. The customer must create, scope, monitor, and be able to revoke the service principal.

## Pattern A: Azure Lighthouse

Azure Lighthouse lets a managing tenant access delegated customer subscriptions or resource groups from a single control plane. In the Azure SRE Agent multi-tenant article, the intended value is that SRE Agent can operate across delegated subscriptions and resource groups without engineers switching directories.

### Benefits

| Benefit | Why it matters |
|---|---|
| Azure-native delegated management | Uses Microsoft's supported cross-tenant management model rather than a custom connector. |
| Central operations plane | Operators can see delegated customer resources through Azure-native management surfaces. |
| No customer SP secret in the agent environment | Access is based on delegation and Azure RBAC, not a connector secret stored with the agent. |
| Familiar MSP/enterprise model | Useful for managed service providers or enterprises with formally delegated tenant operations. |
| Built-in Azure audit trail | Activity occurs through Azure control-plane operations with Azure auditing. |
| Scales to many subscriptions | Good fit when the organization manages many delegated Azure estates centrally. |

### Concerns and limitations

| Concern | Impact |
|---|---|
| Not approved by many security teams | Some customers prohibit persistent delegated cross-tenant administration regardless of Microsoft support. |
| Owner is not supported | Azure Lighthouse does not support delegating Owner. This matters because some SRE Agent onboarding flows require Owner or User Access Administrator to create role assignments. |
| User Access Administrator is limited | UAA is supported only for limited managed-identity role assignment scenarios, not as general-purpose RBAC administration. |
| Roles with `DataActions` are not supported | This limits direct data-plane access through delegated roles. Some built-in roles can still expose data indirectly through actions such as listing keys, which security teams may scrutinize. |
| Custom roles are not supported | Customers cannot always express their exact least-privilege model through Lighthouse. |
| Broad delegation perception | Even when scoped carefully, Lighthouse can look like a standing third-party/admin access path. |
| Customer onboarding burden | The customer tenant must onboard the delegation correctly and maintain it. |
| SRE Agent role-assignment friction | If the agent needs to assign RBAC to its managed identity on delegated resources, Lighthouse role limitations can cause failures. |

## Pattern B: client service principal plus skill/connector

In this model, the customer creates an app registration/service principal in their tenant, assigns it explicit roles on specific scopes, and provides credentials through an approved secret path. Azure SRE Agent uses a skill, connector, or MCP-style proxy to call Azure APIs for that tenant.

This is very close to Microsoft's AWS cross-cloud model: the agent reaches an external environment through a tool proxy, and the proxy authenticates with credentials supplied outside the agent prompt.

### Benefits

| Benefit | Why it matters |
|---|---|
| Avoids Azure Lighthouse | Works for customers whose security policy blocks Lighthouse outright. |
| Customer-owned identity | The remote tenant owns the app registration, credentials, role assignments, rotation, and revocation. |
| Narrower operational surface | The connector can expose only the commands needed for investigation, instead of general Azure portal-style management. |
| Strong tenant isolation story | Each customer tenant can have a separate service principal, secret/certificate, and scope. |
| Easier least-privilege review | Security reviewers can inspect the app permissions, RBAC scopes, and tool code. |
| Similar to MCP cross-cloud pattern | Aligns with the documented AWS integration model: external API access through a constrained tool/proxy. |
| Portable beyond Azure | The same architectural pattern works for AWS, GCP, SaaS observability tools, CMDBs, and internal APIs. |
| Good for read-heavy investigation | Works well when the agent mostly needs logs, metrics, resource metadata, and alert state. |

### Concerns and limitations

| Concern | Impact |
|---|---|
| Credential management burden | Secrets/certificates must be stored, rotated, monitored, and revoked safely. |
| App-only access can bypass user controls | Service principals are not protected by user MFA in the same way as interactive users. Conditional Access for workload identities may be required. |
| Custom code/tool surface | A skill or MCP connector must be built, reviewed, tested, and maintained. Bugs become security issues. |
| Audit identity is the app | Remote tenant audit logs show the service principal, not the individual engineer unless the connector adds its own correlation logging. |
| No native Lighthouse control plane | Operators lose the Azure Lighthouse cross-tenant portal experience. |
| Per-customer setup | Every customer tenant needs its own identity, scopes, credential path, and onboarding runbook. |
| Risk of over-broad app permissions | The service principal can become as risky as Lighthouse if assigned Contributor broadly without a constrained tool surface. |
| Write actions need stronger governance | Remediation workflows should require human approval, scoped permissions, and preferably separate identities for read and write. |

## Security comparison

| Dimension | Azure Lighthouse | Service principal plus skill/connector |
|---|---|---|
| Trust model | Delegated Azure management from managing tenant to customer tenant. | Customer-owned application identity used by a specific connector/tool. |
| Customer perception | Often seen as standing delegated admin access. | Often easier to frame as a scoped integration account. |
| Native Azure support | Strong. | Custom, though built on standard Entra ID, Azure RBAC, and Azure APIs. |
| Least privilege | Limited by Lighthouse role support and lack of custom roles. | Flexible if customer uses narrow scopes and custom roles where appropriate. |
| Secret handling | No customer SP secret required in the agent. | Requires secure credential or federation design. |
| Role limitations | No Owner; limited User Access Administrator; no DataActions roles; no custom roles. | Can use normal Azure RBAC in the customer tenant, including custom roles if allowed. |
| Auditability | Azure control-plane audit from delegated principals. | Azure audit from service principal plus any connector-level logs. |
| Revocation | Remove Lighthouse delegation. | Disable/delete service principal, remove role assignments, revoke credential/federation. |
| Operational scale | Strong for MSP-style multi-tenant Azure operations. | More setup per tenant, but more acceptable in environments blocking Lighthouse. |
| Data-plane fit | Constrained by Lighthouse role support. | Can be designed specifically for read-only data-plane or API access. |

## Recommended design for SP-plus-skill

Use this pattern when a customer will not approve Lighthouse.

### Identity model

Prefer one service principal per customer tenant and per access tier:

- `sre-agent-<customer>-reader` for investigation.
- `sre-agent-<customer>-operator` only if approved remediation is required.

Prefer certificate-based auth or workload identity federation over client secrets when practical. If client secrets are unavoidable, store them in Key Vault and rotate them on a defined schedule.

### RBAC model

Start read-only:

- Reader on approved resource groups or subscriptions.
- Monitoring Reader where metrics/alerts are required.
- Log Analytics Reader on approved workspaces.
- Application Insights access where required.

Add write permissions only for narrowly defined, human-approved remediation. Avoid broad Contributor unless the customer explicitly accepts that risk.

### Connector/tool model

Expose a narrow tool surface, not a general Azure CLI shell.

Good tools:

- List subscriptions/resource groups in approved scope.
- Read Azure Monitor alerts.
- Query Log Analytics with bounded time ranges.
- Read Application Insights failures, requests, traces, dependencies, and exceptions.
- Read resource health and activity logs.
- Summarize recent deployments and configuration changes.

Riskier tools:

- Generic `az rest` execution.
- Arbitrary KQL across all workspaces.
- Role assignment changes.
- Resource delete/update operations.
- Commands that list or retrieve access keys.

### Skill behavior

The skill should:

- Require tenant ID, subscription ID, and approved scopes from configuration, not prompt text.
- Keep query time ranges bounded by default.
- Refuse destructive actions unless a dedicated write connector exists and the agent is in an approved run mode.
- Log correlation IDs, tenant IDs, subscription IDs, and requested tool calls.
- Separate findings from proposed remediation.

## When to choose which pattern

| Situation | Recommended pattern |
|---|---|
| Customer already uses Lighthouse for MSP/central operations | Azure Lighthouse |
| Customer security policy prohibits Lighthouse | Service principal plus skill/connector |
| Need broad Azure portal-style cross-tenant management | Azure Lighthouse |
| Need narrow investigation-only access | Service principal plus skill/connector |
| Need exact custom least-privilege role | Service principal plus skill/connector |
| Need no stored customer app credential | Azure Lighthouse |
| Need cross-cloud consistency with AWS/GCP/SaaS connectors | Service principal plus skill/connector |
| Need many customer tenants with standardized central operations | Azure Lighthouse, if approved |
| Need customer-by-customer opt-in with strong revocation | Service principal plus skill/connector |

## Implementation recommendation for this repo

Document both patterns, but build the SP-plus-skill pattern as the practical fallback for Lighthouse-blocked customers.

Suggested future repo layout:

```text
Research/
  multi-tenant-access-patterns.md

plugins/
  azure-remote-tenant-investigation/
    plugin.json
    skills/
      SKILL.md
    README.md

agent/
  data/
    knowledge/
      remote-tenant-access-model.md
```

The `azure-remote-tenant-investigation` plugin should be optional marketplace content, not baseline config, because some customers will approve Lighthouse, some will approve service principals, and some will approve neither.

## Open questions

- Does Azure SRE Agent expose a first-party Azure connector pattern for remote-tenant app-only authentication, or should this be implemented as a custom MCP server?
- Can workload identity federation be used cleanly from the SRE Agent execution environment for cross-tenant Azure API access?
- What exact RBAC roles are sufficient for the first read-only investigation skill: Reader plus Monitoring Reader plus Log Analytics Reader, or narrower custom roles?
- Should write/remediation be implemented as a separate connector identity from read-only investigation?
- How should connector audit logs be retained and correlated with Azure Activity Logs?

## Sources

- Microsoft Tech Community: [Managing Multi-Tenant Azure Resource with SRE Agent and Lighthouse](https://techcommunity.microsoft.com/blog/appsonazureblog/managing-multi%E2%80%91tenant-azure-resource-with-sre-agent-and-lighthouse/4511789).
- Microsoft Tech Community: [Announcing AWS with Azure SRE Agent: Cross-Cloud Investigation using the brand new AWS DevOps Agent](https://techcommunity.microsoft.com/blog/appsonazureblog/announcing-aws-with-azure-sre-agent-cross-cloud-investigation-using-the-brand-ne/4507413).
- Microsoft Tech Community tag page: [Azure SRE Agent](https://techcommunity.microsoft.com/tag/azure%20sre%20agent).
- Microsoft Learn: [Tenants, users, and roles in Azure Lighthouse scenarios](https://learn.microsoft.com/en-us/azure/lighthouse/concepts/tenants-users-roles).
- Microsoft Learn: [Manage permissions for Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/manage-permissions).
- Microsoft Learn: [User roles and permissions in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/roles-permissions-overview).
- Microsoft Learn: [Connectors in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/connectors).
