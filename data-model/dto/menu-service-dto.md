# MenuService External DTOs

This document describes DTOs used for communication between **MenuService** and external services/frontends.

MenuService stores menu items in MongoDB. Internally, MongoDB uses `_id`, but external services should use the public `id` field.

`price` is stored internally as MongoDB `Decimal128`, but in API requests and responses it is serialized as a string, for example `"12.99"`.

---

## Common Types

```ts
export type Currency = "ILS";

export type MenuItemId = string; // UUID
export type RestaurantId = string; // PostgreSQL restaurant UUID
```

---

## MenuItemDto

Used when MenuService returns menu item data to other services or frontends.

```ts
export type MenuItemDto = {
  id: MenuItemId;
  restaurantId: RestaurantId;

  name: string;
  description?: string;
  category?: string;

  price: string; // Decimal128 serialized as string, example: "12.99"
  currency: Currency;

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

Used by Restaurant App or Admin App to add a new menu item.

### Request

```ts
export type AddMenuItemRequestDto = {
  restaurantId: RestaurantId;

  name: string;
  description?: string;
  category?: string;

  price: string; // example: "12.99"
  currency: Currency;

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

## Get Available Menu Items by Restaurant ID

Used by Customer App to display available menu items for a selected restaurant.

### Request

```ts
export type GetMenuItemsByRestaurantRequestDto = {
  restaurantId: RestaurantId;
  available?: true;
};
```

### Response

```ts
export type GetMenuItemsByRestaurantResponseDto = {
  restaurantId: RestaurantId;
  items: MenuItemDto[];
};
```

---

## Get Menu Items by IDs

Used by Customer App when the user opens the cart.

MenuService returns only items that still exist and are available.

### Request

```ts
export type GetMenuItemsByIdsRequestDto = {
  menuItemIds: MenuItemId[];
};
```

### Response

```ts
export type GetMenuItemsByIdsResponseDto = {
  items: MenuItemDto[];
  missingItemIds: MenuItemId[];
  unavailableItemIds: MenuItemId[];
};
```

---

## Validate Menu Items for Order Creation

Used by OrderService before creating an order.

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
  restaurantId: RestaurantId;

  items: {
    menuItemId: MenuItemId;
    quantity: number;
    expectedPrice: string; // price that customer saw, example: "12.99"
    currency: Currency;
  }[];
};
```

### Validated Order Menu Item

```ts
export type ValidatedOrderMenuItemDto = {
  menuItemId: MenuItemId;
  restaurantId: RestaurantId;

  name: string;
  quantity: number;

  price: string;
  currency: Currency;

  subtotal: string;
};
```

### Response

```ts
export type ValidateMenuItemsForOrderResponseDto = {
  valid: boolean;

  restaurantId: RestaurantId;

  items: ValidatedOrderMenuItemDto[];

  subtotal: string;
  currency: Currency;

  errors: MenuValidationErrorDto[];
};
```

### Validation Error

```ts
export type MenuValidationErrorDto = {
  menuItemId?: MenuItemId;

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
