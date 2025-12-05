# Plan: spell-billing

## 実装計画

このファイルは「完遂モード」で AI に渡す実装計画書です。

---

## 目的

spell-billing リポジトリの初期実装を完遂する。

---

## 実装フェーズ

### Phase 1: 環境セットアップ

- [ ] package.json に必要な依存関係を追加
  - `express`, `stripe`, `pg`, `ioredis`, `zod` など
- [ ] TypeScript 設定 (tsconfig.json)
- [ ] ESLint / Prettier 設定
- [ ] Docker Compose (PostgreSQL, Redis)
- [ ] .env.example 作成

### Phase 2: データベーススキーマ

- [ ] usage_events テーブル
  - id (UUID, PK)
  - user_id (UUID)
  - intent_op (string)
  - resource_consumed (JSONB)
  - created_at
- [ ] subscriptions テーブル
  - id (UUID, PK)
  - user_id (UUID, UNIQUE)
  - stripe_customer_id
  - stripe_subscription_id
  - plan (free/pro/enterprise)
  - status (active/canceled/past_due)
  - created_at, updated_at
- [ ] plans テーブル
  - id (UUID, PK)
  - name (free/pro/enterprise)
  - limits (JSONB)
  - price_monthly

### Phase 3: Usage Tracking API 実装

- [ ] POST /usage (内部 API)
  - usage イベントを記録
  - レート制限チェック
- [ ] GET /usage/:userId
  - ユーザーの利用統計を返す

### Phase 4: Subscription API 実装

- [ ] GET /plans
  - プラン一覧を返す
- [ ] POST /subscribe
  - Stripe サブスクリプション作成
  - DB に保存
- [ ] POST /webhook
  - Stripe webhook ハンドラー
  - サブスクリプションステータス更新

### Phase 5: レート制限

- [ ] Redis でレート制限実装
  - リクエスト数 / 分
  - 同時実行ジョブ数
  - Fair usage enforcement

### Phase 6: Stripe 連携

- [ ] Customer 作成
- [ ] Subscription 作成
- [ ] Webhook 処理 (invoice.paid, subscription.updated など)

### Phase 7: テスト

- [ ] ユニットテスト (各エンドポイント)
- [ ] Stripe webhook テスト
- [ ] レート制限テスト

### Phase 8: ドキュメント

- [ ] API ドキュメント
- [ ] プラン設定方法
- [ ] 環境変数リスト

---

## ファイル構成（想定）

```
spell-billing/
├── src/
│   ├── server.ts              # Entry point
│   ├── config/
│   │   ├── db.ts              # PostgreSQL client
│   │   ├── redis.ts           # Redis client
│   │   └── stripe.ts          # Stripe client
│   ├── routes/
│   │   ├── usage.ts           # POST /usage, GET /usage/:userId
│   │   ├── plans.ts           # GET /plans
│   │   ├── subscribe.ts       # POST /subscribe
│   │   └── webhook.ts         # POST /webhook
│   ├── services/
│   │   ├── rate-limit.ts      # Rate limiting logic
│   │   └── stripe.ts          # Stripe operations
│   ├── models/
│   │   ├── usage.ts           # Usage event model
│   │   ├── subscription.ts    # Subscription model
│   │   └── plan.ts            # Plan model
│   └── schemas/
│       └── usage.ts           # Zod schemas
├── tests/
│   ├── usage.test.ts
│   ├── subscribe.test.ts
│   ├── webhook.test.ts
│   └── rate-limit.test.ts
├── migrations/
│   ├── 001_initial.sql
│   └── 002_seed_plans.sql
├── docker-compose.yml
├── .env.example
├── tsconfig.json
├── package.json
└── README.md
```

---

## 技術的決定事項

- **決済ライブラリ**: stripe
- **DB クライアント**: pg (node-postgres)
- **Redis クライアント**: ioredis
- **バリデーション**: zod
- **テストフレームワーク**: vitest
- **HTTP フレームワーク**: express

---

## 注意事項

- Stripe webhook は署名検証必須
- レート制限は Redis で実装（sliding window 方式）
- プランの limits は JSONB で柔軟に管理
- 無料プランのデフォルト limits を設定
- Stripe API キーは環境変数で設定

---

## 完遂モード実行時の指示

ticket.md のチケット内容と、この plan.md を参照して、上記フェーズを全て実装してください。
不明点は合理的な前提を置いて実装を完了させてください。
