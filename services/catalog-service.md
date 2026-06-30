# Catalog Service

**Runtime:** AWS Lambda (Node.js)  
**Domain:** Catalog (customer-facing, read-only)

## Responsibility

Customer-facing menu browsing. Read-only queries against Supabase `menu_items`, `restaurants`, and `categories`. Does not write anything — all writes go through Menu Service.

## Key Operations

| Operation | Endpoint |
|-----------|----------|
| Browse restaurants | `GET /restaurants` |
| Search restaurants by name / cuisine | `GET /restaurants?search=&cuisine=` |
| Get restaurant menu | `GET /restaurants/{id}/menu` |
| Filter items by category / dietary label | `GET /menus/items?category=&label=` |
| Get single menu item | `GET /menus/items/{id}` |

## Data Access

| Store | Table | Access |
|-------|-------|--------|
| Supabase | `menu_items` | Read-only |
| Supabase | `restaurants` | Read-only |
| Supabase | `categories` | Read-only |

Catalog Service owns no tables. It is a pure read projection of data managed by Menu Service and User Service.

## Integrations

- **Order Service** — called synchronously to validate item availability during order creation
- **Supabase REST API** — auto-generated endpoints used for read queries

## Concurrency

| Baseline | Burst |
|----------|-------|
| 50 concurrent | 100 concurrent |

Stateless Lambda — scales automatically with customer browse traffic.
