# Meeting Presenter Notes — Non-AKS Azure Estate (DEV) Discovery

> Read this like a story. Each section is one beat. Say the one-liner, then use the bullets if they ask for detail.

---

## The Story in 30 Seconds

**"Following our completed SRE investigation on Azure Functions, we have now finished a 100% evidence-based discovery of the remaining Non-AKS Azure compute estate in DEV across 37 resources. We mapped the full architecture, resolved inventory baselines, and identified 5 key operational risks — including unmanaged infrastructure, stuck dead-letter message queues, and observability blind spots — along with a ready-to-execute remediation plan."**

---

## 1. Scope & Baseline Reconciliation (Quick Context)

**"We reconciled the 8/3 Audit baseline of 47 resources to exact matches: 10 Function Apps and 37 Non-AKS compute resources across 10 Resource Groups in DEV."**

- Total Non-AKS Resources in DEV: 37 Resources in subscription Helios – Development (`a6498579-cfb7-41e9-a957-14375196a386`).
- **14 Container Apps**: SOP Factory suite (7 apps), Site Artifact Repository (2 apps), SOC2 API, Arcadia Tariff, NLP POCs
- **10 Static Web Apps**: Engineering Dashboard, Operator Console, Reference Architecture, UI WebApp, 2 UUDRI React frontends
- **6 App Services (Web Apps)**: `heliosdev-ui-appservice`, `helios-dev-memo-app`, `qcells-warroom-dashboard`, `helios-mcp-chatbot-app`, 2 UUDRI backends
- **6 Container App Environments**: `cae-sopfactory-dev`, `cae-sar-demo-dev`, `cae-soc2-dev`, `cae-semantic-insights-poc`, etc.
- **1 Logic App (Standard)**: `helios-platform-alert-watcher`

---

## 2. Unmanaged Infrastructure (`ca-sopfactory-ui-dev`)

**"`ca-sopfactory-ui-dev` is an active, production-path UI container for SOP Factory, but its tags and Terraform state are completely null."**

- It was created via manual Azure CLI/Portal commands.
- Any automated IaC pipeline will fail to manage or update it.
- **Solution:** We have captured its full runtime JSON (port 8090, 10 env vars) and prepared a Terraform module to import it cleanly into state.

---

## 3. Service Bus DLQ Backlog (18,169 Stuck Messages)

**"In the `helios-knowledgegraph-events` Service Bus topic, the `kg-event-processor` subscription has 18,169 dead-lettered messages, plus 17,411 messages sitting in `dml-processor-dlq`."**

- Messages failed 50 delivery attempts (max delivery count) and are stuck indefinitely consuming storage and hiding dropped events.
- **Solution:** We propose a triage script to inspect sample payloads, replay or purge them, and lower the max delivery count from 50 to 5.

---

## 4. Solved the "Duplicate" UUDRI Estate Mystery

**"We uncovered why there are duplicate UUDRI resources in DEV — one is a personal stack, and the other is a team shared stack."**

- **Personal Stack (`uudri-dev-rg`):** SWA `white-coast` + App Service `UUDRI-App-Service-dev-01` wired to branch `develop`, owned by personal email divyanshu.arya.
- **Team Shared Stack (`helios-dev-us-west3-rg`):** SWA `thankful-mud` + App Service `UUDRI-Foundry-App-Service-dev-01` wired to branch `helios-develop`, owned by DevOps.
- **Solution:** Consolidate onto the shared team stack and retire the orphaned personal developer stack.

---

## 5. Container Health Probes & Latency Blind Spot (`ca-model-service-dev`)

**"The `ca-model-service-dev` LLM GPT-4.1 inference service has zero health probes configured — no Startup, Liveness, or Readiness probes."**

- During cold starts or long LLM inferences (>30s), Azure ingress returns 502/504 Bad Gateway errors instead of queuing or restarting.
- **Solution:** Add dedicated startup and readiness probes in a new container revision with a generous initial delay.

---

## 6. Observability & Security Baseline Gaps

**"Diagnostic settings are missing on 35 out of 37 resources, meaning platform and host logs are completely lost."**

- **Availability Tests:** 0 URL Ping Tests configured across all DEV resource groups.
- **Network Isolation:** 5 of 6 App Services and 5 of 6 Container App Environments have no VNet integration.
- **Identity:** 4 of 6 App Services have no Managed Identity configured.

---

## 7. Remediation Roadmap

**"We have a ready-to-execute remediation plan broken down into 3 clear phases: Quick Wins, Governance & IaC, and Reliability & Network Security."**

- **Phase 1: Quick Wins (Week 1)**
  - Triage & Decommission Stopped/Abandoned apps (e.g., `helios-mcp-chatbot-app`)
  - Purge / Replay 18,169 Service Bus DLQ messages
  - Add App Insights to 2 blind Web Apps (`qcells-warroom-dashboard`, `chatbot`)
- **Phase 2: Governance & IaC (Week 2)**
  - Import `ca-sopfactory-ui-dev` into Terraform state
  - Standardize ownership tags across all 22 untagged resources
  - Consolidate UUDRI estate into a single team-owned stack
- **Phase 3: Reliability & Network Security (Week 3)**
  - Configure Startup/Readiness probes on `ca-model-service-dev`
  - Enable Diagnostic Settings on all 37 resources -> Log Analytics
  - Configure VNet integration on CAEs and App Services

---

## If They Ask

| Question | Answer |
|---|---|
| Can we approve decommissioning `helios-mcp-chatbot-app` and the two prototype SWAs? | Yes, this will save compute costs and reduce the attack surface. |
| Should we archive the 18,169 dead-letter messages before purging? | That depends on if we need a replay pipeline for historical data or if they can be safely discarded. |
| Can we standardize UUDRI development onto `helios-dev-us-west3-rg`? | Yes, consolidating onto the team stack and decommissioning the personal stack in `uudri-dev-rg` is recommended. |
| What are the next steps? | DEV discovery is complete. We should execute the discovery sequence for QA and PROD to have the full cross-environment capability matrix ready. |

---

## The Numbers to Remember

```text
37 total Non-AKS resources (14 CAs, 10 SWAs, 6 Web Apps, 6 CAEs, 1 Logic App)
18,169 stuck dead-lettered messages in Service Bus
35 out of 37 resources missing Diagnostic Settings
0 URL Ping Tests configured
```
