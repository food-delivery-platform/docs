# Notification Service

**Runtime:** AWS Lambda (Node.js) · SQS subscriber  
**Domain:** Notifications

## Responsibility

Sends order status notifications to customers, restaurants, and couriers via SMS, email, and push — triggered by events from the `order-events` SNS topic.

## Trigger Flow

```
Order Service → SNS Standard (order-events)
             → SQS Standard (notification-queue)
             → Notification Service Lambda
             → SNS Standard (SMS / Email / Push to end users)
```

## Notification Events

| Event | Recipient | Channel |
|-------|-----------|---------|
| `order.preparing` | Customer | Push + Email |
| `order.ready` | Courier (eligible) | Push |
| `delivery.courier_assigned` | Customer | Push + SMS |
| `delivery.picked_up` | Customer | Push |
| `delivery.delivered` | Customer | Push + Email |
| `order.cancelled` | Customer | Push + Email + SMS |

## Data Owned

| Store | Table | Role |
|-------|-------|------|
| Supabase | `notification_log` | Audit log of sent notifications (audit only, not operational) |

## Integrations

- **SQS Standard** (`notification-queue`) — consumes events; DLQ after 3 failures
- **SNS Standard** (SMS / Email / Push topic) — delivers to end users
- **Order Service / User Service** — fetches user contact details via REST when needed

## Resilience

- SNS Standard handles retries (up to 15 min) before moving to DLQ
- DLQ captures permanent failures for manual inspection
- Duplicate events handled idempotently (check `notification_log` before resending)

## Concurrency

| Baseline | Burst |
|----------|-------|
| 200 concurrent | 500 concurrent |

Alert: > 1% send failure rate · > 5 min delivery delay
