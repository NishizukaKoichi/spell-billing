# spell-billing 要件定義書（Usage Metering / Credits / Payments）

## 0. 文書情報

* 対象リポジトリ: `spell-billing`
* 目的: Spell システムにおける **利用量計測・課金計算・残高/上限管理・決済連携** を担う
* ステータス: 確定（本書が仕様の正本）

---

## 1. 背景 / 目的

`spell-billing` は、Spell 全体の「お金・上限・利用量」の単一の真実（Single Source of Truth）である。

* 実行（`spell-core`）が起こした **ユーザー行為** を課金可能な単位に落とし、価格を確定する
* **残高（credits）・上限（limits）・支払い状態**を管理し、実行可否判断に必要な情報を返す
* 外部決済（Stripe 等）と連携し、「トップアップ」「サブスク」「請求」「支払い失敗」などの状態を反映する
* 監査可能な台帳（ledger）を持ち、後から検証できるようにする

> 重要:
>
> * `spell-billing` は **実行ロジック（ジョブ/マルチリポ/LLM実行）を持たない**。
> * `spell-billing` は **Intent の同意・署名の検証をしない**（それは `spell-intent` の責務）。
> * `spell-billing` は「この行為が発生した」「どれだけ消費した」を受け取り、**課金計算と残高/上限管理**を行う。

---

## 2. スコープ（やること / やらないこと）

### 2.1 やること（In Scope）

1. **価格決定**

* `op`（操作種別）と `meters`（測定値）からコスト（credits）を計算する
* 同一 `op` の計算ルールを版管理し、将来変更しても監査可能にする

2. **残高（Credits）管理**

* ユーザーごとの `balance`（利用可能）と `reserved`（確保中）を管理する
* 実行前に「この操作に必要な最大コスト」を **予約（reserve）** できる
* 実行後に実コストで **確定（capture）** し、差額を解放する

3. **上限/利用制限**

* プランごとの月次上限、無料枠、機能制限（必要なら）を管理する
* 支払い失敗・凍結などの **課金状態**で実行可否を左右する

4. **決済連携（外部）**

* Stripe 等でのトップアップ、サブスク、請求状態を取り込み、credits 付与・状態更新を行う（Webhook 受信含む）

5. **可観測性/監査**

* 台帳（ledger）とイベント（usage）を永続化し、後日追跡・再計算検証ができる
* 重複送信に耐える（idempotent）

6. **参照 API**

* `spell-core` や（必要なら）クライアントが「残高」「利用量」「請求状態」を取得できる

---

### 2.2 やらないこと（Out of Scope / 非ゴール）

* 「ユーザー本人か」の認証（`spell-auth` の責務）
* Intent 署名の検証（`spell-intent` の責務）
* 実行計画の生成やジョブ実行（`spell-core` / `spell-runner` の責務）
* 税務・会計上の最終帳票（インボイスPDF生成等）は v1+ で別途（MVPでは不要）
* 不正検知の高度化（ML等）は v1+（MVPではルールベースのみ）

---

## 3. システム境界 / 依存 / 連携

### 3.1 依存

* DB: PostgreSQL（必須）
* 外部決済: Stripe（v0 では「Webhook受信＋最低限の状態反映」までを必須要件とする）

### 3.2 主要な連携相手と責務

* `spell-core`

  * 実行前: 「この intent を実行できるか？」の照会／予約
  * 実行後: 測定値（meters）を添えてコスト確定を通知
* `spell-auth`

  * ユーザーJWTの検証情報（JWKS）利用（**ユーザー向けAPIを開ける場合**）
* `spell-client`

  * 原則 `spell-core` 経由で課金情報を表示（直叩きは必須ではない）

> 決め打ち（ブレ防止）:
>
> * **課金の実行パスは `spell-core → spell-billing` のみ**を必須とする。
> * クライアントが `spell-billing` を直接叩く I/F は “あっても良い” ではなく、**v0 は不要**として固定する。
>   （ユーザー表示は `spell-core` が取りに行って返す）

---

## 4. コア概念（課金モデル）

### 4.1 Credits モデル（採用）

`spell-billing` は **credits（整数）** を用いたプリペイド型の課金モデルを正とする。

* credit は最小単位の整数（例: 1 credit = 1 単位）
* 料金計算は `op` と `meters` から **cost_credits** を算出し、残高から差し引く
* 実行前に最大消費を **予約**し、実行後に実消費で **確定**する

### 4.2 Reserve/Capture フロー（必須）

* **Reserve（予約）**: 実行前に `max_cost_credits` を確保（残高から引当）
* **Capture（確定）**: 実行後に `meters` を渡して実コストを確定（予約から差し引き、余りは解放）
* **Expire（期限切れ）**: 予約が一定時間で期限切れになったら自動解放

---

## 5. 機能要件（Functional Requirements）

### FR-1 価格計算（Pricing Engine）

* 入力: `op` + `meters` +（必要なら）pricing_version
* 出力: `cost_credits` と計算内訳（監査用）
* 価格定義は DB（price_catalog）で管理し、**変更履歴を保持**する

### FR-2 残高（Wallet）

* ユーザーごとに `available_credits` と `reserved_credits` を持つ
* 残高は ledger から再構築可能であること（監査性）

### FR-3 予約（Reserve）

* 実行前に `intent_id` 単位で `max_cost_credits` を予約できる
* 既に同じ `intent_id` が予約済みの場合は **idempotent** に同じ結果を返す
* 残高不足の場合は `allowed=false` を返す

### FR-4 確定（Capture）

* 実行後に `meters` とともに cost を確定できる
* `intent_id` ごとに **一度だけ capture**（重複は idempotent）
* 予約額より実コストが小さい場合は差額を解放
* 予約額より実コストが大きい場合は **上限を予約額にクリップ**（=`max_cost` の保証）

  * これにより「想定外の過請求」を原理的に防ぐ

### FR-5 解放（Release / Expire）

* 実行がキャンセルされた場合に手動 release できる
* 予約には TTL があり、期限切れは自動 expire で解放される

### FR-6 利用量・台帳（Usage/Ledger）

* すべての reserve/capture/release/topup/refund を ledger として永続化する
* 監査のため、関連する `op` `meters` `intent_id` `pricing_version` を保存する

### FR-7 プラン・課金状態（Entitlements）

* ユーザーはプラン（free/pro 等）を持ち、以下を持てる:

  * 月次で付与される credits
  * 上限（hard limit / soft limit）
  * 支払い状態（active / past_due / blocked）
* `spell-core` が実行前に「allowed/blocked と理由」を取得できる

### FR-8 決済連携（Stripe）

* Webhook を受け取り、以下を反映できる:

  * サブスク開始/更新/解約 → plan/status 更新
  * 支払い成功 → credits 付与 or status active
  * 支払い失敗 → status past_due/blocked
  * トップアップ購入 → credits 付与

> v0 必須範囲: Webhook受信・署名検証・最低限の状態更新/credits付与

### FR-9 管理操作（Admin）

* 運用者がユーザーの credits を調整できる（付与/剥奪）
* 調整も ledger に記録し監査可能にする

---

## 6. 外部インターフェース（API 仕様：v0必須）

`spell-billing` の **必須 API はすべて内部用**（`spell-core` からのみ呼ばれる）とする。
認証は **サービス間認証**（後述）を必須。

### 6.1 共通仕様

* HTTPS 必須
* JSON のみ
* **Idempotency-Key** を必須（ヘッダ）

  * 同一 intent を複数回送っても二重請求しないため

#### 共通レスポンス（推奨）

```json
{
  "ok": true,
  "request_id": "uuid"
}
```

エラー（推奨）

```json
{
  "ok": false,
  "error": { "code": "insufficient_credits", "message": "..." },
  "request_id": "uuid"
}
```

---

### 6.2 実行前：Authorize（=Reserve）

#### `POST /internal/billing/authorize`

**目的**: 実行前に最大消費 credits を予約し、実行可否を返す

**入力**

```json
{
  "user_id": "uuid",
  "intent_id": "uuid",
  "op": "string",
  "max_cost_credits": 123,
  "currency": "CREDITS",
  "occurred_at": "2025-12-05T00:00:00Z"
}
```

**出力**

```json
{
  "ok": true,
  "allowed": true,
  "authorization_id": "uuid",
  "reserved_credits": 123,
  "wallet": { "available_credits": 1000, "reserved_credits": 123 },
  "pricing_version": 1
}
```

**前提条件**

* `spell-core` からのサービス間認証が通ること
* `user_id` が存在（存在しない場合は作成 or 404—v0は「存在しないなら作成」を推奨）

**制約**

* `intent_id` はユニーク（同一 intent の二重予約は禁止）
* `max_cost_credits` 以上は絶対に引かない（安全性の上限）

**成功条件**

* 残高が十分なら予約が作られ `allowed=true`
* 不足なら `allowed=false` を返し、予約を作らない

---

### 6.3 実行後：Capture（確定）

#### `POST /internal/billing/capture`

**目的**: 実行結果と測定値を受け、実コストを確定し差額を解放する

**入力**

```json
{
  "authorization_id": "uuid",
  "intent_id": "uuid",
  "status": "succeeded",
  "meters": {
    "llm_tokens_in": 1234,
    "llm_tokens_out": 567,
    "duration_ms": 890,
    "repo_count": 3
  },
  "occurred_at": "2025-12-05T00:02:00Z"
}
```

**出力**

```json
{
  "ok": true,
  "captured_credits": 100,
  "released_credits": 23,
  "wallet": { "available_credits": 900, "reserved_credits": 0 },
  "pricing": { "version": 1, "breakdown": { "base": 10, "tokens": 90 } }
}
```

**前提条件**

* `authorization_id` が存在し、未captureであること

**制約**

* 実コストは pricing engine により算出する
* 実コストが予約額を超えた場合、**予約額にクリップ**する
* capture は idempotent（同じ intent で何度叩かれても二重差し引きしない）

**成功条件**

* ledger に capture が記録され、wallet が更新される
* reserved が 0 になっている（差額解放含む）

---

### 6.4 実行キャンセル：Release（手動解放）

#### `POST /internal/billing/release`

**目的**: 実行取りやめ等で予約を解放する

**入力**

```json
{ "authorization_id": "uuid", "reason": "canceled" }
```

**出力**

```json
{
  "ok": true,
  "released_credits": 123,
  "wallet": { "available_credits": 1000, "reserved_credits": 0 }
}
```

**成功条件**

* 予約が解放され wallet が戻る

---

### 6.5 参照：Wallet / Entitlements

#### `GET /internal/billing/users/{user_id}/status`

**目的**: 実行前判断に必要な「残高・状態・上限」を返す

**出力**

```json
{
  "user_id": "uuid",
  "billing_status": "active",
  "plan": "free",
  "wallet": { "available_credits": 1000, "reserved_credits": 0 },
  "limits": { "monthly_credits_cap": 5000 }
}
```

---

### 6.6 Stripe Webhook（v0必須）

#### `POST /api/billing/webhooks/stripe`

* Stripe 署名を検証し、subscription/payment/topup を反映する
* Webhookは **冪等**（同一イベント二重でも壊れない）
* 反映結果は ledger に記録する

---

## 7. 価格計算（Pricing Engine）要件

### 7.1 入力（meters）

最低限、以下の測定値を受け取れること：

* LLM: `llm_tokens_in`, `llm_tokens_out`
* 実行: `duration_ms`
* マルチリポ: `repo_count`, `files_changed`（将来）

### 7.2 価格定義

* `op` ごとに pricing rule を持つ（固定費＋単価×メーター等）
* pricing rule は DB に保存し、`pricing_version` を持つ
* capture 時に「どの pricing_version で計算したか」を保存する（監査可能）

---

## 8. データモデル（DB 要件：v0必須）

最低限これを持つ（監査性と冪等性のため必須）。

### 8.1 wallets

* `user_id` (PK)
* `available_credits` (bigint)
* `reserved_credits` (bigint)
* `updated_at`

### 8.2 billing_authorizations

* `id` (PK) = authorization_id
* `user_id`
* `intent_id` (unique)
* `op`
* `reserved_credits`
* `status` enum: `reserved | captured | released | expired`
* `pricing_version`
* `expires_at`
* `created_at`, `updated_at`

### 8.3 billing_ledger

* `id` (PK)
* `user_id`
* `intent_id` nullable
* `authorization_id` nullable
* `type` enum: `reserve | capture | release | topup | refund | admin_adjust`
* `delta_credits` (signed bigint)
* `metadata` (jsonb) 例: meters/pricing breakdown/stripe ids
* `created_at`

### 8.4 price_catalog

* `id`
* `op` (unique)
* `pricing_version`
* `rule` (jsonb) 例: base, token_unit_price, etc
* `effective_from`
* `created_at`

### 8.5 billing_accounts（決済状態）

* `user_id` (PK)
* `status` enum: `active | past_due | blocked`
* `plan` text
* `stripe_customer_id` nullable
* `stripe_subscription_id` nullable
* `current_period_end` nullable
* `updated_at`

---

## 9. セキュリティ要件（NFR）

### 9.1 サービス間認証（必須）

`/internal/*` は **`spell-core` からのみ**受け付ける。

* `Authorization: Bearer <service_token>` を必須
* `service_token` は以下を満たす JWT を推奨:

  * `iss = spell-core`
  * `aud = spell-billing`
  * `exp` 短命（例: 5分）
* `spell-billing` は `spell-core` の公開鍵（JWKS または静的公開鍵）で検証する

### 9.2 入力検証

* 全APIで schema validation を必須（Zod 等）
* `intent_id` / `authorization_id` を冪等キーとして利用
* meters の上限（異常値）をガードする（例: tokens > 10^8 は拒否）

### 9.3 ログ秘匿

* 生の決済情報（カード情報等）は扱わない（Stripe に委譲）
* トークン・秘密鍵はログ禁止
* request_id を全応答につけ追跡可能にする

---

## 10. 運用要件

### 10.1 可用性 / 性能

* `authorize` / `capture` の p95 < 300ms を目標（DB込み）
* DB トランザクションで wallet 更新の整合性を保証（競合耐性）

### 10.2 冪等性（絶対要件）

* `authorize` / `capture` / `release` / webhook は冪等であること
* 送信側がリトライしても **二重請求・二重付与が起きない**こと

### 10.3 バックアップ / 監査

* ledger は削除しない（論理削除も原則しない）
* 監査のため最低 1 年保持（期間は運用で設定）

---

## 11. 受け入れ基準（Acceptance Criteria）

v0 の受け入れは以下で判定する：

1. authorize

* 残高十分：`allowed=true`、予約が作られ reserved が増える
* 残高不足：`allowed=false`、予約が作られない

2. capture

* 実行後 meters を渡すと cost が確定し、wallet が減り reserved が解放される
* 二重 capture しても二重に引かれない（冪等）

3. release / expire

* キャンセルで予約が解放される
* TTL 後に expire 処理で解放される

4. pricing

* `op` と meters から妥当な cost が出る
* pricing_version と breakdown が ledger に残る

5. stripe webhook

* 署名検証できる
* topup/subscription 状態が反映される
* 同一 webhook リトライでも二重付与されない

6. 監査

* reserve/capture/release/topup が ledger で追跡できる

---

## 12. 必須環境変数（例）

* `DATABASE_URL`
* `BILLING_ISSUER`（例: `spell-billing`）
* `CORE_JWKS_URL` or `CORE_PUBLIC_KEY_PEM`（サービス間JWT検証用）
* `STRIPE_WEBHOOK_SECRET`
* `DEFAULT_PRICING_VERSION`
* `AUTH_JWKS_URL`（ユーザー向けAPIを将来開ける場合のみ）

---

以上が **`spell-billing` の要件定義書（確定版）**です。
次に「機能仕様書」まで一気に落とすなら、ここから

* `authorize/capture/release/status` の **入出力とエラーコード一覧**
* pricing rule の JSON スキーマ（rule DSL）
* DB トランザクション設計（walletの整合性）
  を固定すると、実装がさらに速く・ブレずに進みます。
