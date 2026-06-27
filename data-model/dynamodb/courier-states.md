# DynamoDB Schema: `courier_states`

## Model Description

The `courier_states` table stores the current live state of couriers.

One item represents one courier and their current operational state.

This table is used by the Delivery Service to quickly check:

- whether a courier is offline, available, or busy
- which order the courier is currently assigned to
- the courier's latest known location

PostgreSQL stores the permanent courier profile data.

DynamoDB stores only live operational courier state.

`courierId` is a UUID value from PostgreSQL. In DynamoDB it is stored as a string.

## Table

```text
courier_states
```

## Primary Key

```text
Partition Key: courierId
Sort Key: none
```

## Item Structure

```ts
type CourierState = {
  courierId: string; // UUID string, partition key

  status: "offline" | "available" | "busy";

  currentOrderId?: string; // UUID string, references active_orders.orderId

  vehicleType?: "bike" | "car" | "walk";

  lastLocation?: {
    lat: number;
    lng: number;
    updatedAt: string;
  };

  updatedAt: string;
};
```

## Example Item

```json
{
  "courierId": "8f8d5a34-2f02-4aa6-bb43-111122223333",

  "status": "busy",
  "currentOrderId": "9c1db3f6-3992-4f27-8f45-0c65a2b703b1",
  "vehicleType": "bike",

  "lastLocation": {
    "lat": 32.08088,
    "lng": 34.78057,
    "updatedAt": "2026-06-27T10:12:00Z"
  },

  "updatedAt": "2026-06-27T10:12:00Z"
}
```

## Indexes

### Courier states by status

```text
GSI name: GSI_couriers_by_status

Partition Key: status
Sort Key: updatedAt
```

Used to find couriers by their current status.

Example access patterns:

```text
Find all available couriers
Find all busy couriers
Find all offline couriers
```

Projected attributes:

```text
courierId
status
currentOrderId
vehicleType
lastLocation
updatedAt
```

---

### Courier state by current order

```text
GSI name: GSI_courier_by_current_order

Partition Key: currentOrderId
Sort Key: none
```

Used to quickly find which courier is currently assigned to a specific active order.

Example access pattern:

```text
Find courier assigned to orderId
```

Projected attributes:

```text
courierId
status
currentOrderId
vehicleType
lastLocation
updatedAt
```
