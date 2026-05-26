# System Design Interview Prompt

## Problem Statement

> **"Design a P2P money transfer system."**

---

## Part 1: Questions You Should Ask

### Scope & Functional Requirements

```
- Are we designing a system similar to Zelle or Venmo: real-time bank-to-bank transfers identified by phone number or email, rather than card-based payments like Square?

- Are we focusing on core transfer features — sending, receiving,
  balance checks, and transaction history —
  and excluding account creation or investment features?

- Are we assuming single-currency (USD) support only,
  without multi-currency or FX conversion?

- Should we enforce transfer limits such as
  $500 per transaction and $2,500 per day?

- Should the recipient be identified by phone number or email address,
  rather than a bank account number directly?

- Are we targeting instant settlement where the recipient sees
  the funds immediately, rather than next-day ACH-style settlement?
```

### Non-Functional Requirements & Scale

```
- Are we designing for a user base of around
  50 million registered users and 5 million DAU?

- Should we target an average of ~200 TPS
  with burst capacity up to ~1,000 TPS?

- Is a 3-second end-to-end latency the target for transfer completion?

- Are we targeting 99.99% availability (~50 minutes downtime/year)?

- Should we retain transaction history for 7 years
  to satisfy regulatory requirements?

- Are we scoping this to US-only for the initial launch?
```

### Security & Compliance

```
- Can we assume users are already KYC-verified at registration,
  and that the auth service is out of scope for this design?

- Should we include basic rule-based fraud checks inline,
  with ML-based scoring treated as an external service call?

- Can we treat AML screening as an external API call
  rather than designing it internally?

- Since we are not handling card data, can we assume
  PCI-DSS is out of scope, with SOC2 as the primary compliance target?
```

### System & Infrastructure Assumptions

```
- Are we assuming AWS as the cloud provider?

- Can we treat authentication and account management
  as existing upstream services and focus on the transfer layer?

- Should we focus on backend design and API contracts,
  and leave frontend/mobile out of scope?

- Is the choice between microservices and monolith
  open for us to propose and justify?
```

---

## Part 2: Expected Answers from Interviewer

### Scope & Functional Requirements

```
Q: Core features limited to send, receive, balance, history?
A: Correct. Account creation and investment features are out of scope.

Q: Single currency (USD) only?
A: Yes, no multi-currency or FX conversion for now.

Q: Transfer limits of $500/transaction and $2,500/day?
A: Yes, that is correct.

Q: Recipient identified by phone number or email?
A: Yes, no need to enter bank account numbers directly.

Q: Instant settlement rather than ACH-style?
A: Yes, recipient should see funds reflected immediately.
```

### Non-Functional Requirements & Scale

```
Q: ~50M registered users, 5M DAU?
A: Yes, that is the scale to design for.

Q: Average 200 TPS, peak 1,000 TPS?
A: Correct.

Q: 3-second end-to-end latency target?
A: Yes, that is the requirement.

Q: 99.99% availability?
A: Correct (~50 minutes downtime per year).

Q: 7-year transaction history retention?
A: Yes, required for regulatory compliance.

Q: US-only for initial launch?
A: Yes, global expansion is out of scope for now.
```

### Security & Compliance

```
Q: KYC handled upstream, auth service out of scope?
A: Correct, assume users are already verified.

Q: Rule-based fraud checks inline, ML scoring as external call?
A: Yes, that separation is fine.

Q: AML screening as external API call?
A: Correct, no need to design it internally.

Q: PCI-DSS out of scope, SOC2 as primary compliance target?
A: Yes, since no card data is involved.
```

### System & Infrastructure Assumptions

```
Q: AWS as cloud provider?
A: Yes, assume AWS.

Q: Auth and account management as existing upstream services?
A: Correct, focus on the transfer layer.

Q: Backend and API contracts only, frontend out of scope?
A: Yes, focus on the backend.

Q: Microservices vs. monolith open for proposal?
A: Yes, propose and justify your choice.
```

---

## Part 3: Interviewer Follow-up Deep Dives

```
1. If the server crashes mid-transfer, how do you recover?

2. If the same request arrives twice due to a network retry,
   how do you prevent a double transfer?

3. If a balance read and update happen concurrently,
   how do you ensure consistency?

4. Where are the SPOFs (Single Points of Failure) in your design,
   and how do you eliminate them?

5. If transaction volume increases 10x,
   where is the bottleneck and how do you scale?
```

---

## Part 4: Evaluation Criteria

```
✅ Requirements gathering
   Do you propose concrete assumptions rather than asking open-ended questions?
   Can you quickly narrow scope and confirm trade-offs?

✅ Core design decisions
   Can you clearly address idempotency, double-spend prevention,
   and data consistency?

✅ Trade-off articulation
   e.g. Immediate consistency vs. availability
       Synchronous vs. asynchronous transfer confirmation

✅ Incremental design approach
   Start simple, identify weaknesses, and iteratively improve.

✅ Finance domain awareness
   Idempotency keys, compensating transactions, immutable audit logs.
```

---

# System Design Document: P2P Money Transfer System

## 1. Executive Summary

### 1.1 System Overview
This system is a real-time peer-to-peer (P2P) money transfer system similar to Zelle/Venmo. Users can instantly send money to other users simply by specifying a phone number or email address.

### 1.2 Functional Requirements
| Requirement | Details |
|-------------|---------|
| Core features | Send, receive, balance check, transaction history |
| Identification | Phone number or email address |
| Currency | USD single currency only |
| Transfer limits | $500 per transaction, $2,500 per day |
| Settlement | Instant settlement (not ACH-style) |
| Out of scope | Account creation, investment features, multi-currency |

### 1.3 Non-Functional Requirements
| Requirement | Target | Notes |
|-------------|--------|-------|
| User scale | 50M registered users, 5M DAU | |
| Throughput | Average 200 TPS, peak 1,000 TPS | |
| Latency | Within 3 seconds end-to-end | Until transfer completion |
| Availability | 99.99% | ~50 minutes downtime/year |
| Data retention | Retain transaction history for 7 years | Regulatory requirement |
| Region | US only (initial launch) | |

### 1.4 Security & Compliance
- **KYC (identity verification)**: Performed by upstream service (out of scope)
- **Authentication**: Uses existing auth service (out of scope)
- **Fraud detection**: Rule-based fraud checks implemented inline; ML-based scoring as external API
- **AML (anti-money laundering)**: External API call
- **Compliance**: SOC2 compliant (PCI-DSS out of scope)

### 1.5 Technology Stack Assumptions
- **Cloud provider**: AWS
- **Architecture**: Microservices architecture (rationale below)
- **Scope**: Focus on backend and API contracts (frontend/mobile out of scope)

### 1.6 Design Principles
This document progressively elaborates in the order "overview → design principles → details." Because this is a finance domain, the following are treated as top-priority design principles.

1. **Correctness first**: Money must neither be lost nor created. Balances must always be consistent, and double-spending must be structurally prevented.
2. **Idempotency**: Reliably prevent double transfers caused by network retries using an Idempotency-Key.
3. **Immutable Audit Log**: The ledger is append-only. Balance is the aggregated result of ledger entries and is never overwritten.
4. **Tiered consistency**: Balances/ledger use **strong consistency (synchronous)**; notifications/history reflection use **eventual consistency (asynchronous)**.
5. **Failure-oriented design**: Assume every component can fail; eliminate SPOFs and add redundancy across Multi-AZ.

#### Back-of-the-envelope Capacity Estimate
| Item | Calculation | Result |
|------|-------------|--------|
| Average writes | 200 TPS | 2 ledger rows per transfer → 400 rows/s |
| Peak writes | 1,000 TPS | 2,000 rows/s (burst) |
| Reads (balance/history) | ~10x writes | ~2,000–10,000 RPS (assuming cache) |
| Transaction data volume/year | 200 TPS × 86,400 × 365 × ~1KB | ~6.3 TB/year (~44 TB over 7 years) |
| Required balance precision | Integer (minor unit = cents) | No floating point |

> **Important**: All monetary amounts are held as `bigint` (minor units such as cents) to eliminate rounding errors.

---

## 2. High-Level Architecture

### 2.1 Component-Based Architecture

![Architecture Overview](./architecture-overview.png)

Requests pass through the edge layer `CloudFront → WAF → API Gateway/ALB` and, after JWT verification, are routed to the core services. The transfer path is orchestrated by the **Transfer Service**, while balance updates are handled by the **Account/Ledger Service** in strongly consistent transactions. Notifications and history reflection are made asynchronous via the event bus.

#### Why Microservices (vs Monolith)
| Aspect | Decision |
|--------|----------|
| **Independent scaling** | Transfers (writes) and history queries (reads) have very different load profiles. We want to scale them separately |
| **Fault isolation** | Prevent failures in the notification service from cascading into the transfer core |
| **Compliance boundary** | Clearly isolate the ledger service to centralize auditing and access control |
| **Trade-off** | Increased complexity of distributed transactions → addressed by the Saga/eventual consistency described later. Keep the number of services small initially |

> **Incremental approach**: The minimal configuration is two services, "Transfer + Account/Ledger." History / Notification / Fraud are carved out as demand requires. Avoid over-decomposition.

#### Responsibilities of Key Components
| Component | Responsibility |
|-----------|----------------|
| **API Gateway / ALB** | JWT verification, rate limiting, routing, TLS termination |
| **Transfer Service** | Orchestration of the transfer flow, idempotency control, limit checks, invoking fraud detection |
| **Account/Ledger Service** | Strongly consistent balance updates, appending to the double-entry ledger. **The sole owner of balance updates** |
| **User Directory Service** | Resolving phone/email → `user_id` (recipient identification) |
| **Transaction History Service** | The read side of CQRS. Optimized for history queries. 7-year retention |
| **Notification Service** | Push/SMS/Email notifications (asynchronous, eventually consistent) |
| **Fraud Rule Engine** | Inline rule-based fraud checks |
| **Event Bus (Kafka/SQS)** | Asynchronous inter-service communication. Event delivery from the outbox |

#### External Integrations
- **ML fraud scoring API**: Synchronous call only when judged high-risk (falls back on timeout)
- **AML screening API**: Sanctions-list matching
- **Bank/payment network (RTP / FedNow)**: Instant gross settlement. Externalized in this design as the "final network for moving funds"

### 2.2 Kubernetes (EKS)-Based Deployment

![Kubernetes Architecture](./architecture-k8s.png)

Core services are deployed on **Amazon EKS** on AWS. Data stores (Aurora / ElastiCache / MSK) are placed outside the cluster as managed services to reduce operational burden and blast radius.

| Item | Choice | Rationale |
|------|--------|-----------|
| **Orchestration** | Amazon EKS (Multi-AZ) | Automatic recovery from node/AZ failures |
| **Autoscaling** | HPA (CPU/custom metric TPS) + Cluster Autoscaler | Keep up with the 1,000 TPS peak |
| **Service mesh** | Istio / App Mesh | mTLS, retries, circuit breaking, observability |
| **Secret management** | External Secrets Operator → Secrets Manager | Separate DB credentials from code |
| **Deployment strategy** | Rolling / canary (Argo Rollouts) | Zero-downtime updates (99.99% target) |
| **Pod placement** | PodAntiAffinity + topologySpreadConstraints | Avoid single-AZ concentration; eliminate SPOFs |

---

## 3. Cross-Cutting Concerns

| Area | Technology | Details |
|------|-----------|---------|
| **AuthN / AuthZ** | Upstream auth service (JWT) + verification at API Gateway | Assumes KYC done. Inter-service is **mTLS**. Resource access authorized via JWT claims (sub=userId) |
| **Metrics monitoring** | Prometheus + Grafana / CloudWatch | TPS, latency (p50/p95/p99), error rate, balance-update failure rate |
| **Distributed tracing** | OpenTelemetry + AWS X-Ray | Propagate `traceId` to all services. Identify bottlenecks in the transfer flow |
| **Logging** | Fluent Bit → OpenSearch / S3 | Structured logs (JSON). PII is masked. Audit logs kept on a separate, tamper-resistant path |
| **Alerting** | Alertmanager / PagerDuty | SLO violations (availability/latency), balance-inconsistency detection, fraud spikes |
| **Audit log** | Append-only store + S3 Object Lock (WORM) | 7-year retention. SOC2 requirement. Tamper-proof |
| **Configuration management** | AWS AppConfig / ConfigMap | Change limits and flags without downtime |
| **Secrets** | AWS Secrets Manager + automatic rotation | DB/external API credentials |
| **Rate limiting** | API Gateway + Redis (token bucket) | Per-user and per-IP |

#### Key SLIs/SLOs
| SLI | SLO |
|-----|-----|
| Transfer success rate | ≥ 99.95% |
| Transfer latency p99 | ≤ 3 seconds |
| Availability | 99.99% (~50 minutes/year) |
| Balance consistency | 100% (verified by daily reconciliation) |

---

## 4. Application Design

### 4.1 Data Model (Entity List)

![Database Schema](./database-schema.png)

| Entity | Role | Key Points |
|--------|------|------------|
| **users** | Users (KYC done) | Synced from upstream. This service is reference-centric |
| **user_identifiers** | Email/phone → userId | Unique index on `value`. Used for recipient resolution |
| **accounts** | Holds balance | `balance_minor`(bigint) + `version`(optimistic lock). Balance is reconciled daily against the aggregate of ledger entries |
| **transfers** | Transfer transaction | State transitions managed by `status`. Unique constraint on `idempotency_key` |
| **ledger_entries** | Double-entry ledger | **Append-only / immutable**. Two rows (debit/credit) per transfer. Holds `balance_after` for auditing |
| **idempotency_keys** | Idempotency control | Stores request hash and response. Has a TTL |

#### Concept of the Double-Entry Ledger
A transfer `A → B ($10)` is always recorded as **two entries that sum to zero**.

```
ledger_entries:
  (transfer_id=T1, account=A, amount_minor=-1000, balance_after=...)  -- debit
  (transfer_id=T1, account=B, amount_minor=+1000, balance_after=...)  -- credit
  → Σ amount = 0 (conservation of funds)
```

The balance (`accounts.balance_minor`) is a **materialized view** for performance; the source of truth is the aggregation of `ledger_entries`. This makes auditing and reconciliation possible.

### 4.2 Key Endpoints (API Contract)

| Method | Path | Description | Idempotency |
|--------|------|-------------|-------------|
| `POST` | `/v1/transfers` | Create and execute a transfer | **Required** (`Idempotency-Key` header) |
| `GET` | `/v1/transfers/{transferId}` | Query transfer status | Naturally idempotent |
| `GET` | `/v1/accounts/me/balance` | Balance inquiry | Naturally idempotent |
| `GET` | `/v1/transfers?cursor=&limit=` | Transaction history (cursor paging) | Naturally idempotent |
| `POST` | `/v1/transfers/{transferId}/reverse` | Refund/reversal (compensating transaction) | **Required** |

#### Example `POST /v1/transfers` Request
```http
POST /v1/transfers
Authorization: Bearer <JWT>
Idempotency-Key: 5f3c...e9   # client-generated UUID

{
  "recipient": { "type": "EMAIL", "value": "bob@example.com" },
  "amount_minor": 1000,        // $10.00
  "currency": "USD",
  "note": "lunch"
}
```
#### Example Response
```json
{
  "transfer_id": "txn_01H...",
  "status": "COMPLETED",
  "amount_minor": 1000,
  "created_at": "2026-05-25T12:00:00Z"
}
```

### 4.3 Transfer Flow (Sequence)

![Transfer Sequence](./transfer-sequence.png)

The four key points are as follows.
1. **Idempotency check first**: Lock with `SETNX idem:{key}`. Duplicates return the previous response.
2. **Limit and fraud checks before the ledger update**: Avoid wasted writes.
3. **Balance update in a single transaction** (`SERIALIZABLE`): Atomically run balance check, debit, and credit.
4. **Notifications and history reflection are asynchronous**: Keep the synchronous path short to meet the latency target (3 seconds).

---

## 5. Security

| Aspect | Measure |
|--------|---------|
| **Authentication** | Verify upstream-issued JWT at the API Gateway. Short expiry + refresh. Inter-service is mTLS |
| **Authorization** | Enforce that the JWT `sub`(userId) matches the owner of the target account. Least-privilege IAM |
| **Encryption in transit** | TLS 1.2+ on all paths. mTLS within the cluster too |
| **Encryption at rest** | Aurora/Redis/S3 encrypted with KMS. Consider app-layer encryption for PII fields as well |
| **PII protection** | Mask phone/email/amount in logs and traces. Minimal retention |
| **Input validation** | Schema validation, amount upper-bound/positive-value checks, parameterized SQL (ORM to prevent injection) |
| **Rate limiting / WAF** | Per-user and per-IP token bucket. WAF blocks L7 attacks and bots |
| **Fraud / AML** | Inline rule checks (velocity, new recipient, anomalous amount) + external ML/AML calls |
| **Idempotency & replay prevention** | `Idempotency-Key` + request-hash matching (reject same key with a different body) |
| **Audit** | Record all state changes in an immutable log (who, when, what). WORM via S3 Object Lock |
| **Compliance** | SOC2 (PCI-DSS out of scope since no card data). Separation of duties, access reviews |
| **Secrets** | Secrets Manager + automatic rotation. Never embed secrets in code/images |

> **Synchronous/asynchronous split of fraud detection**: Rule-based (low latency) blocks synchronously. ML scoring is synchronous only for high-risk candidates; otherwise it is event-driven for post-hoc analysis (do not stop legitimate transfers on false positives).

---

## 6. Realizing Non-Functional Requirements (Performance)

### 6.1 Latency (Target p99 ≤ 3 seconds)
- **Minimize the synchronous path**: Make notifications, history reflection, and ML analysis asynchronous, narrowing the critical path to "idempotency check → limit → rules → ledger update."
- **Balance cache**: Balance inquiries use Redis (write-through). Update the cache on writes to maintain consistency.
- **Connection pooling**: Reuse DB connections with RDS Proxy / PgBouncer to avoid connection exhaustion and handshake latency.
- **Hot-account mitigation**: To curb row-lock contention, consider the sharding/bucketed-ledger approach described later.

### 6.2 Throughput & Scalability (200→1,000 TPS, future 10x)
- **Stateless core services** + HPA for horizontal scaling. State is externalized to DB/Redis/Kafka.
- **Read/write separation (CQRS)**: Writes go to the Aurora Writer; reads (history/balance) go to Reader replicas + a dedicated History Store.
- **Database horizontal partitioning**: A design where `accounts` / `ledger_entries` can be sharded by `account_id` (see Tunable Decisions below).
- **Event-driven load leveling**: During bursts, the queue acts as a buffer, protecting downstream (e.g., notifications).

### 6.3 Availability (99.99%)
- **Multi-AZ**: EKS nodes, Aurora, ElastiCache, and MSK all span multiple AZs.
- **Eliminate SPOFs**: Remove dependence on single instances (see deep-dive Q4 below).
- **Graceful degradation**: Even if notifications/history go down, **the transfer itself still succeeds** (events remain in the outbox and are delivered later).
- **Circuit breakers & timeouts**: Cut off + fall back so that latency in external APIs (ML/AML/bank) does not drag down the whole system.

### 6.4 Consistency & Reliability
- **Strongly consistent ledger updates**: Balance updates happen in a single DB transaction (`SERIALIZABLE`, or row locks + optimistic version).
- **Transactional Outbox + CDC**: Guarantee "DB update" and "event publication" in the same transaction to prevent double publication / loss (at-least-once + idempotent consumer).
- **Idempotent consumer**: Notifications and history deduplicate by `transfer_id`.

---

## 7. Trade-offs & Tunable Decisions

This section presents points that can change depending on requirements, along with their trade-offs, making it clear that "there is no single right answer."

### 7.1 Changing from RDB (Aurora PostgreSQL) to DynamoDB
| Aspect | RDB (current) | DynamoDB |
|--------|---------------|----------|
| **Transactions** | Multi-row ACID, easy `SERIALIZABLE` | Constrained by `TransactWriteItems` (max 100 items). Double-entry needs care |
| **Scaling** | Handled by vertical + read replicas + sharding | Nearly unlimited horizontal scaling, easy to operate |
| **Hot partitions** | Hot row-lock contention | Key design is crucial to distribute hot partitions |
| **Consistency** | Strong consistency is natural | Achievable via conditional writes (optimistic locking). Poor fit for aggregate queries |
| **Conclusion** | **First choice for a financial ledger's strong consistency and complex queries** | Favorable for ultra-large scale and simple access patterns. For the ledger, an alternative design with conditional writes + idempotency |

> Trade-off: DynamoDB is strong in availability and scale, but complexity increases for the atomicity of double-entry and the aggregate queries used for reconciliation. For this use case (a strongly consistent ledger) we recommend RDB, while leaving open the option of a hybrid "account partitioning + DynamoDB" at ultra-large scale.

### 7.2 Other Tunable Points
| Tunable axis | Option A | Option B | Trade-off |
|--------------|----------|----------|-----------|
| **Transfer confirmation** | Synchronous (immediate COMPLETED) | Asynchronous (PENDING→Webhook) | Sync = good UX / sensitive to availability; async = high availability / UX needs work. The requirement is "instant," so sync is the baseline |
| **Consistency model** | Strong consistency (ledger) | Eventual consistency | Balance requires strong consistency. Notifications/history take eventual consistency to gain availability (CAP trade-off) |
| **Messaging** | Kafka (MSK) | SQS/SNS | Kafka = high throughput / strong reprocessing; SQS = simpler ops. Choose by scale |
| **Fraud detection** | Inline only | Inline + ML | Latency vs detection accuracy. Use sync ML only for high-risk to balance both |
| **Multi-region** | Single region (current, US-only) | Active/passive DR | Cost vs RTO/RPO. Initially single region + regional backups |
| **Payment network** | Closed loop (internal balance only) | RTP/FedNow integration | Internal-only is instant/low-cost; external integration reflects to real bank accounts but adds latency/fees |

---

## 8. Anticipated Q&A / Deep Dives

Centered on the deep-dive questions in Part 3, this section lists points likely to be asked in an interview along with model answers.

### Q1. If the server crashes mid-transfer, how do you recover?
**A.** Because balance updates are done atomically in a single DB transaction, a crash before COMMIT is **automatically rolled back**, so no partial debit occurs. Furthermore, state is managed via `transfers.status` (PENDING→COMPLETED/FAILED), and a recovery job on startup detects "transfers left in PENDING" and finalizes them—completing or compensating (reversal) based on whether ledger entries exist. Event publication is tied to the commit in the same transaction via the Transactional Outbox, which also prevents loss.

### Q2. If the same request arrives twice due to a network retry, how do you prevent a double transfer?
**A.** Require a client-generated `Idempotency-Key`. The Transfer Service first attempts a lock with `SETNX idem:{key}`, and if it already exists, returns **the same response as before** (no new transfer is made). At the persistence layer too, a unique constraint is placed on `transfers.idempotency_key` as the last line of defense in case of a Redis failure. In addition, the request hash is stored so that "the same key with a different body" is rejected as a conflict.

### Q3. If a balance read and update happen concurrently, how do you ensure consistency?
**A.** Use two approaches together. (1) Balance updates use `SELECT ... FOR UPDATE` (pessimistic lock) within a single transaction, or **optimistic locking** via a `version` column (CAS-style update with retry on conflict). (2) Since balance is an aggregation of the double-entry ledger and `ledger_entries` is the source of truth, even if `accounts.balance_minor` drifts it can be detected and corrected by daily reconciliation. The key is not to split the balance check and debit into separate transactions (avoiding TOCTOU).

### Q4. Where are the SPOFs (single points of failure), and how do you eliminate them?
**A.**
- **DB**: Aurora Multi-AZ (automatic failover) + read replicas.
- **Cache**: ElastiCache cluster mode (shards + replicas). On Redis failure, fall back to reading directly from the DB.
- **Messaging**: MSK across multiple brokers/AZs.
- **Compute**: EKS Multi-AZ, AZ distribution via PodAntiAffinity, redundancy via HPA.
- **Edge**: DNS failover via CloudFront/Route 53.
- **External APIs**: Circuit breaker + timeout + fallback (e.g., if ML is down, degrade to rule-based judgment).

### Q5. If transaction volume increases 10x (~10,000 TPS), where is the bottleneck and how do you scale?
**A.** The first limit is the **write DB (especially row-lock contention on hot accounts)**. Countermeasures, applied incrementally:
1. Thoroughly separate reads and writes (offload history/balance queries to replicas and the History Store).
2. **Shard** writes horizontally based on `account_id`.
3. For hot accounts (e.g., merchants), **bucket the ledger entries** (post across multiple sub-balances and compute the total) to relax lock contention.
4. Trim synchronous processing further, moving to Outbox + async.
5. If needed, migrate the ledger storage to "account partitioning + DynamoDB (conditional writes)" (see the trade-off in 7.1).
The next bottleneck, connection count, is absorbed by RDS Proxy; messaging is handled by adding partitions.

### Q6. Why microservices? Is a monolith not acceptable?
**A.** Start small initially with about two services (Transfer + Account/Ledger), and carve out areas (History/Notification/Fraud) as the need arises for load characteristics, fault isolation, and compliance boundaries. The main motivations for splitting are that reads (history) and writes (transfers) have very different scaling characteristics, and the desire to isolate notification failures from the transfer core. Avoid excessive splitting, as it invites the complexity of distributed transactions.

### Q7. How do you represent monetary amounts? What about floating point?
**A.** Do not use it. Hold amounts as `bigint` minor units (cents) to eliminate rounding and representation errors. The currency is fixed to USD (per requirements), but a currency code is retained to prepare for future multi-currency.

### Q8. How do you handle reversals/refunds of a transfer?
**A.** Since existing `ledger_entries` are immutable, do not overwrite them; instead append new reverse entries as a **compensating transaction (reversal)** (`status=REVERSED`). This keeps the audit trail fully intact.

---

