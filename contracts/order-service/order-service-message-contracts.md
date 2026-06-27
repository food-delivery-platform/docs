# Order Service — Inter-Service Message Contracts

> **Service:** `order-service`
> **Runtime:** Node.js / AWS Lambda + AWS Step Functions
> **Role:** Source of truth for the order lifecycle. Owns the canonical `orders` table in Supabase/Postgres and the `order_status` enum. Acts as the orchestrator ("brain") of every order.

This document is the Markdown view of `order-service-message-contracts.yaml`. The YAML remains the machine-readable source; this `.md` is for easier human review.

> **Note on examples:** all examples deliberately reuse the **same ids and addresses** as the delivery-service contract (`order-xyz`, `rest-456`, `user-123`, `courier-001`) so both documents describe **one coherent end-to-end order**.

---

## Channels overview

| # | Channel | Direction | Transport | Counterparties |
|---|---------|-----------|-----------|----------------|
| 1 | `rest_api_inbound` | inbound | REST (API Gateway) | Customer App, Restaurant App, internal services |
| 2 | `sns_fifo_outbound` | outbound | SNS FIFO | Delivery, Payment, Notification, Monitoring |
| 3 | `sqs_fifo_inbound` | inbound | SQS FIFO | results from Payment & Delivery |
| 4 | `rest_api_outbound` | outbound | REST | Catalog, Payment |
| 5 | `step_functions` | internal | Step Functions | order orchestration |
| 6 | `supabase_writes` | outbound | Postgres write | canonical `orders` / `order_items` / `order_status_history` |
| 7 | `supabase_realtime_outbound` | outbound | Supabase Realtime | Customer App (live tracking) |
| 8 | `cloudwatch_metrics` | outbound | CloudWatch | custom metrics + alarms |

---

## Enums

**`order_status`** (owned by Order Service — every other service maps its local stage onto this list):

```
CREATED | PENDING_PAYMENT | PAID | CONFIRMED | PREPARING | READY |
COURIER_ASSIGNED | PICKED_UP | ON_THE_WAY | DELIVERED |
CANCELLED | REFUNDED | FAILED
```

**`cancel_reason`**:

```
PAYMENT_FAILED | RESTAURANT_REJECTED | RESTAURANT_TIMEOUT |
INVALID_ORDER | CUSTOMER_CANCELLED
```

**Shared anchors**

- SNS FIFO topic ARN: `arn:aws:sns:eu-west-1:123456789012:order-events.fifo`
- Restaurant address: `{ lat: 32.0853, lng: 34.7818, street: "Rothschild Blvd 1", city: "Tel Aviv" }`
- Customer address: `{ addressId: "addr-789", lat: 32.0742, lng: 34.7922, street: "Ben Yehuda St 44", city: "Tel Aviv" }`

---

## 1. REST API inbound

Endpoints exposed by Order Service via API Gateway.
Callers: Customer App (Next.js), Restaurant App (React), and internal services (Delivery, Payment) with a service JWT.

| Endpoint | Caller | Purpose |
|----------|--------|---------|
| `POST /api/v1/orders` | customer-app | Create order; starts pre-payment pipeline |
| `GET /api/v1/orders/{orderId}` | customer-app / admin | Fetch single order (tracking screen) |
| `GET /api/v1/orders?customerId=…` | customer-app | Paginated order history |
| `POST /api/v1/orders/{orderId}/cancel` | customer-app | Cancel before restaurant confirms |
| `POST /api/v1/orders/{orderId}/confirm` | restaurant-app | Restaurant accepts order |
| `POST /api/v1/orders/{orderId}/reject` | restaurant-app | Restaurant rejects → refund |
| `PATCH /api/v1/orders/{orderId}/status` | restaurant-app / delivery-service | Status updates |
| `GET /health` | ALB / API Gateway | Health check |

### `POST /api/v1/orders`

Create a new order. Starts the Step Functions pre-payment pipeline. **Caller:** customer-app.

**Request**

```json
{
  "headers": { "Authorization": "Bearer <customer_jwt>" },
  "body": {
    "customerId": "user-123",
    "restaurantId": "rest-456",
    "items": [
      { "menuItemId": "item-1", "quantity": 2, "specialInstructions": "No onions" },
      { "menuItemId": "item-2", "quantity": 1, "specialInstructions": null }
    ],
    "deliveryAddress": { "addressId": "addr-789", "lat": 32.0742, "lng": 34.7922, "street": "Ben Yehuda St 44", "city": "Tel Aviv" },
    "paymentMethod": "CARD"
  }
}
```

`paymentMethod` enum: `CARD | PAYPAL`.

**Response `202 Accepted`**

```json
{
  "orderId": "order-xyz",
  "status": "PENDING_PAYMENT",
  "totalAmount": 78.50,
  "currency": "ILS",
  "createdAt": "2026-06-23T18:45:00.000Z",
  "paymentUrl": "https://checkout.stripe.com/c/pay/cs_test_xyz",
  "restaurantConfirmationDeadline": "2026-06-23T18:50:00.000Z"
}
```

**Response `400 Bad Request`**

```json
{
  "error": "UNAVAILABLE_ITEMS",
  "message": "Some requested menu items are out of stock or unavailable.",
  "details": { "unavailableItems": ["item-2"] }
}
```

### `GET /api/v1/orders/{orderId}`

Fetch a single order (tracking screen). **Caller:** customer-app / admin-dashboard.

**Request:** header `Authorization: Bearer <jwt_access_token>`, path param `orderId: order-xyz`.

**Response `200 OK`**

```json
{
  "orderId": "order-xyz",
  "customerId": "user-123",
  "restaurantId": "rest-456",
  "restaurantName": "HaBurger",
  "status": "ON_THE_WAY",
  "items": [
    { "menuItemId": "item-1", "name": "Shakshuka", "quantity": 2, "unitPrice": 32.00 },
    { "menuItemId": "item-2", "name": "Lemonade", "quantity": 1, "unitPrice": 14.50 }
  ],
  "subtotal": 78.50,
  "deliveryFee": 12.00,
  "totalAmount": 90.50,
  "currency": "ILS",
  "deliveryAddress": { "addressId": "addr-789", "lat": 32.0742, "lng": 34.7922, "street": "Ben Yehuda St 44", "city": "Tel Aviv" },
  "courierId": "courier-001",
  "estimatedDeliveryTime": "2026-06-23T19:30:00.000Z",
  "createdAt": "2026-06-23T18:45:00.000Z",
  "updatedAt": "2026-06-23T19:08:30.000Z"
}
```

`courierId` is `null` until `COURIER_ASSIGNED`.

**Response `404 Not Found`**

```json
{ "error": "ORDER_NOT_FOUND", "message": "No order found with id order-xyz" }
```

### `GET /api/v1/orders?customerId={customerId}&limit={limit}&cursor={cursor}`

Customer's own order history (paginated). **Caller:** customer-app.

**Request:** header `Authorization: Bearer <customer_jwt>`; query params `customerId: user-123`, `limit: 20`, `cursor: null`.

**Response `200 OK`**

```json
{
  "orders": [
    {
      "orderId": "order-xyz",
      "restaurantName": "HaBurger",
      "status": "ON_THE_WAY",
      "totalAmount": 90.50,
      "currency": "ILS",
      "createdAt": "2026-06-23T18:45:00.000Z"
    }
  ],
  "nextCursor": "eyJvZmZzZXQiOjIwfQ=="
}
```

### `POST /api/v1/orders/{orderId}/cancel`

Customer cancels **before** the restaurant confirms (allowed states: `PENDING_PAYMENT`, `PAID`, `CONFIRMED`; rejected once `PREPARING`). **Caller:** customer-app.

**Request body**

```json
{ "reason": "CUSTOMER_CANCELLED" }
```

**Response `200 OK`**

```json
{ "orderId": "order-xyz", "status": "CANCELLED", "refundStatus": "PENDING" }
```

`refundStatus` enum: `NONE | PENDING | COMPLETED`.

**Response `409 Conflict`**

```json
{ "error": "CANCELLATION_NOT_ALLOWED", "message": "Order is already PREPARING and can no longer be cancelled." }
```

### `POST /api/v1/orders/{orderId}/confirm`

Restaurant accepts the order; resumes the post-payment state machine. **Caller:** restaurant-app.

**Request body**

```json
{ "restaurantId": "rest-456", "preparationTimeMinutes": 20 }
```

**Response `200 OK`**

```json
{ "orderId": "order-xyz", "status": "CONFIRMED", "estimatedPickupTime": "2026-06-23T19:05:00.000Z" }
```

### `POST /api/v1/orders/{orderId}/reject`

Restaurant rejects the order → triggers refund + `order.cancelled`. **Caller:** restaurant-app.

**Request body**

```json
{ "restaurantId": "rest-456", "reason": "TOO_BUSY" }
```

`reason` enum: `OUT_OF_STOCK | TOO_BUSY | CLOSED | OTHER`.

**Response `200 OK`**

```json
{ "orderId": "order-xyz", "status": "CANCELLED", "refundStatus": "PENDING" }
```

### `PATCH /api/v1/orders/{orderId}/status`

Status update endpoint. **Two distinct callers:**
- (a) Restaurant App → `PREPARING`, `READY`
- (b) Delivery Service → `PICKED_UP`, `ON_THE_WAY`, `DELIVERED`, `FAILED`

> This is the **same endpoint** delivery-service calls in its `rest_api_outbound`.

**Request**

```json
{
  "headers": { "Authorization": "Bearer <restaurant_jwt | internal_service_jwt>" },
  "pathParams": { "orderId": "order-xyz" },
  "body": { "status": "PICKED_UP", "actorId": "courier-001", "timestamp": "2026-06-23T19:07:00.000Z" }
}
```

**Response `200 OK`**

```json
{ "orderId": "order-xyz", "status": "PICKED_UP", "updatedAt": "2026-06-23T19:07:00.000Z" }
```

**Response `409 Conflict`**

```json
{ "error": "INVALID_STATUS_TRANSITION", "message": "Cannot move from CONFIRMED to DELIVERED" }
```

### `GET /health`

ALB / API Gateway health check.

**Response `200 OK`**

```json
{
  "status": "ok",
  "checks": { "supabase": "ready", "snsPublisher": "ready", "stepFunctions": "ready" },
  "timestamp": "2026-06-23T19:00:00.000Z"
}
```

**Response `503 Service Unavailable`**

```json
{
  "status": "degraded",
  "checks": { "supabase": "error", "snsPublisher": "ready", "stepFunctions": "ready" },
  "timestamp": "2026-06-23T19:00:00.000Z"
}
```

---

## 2. SNS FIFO outbound

Published to `order-events.fifo`. `MessageGroupId = order_id` guarantees strict per-order ordering across all consumers.
Consumers: Delivery Service, Payment Service, Notification Service, Monitoring/SLA Service.

| Event | Emitted when | Key consumers |
|-------|--------------|---------------|
| `order.created` | after validation + price calc, before payment | Payment, Monitoring |
| `order.confirmed` | payment succeeded AND restaurant accepted | **Delivery**, Notification |
| `order.cancelled` | any cancellation path | **Delivery**, Notification |
| `order.status_changed` | intermediate transitions (e.g. CONFIRMED→PREPARING) | Notification, Analytics |
| `order.status.ready` | kitchen marks READY | **Delivery** |
| `order.completed` | delivery confirmed (terminal success) | Notification, Monitoring |
| `order.refunded` | refund processed | Notification, Finance |

All events share the SNS envelope: `SNSTopicArn` = topic ARN above, `MessageGroupId: order_id`, plus a `MessageDeduplicationId` (shown per event), `Subject`, and a `Message` payload.

### `order.created`

`MessageDeduplicationId: "order_id + ':created'"`

```json
{
  "eventType": "order.created",
  "eventId": "evt_01J0OS0001ABCD",
  "timestamp": "2026-06-23T18:45:02.000Z",
  "orderId": "order-xyz",
  "customerId": "user-123",
  "restaurantId": "rest-456",
  "totalAmount": 78.50,
  "currency": "ILS",
  "paymentMethod": "CARD",
  "status": "PENDING_PAYMENT",
  "actorId": "user-123"
}
```

### `order.confirmed`

`MessageDeduplicationId: "order_id + ':confirmed'"`

> **Consumed by delivery-service** (`sqs_fifo_inbound.order.confirmed`). Field shape MUST match what Delivery Service expects.

```json
{
  "eventType": "order.confirmed",
  "eventId": "evt_01J0OS0002ABCD",
  "timestamp": "2026-06-23T18:50:00.000Z",
  "orderId": "order-xyz",
  "customerId": "user-123",
  "restaurantId": "rest-456",
  "restaurantAddress": { "lat": 32.0853, "lng": 34.7818, "street": "Rothschild Blvd 1", "city": "Tel Aviv" },
  "deliveryAddress": { "addressId": "addr-789", "lat": 32.0742, "lng": 34.7922, "street": "Ben Yehuda St 44", "city": "Tel Aviv" },
  "estimatedPickupTime": "2026-06-23T19:05:00.000Z",
  "estimatedDeliveryTime": "2026-06-23T19:30:00.000Z",
  "items": [ { "menuItemId": "item-1", "name": "Shakshuka", "quantity": 2 } ],
  "totalAmount": 78.50,
  "currency": "ILS",
  "actorId": "rest-456"
}
```

### `order.cancelled`

`MessageDeduplicationId: "order_id + ':cancelled'"`

> **Consumed by delivery-service** (`sqs_fifo_inbound.order.cancelled`). Emitted on any cancellation path (payment failed, restaurant rejected, confirmation timeout, customer cancelled).

```json
{
  "eventType": "order.cancelled",
  "eventId": "evt_01J0OS0003ABCD",
  "timestamp": "2026-06-23T18:55:00.000Z",
  "orderId": "order-xyz",
  "reason": "RESTAURANT_REJECTED",
  "actorId": "rest-456"
}
```

### `order.status_changed`

`MessageDeduplicationId: "order_id + ':status_changed:' + newStatus"`

```json
{
  "eventType": "order.status_changed",
  "eventId": "evt_01J0OS0004ABCD",
  "timestamp": "2026-06-23T18:52:00.000Z",
  "orderId": "order-xyz",
  "previousStatus": "CONFIRMED",
  "newStatus": "PREPARING",
  "actorId": "rest-456"
}
```

### `order.status.ready`

`MessageDeduplicationId: "order_id + ':ready'"`

> **Consumed by delivery-service** (`sqs_fifo_inbound.order.status.ready`). `courierId` is included because Order Service already stored it from the `delivery.courier_assigned` event it consumed earlier.

```json
{
  "eventType": "order.status.ready",
  "eventId": "evt_01J0OS0005ABCD",
  "timestamp": "2026-06-23T19:05:00.000Z",
  "orderId": "order-xyz",
  "restaurantId": "rest-456",
  "courierId": "courier-001",
  "actorId": "rest-456"
}
```

### `order.completed`

`MessageDeduplicationId: "order_id + ':completed'"`

```json
{
  "eventType": "order.completed",
  "eventId": "evt_01J0OS0006ABCD",
  "timestamp": "2026-06-23T19:28:00.000Z",
  "orderId": "order-xyz",
  "customerId": "user-123",
  "restaurantId": "rest-456",
  "actualDeliveryTime": "2026-06-23T19:28:00.000Z",
  "totalAmount": 78.50,
  "currency": "ILS",
  "actorId": "SYSTEM"
}
```

### `order.refunded`

`MessageDeduplicationId: "order_id + ':refunded'"`

```json
{
  "eventType": "order.refunded",
  "eventId": "evt_01J0OS0007ABCD",
  "timestamp": "2026-06-23T18:56:00.000Z",
  "orderId": "order-xyz",
  "refundId": "re_01J0PAY9999",
  "amount": 78.50,
  "currency": "ILS",
  "reason": "RESTAURANT_REJECTED",
  "actorId": "SYSTEM"
}
```

---

## 3. SQS FIFO inbound

Order Service subscribes `order-service-inbound.fifo` to the shared SNS FIFO topic. `MessageGroupId = order_id`. DLQ after 3 failures. These are the events Order Service **reacts to** in order to advance the state machine and update the canonical `orders.status`.

| Event | Published by | Order Service reaction |
|-------|--------------|------------------------|
| `payment.succeeded` | Payment | `PENDING_PAYMENT → PAID`, route to restaurant |
| `payment.failed` | Payment | `PENDING_PAYMENT → CANCELLED` (PAYMENT_FAILED) |
| `delivery.courier_assigned` | Delivery | store `courierId`, set `COURIER_ASSIGNED` |
| `delivery.status.changed` | Delivery | sync canonical status; on DELIVERED emit `order.completed` |

### `payment.succeeded`

`MessageDeduplicationId: "order_id + ':payment_succeeded'"`

```json
{
  "eventType": "payment.succeeded",
  "eventId": "evt_01J0PAY0001ABCD",
  "timestamp": "2026-06-23T18:47:00.000Z",
  "orderId": "order-xyz",
  "paymentId": "pi_01J0PAY1234",
  "amount": 78.50,
  "currency": "ILS",
  "actorId": "user-123"
}
```

### `payment.failed`

Decline / timeout (10-min window). `MessageDeduplicationId: "order_id + ':payment_failed'"`

```json
{
  "eventType": "payment.failed",
  "eventId": "evt_01J0PAY0002ABCD",
  "timestamp": "2026-06-23T18:55:30.000Z",
  "orderId": "order-xyz",
  "reason": "EXPIRED_SESSION",
  "actorId": "user-123"
}
```

`reason` enum: `CARD_DECLINED | EXPIRED_SESSION | INSUFFICIENT_FUNDS | OTHER`.

### `delivery.courier_assigned`

Published by delivery-service (`sns_fifo_outbound.delivery.courier_assigned`). `MessageDeduplicationId: "order_id + ':courier_assigned'"`

```json
{
  "eventType": "delivery.courier_assigned",
  "eventId": "evt_01J0DS1234ABCD",
  "timestamp": "2026-06-23T18:51:00.000Z",
  "orderId": "order-xyz",
  "courierId": "courier-001",
  "courierName": "Avi Cohen",
  "courierPhone": "+972501234567",
  "vehicleType": "BICYCLE",
  "estimatedPickupTime": "2026-06-23T19:05:00.000Z",
  "estimatedDeliveryTime": "2026-06-23T19:30:00.000Z",
  "actorId": "SYSTEM"
}
```

### `delivery.status.changed`

Delivery stage events consumed to keep canonical `orders.status` in sync. Maps: `picked_up → PICKED_UP`, `on_the_way → ON_THE_WAY`, `delivered → DELIVERED` (then emits `order.completed`), `failed → FAILED` (then triggers refund flow). `MessageDeduplicationId: "order_id + ':delivery_status:' + stage"`

```json
{
  "eventType": "delivery.status.delivered",
  "eventId": "evt_01J0DS4444ABCD",
  "timestamp": "2026-06-23T19:28:00.000Z",
  "orderId": "order-xyz",
  "courierId": "courier-001",
  "stage": "DELIVERED",
  "location": { "lat": 32.0742, "lng": 34.7922 },
  "actorId": "courier-001"
}
```

`stage` enum: `PICKED_UP | ON_THE_WAY | DELIVERED | FAILED`.

---

## 4. REST API outbound

Synchronous calls made **by** Order Service **to** other services. All use an internal service JWT.

| Endpoint | Target service | Purpose |
|----------|----------------|---------|
| `POST /api/v1/catalog/validate` | catalog-service | Validate items + authoritative prices/availability |
| `POST /api/v1/payments/sessions` | payment-service | Open a payment session |

### `POST /api/v1/catalog/validate`

Validate items + fetch authoritative prices and availability **before** creating the order.

**Request body**

```json
{
  "restaurantId": "rest-456",
  "items": [
    { "menuItemId": "item-1", "quantity": 2 },
    { "menuItemId": "item-2", "quantity": 1 }
  ]
}
```

**Response `200 OK`**

```json
{
  "valid": true,
  "restaurantOpen": true,
  "items": [
    { "menuItemId": "item-1", "name": "Shakshuka", "unitPrice": 32.00, "available": true },
    { "menuItemId": "item-2", "name": "Lemonade", "unitPrice": 14.50, "available": true }
  ],
  "subtotal": 78.50,
  "currency": "ILS"
}
```

**Response `409 Conflict`**

```json
{ "valid": false, "unavailableItems": ["item-2"] }
```

### `POST /api/v1/payments/sessions`

Open a payment session. (May also be invoked as a Step Functions task instead of direct REST.)

**Request body**

```json
{
  "orderId": "order-xyz",
  "customerId": "user-123",
  "amount": 78.50,
  "currency": "ILS",
  "method": "CARD",
  "successUrl": "https://app.fooddelivery.co/orders/order-xyz/success",
  "cancelUrl": "https://app.fooddelivery.co/orders/order-xyz/cancel"
}
```

**Response `201 Created`**

```json
{
  "paymentId": "pi_01J0PAY1234",
  "paymentUrl": "https://checkout.stripe.com/c/pay/cs_test_xyz",
  "expiresAt": "2026-06-23T18:55:00.000Z"
}
```

`expiresAt` reflects the 10-minute payment window.

---

## 5. Step Functions (orchestration)

Per Yuri's guidance the lifecycle is **split into two state machines** rather than one giant long-running execution:

- **(A) pre-payment SM** — short, synchronous: validate → price → create payment session → wait for payment callback.
- **(B) post-payment SM** — wait for restaurant confirm → READY → wait for delivery terminal event → complete.

### (A) pre-payment state machine

`stateMachineArn: arn:aws:states:eu-west-1:123456789012:stateMachine:order-pre-payment`

States: `ValidateItems → CalculateTotals → PersistOrder → CreatePaymentSession → WaitForPayment (task token) → (PaymentSucceeded | PaymentFailed)`

**Execution input**

```json
{
  "orderId": "order-xyz",
  "customerId": "user-123",
  "restaurantId": "rest-456",
  "items": [
    { "menuItemId": "item-1", "quantity": 2, "specialInstructions": "No onions" },
    { "menuItemId": "item-2", "quantity": 1, "specialInstructions": null }
  ],
  "deliveryAddress": { "addressId": "addr-789", "lat": 32.0742, "lng": 34.7922, "street": "Ben Yehuda St 44", "city": "Tel Aviv" },
  "paymentMethod": "CARD"
}
```

**`WaitForPayment` task-token payload** (resumes via `SendTaskSuccess` when `payment.succeeded` arrives)

```json
{ "taskToken": "AAAAKgAAAAIAAAAA...exampleTaskToken", "orderId": "order-xyz", "paymentId": "pi_01J0PAY1234" }
```

**Output**

```json
{ "orderId": "order-xyz", "status": "PAID" }
```

### (B) post-payment state machine

`stateMachineArn: arn:aws:states:eu-west-1:123456789012:stateMachine:order-post-payment`

States: `NotifyRestaurant → WaitForConfirmation (timeout 5 min) → (Confirmed → WaitForReady → EmitReady → WaitForDelivery → Completed) | (Timeout/Reject → Cancel + Refund)`

**Execution input**

```json
{
  "orderId": "order-xyz",
  "customerId": "user-123",
  "restaurantId": "rest-456",
  "restaurantAddress": { "lat": 32.0853, "lng": 34.7818, "street": "Rothschild Blvd 1", "city": "Tel Aviv" },
  "deliveryAddress": { "addressId": "addr-789", "lat": 32.0742, "lng": 34.7922, "street": "Ben Yehuda St 44", "city": "Tel Aviv" },
  "items": [ { "menuItemId": "item-1", "name": "Shakshuka", "quantity": 2 } ],
  "totalAmount": 78.50,
  "currency": "ILS"
}
```

**`WaitForConfirmation`**

```json
{ "timeoutSeconds": 300, "onTimeout": "cancel_with_reason_RESTAURANT_TIMEOUT" }
```

**Output**

```json
{ "orderId": "order-xyz", "status": "DELIVERED", "terminal": true }
```

---

## 6. Supabase / Postgres write shapes

Order Service is the **only writer** of the canonical `orders`, `order_items`, and `order_status_history` tables. Other services read via REST or react to SNS events.

### Table: `orders` (PK: `id`)

Updated on every lifecycle transition; `orders.status` drives Realtime.

```json
{
  "id": "order-xyz",
  "customer_id": "user-123",
  "restaurant_id": "rest-456",
  "status": "ON_THE_WAY",
  "subtotal": 78.50,
  "delivery_fee": 12.00,
  "total_amount": 90.50,
  "currency": "ILS",
  "payment_id": "pi_01J0PAY1234",
  "payment_method": "CARD",
  "courier_id": "courier-001",
  "delivery_address": { "lat": 32.0742, "lng": 34.7922, "street": "Ben Yehuda St 44", "city": "Tel Aviv" },
  "estimated_delivery_time": "2026-06-23T19:30:00.000Z",
  "created_at": "2026-06-23T18:45:00.000Z",
  "updated_at": "2026-06-23T19:08:30.000Z"
}
```

`courier_id` is `null` until `COURIER_ASSIGNED`.

### Table: `order_items` (PK: `id`, FK: `order_id`)

One row per line item.

```json
{
  "id": "oitem-1",
  "order_id": "order-xyz",
  "menu_item_id": "item-1",
  "name": "Shakshuka",
  "quantity": 2,
  "unit_price": 32.00,
  "special_instructions": "No onions"
}
```

### Table: `order_status_history` (PK: `id`, FK: `order_id`)

Append-only audit log of every status transition (required by the spec — "хранит историю смены статусов"). One row written on each transition.

```json
{
  "id": "hist-1",
  "order_id": "order-xyz",
  "previous_status": "PICKED_UP",
  "new_status": "ON_THE_WAY",
  "actor_type": "COURIER",
  "actor_id": "courier-001",
  "changed_at": "2026-06-23T19:08:30.000Z"
}
```

`previous_status` is `null` for the very first row (CREATED). `actor_type` enum: `CUSTOMER | RESTAURANT | COURIER | SYSTEM`.

---

## 7. Supabase Realtime outbound → Customer App

A write to `orders.status` replicates to Supabase Realtime; the Customer App subscribes to channel `orders:order_id` for live tracking. (Delivery Service also writes courier GPS on a separate channel.)

**`realtime.order_status_changed`**

```json
{
  "schema": "public",
  "table": "orders",
  "eventType": "UPDATE",
  "new": { "id": "order-xyz", "status": "ON_THE_WAY", "updated_at": "2026-06-23T19:08:30.000Z" },
  "old": { "id": "order-xyz", "status": "PICKED_UP" }
}
```

---

## 8. CloudWatch custom metrics

Published via `PutMetricData`. Namespace: `FoodDelivery/OrderService`. Alarms fan out to the ops SNS alert topic.

| Metric | Unit | Example value | Alarm threshold |
|--------|------|---------------|-----------------|
| `OrdersCreatedCount` | Count | 1 | — (dashboard only) |
| `PaymentFailureRate` | Percent | 1.8 | > 5% over 15 min |
| `RestaurantRejectionRate` | Percent | 4.2 | > 15% over 15 min |
| `OrderConfirmationLatencySeconds` | Seconds | 95 | p95 > 360 s |
| `StepFunctionsExecutionErrors` | Count | 0 | > 3 errors / 5 min |

`OrderConfirmationLatencySeconds` measures time from `order.created` to `order.confirmed`. Dimensions example: `[{ "Name": "Environment", "Value": "production" }]`.
