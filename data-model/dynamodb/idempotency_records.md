# DynamoDB Schema: `idempotency_records`

## Model Description

The `idempotency_records` table stores idempotency state for important backend operations.

One item represents one logical operation attempt.

This table is used to prevent duplicate execution of the same operation.

The frontend does not create records in this table directly.

The frontend only generates an `Idempotency-Key` and sends it to the backend in the request header.

The backend creates and updates records in the `idempotency_records` table.

The same `Idempotency-Key` must be reused for retries of the same logical operation.

A new `Idempotency-Key` should be generated only for a new operation.

## Table

```text
idempotency_records
```

## Primary Key

```text
Partition Key: idempotencyRecordId
Sort Key: none
```

## Item Structure

```ts
type IdempotencyRecord = {
  idempotencyRecordId: string; // partition key

  requestHash: string;

  status: "processing" | "completed" | "failed";

  response?: object;

  createdAt: string;
  updatedAt: string;

  expiresAt: number; // Unix timestamp, DynamoDB TTL
};
```

## Example Item

```json
{
  "idempotencyRecordId": "create_order#USER#f41a1c2a-7a2b-4c0d-a8f6-8a1a5e901111#KEY#2fd36d89-5c4e-4f79-94d0-8ec924c3a2f5",

  "requestHash": "a8f6c2b9f4d3e1",

  "status": "completed",

  "response": {
    "orderId": "9c1db3f6-3992-4f27-8f45-0c65a2b703b1",
    "paymentUrl": "https://payment-provider.example/checkout/pay_123456789",
    "status": "waiting_payment"
  },

  "createdAt": "2026-06-27T10:00:00Z",
  "updatedAt": "2026-06-27T10:00:03Z",

  "expiresAt": 1782554400
}
```

## Indexes

No Global Secondary Indexes are required for MVP.

The table is accessed directly by `idempotencyRecordId`.

```text
GetItem / PutItem by idempotencyRecordId
```
