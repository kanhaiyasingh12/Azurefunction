# Meeting Presenter Notes — Azure Functions DEV Discovery

> Read this like a story. Each section is one beat. Say the one-liner, then use the bullets if they ask for detail.

---

## The Story in 30 Seconds

Azure Functions was unknown. I, mapped everything in DEV, in dev environment i found 10 function apps are running with 30 individual functions, and documented every trigger, every dependency, every gap 
---

## 1. Why We Did This

**"Azure Functions was marked unknown in every column of the capability matrix, so we picked it up and started with DEV."**

- The matrix covers deploy on merge, promotion gates, scheduled verification, alerting, observability, and reporting
- Azure Functions had zero documentation across all six columns
- We scoped to DEV first — 10 Function Apps, 30 Functions

---

## 2. How We Did It

**"We queried the Azure ARM API directly, pulled every function, every binding, every config — and saved it all as CSV evidence."**

- Used `az rest` calls against the ARM REST API because `az functionapp function list` had CLI bugs
- Every finding maps back to a reproducible command and an evidence file
- No changes were made — discovery only

---

## 3. What We Found — The Big Picture 

**"10 Function Apps running with 30 individual Functions,as well I also found three separate runtimes — Python, Node.js, and .NET isolated — all sitting in DEV."**

- 17 functions are Node.js (mostly the SOP Factory)
- 9 are Python (event processors, webhooks, cost ingestion)
- 4 are .NET isolated (bill processors, knowledge graph)

---

## 4. The SOP Factory — The Biggest Finding

**"One app has 16 of the 30 functions — it's a full Durable Functions pipeline that takes documents in, processes them through 8 activity steps, and opens a pull request at the end."**

- `func-orchestrator-sopfactorydevmlel9` — 16 functions, Node.js
- Documents enter via HTTP POST or blob upload
- Orchestrator chains: classify → ingest → screen → generate 4 artifact types → open PR
- Query endpoints let you check pipeline status
- If this app goes down, the entire SOP Factory stops

---

## 5. Event Hub Consumers

**"Four functions across three apps consume from Event Hubs — ontology events, device telemetry, plan narration, and realized KPIs."**

- `ontology_event_processor` uses a different connection name (`EventHubConnection`) than the other three (`EVENT_HUB_CONNECTION_STRING`)
- `realized_kpi_listener` connects to a completely separate Event Hub instance
- All are Python

---

## 6. Service Bus Consumer

**"One function listens to a Service Bus topic called `helios-knowledgegraph-events` for Knowledge Graph DML events."**

- `ProcessDMLEvent` in `kg-event-processor-dev`, .NET isolated
- Subscription name: `kg-event-processor`

---

## 7. Blob Triggers

**"Three functions fire when files land in storage — two bill processors and one that kicks off the SOP Factory orchestration."**

- Two separate bill processor apps with identical trigger config — we don't know why there are two yet
- `onSourceDocument` triggers on blob AND starts a Durable orchestration

---

## 8. The Timer

**"One function runs daily at 6 AM UTC to ingest cost data."**

- `cost_ingestion` in `helios-dev-cost-ingestion`
- Schedule: `0 0 6 * * *`
- This is also the app with no App Insights — so if it fails silently, nobody knows

---

## 9. Observability

**"9 out of 10 apps have App Insights configured. The cost ingestion app has nothing. And zero out of 10 have diagnostic settings."**

- `helios-dev-cost-ingestion` is missing both connection string and instrumentation key
- 0/10 apps export platform logs, audit logs, or scaling logs anywhere
- App Insights telemetry and diagnostic settings are separate paths — having one doesn't give you the other

---

## 10. CI/CD

**"We examined one repo — GitHub Actions deploys to `ems-plan-narration-function`, but tests aren't wired to the deploy, there's no artifact promotion, and there's no automatic rollback."**

- Separate workflow files for DEV/QA/PROD — each builds independently
- pytest exists in a separate workflow but isn't a dependency of deploy
- Post-deploy validation checks state and function registration, but if it fails, nothing rolls back
- Other apps deploy via ZipDeploy, az CLI, or Azure DevOps Releases — no single standard

---

## 11. Platform Config

**"All apps enforce TLS 1.2, all have public network access, and only the two bill processors have AlwaysOn — everything else cold-starts."**

- No VNet integration verified yet
- SCM types are mixed: Azure DevOps, GitHub Actions, None, CLI

---

## 12. Logic Apps

**"We also found 3 Logic Apps monitoring event streams — data, market, and weather — separate from the 10 Function Apps."**

- These are workflow apps, not standard function apps
- Their HTTP URIs are expressions, not static URLs

---

## 13. Capability Matrix — Where We Stand Now

**"Azure Functions went from unknown to partially mapped across deploy, promotion, and observability — but alerting, scheduled verification, identity, and networking are in progress"**

- Deploy on merge: partially mapped
- Promotion gates: partially mapped
- Scheduled verification: not started
- Alerting: not started
- Observability: partially mapped (9/10 App Insights, 0/10 diagnostic settings)
- Reporting: baseline established

---

## If They Ask

| Question | Answer |
|---|---|
| Why two bill processor apps? | Don't know yet. Same trigger config. Could be migration, versioning, or parallel. Open item. |
| Is anything broken? | We're not here to say broken. We're establishing baseline. Cost ingestion has no telemetry and nobody has diagnostic settings — whether that's acceptable is a team decision. |
| What about QA and PROD? | DEV only. Same method applies to the other environments once DEV is fully mapped. |
| How did you get the data? | ARM REST API queries via `az rest`. Exported to CSVs. |
| What's left? | Identity, RBAC, Key Vault, networking, alerts, runtime health, ownership. We mapped what exists and how it's configured. We haven't mapped who has access or who gets paged. |

---

## The Numbers to Remember

```
10 apps, 30 functions
16 of those are in one app (SOP Factory)
4 Event Hub consumers, 1 Service Bus, 3 blob triggers, 1 timer
9/10 have App Insights, 1 doesn't (cost ingestion)
0/10 have diagnostic settings
No remediation — discovery only
```
