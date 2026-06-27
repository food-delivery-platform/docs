# MongoDB Schema: `menu_items`

## Model Description

The `menu_items` collection stores the current menu catalog for restaurants.

One document represents one dish or product from a restaurant menu.

`restaurantId` links the MongoDB document to a restaurant stored in PostgreSQL.

`menuItemId` links the MongoDB document to a SQL `menu_items` record in PostgreSQL.

Both `restaurantId` and `menuItemId` are UUID values from PostgreSQL. In MongoDB they are stored as strings.

Images are stored in S3. MongoDB stores only `imageKey`, not a public image URL.

## Document Structure

```ts
type MenuItem = {
  id: string;

  restaurantId: string; // UUID string, references PostgreSQL restaurants.id
  menuItemId: string;   // UUID string, references PostgreSQL menu_items.id

  name: string;
  description?: string;
  price: number;
  imageKey?: string;
  category?: string;
  isAvailable: boolean;

  ingredients?: string[];

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

  createdAt: string;
  updatedAt: string;
};
```

## Example Document

```json
{
  "id": "mongo_menu_item_123",

  "restaurantId": "64c1a2b3-0d4e-4f56-8901-234567890abc",
  "menuItemId": "665f9a8b-5e3f-4a1b-8c8d-9e0123456789",

  "name": "Spicy Chicken Ramen",
  "description": "Ramen with chicken, egg, scallions and spicy broth",
  "price": 42.5,
  "imageKey": "restaurants/64c1a2b3-0d4e-4f56-8901-234567890abc/menu/spicy-chicken-ramen.jpg",
  "category": "Noodles",
  "isAvailable": true,

  "ingredients": [
    "ramen noodles",
    "chicken",
    "spicy broth",
    "scallions",
    "egg"
  ],

  "labels": {
    "spicy": true,
    "vegetarian": false,
    "vegan": false,
    "kosher": false,
    "glutenFree": false,
    "lactoseFree": true,
    "halal": false
  },

  "portion": {
    "weightGrams": 420,
    "description": "1 bowl"
  },

  "spicyLevel": 2,

  "nutrition": {
    "calories": 620,
    "protein": 32,
    "fat": 18,
    "carbs": 72
  },

  "createdAt": "2026-06-27T10:00:00Z",
  "updatedAt": "2026-06-27T10:00:00Z"
}
```

## Indexes

### Unique menu item reference

```js
db.menu_items.createIndex(
  { menuItemId: 1 },
  { unique: true }
);
```

Used to find a menu item by its SQL `menu_items.id`.

Example access patterns:

```text
GET /menu-items/:menuItemId
PATCH /menu-items/:menuItemId
```

---

### Restaurant menu

```js
db.menu_items.createIndex(
  { restaurantId: 1 }
);
```

Used to load all menu items for one restaurant.

Example access pattern:

```text
GET /restaurants/:restaurantId/menu
```

---

### Available restaurant menu

```js
db.menu_items.createIndex(
  { restaurantId: 1, isAvailable: 1 }
);
```

Used to load only currently available menu items for one restaurant.

Example access pattern:

```text
GET /restaurants/:restaurantId/menu?available=true
```

This index is also useful during checkout validation.
