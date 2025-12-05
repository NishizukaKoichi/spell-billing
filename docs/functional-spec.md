# spell-billing 機能仕様書（Credits / Usage Metering / Payments）

## 0. 文書情報

* 対象リポジトリ: `spell-billing`
* 文書種別: 機能仕様書
* 目的: Spellの**利用量計測・課金計算・残高/上限制御・決済連携**を提供する
* ステータス: 確定（本書が実装の正本）

---

## 1. 概要（What）

`spell-billing` は、Spellシステムの「お金」と「上限」を司るサービスである。
入力として **(user_id, intent_id, op, max_cost, meters, 決済イベント)** を受け取り、出力として **(allowed/blocked, creditsの予約・確定、残高、課金状態、台帳)** を提供する。

---

## 2. 非ゴール（Non-goals）

`spell-billing` は以下を**行わない**：

* ユーザー認証（JWTの発行、Passkey等） … `spell-auth` の責務
* Intent署名検証/同意の最終判断 … `spell-intent` の責務
* 実行計画生成、ジョブ実行、マルチリポ実行 … `spell-core` / `spell-runner` の責務
* 会計上の最終帳票（インボイスPDF等）の完全実装 … v0では対象外

---

## 3. 前提条件（Prerequisites）

* DB: PostgreSQL が利用可能であること（永続化必須）
* `spell-core` からの呼び出しは **サービス間認証**ができること（後述）
* 決済連携を有効にする場合、Stripe Webhook の署名検証に必要な secret が設定されていること
* `intent_id` は、`spell-core` が生成する**一意なID**であること（冪等性の根）

---

## 4. グローバル制約（Constraints）

### 4.1 API共通

* HTTPS必須
* JSONのみ（`Content-Type: application/json`）
* すべての POST は **Idempotency-Key** ヘッダ必須

  * 目的: リトライに対して二重請求/二重付与を防止

### 4.2 サービス間認証（内部API必須）

`/internal/*` は `spell-core` のみ呼べること。

* ヘッダ: `Authorization: Bearer <service_jwt>`
* 必須検証:

  * 署名検証（`spell-core` の公開鍵/JWKS）
  * `iss = "spell-core"`
  * `aud = "spell-billing"`
  * `exp` 短命（例: 5分以内）

### 4.3 課金安全性（過請求防止）

* `capture` で算出される実コストは **予約額を上限としてクリップ**する

  * `captured_cost = min(calculated_cost, reserved_cost)`
* これにより “想定外の過請求” を構造的に発生させない

### 4.4 Wallet整合性（必須不変条件）

* `available_credits >= 0` かつ `reserved_credits >= 0`
* 予約（reserve）・確定（capture）・解放（release/expire）は **DBトランザクション**で行う

---

## 5. ドメインモデル（Core Concepts）

* **Credits**: 整数の利用単位（プリペイド）
* **Wallet**: ユーザーの残高状態

  * `available_credits`（利用可能）
  * `reserved_credits`（確保中）
* **Authorization**: 実行前の予約（intent単位）
* **Ledger**: 台帳（reserve/capture/release/topup/refund/admin調整を全記録）
* **Pricing**: `op` と `meters` から cost を計算するルール（版管理）
* **Billing Status / Plan**: `active/past_due/blocked` などの課金状態・プラン

---

## 6. 機能仕様（Functions）

### F1. 実行前の可否判定＋予約（Authorize = Reserve）

#### API

`POST /internal/billing/authorize`

#### 入力（Input）

**Headers**

* `Authorization: Bearer <service_jwt>`（必須）
* `Idempotency-Key: <uuid-or-string>`（必須）

**Body**

```json
{
  "user_id": "uuid",
  "intent_id": "uuid",
  "op": "string",
  "max_cost_credits": 123,
  "occurred_at": "2025-12-05T00:00:00Z"
}
```

#### 出力（Output）

成功（予約成立 / allowed=true）

```json
{
  "ok": true,
  "allowed": true,
  "authorization_id": "uuid",
  "reserved_credits": 123,
  "pricing_version": 1,
  "wallet": { "available_credits": 1000, "reserved_credits": 123 }
}
```

成功（不足 / allowed=false）

```json
{
  "ok": true,
  "allowed": false,
  "reason": "insufficient_credits",
  "wallet": { "available_credits": 10, "reserved_credits": 0 }
}
```

#### 前提条件（Prerequisites）

* サービス間認証が通ること
* `intent_id` が一意であること（同一 intent の再送は冪等扱い）

#### 制約（Constraints）

* 予約は **intent_id 単位で一つ**（二重予約禁止）
* `max_cost_credits` は 0 以上の整数
* `billing_status = blocked` の場合は `allowed=false` を返す（予約は作らない）

#### 成功条件（Success Criteria）

* allowed=true の場合

  * `billing_authorizations` が作成され `status=reserved`
  * wallet が

    * `available -= max_cost`
    * `reserved += max_cost`
  * ledger に `reserve` が記録される
* allowed=false の場合

  * wallet/ledger に変更を加えない

---

### F2. 実行後の確定（Capture）

#### API

`POST /internal/billing/capture`

#### 入力（Input）

**Headers**

* `Authorization: Bearer <service_jwt>`（必須）
* `Idempotency-Key`（必須）

**Body**

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

#### 出力（Output）

```json
{
  "ok": true,
  "captured_credits": 100,
  "released_credits": 23,
  "wallet": { "available_credits": 900, "reserved_credits": 0 },
  "pricing": {
    "version": 1,
    "breakdown": { "base": 10, "tokens": 90 }
  }
}
```

#### 前提条件

* `authorization_id` が存在し、`status=reserved` であること
* Authorization の `intent_id` と入力 `intent_id` が一致すること

#### 制約

* 実コスト算出: `calculated_cost = pricing(op, meters, version)`
* **過請求防止**: `captured_credits = min(calculated_cost, reserved_credits)`
* `released_credits = reserved_credits - captured_credits`
* capture は **1回のみ**（2回目以降は冪等で同一結果を返す）

#### 成功条件

* authorization が `captured` に更新される
* wallet が

  * `reserved -= reserved_credits`（予約分を解消）
  * `available += released_credits`（差額を返す）
* ledger に

  * `capture`（差し引き）
  * （必要なら）差額解放情報を metadata に保存
* pricing_version と breakdown と meters が ledger に残る（監査可能）

---

### F3. 実行キャンセル時の解放（Release）

#### API

`POST /internal/billing/release`

#### 入力

**Headers**

* `Authorization`（必須）
* `Idempotency-Key`（必須）

**Body**

```json
{ "authorization_id": "uuid", "reason": "canceled" }
```

#### 出力

```json
{
  "ok": true,
  "released_credits": 123,
  "wallet": { "available_credits": 1000, "reserved_credits": 0 }
}
```

#### 前提条件

* authorization が `reserved` であること（すでに captured のものは冪等で同結果/または no-op）

#### 制約

* release は冪等
* wallet が

  * `reserved -= reserved_credits`
  * `available += reserved_credits`
* ledger に `release` を記録

#### 成功条件

* 予約が解放され、残高が戻る
* 監査ログ的に追跡できる

---

### F4. 予約の期限切れ解放（Expire）

※外部APIではなく **内部処理**として必須機能

#### 入力

* DB上の `billing_authorizations`（`status=reserved` かつ `expires_at < now`）

#### 出力

* `status=expired` に更新
* wallet の予約分を解放
* ledger に `expire` を記録

#### 前提条件

* authorization に `expires_at` が設定されていること

#### 制約

* expire は冪等（同一authorizationに対して二重解放しない）
* 実行は定期ジョブで行う（実装方法はサービスの運用に従う）

#### 成功条件

* “予約が残り続けて残高を食い潰す” 状態を防げる

---

### F5. ユーザー状態参照（残高・課金状態・上限）

#### API

`GET /internal/billing/users/{user_id}/status`

#### 入力

**Headers**

* `Authorization`（必須）

#### 出力

```json
{
  "user_id": "uuid",
  "billing_status": "active",
  "plan": "free",
  "wallet": { "available_credits": 1000, "reserved_credits": 0 },
  "limits": { "monthly_credits_cap": 5000 }
}
```

#### 前提条件

* user_id が存在（存在しない場合の扱いは v0 では「初回参照で作成」でもよいが、実装はこの仕様に合わせて固定すること）

#### 制約

* 参照系は write を伴わない（例外を作らない）

#### 成功条件

* `spell-core` が実行前判断に必要な情報を取得できる

---

### F6. Stripe Webhook 受信（決済状態・付与の反映）

#### API

`POST /api/billing/webhooks/stripe`

#### 入力

**Headers**

* Stripe署名検証に必要なヘッダ（Stripe規約通り）
* `Idempotency-Key`（必須：stripe_event_id を推奨）

**Body**

* Stripe event payload（そのまま）

#### 出力

```json
{ "ok": true }
```

#### 前提条件

* Webhook署名検証が通ること（`STRIPE_WEBHOOK_SECRET`）

#### 制約

* **冪等**：同一イベントの再送で二重付与しない

  * stripe_event_id を記録して二重処理防止
* 反映は ledger に残す（topup/refund/subscription change など）

#### 成功条件

* 支払い成功 → credits付与 or status active
* 支払い失敗 → past_due/blocked へ遷移（仕様で固定）
* サブスク更新/解約 → plan/status が更新される

---

### F7. 管理者による手動調整（Admin Adjust）

（内部運用向け。v0でも必要）

#### API

`POST /internal/billing/admin/adjust`

#### 入力

**Headers**

* `Authorization`（管理者用。運用で別途定義）
* `Idempotency-Key`（必須）

**Body**

```json
{
  "user_id": "uuid",
  "delta_credits": 500,
  "reason": "support_grant"
}
```

#### 出力

```json
{
  "ok": true,
  "wallet": { "available_credits": 1500, "reserved_credits": 0 }
}
```

#### 前提条件

* 管理者認証が通ること

#### 制約

* delta は signed
* ledger に `admin_adjust` を必ず残す

#### 成功条件

* 監査可能な形で残高調整ができる

---

## 7. Pricing Engine（価格計算）仕様

### 入力

* `op: string`
* `meters: object`
* `pricing_version: int`

### 出力

* `calculated_cost_credits: int`
* `breakdown: object`（監査用）

### 制約

* `price_catalog` に `op + pricing_version` が存在しない場合はエラー（`pricing_not_found`）
* meters は上限を持つ（異常値は拒否）
  例: tokens > 1e8 は `invalid_meters`

### 成功条件

* capture 時に必ず「どの version/ルールで計算したか」が記録される

---

## 8. データ保存（最低限の必須テーブル）と整合性

### 必須テーブル

* `wallets`
* `billing_authorizations`
* `billing_ledger`
* `price_catalog`
* `billing_accounts`（plan/status/stripe ids）

### 整合性（必須）

* reserve/capture/release/expire は単一トランザクションで wallet と authorization と ledger を更新する
* `intent_id` は authorization で unique（冪等の核）

---

## 9. エラーコード（固定）

* `invalid_request`
* `unauthorized`
* `forbidden`
* `insufficient_credits`
* `billing_blocked`
* `authorization_not_found`
* `authorization_already_captured`
* `authorization_expired`
* `pricing_not_found`
* `invalid_meters`
* `idempotency_conflict`
* `stripe_signature_invalid`

---

## 10. 全体の成功条件（Acceptance / Done）

* authorize が残高・状態に応じて適切に allowed を返し、予約が整合的に残る
* capture が meters から計算し、**予約額上限でクリップ**しつつ確定できる
* release/expire で予約が確実に解放される
* すべての金銭的変化が ledger に残り、後から追跡できる
* Stripe webhook が署名検証され、冪等に反映される
* `/internal/*` がサービス間認証なしでは呼べない

---

必要なら次の一手として、この機能仕様に完全一致する **OpenAPI（internal用）** と **DBトランザクションの擬似コード（reserve/capture）** まで落として、実装者が迷いようのない形に仕上げられる。
