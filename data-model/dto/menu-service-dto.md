# MenuService External DTOs

This document describes DTOs used for communication between **MenuService** and external services/frontends.

MenuService stores menu items in MongoDB. Internally, MongoDB uses `_id`, but external services should use the public `id` field.

`price` is stored internally as MongoDB `Decimal128`, but in API requests and responses it is serialized as a string, for example `"12.99"`.

---

## REST API Summary

- `POST /api/menu-items` — Add menu item
- `PUT /api/menu-items/{id}` — Edit menu item
- `GET /api/restaurants/{restaurantId}/menu-items` — Get available menu items by restaurant
- `POST /api/menu-items/by-ids` — Get menu items by IDs
- `POST /api/menu-items/validate` — Validate menu items for order creation

---

## MenuItemDto

Used when MenuService returns menu item data to other services or frontends.

```ts
export type MenuItemDto = {
  id: string;
  restaurantId: string;

  name: string;
  description?: string;
  category?: string;

  price: string; // Decimal128 serialized as string, example: "12.99"
  currency: string;

  isAvailable: boolean;

  ingredients?: string[];
  allergens?: string[];

  labels?: {
    spicy?: boolean;
    vegetarian?: boolean;
    vegan?: boolean;
    kosher?: boolean;
    glutenFree?: boolean;
    lactoseFree?: boolean;
    halal?: boolean;
  };

  portion?: {
    weightGrams?: number;
    volumeMl?: number;
    pieces?: number;
    description?: string;
  };

  spicyLevel?: 0 | 1 | 2 | 3;

  nutrition?: {
    calories?: number;
    protein?: number;
    fat?: number;
    carbs?: number;
  };
};
```

---

## Add Menu Item

Used by the Restaurant App to add a new menu item for a restaurant.

- REST method: `POST`
- Endpoint: `/api/menu-items`
- Request body: `AddMenuItemRequestDto`
- Response body: `AddMenuItemResponseDto`
- HTTP statuses:
  - `201 Created` on success
  - `400 Bad Request` for validation errors
  - `403 Forbidden` when the restaurant is not allowed

### Request

```ts
export type AddMenuItemRequestDto = {
  restaurantId: string;

  name: string;
  description?: string;
  category?: string;

  price: string; // example: "12.99"
  currency: string;

  isAvailable?: boolean;

  ingredients?: string[];
  allergens?: string[];

  labels?: MenuItemDto["labels"];
  portion?: MenuItemDto["portion"];
  spicyLevel?: MenuItemDto["spicyLevel"];
  nutrition?: MenuItemDto["nutrition"];
};
```

### Response

```ts
export type AddMenuItemResponseDto = {
  item: MenuItemDto;
};
```

---

## Edit Menu Item

Used by the Restaurant App to update an existing menu item.

- REST method: `PUT`
- Endpoint: `/api/menu-items/{id}`
- Path parameter: `id` — menu item ID
- Request body: `EditMenuItemRequestDto`
- Response body: `EditMenuItemResponseDto`
- HTTP statuses:
  - `200 OK` on success
  - `400 Bad Request` for invalid input
  - `403 Forbidden` when the restaurant is not allowed
  - `404 Not Found` when the item does not exist

### Request

```ts
export type EditMenuItemRequestDto = {
  name?: string;
  description?: string;
  category?: string;

  price?: string; // example: "12.99"
  currency?: string;

  isAvailable?: boolean;

  ingredients?: string[];
  allergens?: string[];

  labels?: MenuItemDto["labels"];
  portion?: MenuItemDto["portion"];
  spicyLevel?: MenuItemDto["spicyLevel"];
  nutrition?: MenuItemDto["nutrition"];
};
```

### Response

```ts
export type EditMenuItemResponseDto = {
  item: MenuItemDto;
};
```

---

## Get Available Menu Items by Restaurant ID

Used by Customer App to display available menu items for a selected restaurant.

- REST method: `GET`
- Endpoint: `/api/restaurants/{restaurantId}/menu-items`
- Request: path parameter `restaurantId` and optional query parameter `available`
- Response body: `GetMenuItemsByRestaurantResponseDto`
- HTTP statuses:
  - `200 OK` on success
  - `400 Bad Request` for invalid query values
  - `404 Not Found` when the restaurant does not exist

### Request

- Path parameter: `restaurantId`
- Query parameter: `available?: true`

### Response

```ts
export type GetMenuItemsByRestaurantResponseDto = {
  restaurantId: string;
  items: MenuItemDto[];
};
```

---

## Get Menu Items by IDs

Used by Customer App when the user opens the cart.

MenuService returns only items that still exist and are available.

- REST method: `POST`
- Endpoint: `/api/menu-items/by-ids`
- Request body: `GetMenuItemsByIdsRequestDto`
- Response body: `GetMenuItemsByIdsResponseDto`
- HTTP statuses:
  - `200 OK` on success
  - `400 Bad Request` for invalid request payload

### Request

```ts
export type GetMenuItemsByIdsRequestDto = {
  menuItemIds: string[];
};
```

### Response

```ts
export type GetMenuItemsByIdsResponseDto = {
  items: MenuItemDto[];
  unavailableItemIds: string[]; // includes IDs for items that are missing or not available
};
```

---

## Validate Menu Items for Order Creation

Used by OrderService before creating an order.

- REST method: `POST`
- Endpoint: `/api/menu-items/validate`
- Request body: `ValidateMenuItemsForOrderRequestDto`
- Response body: `ValidateMenuItemsForOrderResponseDto`
- HTTP statuses:
  - `200 OK` on success
  - `400 Bad Request` for invalid request payload

MenuService checks that all selected menu items:

- exist;
- belong to the selected restaurant;
- are currently available;
- have the expected price;
- use the expected currency;
- have valid quantities.

### Request

```ts
export type ValidateMenuItemsForOrderRequestDto = {
  restaurantId: string;

  items: {
    menuItemId: string;
    quantity: number;
    expectedPrice: string; // price that customer saw, example: "12.99"
    currency: string;
  }[];
};
```

### Validated Order Menu Item

```ts
export type ValidatedOrderMenuItemDto = {
  menuItemId: string;
  restaurantId: string;

  name: string;
  quantity: number;

  price: string;
  currency: string;

  subtotal: string;
};
```

### Response

```ts
export type ValidateMenuItemsForOrderResponseDto = {
  valid: boolean;

  restaurantId: string;

  items: ValidatedOrderMenuItemDto[];

  subtotal: string;
  currency: string;

  errors: MenuValidationErrorDto[];
};
```

### Validation Error

```ts
export type MenuValidationErrorDto = {
  menuItemId?: string;

  code:
    | "ITEM_NOT_FOUND"
    | "ITEM_UNAVAILABLE"
    | "ITEM_RESTAURANT_MISMATCH"
    | "PRICE_CHANGED"
    | "INVALID_QUANTITY"
    | "CURRENCY_MISMATCH";

  message: string;

  currentPrice?: string;
  expectedPrice?: string;
};
```

---

## Important Notes

External DTOs do not expose MongoDB `_id`.

MongoDB `_id` is an internal storage detail. External services use the public `id` field as `menuItemId`.

Price is stored internally as `Decimal128`, but external APIs use string values to avoid floating-point precision issues.

Example:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "restaurantId": "64c1a2b3-d4e5-4678-9012-345600000000",
  "name": "Spicy Chicken Ramen",
  "price": "12.99",
  "currency": "ILS",
  "category": "Noodles",
  "isAvailable": true
}
```
