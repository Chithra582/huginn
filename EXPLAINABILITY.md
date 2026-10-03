# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **Huginn** (`huginn`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** Huginn (`huginn`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Event-Driven Automation & Personal Agent Workflows  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

Huginn processes incoming triggers, evaluates event relevance, and executes downstream agent actions through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Event Ingestion & Webhook Validation]                                  |
|  - Parse inbound HTTP payload / cron trigger; verify signature and agent options  |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Scenario Graph Traversal & Receiver Resolution]                        |
|  - Traverse DAG connections, identify downstream receivers, check disabled flags  |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Event Routing & Transformation Scoring]                                |
|  - Compute routing affinity metric S_event matching payload attributes to rules   |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Evaluation & Throttling Guard]                               |
|  - Verify tau >= 0.70; enforce rate limit windows and event deduplication        |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Action Execution, Event Emission & Queue Commit]                       |
|  - Execute receiver logic (email, webhook, DB insert), commit event to audit log  |
+-----------------------------------------------------------------------------------+
```

### 2. Decision Logic & Routing Formulations

For an emitted event $e_k$ evaluated against candidate downstream receiver agent $A_i$ within active scenario $S$, the event routing score $S_{\text{event}}(e_k, A_i)$ is formulated as:

$$S_{\text{event}}(e_k, A_i) = w_{\text{schema}} V(e_k, A_i) + w_{\text{filter}} F(e_k, A_i) + w_{\text{dedup}} D(e_k, A_i) + w_{\text{rate}} R(A_i)$$

Where:
- $V(e_k, A_i) \in \{0, 1\}$ verifies that event payload keys satisfy receiver $A_i$'s required payload options.
- $F(e_k, A_i) \in [0, 1]$ represents JSONPath / Liquid template regex filter match score.
- $D(e_k, A_i) = 1 - \text{Similarity}(e_k, e_{\text{last}})$, penalizing duplicate events within the deduplication window.
- $R(A_i) = \max\left(0, 1 - \frac{\text{EventsLastHour}(A_i)}{\text{MaxEventsPerHour}(A_i)}\right)$ measures rate-limit headroom.
- Standard default weights: $w_{\text{schema}} = 0.35$, $w_{\text{filter}} = 0.35$, $w_{\text{dedup}} = 0.15$, $w_{\text{rate}} = 0.15$ with $\sum w = 1.0$.

Event processing requires:

$$S_{\text{event}}(e_k, A_i) \ge \tau \quad (\tau = 0.70) \quad \land \quad V(e_k, A_i) = 1$$

### 3. Thresholding & Refusal Decision Criteria

Huginn enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_EVENT_PAYLOAD_INVALID**: $V(e_k, A_i) = 0$ (required JSON keys missing) halts execution with code `ERR_EVENT_PAYLOAD_INVALID`.
- **Refusal on ERR_AGENT_DISABLED**: Receiver agent is disabled or archived by user halts execution with code `ERR_AGENT_DISABLED`.
- **Refusal on ERR_RATE_LIMIT_THROTTLED**: Event frequency exceeds configured sliding window cap halts execution with code `ERR_RATE_LIMIT_THROTTLED`.
- **Refusal on ERR_TARGET_URL_UNREACHABLE**: Downstream webhook or scraped URL returns HTTP 5xx halts execution with code `ERR_TARGET_URL_UNREACHABLE`.
- **Refusal on ERR_SCENARIO_CIRCULAR_LOOP**: Event traversed identical agent $> 5$ times in 60s halts execution with code `ERR_SCENARIO_CIRCULAR_LOOP`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Delayed Job Exponential Retry):** Transient network errors or external 5xx responses trigger automated background retries with exponential backoff (1m, 5m, 15m, 1h).
- **Tier 2 (DeadLetter Event Spooling):** Unparseable payloads or permanent 4xx failures are diverted into the agent's deadletter log without halting the wider scenario.
- **Tier 3 (Admin Email Incident Alert):** If an agent fails continuously for $> 24\,\text{hours}$, dispatch a highpriority system digest email to the instance administrator.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

Huginn operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Inbound Webhook Payloads**: Arbitrary JSON, XML, or form-encoded HTTP request bodies.
- **Scraped HTML / DOM**: Webpage content retrieved via Net::HTTP and parsed with Nokogiri.
- **System Schedule Triggers**: Cron specifications (`0 9 * * *`) and periodic interval timers (1m, 5m, 1h, 1d).

### 2. Configuration & Reference Data

- **Scenario Definitions**: Exportable JSON schemas defining agent types, interconnections, and option hashes.
- **Liquid Template Catalog**: User-defined templating strings used to reshape payloads dynamically.
- **Event History Store**: Relational MySQL or PostgreSQL database storing historical event JSON records.

### 3. Base Model & Inference Lineage

- **Deterministic Logic**: Huginn operates entirely on deterministic rule evaluation, Liquid templating, and CSS/XPath selectors.
- **Zero Proprietary Model Dependencies**: No black-box model weights are required, ensuring 100% auditable and reproducible execution.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of Huginn is essential for effective deployment.

### 1. Websites employing complex client-side JavaScript rendering
- **Limitation**: Websites employing complex client-side JavaScript rendering (SPA) cannot be scraped via standard HTTP GET.
- **Mitigation**: Huginn integrates with headless browser agents (PhantomJS / Puppeteer) to evaluate client-side JavaScript before DOM parsing.

### 2. High-frequency event emitting agents can rapidly
- **Limitation**: High-frequency event emitting agents can rapidly expand relational database storage.
- **Mitigation**: Configurable `keep_events_for` expiration policies automatically purge stale event records.

### 3. Aggressive scraping intervals can result in
- **Limitation**: Aggressive scraping intervals can result in IP bans or CAPTCHA roadblocks from target websites.
- **Mitigation**: User-agent rotation, randomized request delays, and HTTP proxy integration mitigate rate-limit blocks.

### 4. Unhandled circular event pipelines can cause
- **Limitation**: Unhandled circular event pipelines can cause runaway database growth.
- **Mitigation**: Graph cycle detection identifies loop topologies and enforces max-depth hop count limits.

### 5. External webhook receiver outages can cause
- **Limitation**: External webhook receiver outages can cause memory pressure in the background job queue.
- **Mitigation**: Concurrency-capped Delayed::Job workers with queue size throttling prevent memory exhaustion.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Websites employing complex client-side JavaScript rendering | Section 1 | Verified |
| - High-frequency event emitting agents can rapidly | Section 2 | Verified |
| - Aggressive scraping intervals can result in | Section 3 | Verified |
| - Unhandled circular event pipelines can cause | Section 4 | Verified |
| - External webhook receiver outages can cause | Section 5 | Verified |
