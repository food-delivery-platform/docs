# Delivery Service

**Runtime:** AWS Fargate (Node.js + Express) · 1–4 tasks (auto-scale on CPU > 70%)  
**Domain:** Delivery Coordination

## Why Fargate (not Lambda)

- Background GPS sync job runs every 10 min (in-process scheduler — Lambda can't host this)
- Waze API chains (~20 calls per order) need a warm server to avoid cold-start latency
- Persistent PostgreSQL connection pool for PostGIS spatial queries

## Key Modules

| Module | Trigger | What It Does |
|--------|---------|--------------|
| GPS handler | `PATCH /couriers/{id}/gps` every 10 s | Writes courier lat/lng to DynamoDB `courier_states` |
| GPS sync scheduler | `setInterval` every 10 min | Batch-upserts `courier_locations` (PostGIS geometry) from DynamoDB to Supabase |
| ETA handler | `GET /deliveries/eta` | Calls Waze (restaurant → customer); returns minutes to Order Service at checkout |
| Assignment engine | SQS `order.preparing` event | PostGIS top-20 → Waze filter → writes `eligible_courier_ids` to DynamoDB |
| Poll handler | `GET /deliveries/available` every 30 s | Returns eligible orders for the polling courier |
| Accept handler | `POST /deliveries/{id}/accept` | Conditional DynamoDB write — first writer wins (409 on race) |
| Stage machine | `POST /stage`, `POST /confirm-delivery` | `PICKED_UP → DELIVERED`; publishes to SNS |
| SQS consumer | SQS Standard `delivery-events` queue | Routes `order.preparing` / `order.cancelled` events to assignment engine |

## Courier Assignment Flow

```
1. Courier App sends GPS every 10 s  →  DynamoDB courier_states
2. GPS sync (10 min)                 →  Supabase courier_locations (PostGIS)
3. order.preparing (SQS)             →  PostGIS: 20 nearest couriers to restaurant
4. Waze API per courier              →  filter by arrival time ≤ threshold
5. Write eligible_courier_ids        →  DynamoDB active_orders
6. Courier polls /available (30 s)   →  sees order in list
7. Courier taps "Take order"         →  conditional write; first wins
```

## Data Owned

| Store | Table | Role |
|-------|-------|------|
| DynamoDB | `courier_states` | Live GPS + availability status |
| DynamoDB | `active_orders` | `eligible_courier_ids`, `assignedCourierId` |
| DynamoDB | `order_events` | Stage transition audit log |
| Supabase | `courier_locations` | PostGIS geometry for nearest-courier queries |
| Supabase | `assignments` | Permanent courier-order assignment records |

## Integrations

- **Waze API** — ETA per courier and per order; fallback to distance-only if unavailable
- **SNS Standard** (`order-events`) — publishes delivery stage changes
- **SQS Standard** (`delivery-events`) — consumes `order.preparing` events from Order Service

## Key Alerts

| Metric | Threshold |
|--------|-----------|
| Assignment lag (`order.preparing` → accepted) | > 120 sec |
| Unaccepted orders | > 5 for > 5 min |
| Waze API error rate | > 10% / 5 min |
| GPS sync failure | Any failure |
