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
6. **Sync debit, async settle**: The sender is debited into escrow synchronously (immediate, authoritative feedback + funds reserved), and the recipient is credited by an async workflow (fraud/AML → moratorium → settle). The "instant settlement" requirement is met via a **risk-tiered moratorium** that defaults to ~0 for low-risk transfers (effectively instant) and applies a short hold only to higher-risk ones — see §4.3 and the trade-offs in §7.2.

#### Back-of-the-envelope Capacity Estimate
| Item | Calculation | Result |
|------|-------------|--------|
| Average writes | 200 TPS | 2 ledger rows per transfer → 400 rows/s |
| Peak writes | 1,000 TPS | 2,000 rows/s (burst) |
| Reads (balance/history) | ~10x writes | ~2,000–10,000 RPS |
| **Ledger rows/year** | 200 TPS × 2 rows × 86,400 × 365 | **~12.6 billion rows/year** (~88B over 7 years) |
| Transaction data volume/year | ~12.6B rows × ~1KB | ~13 TB/year (~90 TB over 7 years) |
| Required balance precision | Integer (minor unit = cents) | No floating point |

> **Important**: All monetary amounts are held as `bigint` (minor units such as cents) to eliminate rounding errors.

> **Key sizing decision — polyglot persistence, DynamoDB for money**: ~12.6B ledger rows/year is **too much for PostgreSQL**, so the entire transaction/ledger system and balances live in **DynamoDB**, which scales horizontally for this volume. PostgreSQL holds only **non-money metadata** plus a balance **display cache**:
> - **DynamoDB (source of truth for money)** — the full `ledger_entries` (append-only, 7-year, partitioned by `account_id`, sort key `seq`). **There is no separate balance record: each ledger entry stores the post-transaction `balance_after`, so the current balance is simply the latest entry's `balance_after`.** Money movement is an atomic **`TransactWriteItems`** that appends the balanced entries; double-processing is blocked by the client's **Idempotency-Key** (conditional `attribute_not_exists`).
> - **PostgreSQL (metadata + cache)** — `users`, `user_identifiers`, `transfers` metadata (status, searchable memo), `transfer_events` (workflow state), `idempotency_keys`. It also keeps a **balance display cache** (`balance_cache_minor`) refreshed from **DynamoDB Streams** — *never* authoritative and *never* used for the debit decision.

---

## 2. High-Level Architecture

### 2.1 Component-Based Architecture

![Architecture Overview](./architecture-overview.png)

Requests pass through the edge layer `CloudFront → WAF → API Gateway/ALB` and, after JWT verification, are routed to the appropriate service. The transfer flow, the double-entry ledger, and history queries are all handled inside a single **Payments Service** within strongly consistent transactions. Notifications, recipient resolution, and fraud checks are delegated to independent services, and notifications/history fan-out are made asynchronous via the event bus.

#### Service Decomposition Principle
We do **not** split services along URL paths (e.g., one service per `/transfers`, `/transactions`, `/accounts`). Services are split along **transactional and data-ownership boundaries**, not API surface.

- The transfer flow, balance updates, the double-entry ledger, and transaction history all read and write the **same data** and must participate in the **same ACID transaction** (e.g., debit/credit and the `transfers` status update must be atomic). Splitting them would force distributed transactions / Sagas for the core money-movement path — accidental complexity with no benefit. → Therefore they are consolidated into a single **Payments Service**.
- **Fraud Engine**, **User Directory**, and **Notification** are genuinely independent concerns (different data, different failure modes, different scaling profiles, callable in isolation) → they **remain separate services**.

#### Why This Split (vs Monolith / vs URL-based split)
| Aspect | Decision |
|--------|----------|
| **Single transaction for money movement** | Transfer + ledger + history share data and must be atomic. Keep them in one **Payments Service** to use a single DB transaction instead of a distributed Saga |
| **Fault isolation** | Keep Notification / Fraud separate so their failures never block the money-movement core (graceful degradation) |
| **Independent scaling where it matters** | Read-heavy history is served from read replicas *within* Payments Service; Fraud/Notification scale on their own load profiles |
| **Trade-off** | The Payments Service is larger (it owns reads and writes), but avoids distributed-transaction complexity on the critical path. Read scaling is handled by CQRS read replicas, not by carving out a separate service |

> **Incremental approach**: Start with **Payments Service** as the core, plus the independent **Fraud / User Directory / Notification** services. Split further only when a real transactional/scaling boundary demands it — never merely because a different URL prefix exists.

#### Responsibilities of Key Components
| Component | Responsibility |
|-----------|----------------|
| **API Gateway / ALB** | JWT verification, rate limiting, routing, TLS termination |
| **Payments Service** | The money-movement core. Orchestrates the transfer flow, idempotency control, limit checks, invokes fraud detection; performs strongly consistent balance updates and appends to the double-entry ledger (**the sole owner of balance updates**); serves transaction history via CQRS reads (7-year retention). All money-movement state lives here in one transactional boundary |
| **User Directory Service** | Resolving phone/email → `user_id` (recipient identification) |
| **Notification Service** | Push/SMS/Email notifications (asynchronous, eventually consistent) |
| **Fraud Rule Engine** | Inline rule-based fraud checks. **Owns all integrations with external risk APIs** — calls the ML fraud-scoring API and the AML screening API on behalf of the Payments Service |
| **Event Bus (Kafka/SQS)** | Asynchronous inter-service communication. Event delivery from the outbox |

#### External Integrations
External risk APIs are **not called directly by the Payments Service**; they are accessed through the **Fraud Service**, which acts as the single integration point for risk/compliance. This keeps fraud/AML concerns (vendor SDKs, credentials, fallback policy, false-positive handling) out of the money-movement core.

- **ML fraud scoring API**: Called by the Fraud Service, synchronously only when judged high-risk (falls back on timeout)
- **AML screening API**: Called by the Fraud Service for sanctions-list matching
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

| Entity | Store | Role | Key Points |
|--------|-------|------|------------|
| **ledger** | **DynamoDB** | Full double-entry ledger **and** balance | All entries, append-only, 7-year. Partition key `account_id`, sort key `seq` (per-account sequence); GSI on `transfer_id`. Each entry carries `balance_after`, so **the current balance is the latest entry's `balance_after`** — no separate balance record. ~12.6B rows/yr. The source of truth for **money + history/audit** |
| **users** | PostgreSQL | Users (KYC done) | Synced from upstream. Reference-centric metadata |
| **user_identifiers** | PostgreSQL | Email/phone → userId | Unique index on `value`. Used for recipient resolution |
| **accounts (meta)** | PostgreSQL | Account metadata + balance **cache** | `type` (USER / **ESCROW**), currency, etc. `balance_cache_minor` is a **display cache only** (refreshed from DynamoDB Streams), never authoritative, never used for the debit decision |
| **transfers (meta)** | PostgreSQL | Transfer metadata / search / workflow | Lifecycle `status` (`DEBITED → FRAUD_REVIEW → MORATORIUM → SETTLED`, or `CANCELLED`/`REVERSED`), searchable `note`, `settle_after`. Unique constraint on `idempotency_key`. **Money amounts live in DynamoDB, not here** |
| **transfer_events** | PostgreSQL | Workflow tracking | Each async step (`FRAUD`/`MORATORIUM`/`SETTLE`/`NOTIFY`) and its state — the durable state of the settlement workflow |
| **idempotency_keys** | PostgreSQL | Idempotency control | Request hash + response. (The DynamoDB transaction also carries its own idempotency guard item — see below.) |

#### Concept of the Double-Entry Ledger (with escrow)
Because the recipient credit is **deferred** (async settlement after fraud + moratorium), money never vanishes in between: the synchronous step moves funds from the sender **into a system escrow (clearing) account**, and settlement later moves them from escrow to the recipient. Every phase is a **balanced pair that sums to zero**.

Each phase is one **DynamoDB `TransactWriteItems`** that appends the balanced ledger entries. Each entry records the running `balance_after`, computed from the previous latest entry — so **the balance is carried by the ledger itself**, not a separate item:

```
Phase 1 (SYNC, at POST):   sender → escrow   [one TransactWriteItems]
  read latest entry of sender (balance_after=B_s, seq=N_s) and escrow (B_e, N_e)
  - put sender entry (seq=N_s+1, amount=-1000, balance_after=B_s-1000, phase=DEBIT_TO_ESCROW)
        condition: no item at (sender, N_s+1)         ← optimistic lock on the latest entry
  - put escrow entry (seq=N_e+1, amount=+1000, balance_after=B_e+1000, phase=DEBIT_TO_ESCROW)
        condition: idempotency_key not already recorded ← blocks double-processing

Phase 2 (ASYNC, at settlement):  escrow → recipient   [one TransactWriteItems]
  - put escrow + recipient entries (phase=ESCROW_TO_RECIPIENT), same conditions

  → Σ amount = 0 at every phase (conservation of funds; escrow nets to 0 once settled)
  Cancellation instead writes a REVERSAL TransactWriteItems (escrow → sender).
```

**Where is the truth?** **Money lives in DynamoDB, in the ledger.** Each ledger entry is one transaction and records the post-transaction `balance_after`; the current balance is read directly as the **latest entry's `balance_after`** (query the account partition, highest `seq`) — there is no separate balance record to keep in sync. Correctness comes from two conditions on the append: the `seq=N+1`-not-exists condition is an **optimistic lock** so two concurrent writes can't both chain off the same latest entry (the loser retries against the new latest), and the **Idempotency-Key** condition makes a retried request a no-op (no double debit). **PostgreSQL holds no authoritative money** — its `balance_cache_minor` is a display cache refreshed asynchronously from DynamoDB Streams and is never read for the debit decision. This is a deliberate **polyglot-persistence** split: DynamoDB for the massive, horizontally-scaled ledger; PostgreSQL for rich relational metadata, search, and workflow state.

### 4.2 Key Endpoints (API Contract)

| Method | Path | Description | Idempotency |
|--------|------|-------------|-------------|
| `POST` | `/v1/transfers` | Create and execute a transfer | **Required** (`Idempotency-Key` header) |
| `GET` | `/v1/transfers/{transferId}` | Query transfer status | Naturally idempotent |
| `GET` | `/v1/accounts/me/balance` | Balance inquiry | Naturally idempotent |
| `GET` | `/v1/transfers?cursor=&limit=` | Transaction history (cursor paging) | Naturally idempotent |
| `POST` | `/v1/transfers/{transferId}/cancel` | Cancel during the moratorium window (before SETTLED) — appends a reversal | **Required** |
| `POST` | `/v1/transfers/{transferId}/reverse` | Post-settlement refund (compensating transaction) | **Required** |

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
The synchronous response returns immediately after the sender is debited into escrow; the recipient is credited later by the async workflow.
```json
{
  "transfer_id": "txn_01H...",
  "status": "DEBITED",
  "amount_minor": 1000,
  "settle_after": "2026-05-25T12:01:00Z",
  "created_at": "2026-05-25T12:00:00Z"
}
```

### 4.3 Transfer Flow (Sequence)

![Transfer Sequence](./transfer-sequence.png)

The flow is deliberately split into a **short synchronous path** (so `POST /v1/transfers` returns fast) and an **asynchronous settlement workflow**.

**Synchronous (`POST /v1/transfers` → 200 OK + event):**
1. **Metadata + idempotency**: Write the `transfers` metadata row to PostgreSQL (unique constraint on `idempotency_key`); a duplicate conflicts and the stored response is returned.
2. **Limit check** (e.g. $500/txn, $2,500/day) and **recipient resolution** happen here — cheap to reject early.
3. **Debit only, into escrow — in DynamoDB**: Read the sender's and escrow's latest ledger entries (their `balance_after`, `seq`), then one **`TransactWriteItems`** appends the sender entry (`seq+1`, `balance_after-=amount`) and escrow entry (`seq+1`, `balance_after+=amount`), both `phase=DEBIT_TO_ESCROW`. Conditions: `seq+1` must not already exist (optimistic lock; also fails if `balance_after` would go negative → insufficient funds) and the Idempotency-Key must be unseen. `ConditionalCheckFailed` → retry against the new latest, or dedupe a replay. The recipient is **not** credited yet.
4. Return `200 OK (status=DEBITED)` and publish `TransferRequested` to Kafka.

**Asynchronous settlement workflow (state machine; AWS Step Functions if it grows complex):**
1. **Fraud / AML check**: Suspicious-merchant and AML screening. A small fraction may require a **human approval flow**. On deny → compensate with a `REVERSAL` `TransactWriteItems` (escrow → sender) and set `transfers.status=CANCELLED`.
2. **Moratorium**: Wait until `settle_after` so the user can still cancel the transfer (cancellable window).
3. **Settlement (the actual transfer)**: One `TransactWriteItems` moves escrow → recipient (`phase=ESCROW_TO_RECIPIENT`); update `transfers.status=SETTLED` in PostgreSQL.
4. **Notification**: Notify both the sender and the recipient (Push/SMS/Email).

> **Why this split?** Debiting synchronously gives the sender immediate, authoritative feedback and reserves the funds (the conditional DynamoDB write prevents double-spend), while deferring the credit lets us run fraud/AML and offer a cancellation window without holding the client request open. Escrow keeps the ledger balanced at every step, and `transfer_events` (PostgreSQL) makes the workflow durable and resumable after a crash.

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
| **Fraud / AML** | The Fraud Service performs inline rule checks (velocity, new recipient, anomalous amount) and is the sole caller of the external ML/AML APIs; the Payments Service only calls the Fraud Service |
| **Idempotency & replay prevention** | `Idempotency-Key` + request-hash matching (reject same key with a different body) |
| **Audit** | Record all state changes in an immutable log (who, when, what). WORM via S3 Object Lock |
| **Compliance** | SOC2 (PCI-DSS out of scope since no card data). Separation of duties, access reviews |
| **Secrets** | Secrets Manager + automatic rotation. Never embed secrets in code/images |

> **Synchronous/asynchronous split of fraud detection**: Rule-based (low latency) blocks synchronously. ML scoring is synchronous only for high-risk candidates; otherwise it is event-driven for post-hoc analysis (do not stop legitimate transfers on false positives).

---

## 6. Realizing Non-Functional Requirements (Performance)

### 6.1 Latency (Target p99 ≤ 3 seconds)
- **Minimize the synchronous path**: The synchronous request only does "idempotency check → limit check → recipient resolution → debit into escrow → return 200." Fraud/AML, the moratorium, settlement, and notifications are all moved to the async workflow, keeping the user-facing latency well under 3 seconds.
- **Balance reads**: The authoritative balance is the **latest ledger entry's `balance_after`** — a single query on the account partition for the highest `seq` (low-latency, strongly-consistent when needed). The PostgreSQL `balance_cache_minor` may back fast UI display, but the **debit decision always reads/conditions on DynamoDB**, never the cache.
- **Connection pooling**: Reuse PostgreSQL connections with RDS Proxy / PgBouncer; DynamoDB is accessed over HTTP with the SDK's connection reuse.
- **Hot-account note**: Because each account's writes chain through `seq`, a single very hot account (e.g. a popular merchant) serializes its own writes. That is correct (its true running balance is inherently sequential); to scale such an account, see the sub-account option in §7.2.

### 6.2 Throughput & Scalability (200→1,000 TPS, future 10x)
- **Stateless core services** + HPA for horizontal scaling. State is externalized to DB/Redis/Kafka.
- **Polyglot storage**: All money (the ledger, which carries balance) is in **DynamoDB**, which scales horizontally for the ~12.6B rows/yr write/read volume; balance and history reads go there. PostgreSQL handles only bounded metadata/search/workflow, so the relational tier never carries the huge stream.
- **Native partitioning**: DynamoDB partitions the ledger by `account_id` (sort key `seq`), spreading load across accounts; PostgreSQL metadata is small and can be partitioned/sharded by `account_id` if ever needed (see Tunable Decisions below).
- **Event-driven load leveling**: During bursts, the queue acts as a buffer, protecting downstream (e.g., notifications).

### 6.3 Availability (99.99%)
- **Multi-AZ**: EKS nodes, Aurora, ElastiCache, and MSK all span multiple AZs.
- **Eliminate SPOFs**: Remove dependence on single instances (see deep-dive Q4 below).
- **Graceful degradation**: Even if notifications/history go down, **the transfer itself still succeeds** (events remain in the outbox and are delivered later).
- **Circuit breakers & timeouts**: Cut off + fall back so that latency in external APIs (ML/AML/bank) does not drag down the whole system.

### 6.4 Consistency & Reliability
- **Atomic money movement in DynamoDB**: The balanced ledger entries are appended in one **`TransactWriteItems`** with conditional expressions: the next-`seq`-not-exists condition prevents a lost update (overspend), and the `attribute_not_exists` on the Idempotency-Key blocks double-processing. `balance_after` is computed in the same write. All-or-nothing — no partial debit/credit, no overspend, no double-apply.
- **Cache is asynchronous, never authoritative**: The PostgreSQL `balance_cache_minor` is refreshed from **DynamoDB Streams** (eventually consistent). It is display-only; staleness can never cause a wrong debit because debits read/condition on the DynamoDB ledger.
- **Cross-store metadata**: The `transfers` metadata write (PostgreSQL) and the money write (DynamoDB) are two stores; we write the metadata row (with the idempotency key) first, then the conditional DynamoDB transaction. A crash between them is reconciled by the workflow (a `DEBITED` metadata row with no matching ledger entry is retried or rolled back).
- **Idempotent everywhere**: The Idempotency-Key condition in the DynamoDB transaction and the Kafka consumers' dedupe by `transfer_id` make retries safe.
- **Continuous reconciliation**: A job replays an account's ledger entries and checks that each `balance_after` chains correctly (prev + amount = next) and that escrow nets to the in-flight total; divergence raises an alert.

---

## 7. Trade-offs & Tunable Decisions

This section presents points that can change depending on requirements, along with their trade-offs, making it clear that "there is no single right answer."

### 7.1 Polyglot Persistence: DynamoDB for money, PostgreSQL for metadata
The volume (~12.6B ledger rows/year) rules out PostgreSQL for the ledger, so money (the ledger, which carries the balance) lives in **DynamoDB**, and PostgreSQL keeps only metadata + a display cache:

| Aspect | DynamoDB (money: ledger + balance) | PostgreSQL (metadata + cache) |
|--------|------------------------------------|-------------------------------|
| **Holds** | Full ledger, 7-year, append-only; balance = latest entry's `balance_after` | users, transfers meta, workflow events, **balance display cache** |
| **Transactions** | `TransactWriteItems` (up to 25 items) — atomic balanced entries + conditions | Multi-row ACID for metadata (not money) |
| **Scaling** | Near-unlimited horizontal scale, partitioned by `account_id` | Bounded, small relational data; easy to operate |
| **Consistency** | Strongly-consistent read of latest entry + conditional append for the debit | The cache is eventually consistent (DynamoDB Streams); never authoritative |
| **Why** | Massive write throughput + cheap long-term retention for the ledger | Rich queries: search by memo, joins, workflow state, reporting |

> Trade-off: putting money in DynamoDB means **overspend prevention and ordering must be expressed as conditional appends** (optimistic lock on the next `seq`) rather than SQL `SERIALIZABLE`, and per-account writes are serialized by the `seq` chain. We accept this because the ledger volume is infeasible for PostgreSQL, and the conditional `TransactWriteItems` gives both scale (across accounts) and correctness (within an account). The metadata/cache in PostgreSQL is eventually consistent with DynamoDB (via Streams), reconciled by a continuous job. **Alternatives**: (a) all-PostgreSQL with partitioning + S3 archival — simplest consistency story but the row volume is operationally infeasible; (b) money in PostgreSQL, history in DynamoDB (an earlier iteration) — keeps SQL ACID for balances but still needs cross-store sync and caps write scale at the relational Writer. Putting the whole ledger in DynamoDB is the choice when ledger scale dominates.

### 7.2 Other Tunable Points
| Tunable axis | Option A | Option B | Trade-off |
|--------------|----------|----------|-----------|
| **Transfer confirmation** | Fully synchronous (immediate COMPLETED) | **Sync debit + async settle** (chosen) | Fully sync gives the simplest UX but leaves no room for fraud review / cancellation and couples settlement to request availability. We chose sync-debit-into-escrow + async settlement: the sender gets immediate authoritative feedback, while fraud/AML and a moratorium run off the critical path. Trade-off: the recipient sees funds after settlement, not instantly — acceptable given the moratorium/fraud requirements |
| **Moratorium length** | 0 (settle immediately) | Minutes–hours | Longer window = more cancellation/fraud safety but slower perceived delivery. Tunable per risk tier (e.g. new recipient, large amount) via `settle_after` |
| **Consistency model** | Strong consistency (balance/ledger in DynamoDB) | Eventual consistency (PG balance cache, notifications) | Money uses strongly-consistent conditional writes in DynamoDB. The PostgreSQL display cache and notifications are eventually consistent for availability (CAP trade-off) |
| **Very hot account** | Single ledger per account (chosen) | Split into N **sub-accounts** | Because per-account writes serialize on the `seq` chain, an extremely hot account (a giant merchant) could become a write hotspot. If needed, model it as N sub-accounts (each its own ledger partition) and sum/route across them — adds routing + aggregation complexity, so apply only to the rare hotspot, not by default |
| **Messaging** | Kafka (MSK) | SQS/SNS | Kafka = high throughput / strong reprocessing; SQS = simpler ops. Choose by scale |
| **Fraud detection** | Inline only | Inline + ML | Latency vs detection accuracy. Use sync ML only for high-risk to balance both |
| **Multi-region** | Single region (current, US-only) | Active/passive DR | Cost vs RTO/RPO. Initially single region + regional backups |
| **Payment network** | Closed loop (internal balance only) | RTP/FedNow integration | Internal-only is instant/low-cost; external integration reflects to real bank accounts but adds latency/fees |

---

## 8. Anticipated Q&A / Deep Dives

Centered on the deep-dive questions in Part 3, this section lists points likely to be asked in an interview along with model answers.

### Q1. If the server crashes mid-transfer, how do you recover?
**A.** Each money-moving step (the synchronous debit-to-escrow, and the asynchronous settlement) is a single **DynamoDB `TransactWriteItems`**, which is all-or-nothing — a crash before it commits leaves no partial debit/credit. The settlement workflow is **durable, not in-memory**: its progress lives in `transfers.status` and `transfer_events` (PostgreSQL). A crash mid-workflow is recovered by re-driving from the last persisted state; each step is idempotent — the DynamoDB transaction carries a guard item keyed by `transfer_id`/phase, so re-execution is a no-op (`ConditionalCheckFailed`) rather than a double-apply. Because the recipient is only credited at settlement (not at the synchronous step), a crash after the debit simply leaves funds safely in **escrow** until the workflow resumes. The one cross-store gap — `transfers` metadata in PostgreSQL written just before the DynamoDB money write — is reconciled: a `DEBITED` metadata row with no matching DynamoDB ledger entry is retried or rolled back by a sweeper. If the orchestration grows complex, AWS Step Functions provides this durable execution out of the box.

### Q2. If the same request arrives twice due to a network retry, how do you prevent a double transfer?
**A.** Require a client-generated `Idempotency-Key`. Two layers: (1) the PostgreSQL `transfers` metadata row has a **unique constraint** on `idempotency_key`, so a duplicate create conflicts and returns the prior response; (2) more importantly, the money mutation's **`TransactWriteItems` includes a guard item** keyed by `transfer_id`/phase with an `attribute_not_exists` condition — so even if the same transaction is retried, the conditional write fails (`ConditionalCheckFailed`) and the balance is debited exactly once. Idempotency thus protects the money write atomically inside DynamoDB, not just at the metadata layer. The stored request hash also lets us reject "the same key with a different body" as a conflict.

### Q3. If a balance read and update happen concurrently, how do you ensure consistency?
**A.** The balance is carried by the ledger: each new entry records `balance_after`, computed from the account's current latest entry (`seq=N`, `balance_after=B`). The append writes the next entry at `seq=N+1` with a condition that **no item at `seq=N+1` already exists** (optimistic lock). Two concurrent debits both read `seq=N` and try to write `seq=N+1`; DynamoDB lets exactly one win, the other gets `ConditionalCheckFailed` and retries against the new latest (`seq=N+1`, recomputing `balance_after`). So there is no lost update and no overspend — the read-compute-append is effectively serialized per account, and the check and write are one atomic conditional operation (no TOCTOU). The balance is **never** read from the PostgreSQL cache for this decision. A reconciliation job replays the chain to confirm each `balance_after` is consistent.

### Q4. Where are the SPOFs (single points of failure), and how do you eliminate them?
**A.**
- **Money store**: DynamoDB is multi-AZ and regionally replicated by design (no single node to lose); the authoritative balance/ledger has no single point of failure.
- **Metadata DB**: Aurora Multi-AZ (automatic failover) + read replicas.
- **Cache**: ElastiCache cluster mode (shards + replicas); also, the PostgreSQL balance cache is non-authoritative, so its staleness/failure never blocks a debit (which reads DynamoDB).
- **Messaging**: MSK across multiple brokers/AZs.
- **Compute**: EKS Multi-AZ, AZ distribution via PodAntiAffinity, redundancy via HPA.
- **Edge**: DNS failover via CloudFront/Route 53.
- **External APIs**: Circuit breaker + timeout + fallback (e.g., if ML is down, degrade to rule-based judgment).

### Q5. If transaction volume increases 10x (~10,000 TPS), where is the bottleneck and how do you scale?
**A.** The ledger is in **DynamoDB**, partitioned by `account_id`, so aggregate throughput scales horizontally near-linearly — the money store is not the first wall. The pressure points and countermeasures, incrementally:
1. **A single very hot account** is the main risk — its writes serialize on the per-account `seq` chain (correct, but a hotspot). If one account dominates, model it as N **sub-accounts** (each its own ledger partition) and route/aggregate across them (see §7.2). Most accounts never need this.
2. Keep the synchronous path minimal (one `TransactWriteItems` debit-to-escrow); push fraud/moratorium/settlement/notify to the async workflow.
3. Use **DynamoDB on-demand or auto-scaling** so capacity tracks the write stream; the `account_id` partition key spreads load and avoids hot partitions in aggregate.
4. The **PostgreSQL metadata** tier is bounded and small relative to the ledger; scale it with read replicas, and partition by `account_id` only if needed. The balance cache is refreshed by DynamoDB Streams consumers.
5. Messaging scales by adding Kafka partitions; the workflow scales as stateless workers driven by the durable state.
The next bottleneck, connection count to PostgreSQL, is absorbed by RDS Proxy.

### Q6. How did you decide service boundaries? Why not split by URL path, and why not a full monolith?
**A.** Boundaries follow **transactional and data-ownership lines, not URL paths**. The transfer flow, balance/ledger mutations (DynamoDB), and transfer metadata (PostgreSQL) are all part of one money-movement unit and must be coordinated atomically, so splitting them by URL (`/transfers`, `/transactions`, `/accounts`) would force a distributed transaction / Saga on the critical path for no benefit. We therefore keep them in a single **Payments Service** that owns both stores. Conversely, **Fraud**, **User Directory**, and **Notification** are genuinely independent (separate data, separate failure modes, separate scaling) and stay as their own services so their failures don't block money movement. This is neither a full monolith (independent concerns are isolated) nor an over-decomposed mesh (the atomic core stays together).

### Q7. How do you represent monetary amounts? What about floating point?
**A.** Do not use it. Hold amounts as `bigint` minor units (cents) to eliminate rounding and representation errors. The currency is fixed to USD (per requirements), but a currency code is retained to prepare for future multi-currency.

### Q8. How do you handle cancellation and reversals/refunds?
**A.** Two cases, both append-only (existing `ledger_entries` are never mutated):
- **Cancellation during the moratorium** (before `SETTLED`): the funds are still in escrow, so we append a `REVERSAL` pair (escrow → sender) and set `status=CANCELLED`. The recipient was never credited, so nothing to claw back.
- **Post-settlement refund**: append a compensating `REVERSAL` pair (recipient → sender, via escrow) and set `status=REVERSED`.

In both cases the immutable ledger preserves a complete audit trail, and `transfer_events` records who/what triggered the change.

---

