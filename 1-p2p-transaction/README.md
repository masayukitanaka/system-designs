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

# System Design Solution

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [High-Level Architecture](#high-level-architecture)
3. [Core Design Decisions](#core-design-decisions)
4. [Database Design](#database-design)
5. [API Design](#api-design)
6. [Key Transaction Flow](#key-transaction-flow)
7. [Scalability & Performance](#scalability--performance)
8. [Security & Compliance](#security--compliance)
9. [Failure Handling & Recovery](#failure-handling--recovery)
10. [Trade-offs & Alternatives](#trade-offs--alternatives)

---

## Executive Summary

### Design Philosophy
This P2P money transfer system prioritizes **strong consistency** and **correctness** over eventual consistency, as financial transactions require ACID guarantees. The design uses a **microservices architecture** with PostgreSQL as the primary datastore, ensuring data integrity through database transactions.

### Key Metrics
- **Scale**: 50M registered users, 5M DAU
- **Throughput**: 200 TPS average, 1,000 TPS peak
- **Latency**: < 3 seconds end-to-end
- **Availability**: 99.99% (52 minutes downtime/year)
- **Data Retention**: 7 years for compliance

### Core Principles
1. **Idempotency**: Every transfer request includes an idempotency key to prevent double-spend
2. **Atomicity**: All balance updates happen within database transactions
3. **Auditability**: Immutable transaction logs for compliance
4. **Scalability**: Horizontal scaling through read replicas and caching
5. **Resilience**: Circuit breakers, retries, and compensating transactions

---

## High-Level Architecture

![Architecture Overview](./architecture-overview.png)

### Architecture Layers

#### 1. Client Layer
- Mobile apps (iOS/Android)
- Web application
- All communication via HTTPS with TLS 1.3

#### 2. API Gateway
- **Authentication**: JWT token validation
- **Rate Limiting**: Per-user and per-endpoint limits
- **Request Routing**: Routes to appropriate microservices
- **API Versioning**: Support for backward compatibility
- **Technology**: AWS API Gateway or Kong

#### 3. Core Services (Microservices)

**Transfer Service**
- Orchestrates money transfers
- Implements idempotency checking
- Coordinates with other services
- Primary writer to transactions table

**Balance Service**
- Manages account balances
- Implements optimistic locking
- Handles concurrent balance reads
- Caches frequently accessed balances

**Transaction History Service**
- Provides transaction queries
- Reads from read replicas
- Implements pagination
- Archives old transactions

**Fraud Detection Service**
- Rule-based fraud checks (inline)
- Calls external ML scoring service
- Implements velocity checks
- Blocks suspicious transactions

**Notification Service**
- Sends push notifications
- Email confirmations
- SMS alerts for large transfers
- Consumes events from message queue

#### 4. Data Layer

**PostgreSQL (Primary)**
- ACID-compliant relational database
- Stores users, accounts, transactions
- Multi-AZ deployment for HA
- Write-ahead logging for durability

**Redis Cache**
- Caches user balances
- Stores idempotency keys (TTL: 24 hours)
- Rate limiting counters
- Session data

**Message Queue (Kafka/SQS)**
- Asynchronous event processing
- Transaction completed events
- Notification delivery
- Audit log streaming

**Object Storage (S3)**
- Long-term archive storage
- Compliance documents
- Transaction receipts
- Encrypted at rest

**Archive Database**
- Cold storage for transactions > 2 years
- Columnar format for analytics
- 7-year retention for compliance

#### 5. External Services
- **Bank APIs**: ACH/wire transfer integration
- **AML Service**: Anti-money laundering checks
- **ML Fraud Scoring**: Advanced fraud detection
- **SMS/Email Providers**: Twilio, SendGrid

---

## Core Design Decisions

### 1. Microservices vs. Monolith

**Choice**: Microservices

**Rationale**:
- **Team Autonomy**: Different teams can own transfer, fraud, notifications
- **Independent Scaling**: Scale fraud detection independently from transfers
- **Technology Diversity**: Use different tech stacks where appropriate
- **Failure Isolation**: Notification failures don't block transfers

**Trade-off**: Increased operational complexity vs. simpler monolith deployment

### 2. Synchronous vs. Asynchronous Processing

**Choice**: Hybrid approach

**Synchronous** (in critical path):
- Balance validation
- Fraud rule checks
- Database transaction commit

**Asynchronous** (out of critical path):
- Notifications
- ML fraud scoring (post-transaction)
- Analytics events
- Audit log archiving

**Rationale**: Keep critical path fast (<3s) while offloading non-critical work

### 3. Database Choice: PostgreSQL

**Why PostgreSQL**:
- **ACID Transactions**: Critical for financial consistency
- **Row-Level Locking**: `SELECT FOR UPDATE` for balance checks
- **JSONB Support**: Flexible fraud check data storage
- **Mature Ecosystem**: Well-understood operations and tooling
- **Read Replicas**: Easy horizontal scaling for reads

**Alternatives Considered**:
- **MySQL**: Similar benefits, chose PostgreSQL for JSONB and better concurrency
- **NoSQL (DynamoDB)**: Rejected due to lack of multi-row transactions
- **NewSQL (CockroachDB)**: Over-engineered for initial scale

### 4. Idempotency Strategy

**Implementation**:
```
1. Client generates UUID as idempotency key
2. Server checks Redis cache for existing result
3. If found, return cached response (304 Not Modified)
4. If not found, proceed with transaction
5. Cache result with 24-hour TTL
```

**Database Constraint**:
```sql
CREATE UNIQUE INDEX idx_idempotency ON transactions(idempotency_key);
```

This ensures database-level idempotency even if cache fails.

### 5. Consistency Model

**Choice**: Strong Consistency

**Implementation**:
- All balance updates within a single database transaction
- Pessimistic locking with `SELECT FOR UPDATE`
- No eventual consistency for money transfers

**Trade-off**: Slightly higher latency (3s) vs. weaker consistency models

---

## Database Design

![Database Schema](./database-schema.png)

### Schema Details

#### users table
```sql
CREATE TABLE users (
  user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone_number VARCHAR(20) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_phone ON users(phone_number);
CREATE INDEX idx_users_email ON users(email);
```

#### accounts table
```sql
CREATE TABLE accounts (
  account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(user_id),
  balance DECIMAL(19, 4) NOT NULL DEFAULT 0 CHECK (balance >= 0),
  currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  version INTEGER NOT NULL DEFAULT 1, -- Optimistic locking
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_accounts_user ON accounts(user_id);
```

**Why DECIMAL(19,4)**:
- 19 digits total, 4 decimal places
- Supports up to $999 trillion with cent precision
- Avoids floating-point rounding errors

#### transactions table
```sql
CREATE TABLE transactions (
  transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key UUID UNIQUE NOT NULL,
  sender_account_id UUID NOT NULL REFERENCES accounts(account_id),
  receiver_account_id UUID NOT NULL REFERENCES accounts(account_id),
  amount DECIMAL(19, 4) NOT NULL CHECK (amount > 0),
  currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  status VARCHAR(20) NOT NULL, -- pending, completed, failed, reversed
  failure_reason TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE UNIQUE INDEX idx_transactions_idempotency ON transactions(idempotency_key);
CREATE INDEX idx_transactions_sender ON transactions(sender_account_id, created_at DESC);
CREATE INDEX idx_transactions_receiver ON transactions(receiver_account_id, created_at DESC);
CREATE INDEX idx_transactions_status ON transactions(status, created_at);
```

**Partitioning Strategy**:
```sql
-- Monthly partitions
CREATE TABLE transactions_2024_01 PARTITION OF transactions
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE transactions_2024_02 PARTITION OF transactions
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... continue for each month
```

#### transaction_events table (Audit Log)
```sql
CREATE TABLE transaction_events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL REFERENCES transactions(transaction_id),
  event_type VARCHAR(50) NOT NULL, -- initiated, fraud_checked, completed, failed
  event_data JSONB, -- Flexible storage for event metadata
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_events_transaction ON transaction_events(transaction_id, created_at);
CREATE INDEX idx_events_type ON transaction_events(event_type, created_at);
```

#### fraud_checks table
```sql
CREATE TABLE fraud_checks (
  check_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL REFERENCES transactions(transaction_id),
  risk_score INTEGER NOT NULL CHECK (risk_score BETWEEN 0 AND 100),
  rules_triggered JSONB, -- Store which rules matched
  decision VARCHAR(20) NOT NULL, -- allow, block, review
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_fraud_transaction ON fraud_checks(transaction_id);
```

#### transfer_limits table
```sql
CREATE TABLE transfer_limits (
  limit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(user_id),
  daily_limit DECIMAL(19, 4) NOT NULL DEFAULT 2500.00,
  per_txn_limit DECIMAL(19, 4) NOT NULL DEFAULT 500.00,
  daily_used DECIMAL(19, 4) NOT NULL DEFAULT 0,
  reset_at TIMESTAMP NOT NULL, -- Reset daily at midnight
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_limits_user ON transfer_limits(user_id);
```

### Data Archival Strategy

**Hot Data** (< 2 years):
- Stored in PostgreSQL primary database
- Indexed for fast queries
- Accessible via Transaction History Service

**Cold Data** (2-7 years):
- Archived to AWS S3 in Parquet format
- Loaded into Redshift/Athena for analytics
- Compressed and encrypted
- Retrieved only for compliance/legal requests

---

## API Design

### REST API Endpoints

#### 1. Initiate Transfer
```http
POST /v1/transfers
Authorization: Bearer <JWT_TOKEN>
Idempotency-Key: <UUID>

Request Body:
{
  "recipient": {
    "phone_number": "+1234567890",  // or email
    "email": "recipient@example.com"
  },
  "amount": "100.00",
  "currency": "USD",
  "note": "Dinner split"
}

Response (201 Created):
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "amount": "100.00",
  "currency": "USD",
  "sender": {
    "user_id": "...",
    "phone_number": "+1987654321"
  },
  "recipient": {
    "user_id": "...",
    "phone_number": "+1234567890"
  },
  "created_at": "2024-05-25T10:30:00Z",
  "completed_at": "2024-05-25T10:30:02Z"
}

Error Responses:
400 Bad Request - Invalid amount or recipient
402 Payment Required - Insufficient balance
409 Conflict - Duplicate idempotency key with different params
422 Unprocessable Entity - Exceeds transfer limits
429 Too Many Requests - Rate limit exceeded
503 Service Unavailable - Fraud check timeout
```

#### 2. Get Balance
```http
GET /v1/accounts/{account_id}/balance
Authorization: Bearer <JWT_TOKEN>

Response (200 OK):
{
  "account_id": "...",
  "balance": "1250.75",
  "currency": "USD",
  "available_balance": "1250.75", // Excluding pending transactions
  "pending_balance": "0.00",
  "updated_at": "2024-05-25T10:30:00Z"
}
```

#### 3. Transaction History
```http
GET /v1/transactions?limit=50&cursor=<pagination_cursor>
Authorization: Bearer <JWT_TOKEN>

Query Parameters:
- limit: Number of results (default: 50, max: 100)
- cursor: Pagination cursor
- start_date: Filter by start date (ISO 8601)
- end_date: Filter by end date
- status: Filter by status (completed, pending, failed)

Response (200 OK):
{
  "transactions": [
    {
      "transaction_id": "...",
      "type": "sent", // or "received"
      "counterparty": {
        "name": "John Doe",
        "phone_number": "+1234567890"
      },
      "amount": "100.00",
      "status": "completed",
      "created_at": "2024-05-25T10:30:00Z"
    },
    // ... more transactions
  ],
  "pagination": {
    "next_cursor": "eyJsYXN0X2lkIjoi...",
    "has_more": true
  }
}
```

#### 4. Get Transaction Details
```http
GET /v1/transactions/{transaction_id}
Authorization: Bearer <JWT_TOKEN>

Response (200 OK):
{
  "transaction_id": "...",
  "idempotency_key": "...",
  "sender": {
    "user_id": "...",
    "phone_number": "+1987654321"
  },
  "recipient": {
    "user_id": "...",
    "phone_number": "+1234567890"
  },
  "amount": "100.00",
  "currency": "USD",
  "status": "completed",
  "note": "Dinner split",
  "created_at": "2024-05-25T10:30:00Z",
  "completed_at": "2024-05-25T10:30:02Z",
  "events": [
    {
      "event_type": "initiated",
      "timestamp": "2024-05-25T10:30:00Z"
    },
    {
      "event_type": "fraud_checked",
      "timestamp": "2024-05-25T10:30:01Z"
    },
    {
      "event_type": "completed",
      "timestamp": "2024-05-25T10:30:02Z"
    }
  ]
}
```

### API Design Principles

**1. Idempotency**:
- All write operations accept `Idempotency-Key` header
- Clients should generate UUID v4 for each unique request
- Server returns 409 if same key used with different parameters

**2. Rate Limiting**:
```
Per User:
- 10 transfers per minute
- 50 balance checks per minute
- 100 history requests per minute

Per IP:
- 100 requests per minute across all endpoints
```

**3. Versioning**:
- URI versioning: `/v1/transfers`
- Support N-1 version for 6 months after deprecation

**4. Error Handling**:
- Consistent error response format
- Include request_id for tracing
- Provide actionable error messages

**5. Security**:
- All endpoints require JWT authentication
- TLS 1.3 for encryption in transit
- No sensitive data in URLs (use POST body)

---

## Key Transaction Flow

![Transfer Sequence Diagram](./transfer-sequence.png)

### Detailed Transfer Flow

#### Step-by-Step Execution

**1. Client Request**
```
POST /v1/transfers
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
{
  "recipient": {"phone_number": "+1234567890"},
  "amount": "100.00"
}
```

**2. API Gateway Validation**
- Verify JWT token
- Check rate limits
- Route to Transfer Service

**3. Transfer Service - Idempotency Check**
```python
# Check Redis cache
cached_result = redis.get(f"idempotency:{idempotency_key}")
if cached_result:
    return cached_result  # Return immediately

# Check database
existing_txn = db.query("SELECT * FROM transactions WHERE idempotency_key = ?",
                        idempotency_key)
if existing_txn:
    cache_and_return(existing_txn)
```

**4. Fraud Detection Service - Rule-Based Checks**
```python
fraud_rules = [
    check_velocity_limits(user_id),  # Max 10 txns/hour
    check_amount_threshold(amount),   # Flag if > $1000
    check_recipient_reputation(recipient_id),
    check_geolocation(user_ip),
]

risk_score = calculate_risk_score(fraud_rules)

if risk_score > 80:
    return "BLOCK"
elif risk_score > 50:
    return "REVIEW"  # Manual review queue
else:
    return "ALLOW"
```

**5. Balance Service - Check Sufficient Funds**
```sql
-- Pessimistic locking to prevent race conditions
SELECT balance
FROM accounts
WHERE account_id = :sender_account_id
FOR UPDATE;

-- Check balance
IF balance < :amount THEN
    ROLLBACK;
    RETURN "INSUFFICIENT_FUNDS";
END IF;
```

**6. Transfer Service - Execute Transaction**
```sql
BEGIN TRANSACTION;

-- Debit sender
UPDATE accounts
SET balance = balance - :amount,
    version = version + 1,
    updated_at = NOW()
WHERE account_id = :sender_account_id;

-- Credit recipient
UPDATE accounts
SET balance = balance + :amount,
    version = version + 1,
    updated_at = NOW()
WHERE account_id = :receiver_account_id;

-- Insert transaction record
INSERT INTO transactions (
    transaction_id,
    idempotency_key,
    sender_account_id,
    receiver_account_id,
    amount,
    currency,
    status,
    created_at,
    completed_at
) VALUES (
    :transaction_id,
    :idempotency_key,
    :sender_account_id,
    :receiver_account_id,
    :amount,
    'USD',
    'completed',
    NOW(),
    NOW()
);

-- Insert audit event
INSERT INTO transaction_events (
    transaction_id,
    event_type,
    event_data,
    created_at
) VALUES (
    :transaction_id,
    'completed',
    '{"ip": "192.168.1.1", "user_agent": "..."}',
    NOW()
);

COMMIT;
```

**7. Publish Event to Message Queue**
```json
{
  "event_type": "transfer_completed",
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "sender_user_id": "...",
  "receiver_user_id": "...",
  "amount": "100.00",
  "timestamp": "2024-05-25T10:30:02Z"
}
```

**8. Notification Service (Asynchronous)**
```python
# Consume event from queue
event = kafka.consume("transfer_events")

# Send notifications
send_push_notification(sender_user_id, "Transfer sent: $100")
send_push_notification(receiver_user_id, "Money received: $100")
send_email(sender_email, "Transfer Receipt", email_template)
```

**9. Cache Result**
```python
# Cache for 24 hours
redis.setex(
    f"idempotency:{idempotency_key}",
    86400,  # TTL: 24 hours
    json.dumps(transaction_result)
)
```

**10. Return Response**
```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "amount": "100.00",
  "completed_at": "2024-05-25T10:30:02Z"
}
```

### Latency Breakdown

| Step | Target Latency | Notes |
|------|---------------|-------|
| API Gateway | 10ms | Token validation, routing |
| Idempotency Check (Redis) | 5ms | Cache hit |
| Fraud Rules Check | 50ms | Rule evaluation |
| Balance Check (DB) | 20ms | SELECT FOR UPDATE |
| Transaction Commit (DB) | 100ms | Multi-row updates |
| Event Publish (Kafka) | 10ms | Fire and forget |
| API Response | 5ms | JSON serialization |
| **Total** | **~200ms** | Well under 3s target |

---

## Scalability & Performance

### Capacity Planning

#### Current Scale (Year 1)
- **Users**: 50M registered, 5M DAU
- **Transactions**: 200 TPS average, 1,000 TPS peak
- **Daily Volume**: 17.3M transactions/day
- **Data Growth**: ~100GB/month

#### Database Scaling

**Write Capacity**:
```
PostgreSQL (Primary):
- Instance: db.r6g.2xlarge (8 vCPU, 64GB RAM)
- Provisioned IOPS: 20,000 IOPS
- Handles 1,000 TPS comfortably
```

**Read Capacity**:
```
Read Replicas (3):
- Route read-heavy queries (transaction history, balance checks)
- Cross-region replicas for disaster recovery
- Lag < 100ms for strong consistency
```

**Connection Pooling**:
```
PgBouncer:
- Pool size: 100 connections per service
- Transaction mode for short-lived connections
- Reduces connection overhead
```

#### Caching Strategy

**Redis Cluster**:
```yaml
Configuration:
  Nodes: 3 (primary + 2 replicas)
  Instance: cache.r6g.large (2 vCPU, 16GB RAM)
  Eviction: LRU (Least Recently Used)

Cached Data:
  - User balances (TTL: 60s)
  - Idempotency keys (TTL: 24 hours)
  - Rate limit counters (TTL: 1 minute)
  - Fraud rule configurations (TTL: 5 minutes)

Hit Rate Target: > 95%
```

**Cache Invalidation**:
```python
# Invalidate balance cache after transfer
def on_transfer_complete(transaction):
    redis.delete(f"balance:{transaction.sender_account_id}")
    redis.delete(f"balance:{transaction.receiver_account_id}")
```

#### Horizontal Scaling

**Microservices Scaling**:
```yaml
Transfer Service:
  Min Replicas: 5
  Max Replicas: 50
  CPU Threshold: 70%
  Memory Threshold: 80%

Balance Service:
  Min Replicas: 10  # High read volume
  Max Replicas: 100

Transaction History Service:
  Min Replicas: 5
  Max Replicas: 30

Fraud Detection Service:
  Min Replicas: 3
  Max Replicas: 20

Notification Service:
  Min Replicas: 3
  Max Replicas: 20
```

**Load Balancing**:
```
AWS Application Load Balancer:
- Health checks every 10s
- Unhealthy threshold: 3 consecutive failures
- Drain connections: 30s before termination
- Sticky sessions: Not required (stateless services)
```

#### Message Queue Scaling

**Kafka Configuration**:
```yaml
Topics:
  transfer_events:
    Partitions: 10
    Replication Factor: 3
    Retention: 7 days

  notification_events:
    Partitions: 5
    Replication Factor: 3
    Retention: 1 day

Consumer Groups:
  - notification_service (3 consumers)
  - analytics_service (2 consumers)
  - audit_logger (1 consumer)
```

### Performance Optimizations

**1. Database Query Optimization**
```sql
-- Use covering index for common queries
CREATE INDEX idx_txn_history_covering
ON transactions(sender_account_id, created_at DESC)
INCLUDE (amount, receiver_account_id, status);

-- Partition pruning for date range queries
SELECT * FROM transactions
WHERE sender_account_id = :user_id
  AND created_at >= '2024-01-01'
  AND created_at < '2024-02-01';
-- Only scans transactions_2024_01 partition
```

**2. Connection Pooling**
```python
# Use connection pool to avoid connection overhead
db_pool = create_pool(
    host="db.example.com",
    port=5432,
    min_size=10,
    max_size=100,
    timeout=5
)
```

**3. Batch Operations**
```python
# Batch notification sends
def send_notifications_batch(events):
    notifications = [create_notification(e) for e in events]
    notification_service.send_batch(notifications)  # Single RPC call
```

**4. Read-After-Write Consistency**
```python
# For balance checks after transfer, read from primary
if recently_transferred(user_id):
    balance = db_primary.query("SELECT balance FROM accounts WHERE ...")
else:
    balance = db_replica.query("SELECT balance FROM accounts WHERE ...")
```

### Monitoring & Observability

**Key Metrics**:
```
Business Metrics:
- Transactions per second (TPS)
- Success rate (target: > 99.9%)
- Average transaction value
- Failed transaction reasons

Technical Metrics:
- API latency (p50, p95, p99)
- Database query time
- Cache hit rate
- Queue lag
- Error rate by service

Infrastructure Metrics:
- CPU utilization
- Memory usage
- Network throughput
- Disk IOPS
```

**Alerting**:
```yaml
Alerts:
  - name: high_error_rate
    condition: error_rate > 1%
    duration: 5m
    severity: critical

  - name: high_latency
    condition: p95_latency > 5s
    duration: 2m
    severity: warning

  - name: low_cache_hit_rate
    condition: cache_hit_rate < 90%
    duration: 10m
    severity: warning

  - name: database_connection_pool_exhausted
    condition: available_connections < 10
    duration: 1m
    severity: critical
```

---

## Security & Compliance
### Security Measures

#### 1. Authentication & Authorization

**JWT Tokens**:
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "account_id": "...",
  "iat": 1716630000,
  "exp": 1716633600,  // 1 hour expiration
  "scope": ["transfer:write", "balance:read"]
}
```

**Token Management**:
- Short-lived access tokens (1 hour)
- Refresh tokens with 30-day expiration
- Token rotation on each refresh
- Revocation list for compromised tokens

**Authorization Checks**:
```python
def authorize_transfer(user_id, sender_account_id):
    # Verify user owns the sender account
    account = db.query("SELECT user_id FROM accounts WHERE account_id = ?",
                       sender_account_id)
    if account.user_id != user_id:
        raise UnauthorizedException("Cannot transfer from another user's account")
```

#### 2. Data Encryption

**Encryption in Transit**:
- TLS 1.3 for all API communication
- Certificate pinning for mobile apps
- Perfect Forward Secrecy (PFS) enabled

**Encryption at Rest**:
```yaml
Database:
  - AWS RDS encryption with KMS
  - Encryption key rotation every 90 days
  - Transparent Data Encryption (TDE)

S3 Archive:
  - Server-side encryption (SSE-KMS)
  - Bucket policies restrict public access
  - Versioning enabled for audit trail

Redis Cache:
  - ElastiCache encryption at rest
  - Encryption in transit between nodes
```

**Sensitive Data Handling**:
```python
# PII (Personally Identifiable Information) encryption
def encrypt_pii(data):
    return aes_encrypt(data, key=get_pii_key())

# Store phone numbers hashed for lookup
phone_hash = sha256(phone_number + salt)
```

#### 3. Fraud Prevention

**Rule-Based Checks** (Synchronous):
```python
fraud_checks = {
    "velocity_check": lambda: user_txn_count_last_hour < 10,
    "amount_check": lambda: amount <= daily_limit,
    "recipient_check": lambda: not is_blocked_recipient(recipient_id),
    "geolocation_check": lambda: is_valid_country(user_ip),
    "device_fingerprint": lambda: is_known_device(device_id),
}
```

**ML-Based Scoring** (Asynchronous):
```python
# Called after transaction completes for learning
def ml_fraud_scoring(transaction):
    features = extract_features(transaction)
    risk_score = ml_model.predict(features)
    
    if risk_score > 0.9:
        flag_for_review(transaction)
        potentially_reverse(transaction)
```

**Fraud Detection Features**:
- Transaction amount
- Time of day
- Sender/recipient relationship history
- Device fingerprint
- IP address geolocation
- Transaction velocity
- Account age

#### 4. Security Headers

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
X-XSS-Protection: 1; mode=block
```

#### 5. Audit Logging

**Comprehensive Logging**:
```python
audit_log = {
    "event_type": "transfer_initiated",
    "user_id": "...",
    "ip_address": "192.168.1.1",
    "user_agent": "Mozilla/5.0...",
    "request_id": "...",
    "timestamp": "2024-05-25T10:30:00Z",
    "details": {
        "amount": "100.00",
        "recipient_id": "...",
        "idempotency_key": "..."
    }
}
```

**Immutable Audit Trail**:
- All events written to append-only log
- Streamed to SIEM (Security Information and Event Management)
- Retention: 7 years for compliance
- Tamper-evident using hash chains

### Compliance Requirements

#### 1. SOC 2 Type II

**Security Controls**:
- Multi-factor authentication for admin access
- Regular penetration testing (quarterly)
- Incident response plan
- Security awareness training
- Vendor security assessments

#### 2. Data Retention

**Transaction Records**:
- Hot storage: 2 years (PostgreSQL)
- Cold storage: 5 years (S3 + Redshift)
- Total retention: 7 years

**User Data**:
- Active accounts: Indefinite retention
- Deleted accounts: 30-day grace period, then anonymized
- Right to deletion: GDPR compliance (where applicable)

#### 3. AML (Anti-Money Laundering)

**Transaction Monitoring**:
```python
aml_checks = {
    "large_transaction": amount > 10000,  # Report to FinCEN
    "structured_transaction": detect_structuring(user_txns),
    "high_risk_jurisdiction": recipient_country in high_risk_list,
}

if any(aml_checks.values()):
    file_suspicious_activity_report(transaction)
```

**Know Your Customer (KYC)**:
- Identity verification at registration (out of scope)
- Ongoing monitoring for suspicious activity
- Enhanced due diligence for high-risk customers

#### 4. PSD2 / Open Banking (Future)

**Strong Customer Authentication (SCA)**:
- Two-factor authentication for transfers > €30
- Biometric authentication support
- Transaction risk analysis exemptions

---

## Failure Handling & Recovery

### Failure Scenarios & Solutions

#### 1. Server Crash Mid-Transaction

**Problem**: Server crashes after debiting sender but before crediting recipient.

**Solution**: Database transaction ensures atomicity
```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE account_id = sender;
  UPDATE accounts SET balance = balance + 100 WHERE account_id = receiver;
  INSERT INTO transactions (...);
COMMIT;  -- Either all succeed or all rollback
```

If server crashes:
- Transaction is rolled back by PostgreSQL
- No partial state (both balances unchanged)
- Write-Ahead Logging (WAL) ensures durability

**Recovery**:
```python
# Client retries with same idempotency key
POST /v1/transfers
Idempotency-Key: same-uuid-as-before

# Server checks: transaction never completed, proceeds normally
```

#### 2. Duplicate Request (Network Retry)

**Problem**: Network timeout causes client to retry, risking double transfer.

**Solution**: Idempotency key prevents duplicate processing

```python
# First request
idempotency_key = "550e8400-..."
result = process_transfer(idempotency_key, amount, recipient)
cache_result(idempotency_key, result)

# Retry request (same key)
cached = get_cached_result(idempotency_key)
if cached:
    return cached  # Return previous result, no duplicate transfer
```

**Database Constraint**:
```sql
-- Unique constraint prevents duplicate insertion
CREATE UNIQUE INDEX idx_idempotency ON transactions(idempotency_key);
-- INSERT will fail if same key used again
```

#### 3. Concurrent Balance Updates

**Problem**: Two transfers from same account execute simultaneously.

**Solution 1**: Pessimistic Locking (Recommended)
```sql
BEGIN TRANSACTION;

-- Lock the row for update
SELECT balance FROM accounts
WHERE account_id = :sender_id
FOR UPDATE;

-- Other transactions wait here until lock is released

IF balance >= :amount THEN
    UPDATE accounts SET balance = balance - :amount
    WHERE account_id = :sender_id;
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

**Solution 2**: Optimistic Locking (Alternative)
```sql
-- Add version column
UPDATE accounts
SET balance = balance - :amount,
    version = version + 1
WHERE account_id = :sender_id
  AND version = :expected_version;

-- If version mismatch (concurrent update), UPDATE affects 0 rows
-- Application retries with new version
```

**Trade-off**: Pessimistic locking has slightly higher latency but guarantees consistency.

#### 4. Database Failure

**Problem**: Primary database becomes unavailable.

**Solution**: Automated failover

```yaml
PostgreSQL High Availability:
  Primary: us-east-1a
  Standby: us-east-1b, us-east-1c
  
  Failover:
    Detection Time: 30 seconds (health check)
    Promotion Time: 60 seconds
    DNS Update: 30 seconds
    Total Downtime: ~2 minutes
    
  Synchronous Replication:
    - At least 1 standby must acknowledge writes
    - Prevents data loss (RPO = 0)
```

**Circuit Breaker Pattern**:
```python
@circuit_breaker(failure_threshold=5, timeout=60)
def query_database(query):
    return db.execute(query)

# After 5 consecutive failures:
# - Circuit opens
# - Fail fast without trying database
# - Retry after 60 seconds
```

#### 5. External Service Failure

**Problem**: Fraud detection service is down.

**Solution**: Graceful degradation

```python
def check_fraud(transaction):
    try:
        result = fraud_service.check(transaction, timeout=1s)
        return result
    except TimeoutError:
        # Fallback: Allow transaction with basic checks only
        if transaction.amount < 500:
            log_warning("Fraud service timeout, allowing small transaction")
            return "ALLOW"
        else:
            log_alert("Fraud service timeout, blocking large transaction")
            return "BLOCK"
    except Exception as e:
        log_error(f"Fraud service error: {e}")
        return "BLOCK"  # Fail closed for security
```

**Retry with Exponential Backoff**:
```python
@retry(max_attempts=3, backoff=exponential, exceptions=[NetworkError])
def call_external_service(request):
    return external_api.call(request)
```

#### 6. Message Queue Failure

**Problem**: Kafka cluster is unavailable, notifications cannot be sent.

**Solution**: Async processing with dead letter queue

```python
try:
    kafka.publish("transfer_events", event)
except KafkaException:
    # Fallback: Store in database for later processing
    db.execute("INSERT INTO pending_notifications (...)")

# Background job retries failed notifications
def retry_pending_notifications():
    pending = db.query("SELECT * FROM pending_notifications WHERE retries < 3")
    for notification in pending:
        try:
            kafka.publish("transfer_events", notification)
            db.execute("DELETE FROM pending_notifications WHERE id = ?", notification.id)
        except:
            db.execute("UPDATE pending_notifications SET retries = retries + 1 WHERE id = ?",
                      notification.id)
```

### Disaster Recovery

**Backup Strategy**:
```yaml
Database Backups:
  Full Backup: Daily at 2 AM UTC
  Incremental Backup: Every 6 hours
  Point-in-Time Recovery: 7 days
  Retention: 30 days
  
  Cross-Region Replication:
    Primary: us-east-1
    Backup: us-west-2
    Replication Lag: < 5 minutes
```

**Recovery Time Objective (RTO)**: 1 hour
**Recovery Point Objective (RPO)**: 5 minutes

**Disaster Recovery Plan**:
1. **Detection**: Automated monitoring alerts on-call engineer (5 min)
2. **Assessment**: Evaluate scope of failure (10 min)
3. **Failover**: Switch to backup region (20 min)
4. **Verification**: Test critical paths (15 min)
5. **Communication**: Notify stakeholders (10 min)

---

## Trade-offs & Alternatives

### Key Design Trade-offs

#### 1. Strong Consistency vs. Availability

**Choice**: Strong consistency (ACID transactions)

**Trade-off**:
- ✅ **Pro**: No double-spend, correct balances guaranteed
- ✅ **Pro**: Simpler reasoning about system state
- ❌ **Con**: Lower availability during network partitions (CAP theorem)
- ❌ **Con**: Slightly higher latency (~200ms vs. ~50ms for eventual consistency)

**Alternative**: Eventual consistency with compensating transactions
```
Pros: Higher availability, lower latency
Cons: Complex conflict resolution, potential for temporary inconsistency
Verdict: Rejected - financial correctness is paramount
```

#### 2. Microservices vs. Monolith

**Choice**: Microservices

**Trade-off**:
- ✅ **Pro**: Independent scaling (fraud detection vs. notifications)
- ✅ **Pro**: Team autonomy and faster feature velocity
- ✅ **Pro**: Fault isolation (notification failure doesn't block transfers)
- ❌ **Con**: Increased operational complexity (service mesh, distributed tracing)
- ❌ **Con**: Network latency between services
- ❌ **Con**: Distributed transaction complexity

**Alternative**: Modular monolith
```
Pros: Simpler deployment, lower latency, easier local development
Cons: Harder to scale components independently
Verdict: Acceptable for early stage, plan migration path to microservices
```

#### 3. Synchronous vs. Asynchronous Notifications

**Choice**: Asynchronous (via message queue)

**Trade-off**:
- ✅ **Pro**: Transfer completes quickly (<200ms) without waiting for notification
- ✅ **Pro**: Retry logic for failed notifications
- ✅ **Pro**: Can add new consumers (analytics) without changing transfer code
- ❌ **Con**: User might not receive immediate push notification
- ❌ **Con**: Need infrastructure for message queue

**Alternative**: Synchronous notifications
```
Pros: Simpler architecture, guaranteed notification before API response
Cons: Higher latency (500ms+), notification failures block transfers
Verdict: Rejected - notifications are not critical to transaction correctness
```

#### 4. PostgreSQL vs. NoSQL

**Choice**: PostgreSQL

**Trade-off**:
- ✅ **Pro**: ACID transactions for correctness
- ✅ **Pro**: Rich query capabilities (transaction history with filters)
- ✅ **Pro**: Mature tooling and expertise
- ❌ **Con**: Vertical scaling limits (~10K TPS per node)
- ❌ **Con**: More complex sharding if needed

**Alternative**: DynamoDB (NoSQL)
```
Pros: Unlimited horizontal scaling, managed service
Cons: No multi-item transactions (cannot atomically debit + credit)
Cons: Complex query patterns require secondary indexes
Verdict: Rejected - lack of transactions is dealbreaker for finance
```

**Alternative**: CockroachDB (NewSQL)
```
Pros: ACID + horizontal scaling, distributed by design
Cons: Higher operational complexity, less mature ecosystem
Verdict: Consider for future when scale exceeds PostgreSQL (>10K TPS)
```

#### 5. Pessimistic vs. Optimistic Locking

**Choice**: Pessimistic locking (`SELECT FOR UPDATE`)

**Trade-off**:
- ✅ **Pro**: Prevents race conditions without retry logic
- ✅ **Pro**: Simpler application code
- ❌ **Con**: Holds database locks longer
- ❌ **Con**: Potential for deadlocks if not careful

**Alternative**: Optimistic locking (version field)
```
Pros: Better concurrency, no lock contention
Cons: Requires retry logic, wasted work on conflicts
Verdict: Acceptable alternative, use for read-heavy workloads (balance checks)
```

### Future Enhancements

#### 1. Multi-Currency Support

**Changes Required**:
- Add `currency` field to all amount columns
- Implement foreign exchange service integration
- Handle currency conversion rates and fees
- Store exchange rates history for audit

**Estimated Effort**: 2-3 months

#### 2. Recurring Transfers

**Changes Required**:
- New `recurring_transfers` table with schedule
- Background job to execute scheduled transfers
- Cancellation and modification APIs
- Balance hold for pending recurring transfers

**Estimated Effort**: 1 month

#### 3. Multi-Party Transfers (Split Bills)

**Changes Required**:
- Extend transaction model to support multiple recipients
- Calculate split amounts (equal/custom)
- Handle partial failures (some recipients unavailable)
- UI for managing split requests

**Estimated Effort**: 2 months

#### 4. Transfer Reversal / Refund

**Changes Required**:
- Add `reversed_transaction_id` to transactions table
- Implement reversal API with authorization checks
- Handle partially reversed transactions
- Audit trail for all reversals

**Estimated Effort**: 3 weeks

#### 5. Blockchain Audit Trail

**Concept**: Immutable blockchain-based audit log

**Benefits**:
- Cryptographic proof of transaction integrity
- Non-repudiation for compliance
- Transparent audit trail for regulators

**Challenges**:
- Blockchain scaling for 200 TPS
- Privacy concerns (public ledger)
- Operational complexity

**Verdict**: Interesting for future, not MVP

---

## Appendix: Detailed Interview Answers

### Question 1: If the server crashes mid-transfer, how do you recover?

**Answer**: 
Database transactions ensure atomicity. If the server crashes:
1. PostgreSQL's Write-Ahead Logging (WAL) records all operations
2. On restart, PostgreSQL automatically rolls back uncommitted transactions
3. Client retries with the same idempotency key
4. Server detects no completed transaction for that key and processes normally
5. Database-level unique constraint on `idempotency_key` prevents duplicates

**Key Point**: Never have partial state. Both debit and credit happen atomically within a single `BEGIN...COMMIT` block.

### Question 2: If the same request arrives twice due to network retry, how do you prevent double transfer?

**Answer**:
Two-layer idempotency:
1. **Redis Cache** (fast path): Check `idempotency:<key>` in Redis. If found, return cached result (24-hour TTL)
2. **Database Constraint** (safety net): `UNIQUE INDEX` on `transactions(idempotency_key)` prevents duplicate insertion

Client generates UUID for each unique request. Server stores result in cache and database. Retries with same key return cached result without re-executing.

**Key Point**: Idempotency key must be client-generated (not server-generated transaction ID) to work across retries.

### Question 3: If a balance read and update happen concurrently, how do you ensure consistency?

**Answer**:
Pessimistic locking with `SELECT FOR UPDATE`:

```sql
BEGIN TRANSACTION;

-- This locks the row, other transactions wait
SELECT balance FROM accounts WHERE account_id = :sender_id FOR UPDATE;

-- Check sufficient funds
IF balance >= :amount THEN
    UPDATE accounts SET balance = balance - :amount WHERE account_id = :sender_id;
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

Concurrent transfers queue up. Only one executes at a time per account. Database serializes updates automatically.

**Alternative**: Optimistic locking with `version` field and retry on conflict (preferred for read-heavy workloads).

### Question 4: Where are the SPOFs (Single Points of Failure) and how do you eliminate them?

**SPOFs Identified**:

1. **Database Primary**: 
   - Mitigation: Multi-AZ standby with automated failover (2 min RTO)
   
2. **API Gateway**:
   - Mitigation: Multiple availability zones, health checks, auto-scaling

3. **Redis Cache**:
   - Mitigation: Redis Cluster with replicas, cache-aside pattern (can fallback to DB)

4. **Message Queue**:
   - Mitigation: Kafka cluster (3 brokers), replication factor 3, fallback to DB storage

5. **Load Balancer**:
   - Mitigation: AWS ALB is multi-AZ by default

**Key Point**: Every stateful component has redundancy. Stateless services scale horizontally automatically.

### Question 5: If transaction volume increases 10x, where is the bottleneck and how do you scale?

**Bottlenecks & Solutions**:

1. **Database Writes** (most likely bottleneck at 2,000 TPS):
   - Short-term: Vertical scaling (larger instance)
   - Medium-term: Connection pooling optimization, query optimization
   - Long-term: Sharding by `account_id` hash (10 shards = 10x capacity)

2. **API Gateway**:
   - Solution: Horizontal scaling (stateless), already auto-scales

3. **Redis Cache**:
   - Solution: Cluster mode with more shards

4. **Microservices**:
   - Solution: Already horizontally scalable, increase max replicas

**Scaling Plan**:
```
1,000 TPS:   Current design (single PostgreSQL)
10,000 TPS:  Vertical scaling + read replicas + aggressive caching
100,000 TPS: Database sharding + NewSQL (CockroachDB)
```

**Key Point**: Start simple, measure bottlenecks, optimize incrementally. Don't prematurely shard.

---

## Summary

This P2P money transfer system design prioritizes **correctness** and **consistency** while maintaining **high availability** and **scalability** to handle 50M users and 1,000 TPS.

**Core Strengths**:
- ✅ Strong consistency via ACID transactions
- ✅ Idempotency prevents double-spend
- ✅ Horizontal scalability for stateless services
- ✅ Comprehensive audit trail for compliance
- ✅ Defense-in-depth security measures
- ✅ Graceful degradation and fault tolerance

**Future Work**:
- Multi-currency support
- Advanced ML fraud detection
- Blockchain audit trail
- Mobile SDK for easier integration

**Technology Stack Summary**:
```yaml
Backend: Python/Go microservices
API Gateway: AWS API Gateway / Kong
Database: PostgreSQL 15+ (primary + 3 replicas)
Cache: Redis Cluster
Message Queue: Apache Kafka
Storage: AWS S3
Container Orchestration: Kubernetes (EKS)
Monitoring: Prometheus + Grafana
Logging: ELK Stack (Elasticsearch, Logstash, Kibana)
Tracing: Jaeger / OpenTelemetry
```

---

**Document Version**: 1.0  
**Last Updated**: 2024-05-25  
**Author**: System Design Exercise  
**Status**: Interview Solution Document
