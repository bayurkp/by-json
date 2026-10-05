# ByJSON Specification (Version 2.0)

> _Connected by JSON, structured ByJSON._

A practical, modern, and type-safe JSON standard for REST API responses and requests.

## What?

ByJSON is an open specification that defines how JSON payloads in REST APIs should be structured — covering both **requests** sent by the client and **responses** returned by the server.

## Why?

Every team invents their own JSON structure. Frontend and mobile developers end up writing custom adapters for every backend they integrate with. Existing standards either go too far or not far enough:

- [**JSON:API**](https://jsonapi.org/) is powerful but overly complex and verbose for most web and mobile applications.
- [**JSend**](https://github.com/omniti-labs/jsend) is refreshingly simple but lacks standardized conventions for pagination, validation paths, partial updates, and distributed sync.
- **Pure Bare Objects** discard universal response envelopes, making top-level metadata (tracing IDs, timestamps, pagination) difficult to transport reliably across diverse clients without polluting business models or relying solely on HTTP headers.

ByJSON v2 sits in the sweet spot: **predictable enough to automate client generation, ergonomic enough for modern frontend & mobile ecosystems, and robust enough for production scale.**

## Who?

- **Backend developers:** Provides a rock-solid contract for every endpoint you build — no more debating response shapes or pagination keys in code reviews.
- **Frontend & Mobile developers:** Allows you to write generic API clients with 100% type safety and zero-overhead Discriminated Unions in TypeScript, Swift, Kotlin, and Dart.
- **Team leads & Architects:** Eliminates API design bikeshedding and aligns naturally with modern cloud patterns, OpenAPI 3.1, and distributed systems.

## Table of Contents

- [1. Principles](#1-principles)
- [2. Naming Conventions](#2-naming-conventions)
  - [2.1 JSON Keys (camelCase)](#21-json-keys-camelcase)
  - [2.2 Persistence Decoupling](#22-persistence-decoupling)
  - [2.3 Error Codes (snake_case)](#23-error-codes-snake_case)
  - [2.4 URL Segments (kebab-case)](#24-url-segments-kebab-case)
- [3. URL & Endpoint Design](#3-url--endpoint-design)
  - [3.1 Resource URLs](#31-resource-urls)
  - [3.2 Process Endpoints](#32-process-endpoints)
  - [3.3 Endpoint Reference](#33-endpoint-reference)
- [4. Requests](#4-requests)
  - [4.1 Create](#41-create)
  - [4.2 Update (Full)](#42-update-full)
  - [4.3 Update (Partial)](#43-update-partial)
  - [4.4 Delete](#44-delete)
  - [4.5 Query Parameters](#45-query-parameters)
- [5. Responses](#5-responses)
  - [5.1 General Payload Structure](#51-general-payload-structure)
  - [5.2 TypeScript & Type-Safety Integration](#52-typescript--type-safety-integration)
  - [5.3 Success Responses](#53-success-responses)
  - [5.4 Error Responses](#54-error-responses)
- [6. Advanced Patterns](#6-advanced-patterns)
  - [6.1 File Resources](#61-file-resources)
  - [6.2 Aggregate & Metric Sub-Resources](#62-aggregate--metric-sub-resources)
  - [6.3 Offline-First Sync](#63-offline-first-sync)
  - [6.4 Multi-Audience API (Namespace Separation)](#64-multi-audience-api-namespace-separation)
  - [6.5 Idempotency Keys](#65-idempotency-keys)
  - [6.6 Identifier Strategy: UUID Selection (v7 vs v5 vs v4)](#66-identifier-strategy-uuid-selection-v7-vs-v5-vs-v4)
- [7. Reference Tables](#7-reference-tables)
  - [7.1 Error Code Reference](#71-error-code-reference)
  - [7.2 Filter Operator Reference](#72-filter-operator-reference)
  - [7.3 Meta Object Specification](#73-meta-object-specification)
- [License](#license)

---

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## 1. Principles

| Principle | Description |
| :--- | :--- |
| **Predictable Envelope** | Every response envelope **MUST** contain exactly four root keys: `success`, `data`, `error`, and `meta`. No missing keys, no guessing. |
| **Type-Safe Discrimination** | The `success` boolean acts as an explicit discriminator. When `success: true`, `data` is populated and `error` **MUST** be `null`. When `success: false`, `error` is populated and `data` **MUST** be `null`. |
| **Client-Centric Ergonomics** | Wire payloads use `camelCase`, perfectly matching modern TypeScript, Swift, Kotlin, and Dart ecosystems without manual property mapping. |
| **Simplicity in Relations** | Relational data is nested directly inside the parent entity as a **summary object** containing at minimum the resource `id` — avoiding disjoint `included` blocks. |
| **Layer Decoupling** | The API wire contract is strictly decoupled from database persistence. Database column conventions do not dictate public API contracts. |
| **Standard HTTP Semantics** | ByJSON honors native HTTP standards (e.g. `204 No Content` for standard deletes, RFC 9457-aligned errors, and standard status codes). |

---

## 2. Naming Conventions

### 2.1 JSON Keys (camelCase)

All keys in JSON request and response bodies **MUST** use `camelCase`.

```json
// ✅ Good
{
  "firstName": "Budi",
  "emailAddress": "budi@example.com",
  "authorId": 1,
  "durationMinutes": 15
}

// ❌ Avoid
{
  "first_name": "Budi",
  "EmailAddress": "budi@example.com",
  "author_id": 1,
  "duration_minutes": 15
}
```

### 2.2 Persistence Decoupling

ByJSON defines the contract exclusively for the **wire/transport layer** (HTTP request and response payloads). 

The internal persistence layer (such as relational database table and column schemas, which traditionally use `snake_case` in SQL engines like PostgreSQL and MySQL) is an internal implementation detail outside the scope of this specification. The application **SHOULD** decouple its persistence models from its API contract using an ORM or serialization mapping layer (e.g., Prisma `@map`, Jackson naming strategies, Pydantic aliases, or Go struct tags).

```
[ Database: SQL ]      --> duration_minutes (snake_case, plural tables)
        │
[ Backend App: ORM ]   --> Maps DB column to Entity property
        │
[ Wire: ByJSON API ]   --> durationMinutes (camelCase, client-ergonomic)
        │
[ Client: TS/Mobile ]  --> durationMinutes (native, zero-friction)
```

### 2.3 Error Codes (snake_case)

Machine-readable error codes **MUST** use `snake_case` (e.g., `validation_error`, `not_found`, `idempotency_key_in_progress`). Error codes represent constant machine tokens across all languages.

```json
// ✅ Good
"code": "validation_error"

// ❌ Avoid
"code": "ValidationError", "code": "VALIDATION_ERROR"
```

### 2.4 URL Segments (kebab-case)

All multi-word path segments in resource URLs **MUST** use `kebab-case`.

```
✅  /cooking-sessions, /recipe-reviews, /bulk-delete
❌  /cooking_sessions, /cookingSessions
```

---

## 3. URL & Endpoint Design

### 3.1 Resource URLs

Resource URLs **MUST** use plural nouns and `kebab-case`.

| Rule | Example |
| :--- | :--- |
| Use **plural nouns** | `/recipes` (not `/recipe`) |
| Use **kebab-case** for multi-word segments | `/cooking-sessions` |
| Use **nesting** only for strict ownership | `/authors/1/recipes` |
| Use **action URLs** for specific non-CRUD operations | `/recipes/{id}/bookmark` |

#### Ownership & Hierarchy

If a resource strictly belongs to a parent entity, its URI **SHOULD** reflect that ownership:

```http
GET /authors/{authorId}/recipes
```

Servers **MAY** support the contextual alias `me` as a shorthand for the authenticated caller:

```http
GET /users/me/recipes
```

If `me` is supported, it **MUST** be applied consistently across all user-owned resources in that namespace.

#### Anti-Collision Rule for Action Endpoints

Action endpoints modifying a single entity **MUST** be placed after `{id}`:

```http
✅ POST /recipes/{id}/bookmark
❌ POST /recipes/bookmark        (ambiguous: is "bookmark" an id or an action?)
```

Static aggregate read-only sub-resources that use `GET` (e.g., `/recipes/summary`) are permitted at the collection level when `{id}` is defined as a UUID, since a word segment can never collide with a valid UUID.

### 3.2 Process Endpoints

Operations that represent protocols, system transactions, or external receivers (which do not manage persistent CRUD entities) are **Process Endpoints**.

Process Endpoints **MUST** be grouped under a subsystem prefix and use a verb or event name:

```http
POST /{subsystem}/{verb-or-event}
```

| Endpoint | Subsystem | Purpose |
| :--- | :--- | :--- |
| `POST /auth/signin` | Authentication | Issue session / token |
| `POST /auth/signout` | Authentication | Revoke token |
| `POST /auth/refresh` | Authentication | Refresh credentials |
| `POST /webhooks/stripe` | Event Ingestion | Handle payment provider event |

Process endpoints **MUST** still return the standard ByJSON 4-key envelope. If an operation yields no return data (e.g. signout), `data` **MUST** be `null`.

### 3.3 Endpoint Reference

The examples throughout this specification use the following `Recipe` resource:

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/recipes` | List recipes (paginated) |
| `GET` | `/recipes/{id}` | Retrieve a single recipe |
| `POST` | `/recipes` | Create a new recipe |
| `PUT` | `/recipes/{id}` | Full update of a recipe |
| `PATCH` | `/recipes/{id}` | Partial update of a recipe |
| `DELETE` | `/recipes/{id}` | Delete a recipe |
| `POST` | `/recipes/{id}/bookmark` | Action: bookmark recipe |
| `GET` | `/recipes/summary` | Read-only aggregate metrics |
| `GET` | `/recipes/sync` | Delta sync feed (offline-first) |
| `POST` | `/recipes/sync` | Push sync changes (offline-first) |
| `POST` | `/recipes/bulk-delete` | Bulk delete multiple recipes |

---

## 4. Requests

When sending data (`POST`, `PUT`, `PATCH`), the request body **MUST** be a flat JSON object (with `Content-Type: application/json`). Foreign entities **MUST** be referenced by foreign key fields (e.g. `authorId`, `photoId`) rather than nested objects.

### 4.1 Create

`POST /recipes`

```json
{
  "title": "Nasi Goreng Spesial",
  "difficulty": "EASY",
  "durationMinutes": 15,
  "authorId": 1,
  "isPublished": true,
  "ingredients": [
    { "name": "Nasi Putih", "amount": 2 },
    { "name": "Bumbu Spesial", "amount": 1 }
  ]
}
```

### 4.2 Update (Full)

`PUT /recipes/1`

Replaces the entire resource. All writable fields **MUST** be provided.

```json
{
  "title": "Nasi Goreng Spesial (Extra Pedas)",
  "difficulty": "MEDIUM",
  "durationMinutes": 20,
  "authorId": 1,
  "isPublished": false,
  "ingredients": [
    { "name": "Nasi Putih", "amount": 2 },
    { "name": "Bumbu Spesial", "amount": 1 },
    { "name": "Cabai Rawit", "amount": 5 }
  ]
}
```

### 4.3 Update (Partial)

`PATCH /recipes/1`

Modifies only the provided fields:
- **Omitted fields** remain unchanged.
- **`null` values** explicitly clear nullable fields.

```json
{
  "durationMinutes": 25
}
```

### 4.4 Delete

`DELETE /recipes/1`

Delete requests do not carry a request body.

### 4.5 Query Parameters

All query parameter keys **MUST** use `camelCase`.

#### Pagination

ByJSON supports **offset-based** and **cursor-based** pagination. The server determines the strategy per endpoint.

- **Offset-based:** `?page=2&perPage=10` (1-indexed page).
- **Cursor-based:** `?cursor=eyJpZCI6MTB9&perPage=10`. Omit `cursor` on the initial request.

#### Sorting

Use `sort` with comma-separated fields. Prefix with `-` for descending, and optional `+` for ascending:

```http
GET /recipes?sort=+difficulty,-durationMinutes
```

#### Filtering

1. **Simple Equality:** Use the field name directly:
   ```http
   GET /recipes?difficulty=EASY&isPublished=true
   ```

2. **Comparison Operators (Dot Notation):**
   To remain 100% URL-safe and prevent WAF/load balancer rejection of square brackets, operators **MUST** use **Dot Notation** (`field.operator=value`):
   ```http
   GET /recipes?durationMinutes.gte=15&durationMinutes.lte=60
   ```
   *(See [Section 7.2](#72-filter-operator-reference) for the full operator list).*

3. **Multi-Value Filtering (Repeated Keys):**
   To avoid parsing ambiguities when values contain commas (such as cities, tags, or addresses), multi-value filters **MUST** use repeated keys:
   ```http
   GET /places?city.in=Jakarta&city.in=Washington,%20D.C.&city.in=London
   ```

#### Field Selection

A client may request specific fields via `fields` with comma-separated names (dot-notation for relations):

```http
GET /recipes?fields=id,title,durationMinutes,author.name
```

#### Full Example

```http
GET /recipes?page=1&perPage=10&sort=-durationMinutes&difficulty=EASY&durationMinutes.gte=15&fields=id,title,durationMinutes,author.name
```

---

## 5. Responses

### 5.1 General Payload Structure

Every ByJSON response envelope — success or failure — **MUST** contain exactly four root keys:

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "meta": {
    "requestId": "8f830a6e-4e8c-4a37-b6e5-22e70df4ab75",
    "timestamp": "2026-10-05T06:00:00Z"
  }
}
```

| Key | Type | Description |
| :--- | :--- | :--- |
| `success` | `boolean` | **Discriminator.** `true` for 2xx responses, `false` for 4xx/5xx responses. |
| `data` | `object \| array \| null` | Populated on success. **MUST** be `null` on error. |
| `error` | `object \| null` | Populated on error. **MUST** be `null` on success. |
| `meta` | `object` | **Always present.** Contains request context (`requestId`, `timestamp`, `pagination`, etc.). |

**Contract Rules:**

| State | `success` | `data` | `error` | `meta` |
| :--- | :--- | :--- | :--- | :--- |
| **Success (2xx)** | `true` | Object, Array, or `{}` | `null` | Always present |
| **Error (4xx/5xx)** | `false` | `null` | Error Object | Always present |

### 5.2 TypeScript & Type-Safety Integration

Because `success` is a boolean literal, ByJSON maps cleanly to a **Discriminated Union** in TypeScript and typed languages without runtime guessing:

```typescript
export interface ApiMeta {
  requestId: string;
  timestamp: string;
  pagination?: {
    type: "offset" | "cursor";
    perPage: number;
    hasNext: boolean;
    hasPrev: boolean;
    [key: string]: unknown;
  };
  [key: string]: unknown;
}

export interface ApiError {
  code: string;
  message: string;
  details?: Array<{
    field: string;
    code: string;
    message: string;
  }>;
}

export type ApiResponse<T> =
  | {
      success: true;
      data: T;
      error: null;
      meta: ApiMeta;
    }
  | {
      success: false;
      data: null;
      error: ApiError;
      meta: ApiMeta;
    };

// Ergonomic usage:
const response: ApiResponse<Recipe> = await api.getRecipe(1);

if (response.success) {
  // TypeScript automatically narrows:
  // response.data is Recipe (not null!)
  console.log(response.data.title);
} else {
  // response.error is ApiError (not null!)
  console.error(response.error.message);
}
```

---

### 5.3 Success Responses

#### A. Single Object

`GET /recipes/1`

```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "Nasi Goreng Spesial",
    "difficulty": "EASY",
    "durationMinutes": 15,
    "isPublished": true,
    "author": {
      "id": 1,
      "name": "Budi"
    }
  },
  "error": null,
  "meta": {
    "requestId": "8f830a6e-4e8c-4a37-b6e5-22e70df4ab75",
    "timestamp": "2026-10-05T06:00:00Z"
  }
}
```

#### B. Collection (No Pagination)

`GET /authors/1/recipes`

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "title": "Nasi Goreng Spesial",
      "durationMinutes": 15,
      "author": { "id": 1, "name": "Budi" }
    }
  ],
  "error": null,
  "meta": {
    "requestId": "764f2010-3c22-4a0b-9df0-94d3e52e5093",
    "timestamp": "2026-10-05T06:00:00Z"
  }
}
```

#### C. Collection with Pagination

`meta.pagination` uses a unified schema containing the discriminator `"type": "offset" | "cursor"` and normalized navigation flags (`hasNext`, `hasPrev`). Navigational `links` use relative paths to ensure reverse-proxy compatibility.

**Offset-based Example (`GET /recipes?page=1&perPage=10`):**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "title": "Nasi Goreng Spesial",
      "durationMinutes": 15
    }
  ],
  "error": null,
  "meta": {
    "requestId": "c138d4e2-629b-4e89-b8e7-df9b43eb72cd",
    "timestamp": "2026-10-05T06:00:00Z",
    "pagination": {
      "type": "offset",
      "page": 1,
      "perPage": 10,
      "totalItems": 45,
      "totalPages": 5,
      "hasNext": true,
      "hasPrev": false,
      "links": {
        "self": "/recipes?page=1&perPage=10",
        "first": "/recipes?page=1&perPage=10",
        "prev": null,
        "next": "/recipes?page=2&perPage=10",
        "last": "/recipes?page=5&perPage=10"
      }
    }
  }
}
```

**Cursor-based Example (`GET /recipes?cursor=eyJpZCI6MTB9&perPage=10`):**

```json
{
  "success": true,
  "data": [
    {
      "id": 11,
      "title": "Mie Aceh",
      "durationMinutes": 25
    }
  ],
  "error": null,
  "meta": {
    "requestId": "d249e5f3-730c-5f9a-c9f8-eg0c54fc83de",
    "timestamp": "2026-10-05T06:05:00Z",
    "pagination": {
      "type": "cursor",
      "perPage": 10,
      "hasNext": true,
      "hasPrev": true,
      "nextCursor": "eyJpZCI6MjB9",
      "prevCursor": "eyJpZCI6MTB9",
      "links": {
        "self": "/recipes?cursor=eyJpZCI6MTB9&perPage=10",
        "next": "/recipes?cursor=eyJpZCI6MjB9&perPage=10",
        "prev": "/recipes?cursor=eyJpZCI6MTB9&perPage=10"
      }
    }
  }
}
```

#### D. Created (HTTP 201)

`POST /recipes` → Returns the full representation of the newly created object.

```json
{
  "success": true,
  "data": {
    "id": 3,
    "title": "Ayam Penyet",
    "difficulty": "MEDIUM",
    "durationMinutes": 30,
    "isPublished": false,
    "author": { "id": 1, "name": "Budi" }
  },
  "error": null,
  "meta": {
    "requestId": "1d8b9e6a-7235-4309-8472-3c22b9df09d2",
    "timestamp": "2026-10-05T06:10:00Z"
  }
}
```

#### E. Updated (HTTP 200)

`PATCH /recipes/3` → Returns the updated resource.

```json
{
  "success": true,
  "data": {
    "id": 3,
    "title": "Ayam Penyet Sambal Ijo",
    "durationMinutes": 35
  },
  "error": null,
  "meta": {
    "requestId": "58498f39-df03-4f9e-a86d-0e42718cf2e7",
    "timestamp": "2026-10-05T06:15:00Z"
  }
}
```

#### F. Deleted

ByJSON supports two standard modes for delete responses:

1. **Standard Mode (RECOMMENDED): `HTTP 204 No Content`**
   Conforming with RFC 9110, the response has **no message body**. Frontend applications use their local component state for toast notifications (e.g. `toast.success(`${recipe.title} deleted`)`).
   
2. **Envelope Feedback Mode: `HTTP 200 OK`**
   If client architectures mandate an in-body confirmation, the server **MUST** return a standardized payload with the resource `id` and an optional human-readable `label`:
   ```json
   {
     "success": true,
     "data": {
       "id": 3,
       "label": "Ayam Penyet Sambal Ijo"
     },
     "error": null,
     "meta": {
       "requestId": "b1b01c10-2184-48de-824c-9f62cd8376de",
       "timestamp": "2026-10-05T06:20:00Z"
     }
   }
   ```

#### G. Action Endpoints (HTTP 200)

`POST /recipes/1/bookmark`

```json
{
  "success": true,
  "data": {
    "id": 1,
    "isBookmarked": true,
    "bookmarkCount": 42
  },
  "error": null,
  "meta": {
    "requestId": "8f830a6e-4e8c-4a37-b6e5-22e70df4ab75",
    "timestamp": "2026-10-05T06:25:00Z"
  }
}
```

#### H. Bulk Operations (HTTP 200 or 207 Multi-Status)

`POST /recipes/bulk-delete`

Bulk operations report per-item outcomes without breaking mutual exclusivity:

```json
{
  "success": true,
  "data": {
    "affectedCount": 2,
    "succeeded": [
      { "id": 1 },
      { "id": 2 }
    ],
    "failed": [
      {
        "id": 3,
        "code": "not_found",
        "message": "Recipe not found."
      }
    ]
  },
  "error": null,
  "meta": {
    "requestId": "9a12bc34-d5e6-4f78-a901-2345cd6789ef",
    "timestamp": "2026-10-05T06:30:00Z"
  }
}
```

#### I. Collection-Wide State Transitions (HTTP 200)

`POST /reviews/approve-all`

```json
{
  "success": true,
  "data": {
    "affectedCount": 12
  },
  "error": null,
  "meta": {
    "requestId": "7c3e1a2b-4d5f-6a7b-8c9d-0e1f2a3b4c5d",
    "timestamp": "2026-10-05T06:35:00Z"
  }
}
```

---

### 5.4 Error Responses

For failed requests (`HTTP 4xx` / `5xx`), `success` is `false` and `data` **MUST** be `null`.

#### A. Validation Errors (HTTP 422 Unprocessable Content)

Validation errors use **Dot Notation** for field paths, allowing zero-adapter binding with frontend form libraries (React Hook Form, Formik, Zod):

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "validation_error",
    "message": "The request data is invalid. Please check your input.",
    "details": [
      {
        "field": "title",
        "code": "missing_field",
        "message": "The title is required."
      },
      {
        "field": "durationMinutes",
        "code": "minimum_value",
        "message": "The duration must be at least 1 minute."
      },
      {
        "field": "ingredients.0.name",
        "code": "invalid_format",
        "message": "Ingredient name must not contain special characters."
      }
    ]
  },
  "meta": {
    "requestId": "a90184b2-c0cf-46d4-8395-927932e6a987",
    "timestamp": "2026-10-05T06:40:00Z"
  }
}
```

#### B. General Errors (HTTP 400 / 401 / 403 / 404 / 409 / 500)

`error.details` is omitted when field-level context is not applicable:

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "unauthorized",
    "message": "Your session has expired. Please sign in again."
  },
  "meta": {
    "requestId": "787c88ae-4f7f-4f05-8968-3d12d4a51eb2",
    "timestamp": "2026-10-05T06:45:00Z"
  }
}
```

---

## 6. Advanced Patterns

### 6.1 File Resources

Files are first-class resources managed under `/files`, decoupled from business entities:

```
status lifecycle: pending  ──>  completed  ──>  expired
```

#### Fields

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `string` (UUID) | Unique file identifier. |
| `status` | `string` | `pending`, `completed`, `failed`, `expired`. |
| `filename` | `string` | Original filename uploaded. |
| `contentType` | `string` | MIME type (e.g. `image/jpeg`). |
| `category` | `string` | Inferred server-side (`image`, `video`, `document`, `archive`, `other`). |
| `sizeBytes` | `integer` | File size in bytes. |
| `url` | `string \| null` | Public access CDN URL (`null` while `pending`). |
| `uploadUrl` | `string \| null` | Destination presigned URL (Presigned URL mode). |
| `uploadMethod`| `string \| null` | HTTP method for direct upload (typically `PUT`). |
| `expiresAt` | `string \| null` | Expiration timestamp for `uploadUrl`. |
| `createdAt` | `string` | Creation timestamp. |

#### Upload Workflows

1. **Explicit Systematic Model (Recommended for Media Pipelines & Live Previews):**
   - **Step 1:** Client calls `POST /files` with `{ filename, contentType, sizeBytes, purpose }`. Server returns `201 Created` with `uploadUrl` and `status: "pending"`.
   - **Step 2:** Client uploads binary directly to `uploadUrl` (S3/GCS/R2).
   - **Step 3:** Client calls `POST /files/{id}/complete`. Server verifies storage existence, triggers background resizing/scanning, updates `status: "completed"`, and returns the CDN `url`.
   - **Step 4:** Client attaches the file to a resource via `PATCH /recipes/1` with `{ "photoId": "uuid" }`.

2. **Cloud-Native Event-Driven Model (S3 Webhook):**
   - Client performs Step 1 and Step 2.
   - Cloud storage triggers a bucket event (`ObjectCreated`) directly to a backend webhook/lambda, which marks the file `completed`. The client skips Step 3 entirely.

3. **Direct Upload Model (Monoliths / Small Files):**
   - Client sends file directly to `POST /files` using `multipart/form-data`. The file is created with `status: "completed"` immediately.

---

### 6.2 Aggregate & Metric Sub-Resources

Endpoints returning computed summary metrics from a collection use `GET`:

```http
GET /recipes/summary
```

```json
{
  "success": true,
  "data": {
    "totalRecipes": 248,
    "averageDurationMinutes": 32,
    "mostCookedCategory": "main-course"
  },
  "error": null,
  "meta": {
    "requestId": "9c1a2b3c-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
    "timestamp": "2026-10-05T06:50:00Z"
  }
}
```

---

### 6.3 Offline-First Sync

For applications synchronizing local databases (SQLite/Room/Dexie) with the server:

#### Delta Sync (Pull)

`GET /recipes/sync?since=2026-09-01T00:00:00Z&limit=250`

**Critical Rules:**
1. **Batching Support:** Delta sync **MUST** support batch limits to prevent device Out-Of-Memory (OOM) and gateway timeouts on large backlogs.
2. **Deterministic Composite Cursor:** Delta sorting **MUST** use `(updatedAt ASC, id ASC)`. If multiple records share the same millisecond timestamp, `id` acts as the deterministic tie-breaker.
3. **Tombstones for Deletions:** Soft-deleted records **MUST** be returned as `{ "id": "...", "isDeleted": true, "updatedAt": "..." }`.
4. **Watermark Progression Rule:** `meta.serverTime` is the authoritative clock. The client **MUST NOT** update its local `since` watermark until **all batches have been completely fetched (`hasNext: false`)**.

```json
{
  "success": true,
  "data": [
    {
      "id": "01925b4a-...",
      "title": "Nasi Goreng Spesial",
      "updatedAt": "2026-09-10T08:30:00Z",
      "isDeleted": false
    },
    {
      "id": "01925b4b-...",
      "updatedAt": "2026-09-10T09:00:00Z",
      "isDeleted": true
    }
  ],
  "error": null,
  "meta": {
    "requestId": "...",
    "timestamp": "...",
    "serverTime": "2026-09-12T10:00:00Z",
    "pagination": {
      "type": "cursor",
      "perPage": 250,
      "hasNext": false,
      "nextCursor": null
    }
  }
}
```

---

### 6.4 Multi-Audience API (Namespace Separation)

URIs identify **data entities**, not caller roles:

```http
✅ GET /recipes         (row-level policy controls data visibility)
❌ GET /admin/recipes vs GET /client/my-recipes (for identical schemas)
```

When different audiences require fundamentally different response schemas, separate namespaces are recommended:

```http
GET /recipes        — Consumer API
GET /admin/recipes  — Backoffice API (includes internal audit fields)
```

---

### 6.5 Idempotency Keys

Clients **SHOULD** include an `Idempotency-Key: <UUID>` header on non-idempotent `POST` requests.

#### Server Processing Rules & Edge Cases:

1. **Distributed Lock & In-Flight Requests:**
   When Request 2 arrives while Request 1 is actively processing with the same key, the server **MUST** hold Request 2 for up to 2–3 seconds awaiting completion. If Request 1 completes, Request 2 receives the identical response. If the lock window expires, the server **MUST** return `409 Conflict` with `idempotency_key_in_progress` and a `Retry-After` header.
2. **Canonical Payload Hashing:**
   The server computes a canonical hash of the request path and body. If a request arrives with an identical key but a mismatched body, the server **MUST** return `409 Conflict` with `idempotency_key_conflict`.
3. **Transient Error Handling:**
   Transient server errors (`500`, `502`, `503`, `504`) **MUST NOT** be cached and **MUST** immediately release the idempotency lock to permit safe client retries.
4. **Tenant Scoping:**
   Keys **MUST** be scoped per authenticated user (`hash(userId + ":" + idempotencyKey)`) to prevent cross-account key collisions or hijacking.

---

### 6.6 Identifier Strategy: UUID Selection (v7 vs v5 vs v4)

Selecting the correct identifier strategy is essential for database health and distributed synchronization:

| Version | Mechanism | Primary Use Case |
| :--- | :--- | :--- |
| **UUID v7** (RFC 9562) | Time-ordered (48-bit Unix timestamp + random bits) | **Default choice for Primary Keys.** Prevents B-Tree index fragmentation and provides natural chronological sorting. |
| **UUID v5** | Name-based deterministic hash `SHA-1(namespace + name)` | **Immutable Natural Keys.** Junction tables (e.g. `user_bookmarks`), system-wide constants, or immutable content-addressed keys. |
| **UUID v4** | Purely random bits | Opaque tokens, one-time verification tokens, or non-indexed IDs. |

> [!CAUTION]
> **Never use mutable user attributes to generate UUID v5 Primary Keys:**
> Generating an entity ID using `uuidv5(email + phoneNumber)` creates a severe vulnerability: when a user updates their phone number or when a recycled phone number is registered by another user, deterministic collisions cause account overwrites or split-brain states. Use **UUID v7** for user entity IDs and enforce email/phone uniqueness using server database constraints and OTP verification.

---

## 7. Reference Tables

### 7.1 Error Code Reference

| Error Code | HTTP Status | Description |
| :--- | :--- | :--- |
| `bad_request` | 400 | Request body is malformed JSON or unparseable. |
| `validation_error` | 422 | Request body failed validation rules. See `details`. |
| `unauthorized` | 401 | Authentication is missing or token has expired. |
| `forbidden` | 403 | Authenticated user lacks permission. |
| `not_found` | 404 | The requested resource was not found. |
| `method_not_allowed` | 405 | HTTP method is not supported on this endpoint. |
| `conflict` | 409 | Request conflicts with current resource state. |
| `idempotency_key_conflict` | 409 | `Idempotency-Key` was reused with a different request payload. |
| `idempotency_key_in_progress` | 409 | A request with this key is currently in-flight. |
| `file_not_ready` | 409 | File cannot be attached because status is not `completed`. |
| `upload_url_expired` | 410 | Presigned `uploadUrl` has passed its expiration window. |
| `file_too_large` | 422 | File size exceeds configured purpose policy. |
| `unsupported_file_type` | 422 | File MIME type is disallowed for this purpose. |
| `too_many_requests` | 429 | Rate limit exceeded. Check `meta.retryAfter`. |
| `internal_error` | 500 | An unexpected server error occurred. |
| `service_unavailable` | 503 | Server temporarily unavailable. Check `meta.retryAfter`. |

### 7.2 Filter Operator Reference

Filter operators use **Dot Notation** on the parameter name (`field.operator=value`).

| Operator | Description | Example |
| :--- | :--- | :--- |
| `eq` | Equal (default when operator is omitted) | `difficulty=EASY` or `difficulty.eq=EASY` |
| `neq` | Not equal | `difficulty.neq=HARD` |
| `gt` | Greater than | `durationMinutes.gt=30` |
| `gte` | Greater than or equal | `durationMinutes.gte=15` |
| `lt` | Less than | `durationMinutes.lt=60` |
| `lte` | Less than or equal | `durationMinutes.lte=60` |
| `in` | In array of values (use repeated keys) | `city.in=Jakarta&city.in=Bandung` |
| `like` | Pattern match | `title.like=%goreng%` |

### 7.3 Meta Object Specification

The `meta` object is present in every response.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `requestId` | `string` | **Yes** | UUID for tracing and debugging. |
| `timestamp` | `string` | **Yes** | ISO 8601 timestamp of server execution. |
| `pagination` | `object` | No | Included on paginated lists. Contains `type`, `page`/`nextCursor`, `hasNext`, `hasPrev`, and `links`. |
| `serverTime` | `string` | No | Authoritative server clock (ISO 8601) for offline delta sync watermarks. |
| `retryAfter` | `integer` | No | Seconds to wait before retry (included in 429 and 503 responses). |
| `deprecation` | `object` | No | Contains `sunsetDate` (ISO 8601) and `successor` endpoint URL. |

---

## License

This project is licensed under the MIT License — see the [LICENSE.md](LICENSE.md) file for details.
