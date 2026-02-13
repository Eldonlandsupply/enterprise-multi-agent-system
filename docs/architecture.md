# System Interaction Flows

This document visualizes the runtime pathways that connect external entry points to the agents defined in this repository.  Use these diagrams to align implementation details (connectors, identities, retries, and observability) with the agent specifications.

## Webhook to Agent Execution (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant External as External Publisher
    participant Ingress as Webhook Ingress (API Management + Function)
    participant Queue as Durable Queue (Service Bus)
    participant Orchestrator as A02 Workflow Orchestrator
    participant Cognitive as A01 Cognitive Analysis
    participant Utility as A04 Utility Function
    participant Dataverse as Dataverse/OneLake
    participant Alerts as Monitoring & Teams

    External->>Ingress: POST event payload (JSON)
    Ingress->>Ingress: Validate signature & schema
    Ingress-->>External: 202 Accepted
    Ingress->>Queue: Enqueue message with correlation ID
    Queue-->>Orchestrator: Triggered run (peek-lock)
    Orchestrator->>Orchestrator: Branch by event type
    alt Content processing
        Orchestrator->>Cognitive: Invoke analysis (HTTP w/ backoff)
        Cognitive-->>Orchestrator: Summaries, insights
    else Utility action
        Orchestrator->>Utility: Call function for enrichment
        Utility-->>Orchestrator: Enriched payload
    end
    Orchestrator->>Dataverse: Persist results & run log
    Orchestrator-->>Queue: Complete message lock
    Orchestrator->>Alerts: Emit success/latency metrics
    Alerts-->>External: Optional callback/notification
```

### Notes
- API Management or an Azure Function implements request validation (signatures, replay protection) and normalizes headers before writing to Service Bus.
- The orchestrator runs under a managed identity to pull from the queue and call downstream agents.
- Results are stored in Dataverse/OneLake with correlation IDs for tracing back to the original webhook call.

## Request Queue and Backoff Path (Flowchart)

```mermaid
flowchart TD
    A[Webhook accepted] --> B[Service Bus enqueue]
    B --> C{Queue length > threshold?}
    C -- Yes --> D[Scale out orchestrator worker]
    C -- No --> E[Standard consumer]
    D --> F[Peek-lock next message]
    E --> F
    F --> G{Processing succeeded?}
    G -- Yes --> H[Complete lock & emit success metric]
    G -- No --> I[Retry with exponential backoff]
    I --> J{Max retries reached?}
    J -- No --> F
    J -- Yes --> K[Move to dead-letter queue + alert]
    K --> L[Create incident in Monitoring/Teams]
```

### Notes
- Exponential backoff parameters align with connector defaults (e.g., 4 retries, max 15 minutes delay) and should mirror the orchestrator’s configuration.
- Dead-letter messages must keep the original payload, correlation ID, and last error for deterministic replay.

## Caching Path for Reused Inputs

```mermaid
flowchart LR
    Start[Orchestrator receives request] --> Check{Cache lookup by hash of payload + model version}
    Check -- Hit --> Return[Return cached response + metadata]
    Check -- Miss --> Call[Invoke downstream agent]
    Call --> Store[Persist response in cache with TTL and tags]
    Store --> Return
```

### Notes
- Use Azure Cache for Redis or a Dataverse table keyed by payload hash and model version to avoid serving stale responses when models change.
- Cache entries store provenance (agent version, timestamp, correlation ID) to support audits.

## Alerting and Observability Signals

```mermaid
flowchart TD
    M[Orchestrator run completes] --> N{Outcome}
    N -- Success --> O[Emit metric: latency, throughput, cache hit rate]
    N -- Transient failure --> P[Retry scheduled]
    P --> Q[Log structured error + correlation ID]
    N -- Fatal failure --> R[Write to dead-letter queue]
    R --> S[Send Teams/Email alert]
    Q --> S
    O --> T[Dashboard/Workbook]
    S --> U[On-call triage SOP]
```

### Notes
- Alerts should include tenant/environment, agent ID, correlation ID, and last error to make triage actionable.
- Dashboards aggregate queue depth, cache hit rate, success/failure counts, and latency percentiles per agent.
- Use Azure Monitor action groups to route alerts to Teams channels and incident systems.
