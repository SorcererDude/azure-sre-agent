# Local prerequisite setup

This guide sets up a Windows workstation for Azure SRE Agent IaC work.

PowerShell is the right fit here because the goal is local tool validation and installation. This does not mean Terraform is deployed through PowerShell as an architecture pattern; Terraform remains the deployment engine for infrastructure.

Validated against Microsoft Learn: [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac), last updated 2026-05-14.

## Required tools

| Tool | Why it is needed |
|---|---|
| Git | Clone this repo and the Microsoft `microsoft/sre-agent` templates. |
| PowerShell 7+ | Windows-friendly local setup and fallback scripts. |
| Azure CLI 2.x+ | Azure login, subscription selection, and Microsoft template scripts. |
| Terraform 1.5+ | Primary IaC engine for the foundation deployment. |
| jq | Required by Microsoft template scripts. |
| Python 3 + PyYAML | Used by Microsoft template tooling. |
| Azure SRE Agent templates | Provides `new-agent`, `deploy-tf`, `export`, `diff`, and `verify` tooling. |

Optional:

| Tool | Why it may be useful |
|---|---|
| Azure Developer CLI (`azd`) | Only needed if testing the azd backend. |
| Bicep CLI | Usually available through Azure CLI; useful if comparing Bicep backend behavior. |

## Permissions

Azure permissions required by Microsoft guidance:

- Owner on the subscription, or
- Contributor plus User Access Administrator.

The deployment identity needs permission to:

- Create resource groups and resources.
- Create role assignments.
- Register required resource providers if they are not already registered.
- Access SRE Agent data-plane APIs for extras such as hooks and knowledge files.

## One-time setup commands

Run in PowerShell 7+.

```powershell
# Check PowerShell version
$PSVersionTable.PSVersion

# Install Git if missing
if (-not (Get-Command git -ErrorAction SilentlyContinue)) {
  winget install --id Git.Git -e --source winget
}

# Install Azure CLI if missing
if (-not (Get-Command az -ErrorAction SilentlyContinue)) {
  winget install --id Microsoft.AzureCLI -e --source winget
}

# Install Terraform if missing
if (-not (Get-Command terraform -ErrorAction SilentlyContinue)) {
  winget install --id Hashicorp.Terraform -e --source winget
}

# Install jq if missing
if (-not (Get-Command jq -ErrorAction SilentlyContinue)) {
  winget install --id jqlang.jq -e --source winget
}

# Install Python if missing
if (-not (Get-Command python -ErrorAction SilentlyContinue)) {
  winget install --id Python.Python.3.12 -e --source winget
}

# Install PyYAML
python -m pip install --upgrade pip
python -m pip install pyyaml
```

Close and reopen PowerShell after installing tools so PATH changes are loaded.

## Validate installed tools

```powershell
git --version
az version
terraform version
jq --version
python --version
python -c "import yaml; print('PyYAML OK')"
```

## Azure login

```powershell
az login
az account set --subscription <subscription-id-or-name>
az account show --query "{name:name,id:id,tenantId:tenantId}" -o table
```

For SRE Agent data-plane operations, Microsoft notes that some scenarios may require an explicit scope login:

```powershell
az login --scope "https://azuresre.dev/.default"
```

Use this when data-plane steps such as hooks, repositories, or knowledge uploads are skipped because a token is unavailable.

## Get Microsoft SRE Agent templates

Use a local tools folder outside this repo, or add it as a documented external dependency. Do not vendor the Microsoft repo unless you intentionally want to maintain a copy.

```powershell
$ToolsRoot = "$HOME\src"
New-Item -ItemType Directory -Force -Path $ToolsRoot | Out-Null
Set-Location $ToolsRoot

if (-not (Test-Path "$ToolsRoot\sre-agent")) {
  git clone https://github.com/microsoft/sre-agent.git
}

Set-Location "$ToolsRoot\sre-agent\sreagent-templates"
```

## Run Microsoft prerequisite check

From the Microsoft `sreagent-templates` directory:

```powershell
.\bin\ps\Install-Prerequisites.ps1 -Check
```

If the check reports missing required tools, run:

```powershell
.\bin\ps\Install-Prerequisites.ps1
```

To include optional tools such as Terraform and Azure Developer CLI where supported by the Microsoft script:

```powershell
.\bin\ps\Install-Prerequisites.ps1 -All
```

## Recommended local folder layout

```text
%USERPROFILE%\src\
  sre-agent\                # Microsoft templates
  azure-sre-agent\          # This repo
```

Keep generated agent config in this repo under `agent/` after it is reviewed and cleaned. Keep temporary exports outside the repo until secrets and sensitive data are removed.

## Minimal recipe bootstrap

From the Microsoft `sreagent-templates` directory:

```powershell
.\bin\ps\New-Agent.ps1 `
  -Recipe minimal `
  -NonInteractive `
  -Set @{
    agentName='sre-<workload>-<env>-<###>'
    resourceGroup='rg-<workload>-<env>-<###>'
    location='<azure-region>'
    targetRGs='rg-<target-workload>-<env>-<###>'
  } `
  -Output <path-to-this-repo>\agent
```

Add a region token to names only when it improves clarity or uniqueness, such as `sre-<workload>-<env>-<region>-<###>` for multi-region deployments.

After bootstrap, add repo-owned config:

- `agent/config/skills/`
- `agent/config/hooks/`
- `agent/automations/scheduled-tasks/`
- `agent/data/knowledge/overview.md`

## Terraform validation

From this repo:

```powershell
terraform -chdir=infra/terraform init
terraform -chdir=infra/terraform validate
terraform -chdir=infra/terraform plan -var-file=environments/dev.tfvars
```

## Troubleshooting

| Symptom | Likely fix |
|---|---|
| `terraform` not recognized | Reopen PowerShell or check PATH after installing Terraform. |
| `az login` works but data-plane extras are skipped | Run `az login --scope "https://azuresre.dev/.default"` and retry extras. |
| Role assignments fail | Confirm the deployment identity has Owner or User Access Administrator. |
| Microsoft scripts cannot find `jq` or Python | Reopen PowerShell and rerun the validation commands. |
| Secrets appear in export files | Stop and redact before committing; use Key Vault or CI/CD secrets instead. |

## Sources

- [Deploy with infrastructure as code in Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/deploy-iac)
- [microsoft/sre-agent templates README](https://github.com/microsoft/sre-agent/blob/main/sreagent-templates/README.md)
- [Terraform install docs](https://developer.hashicorp.com/terraform/install)
- [Azure CLI install docs](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [PowerShell install docs](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell)
