---
name: serialization
description: Use this skill when handling data formats, converting between internal objects and JSON/XML/binary, designing API response shapes, handling dates and timezones in APIs, dealing with missing or extra fields, or when the user asks about serialization, deserialization, DTOs, response models, data transformation, Protocol Buffers, or data format conversion.
---

# Serialization & Deserialization — Backend Engineering

## Core Mental Model

**Serialization** = converting an in-memory object into a format that can be transmitted or stored (object → JSON/binary).
**Deserialization** = converting transmitted/stored data back into an in-memory object (JSON/binary → object).

Never expose your internal data structures directly. Always use a dedicated response model (DTO / serializer / schema). What lives in your database and what you send over the wire are different concerns.

## Standard Format: JSON

JSON is the default for REST APIs. Use it unless you have a specific reason not to.

### JSON Data Types

```json
{
  "string_field": "hello",
  "number_field": 42,
  "float_field": 3.14,
  "boolean_field": true,
  "null_field": null,
  "array_field": [1, 2, 3],
  "object_field": { "nested": "value" }
}
```

**Type rules:**
- Numbers: no quotes. `"age": 25` not `"age": "25"`
- Booleans: `true`/`false` no quotes. Not `"true"`.
- Null: `null` for absent values. Never empty string `""` to represent nothing.
- Dates: always strings in ISO 8601 format.

## Date and Time — The Most Common Source of Bugs

Always transmit dates as ISO 8601 strings. Never as Unix timestamps (they're ambiguous). Never as locale-specific strings.

```
# Correct formats:
"2025-06-15"                          # date only
"2025-06-15T10:30:00Z"               # UTC datetime (Z = UTC)
"2025-06-15T10:30:00+05:30"          # with timezone offset
"2025-06-15T10:30:00.000Z"           # with milliseconds

# Wrong:
"June 15, 2025"                       # locale-specific
1749987600                            # Unix timestamp (ambiguous)
"15/06/2025"                          # ambiguous format
```

**Rules:**
- Store all datetimes in UTC in the database
- Accept datetime input with any timezone offset — convert to UTC before storing
- Always include timezone info in output — never ambiguous local time
- Date-only fields (birthdate, expiry date) use `YYYY-MM-DD` with no time component

## Designing Response Models (DTOs)

Never serialize your database model directly. Create explicit response shapes.

```
# Database model (internal):
{
  id: 1,
  email: "user@example.com",
  password_hash: "$2b$12$...",        # NEVER serialize this
  internal_score: 87,                 # NEVER serialize internal fields
  created_at: 2025-01-15T10:00:00Z,
  stripe_customer_id: "cus_xyz"       # NEVER serialize third-party IDs
}

# Response model (what the API returns):
{
  "id": "usr_abc123",
  "email": "user@example.com",
  "created_at": "2025-01-15T10:00:00Z"
}
```

Fields to always exclude from responses:
- Passwords (any form)
- Internal scoring fields
- Third-party service IDs (Stripe, Sabre, etc.)
- Internal flags used for system logic
- Raw database IDs if you're using UUIDs as public IDs

## Naming Conventions

Pick one and be consistent across the entire API. Never mix.

```json
// snake_case (preferred for REST APIs)
{ "user_id": "123", "created_at": "2025-01-01T00:00:00Z" }

// camelCase (common in JavaScript ecosystems)
{ "userId": "123", "createdAt": "2025-01-01T00:00:00Z" }
```

## Handling Missing Fields

**On input (deserialization):**
- Missing optional field → use the defined default value
- Missing required field → validation error, reject request
- Never silently invent values for missing required fields

**On output (serialization):**
- `null` → include the field with `null` value (the field exists but has no value)
- Not applicable → omit the field entirely
- Be consistent — if a field is sometimes present, document when it appears

```json
// Consistent: include null rather than omitting optional fields
{
  "id": "123",
  "name": "John",
  "middle_name": null,        // field exists, no value
  "phone": null               // field exists, no value
}
```

## Handling Extra/Unknown Fields

**On input:** Ignore unknown fields OR reject them (choose one, document it, apply consistently).
- Ignoring unknown fields is more forward-compatible for clients
- Rejecting unknown fields catches client bugs earlier
- Never crash on unknown fields

**On output:** You control what you return. Unknown fields from third-party sources should be mapped, not passed through.

## Pagination Response Envelope

Always use a consistent envelope for list responses:

```json
{
  "data": [...],
  "meta": {
    "total": 247,
    "page": 2,
    "limit": 20,
    "total_pages": 13,
    "has_next": true,
    "has_prev": true
  }
}
```

## Error Response Shape

Consistent across every endpoint:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "Must be a valid email address" },
      { "field": "age", "message": "Must be between 18 and 120" }
    ],
    "request_id": "req_abc123"
  }
}
```

Never return all errors in a single string. Return all failures at once (not just the first one).

## Binary Formats — When JSON Isn't Enough

Use binary formats only with clear justification:

| Format | Use Case | Tradeoff |
|--------|----------|----------|
| Protocol Buffers | High-throughput microservices | Smaller, faster, but not human-readable |
| MessagePack | Binary JSON-equivalent | Smaller than JSON, harder to debug |
| Avro | Big data pipelines | Schema evolution support |

For public REST APIs: **always JSON**. Binary formats for internal service-to-service communication where performance is measured and proven to matter.

## Large Numbers

JSON numbers lose precision for integers larger than 2^53. Use strings for large IDs:

```json
// Wrong: JavaScript will lose precision on this number
{ "id": 9007199254740993 }

// Correct: send as string
{ "id": "9007199254740993" }
```

## File Uploads

- Small files (< 1MB): base64 encode in JSON body
- Larger files: multipart/form-data
- Large files (> 10MB): generate pre-signed upload URL (S3), client uploads directly

```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
```

## Serialization Security

- Validate input shape before deserializing — reject malformed JSON immediately
- Set maximum body size limit (e.g., 10MB) to prevent memory exhaustion
- Sanitize string fields before using them in any downstream system
- Never deserialize untrusted data into executable formats (no eval, no pickle in Python for untrusted data)
- Log serialization errors — they often indicate an attack or a client bug
