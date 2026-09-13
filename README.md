# ByJSON Specification

> _Connected by JSON, structured ByJSON._

A practical, balanced JSON standard for REST API responses and requests.

## What?

ByJSON is a specification that defines how JSON payloads in REST APIs should be structured — covering both **requests** sent by the client and **responses** returned by the server.

## Why?

Every team invents their own JSON structure. Frontend developers end up writing different parsers for every backend they integrate with. Existing standards either go too far or not far enough:

- [**JSON:API**](https://jsonapi.org/) is powerful but over-engineered for most use cases.
- [**JSend**](https://github.com/omniti-labs/jsend) is refreshingly simple but lacks conventions for pagination, validation errors, request format, and relational data.

ByJSON sits in the middle: **structured enough to be predictable, simple enough to be pleasant.** It borrows proven conventions to maintain consistency, while stripping away unnecessary complexity.

## Who?

- **Backend developers:** Provides a clear contract for every endpoint you build — no more debating response shapes in code review.
- **Frontend developers:** Allows you to write a single generic API client that handles every response identically — success or error, single object or paginated list.
- **Team leads:** Eliminates the "how should we format this?" conversation.

## Table of Contents

- [1. Principles](#1-principles)
- [2. Naming Conventions](#2-naming-conventions)
  - [2.1 JSON Keys](#21-json-keys)
  - [2.2 Error Codes](#22-error-codes)
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
  - [5.2 Success Responses](#52-success-responses)
  - [5.3 Error Responses](#53-error-responses)
- [6. Advanced Patterns](#6-advanced-patterns)
  - [6.1 File Resources](#61-file-resources)
  - [6.2 Aggregate & Metric Sub-Resources](#62-aggregate--metric-sub-resources)
  - [6.3 Offline-First Sync](#63-offline-first-sync)
  - [6.4 Multi-Audience API (Namespace Separation)](#64-multi-audience-api-namespace-separation)
  - [6.5 Idempotency Keys](#65-idempotency-keys)
- [7. Reference Tables](#7-reference-tables)
  - [7.1 Error Code Reference](#71-error-code-reference)
  - [7.2 Filter Operator Reference](#72-filter-operator-reference)
  - [7.3 Meta Object Specification](#73-meta-object-specification)
- [License](#license)

---

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## 1. Principles

| Principle                 | Description                                                                                           |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Predictable Structure** | Every response **MUST** have the same three root keys: `data`, `error`, and `meta`. No guessing.      |
| **Simplicity**            | Relational data is nested directly inside the parent object — no separate `included` block.           |
| **Clarity**               | Success and error payloads are mutually exclusive. If one is populated, the other **MUST** be `null`. |
| **Agnosticism**           | The spec is agnostic to audience (end-user, admin, B2B, B2C, C2C). URIs identify data entities, not the perspective of the caller. Authorization policy — not the URL — determines what data is visible to whom. |

## 2. Naming Conventions

### 2.1 JSON Keys

All keys in request and response bodies **MUST** use `snake_case`.

_Note: `snake_case` is chosen for maximum consistency with backend database columns and system variables._

```
✅  first_name, email_address, author_id
❌  firstName, EmailAddress, authorID
```

### 2.2 Error Codes

Error codes **MUST** also use `snake_case` (e.g. `validation_error`) for maximum consistency.

```
✅  validation_error, not_found, bad_request
❌  VALIDATION_ERROR, NotFound, BadRequest
```

## 3. URL & Endpoint Design

### 3.1 Resource URLs

_Note: URL versioning (e.g., `/v1/`) is **RECOMMENDED** but falls outside the scope of this JSON specification._

| Rule                                        | Example                                 |
| ------------------------------------------- | --------------------------------------- |
| Use **plural nouns**                        | `/recipes` (not `/recipe`)              |
| Use **kebab-case** for multi-word resources | `/cooking-steps` (not `/cooking_steps`) |
| Use **nesting** only for strict ownership   | `/authors/1/recipes`                    |
| Use **action URLs** for specific operations | `/recipes/1/bookmark`                   |

Resource URLs **MUST** use plural nouns and **MUST** use kebab-case for multi-word segments.

#### Ownership & Hierarchy

If a resource belongs to a parent entity, its URI **SHOULD** reflect that ownership through nesting:

```
GET /users/{user_id}/recipes     — recipes belonging to a specific user
GET /authors/{author_id}/recipes — recipes belonging to a specific author
```

Server **MAY** support the alias `me` as a contextual substitute for `{user_id}` when the caller is an authenticated user accessing their own data:

```
GET /users/me/recipes     — equivalent to /users/{caller_user_id}/recipes
```

If `me` is supported, it **MUST** be applied consistently across all resources in the same namespace — not selectively on some endpoints and not others.

Alternatively, when an API exclusively serves a single authenticated owner (i.e., a client can only ever access their own data), implicit scoping via the auth token is valid — provided no admin or cross-user variant of the same endpoint exists in the same namespace:

```
✅ GET /recipes          (scoped implicitly to the calling user via JWT — valid if no admin /recipes exists)
✅ GET /users/{id}/recipes  (explicit hierarchy — always valid regardless of audience)
❌ GET /recipes          (implicit user scope) + GET /recipes (admin all-user scope) — ambiguous, MUST NOT exist in the same namespace
```

See [Section 6.4](#64-multi-audience-api-namespace-separation) for handling APIs that serve multiple audiences.

#### Anti-Collision Rule for Action Endpoints

Action endpoints **MUST** be placed after `{id}`, never directly after the collection name. This prevents routing ambiguity with `GET /{resource}/{id}`.

```
✅ POST /recipes/{id}/bookmark    — action after {id}
❌ POST /recipes/bookmark         — ambiguous: is "bookmark" an {id} or an action?
```

This rule does **NOT** apply to read-only sub-resources (aggregates and metrics). Static path segments that return computed data via `GET` and do not mutate state are not considered "actions" under this rule:

```
✅ GET /reviews/pending-count      — read-only aggregate, not a state-mutating action
✅ GET /recipes/summary            — read-only metric
✅ GET /cooking-sessions/sync      — read-only delta sync feed
```

_Note: `{id}` **SHOULD** be a UUID. Using UUIDs as identifiers eliminates routing collision risk entirely — a static segment like `summary` can never be confused for a valid UUID._

### 3.2 Process Endpoints

Not all API functionality can be modelled as CRUD operations on data entities. **Process Endpoints** represent protocols, system operations, or subsystems that do not have a data resource as their primary subject.

#### Resource Endpoints vs Process Endpoints

| | Resource Endpoint | Process Endpoint |
| --- | --- | --- |
| **Subject** | A data entity with a lifecycle (create, read, update, delete) | A protocol, operation, or system interaction |
| **Verb in URL** | ❌ MUST NOT use verb — HTTP method carries the verb | ✅ MAY use verb/event name as the action segment |
| **Example** | `POST /cooking-sessions` (not `/cooking-sessions/start`) | `POST /auth/signin` |

**Rule:** If a `POST` operation creates a new data record that can later be retrieved by `GET /{resource}/{id}` or deleted by `DELETE /{resource}/{id}`, it is a **Resource Endpoint** and MUST follow standard CRUD conventions — the verb belongs in the HTTP Method, not the URL.

```
✅  POST /cooking-sessions      — creates a session record; retrievable later
❌  POST /cooking-sessions/start — verb in URL; violates Resource Endpoint rules

✅  POST /auth/signin         — no persistent resource created; auth protocol
✅  POST /auth/logout         — triggers token revocation; no resource entity
✅  POST /webhooks/stripe     — receives external event; not a user-created resource
```

#### Process Endpoint URI Format

Process Endpoints **MUST** be grouped under a subsystem prefix and **MUST** use a verb or event name as the action segment:

```
POST /{subsystem}/{verb-or-event}
```

Examples:

| Endpoint | Subsystem | Classification |
| --- | --- | --- |
| `POST /auth/signin` | Authentication protocol | Process |
| `POST /auth/signup` | Authentication protocol | Process |
| `POST /auth/refresh` | Token refresh protocol | Process |
| `POST /auth/logout` | Token revocation | Process |
| `POST /webhooks/{provider}` | External event receiver | Process |

#### Response Contract for Process Endpoints

Process Endpoints **MUST** still return the standard ByJSON envelope (`data`, `error`, `meta`). For operations that do not return a meaningful resource object (e.g., logout), `data` **MUST** be `null`.

### 3.3 Endpoint Reference

The examples throughout this spec use the following `Recipe` and `Author` resources:

| Method   | Endpoint                        | Description                          |
| -------- | ------------------------------- | ------------------------------------ |
| `GET`    | `/recipes`                      | List all recipes                     |
| `GET`    | `/recipes/{id}`                 | Get a single recipe                  |
| `POST`   | `/recipes`                      | Create a new recipe                  |
| `PUT`    | `/recipes/{id}`                 | Full update a recipe                 |
| `PATCH`  | `/recipes/{id}`                 | Partial update a recipe              |
| `DELETE` | `/recipes/{id}`                 | Delete a recipe                      |
| `GET`    | `/authors/{id}/recipes`         | List recipes by a specific author    |
| `POST`   | `/recipes/{id}/bookmark`        | Bookmark a specific recipe (action)  |
| `GET`    | `/recipes/summary`              | Aggregate metrics across recipes     |
| `GET`    | `/recipes/sync`                 | Delta sync feed for offline clients  |
| `POST`   | `/recipes/bulk-delete`          | Bulk delete by ID list               |
| `POST`   | `/recipes/archive-all`          | Collection-wide state transition     |
| `POST`   | `/auth/signin`                  | Authenticate user (process endpoint) |

## 4. Requests

When sending data to the server (`POST`, `PUT`, `PATCH`), the request body **MUST** be a flat JSON object. Relationships **MUST** be referenced by their foreign key (e.g., `author_id`) rather than nested objects.

### 4.1 Create

`POST /recipes`

```json
{
  "title": "Nasi Goreng Spesial",
  "difficulty": "EASY",
  "duration_minutes": 15,
  "author_id": 1,
  "is_published": true,
  "ingredients": [
    { "name": "Nasi Putih", "amount": 2 },
    { "name": "Bumbu Nasi Goreng", "amount": 1 }
  ]
}
```

### 4.2 Update (Full)

`PUT /recipes/1`

A full update replaces the entire resource. All writable fields **MUST** be provided.

```json
{
  "title": "Nasi Goreng Spesial (Extra Pedas)",
  "difficulty": "MEDIUM",
  "duration_minutes": 20,
  "author_id": 1,
  "is_published": false,
  "ingredients": [
    { "name": "Nasi Putih", "amount": 2 },
    { "name": "Bumbu Nasi Goreng", "amount": 1 },
    { "name": "Cabai Rawit", "amount": 5 }
  ]
}
```

### 4.3 Update (Partial)

`PATCH /recipes/1`

A partial update modifies only the fields provided.

- **Omitted fields** remain unchanged.
- **`null` values** explicitly unset or remove the field/relationship (if the field is nullable).

_Note for implementers:_ Distinguishing between an omitted field and an explicit `null` value can be challenging in strongly-typed languages (like Go or Java). It typically requires pointer types (e.g., `*string`) or field presence wrappers.

```json
{
  "title": "Nasi Goreng Spesial (Extra Pedas)"
}
```

### 4.4 Delete

`DELETE /recipes/1`

Delete operations do not require a JSON request body.

### 4.5 Query Parameters

When retrieving collections, use query parameters for pagination, sorting, filtering, and field selection.

#### Pagination

Both **offset-based** and **cursor-based** pagination are supported. **The server determines the strategy per endpoint**; the client does not negotiate this.

**Guidance:**

- **Use offset-based** when the client needs to navigate to specific pages (e.g., "jump to page 5") or needs to display total counts/pages.
- **Use cursor-based** when the client uses infinite scroll / "load more", or for massive/fast-changing datasets where query performance is prioritized over absolute navigation.

**Constraint:**
Cursor pagination **MUST** be used alongside a deterministic sort order that has a unique tiebreaker (usually `id`). Sorting solely by non-unique fields (e.g., `created_at`) will cause items to be skipped or duplicated between pages. Cursors **SHOULD** also be opaque (e.g., base64 encoded strings) to prevent clients from depending on internal data structures.

**Offset-based Parameters:**

| Parameter  | Description              | Example        |
| ---------- | ------------------------ | -------------- |
| `page`     | Page number (1-indexed)  | `?page=2`      |
| `per_page` | Number of items per page | `?per_page=10` |

**Cursor-based Parameters:**

| Parameter  | Description                                      | Example                |
| ---------- | ------------------------------------------------ | ---------------------- |
| `cursor`   | The pointer to a specific item for the next page | `?cursor=eyJpZCI6MTB9` |
| `per_page` | Number of items per page                         | `?per_page=10`         |

_Note: For the very first request in cursor pagination, omit the `cursor` parameter entirely (e.g., `GET /recipes?per_page=10`)._

#### Sorting

Use the `sort` parameter with comma-separated field names. Prefix a field with `-` (minus) for descending order, and optionally `+` (plus) for explicit ascending order. Fields without a prefix are sorted in ascending order by default.

```
GET /recipes?sort=+difficulty,-duration_minutes
```

The above example sorts by `difficulty` ascending first, then by `duration_minutes` descending. Sort fields are applied in the order specified.

| Type          | Example                               |
| ------------- | ------------------------------------- |
| Single (asc)  | `?sort=+duration_minutes`             |
| Single (desc) | `?sort=-duration_minutes`             |
| Multi column  | `?sort=+difficulty,-duration_minutes` |

If the server does not support the requested sort field, it **MUST** return `400 Bad Request`.

#### Filtering

For simple equality, use the field name directly as a query parameter (this is the default behavior and is equivalent to the `[eq]` operator):

```
GET /recipes?difficulty=EASY&is_published=true
```

For advanced filtering (range, comparison), use LHS Brackets. The operator is placed inside brackets on the field name:

```
GET /recipes?duration_minutes[gte]=15&duration_minutes[lte]=60
```

See [Section 7.2](#72-filter-operator-reference) for the full list of supported operators.

#### Field Selection

A client may request that only specific fields be returned in the response by using the `fields` parameter with a comma-separated list of field names. For nested relationships, use dot notation (e.g., `author.name`).

```
GET /recipes?fields=id,title,difficulty,author.name
```

If `fields` is specified, the server **MUST NOT** include additional fields beyond those requested. If `fields` is omitted, the server **MAY** return all fields.

#### Full Example

```
GET /recipes?page=1&per_page=10&sort=-duration_minutes&difficulty=EASY&duration_minutes[gte]=15&fields=id,title,duration_minutes,author.name
```

## 5. Responses

### 5.1 General Payload Structure

Every response — whether success or failure — **MUST** contain exactly three root-level keys:

```json
{
  "data": "...(object, array, or null)",
  "error": "...(object or null)",
  "meta": { "request_id": "...", "timestamp": "..." }
}
```

**Rules:**

| Condition | `data`          | `error` | `meta`             |
| --------- | --------------- | ------- | ------------------ |
| Success   | Object or Array | `null`  | **Always present** |
| Error     | `null`          | Object  | **Always present** |

The `meta` object is **always present** in both success and error responses. It provides debugging context (`request_id`, `timestamp`) that is useful regardless of the outcome.

### 5.2 Success Responses

For successful operations (HTTP `2xx`), `error` **MUST** be `null`.

Relational data (like `author`) is always embedded as a **summary object** directly inside the parent entity. This avoids the need for a separate `included` block while keeping the payload concise. A summary object **MUST** always contain the resource's `id` field, and **MAY** contain other identifying fields (like `name` or `title`).

#### A. Single Object

`GET /recipes/1`

```json
{
  "data": {
    "id": 1,
    "title": "Nasi Goreng Spesial",
    "difficulty": "EASY",
    "duration_minutes": 15,
    "is_published": true,
    "author": {
      "id": 1,
      "name": "Budi"
    }
  },
  "error": null,
  "meta": {
    "request_id": "8f830a6e-4e8c-4a37-b6e5-22e70df4ab75",
    "timestamp": "2026-07-10T08:12:06Z"
  }
}
```

#### B. Collection (No Pagination)

`GET /authors/1/recipes`

```json
{
  "data": [
    {
      "id": 1,
      "title": "Nasi Goreng Spesial",
      "difficulty": "EASY",
      "duration_minutes": 15,
      "is_published": true,
      "author": {
        "id": 1,
        "name": "Budi"
      }
    },
    {
      "id": 2,
      "title": "Mie Tek-Tek",
      "difficulty": "MEDIUM",
      "duration_minutes": 25,
      "is_published": true,
      "author": {
        "id": 1,
        "name": "Budi"
      }
    }
  ],
  "error": null,
  "meta": {
    "request_id": "764f2010-3c22-4a0b-9df0-94d3e52e5093",
    "timestamp": "2026-07-10T08:12:06Z"
  }
}
```

#### C. Collection with Pagination

**Offset-based Example:**

`GET /recipes?page=1&per_page=10`

```json
{
  "data": [
    {
      "id": 1,
      "title": "Nasi Goreng Spesial",
      "difficulty": "EASY",
      "duration_minutes": 15,
      "is_published": true,
      "author": {
        "id": 1,
        "name": "Budi"
      }
    }
  ],
  "error": null,
  "meta": {
    "request_id": "c138d4e2-629b-4e89-b8e7-df9b43eb72cd",
    "timestamp": "2026-07-10T08:12:06Z",
    "pagination": {
      "current_page": 1,
      "per_page": 10,
      "total_items": 45,
      "total_pages": 5,
      "links": {
        "self": "https://api.domain.com/recipes?page=1&per_page=10",
        "first": "https://api.domain.com/recipes?page=1&per_page=10",
        "prev": null,
        "next": "https://api.domain.com/recipes?page=2&per_page=10",
        "last": "https://api.domain.com/recipes?page=5&per_page=10"
      }
    }
  }
}
```

**Cursor-based Example:**

`GET /recipes?cursor=eyJpZCI6MTB9&per_page=10`

```json
{
  "data": [
    {
      "id": 1,
      "title": "Nasi Goreng Spesial",
      "difficulty": "EASY",
      "duration_minutes": 15,
      "is_published": true,
      "author": {
        "id": 1,
        "name": "Budi"
      }
    }
  ],
  "error": null,
  "meta": {
    "request_id": "d249e5f3-730c-5f9a-c9f8-eg0c54fc83de",
    "timestamp": "2026-07-10T08:15:00Z",
    "pagination": {
      "per_page": 10,
      "has_more": true,
      "cursors": {
        "next": "eyJpZCI6MTB9",
        "prev": null
      },
      "links": {
        "self": "https://api.domain.com/recipes?per_page=10",
        "next": "https://api.domain.com/recipes?cursor=eyJpZCI6MTB9&per_page=10",
        "prev": null
      }
    }
  }
}
```

#### D. Created (HTTP 201)

`POST /recipes` → Returns the newly created object with all server-generated fields.

```json
{
  "data": {
    "id": 3,
    "title": "Ayam Penyet",
    "difficulty": "MEDIUM",
    "duration_minutes": 30,
    "is_published": false,
    "author": {
      "id": 1,
      "name": "Budi"
    }
  },
  "error": null,
  "meta": {
    "request_id": "1d8b9e6a-7235-4309-8472-3c22b9df09d2",
    "timestamp": "2026-07-10T08:20:00Z"
  }
}
```

#### E. Updated (HTTP 200)

`PATCH /recipes/3` → Returns the most up-to-date representation of the object.

```json
{
  "data": {
    "id": 3,
    "title": "Ayam Penyet Goreng",
    "difficulty": "MEDIUM",
    "duration_minutes": 35,
    "is_published": true,
    "author": {
      "id": 1,
      "name": "Budi"
    }
  },
  "error": null,
  "meta": {
    "request_id": "58498f39-df03-4f9e-a86d-0e42718cf2e7",
    "timestamp": "2026-07-10T08:25:00Z"
  }
}
```

#### F. Deleted (HTTP 200)

`DELETE /recipes/3` → Returns a minimal representation of the object that was just deleted. Returning a full object for a deleted resource is wasteful and semantically confusing. Provide only enough identifying fields (like `id` and `title`) for the client to confirm the deletion and update the UI.

```json
{
  "data": {
    "id": 3,
    "title": "Ayam Penyet Goreng"
  },
  "error": null,
  "meta": {
    "request_id": "b1b01c10-2184-48de-824c-9f62cd8376de",
    "timestamp": "2026-07-10T08:30:00Z"
  }
}
```

#### G. Action Endpoints (HTTP 200)

Action endpoints that modify the state of a single resource (e.g., `POST /recipes/1/bookmark`) **MUST** return the updated representation of that resource in `data`, identical to a `PATCH` response.

Action endpoints that do not correspond to a single resource's state (e.g., `POST /recipes/1/send-to-email` or `/auth/logout`) **MUST** return `null` in the `data` field, relying purely on the HTTP status code to convey success.

`POST /recipes/1/bookmark`

```json
{
  "data": {
    "id": 1,
    "title": "Nasi Goreng Spesial",
    "is_bookmarked": true,
    "bookmark_count": 42
  },
  "error": null,
  "meta": {
    "request_id": "8f830a6e-4e8c-4a37-b6e5-22e70df4ab75",
    "timestamp": "2026-07-10T08:35:00Z"
  }
}
```

#### H. Bulk Operations (HTTP 207)

Bulk operations (Bulk Create, Update, or Delete) can result in partial failures. The request itself is successful, but individual items may fail. To represent this while maintaining the mutual exclusivity of `data` and `error`, bulk endpoints **SHOULD** use the HTTP `207 Multi-Status` code and return `succeeded` and `failed` arrays inside the `data` object.

The `error` object **MUST** remain `null` because there was no request-level failure (e.g., malformed JSON or unauthorized access).

`POST /recipes/bulk-delete`

```json
{
  "data": {
    "succeeded": [
      { "id": 1, "title": "Nasi Goreng Spesial" },
      { "id": 2, "title": "Mie Tek-Tek" }
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
    "request_id": "9a12bc34-d5e6-4f78-a901-2345cd6789ef",
    "timestamp": "2026-07-10T08:40:00Z"
  }
}
```

#### I. Collection-Wide State Transitions (HTTP 200)

Some operations change the state of an entire collection (or a filtered subset) without requiring individual IDs — for example, marking all pending reviews as approved, or archiving all completed recipes.

These are distinct from Bulk Operations (Section H) because the client does not enumerate individual items. They are also distinct from Resource Endpoint actions (Section G) because there is no single `{id}` being acted upon.

Collection-Wide State Transitions **MUST** use `POST` (not `PATCH`), because the client is issuing a command — not sending a delta payload of field changes.

Format: `POST /{resource}/{collection-action}`

The response **MUST** include `affected_count` in `data` so the client can update its local state without a separate refetch.

`POST /reviews/approve-all`

Request body (optional filter scope):
```json
{
  "before_date": "2026-09-01T00:00:00Z"
}
```

Response:
```json
{
  "data": {
    "affected_count": 12
  },
  "error": null,
  "meta": {
    "request_id": "7c3e1a2b-4d5f-6a7b-8c9d-0e1f2a3b4c5d",
    "timestamp": "2026-09-12T08:00:00Z"
  }
}
```

| Pattern | Example | Classification |
| --- | --- | --- |
| Mark all as approved | `POST /reviews/approve-all` | Collection-Wide Transition |
| Archive all completed | `POST /recipes/archive-completed` | Collection-Wide Transition |
| Bulk delete by IDs | `POST /recipes/bulk-delete` + body with ID list | Bulk Operation (Section H) |
| Update one item | `PATCH /reviews/{id}` + delta body | Standard Partial Update |

### 5.3 Error Responses

For failed operations (HTTP `4xx` / `5xx`), `data` **MUST** be `null`.

#### A. Validation Errors (HTTP 422)

Used when the request body fails validation rules. The `details` array points to the exact location of each error using the `field` key.

**Field Path Notation:** Validation errors **MUST** use JSON Pointer (RFC 6901) notation for nested objects and array items (e.g., `/shipping_address/city_name`, `/ingredients/0/name`).

```json
{
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
        "field": "duration_minutes",
        "code": "minimum_value",
        "message": "The duration must be at least 1 minute."
      },
      {
        "field": "/ingredients/0/name",
        "code": "invalid_format",
        "message": "Ingredient name must not contain special characters."
      }
    ]
  },
  "meta": {
    "request_id": "a90184b2-c0cf-46d4-8395-927932e6a987",
    "timestamp": "2026-07-10T08:40:00Z"
  }
}
```

#### B. General Errors (HTTP 400 / 401 / 403 / 404 / 500)

Used for malformed requests, authentication, authorization, not found, or server errors. The `details` array is omitted since field-level information is not applicable.

```json
{
  "data": null,
  "error": {
    "code": "unauthorized",
    "message": "Your session has expired. Please log in again."
  },
  "meta": {
    "request_id": "787c88ae-4f7f-4f05-8968-3d12d4a51eb2",
    "timestamp": "2026-07-10T08:45:00Z"
  }
}
```

## 6. Advanced Patterns

### 6.1 File Resources

Files — images, videos, documents, archives, or any binary type — are **first-class resources** represented under a single `/files` endpoint. They are differentiated by the `category` field (derived from `content_type`), not by separate endpoints.

**Key principles:**

- Upload never carries raw binary inside the main JSON envelope. JSON is only used for negotiation (request upload permission, complete upload, retrieve metadata). Binary transfer happens in a separate request, outside the JSON envelope.
- File resources have their own lifecycle (`pending` → `completed` → `failed`/`expired`), independent of whichever resource will eventually reference them.

**Two upload modes are supported.** The implementation team **MUST** choose one and document their choice. Both modes produce the same resource representation in responses — clients consuming data do not need to know which mode was used during upload.

| Condition                                                                | Recommended Mode            |
| ------------------------------------------------------------------------ | --------------------------- |
| Large files, high volume, no need for synchronous server-side processing | Presigned URL Model (see below) |
| Small files, need immediate validation/processing, simple monolith setup | Direct Upload Model (see below) |

#### Resource Structure

#### Fields

| Field           | Type             | Description                                                                                |
| --------------- | ---------------- | ------------------------------------------------------------------------------------------ |
| `id`            | `string` (UUID)  | Unique file identifier                                                                     |
| `status`        | `string`         | `pending`, `completed`, `failed`, `expired`                                                |
| `filename`      | `string`         | Original filename from the client                                                          |
| `content_type`  | `string`         | MIME type (e.g., `image/jpeg`, `application/pdf`)                                          |
| `category`      | `string`         | Derived from `content_type` server-side: `image`, `video`, `document`, `archive`, `other`  |
| `size_bytes`    | `integer`        | File size in bytes                                                                         |
| `url`           | `string \| null` | Public/access URL for the file. `null` while status is `pending`                           |
| `upload_url`    | `string \| null` | Destination URL for upload. Only present when `status = pending` (Presigned URL mode only) |
| `upload_method` | `string \| null` | HTTP method for upload, typically `PUT` (Presigned URL mode only)                          |
| `expires_at`    | `string \| null` | ISO 8601 expiration timestamp for `upload_url`. `null` after `status = completed`          |
| `created_at`    | `string`         | ISO 8601 timestamp when the resource was created                                           |

`id` uses UUID (not auto-increment integer) intentionally — this prevents collision with static path segments (see Section 5.7) and prevents files from being guessable or enumerable.

#### Category Rules

The server **MUST** determine `category` from `content_type`. The client must not send it manually.

| `content_type` prefix/value                                                                | `category` |
| ------------------------------------------------------------------------------------------ | ---------- |
| `image/*`                                                                                  | `image`    |
| `video/*`                                                                                  | `video`    |
| `application/pdf`, `application/msword`, `application/vnd.openxmlformats-officedocument.*` | `document` |
| `application/zip`, `application/x-rar-compressed`, `application/x-tar`                     | `archive`  |
| anything else                                                                              | `other`    |

#### Endpoints

| Method   | Endpoint               | Description                                                                                                               |
| -------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `POST`   | `/files`               | Register intent to upload a new file (creates resource with `status = pending`), or upload directly in Direct Upload mode |
| `POST`   | `/files/{id}/complete` | Mark the upload as complete after binary transfer (Presigned URL mode only)                                               |
| `GET`    | `/files/{id}`          | Retrieve metadata of a file                                                                                               |
| `DELETE` | `/files/{id}`          | Delete a file (soft delete recommended)                                                                                   |

All action endpoints (`/complete`) appear **after** `{id}`, never directly after the collection — this avoids routing ambiguity with `GET/DELETE /files/{id}` (see [Routing Rules](#routing-rules) below).

#### Presigned URL Model

A four-step flow where binary transfer is offloaded to a separate URL (object storage or a dedicated upload endpoint on the server itself).

**Guidance:**

| Condition                                                                | Recommended Mode      |
| ------------------------------------------------------------------------ | --------------------- |
| Large files, high volume, no need for synchronous server-side processing | Presigned URL Model   |
| Small files, need immediate validation/processing, simple monolith setup | Direct Upload Model   |

#### Step 1 — Register upload intent

`POST /files`

```json
{
  "filename": "nasi-goreng.jpg",
  "content_type": "image/jpeg",
  "size_bytes": 204800
}
```

The server **MUST** validate `content_type` and `size_bytes` against policy (see Section 5.6) before generating `upload_url`.

Response (`201 Created`):

```json
{
  "data": {
    "id": "9f1c2e3a-1b2c-4d5e-8f9a-0b1c2d3e4f5a",
    "status": "pending",
    "filename": "nasi-goreng.jpg",
    "content_type": "image/jpeg",
    "category": "image",
    "size_bytes": 204800,
    "url": null,
    "upload_url": "https://storage.domain.com/uploads/9f1c2e3a...?signature=...",
    "upload_method": "PUT",
    "expires_at": "2026-07-10T09:15:00Z",
    "created_at": "2026-07-10T09:00:00Z"
  },
  "error": null,
  "meta": {
    "request_id": "b3e1f2a4-5c6d-7e8f-9a0b-1c2d3e4f5a6b",
    "timestamp": "2026-07-10T09:00:00Z"
  }
}
```

#### Step 2 — Upload binary

The client sends the file directly to `upload_url` using `upload_method` (typically `PUT`, raw binary body — **not** `multipart/form-data`). This request is **outside** the ByJSON JSON envelope.

The destination may be external object storage (S3, GCS, R2) or a dedicated endpoint on the application server itself, depending on the team's infrastructure. The main JSON API endpoints (`/recipes`, `/authors`, etc., including `/files` itself) **MUST NOT** accept raw binary or `multipart/form-data` in their body when using this mode.

#### Step 3 — Complete

`POST /files/{id}/complete`

The server **MUST** verify the file's actual existence in the destination storage (not merely trust the client's claim) before changing `status` to `completed`.

Response (`200 OK`):

```json
{
  "data": {
    "id": "9f1c2e3a-1b2c-4d5e-8f9a-0b1c2d3e4f5a",
    "status": "completed",
    "filename": "nasi-goreng.jpg",
    "content_type": "image/jpeg",
    "category": "image",
    "size_bytes": 204800,
    "url": "https://cdn.domain.com/files/9f1c2e3a....jpg",
    "upload_url": null,
    "upload_method": null,
    "expires_at": null,
    "created_at": "2026-07-10T09:00:00Z"
  },
  "error": null,
  "meta": {
    "request_id": "c4f2a3b5-6d7e-8f9a-0b1c-2d3e4f5a6b7c",
    "timestamp": "2026-07-10T09:01:30Z"
  }
}
```

If verification fails (file not found in storage), the server **MUST** return error `file_verification_failed` and `status` remains `pending` or changes to `failed` depending on the team's retry policy.

#### Step 4 — Attach to resource

See [Section 5.5](#55-attaching-files-to-resources).

#### Direct Upload Model

A single-step flow where the client sends the file directly to `POST /files` using `multipart/form-data`. The server receives, validates, stores, and returns the completed resource in one request.

`POST /files` with `Content-Type: multipart/form-data`

Response (`201 Created`):

```json
{
  "data": {
    "id": "9f1c2e3a-1b2c-4d5e-8f9a-0b1c2d3e4f5a",
    "status": "completed",
    "filename": "avatar.png",
    "content_type": "image/png",
    "category": "image",
    "size_bytes": 51200,
    "url": "https://cdn.domain.com/files/9f1c2e3a....png",
    "upload_url": null,
    "upload_method": null,
    "expires_at": null,
    "created_at": "2026-07-10T09:00:00Z"
  },
  "error": null,
  "meta": {
    "request_id": "d5a3b4c6-7e8f-9a0b-1c2d-3e4f5a6b7c8d",
    "timestamp": "2026-07-10T09:00:00Z"
  }
}
```

`multipart/form-data` is the **only exception** to the "request body must be flat JSON" rule (Section 4), specifically for `POST /files` in Direct Upload mode. The resource skips the `pending` state entirely — `status` is `completed` immediately upon success.

#### Attaching Files to Resources

Use the existing foreign key pattern — not a new endpoint:

```
PATCH /recipes/1
```

```json
{
  "photo_id": "9f1c2e3a-1b2c-4d5e-8f9a-0b1c2d3e4f5a"
}
```

The server **MUST** reject the attach if the file's `status` is anything other than `completed` (see error `file_not_ready`).

#### Validation & Policy

Policy **SHOULD** be configured per usage context (not a single global policy), via an optional `purpose` parameter in the upload request:

```json
{
  "filename": "avatar.png",
  "content_type": "image/png",
  "size_bytes": 51200,
  "purpose": "author_avatar"
}
```

Example policy table (defined by the implementation team, not a rigid part of this spec):

| `purpose`         | Max `size_bytes` | Allowed `content_type`                  |
| ----------------- | ---------------- | --------------------------------------- |
| `author_avatar`   | 2 MB             | `image/jpeg`, `image/png`, `image/webp` |
| `recipe_photo`    | 10 MB            | `image/jpeg`, `image/png`, `image/webp` |
| `recipe_document` | 25 MB            | `application/pdf`                       |

If `purpose` is not sent, the server **MAY** apply a more permissive default policy, or **MUST** reject with `purpose_required` — this choice is defined by the implementation team.

#### Routing Rules

- Action endpoints **MUST** be placed after `{id}` (pattern: `/files/{id}/complete`). They **MUST NOT** be placed directly after the collection without `{id}` (pattern: `/files/complete` is forbidden) — this prevents routing ambiguity with `GET/DELETE /files/{id}`.
- `{id}` **MUST** be a UUID (not a free-form string or simple integer). This makes static segments like `complete` automatically invalid as `{id}` values, eliminating collision risk.
- These rules apply universally to all action endpoints in ByJSON, not just `/files` (see [Section 3.1](#31-resource-urls)).

#### Orphan Cleanup

- Files with `status = pending` that are not completed within `expires_at` **MUST** be treated as expired and may be automatically deleted by a scheduled job.
- Files with `status = completed` that are never referenced by any resource within a configurable period (e.g., 24–72 hours) **SHOULD** be cleaned up by a separate scheduled job, to prevent storage from filling with orphaned files caused by client crashes between upload completion and resource attachment.

#### File Representation in Other Resources

When another resource (e.g., `recipe`) has a relation to a file, display it as a summary object — consistent with the relational data pattern in Section 5.2:

```json
{
  "id": 1,
  "title": "Nasi Goreng Spesial",
  "photo": {
    "id": "9f1c2e3a-1b2c-4d5e-8f9a-0b1c2d3e4f5a",
    "url": "https://cdn.domain.com/files/9f1c2e3a....jpg",
    "content_type": "image/jpeg"
  }
}
```

`photo_id` is used as the request field (write), while `photo` (object) is used as the response field (read) — following the same `author_id` (write) vs `author` (read) pattern already established in the core spec.

### 6.2 Aggregate & Metric Sub-Resources

**Aggregate Sub-Resources** are URI endpoints that return computed data (counts, sums, averages, summaries) derived from a collection, rather than the collection's items themselves.

#### Format

```
GET /{resource}/summary            — summary metrics across the collection
GET /{resource}/{metric-name}      — a specific metric (e.g., unread-count, balance)
GET /{resource}/{id}/summary       — summary metrics scoped to one item
```

#### Rules

1. Aggregate sub-resource endpoints **MUST** use `GET` (read-only, idempotent).
2. `data` **MUST** be a flat object containing named metric fields — never an array of items.
3. The full ByJSON response envelope (`data`, `error`, `meta`) **MUST** be used.
4. Aggregate endpoints **MUST NOT** be considered "actions" for the purposes of the anti-collision rule in Section 3.1. Static path segments like `summary`, `count`, or `unread-count` that serve as aggregate endpoints are allowed at the collection level even if `GET /{resource}/{id}` also exists — provided `{id}` is a UUID (which is never ambiguous with a word segment).

#### Examples

`GET /reviews/pending-count`

```json
{
  "data": { "pending_count": 12 },
  "error": null,
  "meta": { "request_id": "...", "timestamp": "..." }
}
```

`GET /recipes/summary`

```json
{
  "data": {
    "total_recipes": 248,
    "average_duration_minutes": 32,
    "most_cooked_category": "main-course",
    "recipes_added_this_week": 14
  },
  "error": null,
  "meta": { "request_id": "...", "timestamp": "..." }
}
```

`GET /recipes/stats`

```json
{
  "data": {
    "bookmarked": 37,
    "published": 210,
    "draft": 38
  },
  "error": null,
  "meta": { "request_id": "...", "timestamp": "..." }
}
```

---

### 6.3 Offline-First Sync

Many client applications (mobile, desktop) maintain a local database and need to synchronize state with the server — fetching changes, uploading locally-created records, and resolving conflicts. ByJSON defines the JSON payload contract for each sync pattern. **The server determines which patterns to support per endpoint and MUST document its choices.**

**Guidance:**

| Pattern | Use When |
| --- | --- |
| **Delta Sync (Pull)** | Client needs to fetch records that changed on the server since its last sync |
| **Push Sync (Upload)** | Client needs to upload records created or modified while offline |
| **Conflict Resolution** | Both client and server may modify the same record concurrently |
| **Optimistic Locking** | Client must guarantee it's not overwriting a record that changed since it last read it |

#### Delta Sync (Pull)

The client fetches all records that changed on the server since a watermark timestamp. Architecturally distinct from pagination:

| | Standard List | Delta Sync |
| --- | --- | --- |
| **Purpose** | Display paginated UI | Update a local offline database |
| **Scope** | Current page of items | All changes since a watermark |
| **Includes deletions?** | No | Yes — via tombstone records |
| **Returns `pagination` meta?** | Yes | No — returns `server_time` instead |

**URI format:**
```
GET /{resource}/sync
GET /{resource}/{id}/sync
```

**Query parameters:**

| Parameter | Type | Description |
| --- | --- | --- |
| `since` | ISO 8601 | Watermark from the previous sync. If omitted, server **MAY** return a bounded historical window or **MUST** document its default behavior. |

**Rules:**
1. **MUST** use `GET` (read-only).
2. Response **MUST** include all records updated **and** soft-deleted since `since`.
3. Soft-deleted records **MUST** appear as tombstone objects with at minimum `id` and `"is_deleted": true`.
4. `meta` **MUST** include `server_time` (ISO 8601). The client **MUST** use this as `since` in the next request — not its own local clock.
5. **MUST NOT** use standard pagination. The full delta set is returned in one response.

`GET /cooking-sessions/sync?since=2026-09-01T00:00:00Z`

```json
{
  "data": [
    { "id": "a1b2c3d4-...", "recipe_id": "9f1c2e3a-...", "duration_seconds": 1800, "updated_at": "2026-09-10T08:30:00Z", "is_deleted": false },
    { "id": "e5f6a7b8-...", "is_deleted": true }
  ],
  "error": null,
  "meta": { "request_id": "...", "timestamp": "...", "server_time": "2026-09-12T10:00:00Z" }
}
```

#### Push Sync (Upload)

The client uploads records that were created or modified locally while offline. The server processes each item and returns a per-item result.

**URI format:**
```
POST /{resource}/sync
```

**Rules:**
1. **MUST** use `POST`.
2. Request body **MUST** contain an `items` array of records to upsert.
3. Each item **MUST** include `id` (client-generated UUID), `updated_at`, and all writable fields.
4. Response **SHOULD** use HTTP `207 Multi-Status` and return `succeeded` and `failed` arrays (same pattern as Bulk Operations in Section 5.2-H).
5. `meta` **MUST** include `server_time`.

`POST /cooking-sessions/sync`

Request:
```json
{
  "items": [
    { "id": "a1b2c3d4-...", "recipe_id": "9f1c2e3a-...", "duration_seconds": 1800, "updated_at": "2026-09-11T07:00:00Z" },
    { "id": "b2c3d4e5-...", "recipe_id": "8e0b1d2a-...", "duration_seconds": 900, "updated_at": "2026-09-11T08:00:00Z" }
  ]
}
```

Response (207):
```json
{
  "data": {
    "succeeded": [ { "id": "a1b2c3d4-..." } ],
    "failed": [ { "id": "b2c3d4e5-...", "code": "conflict", "message": "Record was modified on the server." } ]
  },
  "error": null,
  "meta": { "request_id": "...", "timestamp": "...", "server_time": "2026-09-13T09:00:00Z" }
}
```

#### Conflict Resolution

A conflict occurs when both the client and the server have modified the same record since the client last synced. ByJSON does not mandate a single resolution strategy, but **the server MUST document which strategy it uses per endpoint.**

| Strategy | Description | When to Use |
| --- | --- | --- |
| **Server-Wins** | Server's version always prevails. Client changes are discarded. | Simple data where server is the source of truth (e.g., streaks, scores) |
| **Client-Wins** | Client's version always prevails. Server version is overwritten. | Local-first personal data where user intent is paramount |
| **Last-Write-Wins** | The version with the most recent `updated_at` timestamp wins. | General purpose; requires reliable client clocks |
| **Manual Resolution** | Server returns `conflict` error; client must resolve and retry. | Critical data where no automatic resolution is acceptable |

For **Manual Resolution**, the server **MUST** return HTTP `409 Conflict` with error code `conflict` and include the current server version of the record in the error `details`:

```json
{
  "data": null,
  "error": {
    "code": "conflict",
    "message": "This record was modified on the server since your last sync.",
    "details": [
      { "field": "server_version", "value": { "id": "a1b2-...", "title": "Server title", "updated_at": "2026-09-12T10:00:00Z" } }
    ]
  },
  "meta": { "request_id": "...", "timestamp": "..." }
}
```

#### Optimistic Locking

Optimistic Locking prevents a client from overwriting a record that has changed on the server since the client last read it, without requiring explicit conflict resolution negotiation.

The client includes the `updated_at` timestamp of the version it last read in the request. The server rejects the update if the record has since been modified.

**Option A — `If-Unmodified-Since` header (HTTP standard):**
```
PATCH /cooking-sessions/{id}
If-Unmodified-Since: 2026-09-10T08:30:00Z
```
Server returns `412 Precondition Failed` if the record has been modified after that timestamp.

**Option B — `updated_at` field in request body:**
```json
{ "title": "New title", "updated_at": "2026-09-10T08:30:00Z" }
```
Server returns `409 Conflict` with code `conflict` if the stored `updated_at` does not match.

The server **MUST** document which option is supported per endpoint. Option A is preferred for standard HTTP compliance.

---

### 6.4 Multi-Audience API (Namespace Separation)

A ByJSON-compliant API may need to serve multiple audiences simultaneously — end-users, administrators, automated systems, or third-party partners. **The URI contract must not change based on who the caller is.** Authorization policy — not the URL structure — determines what data a caller may access.

#### The Core Principle

URIs identify **data entities**, not the perspective of the caller.

```
✅  GET /recipes   — identifies the recipes resource; authorization policy resolves what data is returned
❌  GET /my-recipes vs GET /admin/all-recipes — different URIs for the same resource based on caller role
```

However, when different audiences require fundamentally different response schemas, security boundaries, or rate limits, namespace separation is a valid and recommended architectural pattern.

#### Strategies

#### Strategy 1: Authorization Scoping (Recommended for Single-Resource Servers)

The same endpoint URI serves all audiences. The server's authorization layer determines what data is visible:

- `GET /recipes` called by an **end-user** → returns only their own recipes (row-level policy)
- `GET /recipes` called by an **admin** → returns all recipes, or allows filter by `?user_id=...`

Server **MUST** document the role-dependent behavior of any endpoint that uses this strategy.

#### Strategy 2: Hierarchical Ownership (Recommended for Explicit Multi-Owner Access)

All resource access is explicit through the ownership hierarchy:

```
GET /users/{user_id}/recipes    — admin or owner accessing a specific user's recipes
GET /users/me/recipes           — shorthand for the authenticated user's own recipes
```

This strategy is the most explicit and scales well to B2B, C2C, and multi-tenant scenarios.

#### Strategy 3: Namespace Separation (Recommended for Distinct Audiences with Different Schemas)

Separate URI namespaces for each audience:

```
GET /recipes                    — Consumer API (end-user, mobile, browser)
GET /admin/recipes              — Backoffice API (admin panel, internal tools)
GET /system/recipes             — Machine-to-machine, service integrations
```

Each namespace **MAY** have its own authentication mechanism, response schema, and rate limits. This is appropriate when the response shape differs significantly between audiences (e.g., admin responses include audit fields not relevant to end-users).

#### The `/me` Convention

The segment `me` is a **contextual alias** for `{user_id}` — it resolves to the authenticated caller's own identifier. Its use is subject to these rules:

1. `me` **MUST** only be used as a substitute for a resource owner's identifier in a hierarchical URI (`/users/me`, `/users/me/recipes`).
2. `me` **MUST NOT** be appended to collection-level singletons as a workaround for implicit scoping (e.g., `GET /cookbooks/me` is only appropriate if `GET /cookbooks/{user_id}` is also a valid endpoint).
3. If `me` is supported, it **MUST** be applied consistently across all resources in the same namespace. Selective use of `me` on some endpoints and implicit JWT scoping on others creates an inconsistent contract.

#### Audience-Based Design Decision Table

| Scenario | Recommended Strategy |
| --- | --- |
| Single-owner app (no admin access to other users' data) | Strategy 1: Authorization Scoping |
| Admin needs to access any user's data | Strategy 2: Hierarchical Ownership or Strategy 3: Namespace Separation |
| B2B / multi-tenant (organizations own resources, users are members) | Strategy 2: Hierarchical Ownership (`/publishers/{id}/recipes`) |
| C2C marketplace (same user is both creator and remixer) | Strategy 2: Hierarchical Ownership (`/users/{id}/recipes/as-creator`, `/users/{id}/recipes/as-remixer`) |
| Different auth mechanisms or response schemas per audience | Strategy 3: Namespace Separation |

---

### 6.5 Idempotency Keys

Network failures can cause a client to retry a `POST` request, resulting in duplicate data (e.g., a duplicate order, a duplicate payment). **Idempotency Keys** allow a client to safely retry without fear of duplication.

#### Usage

Clients **SHOULD** include an `Idempotency-Key` header on `POST` requests that are not naturally idempotent:

```
POST /recipes
Idempotency-Key: a3e9b1c2-f345-4d67-8a90-b1c2d3e4f5a6
Content-Type: application/json

{ "title": "Nasi Goreng Spesial", "difficulty": "EASY", "duration_minutes": 15 }
```

#### Server Behavior

1. Server **MUST** store the response of the first successful request keyed by the `Idempotency-Key` value for a minimum window of **24 hours**.
2. If a second request arrives with the same key within that window, the server **MUST** return the stored response identically — including the same HTTP status code — without re-executing the operation.
3. If a request arrives with the same key but a **different request body**, the server **MUST** return `409 Conflict` with error code `idempotency_key_conflict`.
4. `Idempotency-Key` values **MUST** be UUIDs generated by the client.
5. Idempotency-Key headers are **OPTIONAL** from the spec's perspective. Endpoints that require them **MUST** document this requirement explicitly.

#### When to Use

| Operation | Idempotency Key Recommended? |
| --- | --- |
| Create recipe / publish | ✅ Yes |
| Send message | ✅ Yes |
| Create resource (general POST) | Recommended |
| Read (GET) | No — GET is already idempotent |
| Update (PATCH / PUT) | No — already idempotent by nature if using absolute values |
| Delete | Optional |

---

## 7. Reference Tables

### 7.1 Error Code Reference

| Error Code                    | HTTP Status | Description                                                                    |
| ----------------------------- | ----------- | ------------------------------------------------------------------------------ |
| `bad_request`                 | 400         | The request body is malformed JSON and cannot be parsed.                       |
| `validation_error`            | 422         | The request data failed validation rules. See `details`.                       |
| `unauthorized`                | 401         | Authentication is required or the token has expired.                           |
| `forbidden`                   | 403         | The authenticated user does not have permission.                               |
| `not_found`                   | 404         | The requested resource does not exist.                                         |
| `method_not_allowed`          | 405         | The HTTP method is not supported for this endpoint.                            |
| `conflict`                    | 409         | The request conflicts with the current state of the resource.                  |
| `idempotency_key_conflict`    | 409         | An `Idempotency-Key` was reused with a different request body. See Section 6.5. |
| `file_not_ready`              | 409         | File cannot be attached because its `status` is not `completed`.               |
| `upload_url_expired`          | 410         | The `upload_url` has passed its `expires_at` timestamp.                        |
| `file_too_large`              | 422         | `size_bytes` exceeds the limit for the given `purpose`.                        |
| `unsupported_file_type`       | 422         | `content_type` is not allowed for the given `purpose`.                         |
| `purpose_required`            | 422         | `purpose` is required but was not provided.                                    |
| `file_verification_failed`    | 409         | Server could not find the file in storage during completion.                   |
| `too_many_requests`           | 429         | The client has exceeded the rate limit. Check `meta.retry_after`.              |
| `endpoint_deprecated`         | 200 / 4xx   | The endpoint is deprecated. Check `meta.deprecation` for the replacement.      |
| `internal_error`              | 500         | An unexpected server error occurred.                                           |
| `service_unavailable`         | 503         | The server is temporarily unavailable. Check `meta.retry_after`.               |

### 7.2 Filter Operator Reference

These operators are used inside LHS Brackets for advanced filtering (e.g., `field[operator]=value`).

| Operator | Description           | Example                      |
| -------- | --------------------- | ---------------------------- |
| `eq`     | Equal                 | `difficulty[eq]=EASY`        |
| `neq`    | Not equal             | `difficulty[neq]=HARD`       |
| `gt`     | Greater than          | `duration_minutes[gt]=30`    |
| `gte`    | Greater than or equal | `duration_minutes[gte]=15`   |
| `lt`     | Less than             | `duration_minutes[lt]=60`    |
| `lte`    | Less than or equal    | `duration_minutes[lte]=60`   |
| `in`     | In array of values    | `difficulty[in]=EASY,MEDIUM` |
| `like`   | Pattern match         | `title[like]=%goreng%`       |

_Note 1:_ For simple equality checks, you may omit the bracket notation entirely and use the field name directly (e.g., `difficulty=EASY` is equivalent to `difficulty[eq]=EASY`). By default, any field without an operator bracket uses `eq`.

_Note 2:_ For the `in` operator, if a value itself contains a comma, it **MUST** be URL-encoded (e.g., `Washington%2CD.C.`). Alternatively, the server **MAY** support repeated keys (e.g., `city[in]=Washington, D.C.&city[in]=New York`).

### 7.3 Meta Object Specification

The `meta` object is present in every response.

| Field         | Type      | Required | Description                                                                                                                                                                              |
| ------------- | --------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `request_id`  | `string`  | **Yes**  | A UUID identifier for the request, useful for debugging and tracing.                                                                                                                     |
| `timestamp`   | `string`  | **Yes**  | ISO 8601 timestamp of when the server processed the request.                                                                                                                             |
| `pagination`  | `object`  | No       | Present only in paginated collections. The schema varies based on the strategy (offset vs cursor). See [Section 5.2-C](#c-collection-with-pagination).                                   |
| `server_time` | `string`  | No       | ISO 8601 timestamp of the server's authoritative clock. **MUST** be included in all Offline-First Sync responses (Section 6.3). The client **MUST** use this as `since` in the next request. |
| `retry_after` | `integer` | No       | Seconds the client should wait before retrying. **MUST** be included in `429 Too Many Requests` and `503 Service Unavailable` responses.                                                 |
| `deprecation` | `object`  | No       | Present when the endpoint is deprecated. Contains `sunset_date` (ISO 8601) and `successor` (URI of the replacement endpoint).                                                            |

## License

This project is licensed under the MIT License — see the [LICENSE.md](LICENSE.md) file for details.
