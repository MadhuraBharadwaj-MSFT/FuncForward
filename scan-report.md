# Azure Functions Upgrade Report

**Generated:** January 22, 2026  
**Repository:** MadhuraBharadwaj-MSFT/OutdatedFunctionApp  
**Language:** Node.js (JavaScript)

---

## Scan Results

| Component | Current | Recommended | Status |
|-----------|---------|-------------|--------|
| **Language Runtime** | Node.js 16 | Node.js 22 (GA) or Node.js 24 (Preview) | 🔴 **END OF LIFE** |
| **Extension Bundles** | `[2.*, 3.0.0)` | `[4.*, 5.0.0)` | ⚠️ **OUTDATED** |
| **Programming Model** | v3 (function.json) | v4 (code-based) | ⚠️ **LEGACY** |
| **Hosting SKU** | Y1 (Consumption/Dynamic) | FC1 (Flex Consumption) | ⚠️ **UPGRADE RECOMMENDED** |
| **Functions Host** | v4 | v4 | ✅ **OK** |

---

## Files Scanned

| File | Finding |
|------|---------|
| `host.json` | Extension bundles `[2.*, 3.0.0)` - outdated |
| `package.json` | `engines.node: "~16"` - end of life |
| `infra/main.bicep` | `linuxFxVersion: 'NODE\|16'`, SKU: `Y1` (Dynamic) |
| `HttpTrigger/function.json` | v3 programming model detected |
| `TimerTrigger/function.json` | v3 programming model detected |

---

## 🔴 HIGH PRIORITY Recommendations

### 1. Upgrade Node.js Runtime (Node.js 16 → Node.js 22)

**Severity:** Critical - Node.js 16 reached End of Life

**Current supported versions** (from [official docs](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages?pivots=programming-language-javascript)):

| Version | Status | End of Support |
|---------|--------|----------------|
| Node.js 24 | Preview | April 30, 2028 |
| Node.js 22 | **GA** | April 30, 2027 |
| Node.js 20 | GA | April 30, 2026 |

**Files to update:**
- `package.json`: Change `"node": "~16"` → `"node": "~22"`
- `infra/main.bicep`: Change `linuxFxVersion: 'NODE|16'` → `linuxFxVersion: 'NODE|22'`
- `infra/main.bicep`: Change `WEBSITE_NODE_DEFAULT_VERSION: '~16'` → `'~22'`

---

### 2. Migrate to Programming Model v4

**Severity:** High - v3 model is legacy

**Changes required:**
- Delete all `function.json` files
- Install `@azure/functions` v4.x package
- Refactor functions to use `app.http()` and `app.timer()` patterns

**Example migration for HttpTrigger:**

```javascript
// Before (v3 - requires function.json)
module.exports = async function (context, req) {
    context.log('HTTP trigger function processed a request.');
    // ...
}

// After (v4 - code-based registration)
const { app } = require('@azure/functions');

app.http('HttpTrigger', {
    methods: ['GET', 'POST'],
    authLevel: 'function',
    handler: async (request, context) => {
        context.log('HTTP trigger function processed a request.');
        const name = request.query.get('name') || await request.text();
        return { body: `Hello, ${name}!` };
    }
});
```

**Example migration for TimerTrigger:**

```javascript
// Before (v3 - requires function.json)
module.exports = async function (context, myTimer) {
    context.log('Timer trigger function ran!');
}

// After (v4 - code-based registration)
const { app } = require('@azure/functions');

app.timer('TimerTrigger', {
    schedule: '0 */5 * * * *',
    handler: async (myTimer, context) => {
        context.log('Timer trigger function ran!');
    }
});
```

---

## 🔴 HIGH PRIORITY Recommendations (continued)

### 3. Upgrade Extension Bundles

**Files to update:**
- `host.json`: Change `"version": "[2.*, 3.0.0)"` → `"version": "[4.*, 5.0.0)"`

**Updated host.json:**

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  }
}
```

---

### 4. Migrate to Flex Consumption (FC1) SKU

**Current:** Y1 (Consumption/Dynamic)  
**Recommended:** FC1 (Flex Consumption)

**Benefits:**
- Better cold start performance
- Per-second billing with no minimum
- VNet integration support
- Improved scaling

**Files to update in `infra/main.bicep`:**
- Change App Service Plan SKU from `Y1`/`Dynamic` to Flex Consumption
- Add `functionAppConfig` with deployment storage configuration

**Reference samples for Flex Consumption deployment:**
- [JavaScript AZD Sample](https://github.com/Azure-Samples/functions-quickstart-javascript-azd/tree/main/infra)
- [.NET EventGrid Sample](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-eventgrid-blob/tree/main/infra)

---

## Upgrade Confirmation

The following upgrades are recommended:

| # | Upgrade | Priority | Impact |
|---|---------|----------|--------|
| ☐ 1 | Upgrade Node.js 16 → 22 | 🔴 **HIGH** | Critical - Node.js 16 is EOL, security risk |
| ☐ 2 | Upgrade Extension Bundles to `[4.*, 5.0.0)` | 🔴 **HIGH** | Required for latest features and security fixes |
| ☐ 3 | Migrate to v4 Programming Model | 🔴 **HIGH** | v3 is legacy, v4 is the supported model |
| ☐ 4 | Migrate to Flex Consumption (FC1) | 🔴 **HIGH** | Y1 is deprecated, FC1 is recommended |

---

## Next Steps

**Would you like to proceed with making these upgrades?**

| Option | Description |
|--------|-------------|
| **[A]** | Apply all upgrades |
| **[S]** | Select specific upgrades |
| **[P]** | Preview changes first |
| **[C]** | Cancel |

---

## References

- [Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)
- [Azure Functions Language Stack Support Policy](https://learn.microsoft.com/en-us/azure/azure-functions/language-support-policy)
- [Node.js Developer Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-node)
- [Migrate to v4 Programming Model](https://learn.microsoft.com/en-us/azure/azure-functions/functions-node-upgrade-v4)
