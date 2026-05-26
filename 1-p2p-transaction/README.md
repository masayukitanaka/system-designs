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

## 1. サマリ（Executive Summary）

### 1.1 システム概要
本システムは、Zelle/Venmoライクなリアルタイムのピアツーピア（P2P）送金システムです。ユーザーは電話番号またはメールアドレスを指定するだけで、即座に他のユーザーへ送金できます。

### 1.2 機能要件
| 要件 | 内容 |
|------|------|
| コア機能 | 送金、受取、残高照会、取引履歴表示 |
| 識別方法 | 電話番号またはメールアドレス |
| 通貨 | USD単一通貨のみ |
| 送金制限 | 1回あたり$500、1日あたり$2,500 |
| 決済方式 | 即時決済（ACH方式ではない） |
| 対象外 | アカウント作成、投資機能、多通貨対応 |

### 1.3 非機能要件
| 要件 | 目標値 | 備考 |
|------|--------|------|
| ユーザー規模 | 登録ユーザー5,000万人、DAU 500万 | |
| スループット | 平均200 TPS、ピーク1,000 TPS | |
| レイテンシ | エンドツーエンドで3秒以内 | 送金完了まで |
| 可用性 | 99.99% | 年間ダウンタイム約50分 |
| データ保持 | 取引履歴を7年間保持 | 規制要件 |
| 地域 | 米国のみ（初期ローンチ） | |

### 1.4 セキュリティ・コンプライアンス
- **KYC（本人確認）**: 上流サービスで実施済み（スコープ外）
- **認証**: 既存の認証サービスを利用（スコープ外）
- **不正検知**: ルールベースの不正チェックをインライン実装、ML-basedスコアリングは外部API
- **AML（マネーロンダリング対策）**: 外部APIコール
- **コンプライアンス**: SOC2準拠（PCI-DSSは対象外）

### 1.5 技術スタック前提
- **クラウドプロバイダー**: AWS
- **アーキテクチャ**: マイクロサービスアーキテクチャ（理由は後述）
- **スコープ**: バックエンドとAPIコントラクトに集中（フロントエンド/モバイルは対象外）

---

