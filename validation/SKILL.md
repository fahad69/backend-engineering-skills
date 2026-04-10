---
name: validation
description: Use this skill when implementing input validation, data transformation, request sanitization, or when the user asks about validating API inputs, handling bad data, transforming request fields, building validation pipelines, error messages for invalid input, or sanitization to prevent injection.
---

# Validation & Transformation — Backend Engineering

## Core Mental Model

**Validation** = checking that input meets your expectations before you process it.
**Transformation** = converting valid input into the form your system needs.

Order: **validate first, transform second, use third**. Never use data before validating it.

The goal of validation is to **fail fast** — reject bad data at the entry point before it contaminates business logic or the database.

## Where Validation Lives

```
Request
  → Route (path param format check)
  → Middleware (auth)
  → Controller/Handler ← VALIDATION HAPPENS HERE
  → Service (business logic)
  → Repository (data access)
```

Validation belongs at the entry point — the controller/handler layer. Business rules belong in the service layer. Do not mix them.

**Controller-level validation:** format, type, required fields, length, patterns
**Service-level validation:** business rules (does this user have enough balance? is this date available?)

## Types of Validation

### 1. Presence / Required Fields
```
email: required
phone: optional
name: required
```
Return all missing required fields at once — not one at a time.

### 2. Type Validation
```
age: must be integer
price: must be number (float or integer)
active: must be boolean
tags: must be array
address: must be object
```

### 3. Format Validation
```
email: RFC 5322 email format
phone: E.164 format (+14155552671)
url: valid URL with http/https scheme
uuid: valid UUID v4 format
date: YYYY-MM-DD
datetime: ISO 8601
postal_code: format by country
credit_card: Luhn algorithm check
ip_address: valid IPv4 or IPv6
```

### 4. Range / Length Validation
```
age: min 0, max 120
password: min 8 chars, max 128 chars
title: min 1 char, max 255 chars
price: min 0.01, max 999999.99
page: min 1
limit: min 1, max 100
rating: min 1, max 5
latitude: min -90, max 90
longitude: min -180, max 180
```

### 5. Semantic Validation
```
date_of_birth: must be in the past, user must be at least 18
check_out_date: must be after check_in_date
passport_expiry: must be at least 6 months after travel date
return_date: must be after departure_date (if round trip)
start_date: must be before end_date
discount_end: must be after discount_start
```

### 6. Enum Validation
```
status: must be one of [active, inactive, suspended]
currency: must be valid ISO 4217 code
country: must be valid ISO 3166-1 alpha-2 code
language: must be valid ISO 639-1 code
cabin_class: must be one of [economy, premium_economy, business, first]
```

### 7. Relational / Conditional Validation
```
if trip_type == "round_trip": return_date is required
if payment_method == "card": card_token is required
if has_children == true: children_count must be > 0
if address.country == "US": state is required
```

### 8. Uniqueness Validation (Service Layer)
```
email must not already exist in database
username must not already be taken
order_reference must be unique
```
This requires a database check — belongs in service layer, not input validation layer.

## Validation Error Response

Return ALL validation errors at once. Never make users fix one error at a time.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address",
        "received": "not-an-email"
      },
      {
        "field": "age",
        "message": "Must be between 18 and 120",
        "received": 15
      },
      {
        "field": "phone",
        "message": "Must be in E.164 format (e.g., +14155552671)",
        "received": "555-1234"
      }
    ]
  }
}
```

Rules for error messages:
- Human-readable, actionable — tell users what to fix
- Reference the field name exactly as it appears in the request
- Include what was received vs what was expected (safe to include — no sensitive info)
- Never leak internal details (stack traces, database errors)

## Transformations — Applied After Validation

### Normalization
```
email.trim().toLowerCase()              # "  USER@EXAMPLE.COM  " → "user@example.com"
name.trim()                             # remove leading/trailing whitespace
phone: normalize to E.164 format        # "555-123-4567" → "+15551234567"
tags: deduplicate array                 # ["a", "b", "a"] → ["a", "b"]
currency: toUpperCase()                 # "usd" → "USD"
country_code: toUpperCase()             # "us" → "US"
```

### Type Coercion (Query Parameters are Always Strings)
```
# Query params come as strings — coerce to correct type:
"?page=2"     → page: 2 (integer)
"?active=true" → active: true (boolean)
"?limit=20"   → limit: 20 (integer)
"?price=9.99" → price: 9.99 (float)
```

### Date Normalization
```
# Input: any valid timezone
"2025-06-15T10:30:00+05:30"

# Store: UTC
"2025-06-15T05:00:00Z"
```

### Sanitization (Security)

Sanitize string inputs before using them in:
- HTML output → escape `<`, `>`, `&`, `"`, `'`
- SQL → use parameterized queries (sanitization is the backup, parameterization is the primary)
- File paths → reject `..`, `/`, `\`
- Shell commands → never put user input in shell commands (use APIs, not shell)
- URLs → validate and encode properly

Sanitization does not replace validation. Do both.

## Query Parameter Validation

Query params are always strings. Validate and transform them:

```
GET /products?min_price=abc&page=-1&limit=999

min_price: "abc" → validation error: must be a positive number
page: -1 → validation error: must be positive integer
limit: 999 → enforce maximum: cap at 100
```

## Path Parameter Validation

```
GET /users/not-a-uuid

user_id: "not-a-uuid" → 400 Bad Request: user_id must be a valid UUID
# NOT 404 — the path param is malformed, not missing
```

## Validation Libraries vs Custom

Use established validation libraries. Do not write validation from scratch.

- They handle edge cases you haven't thought of
- They provide consistent error formatting
- They're battle-tested

After library validation, add custom business-rule validations on top.

## Fail Fast Principle

Reject invalid input before any work happens:
- Do not call the database with invalid data
- Do not call external APIs with invalid data
- Do not start a transaction you'll have to roll back
- Return 400/422 immediately with all validation errors

## Server-Side Validation is Always Required

Client-side validation is UX, not security. Users can bypass it. An API call goes directly to your server — no browser, no client validation.

**Always validate on the server regardless of what the client does.**
