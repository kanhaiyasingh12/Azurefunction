# SRE Gap Remediation Plan — Production Environment

This plan details the SRE, Security, and Operational gap remediation steps for the **Production environment** (7 Function Apps / 25 Functions / 3 Logic Apps).

---

## 1. Key Gaps & Remediation Actions (PROD-Specific)

### Gap 1: Critical Service and Code Mismatches (Branch Drift)
*   **Gap:** 
    *   **Missing Apps:** The `UUDRI` bill processor apps and `helios-device-telemetry-func` are completely missing in Production, meaning these workloads are only active in DEV and QA.
    *   **Missing Function (`realized_kpi_listener`):** The `realized_kpi_listener` Event Hub trigger function exists in DEV and QA but is missing in `ems-plan-narration-function-prod`, preventing KPI data collection.
    *   **Missing SOP Orchestrator Functions:** `pipelineSmeQueue` and `buildPullRequest` are missing in the PROD orchestrator app, indicating that pull-request automation and queue features were never promoted.
    *   **Missing Weather Monitor:** `helios-data-weather-eh-eventstream-monitor-prod` Logic App is missing.
*   **Action:** 
    1. Align with product teams to determine whether the missing UUDRI and Device Telemetry services should be promoted to Production.
    2. Update the PROD deployment workflows to promote the correct branch and trigger deployment of the missing functions (`realized_kpi_listener`, `pipelineSmeQueue`, etc.).
    3. Deploy the Weather Monitor Logic App to PROD using the ARM template configuration from QA.

### Gap 2: Key Vault Legacy Access Policies
*   **Gap:** 2 out of 8 Key Vaults (`helios-prod-ui-kv` and `helios-prd-spkplg-pki-kv`) still use legacy Access Policies instead of modern Azure RBAC.
*   **Action:** Convert both Key Vaults to use **Azure RBAC permission model** using the Azure CLI command:
    ```bash
    az keyvault update --name <vault-name> --enable-rbac-authorization true
    ```
    Once enabled, configure RBAC roles (such as `Key Vault Secrets User`) for the PROD function app managed identities.

### Gap 3: Observability Gaps on Cost App
*   **Gap:** `helios-prod-cost-ingestion` lacks Application Insights.
*   **Action:** Set `APPLICATIONINSIGHTS_CONNECTION_STRING` on the app, linking it to the PROD Application Insights resource to eliminate cost-tracking blind spots.

### Gap 4: Zero Diagnostic Settings (0 out of 7 Apps)
*   **Gap:** 0% platform diagnostic logging coverage.
*   **Action:** Enable diagnostic settings routing host, scale controller, and platform logs to the production Log Analytics Workspace (such as `helios-prod-logs`).

### Gap 5: Null Health Checks & Availability Tests
*   **Gap:** 0/7 health paths configured. 0 App Insights availability ping tests.
*   **Action:**
    1. Deploy code modifications exposing anonymous `/health` endpoints.
    2. Configure `healthCheckPath = /health` on all 7 PROD apps.
    3. Configure Azure Monitor URL Ping web tests targeting the public endpoint URLs.

### Gap 6: Public Exposure & Outbound VNet Integration
*   **Gap:** All 7 apps are open to the public internet. No IP security restrictions or private endpoints. Outbound VNet integration is configured only for `ems-plan-narration-function-prod` (subnet: `/subscriptions/9b9e9af9-5917-4cae-88b4-1304f3ea98b4/resourceGroups/helios-prod-us-west3-rg/providers/Microsoft.Network/virtualNetworks/helios-aks-prod-vnet/subnets/appservice-subnet`).
*   **Action:** 
    1. Integrate the remaining 6 PROD apps with the outbound VNet subnet.
    2. Apply IP Security Restrictions (Access Restrictions) to permit inbound traffic only from the PROD APIM gateway subnet or specific frontends.

### Gap 7: Cold Starts (AlwaysOn Disabled on EP/Dedicated Plans)
*   **Gap:** AlwaysOn is disabled (`False`) for all 7 apps in PROD, causing execution latency on critical production workloads.
*   **Action:** Set `AlwaysOn = True` via CLI for all Dedicated and Premium plan function apps in PROD.

---
