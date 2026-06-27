# DynamoDB Schema: `active_orders`

## Model Description

The `active_orders` table stores active operational order state.

One item represents one currently active order.

This table is used for fast reads and updates while the order is in progress.

Completed orders should be archived in PostgreSQL. DynamoDB keeps only live / temporary order state.

The table uses `orderId` as the partition key.

`customerId`, `restaurantId`, and `courierId` are UUID values from PostgreSQL. In DynamoDB they are stored as strings.

`items` contains an order-time snapshot of selected menu items. The snapshot is required because menu item data in MongoDB can change after the order is created.

## Table

```text
active_orders
```

## Primary Key

```text
Partition Key: orderId
Sort Key: none
```

## Item Structure

```ts
type ActiveOrder = {
  orderId: string; // UUID string, partition key

  customerId: string;   // UUID string, references PostgreSQL users.id
  restaurantId: string; // UUID string, references PostgreSQL restaurants.id
  courierId?: string;   // UUID string, references PostgreSQL users.id / courier profile

  status:
    | "waiting_payment"
    | "paid"
    | "waiting_restaurant_confirmation"
    | "restaurant_accepted"
    | "restaurant_rejected"
    | "searching_courier"
    | "courier_assigned"
    | "picked_up"
    | "delivered"
    | "cancelled"
    | "failed";

  items: {
    menuItemId: string; // UUID string, references PostgreSQL menu_items.id
    name: string;
    priceAtOrderTime: number;
    quantity: number;
    imageKey?: string;
  }[];

  deliveryAddress: {
    city: string;
    street: string;
    building: string;
    apartment?: string;
    lat: number;
    lng: number;
    comment?: string;
  };

  pricing: {
    itemsTotal: number;
    deliveryPrice: number;
    serviceFee: number;
    totalPrice: number;
  };

  paymentId?: string;
  stepFunctionExecutionArn: string;

  createdAt: string;
  updatedAt: string;

  expiresAt: number; // Unix timestamp, DynamoDB TTL
};
```

## Example Item

```json
{
  "orderId": "9c1db3f6-3992-4f27-8f45-0c65a2b703b1",

  "customerId": "f41a1c2a-7a2b-4c0d-a8f6-8a1a5e901111",
  "restaurantId": "64c1a2b3-0d4e-4f56-8901-234567890abc",
  "courierId": "8f8d5a34-2f02-4aa6-bb43-111122223333",

  "status": "courier_assigned",

  "items": [
    {
      "menuItemId": "665f9a8b-5e3f-4a1b-8c8d-9e0123456789",
      "name": "Spicy Chicken Ramen",
      "priceAtOrderTime": 42.5,
      "quantity": 2,
      "imageKey": "restaurants/64c1a2b3-0d4e-4f56-8901-234567890abc/menu/spicy-chicken-ramen.jpg"
    },
    {
      "menuItemId": "8f6d6c3a-1122-4f77-9f4b-123456789abc",
      "name": "Iced Tea",
      "priceAtOrderTime": 9.9,
      "quantity": 1,
      "imageKey": "restaurants/64c1a2b3-0d4e-4f56-8901-234567890abc/menu/iced-tea.jpg"
    }
  ],

  "deliveryAddress": {
    "city": "Tel Aviv",
    "street": "Dizengoff",
    "building": "100",
    "apartment": "12",
    "lat": 32.08088,
    "lng": 34.78057,
    "comment": "Call on arrival"
  },

  "pricing": {
    "itemsTotal": 94.9,
    "deliveryPrice": 12.0,
    "serviceFee": 4.5,
    "totalPrice": 111.4
  },

  "paymentId": "pay_123456789",
  "stepFunctionExecutionArn": "arn:aws:states:eu-central-1:123456789012:execution:order-state-machine:9c1db3f6-3992-4f27-8f45-0c65a2b703b1",

  "createdAt": "2026-06-27T10:00:00Z",
  "updatedAt": "2026-06-27T10:12:00Z",

  "expiresAt": 1782554400
}
```

## Global Secondary Indexes

### Customer orders

```text
GSI name: GSI_customer_orders

Partition Key: customerId
Sort Key: createdAt
```

Used to load active orders for one customer.

Projected attributes:

```text
orderId
customerId
restaurantId
courierId
status
items
pricing
createdAt
updatedAt
expiresAt
```

Example access pattern:

```text
GET /customers/:customerId/active-orders
```

---

### Restaurant orders

```text
GSI name: GSI_restaurant_orders

Partition Key: restaurantId
Sort Key: status
```

Used to load active orders for one restaurant grouped or filtered by status.

Projected attributes:

```text
orderId
customerId
restaurantId
courierId
status
items
pricing
createdAt
updatedAt
expiresAt
```

Example access pattern:

```text
GET /restaurants/:restaurantId/active-orders
GET /restaurants/:restaurantId/active-orders?status=waiting_restaurant_confirmation
```

---

### Courier orders

```text
GSI name: GSI_courier_orders

Partition Key: courierId
Sort Key: updatedAt
```

Used to load active orders assigned to one courier.

This index is optional if the courier app gets the current order id from the `courier_states` table.

Projected attributes:

```text
orderId
customerId
restaurantId
courierId
status
items
deliveryAddress
pricing
createdAt
updatedAt
expiresAt
```

Example access pattern:

```text
GET /couriers/:courierId/active-orders
```
