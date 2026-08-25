# SRE Discovery & Architecture Mapping --- DEV Environment

**Project:** SRE-Project\
**Purpose:** Current-state discovery, architecture mapping,
release-orchestration assessment, and evidence collection\
**Environment:** DEV\
**Status:** Discovery only --- no remediation or implementation changes
are being made.

## 1. Executive Summary

The assignment is to understand the company's current platform from an
SRE perspective and present the current-state architecture before
implementing changes.

The investigation follows:

**Discover → Verify → Record Evidence → Map → Explain → Identify Gaps →
Decide → Implement later**

The broader capability matrix contains Kubernetes/ArgoCD, Frontend,
Foundry, Azure Functions, Flyway DB migrations, Non-AKS Azure estate,
and Fabric/Knowledge Graphs.

We selected **Azure Functions in DEV** as the first deep-dive because
that area was largely marked `unknown`.

The broader inventory discussion identified approximately **27 Function
Apps across DEV/QA/PROD**. The matrix also refers to approximately 87
Azure Functions across three environments. These numbers still need
reconciliation. For the current DEV deep-dive, the working scope
contains **10 Function Apps and 30 registered Functions**.

## 2. Investigation Method

We are mapping the system layer by layer:

1.  Scope and resource inventory
2.  Function App inventory
3.  Individual Function inventory
4.  Runtime/language
5.  Trigger and binding mapping
6.  Event/data dependencies
7.  Platform/SCM configuration
8.  Deployment history
9.  CI/CD/release orchestration
10. Application Insights configuration
11. Azure Monitor diagnostic settings
12. Managed identity
13. RBAC
14. Key Vault/access relationships
15. Networking
16. Alerts and Action Groups
17. Runtime health/metrics
18. Ownership/SLO/incident path
19. Final end-to-end architecture

Rule: do not conclude from one API. Cross-check related configuration
and preserve the evidence.

## 3. DEV Function App Inventory

Current working DEV Function Apps:

``` text
uudri-bill-processor-dev
helios-ontology-event-processor-func
helios-github-activity-logger-dev-func
func-projector-sopfactorydevmlel9
helios-dev-cost-ingestion
func-orchestrator-sopfactorydevmlel9
helios-device-telemetry-dev-func
ems-plan-narration-function
kg-event-processor-dev
UUDRI-Bill-Processor-dev-01
```

These are Function Apps, not individual Functions.

Current result:

``` text
10 Function Apps
30 registered Functions
```

## 4. Function Inventory --- Command

``` powershell
$apps = az functionapp list `
  --query "[?contains(resourceGroup,'helios-dev') || contains(resourceGroup,'uudri-dev')].{Name:name,RG:resourceGroup}" `
  -o json |
  ConvertFrom-Json

$apps | Format-Table Name,RG -AutoSize
```

For individual Functions we used the ARM REST API because
`az functionapp function list` encountered Azure CLI/Python/JMESPath
problems.

``` powershell
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
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-Function-Inventory.csv" `
    -NoTypeInformation
```

Count:

``` powershell
$functionInventory |
    Where-Object { $_.FunctionName -ne "" } |
    Measure-Object
```

Per-App count:

``` powershell
$functionInventory |
    Where-Object { $_.FunctionName -ne "" } |
    Group-Object FunctionApp |
    Select-Object Name,Count |
    Format-Table -AutoSize
```

Current total: **30 Functions**.

## 5. Trigger / Binding Mapping

We mapped inbound bindings to understand how workloads are invoked.

Current trigger counts:

``` text
httpTrigger             12
activityTrigger          8
durableClient             5
eventHubTrigger           4
blobTrigger               3
serviceBusTrigger         1
orchestrationTrigger      1
timerTrigger              1
```

Binding inventory script:

``` powershell
$bindingInventory = foreach ($app in $apps) {

    Write-Host "Reading bindings: $($app.Name)" -ForegroundColor Cyan

    $url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/functions?api-version=2022-03-01"

    $response = az rest --method get --url $url -o json |
        ConvertFrom-Json

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
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-Binding-Inventory.csv" `
    -NoTypeInformation
```

## 6. Event / External Dependency Mapping

We isolated event-driven and scheduled triggers:

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
    Select-Object `
        FunctionApp,
        FunctionName,
        TriggerType,
        Connection,
        EventHubName,
        QueueName,
        TopicName,
        SubscriptionName,
        BlobPath,
        Schedule |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-Event-Dependency-Inventory.csv" `
    -NoTypeInformation
```

Confirmed examples:

``` text
ontology_event_processor
    → Event Hub

new_device_and_telemetry_classification_agent
    → Event Hub

plan_narration_agent
    → Event Hub

realized_kpi_listener
    → Event Hub

ProcessDMLEvent
    → Service Bus
    → Topic: helios-knowledgegraph-events
    → Subscription: kg-event-processor

cost_ingestion
    → Timer
    → 0 0 6 * * *

BillProcessor / onSourceDocument
    → Blob trigger
```

This converts a resource list into a runtime dependency map.

## 7. Logic App Discovery

Three Logic Apps/workflows were investigated:

``` text
helios-data-eventstream-monitor-dev
helios-data-market-eh-eventstream-monitor-dev
helios-data-weather-eh-eventstream-monitor-dev
```

Important workflow actions included:

``` text
Get_Config
For_each_Stream
Query_Eventhouse
Query_IncomingMessages
Compute_Delta
Compute_EventDeliveryRatio
Condition_Non_Ok
Emit_HealthCheck
```

The HTTP URI values were found to be workflow expressions rather than
simple literal URLs. Therefore they should not be treated as static
dependency hosts without resolving the expression/config source.

## 8. SCM / Platform Configuration

We queried:

``` text
Microsoft.Web/sites/<app>/config/web
```

Fields:

``` text
SCMType
LinuxFxVersion
AlwaysOn
FTPS
PublicNetwork
MinTLS
```

Script:

``` powershell
$scmInventory = foreach ($app in $apps) {

    $url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/config/web?api-version=2022-03-01"

    $obj = az rest --method get --url $url -o json |
        ConvertFrom-Json

    [PSCustomObject]@{
        Environment = "DEV"
        FunctionApp = $app.Name
        ResourceGroup = $app.RG
        SCMType = $obj.properties.scmType
        LinuxFxVersion = $obj.properties.linuxFxVersion
        AlwaysOn = $obj.properties.alwaysOn
        FTPS = $obj.properties.ftpsState
        PublicNetwork = $obj.properties.publicNetworkAccess
        MinTLS = $obj.properties.minTlsVersion
    }
}

$scmInventory |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-SCM-Configuration.csv" `
    -NoTypeInformation
```

Observed runtimes include:

``` text
PYTHON|3.11
PYTHON|3.12
NODE|20
dotnet-isolated
```

The displayed DEV inventory showed `MinTLS = 1.2`.

## 9. Deployment Evidence

Deployment history was queried through:

``` text
Microsoft.Web/sites/<app>/deployments
```

Example:

``` powershell
$app = $apps | Where-Object {
    $_.Name -eq "helios-github-activity-logger-dev-func"
}

$url = "https://management.azure.com/subscriptions/a6498579-cfb7-41e9-a957-14375196a386/resourceGroups/$($app.RG)/providers/Microsoft.Web/sites/$($app.Name)/deployments?api-version=2022-03-01"

az rest --method get --url $url -o json
```

Observed evidence for `helios-github-activity-logger-dev-func` included:

``` text
deployer: az_cli_functions
message: Created via a push deployment
complete: true
status: 4
```

Some apps returned an empty deployment list. That means only that the
queried API returned no deployment records; it is not proof that an app
was never deployed.

## 10. CI/CD / Release Orchestration

Repository investigated:

``` text
qcells-hqct / helios-plan-narration-backend
```

Workflows:

``` text
.github/workflows/deploy-function-app.yml
.github/workflows/deploy-function-app-qa.yml
.github/workflows/deploy-function-app-prod.yml
.github/workflows/test-function-app.yml
```

DEV target:

``` text
ems-plan-narration-function
```

DEV Key Vault referenced by the workflow:

``` text
helios-dev-backend-kv
```

DEV release flow:

``` text
push/dispatch
    ↓
CI
    ↓
python -m compileall
    ↓
Deploy
    ↓
ems-plan-narration-function
    ↓
state/registration validation
```

Important SRE observations:

1.  The deploy workflow does not directly run pytest.
2.  `test-function-app.yml` is separate and is not linked through
    `needs:` or `workflow_call`.
3.  CI is therefore primarily a syntax/compile gate.
4.  There is no demonstrated build-once/immutable-artifact promotion.
5.  DEV/QA/PROD are represented by separate workflows rather than a
    fully enforced promotion chain.
6.  PROD is manually dispatched in the examined workflow.
7.  Post-deployment validation exists, but there is no automatic
    rollback step.
8.  The exact approval behavior depends partly on GitHub Environment
    protection rules, which must be verified separately.

## 11. GitHub Secrets and Variables

Repository Actions secrets visible during the investigation:

``` text
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_SUBSCRIPTION_ID
AZURE_TENANT_ID
```

Repository variables included:

``` text
APIM_CLIENT_ID
APIM_SCOPE
APIM_TOKEN_URL
AZURE_AI_FOUNDRY_ENDPOINT
EMS_PLAN_NARRATION_AGENT_ID
EVENTHUB_CONSUMER_GROUP
EVENT_HUB_NAME
OBJECT_STORE_SERVICE_URL
```

Never place secret values in the report. Record only names and whether
configuration is present.

## 12. Application Insights Configuration

App settings were queried with:

``` powershell
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
        Environment = "DEV"
        FunctionApp = $app.Name
        ResourceGroup = $app.RG
        AppInsightsConnection = if ($settings.ContainsKey("APPLICATIONINSIGHTS_CONNECTION_STRING")) { "Present" } else { "Missing" }
        AppInsightsInstrumentationKey = if ($settings.ContainsKey("APPINSIGHTS_INSTRUMENTATIONKEY")) { "Present" } else { "Missing" }
        FunctionsExtensionVersion = if ($settings.ContainsKey("FUNCTIONS_EXTENSION_VERSION")) { $settings["FUNCTIONS_EXTENSION_VERSION"] } else { "" }
        FunctionsWorkerRuntime = if ($settings.ContainsKey("FUNCTIONS_WORKER_RUNTIME")) { $settings["FUNCTIONS_WORKER_RUNTIME"] } else { "" }
    }
}

$aiInventory |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-AppInsights-Configuration.csv" `
    -NoTypeInformation
```

Current observation:

-   Most DEV apps have an Application Insights connection string.
-   Some also have the legacy instrumentation-key setting.
-   `helios-dev-cost-ingestion` showed neither of the two checked App
    Insights settings.

Important interpretation:

> Missing instrumentation key alone does not mean App Insights is
> absent; connection-string based configuration may be used.

## 13. Azure Monitor Diagnostic Settings

We queried:

``` text
Microsoft.Insights/diagnosticSettings
```

for all 10 Function Apps.

Summary script:

``` powershell
$diagnosticSummary = foreach ($app in $apps) {

    $found = @(
        $diagnosticInventory |
        Where-Object { $_.FunctionApp -eq $app.Name }
    )

    [PSCustomObject]@{
        Environment = "DEV"
        FunctionApp = $app.Name
        DiagnosticSettingsFound = if ($found.Count -gt 0) { "Yes" } else { "No" }
        DiagnosticSettingCount = $found.Count
    }
}

$diagnosticSummary |
    Export-Csv `
    "C:\AzureFunctionInventory\Day3\Investigation\DEV-Diagnostic-Summary.csv" `
    -NoTypeInformation
```

Current result:

``` text
10/10 queried DEV Function Apps
DiagnosticSettingsFound = No
```

Correct SRE interpretation:

> No resource-level Azure Monitor diagnostic settings were returned for
> the queried Function App resources.

Do not claim that "there is no monitoring" until the actual App Insights
resources, telemetry destinations, alerts, and platform-level monitoring
are traced.

## 14. Evidence Files Created

All are under:

``` text
C:\AzureFunctionInventory\Day3\Investigation
```

``` text
DEV-Function-Inventory.csv
DEV-Binding-Inventory.csv
DEV-Event-Dependency-Inventory.csv
DEV-SCM-Configuration.csv
DEV-AppInsights-Configuration.csv
DEV-Diagnostic-Summary.csv
```

## 15. What Each CSV Proves

  ------------------------------------------------------------------------
  CSV                                  Evidence
  ------------------------------------ -----------------------------------
  DEV-Function-Inventory.csv           Which Functions exist inside each
                                       Function App

  DEV-Binding-Inventory.csv            How each Function is
                                       triggered/bound

  DEV-Event-Dependency-Inventory.csv   Event Hub, Service Bus, Blob and
                                       Timer dependencies exposed by
                                       bindings

  DEV-SCM-Configuration.csv            Runtime/SCM/platform configuration

  DEV-AppInsights-Configuration.csv    Presence/absence of selected App
                                       Insights settings and Functions
                                       runtime settings

  DEV-Diagnostic-Summary.csv           Whether resource-level diagnostic
                                       settings were returned
  ------------------------------------------------------------------------

## 16. What We Have Verified vs What Remains

### Verified / partially verified

``` text
10 DEV Function Apps
30 registered Functions
Function names
Languages/runtimes
Trigger types
Event Hub dependencies
Service Bus dependency
Blob-triggered workloads
Timer schedule
Durable Function patterns
SCM/web configuration
Deployment history where exposed by ARM
DEV CI/CD workflow for plan-narration
Repository secret/variable names
App Insights-related app settings
Resource diagnostic settings
Three event-stream monitoring Logic Apps
```

### Not yet fully verified

``` text
Managed identity
RBAC
Key Vault access path
Storage permissions
Event Hub permissions
Service Bus permissions
Network/VNet integration
Private endpoints
Actual App Insights resource IDs
Alert rules
Action Groups
SLO/SLI
Runtime failure rate
Latency
throttling
retry/dead-letter behavior
Event Hub lag
Service Bus backlog
ownership/on-call
RTO/RPO
incident escalation
full QA/PROD comparison
```

## 17. Current DEV Architecture

Evidence-based conceptual model:

``` text
                         GitHub
                           |
                           | Actions / CI-CD
                           v
                  Azure Function Apps
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
       HTTP/API       Event-driven        Scheduled
                       Functions           Functions
          |                |                |
          |          +-----+-----+          |
          |          |     |     |          |
          |          v     v     v          v
          |       EventHub ServiceBus Blob  Timer
          |
          v
      Application dependencies

                           |
                           v
                    Observability
                  +--------+--------+
                  |                 |
             App Insights     Azure Monitor
```

This is a current-state discovery diagram, not the final architecture.
Identity, network paths, alerting and actual telemetry destinations
still need to be mapped.

## 18. How to Explain the Method in the Meeting

Use this sentence:

> "We did not start by looking for problems. We started by establishing
> evidence for what exists, how it is invoked, what it depends on, how
> it is deployed, and how it is observed."

Then explain the layers:

``` text
Resource
  ↓
Function
  ↓
Trigger
  ↓
Dependency
  ↓
Platform
  ↓
Deployment
  ↓
Observability
  ↓
Identity
  ↓
Network
  ↓
Alerts
  ↓
SLO/Operations
```

For every layer answer three questions:

1.  **What exists?**
2.  **How did we verify it?**
3.  **Where is the evidence?**

Example:

``` text
Question: How many Functions exist?

Method:
ARM /functions API

Evidence:
DEV-Function-Inventory.csv

Result:
30 registered Functions
```

## 19. Monday Demo Sequence

1.  Start with the capability matrix.
2.  Explain that Azure Functions was marked largely `unknown`.
3.  Explain why DEV was selected as the first deep-dive.
4.  Show the 10 Function Apps.
5.  Show the 30 Functions.
6.  Show trigger counts.
7.  Show event dependencies.
8.  Show SCM/runtime configuration.
9.  Show deployment evidence.
10. Open the GitHub workflow and explain the DEV release path.
11. Show App Insights configuration evidence.
12. Show diagnostic-settings result.
13. Show the CSV evidence folder.
14. Explain what is verified and what remains unknown.
15. Present the next discovery phase: Identity → RBAC → Networking →
    Alerts → Runtime health → SLO/ownership.
16. State clearly that no remediation has been implemented during
    discovery.

## 20. Current SRE Statement

Use this as the closing statement:

> "The DEV Azure Functions estate is now partially mapped at resource,
> workload, trigger, dependency, platform, deployment, CI/CD and
> observability-configuration levels. We have evidence for 10 Function
> Apps and 30 registered Functions in the current DEV scope. The next
> phase is to complete the operational view by mapping identity, RBAC,
> networking, actual telemetry destinations, alerting, runtime health,
> ownership and SLOs. Once DEV is coherent, the same model can be
> applied to QA and PROD for comparison and promotion-path analysis."

## 21. Investigation Principle

The project is following:

``` text
DISCOVER
   ↓
VERIFY
   ↓
EVIDENCE
   ↓
MAP
   ↓
EXPLAIN
   ↓
IDENTIFY GAPS
   ↓
GET DECISION
   ↓
IMPLEMENT LATER
```

No implementation/remediation should be performed simply because a gap
was discovered. The current phase is to establish the defensible
current-state SRE baseline.
