# SRE Gap Remediation Plan — QA Environment

This plan details the SRE, Security, and Operational gap remediation steps for the **QA environment** (10 Function Apps / 28 Functions / 4 Logic Apps).

---

## 1. Key Gaps & Remediation Actions (QA-Specific)

### Gap 1: Key Vault Legacy Access Policies
*   **Gap:** 3 out of 8 Key Vaults in QA (`UUDRI-Key-Vault-qa-02`, `helios-qa-ui-kv`, and `helios-qa-spkplug-pki-kv`) still use legacy Access Policies instead of modern Azure RBAC.
*   **Action:** Convert these 3 Key Vaults to use **Azure RBAC permission model** using the Azure CLI command:
    ```bash
    az keyvault update --name <vault-name> --enable-rbac-authorization true
    ```
    Once enabled, configure RBAC roles (such as `Key Vault Secrets User`) for the function app managed identities.

### Gap 2: Disabled Managed Identities in QA
*   **Gap:** Managed Identity is disabled (`None`) on the two UUDRI apps in QA: `UUDRI-bill-processor-qa-01` and `UUDRI-Function-App-qa-01`.
*   **Action:** Enable **System-Assigned Managed Identity** for both apps:
    ```bash
    az functionapp identity assign --name <app-name> --resource-group <rg-name>
    ```
    This brings QA in alignment with the DEV environment security baseline.

### Gap 3: Observability Gaps on Idle & Cost Apps
*   **Gap:** 
    *   `helios-qa-cost-ingestion` lacks Application Insights.
    *   `UUDRI-Function-App-qa-01` (currently empty/idle) lacks Application Insights.
*   **Action:** Set `APPLICATIONINSIGHTS_CONNECTION_STRING` on both apps, linking them to their corresponding QA Application Insights resources to eliminate blind spots.

### Gap 4: Zero Diagnostic Settings (0 out of 10 Apps)
*   **Gap:** 0% platform diagnostic logging coverage.
*   **Action:** Enable diagnostic settings routing logs to their respective QA Log Analytics Workspaces (such as `uudri-log-analytics-qa` or `helios-qa-logs`).

### Gap 5: Null Health Checks & Availability Tests
*   **Gap:** 0/10 health paths configured. 0 App Insights availability ping tests.
*   **Action:**
    1. Deploy code modifications exposing anonymous `/health` endpoints.
    2. Configure `healthCheckPath = /health` on all 10 QA apps.
    3. Configure Azure Monitor URL Ping web tests targeting the public endpoint URLs.

### Gap 6: Public Exposure & No IP Restrictions
*   **Gap:** All 10 apps are open to the public internet. No IP security restrictions or private endpoints.
*   **Action:** 
    1. Integrate the remaining 9 QA apps with the QA outbound VNet subnet: `/subscriptions/663afada-2155-4c4d-b908-ac771ef2d133/resourceGroups/helios-qa-us-west3-rg/providers/Microsoft.Network/virtualNetworks/helios-aks-qa-vnet/subnets/appservice-subnet`.
    2. Configure Access Restrictions (IP security filtering) to permit inbound traffic only from the QA APIM gateway subnet or frontend IPs.

### Gap 7: Cold Starts (AlwaysOn Disabled on EP/Dedicated Plans)
*   **Gap:** AlwaysOn is disabled (`False`) for all 10 apps in QA, causing severe execution latency.
*   **Action:** Set `AlwaysOn = True` via CLI for all Dedicated and Premium plan function apps in QA.

---
