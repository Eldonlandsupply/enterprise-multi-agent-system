## A01 – Cognitive Analysis & Generation (Azure OpenAI Agent)

### Summary

The Cognitive Analysis & Generation agent uses Azure OpenAI (GPT‑based) capabilities to analyse unstructured text and generate human¡readable narratives.  It summarises reports, drafts communications and derives insights from data by performing natural¡language reasoning.  It supports business decision¡making by translating raw information into actionable summaries.

### Responsibilities & Tasks

**Responsibilities**

- Generate summaries and reports from unstructured data sources.
- Analyse sentiment, intent and key themes in communications.
- Retrieve additional information from knowledge bases when composing responses.
- Persist outputs to shared storage for downstream consumption.

**Concrete tasks**

1. **Daily summary generation** – runs on a scheduled trigger (e.g. 08:00) to summarise the latest business data and ML predictions.  Inputs: Dataverse tables containing overnight data and A03 prediction results.  Outputs: Writes summaries to a Dataverse table and posts a notification to the orchestrator (A02).
2. **Ad hoc analysis** – triggered by an event from A02 when a new request arrives (e.g. user asks a question via Teams).  Inputs: Query parameters and relevant data fetched via Dataverse or external APIs.  Outputs: Returns a generated response to A02; stores a record of the response in Dataverse for audit.
3. **Knowledge retrieval** – uses retrieval‑augmented generation via Cognitive Search or custom tools to fetch facts from structured data stores.  Runs as part of other tasks when additional context is required.  Inputs: Search query; outputs: list of retrieved documents passed to the language model.

### Integrations & Dependencies

| Component | Purpose | Direction | Connector/Protocol |
|-----------|--------|---------|--------------------|
| **Dataverse** | Store and retrieve input data (e.g. daily metrics, ML predictions); persist generated outputs and metadata | Read & write | Power Automate Dataverse connector |
| **Azure ML endpoint** | Access predictive scores for inclusion in reports | Read (call) | HTTPS/REST |
| **Cognitive Search / custom data API** | Retrieve relevant knowledge documents for grounding responses | Read | HTTP via custom tool |
| **Power Automate/Logic App** | Receive triggers and return outputs | Trigger & callback | HTTP actions |
| **Teams/Outlook** | Send notifications or responses to users when appropriate | Write | Teams connector |

### Triggers, Inputs & Outputs

**Triggers**: Scheduled recurrence (daily at a specified time); external events via HTTP when A02 requests ad‑hoc analysis.

**Inputs**: Dataverse tables containing raw data and predictions, ML endpoint scores, retrieval documents.  Each request includes a JSON payload specifying the context and desired output type.

**Outputs**: Writes structured summaries to a Dataverse table (`Summaries`), posts notifications/messages via Power Automate/Teams, and returns JSON responses to A02.

### Error Handling & Resilience

The agent retries transient failures (e.g. API timeouts) up to three times with exponential backoff.  It logs all inputs and outputs to Dataverse for auditing.  If the language model returns an error or the response fails policy checks, the agent returns a fallback message and records the incident.  State is not held in memory; every request is stateless, drawing context from Dataverse.  Idempotency keys ensure duplicate triggers do not produce duplicate records.

### Security & Access

Runs under a dedicated managed identity registered in Microsoft Entra ID.  The identity has read access to input tables and write access to the `Summaries` table in Dataverse, and permission to invoke Azure ML endpoints and Cognitive Search.  Network communication uses HTTPS.  Prompt inputs are screened via Purview/AI guardrails to prevent sensitive data leaks.  Logs are emitted to Azure Monitor under this identity’s scope.
