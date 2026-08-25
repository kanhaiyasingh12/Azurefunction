# Meeting Notes — Azure Functions SRE Discovery: DEV Complete

**Date:** 2026-08-25  
**Presenter:** SRE Team  
**Status:** DEV discovery finalized and published to Azure DevOps Wiki

---

## Quick Summary

DEV environment Azure Functions discovery is **complete**. The report is live on our ADO Wiki. Azure Functions has moved from `unknown` to `fully mapped` in the capability matrix for DEV. We mapped 10 apps, 30 functions, 3 Logic Apps, 153 RBAC assignments, 23 metric alerts, 51 log alerts, and verified network exposure across the board. Next step: replicate to QA and PROD.

---

## Opening

> Hi everyone, I have a status update on the Azure Functions SRE Discovery. I finalized the comprehensive DEV discovery report and published it to our Azure DevOps Wiki. With this report complete, we've successfully moved Azure Functions in DEV from 'unknown' to 'fully mapped' in our Release Orchestration Capability Matrix.

---

## 1. Scope & Runtimes

**"10 Function Apps running 30 functions across Python, Node.js, and .NET isolated — plus 3 Logic Apps monitoring event streams."**

- Python 3.11/3.12 — 5 apps, 9 functions
- Node.js 20 — 2 apps, 17 functions
- .NET isolated — 3 apps, 4 functions
- 3 Logic Apps (workflow apps) in the same resource group monitoring weather, market, and data event streams

---

## 2. SOP Factory — The Critical Component

**"One app contains 16 of the 30 functions — it's a complete Durable Functions document processing pipeline and a major single point of failure."**

- `func-orchestrator-sopfactorydevmlel9` — Node.js 20, 16 functions
- Documents enter via HTTP POST (`intakeDocument`) or Blob Storage (`onSourceDocument`)
- Orchestrator chains: classify → ingest → screen → generate 4 artifact types (FDD, ontology, policy, SOP) → open GitHub PR
- Query endpoints for pipeline status (`pipelineRuns`, `pipelineRunDetail`, `pipelineSmeQueue`)
- **Single point of failure** — if this app goes down, the entire SOP Factory workflow stops

---

## 3. Network Security

**"Zero VNet integration, zero private endpoints, zero IP restrictions — all 10 apps are fully exposed to the public internet."**

- 0/10 apps have outbound VNet integration
- 0/10 apps have inbound private endpoints
- All inbound IP restrictions for both SCM and runtime endpoints are disabled (defaulting to `Allow All`)
- All 10 apps are publicly accessible with no network boundary

---

## 4. Observability Gaps

**"9 out of 10 have App Insights, but the cost ingestion app has zero telemetry — and none of the 10 apps have diagnostic settings."**

- 9/10 apps have `APPLICATIONINSIGHTS_CONNECTION_STRING` configured
- `helios-dev-cost-ingestion` (daily cost job) has **no instrumentation key and no connection string** — zero telemetry
- 0/10 apps have resource-level diagnostic settings — platform logs, host startup errors, and scaling logs are not being forwarded to Log Analytics

---

## 5. Identity & Security

**"All 10 apps use System-Assigned Managed Identity, I mapped 153 RBAC assignments, and all 4 Key Vaults in scope use Azure RBAC — no legacy Access Policies."**

- All 10 Function Apps verified with System-Assigned Managed Identity enabled
- 153 RBAC assignments mapped across the estate
- 4 Key Vaults in scope:
  - `helios-dev-backend-kv`
  - `uudri-dev-kv`
  - (+ 2 others)
- All 4 Key Vaults configured with **Azure RBAC authorization enabled** — secrets access is governed via RBAC permissions (e.g., `Key Vault Secrets User`) rather than legacy Access Policies

---

## 6. Alerting Coverage

**"23 metric alerts and 51 log alerts routing through 5 Action Groups — 8 out of 10 apps are covered, but the two UUDRI bill processors have no metric alerts."**

- 23 metric alerts configured — watching for `Http5xx` errors (Severity 3)
- 51 log alerts configured
- 5 Action Groups routing notifications
- 8/10 apps are covered and routed to the Microsoft Teams alert channel via webhook
- **Gap:** The two UUDRI bill processor apps (`uudri-bill-processor-dev` and `UUDRI-Bill-Processor-dev-01`) lack any metric alerts

---

## 7. CI/CD (Previously Reported)

**"GitHub Actions for plan narration, Azure DevOps for UUDRI, az CLI for others — no single deployment standard, no test gates, no rollback."**

- Separate workflow files for DEV/QA/PROD — no immutable artifact promotion
- pytest not wired to deploy workflows
- No automatic rollback on post-deploy validation failure

---

## 8. Capability Matrix — Final DEV Position

| Capability | Previous | Current DEV |
|---|---|---|
| Deploy on merge | `unknown` | **Partially mapped** |
| Promotion gates (dev > QA > prod) | `unknown` | **Partially mapped** |
| Scheduled verification | `not started` | **Not started** |
| Alerting | `not started` | **Mapped** — 23 metric + 51 log alerts, 5 Action Groups, 2 apps uncovered |
| Observability + cost | `not started` | **Mapped** — 9/10 App Insights, 0/10 diagnostic settings, 1 app blind |
| Reporting / audit | `not started` | **Baseline established** |
| Identity / RBAC | `not started` | **Mapped** — 153 assignments, all Managed Identity, RBAC-enabled Key Vaults |
| Networking | `not started` | **Mapped** — 0/10 VNet, 0/10 private endpoints, all public |

---

## 9. Next Steps

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

## The Numbers to Remember

```
10 apps, 30 functions, 3 Logic Apps
16 functions in one app (SOP Factory) — single point of failure
153 RBAC assignments — all Managed Identity
4 Key Vaults — all Azure RBAC (no legacy Access Policies)
23 metric alerts, 51 log alerts, 5 Action Groups
8/10 apps have alerting — 2 UUDRI apps don't
9/10 have App Insights — cost ingestion has nothing
0/10 have diagnostic settings
0/10 have VNet integration or private endpoints
0/10 have IP restrictions — all public
DEV is fully mapped — next: QA and PROD
```
