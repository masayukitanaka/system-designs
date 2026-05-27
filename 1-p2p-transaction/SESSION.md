# SESSION.md — 作業引き継ぎメモ

このファイルは次回セッションで文脈を素早く取り戻すためのメモ。
**設計の最終内容は `README.md`（英語）/ `README-ja.md`（日本語）が正**。ここには README に書かない「経緯・確定済みの判断・運用知識・落とし穴」を残す。

最終更新: 2026-05-26

---

## 1. このプロジェクトは何か

- システムデザイン面接想定の成果物。お題は **「P2P送金システム（Zelle/Venmo風）」**。
- 要件・問答は README の Part 1〜4（英語、面接プロンプト）に固定。**設計ドキュメント本体は §1〜§8**。
- 成果物 = README 2言語版 + 図4種 × 2言語（PNG）。図のソースは `diagrams/*.mmd`。

### ファイル対応（重要）
| 言語 | README | 図PNG | 図ソース |
|------|--------|-------|----------|
| 英語 | `README.md` | `*.png`（ベース名） | `diagrams/*-en.mmd` |
| 日本語 | `README-ja.md` | `*-ja.png` | `diagrams/*.mmd`（接尾辞なし=日本語） |

図は4種: `architecture-overview` / `architecture-k8s` / `transfer-sequence` / `database-schema`。
**EN/JAで内容が一致しているか常に両方を更新すること**（過去に片方だけ直して非対称になった事故あり）。

---

## 2. 確定している設計判断（ユーザーが明示的に決めたこと）

ユーザー指示で何度も方針転換した。**現在有効なのは以下**。過去案（取り消し済み）は §3 に。

1. **サービス構成**: URLパスで分割しない。トランザクション境界で分ける。
   - **Payments Service** に統合: 送金・台帳・残高・取引履歴（Transfer + Transaction History + Account/Ledger を1つに）。
   - **独立サービスとして存続**: Fraud Engine / User Directory / Notification。
2. **外部API（ML不正スコアリング・AML）は Fraud Service 経由**で呼ぶ。Payments から直接呼ばない。
3. **データストア（ポリグロット、金額は DynamoDB）**:
   - **DynamoDB = 金額の正**。残高アイテム（`available` + `held`）+ 全台帳（約126億行/年）。
   - **PostgreSQL = メタデータ + 残高の表示キャッシュのみ**（`balance_cache_minor` は非正、減算判定に使わない）。
   - 容量根拠: 200 TPS × 2行 × 86,400 × 365 ≒ **約126億行/年** → PostgreSQL単独は不可能、というのが DynamoDB 採用の決め手。
4. **送金フロー = 同期hold → 非同期決済（エスクローなし）**:
   - 同期: お金を動かさず `held += 金額`（条件 `available - held >= 金額`）で予約。見た目の残高だけ減る。台帳記帳なし。
   - 非同期ワークフロー: (1) Fraud/AML（稀に人手承認）→ (2) モラトリアム（取消可能な待機）→ (3) 決済（**ここで初めて実送金**、台帳に DEBIT/CREDIT 追記）→ (4) 通知。
   - **お金が実際に減るのは決済時のみ**。取消は `held -= 金額`（hold解放、台帳記帳なし、取り戻し不要）。
   - 二重支払い防止 = 同期hold時の条件付き書込 + クライアント発行 **Idempotency-Key**。
   - ワークフロー基盤: Kafka + サービス内ステートマシン（複雑化したら Step Functions、と併記）。
5. **冪等性は DB管理**（Redis の SETNX ではない）。Redis は「キャッシュ / レート制限」のみ。
6. **残高は集約・シャードしない**。`available`/`held` を持つ勘定アイテム1つで表現（ホット勘定対策は §7.2 のサブアカウント案として記載するに留める）。

---

## 3. 設計の変遷（同じ轍を踏まないため）

ユーザーは段階的に方針を変えてきた。**逆戻りした案を再提案しないこと**。

1. 初版: エスクロー + 残高シャード合計 + Outbox で DynamoDB へ履歴転写、PostgreSQL が現在状態の正。
2. 「全トランザクションを DynamoDB に」→ 金額の正を DynamoDB へ反転、PostgreSQL は残高キャッシュ＋メタに降格。
3. 「残高シャードの合計はおかしい（1レコード=1トランザクション、各レコードに処理後残高）」→ 一度「最新台帳エントリの `balance_after` が現在残高」モデルにした。
4. 「二重処理は Idempotency-Key チェックで防げる」→ version/シャードの楽観ロック的な複雑機構を撤回。
5. **「エスクロー不要、お金が減るのは決済時。同期は見た目だけ減る」→ 現行の `available`/`held` hold モデルに確定**（最新コミット `remove escrow`）。

→ つまり「残高シャード」「balance_after を現在残高として読む」「エスクロー勘定」は**いずれも撤回済み**。現行は `available`/`held` 方式。

---

## 4. 図の生成方法（mmdc）と落とし穴

```bash
cd diagrams
# EN（→ ../<name>.png）、JA（→ ../<name>-ja.png）
mmdc -i <name>-en.mmd -o ../<name>.png    -p puppeteer.json -b white -s 2
mmdc -i <name>.mmd    -o ../<name>-ja.png -p puppeteer.json -b white -s 2
```
- `mmdc`（@mermaid-js/mermaid-cli）と `puppeteer.json`（`--no-sandbox`）は既に用意済み。日本語フォントもそのまま出る。
- **エラー検出注意**: `mmdc` は失敗しても終了コードや stdout が紛らわしいことがある。`2>&1` して `grep -i "parse error"` で必ず確認する（過去に「OK」表示のまま古いPNGが残った事故あり）。

### mermaid 構文の地雷（実際に踏んだもの）
- **`sequenceDiagram` のメッセージ／Note 内に `;`（セミコロン）を書かない** → 文区切りと解釈されパースエラー。`,` や `・`、`—` に置換。
- **`erDiagram` の属性コメント内に `#` を書かない**（例 `ACCOUNT#id#SHARD#n`）→ パースエラー。`-` 区切りにする。
- **`erDiagram` の属性名に `pk`/`sk` を使わない** → キー制約キーワードと衝突。`partition_key`/`sort_key` にする。
- **`flowchart` のエッジラベル `|...|` 内に丸括弧 `()` を書かない**（例 `|outbox relay (CDC)|`）→ ノード形状構文と誤認。`/` 等に置換。

---

## 5. 図の現在の中身（README と重複するが認識しておくと速い）

- **architecture-overview**: Payments Service（同期hold / 決済送金）+ Fraud/User/Notification、Settlement Workflow ノード、Data層は DynamoDB（available/held + 台帳, 正）と PostgreSQL（メタ + キャッシュ, Streamsでキャッシュ更新）。Fraud → ML/AML。
- **architecture-k8s**: EKS の core namespace に `payments-svc`/`fraud-svc`/`user-svc`/`notify-svc`。マネージドは DynamoDB（金額の正）+ Aurora（メタ+キャッシュ）+ Redis + MSK + S3。**このセッションでは k8s 図は最後の数回の変更で再生成不要だった**（payments/fraud/user/notify 構成は不変）。
- **transfer-sequence**: 青=同期hold、黄=DynamoDB Streamsでキャッシュ更新、緑=非同期ワークフロー（fraud→モラトリアム→決済→通知）。
- **database-schema**: DynamoDB=`BALANCE_DDB`(available/held/version)+`LEDGER_DDB`(seq, balance_after, type=DEBIT/CREDIT/REVERSAL, 決済時のみ追記)。PostgreSQL=USERS/USER_IDENTIFIERS/ACCOUNT_META(balance_cache_minor)/TRANSFERS_META/TRANSFER_EVENTS。

---

## 6. 状態・未了事項

- **git**: 作業はコミット済み（最新 `88367dd remove escrow`）。ワーキングツリーはクリーン。ブランチは `main`。
- リモート upstream は gone（`git status` に出る）。push 先は要確認。
- 次に設計変更の指示が来たら: **README 2言語 + 該当する図ソース 2言語を更新 → PNG再生成 → escrow等の旧語が残っていないか grep で全ファイル確認**、という順序を守る（§4の検証を毎回やる）。
