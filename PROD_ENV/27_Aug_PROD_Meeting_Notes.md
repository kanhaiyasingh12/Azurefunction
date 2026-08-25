# Meeting Notes — Azure Functions SRE Discovery: PROD Complete

**Date:** 2026-08-27  
**Status:** PROD discovery finalized and published to Azure DevOps Wiki

---

## Opening

> Hi everyone, I have our final status update on the Azure Functions SRE Discovery. I have successfully completed the discovery for the Production environment and published the report to our ADO Wiki. With this final environment mapped, I have fully completed the Release Orchestration Capability Matrix for Azure Functions across the entire estate.

---

## Quick Summary

PROD environment Azure Functions discovery is **complete**. I mapped 7 Function Apps, 25 functions, and 3 Logic Apps. Production shows a noticeably smaller footprint than DEV and QA, indicating several apps are either legacy or have not been promoted.  verified cold starts across the board, strict manual deployment gates, and 100% metric alert coverage. 

---

## 1. Scope & Runtimes — Scope Gaps

**"We run 7 apps and 25 functions in PROD — and several major components are completely absent."**

- **Missing Services:** The two UUDRI utility apps and the `helios-device-telemetry` app **do not exist** in the Production subscription.
- **Total Functions:** 25 registered functions (down from 28 in QA and 30 in DEV).
- **Runtimes:** Python 3.11/3.12, Node.js 20, and .NET isolated.
- **Cold Starts:** Like QA, **AlwaysOn is False for all 7 apps in PROD**, so every app experiences cold starts.

---

## 2. Structural & Function-Level Deltas

**"PROD is running a subset of functions compared to DEV and QA, indicating branch sync drift."**

- **Plan Narration Delta:** `ems-plan-narration-function-prod` runs only **1 function** (`plan_narration_agent`). The `realized_kpi_listener` function present in DEV/QA is missing in PROD.
- **SOP Factory Orchestrator:** Runs **15 functions** (lacking both `pipelineSmeQueue` from DEV and `buildPullRequest` from QA).
- **Logic Apps Delta:** We have **only 3 workflow apps** in PROD. The weather eventstream monitor (`helios-data-weather-eh-eventstream-monitor`) present in DEV/QA does not exist in PROD.

---

## 3. Network Security

**"All 7 apps are public to the internet with zero IP restrictions, and only one uses VNet integration."**

- **Public Exposure:** All 7 apps are fully public to the internet with zero inbound IP restrictions.
- **Outbound integration:** Only **1 out of 7 apps** (`ems-plan-narration-function-prod`) has outbound VNet integration configured.

---

## 4. Observability & Alerting

**"Alerting coverage is 100% in PROD, but platform diagnostics are completely missing."**

- **Alert Coverage:** All 7 apps (100%) route `Http5xx` metric alerts to the Action Group **`ag-helios-prod-ops`** (Teams/webhook channel).
- **App Insights Gaps:** 6 out of 7 have App Insights. `helios-prod-cost-ingestion` completely lacks App Insights telemetry. 
- **Diagnostic Settings Gaps:** **0 out of 7** apps have diagnostic settings configured, matching DEV/QA.

---

## 5. Identity & Key Vault Security

**"All 7 apps run on System-Assigned Managed Identity, but Key Vault security is a mixed posture."**

- **Identity:** 100% of the active apps (7/7) run using System-Assigned Managed Identity.
- **Key Vault Models:** 6 out of 8 Key Vaults in the PROD subscription use Azure RBAC. However, the UI vault (`helios-prod-ui-kv`) and Sparkplug PKI vault (`helios-prd-spkplg-pki-kv`) still use legacy Access Policies.

---

## 6. Scheduled Verification

**"Zero automated platform health checks exist in PROD."**

- **Scheduled Verification Gaps:** **0 out of 7 apps** have platform health check paths, and there are zero availability web tests set up.

---

## 7. CI/CD: Strict Manual Gates

**"The production deployment workflow is strictly manual, with zero automated triggers."**

- **Manual Trigger Only:** The PROD workflow (`deploy-function-app-prod.yml`) runs *only* on manual `workflow_dispatch`. There are no push or pull request triggers configured.
- **Runner & Gates:** Runs on `arc-runner-set-prod` and utilizes the GitHub `prod` environment for mandatory approval gates.
- **No Immutable Promotion:** Rebuilds code from scratch rather than promoting a tested QA package.

---

## 8. Capability Matrix — Final PROD Position

| Capability | Status | Evidence |
|---|---|---|
| Deploy on merge | **Mapped (PROD)** | Manual-only triggers for PROD pipelines. |
| Promotion gates (dev > QA > prod) | **Mapped (PROD)** | Separate repo workflow, manual trigger requirements, GitHub env approval gates. |
| Scheduled verification | **Mapped (PROD)** | 0/7 health checks or ping tests. |
| Alerting | **Mapped (PROD)** | 7/7 apps route metric alerts to `ag-helios-prod-ops`. |
| Observability + cost | **Mapped (PROD)** | 6/7 App Insights configured, 0/7 diagnostic settings. |
| Reporting / audit | **Mapped (PROD)** | Mapped Managed Identity, RBAC roles, and Key Vault access types. |

---

## 9. Next Steps

**"With DEV, QA, and PROD fully mapped, discovery is complete."**

- Now i make a implementation paln for DEV, QA, PROD.
- Present the final matrix findings to engineering leadership.
- Prioritize remediation for the identified gaps (e.g., diagnostic settings, App Insights on cost-ingestion).
- Address the branch synchronization drift identified between environments.

---

## If They Ask

| Question | Answer |
|---|---|
| Why are UUDRI and telemetry apps missing in PROD? | They are either legacy components that are being retired, or they are not approved/deployed to the production landing zones yet. |
| Why is `realized_kpi_listener` missing in PROD? | It suggests a deployment mismatch where code for the KPI listener has not been promoted or activated in the production branch. |
| What's the impact of `AlwaysOn = False` in PROD? | High cold start latencies (up to 10-15 seconds) when functions are invoked after periods of inactivity. |
| What's next? | With DEV, QA, and PROD fully mapped, we have completed the **Release Orchestration Capability Matrix** for Azure Functions! |

---

## The Numbers to Remember

```text
7 apps, 25 functions (UUDRI and telemetry apps are absent)
All 7 apps have AlwaysOn = False (cold starts enabled)
1/7 has outbound VNet integration (ems-plan-narration-function-prod)
6/7 have App Insights, 1 doesn't (cost-ingestion)
0/7 have diagnostic settings, 0/7 have health check paths
7/7 covered by metric alerts routing to ag-helios-prod-ops (100% alert coverage)
Strictly manual GitHub actions deployment to PROD
```
