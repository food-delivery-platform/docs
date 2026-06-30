# Menu Service

**Runtime:** AWS Fargate (Node.js + Express) · 1–2 tasks  
**Domain:** Menu Management (restaurant-facing, write-heavy)

## Responsibility

Restaurant-facing CRUD for menus, categories, and availability. Also generates AI dish cards via AWS Bedrock when a restaurant adds a new item.

## Why Fargate (not Lambda)

- Write-heavy with image uploads to S3 (Lambda cold starts on image flows are costly)
- Shares the same ECS cluster as Delivery Service — zero extra ops overhead

## Key Operations

| Operation | Endpoint |
|-----------|----------|
| Create / update / delete menu item | `POST/PUT/DELETE /menus/items` |
| Manage categories | `POST/PUT/DELETE /menus/categories` |
| Toggle item availability | `PATCH /menus/items/{id}/availability` |
| Toggle restaurant open/closed | `PATCH /restaurants/{id}/availability` |
| Generate AI dish card | `POST /menus/items/ai-card` |

## AI Dish Card (Bedrock)

Restaurant provides item name or uploads a photo → Bedrock generates description, calories, and nutrition data → stored in `menu_items.ai_generated` JSONB column.

## Data Owned

| Store | Table | Role |
|-------|-------|------|
| Supabase | `menu_items` | Full menu item records including `ai_generated` JSONB |
| Supabase | `categories` | Menu category definitions per restaurant |
| S3 | `food-images/` | Uploaded dish photos |

## Integrations

- **AWS Bedrock** — LLM call for AI-assisted dish card generation
- **S3** — stores uploaded dish images; pre-signed URL returned to client
- **Catalog Service** — reads `menu_items` written here (read-only, separate service)

## Notes

Menu Service (write path) and Catalog Service (read path) are intentionally split: each can scale independently, and the write-heavy restaurant path does not compete with the high-read customer browsing path.
