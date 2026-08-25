# SRE DEV Architecture --- 30 Function Inventory

## Purpose

This document is the **DEV current-state architecture evidence pack**
for the Monday SRE team meeting.

We are not implementing changes yet. The purpose is to demonstrate:

-   what exists,
-   how we discovered it,
-   how the 30 Functions are grouped into 10 Function Apps,
-   how each Function is triggered,
-   which external dependency is visible from the binding,
-   how deployment works,
-   what observability configuration was found,
-   what remains unknown,
-   and what the next SRE discovery phase will be.

## 1. Current DEV Architecture at a Glance

``` text
                              GitHub Repository
                         helios-plan-narration-backend
                                      |
                                      | GitHub Actions
                                      v
                         +---------------------------+
                         | Release / CI-CD workflows |
                         +-------------+-------------+
                                       |
                                       v
                         +---------------------------+
                         | Azure Function Apps - DEV |
                         |        10 Apps             |
                         +-------------+-------------+
                                       |
             +-------------------------+-------------------------+
             |                         |                         |
             v                         v                         v
        HTTP-triggered          Event-driven             Durable / Timer
          Functions              Functions                 Functions
             |                         |                         |
             v                         v                         v
          APIs/Webhooks          Event Hub              Durable runtime
                                Service Bus                    |
                                  Blob                      Orchestrator
             |                         |                         |
             +-------------------------+-------------------------+
                                       |
                                       v
                              Application dependencies
                                       |
                                       v
                             Observability layer
                         +-------------+-------------+
                         |                           |
                    App Insights              Azure Monitor
                 app-setting evidence        diagnostic settings
```

### Important interpretation

This is an **evidence-based current-state model**, not a target
architecture.

We have not yet completed:

``` text
Managed Identity → RBAC → Key Vault access
             ↓
       Network / VNet
             ↓
     Alerts / Action Groups
             ↓
 Runtime health / SLI / SLO / ownership
```

## 2. DEV Scope

Current working scope:

-   **10 Function Apps**
-   **30 registered Functions**
-   Multiple trigger/binding types
-   Event Hub, Service Bus, Blob and Timer dependencies
-   Durable Functions patterns
-   GitHub Actions release orchestration for the examined repository

The broader project identified approximately **27 Function Apps across
DEV/QA/PROD**. The capability matrix separately refers to approximately
87 Azure Functions across three environments. These broader figures are
not yet reconciled and should remain an open discovery item.

## 3. Complete 30-Function DEV Map

  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
              \# Function App                               Function                                          Runtime             Trigger / Binding               Dependency / Evidence
  -------------- ------------------------------------------ ------------------------------------------------- ------------------- ------------------------------- ------------------------------------------
               1 `uudri-bill-processor-dev`                 `BillProcessor`                                   `dotnet-isolated`   `blobTrigger`                   StorageAccount --- Blob dependency
                                                                                                                                                                  discovered; exact blob path not exposed in
                                                                                                                                                                  current inventory

               2 `helios-ontology-event-processor-func`     `health_check`                                    `python`            `httpTrigger`                   --- HTTP endpoint

               3 `helios-ontology-event-processor-func`     `ontology_event_processor`                        `python`            `eventHubTrigger`               EventHubConnection / %EVENT_HUB_NAME% ---
                                                                                                                                                                  Event Hub

               4 `helios-ontology-event-processor-func`     `test_relationship_processor`                     `python`            `httpTrigger`                   --- HTTP endpoint

               5 `helios-ontology-event-processor-func`     `test_schema_processor`                           `python`            `httpTrigger`                   --- HTTP endpoint

               6 `helios-github-activity-logger-dev-func`   `github_webhook`                                  `python`            `httpTrigger`                   --- GitHub webhook endpoint

               7 `func-projector-sopfactorydevmlel9`        `projectOnPublish`                                `node`              `httpTrigger`                   --- HTTP endpoint

               8 `helios-dev-cost-ingestion`                `cost_ingestion`                                  `python`            `timerTrigger`                  0 0 6 \* \* \* --- Runs on timer; schedule
                                                                                                                                                                  discovered

               9 `func-orchestrator-sopfactorydevmlel9`     `cancelOnboardingSession`                         `node`              `httpTrigger`                   --- HTTP endpoint

              10 `func-orchestrator-sopfactorydevmlel9`     `classifySection`                                 `node`              `activityTrigger`               --- Durable activity

              11 `func-orchestrator-sopfactorydevmlel9`     `generateFddArtifact`                             `node`              `activityTrigger`               --- Durable activity

              12 `func-orchestrator-sopfactorydevmlel9`     `generateOntologyArtifact`                        `node`              `activityTrigger`               --- Durable activity

              13 `func-orchestrator-sopfactorydevmlel9`     `generatePolicyArtifact`                          `node`              `activityTrigger`               --- Durable activity

              14 `func-orchestrator-sopfactorydevmlel9`     `generateSopArtifact`                             `node`              `activityTrigger`               --- Durable activity

              15 `func-orchestrator-sopfactorydevmlel9`     `ingestDocument`                                  `node`              `activityTrigger`               --- Durable activity

              16 `func-orchestrator-sopfactorydevmlel9`     `intakeDocument`                                  `node`              `httpTrigger + durableClient`   --- HTTP + Durable client binding

              17 `func-orchestrator-sopfactorydevmlel9`     `onSourceDocument`                                `node`              `blobTrigger + durableClient`   SOURCE_STORAGE --- Blob + Durable client
                                                                                                                                                                  binding

              18 `func-orchestrator-sopfactorydevmlel9`     `openPullRequest`                                 `node`              `activityTrigger`               --- Durable activity

              19 `func-orchestrator-sopfactorydevmlel9`     `pipelineRunDetail`                               `node`              `httpTrigger + durableClient`   --- HTTP + Durable client binding

              20 `func-orchestrator-sopfactorydevmlel9`     `pipelineRuns`                                    `node`              `httpTrigger + durableClient`   --- HTTP + Durable client binding

              21 `func-orchestrator-sopfactorydevmlel9`     `pipelineSmeQueue`                                `node`              `httpTrigger + durableClient`   --- HTTP + Durable client binding

              22 `func-orchestrator-sopfactorydevmlel9`     `publishOnboardingSession`                        `node`              `httpTrigger`                   --- HTTP endpoint

              23 `func-orchestrator-sopfactorydevmlel9`     `screenSourceText`                                `node`              `activityTrigger`               --- Durable activity

              24 `func-orchestrator-sopfactorydevmlel9`     `sopFactoryOrchestrator`                          `node`              `orchestrationTrigger`          --- Durable orchestrator

              25 `helios-device-telemetry-dev-func`         `new_device_and_telemetry_classification_agent`   `python`            `eventHubTrigger`               EVENT_HUB_CONNECTION_STRING /
                                                                                                                                                                  %EVENT_HUB_NAME% --- Event Hub

              26 `ems-plan-narration-function`              `plan_narration_agent`                            `python`            `eventHubTrigger`               EVENT_HUB_CONNECTION_STRING /
                                                                                                                                                                  %EVENT_HUB_NAME% --- Event Hub

              27 `ems-plan-narration-function`              `realized_kpi_listener`                           `python`            `eventHubTrigger`               REALIZED_KPI_EVENT_HUB_CONNECTION_STRING /
                                                                                                                                                                  %REALIZED_KPI_EVENT_HUB_NAME% --- Event
                                                                                                                                                                  Hub

              28 `kg-event-processor-dev`                   `HealthCheck`                                     `dotnet-isolated`   `httpTrigger`                   --- HTTP endpoint

              29 `kg-event-processor-dev`                   `ProcessDMLEvent`                                 `dotnet-isolated`   `serviceBusTrigger`             SERVICEBUS_CONNECTION_STRING --- Service
                                                                                                                                                                  Bus topic helios-knowledgegraph-events /
                                                                                                                                                                  subscription kg-event-processor

              30 `UUDRI-Bill-Processor-dev-01`              `BillProcessor`                                   `dotnet-isolated`   `blobTrigger`                   StorageAccount --- Blob dependency
                                                                                                                                                                  discovered; exact blob path not exposed in
                                                                                                                                                                  current inventory
  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 4. Function App → Function Count

  Function App                                 Functions
  ------------------------------------------ -----------
  `uudri-bill-processor-dev`                           1
  `helios-ontology-event-processor-func`               4
  `helios-github-activity-logger-dev-func`             1
  `func-projector-sopfactorydevmlel9`                  1
  `helios-dev-cost-ingestion`                          1
  `func-orchestrator-sopfactorydevmlel9`              16
  `helios-device-telemetry-dev-func`                   1
  `ems-plan-narration-function`                        2
  `kg-event-processor-dev`                             2
  `UUDRI-Bill-Processor-dev-01`                        1
  **TOTAL**                                       **30**

## 5. Runtime / Trigger Architecture

The current DEV Function map contains:

``` text
dotnet-isolated
    ├── BillProcessor
    ├── HealthCheck
    └── ProcessDMLEvent

python
    ├── health_check
    ├── ontology_event_processor
    ├── test_relationship_processor
    ├── test_schema_processor
    ├── github_webhook
    ├── cost_ingestion
    ├── new_device_and_telemetry_classification_agent
    ├── plan_narration_agent
    └── realized_kpi_listener

node
    └── SOP Factory / orchestration functions
```

The primary trigger families represented in the 30-function map are:

``` text
HTTP
Event Hub
Blob
Service Bus
Timer
Durable activity
Durable client
Durable orchestration
```

### Important binding detail

Some Functions have more than one inbound binding. For example:

``` text
intakeDocument
    → httpTrigger
    → durableClient

onSourceDocument
    → blobTrigger
    → durableClient

pipelineRunDetail
    → httpTrigger
    → durableClient

pipelineRuns
    → httpTrigger
    → durableClient

pipelineSmeQueue
    → httpTrigger
    → durableClient
```

Therefore:

> 30 Functions does not mean 30 binding rows.

A Function can produce multiple binding records.

## 6. Major Event/Data Flows

### Event Hub flows

``` text
Event Hub
   |
   +--> ontology_event_processor
   |       App: helios-ontology-event-processor-func
   |
   +--> new_device_and_telemetry_classification_agent
   |       App: helios-device-telemetry-dev-func
   |
   +--> plan_narration_agent
   |       App: ems-plan-narration-function
   |
   +--> realized_kpi_listener
           App: ems-plan-narration-function
           Dependency:
           REALIZED_KPI_EVENT_HUB_CONNECTION_STRING
```

The binding inventory exposes placeholders such as:

``` text
%EVENT_HUB_NAME%
%REALIZED_KPI_EVENT_HUB_NAME%
```

This is intentionally recorded as configuration evidence rather than
replaced with an assumed resource name.

### Service Bus flow

``` text
Service Bus
    |
    v
Topic: helios-knowledgegraph-events
    |
    v
Subscription: kg-event-processor
    |
    v
ProcessDMLEvent
    |
    v
kg-event-processor-dev
```

Connection setting:

``` text
SERVICEBUS_CONNECTION_STRING
```

### Blob flows

``` text
Storage
   |
   +--> BillProcessor
   |      uudri-bill-processor-dev
   |
   +--> onSourceDocument
   |      func-orchestrator-sopfactorydevmlel9
   |      connection: SOURCE_STORAGE
   |
   +--> BillProcessor
          UUDRI-Bill-Processor-dev-01
```

The exact blob path was not exposed in the current inventory output, so
it should not be invented.

### Timer flow

``` text
Timer
  |
  v
cost_ingestion
  |
  v
helios-dev-cost-ingestion

Schedule:
0 0 6 * * *
```

## 7. Durable Functions Architecture

The SOP Factory Function App is a major orchestration component:

``` text
                 HTTP/API entry points
                         |
       +-----------------+-----------------+
       |                 |                 |
       v                 v                 v
 intakeDocument   pipelineRuns      pipelineRunDetail
       |                 |                 |
       +-----------------+-----------------+
                         |
                         v
              Durable client/orchestration
                         |
                         v
              sopFactoryOrchestrator
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
 classifySection   ingestDocument   screenSourceText
        |                |                |
        +----------------+----------------+
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
 generateFdd       generateOntology   generatePolicy
 generateSop       openPullRequest    publishOnboardingSession
```

Also:

``` text
Blob
 |
 v
onSourceDocument
 |
 +--> durableClient
 |
 v
SOP Factory orchestration
```

This is one of the most important DEV application relationships
discovered so far because a single Function App contains multiple
orchestration/activity roles.

## 8. Application-by-Application Architecture

### 8.1 uudri-bill-processor-dev

``` text
Blob Storage
     |
     v
BillProcessor
     |
     v
dotnet-isolated
```

Trigger:

``` text
blobTrigger
```

Connection setting:

``` text
StorageAccount
```

### 8.2 helios-ontology-event-processor-func

``` text
HTTP
 ├── health_check
 ├── test_relationship_processor
 └── test_schema_processor

Event Hub
 └── ontology_event_processor
```

Runtime:

``` text
Python
```

### 8.3 helios-github-activity-logger-dev-func

``` text
GitHub webhook / HTTP
          |
          v
   github_webhook
          |
          v
        Python
```

### 8.4 func-projector-sopfactorydevmlel9

``` text
HTTP
 |
 v
projectOnPublish
 |
 v
Node.js
```

### 8.5 helios-dev-cost-ingestion

``` text
Timer
 |
 | 0 0 6 * * *
 v
cost_ingestion
 |
 v
Python
```

### 8.6 func-orchestrator-sopfactorydevmlel9

``` text
HTTP / Blob
     |
     v
Durable Client
     |
     v
sopFactoryOrchestrator
     |
     +--> activity functions
     |      ├── classifySection
     |      ├── generateFddArtifact
     |      ├── generateOntologyArtifact
     |      ├── generatePolicyArtifact
     |      ├── generateSopArtifact
     |      ├── ingestDocument
     |      ├── openPullRequest
     |      └── screenSourceText
     |
     +--> HTTP/API functions
     |      ├── cancelOnboardingSession
     |      ├── intakeDocument
     |      ├── pipelineRunDetail
     |      ├── pipelineRuns
     |      ├── pipelineSmeQueue
     |      └── publishOnboardingSession
     |
     └--> Blob entry
            └── onSourceDocument
```

Runtime:

``` text
Node.js
```

### 8.7 helios-device-telemetry-dev-func

``` text
Event Hub
    |
    v
new_device_and_telemetry_classification_agent
    |
    v
Python
```

### 8.8 ems-plan-narration-function

``` text
Event Hub
    |
    +--> plan_narration_agent
    |
    +--> realized_kpi_listener
```

Runtime:

``` text
Python 3.12
```

This Function App is also the target of the DEV GitHub Actions workflow
examined in the release-orchestration analysis.

### 8.9 kg-event-processor-dev

``` text
HTTP
 |
 v
HealthCheck

Service Bus
 |
 v
Topic: helios-knowledgegraph-events
 |
 v
Subscription: kg-event-processor
 |
 v
ProcessDMLEvent
```

Runtime:

``` text
dotnet-isolated
```

### 8.10 UUDRI-Bill-Processor-dev-01

``` text
Blob Storage
     |
     v
BillProcessor
     |
     v
dotnet-isolated
```

Connection:

``` text
StorageAccount
```

## 9. Evidence Commands Used

### Function Apps

``` powershell
$apps | Format-Table Name,RG -AutoSize
```

### Functions

``` powershell
$functionInventory | Format-Table -AutoSize
```

### Count

``` powershell
$functionInventory |
    Where-Object { $_.FunctionName -ne "" } |
    Measure-Object
```

### Trigger inventory

``` powershell
$bindingInventory |
    Select-Object FunctionApp,FunctionName,TriggerType,Connection,
                  EventHubName,QueueName,TopicName,SubscriptionName,
                  BlobPath,Schedule |
    Format-List
```

### Event dependency CSV

``` powershell
$bindingInventory |
    Where-Object {
        $_.TriggerType -in @(
            "eventHubTrigger",
            "blobTrigger",
            "serviceBusTrigger",
            "timerTrigger"
        )
    } |
    Select-Object FunctionApp,FunctionName,TriggerType,Connection,
                  EventHubName,QueueName,TopicName,SubscriptionName,
                  BlobPath,Schedule |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-Event-Dependency-Inventory.csv" `
    -NoTypeInformation
```

### SCM configuration

``` powershell
$scmInventory |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-SCM-Configuration.csv" `
    -NoTypeInformation
```

### App Insights configuration

``` powershell
$aiInventory |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-AppInsights-Configuration.csv" `
    -NoTypeInformation
```

### Diagnostic settings

``` powershell
$diagnosticSummary |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-Diagnostic-Summary.csv" `
    -NoTypeInformation
```

## 10. Evidence Files

Current evidence set:

``` text
C:\AzureFunctionInventory\Day3\Investigation
DEV-Function-Inventory.csv
DEV-Binding-Inventory.csv
DEV-Event-Dependency-Inventory.csv
DEV-SCM-Configuration.csv
DEV-AppInsights-Configuration.csv
DEV-Diagnostic-Summary.csv
```

The CSVs are not merely attachments. They are the reproducible evidence
behind the architecture.

## 11. What the 30-Function Map Demonstrates

This map demonstrates that the work is deeper than simply saying:

> "There are 10 Azure Function Apps."

We can now explain:

``` text
10 Function Apps
       ↓
30 Functions
       ↓
runtime/language
       ↓
trigger/binding
       ↓
external dependency
       ↓
application relationship
       ↓
deployment model
       ↓
observability configuration
```

That is the core SRE discovery progression.

## 12. Current Observability Findings

Application Insights app-setting checks found:

-   Most DEV apps have `APPLICATIONINSIGHTS_CONNECTION_STRING`.
-   Some also have `APPINSIGHTS_INSTRUMENTATIONKEY`.
-   `helios-dev-cost-ingestion` showed neither of the two checked
    settings.

Resource-level diagnostic settings:

``` text
10 / 10 queried Function Apps
DiagnosticSettingsFound = No
```

Interpretation:

> No resource-level diagnostic settings were returned by the queried
> API. This does not by itself prove that no telemetry or monitoring
> exists.

Next we must trace the actual Application Insights resource, telemetry
destination, alert rules and Action Groups.

## 13. Current Release-Orchestration Findings

For:

``` text
qcells-hqct / helios-plan-narration-backend
```

DEV workflow:

``` text
.github/workflows/deploy-function-app.yml
```

Target:

``` text
ems-plan-narration-function
```

Current conceptual path:

``` text
GitHub push/dispatch
        |
        v
CI
  pip install
  compileall
        |
        v
Deploy
  azure/login
  app settings
  Azure/functions-action
        |
        v
ems-plan-narration-function
        |
        v
state + function registration validation
```

Current SRE observations:

-   pytest is in a separate workflow and is not wired as a deployment
    dependency.
-   The deploy CI therefore provides a weak syntax gate.
-   Promotion DEV → QA → PROD is not fully enforced by the examined
    workflows.
-   Build-once immutable artifact promotion is not demonstrated.
-   Deployment validation exists.
-   Automatic rollback is not present in the examined workflow.
-   Exact Environment approval behavior still needs repository settings
    verification.

## 14. Current SRE Capability Position

The capability matrix started with Azure Functions largely `unknown`.

Current DEV evidence moves these areas toward:

``` text
Resource inventory       = mapped
Function inventory       = mapped
Trigger/bindings         = mapped
Event dependencies      = partially mapped
Durable architecture    = mapped for identified app
Platform configuration  = partially mapped
CI/CD                   = mapped for examined repository
Observability config    = partially mapped
Alerting                = not yet mapped
Scheduled verification  = not yet mapped
Reporting/audit         = not yet mapped
Identity/RBAC           = not yet mapped
Networking              = not yet mapped
SLO/SLI                 = not yet mapped
```

## 15. What We Will Do Next in DEV

Do not implement remediation yet.

Next discovery sequence:

``` text
1. Managed Identity
        ↓
2. RBAC
        ↓
3. Key Vault access
        ↓
4. Storage/Event Hub/Service Bus permissions
        ↓
5. VNet / network integration
        ↓
6. Actual Application Insights resource mapping
        ↓
7. Alert rules + Action Groups
        ↓
8. Runtime metrics
        ↓
9. Event Hub / Service Bus health
        ↓
10. Ownership + SLO/SLI + incident path
        ↓
11. Final DEV architecture
```

Then compare the completed DEV model with QA and PROD.

## 16. Monday Meeting --- How to Prove the Work

The strongest presentation is not a list of commands. It is an evidence
chain.

Say:

> "I started from the capability matrix where Azure Functions was
> unknown. I selected DEV and built the inventory from the Azure
> resource layer down to individual Functions and their bindings. I then
> mapped how each Function is triggered, what event or service it
> depends on, how the application is deployed, and what observability
> configuration exists. Every completed layer has a CSV evidence file. I
> am now moving into identity, RBAC, networking and alerting before
> proposing any implementation."

### Demonstration sequence

``` text
1. Capability matrix
2. DEV Function App inventory
3. 30-function table
4. Trigger/binding map
5. Event dependency map
6. Durable Functions architecture
7. SCM/runtime CSV
8. CI/CD workflow
9. App Insights CSV
10. Diagnostic Summary CSV
11. Unknown/open areas
12. Next SRE discovery plan
```

## 17. Management-Level Architecture Statement

Use this:

> "The DEV Function estate is not one homogeneous workload. It contains
> HTTP APIs, webhooks, Event Hub consumers, Blob-triggered processors, a
> scheduled ingestion function, Service Bus consumers, and a Durable
> Functions orchestration system. We have mapped these workloads down to
> the individual Function level and preserved the evidence in CSV
> inventories. The next step is to connect this workload map to
> identity, permissions, network paths, observability and operational
> ownership so that we can produce a complete SRE view rather than only
> an Azure resource inventory."

## 18. Final Principle

``` text
DISCOVER
   ↓
VERIFY
   ↓
SAVE EVIDENCE
   ↓
MAP ARCHITECTURE
   ↓
EXPLAIN CURRENT STATE
   ↓
IDENTIFY OPERATIONAL GAPS
   ↓
GET ENGINEERING DECISION
   ↓
IMPLEMENT LATER
```

**No remediation is being performed during this discovery phase.**
