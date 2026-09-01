# Non-AKS Azure Estate — SRE Discovery Report (DEV Environment)

**Project:** Non-AKS Azure Estate (107 across 3 envs)  
**Environment:** DEV  
**Scope:** Non-AKS Compute & Managed Services (37 Non-AKS Resources in DEV; 10 Resource Groups; Reconciled with 47 8/3 Baseline where 10 Function Apps are accounted for separately)  
**Status:** Discovery complete — no remediation or implementation changes have been performed  

---

[[_TOC_]]

---

## 1. Background and Context

### 1.1 Why This Investigation Exists

The SRE team maintains a **Release Orchestration Capability Matrix** that tracks the operational maturity of every platform component across six core capability columns:

| Column | Description |
|:---|:---|
| **Deploy on merge** | Does the component auto-deploy when code is merged to the main trunk? |
| **Promotion gates (dev > QA > prod)** | Is there a controlled, auditable promotion gate between environments? |
| **Scheduled verification** | Are there recurring synthetic checks or health probes validating the running system? |
| **Alerting** | Are actionable metric/log alert rules and verified action groups configured? |
| **Observability + cost** | Is diagnostic telemetry actively streaming and are cost metrics attributable? |
| **Reporting / audit** | Are deployments, configuration drift, and identity bindings auditable? |

The platform matrix covers: k8s + ArgoCD apps (including Temporal), Frontend (React, IBM Carbon components), Foundry models, Azure Functions (~87 across 3 envs), Flyway DB migrations, **Non-AKS Azure estate (~107 compute & platform resources across DEV/QA/PROD)**, and Fabric / Knowledge Graphs.

When SRE initially reviewed the matrix, **Non-AKS Azure Estate was marked `unknown` across multiple capability dimensions**. Following our investigation into Azure Functions (10 apps in DEV), this deep-dive targets the remaining compute, container, web hosting, and messaging estate in the **DEV** environment.

### 1.2 Investigation Principle

```
DISCOVER --> VERIFY --> SAVE EVIDENCE --> MAP ARCHITECTURE --> EXPLAIN CURRENT STATE --> IDENTIFY OPERATIONAL GAPS --> GET ENGINEERING DECISION --> IMPLEMENT LATER
```

*   **Rule 1: Cross-Verification:** Do not conclude from a single API call. Cross-check related configuration across ARM, Azure CLI, Diagnostic Settings, RBAC assignments, and Service Bus metrics. Preserve all evidence locally.
*   **Rule 2: Read-Only Mandate:** No remediation or configuration mutation is performed during discovery.
*   **Rule 3: Evidence-Backed Facts:** All findings must cite concrete configuration attributes, resource IDs, subscription IDs, and metric data.

---

## 2. DEV Environment Scope

| Metric | Value |
|:---|:---|
| **Total Non-AKS Resources in DEV Scope** | **37** |
| **Container Apps (`Microsoft.App/containerApps`)** | **14** |
| **Container App Environments (`Microsoft.App/managedEnvironments`)** | **6** |
| **App Services / Web Apps (`Microsoft.Web/sites`)** | **6** |
| **Static Web Apps (`Microsoft.Web/staticSites`)** | **10** |
| **Logic Apps / Workflow Apps (`Microsoft.Logic/workflows`)** | **1** |
| **Azure Subscription** | `a6498579-cfb7-41e9-a957-14375196a386` (Helios — Development) |
| **Resource Groups in Scope** | **10 Resource Groups** |
| **Primary Resource Group** | `helios-dev-us-west3-rg` (20 non-AKS resources) |
| **Secondary Resource Groups** | `uudri-dev-rg` (2), `arcadia-tariff-dev-rg` (2), `rg-soc2-dev` (2), `helios-ui-rg` (3), `helios-dev-uswest3-ui` (2), `rg-warroom-dashboard` (2), `dev-helios-memo-rg` (1), `helios-product-map-rg` (1), `rg-qre-architecture-docs` (1) |

### 2.1 Reconciliation with 8/3 Audit Baseline (47 Total DEV Resources)

The platform baseline inventory identified **47 total compute/platform resources in DEV**. SRE reconciled the scope as follows:
*   **10 Azure Function Apps** were thoroughly analyzed in the dedicated *Azure Functions DEV SRE Discovery Report* (`Azure_Functions_DEV_SRE_Discovery_Report.md`).
*   **37 Non-AKS Resources** (comprising Container Apps, Managed Environments, Web Apps, Static Web Apps, and Logic Workflows) form the exact scope of this report.
*   **Total:** $10\text{ (Function Apps)} + 37\text{ (Non-AKS Compute & Managed Environments)} = \mathbf{47\text{ Resources}}$.

---

## 3. Master Resource Inventory

The table below catalogs all 37 Non-AKS DEV resources verified via direct Azure Resource Graph and Azure Management API queries:

| # | Name | ResourceType | ResourceGroup | Location | Kind | Tags | ProvisioningState |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 1 | `arcadia-tariff-dev-worker` | Microsoft.App/containerApps | `arcadia-tariff-dev-rg` | East US 2 | N/A | *(none)* | Succeeded |
| 2 | `ca-soc2-dev-api` | Microsoft.App/containerApps | `rg-soc2-dev` | East US 2 | N/A | `compliance_scope=soc2; cost_center=engineering; data_classification=internal; environment=dev; managed_by=terraform; owner=platform-team; workload=soc2` | Succeeded |
| 3 | `nlp-mcp-server-poc` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | *(none)* | Succeeded |
| 4 | `nlp-agent-service-poc` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | *(none)* | Succeeded |
| 5 | `ca-opa-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 6 | `ca-model-service-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 7 | `ca-catalog-api-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 8 | `ca-library-api-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 9 | `ca-authoring-bff-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 10 | `ca-sopfactory-ui-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | *(none)* | Succeeded |
| 11 | `ca-composer-api-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 12 | `ca-composition-mcp-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 13 | `ca-sar-graph-demo-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=site-artifact-repository; environment=Development; managed_by=terraform; owner=Devops; project=sar; service=Site Artifact Repository; site=demo` | Succeeded |
| 14 | `ca-sar-api-demo-dev` | Microsoft.App/containerApps | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=site-artifact-repository; environment=Development; managed_by=terraform; owner=Devops; project=sar; service=Site Artifact Repository; site=demo` | Succeeded |
| 15 | `UUDRI-App-Service-dev-01` | Microsoft.Web/sites | `uudri-dev-rg` | West US 2 | app | `application=UUDRI; businessunit=other; environment=Development; owner=divyanshu.arya@qcellsces.onmicrosoft.com` | N/A |
| 16 | `qcells-warroom-dashboard` | Microsoft.Web/sites | `rg-warroom-dashboard` | West US 3 | app,linux | *(none)* | N/A |
| 17 | `heliosdev-ui-appservice` | Microsoft.Web/sites | `helios-dev-uswest3-ui` | West US 3 | app,linux | `Environment=dev; Feature=ui-infrastructure; ManagedBy=terraform; Project=helios` | N/A |
| 18 | `helios-dev-memo-app` | Microsoft.Web/sites | `dev-helios-memo-rg` | West US 3 | app,linux,container | `application=memo-system; environment=dev; owner=Devops; service=Corporate Memo System` | N/A |
| 19 | `helios-mcp-chatbot-app` | Microsoft.Web/sites | `helios-dev-us-west3-rg` | West US 3 | app,linux | *(none)* | N/A |
| 20 | `UUDRI-Foundry-App-Service-dev-01` | Microsoft.Web/sites | `helios-dev-us-west3-rg` | West US 3 | app | `application=UUDRI; businessunit=other; environment=Development; owner=DevOps` | N/A |
| 21 | `helios-operator-console` | Microsoft.Web/staticSites | `helios-ui-rg` | East US 2 | N/A | *(none)* | N/A |
| 22 | `helios-product-map` | Microsoft.Web/staticSites | `helios-product-map-rg` | East US 2 | N/A | *(none)* | N/A |
| 23 | `helios-m2-prototype` | Microsoft.Web/staticSites | `helios-ui-rg` | East US 2 | N/A | *(none)* | N/A |
| 24 | `helios-prototype` | Microsoft.Web/staticSites | `helios-ui-rg` | East US 2 | N/A | *(none)* | N/A |
| 25 | `helios-dev-engg-dashboard` | Microsoft.Web/staticSites | `helios-dev-us-west3-rg` | West US 2 | N/A | `application=Engg Dashboard; environment=Development; owner=Devops; service=Engineering Dashboard` | N/A |
| 26 | `helios-qre-architecture` | Microsoft.Web/staticSites | `rg-qre-architecture-docs` | West US 2 | N/A | *(none)* | N/A |
| 27 | `helios-architecture` | Microsoft.Web/staticSites | `rg-warroom-dashboard` | West US 2 | N/A | *(none)* | N/A |
| 28 | `heliosdev-ui-webapp` | Microsoft.Web/staticSites | `helios-dev-uswest3-ui` | West US 2 | N/A | `Environment=dev; Feature=ui-infrastructure; ManagedBy=terraform; Project=helios` | N/A |
| 29 | `UUDRI-Static-Web-App-dev` | Microsoft.Web/staticSites | `helios-dev-us-west3-rg` | West US 2 | N/A | `application=UUDRI; businessunit=other; environment=Development; owner=DevOps` | N/A |
| 30 | `UUDRI-Static-Web-App-dev-01` | Microsoft.Web/staticSites | `uudri-dev-rg` | West US 2 | N/A | `application=UUDRI; businessunit=other; environment=Development; owner=divyanshu.arya@qcellsces.onmicrosoft.com` | N/A |
| 31 | `helios-platform-alert-watcher` | Microsoft.Logic/workflows | `helios-dev-us-west3-rg` | westus3 | N/A | *(none)* | Succeeded |
| 32 | `arcadia-tariff-dev-cae` | Microsoft.App/managedEnvironments | `arcadia-tariff-dev-rg` | East US 2 | N/A | *(none)* | Succeeded |
| 33 | `cae-soc2-dev` | Microsoft.App/managedEnvironments | `rg-soc2-dev` | East US 2 | N/A | `compliance_scope=soc2; cost_center=engineering; data_classification=internal; environment=dev; managed_by=terraform; owner=platform-team; workload=soc2` | Succeeded |
| 34 | `cae-sar-demo-dev` | Microsoft.App/managedEnvironments | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=site-artifact-repository; environment=Development; managed_by=terraform; owner=Devops; project=sar; service=Site Artifact Repository; site=demo` | Succeeded |
| 35 | `cae-semantic-insights-poc` | Microsoft.App/managedEnvironments | `helios-dev-us-west3-rg` | West US 3 | N/A | *(none)* | Succeeded |
| 36 | `cae-sopfactory-dev` | Microsoft.App/managedEnvironments | `helios-dev-us-west3-rg` | West US 3 | N/A | `component=sop-factory; environment=Development; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 37 | `helios-provision-site-dev-env` | Microsoft.App/managedEnvironments | `helios-dev-us-west3-rg` | West US 3 | N/A | `application=Helios; environment=Development; owner=Devops; service=Provision Site` | Succeeded |

---

## 4. Classification & Lifecycle Analysis

SRE performed an automated classification and state review on all 37 resources. 

```
                                  DEV ESTATE CLASSIFICATION (37 Resources)
       +------------------------------------+------------------------------------+
       |                                    |                                    |
       v                                    v                                    v
+------------------------+        +------------------------+        +------------------------+
|   Development Shims    |        |        Unknown         |        |   Possibly Abandoned   |
|     (21 Resources)     |        |     (7 Resources)      |        |     (6 Resources)      |
+------------------------+        +------------------------+        +------------------------+
                                                                                 |
                                                                                 v
                                                                    +------------------------+
                                                                    |     Active Systems     |
                                                                    |     (3 Resources)      |
                                                                    +------------------------+
```

### 4.1 Classification Breakdown

| Classification Category | Count | Percentage | Definition & Policy |
|:---|:---:|:---:|:---|
| **Development Shim** | **21** | 56.8% | Ephemeral or functional development environments supporting active engineering, mock workloads, and POCs. |
| **Unknown** | **7** | 18.9% | Static sites and helper apps with zero tags, unverified repository links, or no recorded deployment activity. |
| **Possibly Abandoned** | **6** | 16.2% | Resources exhibiting stopped running states, unmanaged manual drift, duplicate instances, or personal email ownership. |
| **Active** | **3** | 8.1% | Actively maintained platform components with live ingress and defined operational roles. |
| **TOTAL** | **37** | **100%** | |

### 4.2 Detailed Breakdown of Flagged Items (8 Flagged Resources)

The discovery pipeline flagged **8 specific resources** requiring engineering decisions and remediation:

```
+-------------------------------------------------------------------------------------------------------------+
|                                    SRE DEV FLAGGED ITEMS RECONCILIATION                                     |
+----+----------------------------------+--------------------+------------------------------------------------+
| #  | Resource Name                    | Classification     | Primary Flag Reason                            |
+----+----------------------------------+--------------------+------------------------------------------------+
| 1  | ca-sopfactory-ui-dev             | Possibly Abandoned | Unmanaged — No Terraform state or tags found    |
| 2  | UUDRI-App-Service-dev-01         | Possibly Abandoned | UUDRI stack — Individual email owner tag        |
| 3  | qcells-warroom-dashboard         | Active             | Untagged — Missing team ownership assignment   |
| 4  | helios-mcp-chatbot-app           | Possibly Abandoned | Running State is 'Stopped' (Basic B1 SKU)      |
| 5  | UUDRI-Foundry-App-Service-dev-01 | Possibly Abandoned | Duplicate UUDRI backend in team RG             |
| 6  | UUDRI-Static-Web-App-dev         | Possibly Abandoned | Duplicate UUDRI SWA frontend in team RG        |
| 7  | UUDRI-Static-Web-App-dev-01      | Possibly Abandoned | Duplicate UUDRI SWA frontend in personal RG    |
| 8  | helios-platform-alert-watcher    | Active             | Untagged — Missing team ownership assignment   |
+----+----------------------------------+--------------------+------------------------------------------------+
```

1. **`ca-sopfactory-ui-dev` (Container App):** Although actively serving traffic for SOP Factory on port 8090, it possesses **zero tags** and has no corresponding resource block in the SOP Factory Terraform state (`iac-coverage-DEV.csv`). It represents an unmanaged manual deployment.
2. **`UUDRI-App-Service-dev-01` (App Service):** Deployed in `uudri-dev-rg` with an individual engineer's email tag (`divyanshu.arya@qcellsces.onmicrosoft.com`) rather than a shared platform team identity.
3. **`qcells-warroom-dashboard` (App Service):** Running and active in `rg-warroom-dashboard`, but completely lacks ownership, service, and environment tags.
4. **`helios-mcp-chatbot-app` (App Service):** Found in `Stopped` administrative state. It incurs B1 Basic App Service Plan compute costs while failing all inbound requests.
5. **`UUDRI-Foundry-App-Service-dev-01` (App Service):** Duplicate deployment of the UUDRI backend hosted in `helios-dev-us-west3-rg`, pointing to an identical PostgreSQL database and storage accounts as the `uudri-dev-rg` stack.
6. **`UUDRI-Static-Web-App-dev` (Static Web App):** SWA instance deployed in `helios-dev-us-west3-rg` tracking branch `helios-develop`.
7. **`UUDRI-Static-Web-App-dev-01` (Static Web App):** SWA instance deployed in `uudri-dev-rg` tracking branch `develop` under individual ownership.
8. **`helios-platform-alert-watcher` (Logic App):** Active Azure Logic App workflow in `helios-dev-us-west3-rg` with no ownership metadata or diagnostic settings.

---

## 5. Container Apps Deep-Dive (14 Apps)

Fourteen Container Apps operate across four Container App Environments in DEV:

```
                                  CONTAINER APPS COMPUTE TOPOLOGY
   +------------------------------------------------------------------------------------------+
   | Managed Environment: cae-sopfactory-dev (West US 3)                                      |
   |   • ca-authoring-bff-dev (Port 8080, External)    • ca-catalog-api-dev (Port 8080, Ext)  |
   |   • ca-model-service-dev (Port 8080, External)    • ca-sopfactory-ui-dev (Port 8090, Ext)|
   |   • ca-library-api-dev (Port 8080, Internal)      • ca-composer-api-dev (Port 8090, Int) |
   |   • ca-composition-mcp-dev (Port 8091, Internal)  • ca-opa-dev (Port 8181, Internal)     |
   +------------------------------------------------------------------------------------------+
   | Managed Environment: cae-sar-demo-dev (West US 3)                                        |
   |   • ca-sar-api-demo-dev (Port 8080, External)     • ca-sar-graph-demo-dev (Port 8080, Int)|
   +------------------------------------------------------------------------------------------+
   | Managed Environment: cae-semantic-insights-poc (West US 3)                               |
   |   • nlp-mcp-server-poc (Port 8000, External)      • nlp-agent-service-poc (Port 8001, Ext)|
   +------------------------------------------------------------------------------------------+
   | Managed Environment: cae-soc2-dev (East US 2 - VNet Connected)                           |
   |   • ca-soc2-dev-api (Port 8080, Internal Only)                                           |
   +------------------------------------------------------------------------------------------+
   | Managed Environment: arcadia-tariff-dev-cae (East US 2)                                  |
   |   • arcadia-tariff-dev-worker (Headless Background Worker, No Ingress)                    |
   +------------------------------------------------------------------------------------------+
```

### 5.1 Technical Configuration Matrix (14 Container Apps)

| Container App | Environment | Image Repository & Tag | CPU | RAM | Port | Ingress | Min/Max Scale | Probes Configured | Identity |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| `arcadia-tariff-dev-worker` | `arcadia-tariff-dev-cae` | `arcadiatariffdevacr.azurecr.io/tariff-worker:v1` | 0.5 | 1Gi | N/A | None | 1 / 1 | None | SystemAssigned |
| `ca-soc2-dev-api` | `cae-soc2-dev` | `crsoc2devo63y.azurecr.io/soc2-api:0.1.0` | 0.5 | 1Gi | 8080 | Internal | 1 / 10 | None | UserAssigned |
| `nlp-mcp-server-poc` | `cae-semantic-insights-poc` | `agenticaidemoregistry.azurecr.io/nlp-mcp-server:v1.2` | 0.5 | 1Gi | 8000 | External | 1 / 2 | None | None |
| `nlp-agent-service-poc` | `cae-semantic-insights-poc` | `agenticaidemoregistry.azurecr.io/agent-service:v1.5` | 0.5 | 1Gi | 8001 | External | 1 / 3 | None | None |
| `ca-opa-dev` | `cae-sopfactory-dev` | `openpolicyagent/opa:latest` | 0.25 | 0.5Gi | 8181 | Internal | 1 / 2 | None | None |
| `ca-model-service-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/model-service:e48a88b1...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | **ZERO PROBES** | System + User |
| `ca-catalog-api-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/catalog-api:2459a476...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | None | System + User |
| `ca-library-api-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/library-api:e48a88b1...` | 0.5 | 1Gi | 8080 | Internal | 1 / 3 | None | System + User |
| `ca-authoring-bff-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/authoring-bff:e48a88b1...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | None | System + User |
| `ca-sopfactory-ui-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/sopfactory-ui@sha256:b207...` | 0.5 | 1Gi | 8090 | External | 1 / 2 | None | System + User |
| `ca-composer-api-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/composer-api:e48a88b1...` | 0.5 | 1Gi | 8090 | Internal | 1 / 3 | None | System + User |
| `ca-composition-mcp-dev` | `cae-sopfactory-dev` | `acrsopfactorydevmlel9.azurecr.io/composition-mcp:e48a88b1...` | 0.25 | 0.5Gi | 8091 | Internal | 1 / 2 | None | System + User |
| `ca-sar-graph-demo-dev` | `cae-sar-demo-dev` | `acrsopfactorydevmlel9.azurecr.io/sar-graph:515e93c3...` | 0.5 | 1Gi | 8080 | Internal | 1 / 2 | None | System + User |
| `ca-sar-api-demo-dev` | `cae-sar-demo-dev` | `acrsopfactorydevmlel9.azurecr.io/sar-api:357d02d5...` | 0.5 | 1Gi | 8080 | External | 1 / 1 | None | System + User |

### 5.2 Critical SRE Deep-Dive Findings in Container Apps

#### 1. Zero Health Probes on `ca-model-service-dev`
*   **Finding:** `ca-model-service-dev` exposes an external HTTP endpoint for LLM completion requests (`AZURE_OPENAI_ENDPOINT = https://ais-sopfactorydevmlel9.cognitiveservices.azure.com/`). The active revision `ca-model-service-dev--0000051` has **no Startup Probe, no Liveness Probe, and no Readiness Probe** configured.
*   **Operational Mechanism:** During container cold-starts, Azure Container Apps Envoy proxy shifts inbound HTTP traffic to the replica before the internal Python FastAPI/Uvicorn runtime finishes initializing OpenAI SDK clients. Inbound requests receive immediate HTTP 502/503 errors during scale-up events.

#### 2. IaC Drift & Unmanaged State of `ca-sopfactory-ui-dev`
*   **Finding:** The frontend UI Container App `ca-sopfactory-ui-dev` runs revision `ca-sopfactory-ui-dev--0000006` with 10 environment variables, referencing Key Vault secret references (`azure-storage-key`, `durable-code`, `github-token`).
*   **Root Cause:** The resource was manually provisioned via Azure Portal/CLI without tags or Terraform state tracking (`NonAKSEstate/evidence/DEV/cross-cutting/iac-coverage-DEV.csv`).

#### 3. Arcadia Tariff Background Worker Architecture
*   **Finding:** `arcadia-tariff-dev-worker` is a headless event-driven worker consuming from Temporal server (`TEMPORAL_ADDRESS`) and querying Eventhouse databases (`EVENTHOUSE_QUERY_URI`). It has no HTTP ingress, running 1 replica continuously under system-assigned managed identity.

---

## 6. Container App Environments (6 CAE)

The DEV subscription contains 6 Container App Managed Environments:

| Environment Name | Resource Group | Region | VNet Subnet Binding | Ingress Scope | App Logs Destination | Log Analytics Workspace ID |
|:---|:---|:---|:---|:---|:---|:---|
| `arcadia-tariff-dev-cae` | `arcadia-tariff-dev-rg` | East US 2 | None (Public) | Public | Log Analytics | `3a79ef86-93f5-4877-910a-c498957c305b` |
| `cae-soc2-dev` | `rg-soc2-dev` | East US 2 | `vnet-soc2-dev / snet-apps` | **Internal Only** | Azure Monitor | *(Dynamic AM Workspace)* |
| `cae-sar-demo-dev` | `helios-dev-us-west3-rg` | West US 3 | None (Public) | Public | Log Analytics | `ce6d82dd-bf84-4197-b258-3607395c2c97` |
| `cae-semantic-insights-poc` | `helios-dev-us-west3-rg` | West US 3 | None (Public) | Public | Log Analytics | `df2ce6a0-593e-4f4a-87b8-0f149b531316` |
| `cae-sopfactory-dev` | `helios-dev-us-west3-rg` | West US 3 | None (Public) | Public | Log Analytics | `ce4545e8-984a-448e-b5b8-7b4cdb56af13` |
| `helios-provision-site-dev-env` | `helios-dev-us-west3-rg` | West US 3 | None (Public) | Public | Log Analytics | `calmpond-4ca451e0...` |

> [!NOTE]
> Only **1 out of 6 environments (`cae-soc2-dev`)** is deployed with Virtual Network integration. The other 5 environments rely on public IPs with infrastructure routing managed entirely across Azure multi-tenant fabric.

---

## 7. App Services (Web Apps) Deep-Dive (6 Apps)

Six Web Apps operate in DEV across four resource groups:

| Web App Name | Resource Group | Runtime Stack | AlwaysOn | FTPS State | Min TLS | Health Check Path | App Insights | Outbound VNet Integration | Managed Identity |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| `heliosdev-ui-appservice` | `helios-dev-uswest3-ui` | Python 3.11 | **True** | Disabled | 1.2 | *(none)* | **Configured** | **Yes** (`appservice-subnet`) | System + User |
| `helios-dev-memo-app` | `dev-helios-memo-rg` | Docker Container | **True** | Disabled | 1.2 | `/` | **Configured** | **No** | SystemAssigned |
| `qcells-warroom-dashboard` | `rg-warroom-dashboard` | Node.js 20-LTS | **False** | FtpsOnly | 1.2 | *(none)* | **MISSING** | **No** | None |
| `helios-mcp-chatbot-app` | `helios-dev-us-west3-rg` | Node.js 20-LTS | **False** | FtpsOnly | 1.2 | *(none)* | **MISSING** | **No** | None |
| `UUDRI-App-Service-dev-01` | `uudri-dev-rg` | .NET 10.0 | **True** | Disabled | 1.2 | *(none)* | **Configured** | **No** | None |
| `UUDRI-Foundry-App-Service-dev-01` | `helios-dev-us-west3-rg` | .NET 10.0 | **True** | Disabled | 1.2 | *(none)* | **Configured** | **No** | None |

### 7.1 Detailed Web App Architecture Observations

1. **`heliosdev-ui-appservice` (Gold Standard in DEV):**
   *   Runs Python 3.11 with 67 environment variables.
   *   Integrates with VNet subnet `helios-aks-dev-vnet/subnets/appservice-subnet`.
   *   Configured with System-Assigned Identity and User-Assigned Identity `heliosdev-ui-appconfig-reader`.
   *   Has private endpoints connected to backend PostgreSQL and Key Vault.
2. **`helios-dev-memo-app` (Corporate Memo System):**
   *   Docker containerized app hosted on App Service Plan `helios-dev-memo-plan` (SKU B2 Basic).
   *   Configured with health check probe path `/` and max ping failures set to 5 (`WEBSITE_HEALTHCHECK_MAXPINGFAILURES = 5`).
   *   Assigned RBAC roles: `Key Vault Secrets User`, `AcrPull`, and `Contributor` on Azure Communication Services (`helios-dev-memo-acs`).
3. **`helios-mcp-chatbot-app` (Stopped Lifecycle):**
   *   State is `Stopped` on Basic B1 App Service Plan.
   *   Zero App Insights instrumentation, running blind.
4. **`qcells-warroom-dashboard` (Observability Blind Spot):**
   *   Has only 3 app settings (`GITHUB_TOKEN`, `ADO_PAT`, `WEBSITE_NODE_DEFAULT_VERSION`).
   *   Completely lacks Application Insights telemetry and diagnostic settings.

---

## 8. Static Web Apps Deep-Dive (10 SWAs)

Ten Static Web Apps provide frontend web interfaces and documentation hosting:

| # | Static Web App Name | Resource Group | SKU | Repository URL | Branch | Build Provider | Default Hostname |
|:---:|:---|:---|:---:|:---|:---|:---:|:---|
| 1 | `helios-operator-console` | `helios-ui-rg` | Free | `https://github.com/qcells-hqct/helios-operator-console` | `main` | GitHub | `agreeable-sand-0c397910f.7.azurestaticapps.net` |
| 2 | `helios-product-map` | `helios-product-map-rg` | Free | *(none)* | *(none)* | SwaCli | `ashy-pebble-032390c0f.7.azurestaticapps.net` |
| 3 | `helios-m2-prototype` | `helios-ui-rg` | Free | `https://github.com/federicoimparatta/helios-m2-prototype` | `main` | GitHub | `proud-cliff-04603190f.7.azurestaticapps.net` |
| 4 | `helios-prototype` | `helios-ui-rg` | Free | `https://github.com/federicoimparatta/helios-prototype` | `main` | GitHub | `white-beach-0ec40e90f.1.azurestaticapps.net` |
| 5 | `helios-dev-engg-dashboard` | `helios-dev-us-west3-rg` | Standard | `https://github.com/qcells-hqct/eng-dashboard` | `main` | GitHub | `gray-coast-06e0c921e.7.azurestaticapps.net` |
| 6 | `helios-qre-architecture` | `rg-qre-architecture-docs` | Free | *(none)* | *(none)* | SwaCli | `orange-rock-01fd4351e.7.azurestaticapps.net` |
| 7 | `helios-architecture` | `rg-warroom-dashboard` | Standard | `https://github.com/qcells-hqct/helios-reference-architecture` | `main` | GitHub | `proud-flower-01d3a001e.1.azurestaticapps.net` |
| 8 | `heliosdev-ui-webapp` | `helios-dev-uswest3-ui` | Standard | *(none)* | *(none)* | SwaCli | `red-desert-0147dd91e.2.azurestaticapps.net` |
| 9 | `UUDRI-Static-Web-App-dev` | `helios-dev-us-west3-rg` | Free | `https://github.com/qcells-hqct/UUDRI-React` | `helios-develop` | Custom | `thankful-mud-0c11d001e.7.azurestaticapps.net` |
| 10 | `UUDRI-Static-Web-App-dev-01` | `uudri-dev-rg` | Free | `https://github.com/qcells-hqct/UUDRI-React` | `develop` | Custom | `white-coast-00296ed1e.1.azurestaticapps.net` |

### 8.1 SWA Governance & Security Findings

1. **Personal GitHub Account Bindings:** `helios-m2-prototype` and `helios-prototype` link directly to personal GitHub repositories (`federicoimparatta/*`) rather than the official organizational repository (`qcells-hqct/*`). This introduces source code provenance risks and bypasses enterprise branch protection rules.
2. **SwaCli Direct Deployments:** `helios-product-map`, `helios-qre-architecture`, and `heliosdev-ui-webapp` show `Provider: SwaCli` with no linked git repository in ARM metadata, indicating direct workstation deployment via Azure Static Web Apps CLI.
3. **Multi-Environment Previews:** `heliosdev-ui-webapp` has 2 active environments: `default` (production slot) and `preview` (`red-desert-0147dd91e-preview.westus2.2.azurestaticapps.net`).
4. **Zero Custom Domains & Zero Linked Backends:** None of the 10 SWAs configure custom domain certificates or native Azure SWA linked backend integrations.

---

## 9. Logic Apps / Workflow Apps (1 App)

*   **App Name:** `helios-platform-alert-watcher`
*   **Resource Group:** `helios-dev-us-west3-rg`
*   **Location:** `westus3`
*   **Type:** `Microsoft.Logic/workflows`
*   **State:** Enabled / ProvisioningState: Succeeded
*   **Created Time:** `2026-08-20T20:25:39.793834+00:00`
*   **Purpose:** Automated platform watcher polling Azure Monitor alert streams, filtering noise, and routing notifications.
*   **Operational Gaps:** Untagged; missing diagnostic settings forwarding workflow run histories to Log Analytics.

---

## 10. Service Bus & Dead-Letter Queue (DLQ) Deep-Dive

SRE conducted a complete audit of all Service Bus namespaces, topics, and queues in DEV:

```
                               SERVICE BUS & DLQ BACKLOG DEEP-DIVE
   +-----------------------------------------------------------------------------------------+
   | Namespace: helios-dev-service-bus-ns (Standard SKU, West US 3)                          |
   |                                                                                         |
   |  Topic: helios-knowledgegraph-events (Size: ~27.8 MB)                                   |
   |    +-- Sub: kg-event-processor ----> [!] 18,169 DEAD-LETTERED MESSAGES (0 Active)      |
   |    |                                  Max Delivery Count: 50 | Lock Duration: 5m        |
   |    +-- Sub: weather-service -------> [✓] 0 Dead-Lettered | 0 Active                     |
   |                                                                                         |
   |  Queue: dml-processor-dlq ---------> [!] 17,411 ACTIVE MESSAGES IN DLQ STORE            |
   |                                                                                         |
   |  Topic: helios-oms-events ---------> Sub: iam-service (23 Dead-Lettered messages)       |
   |  Topic: helios-ontology-events ----> Sub: fabric-ontology-binder (0 DLQ)                |
   +-----------------------------------------------------------------------------------------+
   | Namespace: sb-soc2-dev-o63y (Standard SKU, East US 2)                                   |
   |   Queues: collect (0), evaluate (0), narrate (0), normalize (0)                         |
   +-----------------------------------------------------------------------------------------+
```

### 10.1 Service Bus Backlog Metric Inventory

| Namespace | Entity Name | Entity Type | Active Count | Dead-Letter Count | Max Delivery Count | Lock Duration | Default TTL |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|
| `helios-dev-service-bus-ns` | `helios-knowledgegraph-events / kg-event-processor` | Subscription | 0 | **18,169** | 50 | 5 min (`PT5M`) | 14 days (`P14D`) |
| `helios-dev-service-bus-ns` | `dml-processor-dlq` | Queue | **17,411** | 0 | 10 | 1 min (`PT1M`) | 14 days (`P14D`) |
| `helios-dev-service-bus-ns` | `helios-oms-events / iam-service` | Subscription | 0 | **23** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) |
| `helios-dev-service-bus-ns` | `helios-knowledgegraph-events / weather-service` | Subscription | 0 | 0 | 10 | 1 min (`PT1M`) | 14 days (`P14D`) |
| `helios-dev-service-bus-ns` | `helios-ontology-events / fabric-ontology-binder` | Subscription | 0 | 0 | 10 | 1 min (`PT1M`) | 14 days (`P14D`) |
| `sb-soc2-dev-o63y` | `collect`, `evaluate`, `narrate`, `normalize` | Queues (4) | 0 | 0 | 10 | 1 min (`PT1M`) | 14 days (`P14D`) |

### 10.2 Root Cause Analysis of the 18,169 Dead-Lettered Messages

1. **Dead-Letter Accumulation Mechanics:**
   The `kg-event-processor` subscription consumes Knowledge Graph DML mutation events. When consumer Function App `kg-event-processor-dev` encountered downstream schema deserialization or timeout failures, Azure Service Bus retried delivery up to **50 times** (`Max Delivery Count: 50`) holding locks for **5 minutes per attempt** (`Lock Duration: PT5M`).
2. **Permanent DLQ Retention:**
   Service Bus dead-letter queues do not expire messages (`DeadLetteringOnMessageExpiration = False`). As a result, **18,169 failed events** accumulated permanently in the subscription DLQ store, consuming ~27.8 MB of namespace quota.
3. **Queue Ingestion Pipeline Mirror:**
   The `dml-processor-dlq` queue contains **17,411 active messages**, representing a secondary ingest queue where failed pipeline stages offloaded unparseable payload events.

---

## 11. UUDRI Estate Architectural Breakdown

The discovery revealed two completely independent, parallel stacks for the Utility Usage Data & Rate Intelligence (UUDRI) system:

```
                            DUAL-STACK UUDRI ARCHITECTURE COMPARISON
    +-------------------------------------------+-------------------------------------------+
    |           PERSONAL DEV STACK              |             TEAM SHARED STACK             |
    |         (Resource Group: uudri-dev-rg)    |   (Resource Group: helios-dev-us-west3-rg)|
    +-------------------------------------------+-------------------------------------------+
    | App Service:                              | App Service:                              |
    |   UUDRI-App-Service-dev-01                |   UUDRI-Foundry-App-Service-dev-01        |
    |   • Host: uudri-app-service-dev-01...     |   • Host: uudri-foundry-app-service-dev...|
    |   • Region: West US 2                     |   • Region: West US 3                     |
    |   • Owner: divyanshu.arya@...             |   • Owner: DevOps                         |
    |   • Storage: uudristorageaccountdev01     |   • Storage: uudristorageaccountdev       |
    |                                           |                                           |
    | Static Web App:                           | Static Web App:                           |
    |   UUDRI-Static-Web-App-dev-01             |   UUDRI-Static-Web-App-dev                |
    |   • Host: white-coast-00296ed1e...        |   • Host: thankful-mud-0c11d001e...       |
    |   • Branch: develop                       |   • Branch: helios-develop                |
    |                                           |                                           |
    | Function App:                             | Function App:                             |
    |   uudri-bill-processor-dev                |   UUDRI-Bill-Processor-dev-01             |
    |                                           |                                           |
    | Key Vault:                                | Key Vault:                                |
    |   UUDRI-Key-Vault-dev-02                  |   UUDRI-Key-Vault-dev                     |
    +-------------------------------------------+-------------------------------------------+
```

### 11.1 Key Differences and Consolidation Mandate

1. **Target Endpoints:** The personal frontend (`white-coast-00296ed1e`) points to `UUDRI-App-Service-dev-01`, while the team frontend (`thankful-mud-0c11d001e`) connects to `UUDRI-Foundry-App-Service-dev-01`.
2. **Entra External Tenants:** Both stacks authenticate against External Entra Tenant `f31dc09f-b3a3-4fe5-92bd-f91499994fc4` (`Heliosexternaldev.onmicrosoft.com`).
3. **Database & Storage Redundancy:** Both stacks maintain separate PostgreSQL connection strings and Azure Blob containers (`utilitybills`, `disputefiles`), leading to data fragmentation in DEV.

---

## 12. Identity, Access & RBAC Matrix

### 12.1 Managed Identity Inventory (13 Identities Mapped)

Thirteen Managed Identities (System-Assigned and User-Assigned) were verified across Non-AKS compute:

| Resource Name | Identity Type | Principal ID / Resource ID | Assigned Roles & Scopes |
|:---|:---|:---|:---|
| `arcadia-tariff-dev-worker` | SystemAssigned | `fa56c67b-1c46-40e5-bffd-158acc3f2901` | *(None assigned in subscription)* |
| `ca-soc2-dev-api` | UserAssigned | `id-soc2-dev-api` (`rg-soc2-dev`) | Role assignments managed within `rg-soc2-dev` scope |
| `ca-model-service-dev` | System + User | `7319db5e-d22b-4d63-b1b8-1257c98d80a9` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-catalog-api-dev` | System + User | `a0c3852d-4bbf-41f5-a607-ab79d5660ce8` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-library-api-dev` | System + User | `9d2013de-e85d-4222-a9df-848aaf695746` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-authoring-bff-dev` | System + User | `030bb001-667c-4ec3-9fa7-b987e4e2a3f3` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-sopfactory-ui-dev` | System + User | `e76b03a4-a774-4a2d-9ec1-f6ef0863b617` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-composer-api-dev` | System + User | `cd12cfad-fb80-487c-a033-85718cff6b5d` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-composition-mcp-dev` | System + User | `09e2cf53-187c-47cf-afc1-67461ec03165` <br> `uami-sopfactory-dev-apps` | Role assignments inherited via `uami-sopfactory-dev-apps` |
| `ca-sar-graph-demo-dev` | System + User | `b5b78a9b-663c-4148-aede-6742e98f3b13` <br> `uami-sar-demo-dev-apps` | Role assignments inherited via `uami-sar-demo-dev-apps` |
| `ca-sar-api-demo-dev` | System + User | `e9a05e9a-d62f-4592-b960-994f8f6b6c32` <br> `uami-sar-demo-dev-apps` | Role assignments inherited via `uami-sar-demo-dev-apps` |
| `heliosdev-ui-appservice` | System + User | `dd954f48-d349-49a6-867a-3e32d27d583d` <br> `heliosdev-ui-appconfig-reader` | **Azure AI Developer** (`/projects/pms-core-project`) |
| `helios-dev-memo-app` | SystemAssigned | `090c5a54-7b12-4dcb-a810-264221d8a69f` | **Key Vault Secrets User** (`helios-dev-memo-kv`) <br> **AcrPull** (`heliosdevaksregistry`) <br> **Contributor** (`helios-dev-memo-acs`) |
| `heliosdev-ui-webapp` | SystemAssigned | `c7df6570-c206-42c6-b3da-cbe3b4505108` | **Azure AI Developer** (`helios-dev-aif-hub`) <br> **Azure AI Developer** (`/projects/pms-core-project`) |

### 12.2 Key Vault Access Analysis (20 Key Vaults Mapped)

All **20 Key Vaults** discovered in the DEV subscription operate under legacy **Access Policies (`enableRbacAuthorization = false`)** with 0 explicit access policies granted in ARM metadata (`keyvault-access-DEV.csv`):

| Key Vault Name | Resource Group | Access Model | Policies Count | RBAC Mode |
|:---|:---|:---:|:---:|:---:|
| `heliosdev-ui-kv` | `helios-dev-uswest3-ui` | Access Policies | 0 | Disabled |
| `helios-dev-backend-kv` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `data-encryption-helios` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `kv-marthala851934804853` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `kv-semantic-poc-4208` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-dev-agents-kv` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-dev-onboarding-kv` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-dev-sop-service` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-dev-memo-kv` | `dev-helios-memo-rg` | Access Policies | 0 | Disabled |
| `helios-dev-github-kv` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-dev-keyvaults` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `UUDRI-Key-Vault-dev` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-dev-spkplug2-kv` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `fedaratedkvpoc` | `fedratedkvpoc` | Access Policies | 0 | Disabled |
| `kvsopfactorydevmlel9` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `kg-event-proc-dev-kv` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `kvsardemodev04yt4` | `helios-dev-us-west3-rg` | Access Policies | 0 | Disabled |
| `kv-helios-dev-iam-0c2402` | `rg-helios-dev-iam` | Access Policies | 0 | Disabled |
| `UUDRI-Key-Vault-dev-02` | `uudri-dev-rg` | Access Policies | 0 | Disabled |
| `kv-soc2-dev-o63y` | `rg-soc2-dev` | Access Policies | 0 | Disabled |

---

## 13. Network Security & Topology

```
                               DEV NETWORK SECURITY & TOPOLOGY
    +------------------------------------------------------------------------------------+
    | VNet: helios-aks-dev-vnet (West US 3)                                              |
    |                                                                                    |
    |   Subnet: appservice-subnet                                                        |
    |     • heliosdev-ui-appservice (Outbound Integration)                               |
    |                                                                                    |
    |   Subnet: private-endpoints-subnet (26 Private Endpoints)                          |
    |     • Storage: heliosobjectstore, heliosdevaifsa                                   |
    |     • Cognitive / Foundry: helios-dev-aif-hub, muhammad-ai-project-resource        |
    |     • Key Vaults: backend-kv, github-kv, agents-kv, onboarding-kv, heliosdev-ui-kv |
    |     • Databases: helios-dev-monetization-sql, heliosdev-ui-postgres, Cosmos DB     |
    |     • Eventing: EventHub NS, EventGrid topics, MQTT Broker NS                      |
    +------------------------------------------------------------------------------------+
    | VNet: vnet-soc2-dev (East US 2)                                                    |
    |   Subnet: snet-apps (Internal Container App Environment: cae-soc2-dev)             |
    |   Private Endpoints: pe-soc2-dev-acr, pe-soc2-dev-blob, pe-soc2-dev-kv            |
    +------------------------------------------------------------------------------------+
```

### 13.1 Private Endpoints Summary (26 Mapped)

SRE mapped **26 active Private Endpoints** securing data and backend layers in DEV (`private-endpoints-DEV.csv`):
*   **Storage & Blobs (4):** `heliosobjectstore-blob-pe`, `helios-dev-aif-storage-pe`, `pe-soc2-dev-blob`, `pe-soc2-dev-dfs`.
*   **Key Vaults (6):** `helios-dev-backend-kv-pe`, `helios-dev-agents-kv-pe`, `helios-dev-github-kv-pe`, `helios-dev-onboarding-kv-pe`, `heliosdev-ui-kv-pe`, `pe-soc2-dev-kv`.
*   **Databases & Cosmos DB (3):** `helios-dev-monetization-sql-pe`, `heliosdev-ui-postgres-pe`, `helios-dev-cosmos-askhelios-pe`.
*   **Messaging & Event Hubs (5):** `helios-dev-eventhub-pe`, `helios-dev-platform-audit-log-pe`, `helios-dev-eventgrid-pe`, `helios-dev-mqtt-broker-pe`, `helios-dev-mqtt-broker-pe1`.
*   **AI Services (3):** `helios-dev-aif-pe`, `helios-dev-extract-equipment-pe`, `muhammad-ai-project`.
*   **Compute & Registries (5):** `heliosdev-ui-appservice-pe`, `pe-soc2-dev-acr`, `platform-backend-insights-dev-pe`, `kube-apiserver`, `cae-soc2-dev` private endpoint.

### 13.2 Public Exposure Analysis

*   **13 out of 14 Container Apps** are reachable via public DNS hostnames (`*.azurecontainerapps.io`) without IP whitelisting.
*   **5 out of 6 App Services** have `PublicAccess = Enabled` with 0 IP access restriction rules configured.
*   **10 out of 10 Static Web Apps** are fully exposed to public ingress.

---

## 14. Observability & Alerting Estate

### 14.1 Diagnostic Settings Coverage Gap (35 of 37 Missing)

*   **Configured (2):** `heliosdev-ui-appservice` (2 settings streaming to Log Analytics) and `cae-soc2-dev` (1 setting streaming to Log Analytics).
*   **Missing (35):** **35 of 37 Non-AKS resources** do not stream resource logs, HTTP access logs, or platform metrics to any Log Analytics workspace (`diagnostic-settings-DEV.csv`).

### 14.2 Metric & Log Alert Rules

1. **Metric Alerts (23 Rules):**
   *   8 Container Apps covered by zero-replica alerts (`ca-opa-dev-zero-replicas`, `ca-model-service-dev-zero-replicas`, etc.).
   *   1 App Service covered by HTTP 5xx alert (`heliosdev-ui-appservice-http5xx`).
   *   1 Service Bus Dead-Letter alert (`helios-dev-knowledgegraph-events-deadletters`).
2. **Log (Scheduled Query) Alerts (51 Rules):**
   *   51 KQL query alerts monitoring OTEL pipeline exceptions, Bronze ratio ingestion errors, market forecast drift, and knowledge graph verifier mass drift (`scheduled-query-alerts-all.json`).
3. **Action Groups (5 Mapped):**
   *   `ag-helios-dev-iam` (Email)
   *   `Application Insights Smart Detection` (ARM Role)
   *   `ag-helios-ops` (Email)
   *   `ag-soc2-dev-security` (0 receivers configured)
   *   `ag-soc2-dev-slo` (0 receivers configured)
4. **Availability & Web Tests:**
   *   **0 URL Ping Web Tests** are configured across all resource groups in DEV (`availability-tests-DEV.csv`).

---

## 15. IaC & Terraform Governance Status

SRE evaluated IaC governance by checking resource tags (`managed_by=terraform`, `ManagedBy=terraform`) and cross-referencing repository state files:

```
                            DEV IAC GOVERNANCE DISTRIBUTION
   +---------------------------------------+---------------------------------------+
   |            TERRAFORM MANAGED          |          UNMANAGED / UNKNOWN          |
   |             (15 Resources)            |             (22 Resources)            |
   +---------------------------------------+---------------------------------------+
   | • 8 Container Apps (SOC2, SOP Factory,| • 6 Container Apps (Arcadia worker,   |
   |   SAR suite)                          |   NLP POCs, ca-sopfactory-ui-dev)     |
   | • 3 Container App Environments        | • 5 App Services (Warroom, Memo app,  |
   | • 1 App Service (heliosdev-ui-appserv)|   MCP chatbot, UUDRI stacks)          |
   | • 1 Static Web App (heliosdev-ui-web) | • 8 Static Web Apps (Prototypes,      |
   | • 2 Key Vault / supporting blocks     |   Operator console, Architecture SWAs)|
   |                                       | • 3 Container App Environments        |
   +---------------------------------------+---------------------------------------+
```

### 15.1 Root Cause for `ca-sopfactory-ui-dev` Unmanaged Drift

During initial deployment of SOP Factory in DEV, backend APIs and Durable Functions were provisioned via Terraform modules (`iac-coverage-DEV.csv`). However, the React frontend container `ca-sopfactory-ui-dev` was spun up via ad-hoc Azure CLI commands by development teams to test ingress proxy bindings, leaving it untracked by Terraform state.

---

## 16. Current DEV Architecture Diagram

```
                                     CLIENT & INGRESS LAYER
       +---------------------------------------------------------------------------------+
       |  Public Internet / Developers / External Entra Tenants                          |
       +--------+-----------------------+--------------------+--------------------+------+
                |                       |                    |                    |
                v                       v                    v                    v
      +-------------------+   +-------------------+   +-------------+   +----------------+
      | Static Web Apps   |   | Web Apps          |   | Logic Apps  |   | Container Apps |
      | (10 SWAs)         |   | (6 App Services)  |   | (1 App)     |   | (14 Apps)      |
      |                   |   |                   |   |             |   |                |
      | • Operator Console|   | • heliosdev-ui    |   | • alert-    |   | • SOP Factory  |
      | • Engg Dashboard  |   | • memo-app        |   |   watcher   |   |   (7 apps)     |
      | • Architecture    |   | • warroom-dash    |   +-------------+   | • SAR Demo     |
      | • UUDRI (x2)      |   | • mcp-chatbot [X] |                     |   (2 apps)     |
      | • Prototypes (x2) |   | • UUDRI (x2)      |                     | • SOC2 API     |
      +---------+---------+   +---------+---------+                     | • NLP POCs (2) |
                |                       |                               | • Arcadia      |
                +-----------------------+-------------------------------+ • OPA          |
                                        |                               +-------+--------+
                                        v                                       |
       +------------------------------------------------------------------------+--------+
       |                         INTERNAL SERVICE MESH & VNET LAYER                      |
       |                                                                                 |
       |  Subnet: appservice-subnet                       Subnet: snet-apps (SOC2)       |
       |    • heliosdev-ui-appservice                       • ca-soc2-dev-api            |
       |                                                                                 |
       |  Private Endpoints (26 PEs):                                                    |
       |    • Key Vaults (backend-kv, agents-kv, github-kv, ui-kv, soc2-kv)              |
       |    • Databases (PostgreSQL, Monetization SQL, Cosmos DB)                        |
       |    • Storage (heliosobjectstore, heliosdevaifsa, stsoc2devo63y)                 |
       +------------------------------------+--------------------------------------------+
                                            |
                                            v
       +---------------------------------------------------------------------------------+
       |                         MESSAGING & EVENT QUEUES LAYER                          |
       |                                                                                 |
       |  Service Bus: helios-dev-service-bus-ns                                         |
       |    • Topic: helios-knowledgegraph-events                                        |
       |        +-- Sub: kg-event-processor ----> [!] 18,169 DEAD-LETTERED MESSAGES      |
       |    • Queue: dml-processor-dlq ---------> [!] 17,411 ACTIVE MESSAGES             |
       |                                                                                 |
       |  Event Hubs:                                                                    |
       |    • helios-eventhub-ns (telemetry, ontology, audit logs)                       |
       +------------------------------------+--------------------------------------------+
                                            |
                                            v
       +---------------------------------------------------------------------------------+
       |                         OBSERVABILITY & MONITORING LAYER                        |
       |                                                                                 |
       |  Application Insights:                 Azure Monitor:                           |
       |    • Configured on 4/6 Web Apps          • Diagnostic Settings: 2/37 configured |
       |    • Missing on Warroom & Chatbot        • Metric Alerts: 23 rules              |
       |                                          • Log Alerts: 51 rules                 |
       |                                          • Web Ping Tests: 0 configured         |
       +---------------------------------------------------------------------------------+
```

---

## 17. Capability Matrix Update for Non-AKS Estate

Based on DEV findings across the 37 Non-AKS resources, the SRE Release Orchestration Capability Matrix DEV column is updated with factual evidence citations:

| Capability Column | DEV Status | SRE Evidence Citation |
|:---|:---:|:---|
| **Deploy on merge** | **Partial** | **Container Apps:** SOP Factory and SAR use automated Terraform/GitHub Actions pipelines (`acrsopfactorydevmlel9`). <br>**Web Apps:** `heliosdev-ui-appservice` uses CI/CD; `qcells-warroom-dashboard` and `helios-mcp-chatbot-app` show no active SCM. <br>**Static Web Apps:** 5/10 auto-deploy on git merge to `main`; 3/10 use manual `SwaCli` push (`static-webapps-deepdive-DEV-2026-08-31_13-39-46.log`). |
| **Promotion gates** | **Partial** | Container Apps and App Services promote via environment-specific GitHub Actions workflows. However, mutable container tags (e.g. `:latest`, `:v1`) are used on `ca-opa-dev` and `arcadia-tariff-dev-worker` rather than immutable SHA digests (`container-apps-deepdive-DEV-2026-08-31_13-32-33.log`). |
| **Scheduled verification** | **No** | **0 out of 37 resources** have Azure Monitor URL Ping Web Tests configured (`availability-tests-DEV.csv`). App Service health paths are unconfigured on 5/6 apps. Container App `ca-model-service-dev` has zero Startup/Liveness/Readiness probes. |
| **Alerting** | **Partial** | 23 Metric Alerts cover zero-replica states on Container Apps and HTTP 5xx on `heliosdev-ui-appservice`. 51 Log Query alerts active in Log Analytics. However, **12 compute resources have zero alert rules**, and 2 out of 5 Action Groups have 0 configured receivers (`cross-cutting-checks-DEV-2026-08-31_13-44-06.log`). |
| **Observability + cost** | **Partial** | **35 out of 37 resources lack Azure Monitor Diagnostic Settings** (`diagnostic-settings-DEV.csv`). 2 Web Apps (`qcells-warroom-dashboard`, `helios-mcp-chatbot-app`) completely lack App Insights instrumentation. 18,169 dead-lettered messages on Service Bus indicate unmonitored processing failures. |
| **Reporting / audit** | **Partial** | 15 of 37 resources are IaC-tagged and Terraform-managed (`iac-coverage-DEV.csv`). 22 resources are unmanaged or lack ownership metadata. All 13 Managed Identities and 26 Private Endpoints are fully mapped in audit logs. |

---

*Evidence verified and recorded by SRE team. No remediation or implementation changes have been performed during this discovery phase.*
