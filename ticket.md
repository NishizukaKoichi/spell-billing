# Ticket: spell-billing

## チケット情報

- **リポジトリ**: spell-billing
- **技術スタック**: TypeScript / Node.js / Express / Stripe / PostgreSQL / Redis
- **仕様書パス**: ../README.md, ./README.md
- **ビルドコマンド**: `pnpm build`
- **テストコマンド**: `pnpm test`
- **その他コマンド**: `pnpm lint`

---

## 実装対象チケット

（ここにチケット本文を貼り付けてください）

例:
```
[SPELL-BILLING-001] 利用量トラッキング + Stripe 連携実装

## 概要
Intent 実行ごとの利用量を記録し、Stripe でサブスクリプション管理を行う。

## 受け入れ条件
- [ ] POST /usage - 利用イベント記録（内部 API）
- [ ] GET /usage/:userId - ユーザー利用統計取得
- [ ] GET /plans - プラン一覧取得
- [ ] POST /subscribe - サブスクリプション作成
- [ ] POST /webhook - Stripe webhook ハンドラー
- [ ] レート制限チェック機能

## 技術仕様
- Stripe SDK 使用
- PostgreSQL で利用記録管理
- Redis でレート制限実装
```

---

## 補足情報

- 不明点は合理的な前提を置いて実装してください
- 既存コードスタイルに従ってください
- 破壊的変更が必要な場合は理由を明記してください
