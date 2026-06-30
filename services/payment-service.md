# Payment Service

**Runtime:** AWS Lambda (Node.js)  
**Domain:** Payment

## Responsibility

Manages the full payment lifecycle: create Stripe payment intent, confirm payment via webhook, and trigger refunds on cancellation.

## Key Operations

| Operation | Endpoint |
|-----------|----------|
| Create payment intent | `POST /payments/intent` |
| Confirm payment (Stripe webhook) | `POST /payments/confirm` |
| Refund | `POST /payments/refund` |

## Flow

```
Order Service → create intent → Stripe
Stripe → webhook → API Gateway → confirm callback → Order Service (Step Functions task token)
On CANCELLED → refund flow
```

## Data Owned

| Store | Table | Role |
|-------|-------|------|
| Supabase | `payments` | Payment records (provider_ref, status, amount) |

## Integrations

- **Stripe API** — payment intent, confirmation, idempotency_key = `order_id`
- **SNS Standard** (`order-events`) — publishes `payment.confirmed` / `payment.failed`
- **Order Service** — sends Step Functions task token callback on payment result

## Resilience

- Idempotency: Stripe `idempotency_key` prevents double-charging
- Timeout: 60 sec Lambda timeout; max 3 retries on network errors
- Fail-open on sustained Stripe timeout → manual reconciliation queue
- Throttle behavior: return 429 immediately (not queued)

## Concurrency

| Baseline | Burst |
|----------|-------|
| 50 concurrent | 100 concurrent |
