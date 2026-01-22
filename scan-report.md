# Azure Functions Upgrade Report

**Generated:** January 22, 2026  
**Source:** GitHub Repository  
**Repository:** [MadhuraBharadwaj-MSFT/OutdatedFunctionApp](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp)  
**Language:** Node.js (JavaScript)

---

## Scan Results

| Component | Current | Recommended | Status |
|-----------|---------|-------------|--------|
| **Language Runtime** | Node.js 16 | Node.js 22 (GA) | 🔴 **END OF LIFE** |
| **Extension Bundles** | `[2.*, 3.0.0)` | `[4.*, 5.0.0)` | 🔴 **OUTDATED** |
| **Programming Model** | v3 (function.json) | v4 (code-based) | 🔴 **LEGACY** |
| **Hosting SKU** | Y1 (Consumption/Dynamic) | FC1 (Flex Consumption) | 🔴 **DEPRECATED** |
| **Functions Host** | v4 | v4 | ✅ **OK** |

---

## Files Scanned

| File | Finding |
|------|---------|
| [`host.json`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/host.json) | Extension bundles `[2.*, 3.0.0)` - outdated by 2 major versions |
| [`package.json`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/package.json) | `engines.node: "~16"` - end of life, no security updates |
| [`infra/main.bicep`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/infra/main.bicep) | `linuxFxVersion: 'NODE\|16'`, SKU: `Y1` (Dynamic) |
| [`HttpTrigger/function.json`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/HttpTrigger/function.json) | v3 programming model detected |
| [`HttpTrigger/index.js`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/HttpTrigger/index.js) | Legacy `module.exports` pattern |
| [`TimerTrigger/function.json`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/TimerTrigger/function.json) | v3 programming model detected |

---

## 🔴 HIGH PRIORITY Recommendations

### 1. Upgrade Node.js Runtime (Node.js 16 → Node.js 22)

**Severity:** 🔴 Critical - Node.js 16 reached End of Life, no security updates

**Current supported versions** (from [official docs](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages?pivots=programming-language-javascript)):

| Version | Status | End of Support |
|---------|--------|----------------|
| Node.js 24 | Preview | April 30, 2028 |
| Node.js 22 | **GA (Recommended)** | April 30, 2027 |
| Node.js 20 | GA | April 30, 2026 |

**Files to update:**
- `package.json`: Change `"node": "~16"` → `"node": "~22"`
- `infra/main.bicep`: Change `linuxFxVersion: 'NODE|16'` → `linuxFxVersion: 'NODE|22'`
- `infra/main.bicep`: Change `WEBSITE_NODE_DEFAULT_VERSION: '~16'` → `'~22'`

---

### 2. Upgrade Extension Bundles (`[2.*, 3.0.0)` → `[4.*, 5.0.0)`)

**Severity:** 🔴 Critical - Missing security fixes and latest binding features

**File to update:** `host.json`

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

---

### 3. Migrate to Programming Model v4

**Severity:** 🔴 Critical - v3 model is legacy, v4 is the supported model

**Changes required:**
1. Install `@azure/functions` v4.x package
2. Delete all `function.json` files
3. Refactor functions to use `app.http()` and `app.timer()` patterns
4. Update project structure to v4 layout

**Migration for HttpTrigger/index.js:**

```javascript
// BEFORE (v3 - requires function.json)
module.exports = async function (context, req) {
    context.log('JavaScript HTTP trigger function processed a request.');
    const name = (req.query.name || (req.body && req.body.name));
    context.res = {
        body: "Hello, " + name
    };
};

// AFTER (v4 - code-based registration)
const { app } = require('@azure/functions');

app.http('HttpTrigger', {
    methods: ['GET', 'POST'],
    authLevel: 'function',
    handler: async (request, context) => {
        context.log('HTTP trigger function processed a request.');
        const name = request.query.get('name') || await request.text();
        return { 
            body: `Hello, ${name}. This HTTP triggered function executed successfully.` 
        };
    }
});
```

**Migration for TimerTrigger:**

```javascript
const { app } = require('@azure/functions');

app.timer('TimerTrigger', {
    schedule: '0 */5 * * * *',
    handler: async (myTimer, context) => {
        context.log('Timer trigger function ran!');
    }
});
```

**Updated package.json:**

```json
{
  "name": "sub-par-function-app",
  "version": "2.0.0",
  "description": "Node.js Azure Function App using v4 programming model",
  "main": "src/functions/*.js",
  "scripts": {
    "start": "func start",
    "test": "echo \"No tests configured\""
  },
  "engines": {
    "node": "~22"
  },
  "dependencies": {
    "@azure/functions": "^4.0.0"
  }
}
```

---

### 4. Migrate to Flex Consumption (FC1) SKU

**Severity:** 🔴 Critical - Y1 is deprecated, FC1 is the recommended plan

**Current:** Y1 (Consumption/Dynamic)  
**Recommended:** FC1 (Flex Consumption)

**Benefits of FC1:**
- ⚡ Better cold start performance
- 💰 Per-second billing with no minimum
- 🔒 VNet integration support out of the box
- 📈 Improved scaling capabilities

**Reference for Flex Consumption Bicep:**
- [JavaScript AZD Sample](https://github.com/Azure-Samples/functions-quickstart-javascript-azd/tree/main/infra)

---

## Upgrade Confirmation

| # | Upgrade | Priority | Impact |
|---|---------|----------|--------|
| ☐ 1 | Upgrade Node.js 16 → 22 | 🔴 **HIGH** | Critical - Node.js 16 is EOL, security risk |
| ☐ 2 | Upgrade Extension Bundles to `[4.*, 5.0.0)` | 🔴 **HIGH** | Required for latest features and security fixes |
| ☐ 3 | Migrate to v4 Programming Model | 🔴 **HIGH** | v3 is legacy, v4 is the supported model |
| ☐ 4 | Migrate to Flex Consumption (FC1) | 🔴 **HIGH** | Y1 is deprecated, FC1 is recommended |

---

## Next Steps

**Source detected:** GitHub Repository

**How would you like to apply the upgrades?**

| Option | Description |
|--------|-------------|
| **[1] 🔀 Create Pull Request** | Clone repo, create branch `azure-functions-upgrade`, apply all changes, submit PR |
| **[2] 📝 Apply Locally** | Clone to workspace, show diff, apply changes for manual commit |
| **[3] 📋 Generate Script** | Create `upgrade-script.sh` with all required changes |
| **[C] ❌ Cancel** | Cancel upgrade |

---

## References

- [Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)
- [Azure Functions Language Stack Support Policy](https://learn.microsoft.com/en-us/azure/azure-functions/language-support-policy)
- [Node.js Developer Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-node)
- [Migrate to v4 Programming Model](https://learn.microsoft.com/en-us/azure/azure-functions/functions-node-upgrade-v4)
- [Flex Consumption Plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)

---
*Generated by FuncForward - Azure Functions Upgrade Skill*
