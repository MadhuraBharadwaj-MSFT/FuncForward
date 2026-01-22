---
name: Functions Upgrade
description: Upgrade Azure Functions to run on the latest and greatest lang versions, bundles, programming model, and the recommended SKU. Use this skill to evaluate and upgrade Function Apps
---

# Azure Functions Upgrade Skill

This skill scans Azure Function Apps from **any source** (GitHub repos, Azure-hosted apps, or local workspace) and generates a comprehensive upgrade report with actionable recommendations. It can apply upgrades via code changes, Azure CLI, or submit a Pull Request.

---

## Supported Sources

| Source | Detection Method | Upgrade Method |
|--------|-----------------|----------------|
| **GitHub Repository** | Fetch files via raw URLs or clone | Create PR with changes |
| **Azure-Hosted Function App** | Query via Azure MCP tools / Azure CLI | Azure CLI commands or Bicep deployment |
| **Local Workspace** | Scan files directly | Direct file modifications |
| **Azure DevOps Repo** | Fetch via Azure DevOps API | Create PR via Azure DevOps CLI |

### Source Detection Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     SOURCE DETECTION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User Input Analysis:                                            │
│                                                                  │
│  ├── URL starts with "github.com" or "https://github.com"       │
│  │   └── GitHub Repository → Use GitHub API/raw files           │
│  │                                                               │
│  ├── URL starts with "dev.azure.com" or "azure.com/devops"      │
│  │   └── Azure DevOps Repo → Use Azure DevOps API               │
│  │                                                               │
│  ├── User mentions Function App name or resource group          │
│  │   └── Azure-Hosted → Use Azure MCP tools to query            │
│  │                                                               │
│  ├── User mentions "my workspace" or "current project"          │
│  │   └── Local Workspace → Scan files in current directory      │
│  │                                                               │
│  └── User provides Azure resource ID                             │
│      └── Azure-Hosted → Parse and query via Azure API           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Report Output

> ⚠️ **IMPORTANT**: Always create a `scan-report.md` file in the user's workspace root directory to store the upgrade report.
> 
> The report file should be named `scan-report.md` and contain:
> - Scan results summary table
> - List of files scanned with findings
> - Prioritized recommendations (High/Medium/Low)
> - Code examples for migrations
> - Upgrade confirmation prompt with options
> - Reference links to official documentation

---

## Workspace Structure for Upgrades

When applying upgrades to a GitHub or Azure DevOps repository, clone the target repo into the `upgrades/` folder using **date-based naming**:

```
FuncForward/                              ← Main workspace
├── .github/
│   └── skills/SKILL.md
├── .gitignore                            ← Contains: upgrades/
├── GOAL.md
├── scan-report.md                        ← Generated scan report
└── upgrades/                             ← Git-ignored folder for cloned repos
    └── {repo-name}-{YYYYMMDD}/           ← Date-based folder name
        ├── scan-report.md                ← Copy of report (included in PR)
        ├── host.json                     ← Modified files
        ├── package.json
        ├── src/functions/                ← v4 structure
        └── ...
```

### Folder Naming Convention

Use date-based naming: `{repo-name}-{YYYYMMDD}`

Examples:
- `OutdatedFunctionApp-20260122`
- `my-function-app-20260122`

If same repo scanned multiple times on same day, append time: `{repo-name}-{YYYYMMDD}-{HHMM}`

### Include Scan Report in PR

When creating a PR, **copy the `scan-report.md` into the cloned repository root** so it:
1. Becomes part of the commit history
2. Can be used as the PR description
3. Provides permanent documentation of what was changed and why

---

## Scanning Capabilities

### 1. Language Runtime Version

Scan and detect the current language runtime version:

| Language | Detection Source |
|----------|-----------------|
| **Node.js** | `package.json` → `engines.node`, Function App configuration |
| **Python** | `.python-version`, `requirements.txt`, Function App configuration |
| **.NET** | `.csproj` → `TargetFramework`, Function App configuration |
| **Java** | `pom.xml` → `java.version`, Function App configuration |
| **PowerShell** | Function App configuration |

> 📖 **Latest Supported Versions**: Always fetch the current supported versions from the official documentation:
> 
> **[Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)**
> 
> This page contains up-to-date tables for each language with:
> - Supported runtime versions
> - GA vs Preview status
> - End-of-support dates
> - OS compatibility (Linux/Windows)

**Files to inspect:**
- `host.json` - runtime configuration
- `local.settings.json` - local runtime settings
- Language-specific project files (`package.json`, `.csproj`, `requirements.txt`, `pom.xml`)

---

### 2. Extension Bundles Version

Scan the `host.json` file for the current extension bundles configuration:

```json
{
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

**Recommended Version:** `[4.*, 5.0.0)`

**Check for:**
- Missing extension bundles configuration
- Outdated bundle versions (e.g., `[2.*, 3.0.0)` or `[3.*, 4.0.0)`)
- Manual SDK references that should be replaced with bundles

---

### 3. Programming Model Version

Detect the programming model in use:

| Language | Legacy Model | Latest Model | Detection Method |
|----------|-------------|--------------|------------------|
| **JavaScript/TypeScript** | v3 (function.json) | **v4** (code-based) | Check for `function.json` files vs `@azure/functions` v4 patterns |
| **Python** | v1 (function.json) | **v2** (decorators) | Check for `function.json` files vs `@app.` decorator patterns |
| **.NET** | In-process | **Isolated process** | Check `.csproj` for worker references |
| **Java** | v1 | v2 | Check annotations and dependencies |

**Key Indicators:**
- Presence of `function.json` files (legacy for Node.js/Python)
- Import patterns in code (`azure-functions` vs `@azure/functions`)
- `.csproj` references to `Microsoft.Azure.Functions.Worker` (isolated) vs `Microsoft.NET.Sdk.Functions` (in-process)

---

### 4. SKU / Hosting Plan

Scan Azure deployment or infrastructure files for the current hosting plan:

**Detection Sources:**
- **Azure Portal/API**: Query Function App configuration
- **Bicep files**: `infra/*.bicep` - look for `Microsoft.Web/serverfarms`
- **ARM templates**: `*.json` - look for App Service Plan resources
- **Terraform**: `*.tf` - look for `azurerm_service_plan`

| SKU | Type | Recommendation |
|-----|------|----------------|
| **Y1** | Consumption (Dynamic) | ⚠️ Migrate to **Flex Consumption (FC1)** |
| **EP1/EP2/EP3** | Elastic Premium | ✅ Good, consider FC1 for cost savings |
| **FC1** | Flex Consumption | ✅ **Recommended** |
| **B1/S1/P1v2** | Dedicated (App Service) | ⚠️ Consider EP or FC1 for serverless benefits |

---

## Scan Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    FUNCTION APP SCANNER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  0. FETCH BEST PRACTICES (Required First Step)                  │
│     └── Call: mcp_azure_mcp_get_bestpractices                   │
│         ├── command: "get_bestpractices_get"                    │
│         ├── resource: "azurefunctions"                          │
│         └── action: "all"                                       │
│                                                                  │
│  1. DETECT SOURCE TYPE                                           │
│     ├── GitHub Repo?     → Fetch via raw URLs / clone           │
│     ├── Azure DevOps?    → Fetch via Azure DevOps API           │
│     ├── Azure Hosted?    → Query via Azure MCP tools            │
│     └── Local Workspace? → Scan project files directly          │
│                                                                  │
│  2. FETCH SUPPORTED VERSIONS                                     │
│     └── Fetch from: https://learn.microsoft.com/azure/azure-functions/supported-languages
│                                                                  │
│  3. SCAN COMPONENTS                                              │
│     ├── Language Runtime Version                                │
│     ├── Extension Bundles Version                               │
│     ├── Programming Model Version                               │
│     └── Hosting SKU                                             │
│                                                                  │
│  4. GENERATE REPORT (save to scan-report.md)                    │
│     ├── Current State Summary                                   │
│     ├── Recommendations with Priority                           │
│     └── Upgrade Impact Analysis                                 │
│                                                                  │
│  5. PROMPT FOR ACTION                                            │
│     └── "Would you like to proceed with the upgrades?"          │
│                                                                  │
│  6. APPLY UPGRADES (Based on Source Type)                        │
│     ├── GitHub Repo     → Create PR with changes                │
│     ├── Azure DevOps    → Create PR via Azure DevOps CLI        │
│     ├── Azure Hosted    → Apply via Azure CLI                   │
│     └── Local Workspace → Direct file modifications             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Scanning by Source Type

### GitHub Repository

**How to scan:**
```
1. Parse GitHub URL to extract owner/repo
2. Fetch files via raw.githubusercontent.com:
   - https://raw.githubusercontent.com/{owner}/{repo}/{branch}/host.json
   - https://raw.githubusercontent.com/{owner}/{repo}/{branch}/package.json
   - etc.
3. Or use GitHub API for file listing and content
```

**How to apply upgrades:**
```
1. Clone the repository locally:
   git clone https://github.com/{owner}/{repo}.git

2. Create upgrade branch:
   git checkout -b azure-functions-upgrade

3. Copy scan-report.md into cloned repo:
   Copy-Item scan-report.md upgrades/{repo-name}-{date}/

4. Apply code changes to files

5. Commit changes (including scan-report.md):
   git add .
   git commit -m "Upgrade Azure Functions to latest standards"

6. Push and create PR:
   git push origin azure-functions-upgrade
   gh pr create --title "Azure Functions Upgrade" --body-file scan-report.md
```

---

### Azure-Hosted Function App

**How to scan:**
```
Use Azure MCP tools to query the Function App:

1. List Function Apps:
   Tool: mcp_azure_mcp_functionapp
   Command: functionapp_list
   
2. Get Function App details:
   Tool: mcp_azure_mcp_functionapp
   Command: functionapp_get
   Parameters: { subscriptionId, resourceGroup, name }

3. Get App Settings (runtime version, etc.):
   Tool: mcp_azure_mcp_appservice
   Command: appservice_appsettings_list
```

**How to apply upgrades via Azure CLI:**
```bash
# Update runtime version
az functionapp config set \
  --name <function-app-name> \
  --resource-group <resource-group> \
  --linux-fx-version "NODE|22"

# Update app settings
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <resource-group> \
  --settings "FUNCTIONS_EXTENSION_VERSION=~4" \
             "WEBSITE_NODE_DEFAULT_VERSION=~22"

# Update to Flex Consumption (requires redeployment with new Bicep/ARM)
# Generate new Bicep template and deploy:
az deployment group create \
  --resource-group <resource-group> \
  --template-file infra/main.bicep
```

---

### Local Workspace

**How to scan:**
```
Use file system tools to read project files:
- read_file for host.json, package.json, *.csproj, etc.
- file_search for finding function.json files
- grep_search for pattern detection
```

**How to apply upgrades:**
```
Use file editing tools directly:
- replace_string_in_file for updating configurations
- create_file for generating new files
- run_in_terminal for npm install, dotnet commands, etc.
```

---

### Azure DevOps Repository

**How to scan:**
```
1. Parse Azure DevOps URL to extract organization/project/repo
2. Use Azure DevOps REST API or az repos commands:
   az repos show --repository <repo> --org <org> --project <project>
3. Fetch file contents via API
```

**How to apply upgrades:**
```bash
# Clone repository
az repos clone --repository <repo> --org <org> --project <project>

# Create branch and make changes
git checkout -b azure-functions-upgrade

# Push and create PR
git push origin azure-functions-upgrade
az repos pr create \
  --repository <repo> \
  --source-branch azure-functions-upgrade \
  --target-branch main \
  --title "Azure Functions Upgrade" \
  --description "$(cat scan-report.md)"
```

---

## User Consent and PR Workflow

### Consent Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                      USER CONSENT WORKFLOW                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  After generating the scan report, prompt the user:               │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ UPGRADE OPTIONS                                             │  │
│  ├────────────────────────────────────────────────────────────┤  │
│  │                                                             │  │
│  │ Source detected: [GitHub Repository / Azure / Local]       │  │
│  │                                                             │  │
│  │ How would you like to apply the upgrades?                   │  │
│  │                                                             │  │
│  │ [1] 🔀 Create a Pull Request (recommended for GitHub/ADO)  │  │
│  │     - Creates branch: azure-functions-upgrade               │  │
│  │     - Commits all changes                                   │  │
│  │     - Opens PR with scan report as description              │  │
│  │                                                             │  │
│  │ [2] 📝 Apply changes locally (preview first)               │  │
│  │     - Shows diff of all proposed changes                    │  │
│  │     - Applies to local workspace                            │  │
│  │     - You commit/push manually                              │  │
│  │                                                             │  │
│  │ [3] ☁️ Apply via Azure CLI (for Azure-hosted apps)         │  │
│  │     - Generates Azure CLI commands                          │  │
│  │     - Runs commands to update Function App config           │  │
│  │     - Updates runtime, settings, and SKU                    │  │
│  │                                                             │  │
│  │ [4] 📋 Generate upgrade script only                        │  │
│  │     - Creates upgrade-script.sh/ps1                         │  │
│  │     - You run it manually when ready                        │  │
│  │                                                             │  │
│  │ [C] ❌ Cancel                                               │  │
│  │                                                             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Pull Request Template

When creating a PR, use this template for the PR body:

```markdown
## 🚀 Azure Functions Upgrade

This PR upgrades the Azure Function App to the latest recommended standards.

### Changes Made

| Component | Before | After |
|-----------|--------|-------|
| Language Runtime | [OLD] | [NEW] |
| Extension Bundles | [OLD] | [NEW] |
| Programming Model | [OLD] | [NEW] |
| Hosting SKU | [OLD] | [NEW] |

### Files Modified

- [ ] `host.json` - Updated extension bundles
- [ ] `package.json` - Updated Node.js version
- [ ] `infra/main.bicep` - Updated SKU and runtime
- [ ] Function files - Migrated to v4 programming model

### Testing Checklist

- [ ] Run `npm install` to update dependencies
- [ ] Run `func start` to test locally
- [ ] Verify all functions execute correctly
- [ ] Deploy to staging environment for validation

### References

- [Azure Functions Supported Languages](https://learn.microsoft.com/azure/azure-functions/supported-languages)
- [Migrate to v4 Programming Model](https://learn.microsoft.com/azure/azure-functions/functions-node-upgrade-v4)
- [Flex Consumption Plan](https://learn.microsoft.com/azure/azure-functions/flex-consumption-plan)

---
*Generated by Azure Functions Upgrade Skill*
```

### GitHub CLI Commands for PR Creation

```bash
# Ensure GitHub CLI is authenticated
gh auth status

# Clone the repository
gh repo clone {owner}/{repo}
cd {repo}

# Create and checkout upgrade branch
git checkout -b azure-functions-upgrade-$(date +%Y%m%d)

# Apply changes (done by the skill)

# Stage and commit
git add .
git commit -m "chore: upgrade Azure Functions to latest standards

- Update Node.js runtime to v22
- Update extension bundles to [4.*, 5.0.0)
- Migrate to v4 programming model
- Update infrastructure for Flex Consumption"

# Push branch
git push -u origin azure-functions-upgrade-$(date +%Y%m%d)

# Create PR with scan report
gh pr create \
  --title "🚀 Azure Functions Upgrade to Latest Standards" \
  --body-file scan-report.md \
  --label "dependencies,azure-functions" \
  --reviewer "@me"
```

---

## Report Format

> 📖 **Note**: The "RECOMMENDED" column values should be dynamically fetched from:
> **[Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)**

### Sample Upgrade Report

```
╔══════════════════════════════════════════════════════════════════╗
║              AZURE FUNCTIONS UPGRADE REPORT                       ║
║              Generated: [DATE]                                    ║
╠══════════════════════════════════════════════════════════════════╣
║ Function App: [NAME]                                              ║
║ Resource Group: [RG_NAME]                                         ║
║ Location: [REGION]                                                ║
╠══════════════════════════════════════════════════════════════════╣

┌──────────────────────────────────────────────────────────────────┐
│ COMPONENT              │ CURRENT        │ RECOMMENDED  │ STATUS  │
├──────────────────────────────────────────────────────────────────┤
│ Language Runtime       │ [DETECTED]     │ [FROM DOCS]  │ ⚠️ UPDATE│
│ Extension Bundles      │ [DETECTED]     │ [FROM DOCS]  │ ⚠️ UPDATE│
│ Programming Model      │ v3             │ v4           │ 🔴 MAJOR │
│ Hosting SKU            │ Y1 (Dynamic)   │ FC1 (Flex)   │ ⚠️ UPDATE│
│ Functions Host         │ v4             │ v4           │ ✅ OK    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ RECOMMENDATIONS                                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 🔴 HIGH PRIORITY                                                  │
│ ─────────────────                                                 │
│ 1. Upgrade Language Runtime to latest supported version           │
│    - Refer to: https://learn.microsoft.com/azure/azure-functions/supported-languages
│    - Update project config and Azure Function App configuration   │
│    - Impact: Critical - EOL versions have security vulnerabilities│
│                                                                   │
│ 2. Upgrade Extension Bundles                                      │
│    - Update host.json extensionBundle version                     │
│    - Impact: Required for latest features and security fixes      │
│                                                                   │
│ 3. Migrate to Programming Model v4                                │
│    - Remove function.json files                                   │
│    - Update @azure/functions to v4.x                              │
│    - Refactor function definitions to use app.http() pattern     │
│    - Impact: v3 is legacy, v4 is the supported model              │
│                                                                   │
│ 4. Upgrade to Flex Consumption (FC1)                              │
│    - Update Bicep/infrastructure files                            │
│    - Impact: Y1 is deprecated, FC1 is recommended                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘

╚══════════════════════════════════════════════════════════════════╝
```

---

## Upgrade Confirmation

After generating the report, prompt the user with source-aware options:

```
┌──────────────────────────────────────────────────────────────────┐
│                      UPGRADE CONFIRMATION                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Source: [GitHub Repository / Azure-Hosted / Local / Azure DevOps]
│                                                                   │
│  The following upgrades are recommended:                          │
│                                                                   │
│  ☐ 1. 🔴 Upgrade Language Runtime to [LATEST_FROM_DOCS]          │
│  ☐ 2. 🔴 Upgrade Extension Bundles to [LATEST_FROM_DOCS]         │
│  ☐ 3. 🔴 Migrate to Programming Model [LATEST_FROM_DOCS]         │
│  ☐ 4. 🔴 Migrate to Flex Consumption (FC1) SKU                   │
│                                                                   │
│  How would you like to apply the upgrades?                        │
│                                                                   │
│  [1] 🔀 Create Pull Request                                       │
│      → Clone repo, create branch, apply changes, submit PR        │
│      → Best for: GitHub, Azure DevOps repositories                │
│                                                                   │
│  [2] 📝 Apply Locally (with preview)                              │
│      → Show diff first, then apply to workspace                   │
│      → Best for: Local workspace                                  │
│                                                                   │
│  [3] ☁️  Apply via Azure CLI                                       │
│      → Run Azure CLI commands to update live Function App         │
│      → Best for: Azure-hosted apps                                │
│                                                                   │
│  [4] 📋 Generate Script Only                                      │
│      → Create upgrade-script.sh/ps1 for manual execution          │
│      → Best for: Review before applying                           │
│                                                                   │
│  [S] Select specific upgrades only                                │
│  [C] Cancel                                                       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Best Practices Reference

> ⚠️ **REQUIRED**: Before generating recommendations, ALWAYS call the Azure Functions best practices MCP tool to fetch the latest guidance:
>
> ```
> Tool: mcp_azure_mcp_get_bestpractices
> Command: get_bestpractices_get
> Parameters:
>   - resource: "azurefunctions"
>   - action: "all"
> ```
>
> This ensures recommendations are based on the most current Azure Functions guidance for:
> - Code generation patterns
> - Deployment configurations
> - SKU recommendations
> - Extension bundle versions
> - Programming model requirements

> 📖 **Also refer to the official documentation for the latest supported versions:**
> - **[Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)** - Runtime versions, end-of-support dates
> - **[Azure Functions Language Stack Support Policy](https://learn.microsoft.com/en-us/azure/azure-functions/language-support-policy)** - Support lifecycle

Based on the latest Azure Functions guidance:

### Code Generation
- Use the **latest programming model** (v4 for JavaScript/TypeScript, v2 for Python)
- Use **isolated process model** for .NET (not in-process)
- **Do NOT create function.json** for Node.js or Python projects
- Use the **latest extension bundles version** in host.json (check docs for current version)
- Blob triggers should use **Event Grid source**
- Ensure Functions Host **v4** is configured

### Deployment
- **ALWAYS use Flex Consumption (FC1)** plan - never Y1 dynamic
- Use **Linux OS** for Python-based Functions
- Deploy **one Function per Function App** for optimal scaling
- Enable **Application Insights** for monitoring
- Configure **VNET integration** or private endpoints for security
- Use **Azure Verified Modules (AVM)** for Bicep templates

### Infrastructure Templates
Reference samples for Flex Consumption deployment:
- [JavaScript AZD Sample](https://github.com/Azure-Samples/functions-quickstart-javascript-azd/tree/main/infra)
- [.NET EventGrid Sample](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-eventgrid-blob/tree/main/infra)

---

## Files to Scan

| File/Pattern | Purpose |
|--------------|---------|
| `host.json` | Extension bundles, runtime settings |
| `local.settings.json` | Local runtime configuration |
| `package.json` | Node.js version, dependencies, programming model |
| `requirements.txt` | Python dependencies |
| `*.csproj` | .NET framework version, worker model |
| `pom.xml` | Java version and dependencies |
| `infra/*.bicep` | Infrastructure as Code - SKU, configuration |
| `infra/*.json` | ARM templates |
| `*.tf` | Terraform configurations |
| `function.json` | Legacy programming model indicator |

---

## Post-PR Cleanup

After a Pull Request is created and merged, prompt the user for cleanup:

```
┌──────────────────────────────────────────────────────────────────┐
│                      POST-PR CLEANUP                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ✅ Pull Request successfully created/merged!                     │
│                                                                   │
│  PR: https://github.com/{owner}/{repo}/pull/{number}             │
│                                                                   │
│  The following upgrade folder exists in your workspace:           │
│                                                                   │
│  📁 upgrades/{repo-name}-{YYYYMMDD}/                              │
│                                                                   │
│  Would you like to clean up this folder?                          │
│                                                                   │
│  [Y] Yes, delete the folder                                       │
│  [N] No, keep it for reference                                    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Cleanup Commands

```powershell
# PowerShell cleanup
Remove-Item -Path "upgrades/{repo-name}-{YYYYMMDD}" -Recurse -Force

# Or clean all upgrades
Remove-Item -Path "upgrades/*" -Recurse -Force
```

### When to Suggest Cleanup

1. **After PR is created** - Remind user the folder exists
2. **After PR is merged** - Suggest cleanup since changes are now in the target repo
3. **On next scan** - If old upgrade folders exist, offer to clean them up

---

## Complete Workflow Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                 FUNCFORWARD COMPLETE WORKFLOW                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. USER REQUEST                                                 │
│     └── "Scan https://github.com/owner/repo"                    │
│                                                                  │
│  2. FETCH BEST PRACTICES                                         │
│     └── Call mcp_azure_mcp_get_bestpractices                    │
│                                                                  │
│  3. FETCH SUPPORTED VERSIONS                                     │
│     └── Fetch from official docs                                │
│                                                                  │
│  4. SCAN SOURCE (via raw GitHub URLs)                           │
│     ├── host.json                                               │
│     ├── package.json / *.csproj / requirements.txt              │
│     ├── infra/*.bicep                                           │
│     └── function.json files                                     │
│                                                                  │
│  5. GENERATE REPORT                                              │
│     └── Create scan-report.md in workspace root                 │
│                                                                  │
│  6. USER SELECTS UPGRADE OPTION                                  │
│     └── [1] Create PR                                           │
│                                                                  │
│  7. CLONE TO WORKSPACE                                           │
│     └── Clone to upgrades/{repo}-{YYYYMMDD}/                    │
│                                                                  │
│  8. APPLY CHANGES                                                │
│     ├── Copy scan-report.md to cloned repo                      │
│     ├── Update host.json                                        │
│     ├── Update package.json                                     │
│     ├── Migrate to v4 programming model                         │
│     ├── Update infra/main.bicep                                 │
│     └── Delete legacy function.json files                       │
│                                                                  │
│  9. CREATE BRANCH & COMMIT                                       │
│     ├── git checkout -b azure-functions-upgrade                 │
│     ├── git add .                                               │
│     └── git commit -m "Upgrade Azure Functions..."              │
│                                                                  │
│ 10. PUSH & CREATE PR                                             │
│     ├── git push origin azure-functions-upgrade                 │
│     └── gh pr create --body-file scan-report.md                 │
│                                                                  │
│ 11. POST-PR CLEANUP (after merge)                                │
│     └── Prompt user to delete upgrades/{repo}-{YYYYMMDD}/       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
