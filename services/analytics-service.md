# Analytics Service

**Runtime:** AWS Lambda (Python)  
**Domain:** Business Intelligence

## Responsibility

Generates daily and weekly aggregations for restaurant performance reports, order trends, and platform-level statistics. Read-only — does not write to operational tables.

## Key Operations

| Operation | Endpoint |
|-----------|----------|
| Daily order report per restaurant | `GET /analytics/restaurants/{id}/daily` |
| Weekly sales summary | `GET /analytics/restaurants/{id}/weekly` |
| Platform-wide order trends | `GET /analytics/orders/trends` |
| Courier performance stats | `GET /analytics/couriers/{id}/stats` |

## Data Access

| Store | Table | Access |
|-------|-------|--------|
| Supabase | `orders`, `order_items` | Read-only (archived terminal orders) |
| Supabase | `restaurants`, `couriers` | Read-only |
| MongoDB Atlas | — | Session data reads if needed |

Analytics Service owns no tables. All data is read from Supabase PostgreSQL after orders reach terminal state.

## Integrations

- **Restaurant Dashboard** — fetches daily/weekly reports via REST
- **Ops Dashboard** — fetches platform-level trend data

## Concurrency

| Baseline | Burst |
|----------|-------|
| 10 concurrent | 50 concurrent |

Aggregation queries run against the archived `orders` table (Supabase) — never against `active_orders` (DynamoDB) — so they do not impact the operational read path.
