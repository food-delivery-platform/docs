# Monitoring Service

**Runtime:** AWS Lambda (Node.js) · Scheduled (EventBridge cron every 5 min)  
**Domain:** Operations & SLA Tracking

## Responsibility

Detects orders stuck in a stage longer than SLA thresholds, publishes metrics to CloudWatch, and triggers alerts to the ops team.

## SLA Thresholds

| Stage | Max Allowed Time |
|-------|-----------------|
| `PENDING → PREPARING` | 30 sec (auto via payment) |
| `PREPARING → READY` | 30 min |
| `READY → PICKED_UP` | 15 min |
| `PICKED_UP → DELIVERED` | 60 min |

## Flow

```
EventBridge (every 5 min)
  → Monitoring Service Lambda
  → Reads order_events from DynamoDB
  → Checks stage age against SLA thresholds
  → CloudWatch PutMetricData (overdue_count)
  → CloudWatch Alarm → SNS → ops team (Email / SMS)
  → Ops Dashboard polls GET /api/alerts every 30 sec
```

## Key Operations

| Operation | Description |
|-----------|-------------|
| `check-sla-violations` | Scan `order_events` for overdue stage durations |
| `publish-alerts` | Write overdue count to CloudWatch; alert ops on threshold breach |
| `GET /api/alerts` | Endpoint polled by Ops Dashboard every 30 sec |

## Data Access

| Store | Table | Access |
|-------|-------|--------|
| DynamoDB | `order_events` | Read-only (scan stage transitions + timestamps) |
| CloudWatch | Metrics + Alarms | Write (PutMetricData) |

## Integrations

- **SQS Standard** (`monitoring-queue`) — also consumes `order.status.changed` events to keep DynamoDB `order_events` up to date
- **CloudWatch Alarms** — triggers ops notification via SNS when `overdue_count` exceeds threshold

## Alert Thresholds

| Metric | Alert |
|--------|-------|
| Overdue orders | > 50 simultaneously overdue |
| False-positive alarms | > 100/day |
