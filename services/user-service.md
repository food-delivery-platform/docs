# User Service

**Runtime:** AWS Lambda (Node.js)  
**Domain:** Identity & Profile

## Responsibility

Handles user registration, authentication, profile management, address book, and role assignment. Bridges two auth providers: Firebase OAuth 2.0 for customers and AWS Cognito for restaurant/admin/courier staff.

## Key Operations

| Operation | Endpoint |
|-----------|----------|
| Register customer (Firebase) | `POST /users/register` |
| Exchange Firebase ID token for session | `POST /auth/customer/session` |
| Exchange Cognito JWT (staff login) | `POST /auth/staff/session` |
| Get / update profile | `GET/PATCH /users/{id}/profile` |
| Manage delivery addresses | `POST/PUT/DELETE /users/{id}/addresses` |
| Assign / update role | `PATCH /users/{id}/role` (Admin only) |

## Auth Model

| User Type | Provider | Token |
|-----------|----------|-------|
| Customer | Firebase OAuth 2.0 (Google / Apple / Facebook) | Firebase ID token → HTTP-only session cookie (Next.js) |
| Restaurant / Admin / Courier | AWS Cognito | JWT access token (1 h) + refresh token (30 d) |

## Data Owned

| Store | Table | Role |
|-------|-------|------|
| Supabase | `users` | Core user record (email, phone, role, cognito_sub, firebase_uid) |
| Supabase | `addresses` | Saved delivery addresses per user |
| Supabase | `profiles` | Extended profile (preferences, avatar) |

## Integrations

- **AWS Cognito** — user pool for staff roles; JWT verification on every request
- **Firebase Auth** — server-side token verification for customer JWTs
- **Secrets Manager** — Firebase service account key, Cognito client secret

## Concurrency

| Baseline | Burst |
|----------|-------|
| 200 concurrent | 500 concurrent |
