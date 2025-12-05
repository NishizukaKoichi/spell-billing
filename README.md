# spell-billing

**Billing & Usage Tracking**

## 責務

**お金とレート制御の責務を分離**

- Intent 単位の利用記録
- プラン / 上限 / レート制御
- Stripe など外部決済連携
- 利用統計・レポート

Spell 本体のロジックと金回りを切り離すための場所

## Architecture

```
┌───────────────┐
│  spell-core   │
└───────┬───────┘
        │ usage event
        ▼
┌────────────────────┐
│  spell-billing     │
│                    │
│ - Usage tracker    │
│ - Plan manager     │
│ - Rate limiter     │
│ - Payment gateway  │
└──────────┬─────────┘
           │
           └──────► Stripe API
```

## Features

### Usage Tracking

- Intent execution count
- Resource consumption (LLM tokens, build minutes, storage)
- Per-user aggregation

### Plans & Quotas

- Free tier limits
- Paid plan benefits
- Custom enterprise quotas

### Rate Limiting

- Requests per minute/hour/day
- Concurrent job limits
- Fair usage enforcement

### Billing Integration

- Stripe subscription management
- Invoice generation
- Payment method handling
- Webhook processing

## API Endpoints

- `POST /usage` - Record usage event (internal)
- `GET /usage/:userId` - Get user usage stats
- `GET /plans` - List available plans
- `POST /subscribe` - Create subscription
- `POST /webhook` - Stripe webhook handler

## Tech Stack

- Node.js / TypeScript
- PostgreSQL (usage records, subscriptions)
- Redis (rate limiting)
- Stripe SDK
