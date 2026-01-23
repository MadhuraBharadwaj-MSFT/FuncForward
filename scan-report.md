# Azure Functions Upgrade Report

**Generated:** January 23, 2026  
**Source:** GitHub Repository  
**Repository:** [MadhuraBharadwaj-MSFT/OutdatedFunctionApp](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp)  
**Language:** Node.js (JavaScript)

---

## Executive Summary

This Function App requires **3 critical upgrades** to meet current Azure Functions best practices. The programming model is already on v4, but runtime, bundles, and SKU need updates.

---

## Scan Results

| Component | Current | Recommended | Status |
|-----------|---------|-------------|--------|
| **Language Runtime** | Node.js 16 | Node.js 22 (GA) | 🔴 **END OF LIFE** |
| **Extension Bundles** | `[2.*, 3.0.0)` | `[4.*, 5.0.0)` | 🔴 **OUTDATED** |
| **Programming Model** | v4 (code-based) | v4 (code-based) | ✅ **OK** |
| **Hosting SKU** | Y1 (Consumption/Dynamic) | FC1 (Flex Consumption) | 🔴 **DEPRECATED** |
| **Functions Host** | v4 | v4 | ✅ **OK** |

---

## Files Scanned

| File | Finding |
|------|---------|
| [`host.json`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/host.json) | Extension bundles `[2.*, 3.0.0)` - outdated by 2 major versions |
| [`package.json`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/package.json) | `engines.node: "~16"` - end of life, no security updates |
| [`infra/main.bicep`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/infra/main.bicep) | `linuxFxVersion: 'NODE\|16'`, SKU: `Y1` (Dynamic) |
| [`src/functions/httpTrigger.js`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/src/functions/httpTrigger.js) | ✅ v4 programming model using `app.http()` |
| [`src/functions/timerTrigger.js`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/blob/main/src/functions/timerTrigger.js) | ✅ v4 programming model |

### 🧹 Cleanup Recommended

| File | Issue |
|------|-------|
| [`HttpTrigger/`](https://github.com/MadhuraBharadwaj-MSFT/OutdatedFunctionApp/tree/main/HttpTrigger) | Legacy v3 folder - should be deleted (replaced by `src/functions/`) |

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

### 3. Migrate to Flex Consumption (FC1) SKU

**Severity:** 🔴 Critical - Y1 is deprecated, FC1 is the recommended plan

**Current:** Y1 (Consumption/Dynamic)  
**Recommended:** FC1 (Flex Consumption)

**Benefits of FC1:**
- ⚡ Better cold start performance
- 💰 Per-second billing
- 🔒 VNet integration support
- 📈 Improved scaling

---

### 4. Delete Legacy v3 Folders (Cleanup)

**Severity:** 🔴 High - Leftover v3 code should be removed

**Folders to delete:**
- `HttpTrigger/` - Legacy v3 folder (replaced by `src/functions/httpTrigger.js`)

---

## Upgrade Confirmation

| # | Upgrade | Priority | Impact |
|---|---------|----------|--------|
| ☐ 1 | Upgrade Node.js 16 → 22 | 🔴 **HIGH** | Critical - EOL, security risk |
| ☐ 2 | Upgrade Extension Bundles | 🔴 **HIGH** | Required for security fixes |
| ☐ 3 | Migrate to Flex Consumption (FC1) | 🔴 **HIGH** | Y1 is deprecated |
| ☐ 4 | Delete legacy `HttpTrigger/` folder | 🔴 **HIGH** | Cleanup leftover v3 code |

---

## Next Steps

**Source detected:** GitHub Repository

**How would you like to apply the upgrades?**

| Option | Description |
|--------|-------------|
| **[1] 🔀 Create Pull Request** | Clone repo, apply changes, submit PR to OutdatedFunctionApp |
| **[2] 📝 Apply Locally** | Clone to workspace for manual review |
| **[3] 📋 Generate Script** | Create upgrade script |
| **[C] ❌ Cancel** | Cancel |

---

## References

- [Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)
- [Migrate to v4 Programming Model](https://learn.microsoft.com/en-us/azure/azure-functions/functions-node-upgrade-v4)
- [Flex Consumption Plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)

---
*Generated by FuncForward - Azure Functions Upgrade Skill*
