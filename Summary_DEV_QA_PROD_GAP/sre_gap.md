# Azure Functions — Consolidated SRE Gap Analysis (DEV vs. QA vs. PROD)

This report consolidates all identified **SRE, Security, and Operational Gaps** across the DEV, QA, and Production environments for our Azure Functions estate. It provides a detailed comparison to guide our engineering decisions and upcoming implementation planning.

---

## 1. Scope and Environment Drift (Branch & Code Sync)

There is a significant mismatch in the services, functions, and active codebases deployed across our three environments, indicating **code promotion and branch drift** in our release pipelines.

### Environment Scope Overview
*   **DEV:** 10 Function Apps / 30 Functions / 3 Logic Apps
*   **QA:** 10 Function Apps / 28 Functions / 4 Logic Apps (2 apps are empty/idle)
*   **PROD:** 7 Function Apps / 25 Functions / 3 Logic Apps (0 empty/idle apps)

### Identified Gaps & Risks
| Component / Gap | DEV | QA | PROD | Impact & Risk / Named Missing Resources |
|:---|:---:|:---:|:---:|:---|
| **Missing Production Apps** | Present | Present (2 idle) | **Absent** | **Missing Apps in PROD:**<br>* `uudri-bill-processor-dev` / `UUDRI-bill-processor-qa-01`<br>* `UUDRI-Bill-Processor-dev-01` / `UUDRI-Function-App-qa-01`<br>* `helios-device-telemetry-dev-func` / `helios-device-telemetry-qa-func`<br>*Risk:* These services are completely absent in Production. Requires team alignment on whether they are legacy, pending deployment, or retired. |
| **`realized_kpi_listener`** | Deployed | Deployed | **Missing** | **Missing in `ems-plan-narration-function-prod`:**<br>The Event Hub-triggered `realized_kpi_listener` function exists in DEV and QA but is absent in PROD, preventing KPI event ingestion in Production. |
| **SOP Factory Orchestration** | 16 Functions | 16 Functions | **15 Functions** | **Missing Functions in `func-orchestrator-sopfactoryprod1fd3k`:**<br>* `pipelineSmeQueue` (present only in DEV)<br>* `buildPullRequest` (present only in QA)<br>*Risk:* PROD lacks both functions, indicating pull-request automation and SME queue endpoints were never promoted. |
| **Weather Monitor Logic App** | Deployed | Deployed | **Missing** | **Missing in PROD:**<br>`helios-data-weather-eh-eventstream-monitor-prod`<br>*Risk:* The weather eventstream monitoring workflow exists in DEV/QA but was never deployed to Production. |

---

## 2. CI/CD and Release Orchestration

Our current deployment models introduce building inconsistencies, lack automation for early environments, and rely heavily on manual dispatching.

| SRE Matrix Column | DEV | QA | PROD | Identified Gap & Risk |
|:---|:---:|:---:|:---:|:---|
| **Deploy on Merge** | **Partial** | **No** (Manual Only) | **No** (Manual Only) | Code merged into `main` only triggers validation builds. Continuous Deployment (CD) requires a developer to manually trigger the workflow via `workflow_dispatch` for QA and PROD. |
| **Promotion Gates** | **No** | **Partial** | **Partial** | * Deployments rebuild the code from the branch from scratch for each environment rather than promoting a verified, pre-built, immutable artifact.<br>* Gates rely on GitHub Environment approvals rather than automated status checks. |
| **Inspection Scope** | **Partial** | **Partial** | **Partial** | Detailed pipeline configurations were only fully verified for 1 repository (`helios-plan-narration-backend`). The remaining apps are unmapped at the repo level. |

### 2.1 The CI/CD Paradox: Structured GitHub Actions vs. Unmanaged CLI ZipDeploy

A major operational inconsistency identified is that **only one Function App** (`ems-plan-narration-function` / `ems-plan-narration-function-qa` / `ems-plan-narration-function-prod`) is deployed via a structured, repository-driven CI/CD workflow from the `helios-plan-narration-backend` repository. The rest of the Function Apps in all three environments are deployed using either legacy tools or unmanaged workstation scripts.

#### How Other Apps are Deployed:
1.  **Legacy Azure DevOps (VSTSRM):** The `UUDRI` apps are configured on legacy Azure DevOps Release Management pipelines.
2.  **Unmanaged CLI ZipDeploy (`az cli`):** Key core systems like the `SOP Factory Orchestrator`, `Ontology Event Processor`, `Device Telemetry`, and `GitHub Activity Logger` are deployed using direct developer workstation CLI commands (`az functionapp deployment source config-zip` / local IDE extensions).

#### Why This Discrepancy Exists:
*   **Decentralized Development Teams:** The development of our Azure Functions estate is fragmented across multiple teams. The AI Plan Narration team adopted modern GitHub Actions workflows, while other teams stuck to legacy Azure DevOps setups or direct CLI deployments.
*   **Ad-Hoc Tooling:** Several applications (such as `cost-ingestion` or `github-activity-logger`) were written as small utility programs and never integrated into a unified organizational platform pipeline template.

#### Critical Risks of Manual CLI ZipDeploy:
*   **Auditability & Accountability Risk:** There is no central build/deployment audit trail indicating who pushed which version of the code, when it was pushed, or what commit SHA is currently running in QA or Production.
*   **Lack of Testing Gates:** Manual CLI deployments bypass code testing gates (e.g. `pytest` or `eslint` checks), introducing potential execution errors directly into QA and PROD environments.
*   **Environment Configuration Drift:** CLI ZipDeploy depends on the local environment settings, OS, and package dependencies of individual developer workstations, leading to "works on my machine" deployment failures.

---

## 3. Observability and Cost Management

Observability is a major gap across the entire estate, with a total lack of platform scaling logs and missing telemetry on cost tracking services.

### Identified Gaps & Risks
*   **Zero Diagnostic Settings (0 out of 27 Apps):** **None** of the Function Apps in DEV, QA, or PROD have Azure Monitor diagnostic settings configured. Centralized platform logs, host logs, scale controller logs, and audit trails are completely lost.
*   **Cost Ingestion App Telemetry Gap:** `helios-dev/qa/prod-cost-ingestion` completely lacks Application Insights configurations. The daily scheduled cost-ingestion job runs with zero telemetry or transaction tracking.
*   **App Insights Configuration Gaps:** The empty site `UUDRI-Function-App-qa-01` has no Application Insights configuration, leaving it unmonitored while active.

---

## 4. Alerting and Incident Response

While metric alert coverage is high in Production, the rules themselves do not verify trigger-level or orchestration-level health.

### Identified Gaps & Risks
*   **UUDRI Alert Gaps (DEV & QA):** The UUDRI bill processor apps lack metric alerts entirely in both DEV and QA.
*   **Standard Alert Limitations (HTTP Only):** Metric alerts are configured exclusively to check for high-level HTTP `Http5xx` counts. 
*   **Lack of Trigger-Specific Alerts:** There are no dedicated alert rules to monitor:
    *   **Event Hubs / Service Bus:** Partition failures, consumer group lag, dead-letter queues (DLQ), or message consumption drops.
    *   **Durable Functions:** Activity timeouts, orchestrator failures, or Durable Task system exceptions.
*   **Incident Routing Gaps:** Alert rules route notifications via Webhooks to MS Teams channels (`ag-helios-qa-ops` / `ag-helios-prod-ops`). There is no integration with on-call rotation tooling (e.g., PagerDuty, OpsGenie) or documented escalation pathways.

---

## 5. Scheduled Verification and Health Monitoring

There is currently **no automated validation** of system uptime or functional availability.

### Identified Gaps & Risks
*   **Null Health Check Paths (0 out of 27 Apps):** Platform health check paths (`healthCheckPath`) are not configured on any Function App. Azure's App Service infrastructure cannot perform automatic instances recycling when hosts become unhealthy.
*   **Zero Availability Tests (0 out of 27 Apps):** No Application Insights ping tests or multi-step web tests are configured to verify that external HTTP endpoints are responsive.

---

## 6. Identity and Security Posture

Our security model represents a mixed security posture, with wide public network exposure and legacy access mechanisms in QA and PROD.

### Identified Gaps & Risks
*   **Legacy Access Policies (Key Vaults):** While DEV utilizes Azure RBAC for all 4 Key Vaults, QA and PROD have a mixed posture:
    *   **QA:** 3 out of 8 Key Vaults (`UUDRI-Key-Vault-qa-02`, `helios-qa-ui-kv`, and `helios-qa-spkplug-pki-kv`) still use legacy, less-granular Access Policies.
    *   **PROD:** 2 out of 8 Key Vaults (`helios-prod-ui-kv` and `helios-prd-spkplg-pki-kv`) use legacy Access Policies.
*   **Managed Identity Disabled in QA:** The two UUDRI apps in QA have Managed Identity disabled (`None`), unlike their DEV counterparts.
*   **Wide Inbound Public Exposure:** **All 27 Function Apps** are fully exposed to the public internet (`PublicNetworkAccess = Enabled`) with **zero** Private Endpoints or IP restriction rules in place.
*   **Outbound Network Integration Gap:** Outbound VNet integration is configured **only** for `ems-plan-narration-function` in all environments. The remaining 24 apps have no access to private resources inside our Virtual Networks.

---

## 7. Reliability and Performance (Cold Starts)

*   **Always On Disabled (QA & PROD):** `AlwaysOn` is disabled (`False`) for **all 10 apps in QA** and **all 7 apps in PROD** (it is only enabled on the Bill Processor apps in DEV).
*   **Impact:** Any app running on a Dedicated/Premium App Service Plan that is inactive will experience idle timeout shutdowns, leading to severe cold starts (latencies up to 10-15 seconds) when next triggered in QA and PROD.
