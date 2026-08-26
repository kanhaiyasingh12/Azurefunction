# Azure Functions — SRE Discovery Report (QA Environment)

**Project:** Azure Functions (87 across 3 envs)  
**Environment:** QA  
**Scope:** Azure Functions (10 Function Apps / 28 Functions in QA)  
**Status:** Discovery complete — no remediation or implementation changes have been performed  

---

[[_TOC_]]

---

## 1. Background and Context

### 1.1 Why This Investigation Exists

The SRE team maintains a **Release Orchestration Capability Matrix** that tracks the operational maturity of every platform component across six capability columns:

| Column | Description |
|:---|:---|
| Deploy on merge | Does the component auto-deploy when code is merged? |
| Promotion gates (dev > QA > prod) | Is there a controlled gate between environments? |
| Scheduled verification | Are there recurring tests that validate the running system? |
| Alerting | Are alert rules and action groups configured? |
| Observability + cost | Is telemetry flowing and are costs tracked? |
| Reporting / audit | Are deployments and changes auditable? |

The matrix covers: k8s + ArgoCD apps (incl. Temporal), Frontend (React, IBM components), Foundry models, **Azure Functions (~87 across 3 envs)**, Flyway DB migrations, Non-AKS Azure estate (86 resources), and Fabric / Knowledge Graphs.

When SRE reviewed the matrix, **Azure Functions was marked `unknown` in every column**. Following the DEV environment baseline, this report maps the **QA environment** to update the matrix.

### 1.2 Investigation Principle

```
DISCOVER --> VERIFY --> SAVE EVIDENCE --> MAP ARCHITECTURE --> EXPLAIN CURRENT STATE --> IDENTIFY OPERATIONAL GAPS --> GET ENGINEERING DECISION --> IMPLEMENT LATER
```

**Rule:** Do not conclude from a single API call. Cross-check related configuration and preserve the evidence.  
**Rule:** No remediation is performed during discovery.

---

## 2. QA Environment Scope

| Metric | Value |
|:---|:---|
| Function Apps in QA | **10** |
| Registered Functions in QA | **28** |
| Active Function Apps (with functions) | **8** |
| Inactive Function Apps (0 functions) | **2** |
| Logic Apps / Workflow Apps in QA | **4** |
| Runtimes in use | `Python 3.11`, `Python 3.12`, `Node.js 20`, `.NET isolated` |
| Azure subscription | `663afada-2155-4c4d-b908-ac771ef2d133` (Helios - QA) |
| Primary resource group | `helios-qa-us-west3-rg` |
| Secondary resource group | `uudri-qa-rg` |

This report covers only the **10 QA Function Apps, 28 QA Functions, and 4 Logic Apps** verified through direct Azure API queries.

---


## 3. Function App Inventory

### 3.1 Complete List of QA Function Apps

| # | Function App | Resource Group | Purpose (inferred from discovery) |
|:---|:---|:---|:---|
| 1 | `UUDRI-bill-processor-qa-01` | `uudri-qa-rg` | Processes utility bills from Blob Storage |
| 2 | `UUDRI-Function-App-qa-01` | `uudri-qa-rg` | Running site (currently has 0 functions deployed) |
| 3 | `kg-event-processor-qa` | `helios-qa-us-west3-rg` | Processes Knowledge Graph DML events from Service Bus |
| 4 | `func-projector-sopfactoryqao80ns` | `helios-qa-us-west3-rg` | HTTP-triggered publish projection (SOP Factory) |
| 5 | `helios-qa-cost-ingestion` | `helios-qa-us-west3-rg` | Scheduled daily cost data ingestion |
| 6 | `helios-device-telemetry-qa-func` | `helios-qa-us-west3-rg` | Device telemetry App (currently has 0 functions deployed) |
| 7 | `func-orchestrator-sopfactoryqao80ns` | `helios-qa-us-west3-rg` | SOP Factory Durable Functions orchestration engine (16 functions) |
| 8 | `ems-plan-narration-function-qa` | `helios-qa-us-west3-rg` | Narrates energy plans and listens for realized KPI events from Event Hub |
| 9 | `helios-github-activity-logger-qa-func` | `helios-qa-us-west3-rg` | Receives GitHub webhook payloads |
| 10 | `helios-ontology-event-processor-func-qa` | `helios-qa-us-west3-rg` | Processes ontology events from Event Hub; exposes test/health HTTP endpoints |


### 3.2 Function App to Function Count

| Function App | Function Count |
|:---|:---|
| `UUDRI-bill-processor-qa-01` | 1 |
| `UUDRI-Function-App-qa-01` | **0** |
| `kg-event-processor-qa` | 2 |
| `func-projector-sopfactoryqao80ns` | 1 |
| `helios-qa-cost-ingestion` | 1 |
| `helios-device-telemetry-qa-func` | **0** |
| `func-orchestrator-sopfactoryqao80ns` | 16 |
| `ems-plan-narration-function-qa` | 2 |
| `helios-github-activity-logger-qa-func` | 1 |
| `helios-ontology-event-processor-func-qa` | 4 |
| **TOTAL** | **28** |

---

## 4. Complete 28-Function Inventory

| # | Function App | Resource Group | Function Name | Type | Language | Disabled |
|:---|:---|:---|:---|:---|:---|:---|
| 1 | UUDRI-bill-processor-qa-01 | uudri-qa-rg | BillProcessor | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |
| 2 | kg-event-processor-qa | helios-qa-us-west3-rg | HealthCheck | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |
| 3 | kg-event-processor-qa | helios-qa-us-west3-rg | ProcessDMLEvent | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |
| 4 | func-projector-sopfactoryqao80ns | helios-qa-us-west3-rg | projectOnPublish | Microsoft.Web/sites/functions | node | FALSE |
| 5 | helios-qa-cost-ingestion | helios-qa-us-west3-rg | cost_ingestion | Microsoft.Web/sites/functions | python | FALSE |
| 6 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | buildPullRequest | Microsoft.Web/sites/functions | node | FALSE |
| 7 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | cancelOnboardingSession | Microsoft.Web/sites/functions | node | FALSE |
| 8 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | classifySection | Microsoft.Web/sites/functions | node | FALSE |
| 9 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | generateFddArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 10 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | generateOntologyArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 11 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | generatePolicyArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 12 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | generateSopArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 13 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | ingestDocument | Microsoft.Web/sites/functions | node | FALSE |
| 14 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | intakeDocument | Microsoft.Web/sites/functions | node | FALSE |
| 15 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | onSourceDocument | Microsoft.Web/sites/functions | node | FALSE |
| 16 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | openPullRequest | Microsoft.Web/sites/functions | node | FALSE |
| 17 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | pipelineRunDetail | Microsoft.Web/sites/functions | node | FALSE |
| 18 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | pipelineRuns | Microsoft.Web/sites/functions | node | FALSE |
| 19 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | publishOnboardingSession | Microsoft.Web/sites/functions | node | FALSE |
| 20 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | screenSourceText | Microsoft.Web/sites/functions | node | FALSE |
| 21 | func-orchestrator-sopfactoryqao80ns | helios-qa-us-west3-rg | sopFactoryOrchestrator | Microsoft.Web/sites/functions | node | FALSE |
| 22 | ems-plan-narration-function-qa | helios-qa-us-west3-rg | plan_narration_agent | Microsoft.Web/sites/functions | python | FALSE |
| 23 | ems-plan-narration-function-qa | helios-qa-us-west3-rg | realized_kpi_listener | Microsoft.Web/sites/functions | python | FALSE |
| 24 | helios-github-activity-logger-qa-func | helios-qa-us-west3-rg | github_webhook | Microsoft.Web/sites/functions | python | FALSE |
| 25 | helios-ontology-event-processor-func-qa | helios-qa-us-west3-rg | health_check | Microsoft.Web/sites/functions | python | FALSE |
| 26 | helios-ontology-event-processor-func-qa | helios-qa-us-west3-rg | ontology_event_processor | Microsoft.Web/sites/functions | python | FALSE |
| 27 | helios-ontology-event-processor-func-qa | helios-qa-us-west3-rg | test_relationship_processor | Microsoft.Web/sites/functions | python | FALSE |
| 28 | helios-ontology-event-processor-func-qa | helios-qa-us-west3-rg | test_schema_processor | Microsoft.Web/sites/functions | python | FALSE |

### 4.1 Functions Grouped by Runtime

```
dotnet-isolated (3 functions / 2 apps)
    +-- BillProcessor          (UUDRI-bill-processor-qa-01)
    +-- HealthCheck            (kg-event-processor-qa)
    +-- ProcessDMLEvent        (kg-event-processor-qa)

python (8 functions / 4 apps)
    +-- cost_ingestion                                  (helios-qa-cost-ingestion)
    +-- plan_narration_agent                            (ems-plan-narration-function-qa)
    +-- realized_kpi_listener                           (ems-plan-narration-function-qa)
    +-- github_webhook                                  (helios-github-activity-logger-qa-func)
    +-- health_check                                    (helios-ontology-event-processor-func-qa)
    +-- ontology_event_processor                        (helios-ontology-event-processor-func-qa)
    +-- test_relationship_processor                     (helios-ontology-event-processor-func-qa)
    +-- test_schema_processor                           (helios-ontology-event-processor-func-qa)

node (17 functions / 2 apps)
    +-- projectOnPublish           (func-projector-sopfactoryqao80ns)
    +-- buildPullRequest           (func-orchestrator-sopfactoryqao80ns)
    +-- cancelOnboardingSession    (func-orchestrator-sopfactoryqao80ns)
    +-- classifySection            (func-orchestrator-sopfactoryqao80ns)
    +-- generateFddArtifact        (func-orchestrator-sopfactoryqao80ns)
    +-- generateOntologyArtifact   (func-orchestrator-sopfactoryqao80ns)
    +-- generatePolicyArtifact     (func-orchestrator-sopfactoryqao80ns)
    +-- generateSopArtifact        (func-orchestrator-sopfactoryqao80ns)
    +-- ingestDocument             (func-orchestrator-sopfactoryqao80ns)
    +-- intakeDocument             (func-orchestrator-sopfactoryqao80ns)
    +-- onSourceDocument           (func-orchestrator-sopfactoryqao80ns)
    +-- openPullRequest            (func-orchestrator-sopfactoryqao80ns)
    +-- pipelineRunDetail          (func-orchestrator-sopfactoryqao80ns)
    +-- pipelineRuns               (func-orchestrator-sopfactoryqao80ns)
    +-- publishOnboardingSession   (func-orchestrator-sopfactoryqao80ns)
    +-- screenSourceText           (func-orchestrator-sopfactoryqao80ns)
    +-- sopFactoryOrchestrator     (func-orchestrator-sopfactoryqao80ns)
```

---

## 5. Trigger and Binding Inventory

### 5.1 Trigger Type Summary

| Trigger Type | Count |
|:---|:---|
| httpTrigger | 10 |
| activityTrigger | 9 |
| durableClient | 4 |
| eventHubTrigger | 3 |
| blobTrigger | 2 |
| serviceBusTrigger | 1 |
| orchestrationTrigger | 1 |
| timerTrigger | 1 |

### 5.2 Complete Binding Inventory

Some Functions register multiple inbound bindings (e.g. `intakeDocument` has both `httpTrigger` and `durableClient`).

| Function App | Function Name | Trigger Type | Direction | Auth Level | Methods | Route | Connection | Event Hub Name | Topic Name | Subscription Name | Schedule |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
| UUDRI-bill-processor-qa-01 | BillProcessor | blobTrigger | In | | | | StorageAccount | | | | |
| kg-event-processor-qa | HealthCheck | httpTrigger | In | Anonymous | get | health | | | | | |
| kg-event-processor-qa | ProcessDMLEvent | serviceBusTrigger | In | | | | SERVICEBUS_CONNECTION_STRING | | | helios-knowledgegraph-events | kg-event-processor |
| func-projector-sopfactoryqao80ns | projectOnPublish | httpTrigger | in | function | POST | | | | | | |
| helios-qa-cost-ingestion | cost_ingestion | timerTrigger | in | | | | | | | | 0 0 6 * * * |
| func-orchestrator-sopfactoryqao80ns | buildPullRequest | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | cancelOnboardingSession | httpTrigger | in | function | POST | v1/onboarding/{session_id}/cancel | | | | | |
| func-orchestrator-sopfactoryqao80ns | classifySection | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | generateFddArtifact | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | generateOntologyArtifact | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | generatePolicyArtifact | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | generateSopArtifact | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | ingestDocument | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | intakeDocument | httpTrigger | in | function | POST | v1/documents | | | | | |
| func-orchestrator-sopfactoryqao80ns | intakeDocument | durableClient | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | onSourceDocument | blobTrigger | in | | | | SOURCE_STORAGE | | | | |
| func-orchestrator-sopfactoryqao80ns | onSourceDocument | durableClient | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | openPullRequest | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | pipelineRunDetail | httpTrigger | in | function | GET | v1/pipeline/runs/{instanceId} | | | | | |
| func-orchestrator-sopfactoryqao80ns | pipelineRunDetail | durableClient | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | pipelineRuns | httpTrigger | in | function | GET | v1/pipeline/runs | | | | | |
| func-orchestrator-sopfactoryqao80ns | pipelineRuns | durableClient | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | publishOnboardingSession | httpTrigger | in | function | POST | v1/onboarding/{session_id}/publish | | | | | |
| func-orchestrator-sopfactoryqao80ns | screenSourceText | activityTrigger | in | | | | | | | | |
| func-orchestrator-sopfactoryqao80ns | sopFactoryOrchestrator | orchestrationTrigger | in | | | | | | | | |
| ems-plan-narration-function-qa | plan_narration_agent | eventHubTrigger | IN | | | | EVENT_HUB_CONNECTION_STRING | %EVENT_HUB_NAME% | | | |
| ems-plan-narration-function-qa | realized_kpi_listener | eventHubTrigger | IN | | | | REALIZED_KPI_EVENT_HUB_CONNECTION_STRING | %REALIZED_KPI_EVENT_HUB_NAME% | | | |
| helios-github-activity-logger-qa-func | github_webhook | httpTrigger | IN | FUNCTION | POST | github/webhook | | | | | |
| helios-ontology-event-processor-func-qa | health_check | httpTrigger | IN | ANONYMOUS | GET | health | | | | | |
| helios-ontology-event-processor-func-qa | ontology_event_processor | eventHubTrigger | IN | | | | EventHubConnection | %EVENT_HUB_NAME% | | | |
| helios-ontology-event-processor-func-qa | test_relationship_processor | httpTrigger | IN | FUNCTION | POST | test_relationship | | | | | |
| helios-ontology-event-processor-func-qa | test_schema_processor | httpTrigger | IN | FUNCTION | POST | test_schema | | | | | |

---

## 6. External Dependencies and Data Flows

### 6.1 Event Dependencies Inventory

| Function App | Function Name | Trigger Type | Connection Setting | Event Hub Name | Topic Name | Subscription Name | Schedule |
|:---|:---|:---|:---|:---|:---|:---|:---|
| `UUDRI-bill-processor-qa-01` | BillProcessor | blobTrigger | StorageAccount | | | | |
| `kg-event-processor-qa` | ProcessDMLEvent | serviceBusTrigger | SERVICEBUS_CONNECTION_STRING | | helios-knowledgegraph-events | kg-event-processor | |
| `helios-qa-cost-ingestion` | cost_ingestion | timerTrigger | | | | | 0 0 6 * * * |
| `func-orchestrator-sopfactoryqao80ns` | onSourceDocument | blobTrigger | SOURCE_STORAGE | | | | |
| `ems-plan-narration-function-qa` | plan_narration_agent | eventHubTrigger | EVENT_HUB_CONNECTION_STRING | %EVENT_HUB_NAME% | | | |
| `ems-plan-narration-function-qa` | realized_kpi_listener | eventHubTrigger | REALIZED_KPI_EVENT_HUB_CONNECTION_STRING | %REALIZED_KPI_EVENT_HUB_NAME% | | | |
| `helios-ontology-event-processor-func-qa` | ontology_event_processor | eventHubTrigger | EventHubConnection | %EVENT_HUB_NAME% | | | |

---

## 7. Durable Functions Architecture — SOP Factory (QA vs DEV)

The QA orchestrator `func-orchestrator-sopfactoryqao80ns` has an architectural delta compared to DEV:
*   **New Activity:** `buildPullRequest` is present in QA.
*   **Missing Endpoint:** `pipelineSmeQueue` is missing in QA.

::: mermaid
graph TD
    A["intakeDocument (HTTP POST)"] -->|durableClient| Client[Durable Client Layer]
    B["onSourceDocument (Blob)"] -->|durableClient| Client
    C["pipelineRuns (HTTP GET)"] -->|durableClient| Client
    D["pipelineRunDetail (HTTP GET)"] -->|durableClient| Client

    Client --> Orchestrator["sopFactoryOrchestrator"]

    Orchestrator --> F[classifySection]
    Orchestrator --> G[ingestDocument]
    Orchestrator --> H[screenSourceText]
    Orchestrator --> I[generateFddArtifact]
    Orchestrator --> J[generateOntologyArtifact]
    Orchestrator --> K[generatePolicyArtifact]
    Orchestrator --> L[generateSopArtifact]
    Orchestrator --> M[buildPullRequest]
    Orchestrator --> N[openPullRequest]
:::

---

## 8. Application-by-Application Architecture

*   **`UUDRI-bill-processor-qa-01`**: 1 Function (`BillProcessor`), dotnet-isolated runtime, triggered via `blobTrigger` on connection `StorageAccount`.
*   **`UUDRI-Function-App-qa-01`**: Currently has **0 functions** deployed (idle site).
*   **`kg-event-processor-qa`**: 2 Functions (`HealthCheck`, `ProcessDMLEvent`), dotnet-isolated. Processes events from Service Bus.
*   **`func-projector-sopfactoryqao80ns`**: 1 Function (`projectOnPublish`), Node.js 20, HTTP triggered.
*   **`helios-qa-cost-ingestion`**: 1 Function (`cost_ingestion`), Python 3.11, timerTrigger running daily at 06:00 UTC. Missing App Insights.
*   **`helios-device-telemetry-qa-func`**: Currently has **0 functions** deployed.
*   **`func-orchestrator-sopfactoryqao80ns`**: 16 Functions, Node.js 20. Main Durable Functions orchestration suite.
*   **`ems-plan-narration-function-qa`**: 2 Functions (`plan_narration_agent`, `realized_kpi_listener`), Python 3.12. Outbound VNet integrated.
*   **`helios-github-activity-logger-qa-func`**: 1 Function (`github_webhook`), Python 3.11, receives GitHub payloads.
*   **`helios-ontology-event-processor-func-qa`**: 4 Functions, Python 3.11, handles Event Hub ontology streams and http endpoints.

---

## 9. SCM and Platform Configuration

| Function App | SCM Type | Linux Fx Version | Always On | FTPS | Public Network | Min TLS |
|:---|:---|:---|:---|:---|:---|:---|
| `UUDRI-bill-processor-qa-01` | VSTSRM | | FALSE | FtpsOnly | Enabled | 1.2 |
| `UUDRI-Function-App-qa-01` | VSTSRM | | FALSE | Disabled | Enabled | 1.2 |
| `kg-event-processor-qa` | None | | FALSE | FtpsOnly | Enabled | 1.2 |
| `func-projector-sopfactoryqao80ns` | None | NODE\|20 | FALSE | Disabled | Enabled | 1.2 |
| `helios-qa-cost-ingestion` | GitHubAction | PYTHON\|3.11 | FALSE | Disabled | Enabled | 1.2 |
| `helios-device-telemetry-qa-func` | None | PYTHON\|3.11 | FALSE | Disabled | Enabled | 1.2 |
| `func-orchestrator-sopfactoryqao80ns` | None | NODE\|20 | FALSE | Disabled | Enabled | 1.2 |
| `ems-plan-narration-function-qa` | None | PYTHON\|3.12 | FALSE | FtpsOnly | Enabled | 1.2 |
| `helios-github-activity-logger-qa-func` | None | PYTHON\|3.11 | FALSE | Disabled | Enabled | 1.2 |
| `helios-ontology-event-processor-func-qa` | None | PYTHON\|3.11 | FALSE | FtpsOnly | Enabled | 1.2 |

---

## 10. Observability Configuration

### 10.1 Application Insights Configuration

| Function App | App Insights Connection String | App Insights Instrumentation Key |
|:---|:---|:---|
| `UUDRI-bill-processor-qa-01` | Present | Missing |
| `UUDRI-Function-App-qa-01` | **Missing** | **Missing** |
| `kg-event-processor-qa` | Present | Present |
| `func-projector-sopfactoryqao80ns` | Present | Missing |
| `helios-qa-cost-ingestion` | **Missing** | **Missing** |
| `helios-device-telemetry-qa-func` | Present | Missing |
| `func-orchestrator-sopfactoryqao80ns` | Present | Missing |
| `ems-plan-narration-function-qa` | Present | Missing |
| `helios-github-activity-logger-qa-func` | Present | Missing |
| `helios-ontology-event-processor-func-qa` | Present | Present |

### 10.2 Azure Monitor Diagnostic Settings

| Function App | Diagnostic Settings Found | Count |
|:---|:---|:---|
| `UUDRI-bill-processor-qa-01` | No | 0 |
| `UUDRI-Function-App-qa-01` | No | 0 |
| `kg-event-processor-qa` | No | 0 |
| `func-projector-sopfactoryqao80ns` | No | 0 |
| `helios-qa-cost-ingestion` | No | 0 |
| `helios-device-telemetry-qa-func` | No | 0 |
| `func-orchestrator-sopfactoryqao80ns` | No | 0 |
| `ems-plan-narration-function-qa` | No | 0 |
| `helios-github-activity-logger-qa-func` | No | 0 |
| `helios-ontology-event-processor-func-qa` | No | 0 |

---

## 11. Identity, Access, and Security

### 11.1 Managed Identity Configuration

| Function App | Identity Type | Principal ID |
|:---|:---|:---|
| `UUDRI-bill-processor-qa-01` | None | None |
| `UUDRI-Function-App-qa-01` | None | None |
| `kg-event-processor-qa` | SystemAssigned | `c29a9c22-6787-42ea-ada3-6a68b10f7a94` |
| `func-projector-sopfactoryqao80ns` | SystemAssigned | `d3100d4b-47a7-4876-9d71-6b4d045d27f0` |
| `helios-qa-cost-ingestion` | SystemAssigned | `6664cb6f-aa2b-48ac-bbee-5bc768391e2f` |
| `helios-device-telemetry-qa-func` | SystemAssigned | `9b519b15-ebf4-4b86-a9dd-7e100ba6fac1` |
| `func-orchestrator-sopfactoryqao80ns` | SystemAssigned | `e5fc577b-6b55-4288-8c16-d190c61e95eb` |
| `ems-plan-narration-function-qa` | SystemAssigned | `30671f99-54db-40de-8865-2f2ed3d564d3` |
| `helios-github-activity-logger-qa-func` | SystemAssigned | `a84fd7de-9735-4ed7-9f89-e190f8e5bb2c` |
| `helios-ontology-event-processor-func-qa` | SystemAssigned | `dd379225-26f6-443b-9509-40a49fa48502` |

### 11.2 RBAC Role Assignments

A total of **9 RBAC role assignments** exist specifically at the Function App resource scopes in QA:
*   `kg-event-processor-qa` holds 4 assignments.
*   `helios-ontology-event-processor-func-qa` holds 2 assignments.
*   `helios-qa-cost-ingestion`, `func-orchestrator-sopfactoryqao80ns`, and `helios-github-activity-logger-qa-func` hold 1 assignment each.

### 11.3 Key Vault Access Relationships

We mapped **8 Key Vaults** in the QA subscription. The authorization profiles are detailed below:

| Key Vault | Resource Group | RBAC Enabled | Scope / Purpose |
|:---|:---|:---|:---|
| `UUDRI-Key-Vault-qa-02` | `uudri-qa-rg` | **False** (Access Policies) | Key storage for UUDRI QA apps |
| `helios-qa-backend-kv` | `helios-qa-us-west3-rg` | **True** (Azure RBAC) | Secrets management for QA backend |
| `helios-qa-agents-kv` | `helios-qa-us-west3-rg` | **True** (Azure RBAC) | Secrets for AI agents in QA |
| `helios-qa-onboarding-kv` | `helios-qa-us-west3-rg` | **True** (Azure RBAC) | Secrets for onboarding pipelines |
| `helios-qa-ui-kv` | `helios-qa-uswest3-ui` | **False** (Access Policies) | UI configurations and secrets |
| `kg-event-processor-qa-kv` | `helios-qa-us-west3-rg` | **True** (Azure RBAC) | Secrets for knowledge graph pipeline |
| `helios-qa-spkplug-pki-kv` | `helios-qa-us-west3-rg` | **False** (Access Policies) | Sparkplug PKI certificates |
| `kvsopfactoryqao80ns` | `helios-qa-us-west3-rg` | **True** (Azure RBAC) | Secrets for SOP Factory QA |

---

## 12. Network Security and Topology

### 12.1 VNet Integration & Private Endpoints
*   **VNet Integration (Outbound):** Unlike DEV (which had 0%), **1 out of 10** apps has regional outbound VNet integration configured in QA:
    *   `ems-plan-narration-function-qa` is integrated into subnet `appservice-subnet` on virtual network `helios-aks-qa-vnet`.
*   **Private Endpoints (Inbound):** None of the 10 QA apps have private endpoint connections.

### 12.2 IP Security Restrictions
*   **Inbound Restrictions:** IP security restrictions are disabled across all 10 apps (IP restriction count is 0, defaults to "Allow All").
*   **SCM Restrictions:** IP restrictions for SCM deployment endpoints are also disabled.

> ⚠️ **WARNING:** All 10 QA Function Apps are public (exposed to the open internet for inbound execution, though 1 app integrates into the VNet for outbound access).

---

## 13. Alerting and Monitoring Estate

### 13.1 Metric Alerts
We discovered **8 Metric Alert rules** matching QA function app scopes:
*   Severity: Severity 3 (Informational).
*   Condition: Triggered on `Http5xx` errors.
*   Coverage: 8 out of 10 apps are covered. The two UUDRI apps lack metric alerts.

### 13.2 Log Alerts
Log alert rules exist at the subscription level querying Log Analytics but are not bound as resource-level diagnostic settings (which are 0/10).

### 13.3 Action Groups & Alert Routing
All 8 active metric alerts route specifically to **`ag-helios-qa-ops`** (Short Name: `qa-ops`), which forwards alert notifications via a webhook receiver to Microsoft Teams channels.

### 13.4 Alert Coverage Matrix

| Function App | Metric Alerts | Log Alerts | Activity Log Alerts | Action Groups Linked | Has Active Alerts |
|:---|:---|:---|:---|:---|:---|
| `UUDRI-bill-processor-qa-01` | 0 | 0 | 0 | None | No |
| `UUDRI-Function-App-qa-01` | 0 | 0 | 0 | None | No |
| `kg-event-processor-qa` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `func-projector-sopfactoryqao80ns` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `helios-qa-cost-ingestion` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `helios-device-telemetry-qa-func` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `func-orchestrator-sopfactoryqao80ns` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `ems-plan-narration-function-qa` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `helios-github-activity-logger-qa-func` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |
| `helios-ontology-event-processor-func-qa` | 1 | 0 | 0 | `ag-helios-qa-ops` | Yes |

---

## 14. Scheduled Verification & Health Monitoring

### 14.1 Platform Health Probes
*   **0 out of 10** apps have health check paths configured (`healthCheckPath` is null/empty for all). 

### 14.2 Availability Tests
*   **0 availability web tests** (ping tests) are configured in Application Insights. 

---

## 15. Service Level Objectives & SLIs

### 15.1 Available Telemetry Metrics (SLIs)
Platform metrics are collected for the active apps:
*   `Http5xx` / `Http4xx` / `Http2xx` (Response codes)
*   `Requests` (Total execution count)
*   `AverageResponseTime` (Latency)

### 15.2 SLO Gaps
No formal Service Level Objectives (SLOs) are defined. Escalatation pathways and ownership are undocumented.

---

## 16. CI/CD and Release Orchestration

### 16.1 Repository Investigated

```
qcells-hqct / helios-plan-narration-backend
```

Deploys to: `ems-plan-narration-function-qa`

### 16.2 GitHub Actions Workflows Found

| Workflow File | Target Environment | Trigger / Rules |
|:---|:---|:---|
| `.github/workflows/deploy-function-app-qa.yml` | QA | Manual dispatch required for deployment; CI validation runs on PR/push |

### 16.3 QA Release Flow

```
GitHub push/PR or manual dispatch
         |
         v
CI Phase (Syntax check)
  pip install & python -m compileall
         |
         v
Deploy Phase (workflow_dispatch ONLY)
  Installs Azure CLI & Logs in (Service Principal credentials)
  Configure settings (Key Vault helios-qa-backend-kv + GitHub env vars)
  Azure/functions-action (ZipDeploy)
         |
         v
Target: ems-plan-narration-function-qa
         |
         v
Post-deploy validation
  Verifies running state and registered functions count
```

### 16.4 GitHub Repository Secrets (Names Only)

*   `AZURE_CLIENT_ID`
*   `AZURE_CLIENT_SECRET`
*   `AZURE_SUBSCRIPTION_ID`
*   `AZURE_TENANT_ID`

### 16.5 GitHub Repository Variables

The following non-sensitive configurations are set directly as variables in the workflow:
*   `APIM_CLIENT_ID` (Value: `07a80643-123e-42e0-b47c-a03ab2ed0a26`)
*   `APIM_SCOPE` (Value: `api://helios-qa-apim-oauth-external/.default`)
*   `APIM_TOKEN_URL`
*   `MICROSOFT_FOUNDRY_PROJECT_ENDPOINT`
*   `OBJECT_STORE_SERVICE_URL`
*   `HITL_SERVICE_URL`
*   `EMS_PLAN_NARRATION_AGENT_NAME`
*   `EVENT_HUB_NAME`
*   `EVENTHUB_CONSUMER_GROUP`
*   `REALIZED_KPI_EVENT_HUB_NAME`
*   `REALIZED_KPI_EVENTHUB_CONSUMER_GROUP`

### 16.6 SRE Observations on Release Pipeline
1.  **No Deploy on Merge:** While the workflow is triggered by pushes/PRs to `main`, the `deploy` job runs *only* if `github.event_name == 'workflow_dispatch'`. Automatic push-to-main does not deploy to QA.
2.  **No Automated Testing in Deploy:** The deploy workflow does not run unit tests (`pytest`). Code verification is limited to syntax validation.
3.  **No Immutable Artifact Promotion:** The deployment builds and packages code from the branch directly rather than promoting a pre-built artifact from DEV.
4.  **GitHub Environment Gates:** Employs `environment: qa` for protection rules, allowing manual approvals or gates to be configured.

### 16.7 Deployment History Evidence

Queried via `Microsoft.Web/sites/<app>/deployments`:

**UUDRI-Function-App-qa-01**

| Deployer | Most Recent | Method |
|:---|:---|:---|
| VSTS_FUNCTIONS_V1 | 2026-01-29 | Azure DevOps Release |

**kg-event-processor-qa**

| Deployer | Most Recent | Method |
|:---|:---|:---|
| az_cli_functions | 2026-08-18 | Push deployment (CLI) |
| az_cli_functions | 2026-08-08 | Push deployment (CLI) |
| az_cli_functions | 2026-08-04 | Push deployment (CLI) |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-04-14 | GitHub Actions |

**ems-plan-narration-function-qa**

| Deployer | Most Recent | Method |
|:---|:---|:---|
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-08-17 | GitHub Actions |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-08-07 | GitHub Actions |
| az_cli_functions | 2026-08-06 | Push deployment (CLI) |

*Note: Other QA Function Apps returned empty deployment logs via the ARM API, indicating their deployment logs have either expired or were performed through external deployment methods not recorded in ARM.*

---

## 17. Logic App / Workflow App Discovery

Four Logic Apps (workflow apps) were identified in the QA resource group:

| Resource Name | Resource Type | Kind | Location | Resource Group |
|:---|:---|:---|:---|:---|
| `helios-qa-pipeline-monitor` | Microsoft.Web/sites | functionapp,workflowapp | westus3 | `helios-qa-us-west3-rg` |
| `helios-data-eventstream-monitor-qa` | Microsoft.Web/sites | functionapp,workflowapp | westus3 | `helios-qa-us-west3-rg` |
| `helios-data-market-eh-eventstream-monitor-qa` | Microsoft.Web/sites | functionapp,workflowapp | westus3 | `helios-qa-us-west3-rg` |
| `helios-data-weather-eh-eventstream-monitor-qa` | Microsoft.Web/sites | functionapp,workflowapp | westus3 | `helios-qa-us-west3-rg` |

---

## 18. Current QA Architecture Diagram

```
                               GitHub Repositories
                    +--------------------------------------+
                    | qcells-hqct/helios-plan-narration-   |
                    |   backend                            |
                    | (Other Repositories Unknown)         |
                    +------------------+-------------------+
                                       |
                     GitHub Actions / Azure DevOps Releases
                                       |
                                       v
                    +--------------------------------------+
                    |     Azure Function Apps -- QA        |
                    |          10 Apps / 28 Functions      |
                    +------------------+-------------------+
                                       |
            +--------------------------+-------------------------+
            |                          |                         |
            v                          v                         v
      HTTP-triggered             Event-driven             Scheduled / Durable
        Functions                  Functions                  Functions
            |                          |                         |
            v                          v                         v
     APIs / Webhooks            +-----------+              Timer: 06:00 UTC
     Health checks              |           |              Durable orchestration
            |                   v           v                    |
            |              Event Hub   Service Bus         SOP Factory
            |                   |           |              (16 functions)
            |                   v           v                    |
            |           ontology_event  ProcessDMLEvent          |
            |           plan_narration  (KG events)              |
            |           realized_kpi                             |
            |                                                    |
            +----------------------------------------------------+
                                       |
                                       v
                           Application Dependencies
                                       |
                                       v
                          Observability & Network Layer
                    +------------------+-------------------+
                    |                                      |
              App Insights                          Azure Monitor
           (8/10 apps configured)           (0/10 diagnostic settings, 
                                             8/10 metric alerts)
                    |                                      |
                    +--------------------------------------+
                                       |
                             1/10 apps VNet Integrated
                           (ems-plan-narration-function-qa)
```

---

## 19. Capability Matrix Update (QA Environment)

Based on QA findings, the capability matrix rows for Azure Functions are now fully mapped:

| Capability | Status | Evidence |
|:---|:---|:---|
| **Deploy on merge** | **Mapped (QA)** | Mapped SCM types (`VSTSRM`, `GitHubAction`, `None`) and manual-only triggers for QA pipelines. |
| **Promotion gates** | **Mapped (QA)** | Confirmed separate repo workflow configuration and manual dispatch requirement for QA environments. |
| **Scheduled verification** | **Mapped (QA)** | Mapped 0/10 health checks or ping tests. Gaps saved. |
| **Alerting** | **Mapped (QA)** | 8/10 apps mapped to metric alerts routing to `ag-helios-qa-ops`. |
| **Observability + cost** | **Mapped (QA)** | 8/10 App Insights configured (cost-ingestion and idle UUDRI app missing). |
| **Reporting / audit** | **Mapped (QA)** | Mapped Managed Identity, RBAC roles, and Key Vault access types. |

---

*Evidence verified and recorded by SRE team. No remediation or implementation changes have been performed during this discovery phase.*

