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
  - [2.3 Resource URLs](#23-resource-urls)
  - [2.4 Endpoint Reference](#24-endpoint-reference)
- [3. Requests](#3-requests)
  - [3.1 Create](#31-create)
  - [3.2 Update (Full)](#32-update-full)
  - [3.3 Update (Partial)](#33-update-partial)
  - [3.4 Delete](#34-delete)
  - [3.5 Query Parameters](#35-query-parameters)
- [4. Responses](#4-responses)
  - [4.1 General Payload Structure](#41-general-payload-structure)
  - [4.2 Success Responses](#42-success-responses)
  - [4.3 Error Responses](#43-error-responses)
- [5. Reference Tables](#5-reference-tables)
  - [5.1 Error Code Reference](#51-error-code-reference)
  - [5.2 Filter Operator Reference](#52-filter-operator-reference)
  - [5.3 Meta Object Specification](#53-meta-object-specification)
- [License](#license)

---

## 1. Principles

| Principle                 | Description                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| **Predictable Structure** | Every response has the same three root keys: `data`, `error`, and `meta`. No guessing.       |
| **Simplicity**            | Relational data is nested directly inside the parent object — no separate `included` block.  |
| **Clarity**               | Success and error payloads are mutually exclusive. If one is populated, the other is `null`. |

## 2. Naming Conventions

### 2.1 JSON Keys

All keys in request and response bodies **MUST** use `snake_case`.

_Note: `snake_case` is chosen for maximum consistency with backend database columns and system variables._

```
✅  first_name, email_address, author_id
❌  firstName, EmailAddress, authorID
```

### 2.2 Error Codes

Error codes must also use `snake_case` (e.g. `validation_error`) for maximum consistency.

```
✅  validation_error, not_found, bad_request
❌  VALIDATION_ERROR, NotFound, BadRequest
```

### 2.3 Resource URLs

_Note: URL versioning (e.g., `/v1/`) is highly recommended but falls outside the scope of this JSON specification._

| Rule                                        | Example                                 |
| ------------------------------------------- | --------------------------------------- |
| Use **plural nouns**                        | `/recipes` (not `/recipe`)              |
| Use **kebab-case** for multi-word resources | `/cooking-steps` (not `/cooking_steps`) |
| Use **nesting** only for strict ownership   | `/authors/1/recipes`                    |
| Use **action URLs** for specific operations | `/recipes/1/bookmark`                   |

### 2.4 Endpoint Reference

The examples throughout this spec use the following `Recipe` and `Author` resources:

| Method   | Endpoint              | Description                       |
| -------- | --------------------- | --------------------------------- |
| `GET`    | `/recipes`            | List all recipes                  |
| `GET`    | `/recipes/1`          | Get a single recipe               |
| `POST`   | `/recipes`            | Create a new recipe               |
| `PUT`    | `/recipes/1`          | Full update a recipe              |
| `PATCH`  | `/recipes/1`          | Partial update a recipe           |
| `DELETE` | `/recipes/1`          | Delete a recipe                   |
| `GET`    | `/authors/1/recipes`  | List recipes by a specific author |
| `POST`   | `/recipes/1/bookmark` | Bookmark a specific recipe        |

## 3. Requests

When sending data to the server (`POST`, `PUT`, `PATCH`), the request body must be a flat JSON object. Relationships are referenced by their foreign key (e.g., `author_id`) rather than nested objects.

### 3.1 Create

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

### 3.2 Update (Full)

`PUT /recipes/1`

A full update replaces the entire resource. All writable fields must be provided.

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

### 3.3 Update (Partial)

`PATCH /recipes/1`

A partial update modifies only the fields provided.

- **Omitted fields** remain unchanged.
- **`null` values** explicitly unset or remove the field/relationship (if the field is nullable).

```json
{
  "title": "Nasi Goreng Spesial (Extra Pedas)"
}
```

### 3.4 Delete

`DELETE /recipes/1`

Delete operations do not require a JSON request body.

### 3.5 Query Parameters

When retrieving collections, use query parameters for pagination, sorting, filtering, and field selection.

#### Pagination

Both **offset-based** and **cursor-based** pagination are supported. **The server determines the strategy per endpoint**; the client does not negotiate this.

**Guidance:**

- **Use offset-based** when the client needs to navigate to specific pages (e.g., "jump to page 5") or needs to display total counts/pages.
- **Use cursor-based** when the client uses infinite scroll / "load more", or for massive/fast-changing datasets where query performance is prioritized over absolute navigation.

**Constraint:**
Cursor pagination **MUST** be used alongside a deterministic sort order that has a unique tiebreaker (usually `id`). Sorting solely by non-unique fields (e.g., `created_at`) will cause items to be skipped or duplicated between pages. Cursors should also be opaque (e.g., base64 encoded strings) to prevent clients from depending on internal data structures.

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

See [Section 5.2](#52-filter-operator-reference) for the full list of supported operators.

#### Field Selection

A client may request that only specific fields be returned in the response by using the `fields` parameter with a comma-separated list of field names. For nested relationships, use dot notation (e.g., `author.name`).

```
GET /recipes?fields=id,title,difficulty,author.name
```

If `fields` is specified, the server **MUST NOT** include additional fields beyond those requested. If `fields` is omitted, the server may return all fields.

#### Full Example

```
GET /recipes?page=1&per_page=10&sort=-duration_minutes&difficulty=EASY&duration_minutes[gte]=15&fields=id,title,duration_minutes,author.name
```

## 4. Responses

### 4.1 General Payload Structure

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

### 4.2 Success Responses

For successful operations (HTTP `2xx`), `error` must be `null`.

Relational data (like `author`) is always embedded as a **summary object** directly inside the parent entity. This avoids the need for a separate `included` block while keeping the payload concise.

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
        "prev": "eyJpZCI6MX0="
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

Action endpoints that do not correspond to a single resource's state (e.g., `POST /recipes/1/send-to-email` or `/auth/logout`) may return a minimal payload (like `{"success": true}`) or `null` in the `data` field, relying purely on the HTTP status code to convey success.

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

Bulk operations (Bulk Create, Update, or Delete) can result in partial failures. The request itself is successful, but individual items may fail. To represent this while maintaining the mutual exclusivity of `data` and `error`, bulk endpoints should use the HTTP `207 Multi-Status` code and return `succeeded` and `failed` arrays inside the `data` object.

The `error` object remains `null` because there was no request-level failure (e.g., malformed JSON or unauthorized access).

`POST /recipes/bulk-delete` (or `DELETE /recipes?ids=1,2,3`)

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

### 4.3 Error Responses

For failed operations (HTTP `4xx` / `5xx`), `data` must be `null`.

#### A. Validation Errors (HTTP 422)

Used when the request body fails validation rules. The `details` array points to the exact location of each error using the `field` key.

**Field Path Notation:** Use dot notation for nested objects (e.g., `shipping_address.city_name`) and array index notation for array items (e.g., `ingredients[0].name`).

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
        "field": "ingredients[0].name",
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

## 5. Reference Tables

### 5.1 Error Code Reference

| Error Code            | HTTP Status | Description                                                   |
| --------------------- | ----------- | ------------------------------------------------------------- |
| `bad_request`         | 400         | The request body is malformed JSON and cannot be parsed.      |
| `validation_error`    | 422         | The request data failed validation rules. See `details`.      |
| `unauthorized`        | 401         | Authentication is required or the token has expired.          |
| `forbidden`           | 403         | The authenticated user does not have permission.              |
| `not_found`           | 404         | The requested resource does not exist.                        |
| `method_not_allowed`  | 405         | The HTTP method is not supported for this endpoint.           |
| `conflict`            | 409         | The request conflicts with the current state of the resource. |
| `too_many_requests`   | 429         | The client has exceeded the rate limit.                       |
| `internal_error`      | 500         | An unexpected server error occurred.                          |
| `service_unavailable` | 503         | The server is temporarily unavailable.                        |

### 5.2 Filter Operator Reference

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

_Note: For simple equality checks, you may omit the bracket notation entirely and use the field name directly (e.g., `difficulty=EASY` is equivalent to `difficulty[eq]=EASY`). By default, any field without an operator bracket uses `eq`._

### 5.3 Meta Object Specification

The `meta` object is present in every response.

| Field        | Type     | Required | Description                                                                                                                                            |
| ------------ | -------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `request_id` | `string` | **Yes**  | A UUID identifier for the request, useful for debugging and tracing.                                                                                   |
| `timestamp`  | `string` | **Yes**  | ISO 8601 timestamp of when the server processed the request.                                                                                           |
| `pagination` | `object` | No       | Present only in paginated collections. The schema varies based on the strategy (offset vs cursor). See [Section 4.2-C](#c-collection-with-pagination). |

## License

This project is licensed under the MIT License — see the [LICENSE.md](LICENSE.md) file for details.
