# Meeting Presenter Notes — Azure Functions PROD Discovery 

> Read this like a story. Each section is one beat. Say the one-liner, then use the bullets if they ask for detail.

---

## The Story in 30 Seconds

> I have successfully completed the SRE discovery for the **Production Environment** (Subscription: `Helios - Production`). Production has a significantly different footprint compared to DEV and QA: it runs **only 7 Function Apps** and **25 functions** (down from 10/28 in QA and 10/30 in DEV).
---

## 1. What I Accomplished (Did)

**"I added the Azure Function production discovery SRE report for our ADO Wiki."**

- Published the ADO Wiki-compliant SRE report [**Azure_Functions_PROD_SRE_Discovery_Report.md**](file:///c:/Users/Dipak.Singh/Documents/antigravity/Azure%20Function/PROD-Discovery/Azure_Functions_PROD_SRE_Discovery_Report.md).

---

## 2. Scope & Runtimes — Scope Gaps

**"I Found 7 Function apps with 25 functions in PROD — and several major components are completely absent."**

- **Missing Services:** The two UUDRI utility apps and the `helios-device-telemetry` app **do not exist** in the Production subscription.
- **Cold Starts:** Like QA, **AlwaysOn is False for all 7 apps in PROD**, so every app experiences cold starts.
- **Runtimes:** Python 3.11/3.12, Node.js 20, and .NET isolated.

---

## 3. Observability & Alerting Gaps

**"Alerting coverage is 100% in PROD, but platform diagnostics are completely missing."**

- **Alert Coverage:** All 7 apps (100%) route `Http5xx` metric alerts to the Action Group **`ag-helios-prod-ops`** (teams/webhook channel).
- **Observability Gaps:** `helios-prod-cost-ingestion` completely lacks App Insights telemetry. **0 out of 7** apps have diagnostic settings configured (meaning no platform, scaling, or host logs are sent to Log Analytics).
- **Alerting & Verification Limitations:** Alerts are restricted to basic HTTP 5xx counts. There are **no queue-specific alerts** (Event Hubs/Service Bus backpressure, lag, or DLQ) and **no Durable Functions alerts** (orchestrator or activity-specific failures). Additionally, **0 out of 7** apps have health check paths or availability ping tests configured.


---

## 4. Security & Key Vault Posture

**"All 7 Function apps run on System-Assigned Managed Identity, but Key Vault security is a mixed posture."**

- **Managed Identities & Roles:** Mapped all 7 function apps to use System-Assigned Managed Identity, with a total of 14 direct RBAC role assignments successfully verified across target production resource scopes.
- **Key Vault Models:** 6 out of 8 Key Vaults in the PROD subscription use Azure RBAC. However, the UI vault (`helios-prod-ui-kv`) and Sparkplug PKI vault (`helios-prd-spkplg-pki-kv`) still use legacy Access Policies.
- **Network Security:** All 7 apps are fully public to the internet with zero IP restrictions. Only `ems-plan-narration-function-prod` has outbound VNet integration.


---
## 5. CI/CD: Strict Manual Gates

**"The production deployment workflow is strictly manual, with zero automated triggers."**

- **Manual Trigger Only:** The PROD workflow ([**deploy-function-app-prod.yml**](file:///c:/Users/Dipak.Singh/Documents/antigravity/Azure%20Function/helios-plan-narration-backend/.github/workflows/deploy-function-app-prod.yml)) runs *only* on manual `workflow_dispatch`. There are no push or pull request triggers configured.
- **Runner & Gates:** Runs on `arc-runner-set-prod` and utilizes the GitHub `prod` environment for mandatory approval gates.
- **No Immutable Promotion:** Rebuilds code from scratch rather than promoting a tested QA package.
- **Diversified Deployment Methods:** Only 1 app (`ems-plan-narration-function-prod`) uses a repository-mapped GitHub Actions workflow. `kg-event-processor-prod` is deployed via manual CLI ZipDeploy, while the other 5 apps have no SCM configuration, indicating direct manual workstation deployments.
---

## 6. Structural & Function-Level Deltas

**"PROD is running a subset of functions compared to DEV and QA, indicating branch sync drift."**

- **Plan Narration Delta:** `ems-plan-narration-function-prod` runs only **1 function** (`plan_narration_agent`). The `realized_kpi_listener` function present in DEV/QA is missing in PROD.
- **SOP Factory Orchestrator:** Runs **15 functions** (lacking both `pipelineSmeQueue` from DEV and `buildPullRequest` from QA).
- **Logic Apps Delta:** We have **only 3 workflow apps** in PROD. The weather eventstream monitor (`helios-data-weather-eh-eventstream-monitor`) present in DEV/QA does not exist in PROD.
---

## If They Ask

| Question | Answer |
|---|---|
| Why are UUDRI and telemetry apps missing in PROD? | They are either legacy components that are being retired, or they are not approved/deployed to the production landing zones yet. |
| Why is `realized_kpi_listener` missing in PROD? | It suggests a deployment mismatch where code for the KPI listener has not been promoted or activated in the production branch. |
| What's the impact of `AlwaysOn = False` in PROD? | High cold start latencies (up to 10-15 seconds) when functions are invoked after periods of inactivity. |
| Why are there no queue-specific or orchestration-specific alerts? | Currently, our monitoring only checks high-level HTTP 5xx errors. Alert rules for Event Hubs, Service Bus, and Durable Functions pipelines (e.g., activity timeouts, queue backpressure) need to be configured in the next phase. |
| Why is there a mixed authorization posture on Key Vaults? | Legacy vaults (`helios-prod-ui-kv` and `helios-prd-spkplg-pki-kv`) still use legacy Access Policies, but active backend vaults (6 out of 8) use Azure RBAC in accordance with modern security standards. |
| What's next? | With DEV, QA, and PROD fully mapped, I have completed the **Release Orchestration Capability Matrix** for Azure Functions! |

---

## The Numbers to Remember (PROD)

```
7 apps, 25 functions (UUDRI and telemetry apps are absent)
All 7 apps have AlwaysOn = False (cold starts enabled)
1/7 has outbound VNet integration (ems-plan-narration-function-prod)
6/7 have App Insights, 1 doesn't (cost-ingestion)
0/7 have diagnostic settings, 0/7 have health check paths
7/7 covered by metric alerts routing to ag-helios-prod-ops (100% alert coverage)
Only 1 app uses GitHub Actions (manual dispatch); others use manual CLI/ZipDeploy
14 RBAC role assignments across 7 system-assigned managed identities
```
