# System Design: Payment Processing Platform

## Architecture Overview

This document describes the system design for a resilient, high-performance payment processing and card authorization platform built with .NET 8, REST APIs, RabbitMQ (MassTransit), PostgreSQL, and Redis.

---

## High-Level Architecture Diagram

```mermaid
graph TD
    %% Clients
    Merchant[Merchant Client]

    %% API Gateway
    Merchant -->|HTTPS REST| Gateway[API Gateway / YARP]

    %% Microservices
    subgraph Microservices[".NET 8 Microservices"]
        Gateway -->|HTTP REST| Idemp_Svc[Idempotency Service]
        Gateway -->|HTTP REST| Card_Auth[Card Authorization Service]
        Gateway -->|HTTP REST| Pay_Svc[Payment Orchestrator API]

        Card_Auth -.->|MassTransit Event| MQ[[RabbitMQ Message Bus]]
        Pay_Svc -.->|Publish Job| MQ

        MQ -.->|Consume Job| Pay_Worker[Payment Processor Worker]
        MQ -.->|Consume Event| Ledger_Svc[Ledger Service]
        MQ -.->|Consume Webhook Job| Webhook_Svc[Webhook Dispatcher]

        Pay_Worker -.->|Publish Event| MQ
    end

    %% Storage & Caching
    Idemp_Svc -->|Locking| Redis[(Redis Idempotency Store)]
    Card_Auth -->|Fast Balance Lookup| Redis_Balance[(Redis Balance Cache)]
    Pay_Svc -->|EF Core 8| Pay_DB[(Payment DB: PostgreSQL)]
    Pay_Worker -->|EF Core 8| Pay_DB
    Ledger_Svc -->|EF Core 8| Ledger_DB[(Ledger DB: PostgreSQL)]

    %% External Connections
    Pay_Worker -->|HttpClient + Polly| Ext_Stripe[[Stripe API]]
    Pay_Worker -->|HttpClient + Polly| Ext_Adyen[[Adyen API]]
    Pay_Worker -->|HttpClient + Polly| Ext_Bank[[ACH / Direct Bank API]]

    Webhook_Svc -->|HttpClient| Merchant
```

---

## Request Flow: Charge Payment (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    actor Merchant as Merchant System
    participant GW as API Gateway (YARP)
    participant Idemp as Idempotency Service
    participant Pay as Payment Orchestrator
    participant MQ as RabbitMQ (MassTransit)
    participant Worker as Payment Processor Worker
    participant Stripe as Stripe API

    Merchant->>GW: POST /v1/charges (Idempotency-Key header)
    GW->>Idemp: POST /internal/idempotency/check
    Note over Idemp: Check/Set lock in Redis

    alt Key already exists (duplicate request)
        Idemp-->>GW: HTTP 409 Conflict or HTTP 200 (cached response)
        GW-->>Merchant: Return cached response
    else Key is unique & locked
        Idemp-->>GW: HTTP 200 OK (Proceed)
        GW->>Pay: POST /internal/payments/process
        Note over Pay: Insert PENDING transaction in PostgreSQL
        Pay->>MQ: Publish ProcessPaymentJob
        Pay-->>GW: HTTP 202 Accepted (payment_intent_id)
        GW->>Idemp: POST /internal/idempotency/finalize
        Note over Idemp: Cache response payload in Redis
        GW-->>Merchant: HTTP 202 Accepted (status: processing)

        Note over MQ,Worker: Async background processing
        Worker->>MQ: Consume ProcessPaymentJob
        Worker->>Stripe: POST /v1/charges (HttpClient + Polly)
        Stripe-->>Worker: HTTP 200 OK (Charge Captured)
        Worker->>Worker: Update transaction state → Captured
        Worker->>MQ: Publish PaymentCapturedEvent
    end
```

---

## Service Responsibilities

```mermaid
graph LR
    subgraph Gateway Layer
        YARP[API Gateway - YARP]
    end

    subgraph Synchronous Services
        IDS[Idempotency Service<br/>Redis locking & caching]
        CAS[Card Authorization Service<br/>Sub-100ms approval SLA]
        POS[Payment Orchestrator<br/>Gateway routing & state mgmt]
    end

    subgraph Async Workers
        PPW[Payment Processor Worker<br/>External gateway calls]
        LS[Ledger Service<br/>Double-entry bookkeeping]
        WD[Webhook Dispatcher<br/>Merchant notifications]
    end

    YARP --> IDS
    YARP --> CAS
    YARP --> POS
    POS -.-> PPW
    PPW -.-> LS
    PPW -.-> WD
```

---

## Data Storage Layout

```mermaid
graph TD
    subgraph PostgreSQL
        PayDB[(Payment DB)]
        LedgerDB[(Ledger DB)]
    end

    subgraph Redis
        IdempStore[(Idempotency Store<br/>Locks & Cached Responses)]
        BalanceCache[(Balance Cache<br/>Credit Limits & Blocklists)]
    end

    PayDB --- |Transactions, Payment Intents,<br/>Outbox Messages| PayOrch[Payment Orchestrator]
    PayDB --- |Transaction State Updates| PayWorker[Payment Worker]
    LedgerDB --- |Journal Entries,<br/>Ledger Lines, Accounts| LedgerSvc[Ledger Service]
    IdempStore --- |Lock/Unlock, Cache Response| IdempSvc[Idempotency Service]
    BalanceCache --- |Read Balance, Check Blocklist| CardAuth[Card Authorization]
```

---

## Fault Tolerance Strategy

```mermaid
graph TD
    subgraph Resilience Patterns
        Outbox[MassTransit Transactional Outbox<br/>Prevents message loss on DB commit]
        Polly[Polly Circuit Breaker<br/>Retry with exponential backoff]
        DLQ[Dead Letter Queue<br/>Failed messages after retries]
        Idemp[Idempotency Keys<br/>Prevents duplicate processing]
    end

    subgraph Applied To
        Outbox -->|Payment Orchestrator| PG[(PostgreSQL + RabbitMQ)]
        Polly -->|Payment Worker| ExtAPI[External APIs: Stripe, Adyen]
        DLQ -->|All Consumers| RMQ[[RabbitMQ]]
        Idemp -->|API Gateway| RedisStore[(Redis)]
    end
```

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| API Gateway | YARP (.NET) | Reverse proxy, auth, load balancing |
| REST APIs | ASP.NET Core Minimal APIs | Low-latency HTTP endpoints |
| Messaging | RabbitMQ + MassTransit | Async event-driven communication |
| Relational DB | PostgreSQL | Transactions, ledger entries |
| Cache/Locks | Redis (StackExchange.Redis) | Idempotency, balance caching |
| ORM | EF Core 8 + Dapper | Writes (EF Core) / Reads (Dapper) |
| Resilience | Polly | Retry, circuit breaker, timeout |
| Containerization | Docker + Kubernetes (AKS) | Deployment & orchestration |
