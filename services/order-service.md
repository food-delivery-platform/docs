# Order Service

**Runtime:** AWS Lambda (Node.js) · AWS Step Functions Express Workflow  
**Domain:** Order Processing

## Responsibility

Owns the full order lifecycle from creation to terminal state. Orchestrates payment, delivery handoff, and event publishing via a Step Functions state machine.

## State Machine

```
PENDING → PREPARING (auto, on payment) → READY → PICKED_UP → DELIVERED
                                       ↘ CANCELLED / FAILED
```

## Key Operations

| Operation | Trigger |
|-----------|---------|
| Validate items | Catalog Service REST call |
| Create order (DynamoDB) | Customer POST /orders |
| Initiate payment | Payment Service REST call |
| Wait for payment callback | WaitForTaskToken (Stripe webhook) |
| Publish `order.preparing` to SNS | On payment confirmed |
| Archive to Supabase | On terminal state (DELIVERED / CANCELLED / FAILED) |

## Data Owned

| Store | Table | Role |
|-------|-------|------|
| DynamoDB | `active_orders` | Live state during active lifecycle (TTL auto-cleanup) |
| DynamoDB | `order_events` | Immutable audit log of every transition |
| Supabase | `orders`, `order_items` | Permanent record, written once on terminal state |

## Integrations

- **Catalog Service** — validates item availability before order creation
- **Payment Service** — creates Stripe payment intent; receives webhook callback
- **Delivery Service** — requests ETA at checkout; receives delivery updates
- **SNS Standard** (`order-events`) — fan-out to Delivery, Notification, Monitoring

## Concurrency

| Baseline | Lunch Peak Burst |
|----------|-----------------|
| 200 concurrent | 500 concurrent |

P99 latency alert: > 2 sec · Error rate alert: > 5%
