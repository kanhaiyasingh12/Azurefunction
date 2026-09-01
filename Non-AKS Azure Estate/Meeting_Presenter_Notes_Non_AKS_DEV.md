# Non-AKS Azure Estate – DEV Discovery
## Meeting Presenter Notes

---

# 1. Executive Summary

###  Opening Statement

> Hi everyone. I completed the discovery of the remaining Non-AKS Azure compute estate in the DEV environment.
> In total, we assessed 37 Non-AKS compute resources across 10 Resource Groups to build a validated baseline.
>
> The Method (Our SRE Principle): I follow a strict, standardized pipeline: DISCOVER ➔ VERIFY ➔ SAVE EVIDENCE ➔ MAP ARCHITECTURE ➔ EXPLAIN CURRENT STATE ➔ IDENTIFY GAPS ➔ GET DECISION ➔ IMPLEMENT LATER

---

# 2. Scope & Inventory Baseline

## 2.1 What I Did

> I validated the current inventory against the previous baseline.
>
> The 8/3 reported **47 Azure compute resources**. After validating the subscription, I confirmed the inventory as **10 Azure Function Apps and 37 Non-AKS Azure compute resources**, distributed across **10 Resource Groups**.

## 2.2 Resource Breakdown

| Resource Type | Count |
|---|---:|
| Container Apps | 14 |
| Static Web Apps | 10 |
| App Services | 6 |
| Container App Environments | 6 |
| Logic App | 1 |
| **Total Non-AKS Resources** | **37** |

## 2.3 Key Takeaway

> This gave us a validated baseline before moving into the detailed technical assessment of each resource.

---

# 3. Finding #1 – Unmanaged Infrastructure

## 3.1 Resource

`ca-sopfactory-ui-dev`

## 3.2 What I Found

> During the infrastructure review, I found Container App `ca-sopfactory-ui-dev`.
>
> This Container App is actively serving the SOP Factory UI on port 8090, but it is completely untracked by our Infrastructure as Code (IaC).

### Current State

- No Terraform state found in `iac-coverage-DEV.csv`.
- Zero ownership tags attached to the resource.
- It is running 10 critical environment variables linking to Key Vault secret references (`azure-storage-key`, `durable-code`, `github-token`).
- It was spun up manually via ad-hoc Azure CLI/Portal commands by development teams for testing.

## 3.3 Why It Matters

> The main risk here is severe configuration drift and a single point of failure.

Without Terraform management:
- Automated pipelines are blind to it and cannot safely deploy updates.
- Manual changes are untraceable, creating configuration inconsistencies.
- In a disaster recovery scenario, recreating this exact environment state would be extremely difficult.

## 3.4 Recommendation

> I captured the existing runtime configuration (including all 10 env vars and port bindings) and prepared a Terraform module.


---

# 4. Finding #2 – Service Bus Dead-Letter Queue Backlog

## 4.1 What I Reviewed

> I reviewed the messaging layer across the DEV Service Bus namespaces to verify message processing health and identify bottlenecks.

## 4.2 What I Found

> I uncovered a massive, permanent backlog of dead-lettered messages silently accumulating in our DEV Service Bus.

### Current Backlog (in `helios-knowledgegraph-events` topic)

- Subscription `kg-event-processor` → **18,169 dead-lettered messages**
- Secondary Queue `dml-processor-dlq` → **17,411 active messages** sitting unprocessed.

## 4.3 What Does This Mean?

> Azure Service Bus is attempting to retry failed messages up to a Maximum Delivery Count of **50 attempts**, holding a lock for 5 minutes each time.
>
> Because dead-letter message expiration (`DeadLetteringOnMessageExpiration`) is set to False, these 18,169 messages failed 50 times and are now permanently stuck, consuming roughly 27.8 MB of namespace quota.

## 4.4 Why Is This Important?

> This is a massive blind spot for our data ingestion pipeline.

Potential impact:
- True business events (like Knowledge Graph DML mutations) are failing and being permanently lost.
- Downstream systems are missing important updates due to schema parsing errors or timeouts.
- The sheer volume of old failures hides any new, critical production issues that might arise today.

## 4.5 Recommendation

> We cannot blindly delete 18,000+ messages, as they represent dropped business events.

---

# 5. Finding #3 – Duplicate UUDRI Environment

## 5.1 What I Found

> While mapping the architecture, I identified what looked like redundant infrastructure for the UUDRI (Utility Usage Data & Rate Intelligence) system.
>
> Tracing the deployment repositories, branches, and ownership revealed two completely separate, parallel stacks running simultaneously.

## 5.2 Personal Development Stack

### Resource Group: `uudri-dev-rg`

- **Frontend:** SWA `white-coast` (tracking `develop` branch)
- **Backend:** App Service `UUDRI-App-Service-dev-01`
- **Ownership:** Tagged to an individual personal email (`divyanshu.arya@...`)
- **Storage/DB:** Uses isolated Blob containers and Postgres connections.

## 5.3 Shared Team Environment

### Resource Group: `helios-dev-us-west3-rg`

- **Frontend:** SWA `thankful-mud` (tracking `helios-develop` branch)
- **Backend:** App Service `UUDRI-Foundry-App-Service-dev-01`
- **Ownership:** Tagged correctly to `DevOps`
- **Storage/DB:** Uses a separate set of storage resources, causing data fragmentation.

## 5.4 Why Is This Important?

> Running dual stacks creates severe data fragmentation and operational confusion.

Potential issues:
- Developers testing on the personal stack are hitting different databases than those on the team stack.
- Unnecessary compute costs from duplicating App Services, Storage Accounts, and Key Vaults.
- No single source of truth for the UUDRI system in DEV.

## 5.5 Recommendation

> We must standardize all UUDRI development onto the shared DevOps-managed environment.

---

# 6. Finding #4 – Container Health Probes

## 6.1 Resource

`ca-model-service-dev`

## 6.2 What I Found

> This Container App hosts our GPT-4.1 inference workload, but it currently has **zero health probes** configured.

### Missing Probes

- Startup Probe
- Readiness Probe
- Liveness Probe

## 6.3 Why Does This Matter?

> Health probes tell Azure when a container is booted, ready to take traffic, and still healthy. Without them, Azure blindly routes traffic during slow cold-starts. This is critical because LLM inference requests can easily exceed 30 seconds.

## 6.4 Potential Impact

- Azure gives up during cold starts and returns **502 Bad Gateway / 504 Gateway Timeout** errors to the caller.

## 6.5 Recommendation

- Add a dedicated Startup Probe with a generous initial delay (e.g., 30s).
- Add a lightweight `/health` Readiness Probe in a new container revision so Azure knows exactly when it is safe to route traffic.

---

# 7. Finding #5 – Observability & Security Gaps

## 7.1 Observability Finding

> The biggest blind spot in our DEV environment is that **35 out of 37 resources do not have Diagnostic Settings configured**.

## 7.2 Impact

Without Diagnostic Settings:
- Platform and host logs are completely lost.
- Incident investigation relies purely on application-level telemetry, ignoring infrastructure faults.

## 7.3 Additional Findings

- **Availability:** 0 URL Ping Tests configured.
- **Network Isolation:** 5/6 App Services and 5/6 Container App Environments have no VNet Integration.
- **Identity:** 4/6 App Services have no Managed Identity configured.

## 7.4 Recommendation

> Enable centralized logging to Log Analytics, improve network isolation, and configure Managed Identities across the board.

---

# 8. Addressing the Automated Health Report (The Slack Message)

## 8.1 The Context

> You may have seen the automated Slack alert from the `helios-qre-deployer[bot]` highlighting health issues. 
> 
> **This alert perfectly validates our SRE discovery.** It flags long-standing technical debt (≥27 days old), not an active crash, and specifically targets two resources we just mapped.

## 8.2 What Was Flagged & How We Are Fixing It

1. **`ca-model-service-dev` (Flagged for Latency):**
   - **Why it fired:** As noted in Finding #4, this container lacks Startup/Readiness probes, causing Azure to throw 502/504 timeouts on cold starts.
   - **The Fix:** We are already scoping the addition of Startup and Readiness probes in Phase 3 of our remediation plan.
2. **`helios-dev-memo-app` (Flagged for Health Check Mismatch):**
   - **Why it fired:** Its Azure App Service health check is pointing to the root `/` path. Because the root path requires authentication/database checks, internal Azure pings are failing.
   - **The Fix:** We will update the code to expose a lightweight, unauthenticated `/health` endpoint and update the App Service configuration. This is scoped for Phase 1.

> **Key Takeaway for Leadership:** We already have complete visibility into the issues flagged by the automated bot, and their remediations are fully integrated into our rollout plan.

---

# 10. Key Numbers

| Metric | Value |
|---|---:|
| Total Non-AKS Resources | **37** |
| Container Apps | **14** |
| Static Web Apps | **10** |
| App Services | **6** |
| `kg-event-processor` DLQ | **18,169** |
| `dml-processor-dlq` | **17,411** |
| Resources Missing Diagnostic Settings | **35 / 37** |
| Availability Tests | **0** |

---

# 11. Expected Questions

## Q1. Can we decommission `helios-mcp-chatbot-app`?
> Yes, provided we confirm it is no longer required. Decommissioning it reduces compute costs and our attack surface.

## Q2. Should we delete the 18,169 DLQ messages?
> Not immediately. We should inspect a sample, understand the failure reason, and determine whether they need to be replayed or safely purged.

## Q3. Which UUDRI environment should we keep?
> The shared DevOps-managed environment in `helios-dev-us-west3-rg` should become the standard. The personal environment can be retired after dependency validation.

## Q4. What should we do about the health probes?
> Configure Startup and Readiness probes with appropriate delays and thresholds, then validate behavior through a new container revision.

## Q5. What are the next steps?
> DEV discovery is now complete. We will apply this exact same methodology to QA and Production to build a complete cross-environment capability matrix.

---

# 12. Closing Statement

> The DEV discovery is now complete. I have a validated inventory, a clear understanding of the Non-AKS architecture, and a detailed view of the key operational risks.
>
> The main areas requiring attention are **Service Bus message failures, infrastructure governance, duplicate environments, application reliability, and observability**. We also have a prioritized remediation plan that can be executed in phases, including resolving the issues highlighted in the daily automated Slack reports.
>
> The next step is to repeat this discovery process for **QA and Production**.
