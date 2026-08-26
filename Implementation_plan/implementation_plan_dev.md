# SRE Gap Remediation Plan — DEV Environment

This document maps **every SRE, Security, and Operational Gap** identified in the consolidated gap analysis for the **DEV environment** (10 Function Apps / 30 Functions) to a concrete remediation action.

---

## 1. Key Gaps & Remediation Actions (DEV-Specific)

### Gap 1: Scope & Code Promotion Drift
*   **Gap:** Code is rebuilt from scratch per environment rather than promoting an immutable artifact. PR automation (`pipelineSmeQueue`) is in DEV but missing in PROD.
*   **Action:** Propose transitioning the CI/CD pipelines to a **build-once, promote-anywhere** model. The GitHub action will build the zip package once, attach it as a GitHub Release Asset or upload it to a storage container, and downstream stages (QA/PROD) will deploy that exact artifact without rebuilding.

### Gap 2: Unmanaged CLI ZipDeploy
*   **Gap:** 7 out of 10 Function Apps in DEV are deployed manually via developer workstation CLI commands, bypassing audit logs and code testing gates.
*   **Action:** Propose creating standardized GitHub Actions workflow templates for the remaining apps (SOP Factory, Ontology, Telemetry, and Activity Logger). These will run automated testing gates (`eslint` / `pytest`) and deploy using Service Principals, eliminating direct workstation deployments.

### Gap 3: Zero Diagnostic Settings (0/10 Apps)
*   **Gap:** No diagnostic settings configured. Platform scaling, audit, and host logs are lost.
*   **Action:** Configure Azure Monitor Diagnostic Settings on all 10 apps, routing logs to Log Analytics Workspaces:
    *   `uudri-bill-processor-dev` -> Workspace: `uudri-log-analytics-dev-01`
    *   `func-orchestrator-sopfactorydevmlel9` & `func-projector-sopfactorydevmlel9` -> Workspace: `log-sopfactory-dev`
    *   `ems-plan-narration-function` -> Workspace: `helios-ems-log-analytics-dev`
    *   Remaining 6 apps -> Workspace: `helios-dev-logs`

### Gap 4: Cost Ingestion Observability Blind Spot
*   **Gap:** `helios-dev-cost-ingestion` has no App Insights connection string, running blindly.
*   **Action:** Add the `APPLICATIONINSIGHTS_CONNECTION_STRING` app setting, linking it to the shared DEV resource `platform-backend-insights-dev`.

### Gap 5: Basic HTTP-Only Alerting & UUDRI Gaps
*   **Gap:** Alerts are limited to HTTP 5xx. UUDRI has no alerts. No alerts for Event Hubs, Service Bus, or Durable Function failures. Teams routing lacks on-call escalation (PagerDuty).
*   **Action:**
    1. Deploy Metric Alerts for HTTP 5xx on `uudri-bill-processor-dev` and `UUDRI-Bill-Processor-dev-01`.
    2. Deploy Scheduled Query (Log) Alerts to monitor:
       *   **Durable Orchestrator Exceptions:** KQL query on `AppRequests` / `AppExceptions` for orchestrator failure events.
       *   **Event Hub / Service Bus Health:** Alert on dead-lettered message counts or ingestion drops.
    3. Add a webhook integration to the Teams Action Group for on-call alerting.

### Gap 6: Null Health Checks & Lack of Web Tests
*   **Gap:** 0/10 health paths configured. 0 availability tests active.
*   **Action:**
    1. Modify the Python, Node, and C# codebases of apps lacking HTTP endpoints to include a simple `/health` anonymous endpoint.
    2. Configure `healthCheckPath = /health` on all 10 DEV apps.
    3. Deploy Azure Monitor URL Ping Availability Web Tests targeting the public health paths.

### Gap 7: Public Network Exposure & No VNet Integration
*   **Gap:** All apps are public, with no inbound IP restrictions, and only 1 app has outbound VNet integration.
*   **Action:**
    1. Configure outbound VNet Integration for the remaining 9 DEV apps to route internal traffic through the DEV virtual network.
    2. Apply IP Security Restrictions (Access Restrictions) to permit inbound traffic only from the DEV APIM Gateway subnet or specific frontend CIDRs, blocking direct public internet access.

### Gap 8: Cold Start Risks (AlwaysOn Disabled)
*   **Gap:** AlwaysOn is disabled on 8 apps running on Dedicated/Premium plans, causing latencies.
*   **Action:** Enable `AlwaysOn = True` via ARM/CLI for all apps hosted on Premium (EP) or Dedicated (S) plans in DEV.

---
