# Azure Functions -- SRE Discovery Report (DEV Environment)

**Project:** SRE-Project  
**Environment:** DEV  
**Scope:** Azure Functions (~87 across 3 environments; 10 Function Apps / 30 Functions in DEV)  
**Status:** Discovery complete -- no remediation or implementation changes have been performed  
**Date:** August 2026  

---

[[_TOC_]]

---

## 1. Background and Context

### 1.1 Why This Investigation Exists

The SRE team maintains a **Release Orchestration Capability Matrix** that tracks the operational maturity of every platform component across six capability columns:

| Column | Description |
|---|---|
| Deploy on merge | Does the component auto-deploy when code is merged? |
| Promotion gates (dev -> QA -> prod) | Is there a controlled gate between environments? |
| Scheduled verification | Are there recurring tests that validate the running system? |
| Alerting | Are alert rules and action groups configured? |
| Observability + cost | Is telemetry flowing and are costs tracked? |
| Reporting / audit | Are deployments and changes auditable? |

The matrix covers:

- k8s + ArgoCD apps (incl. Temporal)
- Frontend (React, IBM components)
- Foundry models
- **Azure Functions (~87 across 3 envs)** &larr; this ticket
- Flyway DB migrations (5 DBs per env?)
- Non-AKS Azure estate (86 resources)
- Fabric / Knowledge Graphs

When SRE reviewed the matrix, **Azure Functions was marked `unknown` in every column**. No one on the team could explain what the Functions did, how they were triggered, what they depended on, or how they were deployed.

We selected Azure Functions as our first ticket and chose **DEV** as the initial deep-dive environment.

### 1.2 Investigation Principle

The project follows a strict evidence-first methodology:

```
DISCOVER
   |
   v
VERIFY
   |
   v
SAVE EVIDENCE
   |
   v
MAP ARCHITECTURE
   |
   v
EXPLAIN CURRENT STATE
   |
   v
IDENTIFY OPERATIONAL GAPS
   |
   v
GET ENGINEERING DECISION
   |
   v
IMPLEMENT LATER
```

**Rule:** Do not conclude from a single API call. Cross-check related configuration and preserve the evidence.

**Rule:** No remediation is performed simply because a gap is discovered. The current phase is to establish a defensible current-state SRE baseline.

### 1.3 How to Read This Document

This document is intended for anyone -- including engineers who have never worked with Azure Functions in this project. It is structured in layers, from the broadest view down to the most granular detail:

1. First we explain **what Azure Function Apps and Functions are** in our context.
2. Then we list **exactly what exists** in DEV.
3. Then we explain **how each Function is invoked** (triggers and bindings).
4. Then we show **what external systems each Function depends on** (Event Hubs, Service Bus, Blob Storage, Timers).
5. Then we map **how they relate to each other** (especially the Durable Functions orchestration).
6. Then we show **how code gets deployed** (CI/CD).
7. Then we assess **what observability exists** (Application Insights, diagnostic settings).
8. Finally we identify **what is still unknown** and what the next investigation phase will cover.

---

## 2. Key Concepts for Non-Azure-Functions Engineers

If you already understand Azure Functions, skip to [Section 3](#3-dev-environment-scope).

### 2.1 What is an Azure Function App?

An **Azure Function App** is a hosting container in Azure that runs one or more individual **Functions**. Think of it like a microservice deployment unit -- it has its own runtime, configuration, scaling rules, and deployment lifecycle. In our DEV environment, we have **10 Function Apps**.

### 2.2 What is a Function?

A **Function** is a single unit of code inside a Function App. Each Function is invoked by a **trigger** and may have additional **bindings** that connect it to external services. In our DEV environment, we have **30 Functions** distributed across the 10 Function Apps.

### 2.3 What is a Trigger?

A **trigger** is the event that causes a Function to execute. A Function has exactly one trigger. Common trigger types in our estate:

| Trigger Type | What It Means |
|---|---|
| `httpTrigger` | The function runs when an HTTP request hits its URL endpoint. |
| `eventHubTrigger` | The function runs when a message arrives in an Azure Event Hub. |
| `blobTrigger` | The function runs when a file (blob) is created or updated in Azure Storage. |
| `serviceBusTrigger` | The function runs when a message arrives on a Service Bus topic/subscription. |
| `timerTrigger` | The function runs on a cron-like schedule. |
| `activityTrigger` | The function is a "worker" step inside a Durable Functions orchestration. |
| `orchestrationTrigger` | The function is the "coordinator" that sequences activity functions. |
| `durableClient` | An additional binding that allows the function to start/query orchestrations. |

### 2.4 What is a Binding?

A **binding** is a declarative connection between a Function and an external resource. Bindings have a direction: `IN` (input) or `OUT` (output). A Function can have multiple bindings. For example, `intakeDocument` has both an `httpTrigger` (its trigger) and a `durableClient` (to start an orchestration).

**Important:** 30 Functions does not mean 30 binding rows. Some Functions have multiple bindings, which means the binding inventory contains more rows than there are Functions.

### 2.5 What are Durable Functions?

Durable Functions is a pattern for writing stateful, long-running workflows in Azure Functions. The pattern uses three roles:

1. **Client function** -- receives the initial request (HTTP, Blob, etc.) and starts an orchestration.
2. **Orchestrator function** -- defines the workflow by calling activity functions in sequence or in parallel.
3. **Activity function** -- performs a single unit of work (e.g., classify a document, generate an artifact).

Our SOP Factory app (`func-orchestrator-sopfactorydevmlel9`) uses this pattern extensively.

---

## 3. DEV Environment Scope

### 3.1 Numbers at a Glance

| Metric | Value |
|---|---|
| Function Apps in DEV | **10** |
| Registered Functions in DEV | **30** |
| Logic Apps / Workflow Apps in DEV | **3** |
| Runtimes in use | `Python 3.11`, `Python 3.12`, `Node.js 20`, `.NET isolated` |
| Azure subscription | `a6498579-cfb7-41e9-a957-14375196a386` |
| Primary resource group | `helios-dev-us-west3-rg` |
| Secondary resource group | `uudri-dev-rg` |

### 3.2 Broader Context

The broader inventory discussion identified approximately **27 Function Apps across DEV/QA/PROD**. The capability matrix separately refers to approximately **87 Azure Functions** across three environments. These broader figures have not yet been reconciled and remain an open discovery item. This report covers only the **10 DEV Function Apps and 30 DEV Functions** that were verified through direct Azure API queries.

---

## 4. Investigation Method

We mapped the system in 19 layers. Each layer answers three questions:

1. **What exists?**
2. **How did we verify it?**
3. **Where is the evidence?**

| # | Layer | Status |
|---|---|---|
| 1 | Scope and resource inventory | Completed |
| 2 | Function App inventory | Completed |
| 3 | Individual Function inventory | Completed |
| 4 | Runtime / language | Completed |
| 5 | Trigger and binding mapping | Completed |
| 6 | Event / data dependencies | Completed |
| 7 | Platform / SCM configuration | Completed |
| 8 | Deployment history | Completed |
| 9 | CI/CD / release orchestration | Completed (for examined repo) |
| 10 | Application Insights configuration | Completed |
| 11 | Azure Monitor diagnostic settings | Completed |
| 12 | Managed identity | **Not yet started** |
| 13 | RBAC | **Not yet started** |
| 14 | Key Vault / access relationships | **Not yet started** |
| 15 | Networking | **Not yet started** |
| 16 | Alerts and Action Groups | **Not yet started** |
| 17 | Runtime health / metrics | **Not yet started** |
| 18 | Ownership / SLO / incident path | **Not yet started** |
| 19 | Final end-to-end architecture | **Not yet started** |

---

## 5. Function App Inventory (Layer 1-2)

### 5.1 Complete List of DEV Function Apps

| # | Function App | Resource Group | Purpose (inferred from discovery) |
|---|---|---|---|
| 1 | `uudri-bill-processor-dev` | `uudri-dev-rg` | Processes utility bills from Blob Storage |
| 2 | `helios-ontology-event-processor-func` | `helios-dev-us-west3-rg` | Processes ontology events from Event Hub; exposes test/health HTTP endpoints |
| 3 | `helios-github-activity-logger-dev-func` | `helios-dev-us-west3-rg` | Receives GitHub webhook payloads |
| 4 | `func-projector-sopfactorydevmlel9` | `helios-dev-us-west3-rg` | HTTP-triggered publish projection (SOP Factory) |
| 5 | `helios-dev-cost-ingestion` | `helios-dev-us-west3-rg` | Scheduled daily cost data ingestion |
| 6 | `func-orchestrator-sopfactorydevmlel9` | `helios-dev-us-west3-rg` | SOP Factory Durable Functions orchestration engine (16 functions) |
| 7 | `helios-device-telemetry-dev-func` | `helios-dev-us-west3-rg` | Classifies new devices and telemetry from Event Hub |
| 8 | `ems-plan-narration-function` | `helios-dev-us-west3-rg` | Narrates energy plans and listens for realized KPI events from Event Hub |
| 9 | `kg-event-processor-dev` | `helios-dev-us-west3-rg` | Processes Knowledge Graph DML events from Service Bus |
| 10 | `UUDRI-Bill-Processor-dev-01` | `helios-dev-us-west3-rg` | Processes utility bills from Blob Storage (second instance) |

### 5.2 How the Inventory Was Discovered

Function Apps were listed using:

```powershell
$apps = az functionapp list `
  --query "[?contains(resourceGroup,'helios-dev') || contains(resourceGroup,'uudri-dev')].{Name:name,RG:resourceGroup}" `
  -o json |
  ConvertFrom-Json
```

Individual Functions were discovered via the ARM REST API (`Microsoft.Web/sites/<app>/functions`) because `az functionapp function list` encountered Azure CLI/Python/JMESPath problems:

```powershell
$url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/functions?api-version=2022-03-01"
$json = az rest --method get --url $url -o json
```

### 5.3 Function App to Function Count

| Function App | Function Count |
|---|---|
| `uudri-bill-processor-dev` | 1 |
| `helios-ontology-event-processor-func` | 4 |
| `helios-github-activity-logger-dev-func` | 1 |
| `func-projector-sopfactorydevmlel9` | 1 |
| `helios-dev-cost-ingestion` | 1 |
| `func-orchestrator-sopfactorydevmlel9` | **16** |
| `helios-device-telemetry-dev-func` | 1 |
| `ems-plan-narration-function` | 2 |
| `kg-event-processor-dev` | 2 |
| `UUDRI-Bill-Processor-dev-01` | 1 |
| **TOTAL** | **30** |

---

## 6. Complete 30-Function Inventory (Layer 3-4)

The following table lists every Function discovered in DEV, its parent Function App, resource group, runtime language, ARM resource type, and whether it is currently disabled.

| # | Function App | Resource Group | Function Name | Type | Language | Disabled |
|---|---|---|---|---|---|---|
| 1 | uudri-bill-processor-dev | uudri-dev-rg | BillProcessor | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |
| 2 | helios-ontology-event-processor-func | helios-dev-us-west3-rg | health_check | Microsoft.Web/sites/functions | python | FALSE |
| 3 | helios-ontology-event-processor-func | helios-dev-us-west3-rg | ontology_event_processor | Microsoft.Web/sites/functions | python | FALSE |
| 4 | helios-ontology-event-processor-func | helios-dev-us-west3-rg | test_relationship_processor | Microsoft.Web/sites/functions | python | FALSE |
| 5 | helios-ontology-event-processor-func | helios-dev-us-west3-rg | test_schema_processor | Microsoft.Web/sites/functions | python | FALSE |
| 6 | helios-github-activity-logger-dev-func | helios-dev-us-west3-rg | github_webhook | Microsoft.Web/sites/functions | python | FALSE |
| 7 | func-projector-sopfactorydevmlel9 | helios-dev-us-west3-rg | projectOnPublish | Microsoft.Web/sites/functions | node | FALSE |
| 8 | helios-dev-cost-ingestion | helios-dev-us-west3-rg | cost_ingestion | Microsoft.Web/sites/functions | python | FALSE |
| 9 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | cancelOnboardingSession | Microsoft.Web/sites/functions | node | FALSE |
| 10 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | classifySection | Microsoft.Web/sites/functions | node | FALSE |
| 11 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | generateFddArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 12 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | generateOntologyArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 13 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | generatePolicyArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 14 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | generateSopArtifact | Microsoft.Web/sites/functions | node | FALSE |
| 15 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | ingestDocument | Microsoft.Web/sites/functions | node | FALSE |
| 16 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | intakeDocument | Microsoft.Web/sites/functions | node | FALSE |
| 17 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | onSourceDocument | Microsoft.Web/sites/functions | node | FALSE |
| 18 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | openPullRequest | Microsoft.Web/sites/functions | node | FALSE |
| 19 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | pipelineRunDetail | Microsoft.Web/sites/functions | node | FALSE |
| 20 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | pipelineRuns | Microsoft.Web/sites/functions | node | FALSE |
| 21 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | pipelineSmeQueue | Microsoft.Web/sites/functions | node | FALSE |
| 22 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | publishOnboardingSession | Microsoft.Web/sites/functions | node | FALSE |
| 23 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | screenSourceText | Microsoft.Web/sites/functions | node | FALSE |
| 24 | func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | sopFactoryOrchestrator | Microsoft.Web/sites/functions | node | FALSE |
| 25 | helios-device-telemetry-dev-func | helios-dev-us-west3-rg | new_device_and_telemetry_classification_agent | Microsoft.Web/sites/functions | python | FALSE |
| 26 | ems-plan-narration-function | helios-dev-us-west3-rg | plan_narration_agent | Microsoft.Web/sites/functions | python | FALSE |
| 27 | ems-plan-narration-function | helios-dev-us-west3-rg | realized_kpi_listener | Microsoft.Web/sites/functions | python | FALSE |
| 28 | kg-event-processor-dev | helios-dev-us-west3-rg | HealthCheck | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |
| 29 | kg-event-processor-dev | helios-dev-us-west3-rg | ProcessDMLEvent | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |
| 30 | UUDRI-Bill-Processor-dev-01 | helios-dev-us-west3-rg | BillProcessor | Microsoft.Web/sites/functions | dotnet-isolated | FALSE |

### 6.1 Functions Grouped by Runtime

```
dotnet-isolated (4 functions across 3 apps)
    +-- BillProcessor          (uudri-bill-processor-dev)
    +-- HealthCheck            (kg-event-processor-dev)
    +-- ProcessDMLEvent        (kg-event-processor-dev)
    +-- BillProcessor          (UUDRI-Bill-Processor-dev-01)

python (9 functions across 5 apps)
    +-- health_check                                   (helios-ontology-event-processor-func)
    +-- ontology_event_processor                       (helios-ontology-event-processor-func)
    +-- test_relationship_processor                    (helios-ontology-event-processor-func)
    +-- test_schema_processor                          (helios-ontology-event-processor-func)
    +-- github_webhook                                 (helios-github-activity-logger-dev-func)
    +-- cost_ingestion                                 (helios-dev-cost-ingestion)
    +-- new_device_and_telemetry_classification_agent   (helios-device-telemetry-dev-func)
    +-- plan_narration_agent                           (ems-plan-narration-function)
    +-- realized_kpi_listener                          (ems-plan-narration-function)

node (17 functions across 2 apps)
    +-- projectOnPublish           (func-projector-sopfactorydevmlel9)
    +-- cancelOnboardingSession    (func-orchestrator-sopfactorydevmlel9)
    +-- classifySection            (func-orchestrator-sopfactorydevmlel9)
    +-- generateFddArtifact        (func-orchestrator-sopfactorydevmlel9)
    +-- generateOntologyArtifact   (func-orchestrator-sopfactorydevmlel9)
    +-- generatePolicyArtifact     (func-orchestrator-sopfactorydevmlel9)
    +-- generateSopArtifact        (func-orchestrator-sopfactorydevmlel9)
    +-- ingestDocument             (func-orchestrator-sopfactorydevmlel9)
    +-- intakeDocument             (func-orchestrator-sopfactorydevmlel9)
    +-- onSourceDocument           (func-orchestrator-sopfactorydevmlel9)
    +-- openPullRequest            (func-orchestrator-sopfactorydevmlel9)
    +-- pipelineRunDetail          (func-orchestrator-sopfactorydevmlel9)
    +-- pipelineRuns               (func-orchestrator-sopfactorydevmlel9)
    +-- pipelineSmeQueue           (func-orchestrator-sopfactorydevmlel9)
    +-- publishOnboardingSession   (func-orchestrator-sopfactorydevmlel9)
    +-- screenSourceText           (func-orchestrator-sopfactorydevmlel9)
    +-- sopFactoryOrchestrator     (func-orchestrator-sopfactorydevmlel9)
```

---

## 7. Trigger and Binding Inventory (Layer 5)

### 7.1 Trigger Type Summary

| Trigger Type | Count | What It Means |
|---|---|---|
| httpTrigger | 12 | Functions invoked by HTTP requests (APIs, webhooks, health checks) |
| activityTrigger | 8 | Durable Functions worker steps called by the orchestrator |
| durableClient | 5 | Functions that can start or query Durable orchestrations (appears as a second binding) |
| eventHubTrigger | 4 | Functions that process messages from Azure Event Hubs |
| blobTrigger | 3 | Functions that react when a file appears in Azure Blob Storage |
| serviceBusTrigger | 1 | Function that processes messages from Azure Service Bus |
| orchestrationTrigger | 1 | The single Durable Functions orchestrator that coordinates activities |
| timerTrigger | 1 | Function that runs on a cron schedule |

### 7.2 Complete Binding Inventory

This table shows every inbound binding discovered via the ARM API. Note that some Functions appear on multiple rows because they have multiple bindings (e.g., `intakeDocument` has both `httpTrigger` and `durableClient`).

| Function App | Function Name | Language | Trigger Type | Direction | Auth Level | Methods | Route | Connection | Event Hub Name | Queue Name | Topic Name | Subscription Name | Blob Path | Schedule | Disabled |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| uudri-bill-processor-dev | BillProcessor | dotnet-isolated | blobTrigger | In | | | | StorageAccount | | | | | | | FALSE |
| helios-ontology-event-processor-func | health_check | python | httpTrigger | IN | ANONYMOUS | GET | health | | | | | | | | FALSE |
| helios-ontology-event-processor-func | ontology_event_processor | python | eventHubTrigger | IN | | | | EventHubConnection | %EVENT_HUB_NAME% | | | | | | FALSE |
| helios-ontology-event-processor-func | test_relationship_processor | python | httpTrigger | IN | FUNCTION | POST | test_relationship | | | | | | | | FALSE |
| helios-ontology-event-processor-func | test_schema_processor | python | httpTrigger | IN | FUNCTION | POST | test_schema | | | | | | | | FALSE |
| helios-github-activity-logger-dev-func | github_webhook | python | httpTrigger | IN | FUNCTION | POST | github/webhook | | | | | | | | FALSE |
| func-projector-sopfactorydevmlel9 | projectOnPublish | node | httpTrigger | in | function | POST | | | | | | | | | FALSE |
| helios-dev-cost-ingestion | cost_ingestion | python | timerTrigger | in | | | | | | | | | | 0 0 6 * * * | FALSE |
| func-orchestrator-sopfactorydevmlel9 | cancelOnboardingSession | node | httpTrigger | in | function | POST | v1/onboarding/{session_id}/cancel | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | classifySection | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | generateFddArtifact | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | generateOntologyArtifact | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | generatePolicyArtifact | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | generateSopArtifact | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | ingestDocument | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | intakeDocument | node | httpTrigger | in | function | POST | v1/documents | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | intakeDocument | node | durableClient | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | onSourceDocument | node | blobTrigger | in | | | | SOURCE_STORAGE | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | onSourceDocument | node | durableClient | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | openPullRequest | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | pipelineRunDetail | node | httpTrigger | in | function | GET | v1/pipeline/runs/{instanceId} | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | pipelineRunDetail | node | durableClient | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | pipelineRuns | node | httpTrigger | in | function | GET | v1/pipeline/runs | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | pipelineRuns | node | durableClient | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | pipelineSmeQueue | node | httpTrigger | in | function | GET | v1/pipeline/sme-queue | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | pipelineSmeQueue | node | durableClient | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | publishOnboardingSession | node | httpTrigger | in | function | POST | v1/onboarding/{session_id}/publish | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | screenSourceText | node | activityTrigger | in | | | | | | | | | | | FALSE |
| func-orchestrator-sopfactorydevmlel9 | sopFactoryOrchestrator | node | orchestrationTrigger | in | | | | | | | | | | | FALSE |
| helios-device-telemetry-dev-func | new_device_and_telemetry_classification_agent | python | eventHubTrigger | IN | | | | EVENT_HUB_CONNECTION_STRING | %EVENT_HUB_NAME% | | | | | | FALSE |
| ems-plan-narration-function | plan_narration_agent | python | eventHubTrigger | IN | | | | EVENT_HUB_CONNECTION_STRING | %EVENT_HUB_NAME% | | | | | | FALSE |
| ems-plan-narration-function | realized_kpi_listener | python | eventHubTrigger | IN | | | | REALIZED_KPI_EVENT_HUB_CONNECTION_STRING | %REALIZED_KPI_EVENT_HUB_NAME% | | | | | | FALSE |
| kg-event-processor-dev | HealthCheck | dotnet-isolated | httpTrigger | In | Anonymous | get | health | | | | | | | | FALSE |
| kg-event-processor-dev | ProcessDMLEvent | dotnet-isolated | serviceBusTrigger | In | | | | SERVICEBUS_CONNECTION_STRING | | | helios-knowledgegraph-events | kg-event-processor | | | FALSE |
| UUDRI-Bill-Processor-dev-01 | BillProcessor | dotnet-isolated | blobTrigger | In | | | | StorageAccount | | | | | | | FALSE |

### 7.3 Functions With Multiple Bindings

These five Functions each have two inbound bindings. This means they appear as two rows in the binding inventory above. They all combine an entry trigger with a `durableClient` binding so that they can start or query Durable orchestrations:

| Function | Binding 1 (Entry Trigger) | Binding 2 |
|---|---|---|
| `intakeDocument` | httpTrigger (POST, route: v1/documents) | durableClient |
| `onSourceDocument` | blobTrigger (connection: SOURCE_STORAGE) | durableClient |
| `pipelineRunDetail` | httpTrigger (GET, route: v1/pipeline/runs/{instanceId}) | durableClient |
| `pipelineRuns` | httpTrigger (GET, route: v1/pipeline/runs) | durableClient |
| `pipelineSmeQueue` | httpTrigger (GET, route: v1/pipeline/sme-queue) | durableClient |

---

## 8. External Dependencies and Data Flows (Layer 6)

This section isolates every Function that depends on an external Azure service (Event Hub, Service Bus, Blob Storage, or Timer). These are the Functions whose availability depends on systems outside the Function App itself.

### 8.1 Event Dependencies Inventory

| Function App | Function Name | Trigger Type | Connection Setting | Event Hub Name | Topic Name | Subscription Name | Schedule |
|---|---|---|---|---|---|---|---|
| uudri-bill-processor-dev | BillProcessor | blobTrigger | StorageAccount | | | | |
| helios-ontology-event-processor-func | ontology_event_processor | eventHubTrigger | EventHubConnection | %EVENT_HUB_NAME% | | | |
| helios-dev-cost-ingestion | cost_ingestion | timerTrigger | | | | | 0 0 6 * * * |
| func-orchestrator-sopfactorydevmlel9 | onSourceDocument | blobTrigger | SOURCE_STORAGE | | | | |
| helios-device-telemetry-dev-func | new_device_and_telemetry_classification_agent | eventHubTrigger | EVENT_HUB_CONNECTION_STRING | %EVENT_HUB_NAME% | | | |
| ems-plan-narration-function | plan_narration_agent | eventHubTrigger | EVENT_HUB_CONNECTION_STRING | %EVENT_HUB_NAME% | | | |
| ems-plan-narration-function | realized_kpi_listener | eventHubTrigger | REALIZED_KPI_EVENT_HUB_CONNECTION_STRING | %REALIZED_KPI_EVENT_HUB_NAME% | | | |
| kg-event-processor-dev | ProcessDMLEvent | serviceBusTrigger | SERVICEBUS_CONNECTION_STRING | | helios-knowledgegraph-events | kg-event-processor | |
| UUDRI-Bill-Processor-dev-01 | BillProcessor | blobTrigger | StorageAccount | | | | |

**Note on placeholders:** The binding inventory exposes configuration placeholders such as `%EVENT_HUB_NAME%` and `%REALIZED_KPI_EVENT_HUB_NAME%`. These are intentionally recorded as configuration evidence. We do not replace them with assumed resource names. The actual runtime values come from the Function App's application settings.

### 8.2 Event Hub Data Flows

Four Functions consume messages from Azure Event Hubs. They are distributed across three separate Function Apps:

```
Azure Event Hub(s)
    |
    +---> ontology_event_processor
    |       App: helios-ontology-event-processor-func
    |       Connection: EventHubConnection
    |       Hub name: %EVENT_HUB_NAME%
    |
    +---> new_device_and_telemetry_classification_agent
    |       App: helios-device-telemetry-dev-func
    |       Connection: EVENT_HUB_CONNECTION_STRING
    |       Hub name: %EVENT_HUB_NAME%
    |
    +---> plan_narration_agent
    |       App: ems-plan-narration-function
    |       Connection: EVENT_HUB_CONNECTION_STRING
    |       Hub name: %EVENT_HUB_NAME%
    |
    +---> realized_kpi_listener
            App: ems-plan-narration-function
            Connection: REALIZED_KPI_EVENT_HUB_CONNECTION_STRING
            Hub name: %REALIZED_KPI_EVENT_HUB_NAME%
```

**What this tells us:** `realized_kpi_listener` uses a different connection string and Event Hub name than the other three Event Hub functions, meaning it likely connects to a separate Event Hub instance dedicated to KPI data.

### 8.3 Service Bus Data Flow

One Function consumes from Azure Service Bus using a topic/subscription model:

```
Azure Service Bus
    |
    v
Topic: helios-knowledgegraph-events
    |
    v
Subscription: kg-event-processor
    |
    v
ProcessDMLEvent (function)
    |
    v
kg-event-processor-dev (app)
    |
Connection: SERVICEBUS_CONNECTION_STRING
```

**What this tells us:** The Knowledge Graph event processor listens to a specific Service Bus topic (`helios-knowledgegraph-events`) through a dedicated subscription (`kg-event-processor`). This is a standard pub/sub pattern where the topic may have other subscribers in other environments or services.

### 8.4 Blob Storage Data Flows

Three Functions react to files being placed in Azure Blob Storage:

```
Azure Blob Storage
    |
    +---> BillProcessor
    |       App: uudri-bill-processor-dev
    |       Connection: StorageAccount
    |
    +---> onSourceDocument
    |       App: func-orchestrator-sopfactorydevmlel9
    |       Connection: SOURCE_STORAGE
    |       (also has a durableClient binding to start SOP Factory orchestration)
    |
    +---> BillProcessor
            App: UUDRI-Bill-Processor-dev-01
            Connection: StorageAccount
```

**What this tells us:**
- There are two separate BillProcessor apps (`uudri-bill-processor-dev` and `UUDRI-Bill-Processor-dev-01`), both triggered by blob storage using the same `StorageAccount` connection name. The relationship between these two apps (active/active, migration, or versioning) has not been determined in this discovery phase.
- The `onSourceDocument` function uses a different storage connection (`SOURCE_STORAGE`), and when a blob arrives it starts a Durable Functions orchestration in the SOP Factory.
- The exact blob paths were not exposed in the current inventory output and should not be assumed.

### 8.5 Timer (Scheduled) Data Flow

One Function runs on a fixed schedule:

```
Timer: 0 0 6 * * *
    |
    v
cost_ingestion (function)
    |
    v
helios-dev-cost-ingestion (app)
```

**Schedule interpretation:** `0 0 6 * * *` means "run every day at 06:00 UTC." This is a daily cost ingestion job.

---

## 9. Durable Functions Architecture -- SOP Factory (Layer 5-6)

The `func-orchestrator-sopfactorydevmlel9` Function App is the most complex component discovered. It contains **16 of the 30 Functions** and implements the **Durable Functions** pattern for document processing orchestration. This section explains how all 16 Functions relate to each other.

### 9.1 Role Classification

| Role | Functions | What They Do |
|---|---|---|
| **HTTP API endpoints** (entry points) | `intakeDocument`, `pipelineRunDetail`, `pipelineRuns`, `pipelineSmeQueue`, `cancelOnboardingSession`, `publishOnboardingSession` | Accept HTTP requests from external clients. Some of these also start or query Durable orchestrations via `durableClient` binding. |
| **Blob entry point** | `onSourceDocument` | Reacts when a document is uploaded to Blob Storage (`SOURCE_STORAGE` connection). Starts a Durable orchestration. |
| **Orchestrator** | `sopFactoryOrchestrator` | Coordinates the entire document processing pipeline by calling activity functions in sequence/parallel. |
| **Activity workers** | `classifySection`, `ingestDocument`, `screenSourceText`, `generateFddArtifact`, `generateOntologyArtifact`, `generatePolicyArtifact`, `generateSopArtifact`, `openPullRequest` | Perform individual processing steps. Called by the orchestrator. |

### 9.2 Execution Flow

```
  Entry Points (how work enters the system)
  ==========================================

  HTTP POST /v1/documents             Blob upload to SOURCE_STORAGE
       |                                     |
       v                                     v
  intakeDocument                       onSourceDocument
  (httpTrigger + durableClient)        (blobTrigger + durableClient)
       |                                     |
       +------------------+------------------+
                          |
                          v
                   Starts Orchestration
                          |
                          v
  Orchestrator
  ==========================================
                sopFactoryOrchestrator
                (orchestrationTrigger)
                          |
         +----------------+----------------+
         |                |                |
         v                v                v
   classifySection  ingestDocument  screenSourceText
   (activityTrigger) (activityTrigger) (activityTrigger)
         |                |                |
         +----------------+----------------+
                          |
           +--------------+--------------+--------------+
           |              |              |              |
           v              v              v              v
   generateFdd    generateOntology  generatePolicy  generateSop
   Artifact       Artifact          Artifact        Artifact
   (activity)     (activity)        (activity)      (activity)
                          |
                          v
                   openPullRequest
                    (activityTrigger)


  Query/Management Endpoints
  ==========================================

  GET  /v1/pipeline/runs                  --> pipelineRuns (httpTrigger + durableClient)
  GET  /v1/pipeline/runs/{instanceId}     --> pipelineRunDetail (httpTrigger + durableClient)
  GET  /v1/pipeline/sme-queue             --> pipelineSmeQueue (httpTrigger + durableClient)
  POST /v1/onboarding/{session_id}/cancel --> cancelOnboardingSession (httpTrigger)
  POST /v1/onboarding/{session_id}/publish--> publishOnboardingSession (httpTrigger)
```

### 9.3 How This Works End-to-End

1. A document arrives either via HTTP POST to `/v1/documents` (handled by `intakeDocument`) or via a blob upload to `SOURCE_STORAGE` (handled by `onSourceDocument`).
2. Both entry points use their `durableClient` binding to start a new instance of the `sopFactoryOrchestrator`.
3. The orchestrator executes activity functions in order: first `classifySection`, `ingestDocument`, `screenSourceText` to analyze the document, then `generateFddArtifact`, `generateOntologyArtifact`, `generatePolicyArtifact`, `generateSopArtifact` to produce output artifacts, and finally `openPullRequest` to submit the results.
4. External clients can query pipeline status through the HTTP GET endpoints (`pipelineRuns`, `pipelineRunDetail`, `pipelineSmeQueue`), which use their `durableClient` bindings to query the Durable Functions runtime for orchestration state.
5. Onboarding sessions can be managed via `cancelOnboardingSession` and `publishOnboardingSession`.

**Why this matters for SRE:** This is the most architecturally significant finding in the DEV estate. A single Function App hosts a complete document processing pipeline with 16 tightly coupled functions. Any outage in this app affects the entire SOP Factory workflow.

---

## 10. Application-by-Application Architecture (Layer 5-6)

This section documents each Function App individually, showing its internal structure and external dependencies.

### 10.1 uudri-bill-processor-dev

```
Blob Storage (StorageAccount)
     |
     v
BillProcessor (blobTrigger)
     |
     v
dotnet-isolated runtime
```

- **Functions:** 1 (BillProcessor)
- **Trigger:** blobTrigger
- **Connection:** StorageAccount
- **Runtime:** dotnet-isolated
- **What it does:** Processes utility bills when files appear in blob storage.

### 10.2 helios-ontology-event-processor-func

```
HTTP
 +-- health_check           (GET /health, ANONYMOUS)
 +-- test_relationship_processor   (POST /test_relationship, FUNCTION auth)
 +-- test_schema_processor         (POST /test_schema, FUNCTION auth)

Event Hub (EventHubConnection / %EVENT_HUB_NAME%)
 +-- ontology_event_processor
```

- **Functions:** 4 (1 event-driven, 3 HTTP)
- **Runtime:** Python 3.11
- **What it does:** Processes ontology events from Event Hub. Also exposes health check and test endpoints for schema and relationship processing.
- **Note:** The health check is unauthenticated (ANONYMOUS). The test endpoints require a Function key (FUNCTION auth level).

### 10.3 helios-github-activity-logger-dev-func

```
GitHub webhook
     |
     v
github_webhook (POST /github/webhook, FUNCTION auth)
     |
     v
Python 3.11 runtime
```

- **Functions:** 1 (github_webhook)
- **Trigger:** httpTrigger (POST)
- **Route:** github/webhook
- **What it does:** Receives GitHub webhook payloads and logs activity.

### 10.4 func-projector-sopfactorydevmlel9

```
HTTP
 |
 v
projectOnPublish (POST, FUNCTION auth)
 |
 v
Node.js 20 runtime
```

- **Functions:** 1 (projectOnPublish)
- **Trigger:** httpTrigger (POST)
- **What it does:** Part of the SOP Factory system; handles publish projections via HTTP.

### 10.5 helios-dev-cost-ingestion

```
Timer (0 0 6 * * *)
 |
 v
cost_ingestion
 |
 v
Python 3.11 runtime
```

- **Functions:** 1 (cost_ingestion)
- **Trigger:** timerTrigger
- **Schedule:** `0 0 6 * * *` (daily at 06:00 UTC)
- **What it does:** Ingests cost data on a daily schedule.
- **SRE concern:** This app is missing Application Insights configuration (see Section 12).

### 10.6 func-orchestrator-sopfactorydevmlel9

See [Section 9](#9-durable-functions-architecture----sop-factory-layer-5-6) for the complete architecture. Summary:

- **Functions:** 16
- **Runtime:** Node.js 20
- **Pattern:** Durable Functions (client/orchestrator/activity)
- **Entry points:** HTTP (POST/GET) and Blob (SOURCE_STORAGE)
- **What it does:** Complete SOP Factory document processing pipeline.

### 10.7 helios-device-telemetry-dev-func

```
Event Hub (EVENT_HUB_CONNECTION_STRING / %EVENT_HUB_NAME%)
    |
    v
new_device_and_telemetry_classification_agent
    |
    v
Python 3.11 runtime
```

- **Functions:** 1 (new_device_and_telemetry_classification_agent)
- **Trigger:** eventHubTrigger
- **What it does:** Classifies new devices and telemetry data from Event Hub events.

### 10.8 ems-plan-narration-function

```
Event Hub (EVENT_HUB_CONNECTION_STRING / %EVENT_HUB_NAME%)
    |
    +---> plan_narration_agent

Event Hub (REALIZED_KPI_EVENT_HUB_CONNECTION_STRING / %REALIZED_KPI_EVENT_HUB_NAME%)
    |
    +---> realized_kpi_listener
```

- **Functions:** 2 (plan_narration_agent, realized_kpi_listener)
- **Runtime:** Python 3.12
- **What it does:** Narrates energy management plans and listens for realized KPI events.
- **Note:** The two functions use different Event Hub connections, meaning they consume from different Event Hub instances.
- **CI/CD:** This is the Function App examined in the release-orchestration analysis (see Section 13).

### 10.9 kg-event-processor-dev

```
HTTP
 |
 v
HealthCheck (GET /health, Anonymous auth)

Service Bus
 |
 v
Topic: helios-knowledgegraph-events
 |
 v
Subscription: kg-event-processor
 |
 v
ProcessDMLEvent (SERVICEBUS_CONNECTION_STRING)
```

- **Functions:** 2 (HealthCheck, ProcessDMLEvent)
- **Runtime:** dotnet-isolated
- **What it does:** Processes Knowledge Graph data manipulation events (DML) from Service Bus. Exposes a health check endpoint.

### 10.10 UUDRI-Bill-Processor-dev-01

```
Blob Storage (StorageAccount)
     |
     v
BillProcessor (blobTrigger)
     |
     v
dotnet-isolated runtime
```

- **Functions:** 1 (BillProcessor)
- **Trigger:** blobTrigger
- **Connection:** StorageAccount
- **What it does:** Identical trigger configuration to `uudri-bill-processor-dev`. The relationship between the two Bill Processor apps (parallel, migration, or separate scope) is not yet determined.

---

## 11. SCM and Platform Configuration (Layer 7)

### 11.1 Platform Configuration Table

Queried via `Microsoft.Web/sites/<app>/config/web`:

| Function App | SCM Type | Linux Fx Version | Always On | FTPS | Public Network | Min TLS |
|---|---|---|---|---|---|---|
| uudri-bill-processor-dev | VSTSRM | | TRUE | FtpsOnly | Enabled | 1.2 |
| helios-ontology-event-processor-func | None | PYTHON\|3.11 | FALSE | FtpsOnly | Enabled | 1.2 |
| helios-github-activity-logger-dev-func | None | PYTHON\|3.11 | FALSE | Disabled | Enabled | 1.2 |
| func-projector-sopfactorydevmlel9 | None | NODE\|20 | FALSE | Disabled | Enabled | 1.2 |
| helios-dev-cost-ingestion | GitHubAction | PYTHON\|3.11 | FALSE | Disabled | Enabled | 1.2 |
| func-orchestrator-sopfactorydevmlel9 | None | NODE\|20 | FALSE | Disabled | Enabled | 1.2 |
| helios-device-telemetry-dev-func | None | PYTHON\|3.11 | FALSE | Disabled | Enabled | 1.2 |
| ems-plan-narration-function | None | PYTHON\|3.12 | FALSE | FtpsOnly | Enabled | 1.2 |
| kg-event-processor-dev | None | | FALSE | FtpsOnly | Enabled | 1.2 |
| UUDRI-Bill-Processor-dev-01 | VSTSRM | | TRUE | Disabled | Enabled | 1.2 |

### 11.2 How to Read This Table

- **SCM Type:** How deployments reach the app. `VSTSRM` = Azure DevOps Release Management. `GitHubAction` = GitHub Actions. `None` = no SCM integration registered (may still be deployed via CLI/ZipDeploy).
- **Linux Fx Version:** The runtime stack. Empty for dotnet-isolated apps (they use a different configuration path).
- **Always On:** Whether the app is kept warm. Only the two Bill Processor apps have this enabled. All others will cold-start after periods of inactivity.
- **FTPS:** `FtpsOnly` = FTPS allowed but only over TLS. `Disabled` = no FTP/FTPS access.
- **Public Network:** All apps have public network access `Enabled`. VNet integration has not been verified yet.
- **Min TLS:** All apps enforce TLS 1.2 as the minimum version.

### 11.3 Observed Runtimes

```
PYTHON|3.11  -- used by 5 apps
PYTHON|3.12  -- used by 1 app (ems-plan-narration-function)
NODE|20      -- used by 2 apps
dotnet-isolated -- used by 3 apps (shown via empty LinuxFxVersion)
```

---

## 12. Observability Configuration (Layer 10-11)

### 12.1 Application Insights Configuration

Queried via `Microsoft.Web/sites/<app>/config/appsettings/list`:

| Function App | Resource Group | App Insights Connection String | App Insights Instrumentation Key | Functions Extension Version | Functions Worker Runtime |
|---|---|---|---|---|---|
| uudri-bill-processor-dev | uudri-dev-rg | Present | Missing | ~4 | dotnet-isolated |
| helios-ontology-event-processor-func | helios-dev-us-west3-rg | Present | Present | ~4 | python |
| helios-github-activity-logger-dev-func | helios-dev-us-west3-rg | Present | Present | ~4 | python |
| func-projector-sopfactorydevmlel9 | helios-dev-us-west3-rg | Present | Missing | ~4 | node |
| helios-dev-cost-ingestion | helios-dev-us-west3-rg | **Missing** | **Missing** | ~4 | python |
| func-orchestrator-sopfactorydevmlel9 | helios-dev-us-west3-rg | Present | Missing | ~4 | node |
| helios-device-telemetry-dev-func | helios-dev-us-west3-rg | Present | Missing | ~4 | python |
| ems-plan-narration-function | helios-dev-us-west3-rg | Present | Present | ~4 | python |
| kg-event-processor-dev | helios-dev-us-west3-rg | Present | Present | ~4 | dotnet-isolated |
| UUDRI-Bill-Processor-dev-01 | helios-dev-us-west3-rg | Present | Present | ~4 | dotnet-isolated |

**Key observations:**

- 9 out of 10 apps have `APPLICATIONINSIGHTS_CONNECTION_STRING` configured.
- 5 out of 10 apps also have the legacy `APPINSIGHTS_INSTRUMENTATIONKEY`.
- **`helios-dev-cost-ingestion` has neither setting.** This means the daily cost ingestion job is potentially running without any Application Insights telemetry.
- All apps run Functions Extension Version `~4`.

**Important interpretation:** Missing instrumentation key alone does not mean App Insights is absent. Connection-string-based configuration is the modern approach and is sufficient. However, `helios-dev-cost-ingestion` is missing both settings.

### 12.2 Azure Monitor Diagnostic Settings

Queried via `Microsoft.Insights/diagnosticSettings` for all 10 Function Apps:

| Function App | Diagnostic Settings Found | Diagnostic Setting Count |
|---|---|---|
| uudri-bill-processor-dev | No | 0 |
| helios-ontology-event-processor-func | No | 0 |
| helios-github-activity-logger-dev-func | No | 0 |
| func-projector-sopfactorydevmlel9 | No | 0 |
| helios-dev-cost-ingestion | No | 0 |
| func-orchestrator-sopfactorydevmlel9 | No | 0 |
| helios-device-telemetry-dev-func | No | 0 |
| ems-plan-narration-function | No | 0 |
| kg-event-processor-dev | No | 0 |
| UUDRI-Bill-Processor-dev-01 | No | 0 |

**Result:** 10 out of 10 DEV Function Apps have zero resource-level diagnostic settings.

**Correct SRE interpretation:** No resource-level Azure Monitor diagnostic settings were returned for the queried Function App resources. This means platform logs, audit logs, and scaling logs are not being forwarded to a Log Analytics Workspace, Event Hub, or Storage Account from these resources. However, this does not by itself prove that no telemetry or monitoring exists -- Application Insights telemetry (confirmed as configured for 9 of 10 apps) is a separate data path. We must still trace the actual Application Insights resources, telemetry destinations, alert rules, and Action Groups.

---

## 13. CI/CD and Release Orchestration (Layer 8-9)

### 13.1 Repository Investigated

```
qcells-hqct / helios-plan-narration-backend
```

This repository deploys to the `ems-plan-narration-function` Function App.

### 13.2 GitHub Actions Workflows Found

| Workflow File | Target Environment |
|---|---|
| `.github/workflows/deploy-function-app.yml` | DEV |
| `.github/workflows/deploy-function-app-qa.yml` | QA |
| `.github/workflows/deploy-function-app-prod.yml` | PROD |
| `.github/workflows/test-function-app.yml` | (test only, not linked to deploy) |

### 13.3 DEV Release Flow

```
GitHub push or workflow_dispatch
         |
         v
CI Phase
  pip install
  python -m compileall  (syntax/compile check)
         |
         v
Deploy Phase
  azure/login  (using repository secrets)
  Set app settings  (from helios-dev-backend-kv Key Vault)
  Azure/functions-action  (ZipDeploy)
         |
         v
Target: ems-plan-narration-function
         |
         v
Post-deploy validation
  Check app state
  Verify function registration
```

### 13.4 GitHub Repository Secrets (Names Only)

| Secret Name |
|---|
| AZURE_CLIENT_ID |
| AZURE_CLIENT_SECRET |
| AZURE_SUBSCRIPTION_ID |
| AZURE_TENANT_ID |

**Note:** Secret values are never recorded. Only the presence of the configuration is documented.

### 13.5 GitHub Repository Variables

| Variable Name |
|---|
| APIM_CLIENT_ID |
| APIM_SCOPE |
| APIM_TOKEN_URL |
| AZURE_AI_FOUNDRY_ENDPOINT |
| EMS_PLAN_NARRATION_AGENT_ID |
| EVENTHUB_CONSUMER_GROUP |
| EVENT_HUB_NAME |
| OBJECT_STORE_SERVICE_URL |

### 13.6 SRE Observations on the Release Pipeline

1. **The deploy workflow does not run pytest.** The `test-function-app.yml` workflow exists but is not linked through `needs:` or `workflow_call`. CI is therefore primarily a syntax/compile gate.
2. **DEV/QA/PROD are separate workflows.** There is no demonstrated build-once/immutable-artifact promotion path. Each environment has its own workflow file, which means the same code may be built differently in each environment.
3. **PROD is manually dispatched** in the examined workflow. The exact approval behavior depends on GitHub Environment protection rules, which must be verified separately.
4. **Post-deployment validation exists** (state check and function registration verification), but there is **no automatic rollback** if the validation fails.
5. **Key Vault reference:** The DEV workflow references `helios-dev-backend-kv` for app settings.

### 13.7 Deployment History Evidence

Deployment records were queried via `Microsoft.Web/sites/<app>/deployments`. The following apps had deployment records returned by the ARM API:

**uudri-bill-processor-dev** -- 10 deployments recorded

| Deployer | Most Recent | Method |
|---|---|---|
| ZipDeploy | 2026-05-04 | Push deployment |
| ZipDeploy | 2026-04-27 | Push deployment |
| ZipDeploy | 2026-04-23 | Push deployment |
| ZipDeploy | 2026-04-22 (4 deployments) | Push deployment |
| ZipDeploy | 2026-04-21 | Push deployment |

**helios-github-activity-logger-dev-func** -- 6 deployments recorded

| Deployer | Most Recent | Method |
|---|---|---|
| az_cli_functions | 2026-04-13 (2 deployments) | Push deployment |
| az_cli_functions | 2026-03-31 | Push deployment |
| az_cli_functions | 2026-03-25 (2 deployments) | Push deployment |
| az_cli_functions | 2026-03-24 | Push deployment |

**ems-plan-narration-function** -- 10 deployments recorded

| Deployer | Most Recent | Method | Repo |
|---|---|---|---|
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-08-13 | GitHub Actions | qcells-hqct/helios-plan-narration-backend |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-08-07 | GitHub Actions | qcells-hqct/helios-plan-narration-backend |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-08-06 | GitHub Actions | qcells-hqct/helios-plan-narration-backend |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-08-05 (3 deployments) | GitHub Actions | qcells-hqct/helios-plan-narration-backend |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-07-30 (3 deployments) | GitHub Actions | qcells-hqct/helios-plan-narration-backend |
| GITHUB_ZIP_DEPLOY_FUNCTIONS_V1 | 2026-07-29 | GitHub Actions | qcells-hqct/helios-plan-narration-backend |

**kg-event-processor-dev** -- 10 deployments recorded

| Deployer | Most Recent | Method |
|---|---|---|
| az_cli_functions | 2026-08-18 | Push deployment |
| az_cli_functions | 2026-08-08 | Push deployment |
| az_cli_functions | 2026-08-07 | Push deployment |
| az_cli_functions | 2026-06-22 | Push deployment |
| az_cli_functions | 2026-06-19 | Push deployment |
| az_cli_functions | 2026-06-16 (3 deployments) | Push deployment |
| az_cli_functions | 2026-06-15 | Push deployment |
| az_cli_functions | 2026-06-12 | Push deployment |

**UUDRI-Bill-Processor-dev-01** -- 3 deployments recorded

| Deployer | Most Recent | Method | Repo |
|---|---|---|---|
| VSTS_FUNCTIONS_V1 | 2026-07-03 | Azure DevOps Release | qcells-hqct/UUDRI-Backend (Release-19, tag-v2.3.2) |
| ZipDeploy | 2026-06-05 | Push deployment | |
| VSTS_FUNCTIONS_V1 | 2026-05-22 | Azure DevOps Release | qcells-hqct/UUDRI-Backend (Release-18, branch: develop) |

**Apps with no deployment records returned:** `helios-ontology-event-processor-func`, `func-projector-sopfactorydevmlel9`, `helios-dev-cost-ingestion`, `func-orchestrator-sopfactorydevmlel9`, `helios-device-telemetry-dev-func`. An empty deployment list means only that the queried ARM API returned no records; it is not proof that an app was never deployed.

---

## 14. Logic App / Workflow App Discovery

Three Logic Apps (workflow apps) were identified in the same resource group:

| Resource Name | Resource Type | Kind | Location | Resource Group |
|---|---|---|---|---|
| helios-data-eventstream-monitor-dev | Microsoft.Web/sites | functionapp,workflowapp | westus3 | helios-dev-us-west3-rg |
| helios-data-market-eh-eventstream-monitor-dev | Microsoft.Web/sites | functionapp,workflowapp | westus3 | helios-dev-us-west3-rg |
| helios-data-weather-eh-eventstream-monitor-dev | Microsoft.Web/sites | functionapp,workflowapp | westus3 | helios-dev-us-west3-rg |

### 14.1 Workflow Actions Discovered

```
Get_Config
For_each_Stream
Query_Eventhouse
Query_IncomingMessages
Compute_Delta
Compute_EventDeliveryRatio
Condition_Non_Ok
Emit_HealthCheck
```

**Important:** The HTTP URI values within these workflows were found to be workflow expressions rather than simple literal URLs. They should not be treated as static dependency hosts without resolving the expression/config source.

---

## 15. Evidence Files

All evidence is stored under:

```
C:\AzureFunctionInventory\Day3\Investigation
```

| CSV File | What It Proves |
|---|---|
| DEV-Function-Inventory.csv | Which Functions exist inside each Function App |
| DEV-Binding-Inventory.csv | How each Function is triggered and bound to external services |
| DEV-Event-Dependency-Inventory.csv | Event Hub, Service Bus, Blob, and Timer dependencies exposed by bindings |
| DEV-SCM-Configuration.csv | Runtime, SCM type, and platform configuration per Function App |
| DEV-AppInsights-Configuration.csv | Presence or absence of Application Insights settings and Functions runtime settings |
| DEV-Diagnostic-Summary.csv | Whether resource-level Azure Monitor diagnostic settings were found |

These CSVs are not merely attachments. They are the reproducible evidence behind the architecture. Every table and diagram in this document can be traced back to the data in these files.

---

## 16. Current DEV Architecture Diagram

This is an evidence-based current-state model showing how all discovered components connect:

```
                               GitHub Repositories
                    +--------------------------------------+
                    | qcells-hqct/helios-plan-narration-   |
                    |   backend                            |
                    | qcells-hqct/UUDRI-Backend            |
                    +------------------+-------------------+
                                       |
                     GitHub Actions / Azure DevOps Releases
                                       |
                                       v
                    +--------------------------------------+
                    |    Azure Function Apps -- DEV         |
                    |          10 Apps / 30 Functions       |
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
            |           telemetry_agent (KG events)             |
            |           plan_narration                          |
            |           realized_kpi                            |
            |                                                   |
            +---------------------------------------------------+
                                       |
                                       v
                           Application Dependencies
                           (Storage, AI Foundry, APIM,
                            Object Store, Key Vault)
                                       |
                                       v
                          Observability Layer
                    +------------------+-------------------+
                    |                                      |
              App Insights                          Azure Monitor
           (9/10 apps configured)            (0/10 diagnostic settings)
```

**This is a current-state discovery diagram, not a target architecture.** Identity, network paths, alerting, and actual telemetry destinations still need to be mapped.

---

## 17. Capability Matrix Update

Based on our DEV findings, the Azure Functions row in the Release Orchestration Capability Matrix can now be updated:

| Capability | Previous | Current DEV Status | Evidence |
|---|---|---|---|
| Deploy on merge | unknown | **Partially mapped** | GitHub Actions workflow found for `ems-plan-narration-function`; Azure DevOps VSTS releases for UUDRI; az_cli_functions deployments for others. Not all repos examined. |
| Promotion gates (dev -> QA -> prod) | unknown | **Partially mapped** | Separate workflow files found for DEV/QA/PROD. No immutable artifact promotion. PROD is manually dispatched. |
| Scheduled verification | not started | **Not started** | No automated post-deployment integration tests or recurring verifiers were found. |
| Alerting | not started | **Not started** | No Alert Rules or Action Groups have been mapped yet (next phase). |
| Observability + cost | not started | **Partially mapped** | 9/10 apps have App Insights connection string. 1/10 is missing. 0/10 have resource-level diagnostic settings. |
| Reporting / audit | not started | **Baseline established** | Deployment history, configuration, and binding data captured in CSV evidence files. |

### 17.1 Detailed SRE Capability Position

| Area | Status |
|---|---|
| Resource inventory | Mapped |
| Function inventory | Mapped |
| Trigger/bindings | Mapped |
| Event dependencies | Partially mapped (connection names recorded; actual Event Hub/Service Bus resource IDs not resolved) |
| Durable architecture | Mapped for `func-orchestrator-sopfactorydevmlel9` |
| Platform configuration | Partially mapped |
| CI/CD | Mapped for `helios-plan-narration-backend` repository |
| Observability config | Partially mapped |
| Alerting | **Not yet mapped** |
| Scheduled verification | **Not yet mapped** |
| Reporting/audit | **Not yet mapped** |
| Identity/RBAC | **Not yet mapped** |
| Networking | **Not yet mapped** |
| SLO/SLI | **Not yet mapped** |

---

## 18. What Has Been Verified vs What Remains Unknown

### 18.1 Verified or Partially Verified

- 10 DEV Function Apps and their resource groups
- 30 registered Functions and their names
- Languages/runtimes for every Function
- Trigger types and binding configurations
- Event Hub dependencies (4 Functions, connection names and hub name placeholders)
- Service Bus dependency (1 Function, topic and subscription identified)
- Blob-triggered workloads (3 Functions, connection names identified)
- Timer schedule (1 Function, cron expression captured)
- Durable Function patterns (orchestrator, activities, clients mapped)
- SCM/web platform configuration (SCM type, runtime version, AlwaysOn, FTPS, TLS, public network)
- Deployment history where exposed by the ARM API
- DEV CI/CD workflow for helios-plan-narration-backend
- Repository secret and variable names (not values)
- App Insights-related app settings (connection string and instrumentation key presence)
- Resource-level diagnostic settings (confirmed absent for all 10 apps)
- Three event-stream monitoring Logic Apps

### 18.2 Not Yet Fully Verified

- Managed identity assignments
- RBAC role assignments
- Key Vault access paths and policies
- Storage account permissions
- Event Hub permissions and consumer groups
- Service Bus permissions
- Network/VNet integration
- Private endpoints
- Actual Application Insights resource IDs and telemetry destinations
- Alert rules
- Action Groups
- SLO/SLI definitions
- Runtime failure rate, latency, and throttling metrics
- Retry and dead-letter behavior
- Event Hub lag and consumer group offsets
- Service Bus backlog
- Ownership and on-call assignment
- RTO/RPO definitions
- Incident escalation paths
- Full QA/PROD comparison

---

## 19. Next Discovery Phase

No remediation should be implemented until the discovery is complete. The next investigation sequence is:

```
Phase 2 Discovery Sequence:

 1. Managed Identity
         |
         v
 2. RBAC role assignments
         |
         v
 3. Key Vault access paths
         |
         v
 4. Storage / Event Hub / Service Bus permissions
         |
         v
 5. VNet / network integration / private endpoints
         |
         v
 6. Actual Application Insights resource mapping
         |
         v
 7. Alert rules + Action Groups
         |
         v
 8. Runtime metrics (failure rate, latency, throttling)
         |
         v
 9. Event Hub / Service Bus health (lag, backlog, dead-letter)
         |
         v
10. Ownership + SLO/SLI + incident path
         |
         v
11. Final DEV architecture
```

Once the DEV model is complete, the same investigation methodology will be applied to **QA** and **PROD** for cross-environment comparison and promotion-path analysis.

---

## 20. Appendix A: Investigation Commands

This appendix documents the exact PowerShell commands used to produce the evidence CSVs. These commands can be re-run to reproduce or update the inventory.

### A.1 Function App List

```powershell
$apps = az functionapp list `
  --query "[?contains(resourceGroup,'helios-dev') || contains(resourceGroup,'uudri-dev')].{Name:name,RG:resourceGroup}" `
  -o json |
  ConvertFrom-Json

$apps | Format-Table Name,RG -AutoSize
```

### A.2 Individual Function Inventory

```powershell
$functionInventory = foreach ($app in $apps) {
    Write-Host "Checking: $($app.Name)" -ForegroundColor Cyan
    $url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/functions?api-version=2022-03-01"
    $json = az rest --method get --url $url -o json
    if ($LASTEXITCODE -ne 0) {
        Write-Host "ARM query failed: $($app.Name)" -ForegroundColor Red
        continue
    }
    $response = $json | ConvertFrom-Json
    foreach ($fn in @($response.value)) {
        [PSCustomObject]@{
            Environment   = "DEV"
            FunctionApp   = $app.Name
            ResourceGroup = $app.RG
            FunctionName  = $fn.properties.name
            Type          = $fn.type
            Language      = $fn.properties.language
            Disabled      = $fn.properties.isDisabled
        }
    }
}
$functionInventory |
    Export-Csv "C:\AzureFunctionInventory\Day3\Investigation\DEV-Function-Inventory.csv" -NoTypeInformation
```

### A.3 Binding Inventory

```powershell
$bindingInventory = foreach ($app in $apps) {
    Write-Host "Reading bindings: $($app.Name)" -ForegroundColor Cyan
    $url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/functions?api-version=2022-03-01"
    $response = az rest --method get --url $url -o json | ConvertFrom-Json
    foreach ($fn in @($response.value)) {
        foreach ($binding in @($fn.properties.config.bindings)) {
            if ($binding.direction -eq "IN") {
                [PSCustomObject]@{
                    FunctionApp       = $app.Name
                    FunctionName      = $fn.properties.name
                    Language          = $fn.properties.language
                    TriggerType       = $binding.type
                    Direction         = $binding.direction
                    AuthLevel         = $binding.authLevel
                    Methods           = ($binding.methods -join ",")
                    Route             = $binding.route
                    Connection        = $binding.connection
                    EventHubName      = $binding.eventHubName
                    QueueName         = $binding.queueName
                    TopicName         = $binding.topicName
                    SubscriptionName  = $binding.subscriptionName
                    BlobPath          = $binding.blobPath
                    Schedule          = $binding.schedule
                    Disabled          = $fn.properties.isDisabled
                }
            }
        }
    }
}
$bindingInventory |
    Export-Csv "C:\AzureFunctionInventory\Day3\Investigation\DEV-Binding-Inventory.csv" -NoTypeInformation
```

### A.4 Event Dependency Inventory

```powershell
$bindingInventory |
    Where-Object {
        $_.TriggerType -in @("eventHubTrigger","blobTrigger","serviceBusTrigger","timerTrigger")
    } |
    Select-Object FunctionApp,FunctionName,TriggerType,Connection,
                  EventHubName,QueueName,TopicName,SubscriptionName,BlobPath,Schedule |
    Export-Csv "C:\AzureFunctionInventory\Day3\Investigation\DEV-Event-Dependency-Inventory.csv" -NoTypeInformation
```

### A.5 SCM Configuration

```powershell
$scmInventory = foreach ($app in $apps) {
    $url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/config/web?api-version=2022-03-01"
    $obj = az rest --method get --url $url -o json | ConvertFrom-Json
    [PSCustomObject]@{
        Environment   = "DEV"
        FunctionApp   = $app.Name
        ResourceGroup = $app.RG
        SCMType       = $obj.properties.scmType
        LinuxFxVersion = $obj.properties.linuxFxVersion
        AlwaysOn      = $obj.properties.alwaysOn
        FTPS          = $obj.properties.ftpsState
        PublicNetwork = $obj.properties.publicNetworkAccess
        MinTLS        = $obj.properties.minTlsVersion
    }
}
$scmInventory |
    Export-Csv "C:\AzureFunctionInventory\Day3\Investigation\DEV-SCM-Configuration.csv" -NoTypeInformation
```

### A.6 Application Insights Configuration

```powershell
$aiInventory = foreach ($app in $apps) {
    Write-Host "Checking App Insights: $($app.Name)" -ForegroundColor Cyan
    $url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/config/appsettings/list?api-version=2022-03-01"
    $json = az rest --method post --url $url -o json
    if ($LASTEXITCODE -ne 0) {
        Write-Host "FAILED: $($app.Name)" -ForegroundColor Red
        continue
    }
    $obj = $json | ConvertFrom-Json
    $settings = @{}
    foreach ($item in @($obj.properties.PSObject.Properties)) {
        $settings[$item.Name] = $item.Value
    }
    [PSCustomObject]@{
        Environment                    = "DEV"
        FunctionApp                    = $app.Name
        ResourceGroup                  = $app.RG
        AppInsightsConnection          = if ($settings.ContainsKey("APPLICATIONINSIGHTS_CONNECTION_STRING")) { "Present" } else { "Missing" }
        AppInsightsInstrumentationKey   = if ($settings.ContainsKey("APPINSIGHTS_INSTRUMENTATIONKEY")) { "Present" } else { "Missing" }
        FunctionsExtensionVersion      = if ($settings.ContainsKey("FUNCTIONS_EXTENSION_VERSION")) { $settings["FUNCTIONS_EXTENSION_VERSION"] } else { "" }
        FunctionsWorkerRuntime         = if ($settings.ContainsKey("FUNCTIONS_WORKER_RUNTIME")) { $settings["FUNCTIONS_WORKER_RUNTIME"] } else { "" }
    }
}
$aiInventory |
    Export-Csv "C:\AzureFunctionInventory\Day3\Investigation\DEV-AppInsights-Configuration.csv" -NoTypeInformation
```

### A.7 Diagnostic Settings Summary

```powershell
$diagnosticSummary = foreach ($app in $apps) {
    $found = @(
        $diagnosticInventory |
        Where-Object { $_.FunctionApp -eq $app.Name }
    )
    [PSCustomObject]@{
        Environment            = "DEV"
        FunctionApp            = $app.Name
        DiagnosticSettingsFound = if ($found.Count -gt 0) { "Yes" } else { "No" }
        DiagnosticSettingCount  = $found.Count
    }
}
$diagnosticSummary |
    Export-Csv "C:\AzureFunctionInventory\Day3\Investigation\DEV-Diagnostic-Summary.csv" -NoTypeInformation
```

### A.8 Deployment History

```powershell
$app = $apps | Where-Object { $_.Name -eq "helios-github-activity-logger-dev-func" }
$url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/deployments?api-version=2022-03-01"
az rest --method get --url $url -o json
```

---

## 21. Appendix B: Glossary

| Term | Definition |
|---|---|
| Function App | An Azure resource that hosts one or more Functions. Similar to a microservice deployment unit. |
| Function | A single unit of code that responds to an event (trigger). Lives inside a Function App. |
| Trigger | The event that causes a Function to execute. Each Function has exactly one trigger. |
| Binding | A declarative connection between a Function and an external resource (Event Hub, Blob, etc.). |
| Durable Functions | An Azure Functions extension for writing stateful, long-running workflows using orchestrator/activity patterns. |
| Event Hub | An Azure streaming platform for high-throughput event ingestion. |
| Service Bus | An Azure messaging service supporting topics and subscriptions for pub/sub patterns. |
| Blob Storage | Azure's object storage for unstructured data (files, documents). |
| ARM API | Azure Resource Manager API -- the management plane used to query resource configuration. |
| Application Insights | Azure's application performance monitoring (APM) service. |
| Diagnostic Settings | Azure Monitor configuration that routes platform logs to Log Analytics, Event Hub, or Storage. |
| SCM | Source Control Manager -- how deployment code reaches the Function App. |
| ZipDeploy | A deployment method where a zip archive of the app is pushed to Azure. |
| VSTSRM | Visual Studio Team Services Release Management (Azure DevOps Releases). |

---

## 22. Closing SRE Statement

> "The DEV Azure Functions estate is now partially mapped at resource, workload, trigger, dependency, platform, deployment, CI/CD, and observability-configuration levels. We have evidence for 10 Function Apps and 30 registered Functions in the current DEV scope. The next phase is to complete the operational view by mapping identity, RBAC, networking, actual telemetry destinations, alerting, runtime health, ownership and SLOs. Once DEV is coherent, the same model can be applied to QA and PROD for comparison and promotion-path analysis."

> "The DEV Function estate is not one homogeneous workload. It contains HTTP APIs, webhooks, Event Hub consumers, Blob-triggered processors, a scheduled ingestion function, Service Bus consumers, and a Durable Functions orchestration system. We have mapped these workloads down to the individual Function level and preserved the evidence in CSV inventories. The next step is to connect this workload map to identity, permissions, network paths, observability and operational ownership so that we can produce a complete SRE view rather than only an Azure resource inventory."

**No remediation or implementation changes have been performed during this discovery phase.**

---

*Document authored by SRE Team | Evidence collected August 2026 | DEV Environment Only*
