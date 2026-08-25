# Meeting Notes — Azure Functions SRE Discovery: QA Complete

**Date:** 2026-08-26  
**Presenter:** SRE Team  
**Status:** QA discovery finalized and published to Azure DevOps Wiki

---

## Opening

> Hi everyone, I have a status update on the Azure Functions QA Discovery. Following our DEV mapping, we automated the scan for the QA environment and published the final QA report to our ADO Wiki. This moves our Azure Functions QA posture from 'unknown' to 'fully mapped' in the capability matrix, revealing a few key configuration differences from DEV.

---

## Quick Summary

QA environment Azure Functions discovery is **complete**. The report is live on our ADO Wiki. We successfully mapped 10 Function Apps (2 of which are currently idle), 28 functions, and 4 Logic Apps. We mapped out cold starts, network integration, a mixed Key Vault security posture, and a manual CI/CD flow. Next step: conclude the mapping with the PROD environment.

---

## 1. Scope & Runtimes

**"10 Function Apps running 28 functions across Node.js, Python, and .NET isolated — but two apps are currently sitting empty."**

- **Total Functions:** 28 registered functions (down from 30 in DEV).
- **Runtimes:** 17 Node.js functions, 8 Python functions, and 3 .NET isolated functions.
- **Inactive Apps:** `UUDRI-Function-App-qa-01` and `helios-device-telemetry-qa-func` are running but currently have **0 functions** deployed to them.
- **Cold Starts:** Unlike DEV, **AlwaysOn is False for all 10 apps in QA**, meaning every app is subject to cold starts.

---

## 2. SOP Factory — Architectural Delta

**"The SOP Factory is still our largest component with 16 functions, but its code structure differs from DEV."**

- Runs Node.js 20 in `func-orchestrator-sopfactoryqao80ns`.
- **New Activity:** QA includes a `buildPullRequest` activity function.
- **Missing Endpoint:** QA is missing the `pipelineSmeQueue` HTTP endpoint found in DEV.

---

## 3. Network Security Delta

**"All 10 apps are public to the internet, but we have our first instance of VNet integration in QA."**

- **Public Exposure:** Like DEV, all 10 apps are public with no inbound private endpoints or IP restrictions (defaulting to Allow All).
- **Outbound integration:** Unlike DEV (which had 0%), **1 out of 10 apps** has outbound VNet integration:
  - `ems-plan-narration-function-qa` is integrated into the `appservice-subnet` on `helios-aks-qa-vnet`.

---

## 4. Observability Gaps

**"8 out of 10 apps run App Insights, but we still have a total lack of platform diagnostic logs."**

- **App Insights Gaps:** Two apps completely lack App Insights telemetry config: `helios-qa-cost-ingestion` and the empty `UUDRI-Function-App-qa-01`.
- **Diagnostic Settings Gaps:** **0 out of 10 apps** have diagnostic settings. Centralized platform logs and scaling telemetry are completely missing in QA, matching the DEV gap.

---

## 5. Identity & Key Vault Security

**"8 out of 10 apps use System-Assigned Managed Identity, but Key Vault security is a mixed posture compared to DEV."**

- **Identity Gaps:** The two UUDRI apps have Managed Identity **disabled** (None).
- **RBAC:** We mapped a total of 9 direct RBAC role assignments across all QA Function App scopes.
- **Key Vault Security:** Unlike DEV (which was 100% Azure RBAC), QA has a mixed posture. While 5 out of 8 Key Vaults in the QA subscription have Azure RBAC enabled, the UUDRI QA vault (`UUDRI-Key-Vault-qa-02`) still relies on legacy Access Policies.

---

## 6. Alerting & Scheduled Verification

**"8 out of 10 apps route metric alerts to Teams, but the two UUDRI apps have no alert coverage."**

- **Metric Alerts:** 8 apps route Severity 3 `Http5xx` alerts to the **`ag-helios-qa-ops`** Action Group, which forwards to Microsoft Teams.
- **Scheduled Verification Gaps:** **0 out of 10 apps** have platform health check paths, and there are zero availability web tests set up.

---

## 7. CI/CD: Deploy on Merge & Promotion Gates

**"The QA deployment pipeline requires manual dispatch and environment gates, with no automated deploy-on-merge."**

- **Deploy on Merge:** Disabled (Manual Dispatch Only). Merging a PR into `main` runs syntax validation, but actual CD deployment requires a developer to manually dispatch the workflow via GitHub UI.
- **Promotion Gates:** Utilizes the GitHub `qa` environment block for manual approval gates, but the code package is rebuilt from scratch rather than promoting a pre-built artifact from DEV.

---

## 8. Capability Matrix — Final QA Position

| Capability | Status | Evidence |
|---|---|---|
| Deploy on merge | **Mapped (QA)** | Manual-only triggers for QA pipelines. |
| Promotion gates (dev > QA > prod) | **Mapped (QA)** | Separate repo workflow, manual dispatch, GitHub env approval gates. |
| Scheduled verification | **Mapped (QA)** | 0/10 health checks or ping tests. |
| Alerting | **Mapped (QA)** | 8/10 apps route metric alerts to `ag-helios-qa-ops`. |
| Observability + cost | **Mapped (QA)** | 8/10 App Insights configured, 0/10 diagnostic settings. |
| Reporting / audit | **Mapped (QA)** | Mapped Managed Identity, RBAC roles, and Key Vault access types. |

---

## 9. Next Steps

**"DEV and QA are done. The final step is running the discovery script for PROD."**

- Finalize the PROD SRE Discovery Report.
- Perform a final cross-environment comparison to highlight configuration drift (DEV → QA → PROD).

---

## If They Ask

| Question | Answer |
|---|---|
| Why does `UUDRI-Function-App-qa-01` have 0 functions? | It is a running app service site but has no code package currently registered or deployed. It is functionally idle. |
| Why is there VNet integration in QA but not DEV? | `ems-plan-narration-function-qa` has regional outbound VNet integration configured. This suggests QA is closer to our target prod architecture, or DEV is lagging behind. |
| Why is `buildPullRequest` present in QA but not DEV? | It indicates a release version mismatch between the DEV and QA code branches. SRE has logged this as an architectural delta. |

---

## The Numbers to Remember

```
10 apps, 28 functions (2 apps are empty)
16 functions in one app (SOP Factory)
All 10 apps have AlwaysOn = False (cold starts enabled)
1/10 has outbound VNet integration (ems-plan-narration-function-qa)
8/10 have App Insights, 2 don't (cost-ingestion & empty UUDRI app)
0/10 have diagnostic settings, 0/10 have health check paths
8/10 covered by metric alerts routing to ag-helios-qa-ops (UUDRI apps missing alerts)
QA is fully mapped — next: PROD
```
