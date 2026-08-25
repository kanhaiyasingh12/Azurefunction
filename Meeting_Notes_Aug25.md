# Meeting Notes — Azure Functions SRE Discovery: DEV Complete

**Date:** 2026-08-25  
**Status:** DEV discovery finalized and published to Azure DevOps Wiki

---

## Opening

> Hi everyone, I have a status update on the Azure Functions SRE Discovery. I finalized the comprehensive DEV discovery report and added it to our Azure DevOps Wiki. , I've successfully moved Azure Functions in DEV from 'unknown' to 'fully mapped' in our Release Orchestration Capability Matrix.
I mapped 10  function apps, with 30 functions, 3 Logic Apps, 153 RBAC assignments, 23 metric alerts, 51 log alerts, and verified network exposure across the board.

---
## 1. Observability Gaps

**"9 out of 10 have App Insights, but the cost ingestion app has zero telemetry — and none of the 10 apps have diagnostic settings."**

- 9/10 apps have `APPLICATIONINSIGHTS_CONNECTION_STRING` configured
- `helios-dev-cost-ingestion` (daily cost job) has **no instrumentation key and no connection string** — zero telemetry
- 0/10 apps have resource-level diagnostic settings — platform logs, host startup errors, and scaling logs are not being forwarded to Log Analytics

---
## 2. Identity & Security

**"All 10 apps use System-Assigned Managed Identity, I mapped 153 RBAC assignments, and all 4 Key Vaults in scope use Azure RBAC — no legacy Access Policies."**

- All 10 Function Apps verified with System-Assigned Managed Identity enabled
- 153 RBAC assignments mapped across the estate
- 4 Key Vaults in scope:
  - `helios-dev-backend-kv`
  - `uudri-dev-kv`
  - (+ 2 others)
- All 4 Key Vaults configured with **Azure RBAC authorization enabled** — secrets access is governed via RBAC permissions (e.g., `Key Vault Secrets User`) rather than legacy Access Policies

  ---
## 3. Network Security

**"Zero VNet integration, zero private endpoints, zero IP restrictions — all 10 apps are fully exposed to the public internet."**

- 0/10 apps have outbound VNet integration
- 0/10 apps have inbound private endpoints
- All inbound IP restrictions for both SCM and runtime endpoints are disabled (defaulting to `Allow All`)
- All 10 apps are publicly accessible with no network boundary

---
---

## 4. Alerting Coverage

**"23 metric alerts and 51 log alerts routing through 5 Action Groups — 8 out of 10 apps are covered, but the two UUDRI bill processors have no metric alerts."**

- 23 metric alerts configured — watching for `Http5xx` errors (Severity 3)
- 51 log alerts configured
- 5 Action Groups routing notifications
- 8/10 apps are covered and routed to the Microsoft Teams alert channel via webhook
- **Gap:** The two UUDRI bill processor apps (`uudri-bill-processor-dev` and `UUDRI-Bill-Processor-dev-01`) lack any metric alerts

---

## 5. CI/CD (Previously Reported)

**"GitHub Actions for plan narration, Azure DevOps for UUDRI, az CLI for others — no single deployment standard, no test gates, no rollback."**

- Separate workflow files for DEV/QA/PROD — no immutable artifact promotion
- pytest not wired to deploy workflows
- No automatic rollback on post-deploy validation failure

---

## 6. Capability Matrix Update

Based on DEV findings, the Azure Functions row can now be updated:

| Capability | Previous | Current DEV Status | Evidence |
|:---|:---|:---|:---|
| Deploy on merge | unknown | **Mapped (DEV)** | Deployment methods (GitHubAction, VSTSRM, CLI/ZipDeploy) documented for all 10 apps in `DEV-CICD-Discovery-All10.csv`. |
| Promotion gates (dev > QA > prod) | unknown | **Mapped (DEV)** | Deployment slots verified for all apps (1 app has slots). Workflow files in `helios-plan-narration-backend` examined. |
| Scheduled verification | not started | **Mapped (DEV)** | Confirmed 0 out of 10 apps have health checks or URL ping tests active. Gaps saved in `DEV-Scheduled-Verification.csv`. |
| Alerting | not started | **Mapped (DEV)** | 8/10 apps are covered by 23 metric and 245 log alerts routing to `dev-teams` action group. Matrix saved in `DEV-Alert-Coverage-Matrix.csv`. |
| Observability + cost | not started | **Mapped (DEV)** | 9/10 apps have App Insights (cost-ingestion missing). 0/10 have diagnostic settings. Saved in `DEV-Observability-Complete.csv`. |
| Reporting / audit | not started | **Mapped (DEV)** | Confirmed 0/10 diagnostic settings. Diagnostic/Log Analytics workspaces mapped in `DEV-LogAnalytics-Workspaces.csv` & `DEV-Diagnostic-Settings.csv`. |


---

## 7. Next Steps

**"DEV is done. Now we replicate the same ARM-based discovery process for QA and PROD."**

- Replicate the same ARM REST API discovery methodology to **QA** environment
- Replicate to **PROD** environment
- Cross-environment comparison once all three are mapped
- Promotion-path analysis (DEV → QA → PROD consistency)

---

## If They Ask

| Question | Answer |
|---|---|
| Why two bill processor apps? | Same trigger config. Relationship not determined — could be migration, versioning, or parallel. Both lack alerting. |
| Is the network exposure a problem? | All 10 apps are public with no IP restrictions. Whether that's acceptable depends on the data sensitivity and compliance requirements — that's a team decision. |
| Are the Key Vaults secure? | All 4 use Azure RBAC, not legacy Access Policies. Specific role assignments need review to confirm least-privilege. |
| Who gets paged when something fails? | 8/10 apps route to Teams via webhook. The 2 UUDRI apps have no metric alerts. Cost ingestion has no telemetry at all. |
| When will QA/PROD be done? | Same methodology, same ARM queries. Timeline depends on access and scope. |

---
### 9 Detailed SRE Capability Position

| Area | Status | Evidence File |
|:---|:---|:---|
| Resource inventory | Mapped | `DEV-Function-Inventory.csv` |
| Function inventory | Mapped | `DEV-Function-Inventory.csv` |
| Trigger/bindings | Mapped | `DEV-Binding-Inventory.csv` |
| Event dependencies | Mapped | `DEV-Event-Dependencies-Resolved.csv` |
| Durable architecture | Mapped | `SRE_DEV_Discovery_Architecture_30_Functions.md` |
| Platform configuration | Mapped | `DEV-Platform-Configuration-Complete.csv` |
| CI/CD | Mapped | `DEV-CICD-Discovery-All10.csv` |
| Observability config | Mapped | `DEV-Observability-Complete.csv` |
| Alerting | Mapped | `DEV-Alert-Coverage-Matrix.csv` |
| Scheduled verification | Mapped (Confirmed Gap) | `DEV-Scheduled-Verification.csv` |
| Reporting/audit | Mapped (Confirmed Gap) | `DEV-Diagnostic-Settings.csv` |
| Identity/RBAC | Mapped | `DEV-Managed-Identity.csv` & `DEV-RBAC-Assignments.csv` |
| Networking | Mapped | `DEV-Networking.csv` |
| SLO/SLI | Mapped (Baseline Metrics) | `DEV-SLI-Metrics-Baseline.csv` & `DEV-SLO-Status.csv` |
