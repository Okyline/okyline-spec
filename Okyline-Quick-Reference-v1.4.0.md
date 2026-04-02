---
description: Okyline quick reference - compact cheat sheet with syntax, types, operators, constraints, and common examples for JSON schema validation by example.
---

# Okyline Core Language Quick Reference

**Version:** 1.4.0
**Date:** April 2026
**Status:** Draft

Okyline is a JSON-based schema language for validating JSON documents **by example**. You write a JSON document with real example values, then add constraints directly in field names.

**Okyline** is a **declarative language** designed to describe the **structure** and **constraints** of **JSON documents** in a **lightweight** and **readable** manner. It enriches **JSON examples** with **inline constraints**, enabling **data validation** while keeping schemas **human-friendly**.

**Okyline** was designed from the outset to support **structural validation**, **conditional logic**, and **computed business invariants** within JSON document definitions.


---

## 1. Document Structure

A minimal Okyline schema:
```json
{
  "$oky": {
    "name|@": "Alice",
    "age|(18..120)": 30
  }
}
```
↳ Two fields: `name` required string, `age` integer between 18 and 120

---

## 2. Field Syntax

```
" fieldName | constraints | label ": exampleValue
```

| Part | Required | Description |
|------|----------|-------------|
| `fieldName` | Yes | JSON attribute name |
| `constraints` | No | Space-separated constraint symbols |
| `label` | No | Human-readable description (no `\|` allowed) |

**Example:**
```json
"email|@ {5,100} ~$Email~|User email": "alice@example.com"
```
↳ Required string, 5-100 chars, must match email format, labeled "User email"

---

## 3. Type Inference

The example value determines the expected type:

| Example value | Inferred type |
|---------------|---------------|
| `"text"` | String |
| `42` | Integer |
| `3.14` | Number |
| `"78.00"` | Number (auto-convert) |
| `true` / `false` | Boolean |
| `[1, 2]` | Array[Integer] |
| `{"k": "v"}` | Object |

**Note:** Since JSON serialization doesn't guarantee trailing zeros are preserved, you can express decimal numbers as strings (e.g., `"78.00"`). These are auto-converted to Number. Use `$str` to force string type if needed.

**Modifiers:**
- `$str` — force string type (disable decimal auto-conversion)
- `$obj` — treat array example as multiple examples of a single-value field

---

## 4. Scalar Constraints

| Symbol | Applies to | Meaning |
|--------|------------|---------|
| `@` | All | Required |
| `?` | All | Nullable |
| `#` | Scalars | Key field (for uniqueness in lists) |
| `%` | All | Default value (informational only) |
| `{max}` | String | Max length |
| `{min,max}` | String | Length range |
| `(min..max)` | Number | Inclusive range |
| `(>n)` `(<n)` `(>=n)` `(<=n)` | Number | Comparison |
| `('a','b','c')` | All | Enum values |
| `~pattern~` | String | Inline regex |
| `~$Name~` | String | Named format |

**Example combining constraints:**
```json
"price|@ ? (0.01..9999.99)": 19.99
```
↳ Required, nullable, decimal between 0.01 and 9999.99

---

## 5. Collection Constraints

| Syntax | Applies to | Meaning |
|--------|------------|---------|
| `[max]` | List | Max size |
| `[min,max]` | List | Size range |
| `[min,*]` | List | Min only (no max) |
| `[*]` | List | Any size |
| `-> constraints` | List/Map | Apply constraints to each element |
| `!` | List | Uniqueness (by `#` keys for objects) |
| `[*:max]` | Map | Any key, max entries |
| `[~pattern~:max]` | Map | Key pattern, max entries |

**Examples:**
```json
"tags|[1,5] -> {2,20}!": ["eco", "bio"]
```
↳ List of 1-5 unique strings, each 2-20 characters

```json
"scores|[15] -> (0..100)": [85, 92]
```
↳ List of maximum 15 elements, each element between 0 and 100

```json
"translations|[~^[a-z]{2}$~:10] -> {1,100}": { "en": "Hello", "fr": "Bonjour" }
```
↳ Map with 2-letter keys, max 10 entries, values 1-100 characters

```json
"users|[10] -> !": [{ "id|#@": 1, "name|@": "Alice" }]
```
↳ List of max 10 unique objects — uniqueness determined by `#` key field (`id`)

---

## 6. Conditional Directives

| Directive | Value | Effect |
|-----------|-------|--------|
| `$requiredIf field(cond)` | `[fields]` | Listed fields required when condition on field is true |
| `$requiredIfNot field(cond)` | `[fields]` | Listed fields required when condition on field is false |
| `$requiredIfExist field` | `[fields]` | Listed fields required when field exists |
| `$requiredIfNotExist field` | `[fields]` | Listed fields required when field is absent |
| `$forbiddenIf field(cond)` | `[fields]` | Listed fields forbidden when condition on field is true |
| `$forbiddenIfNot field(cond)` | `[fields]` | Listed fields forbidden when condition on field is false |
| `$forbiddenIfExist field` | `[fields]` | Listed fields forbidden when field exists |
| `$forbiddenIfNotExist field` | `[fields]` | Listed fields forbidden when field is absent |
| `$appliedIf field(cond)` | `{...}` | Structure applied when condition on field is true |
| `$appliedIfExist field` | `{...}` | Structure applied when field exists |
| `$appliedIfNotExist field` | `{...}` | Structure applied when field is absent |
| `$required` | `[fields]` | Unconditional required fields |
| `$forbidden` | `[fields]` | Unconditional forbidden fields |
| `$atLeastOne` | `[fields]` | At least one field from group must be present |
| `$mutuallyExclusive` | `[fields]` | At most one field from group may be present |
| `$exactlyOne` | `[fields]` | Exactly one field from group must be present |
| `$allOrNone` | `[fields]` | All fields in group present, or none |

**Condition syntax:** `field(values)` or `field(min..max)` or `field(_TypeGuard_)` or `field(null)`

> **Suffix mechanism:** To declare multiple independent groups of the same directive in one object, append a unique suffix: `$atLeastOne_contact`, `$atLeastOne_address`, etc. The suffix is semantically ignored.

**Examples:**
```json
"$requiredIf age(<18)": ["parentConsent"]
```
↳ `parentConsent` required when `age` is less than 18

```json
"$forbiddenIf status('CLOSED')": ["lastLogin"]
```
↳ `lastLogin` forbidden when `status` equals "CLOSED"

```json
"$requiredIfExist shipping": ["shippingAddress"]
```
↳ `shippingAddress` required when `shipping` field exists

```json
"$appliedIf status('ACTIVE')": {
  "workDays|@ (1..22)": 20
}
```
↳ `workDays` field added to schema when `status` equals "ACTIVE"

**Switch form:**
```json
"$appliedIf paymentMethod": {
  "('CARD')": { "cardNumber|@ {16}": "1234..." },
  "('PAYPAL')": { "email|@ ~$Email~": "a@b.c" },
  "$else": { "reference|@": "REF-001" }
}
```
↳ Different fields required depending on `paymentMethod` value

**Unconditional directives:**
```json
"$required": ["firstName", "lastName"],
"$forbidden": ["internalCode"],
"$exactlyOne": ["email", "phone"],
"$allOrNone": ["street", "city", "zip"]
```

**Null literal in conditions:**
```json
"$requiredIf status('CANCELLED', null)": ["reason"]
```
↳ `reason` required when `status` is `"CANCELLED"` or `null`

---

## 7. Path Expressions

Conditions can reference fields in parent or root objects:

| Prefix | Resolves from |
|--------|---------------|
| *(none)* | Current object |
| `this.` | Current object (explicit) |
| `parent.` | Parent object |
| `root.` | Document root |

**Note:** `parent` skips over arrays — it always refers to the nearest parent **object**, not the array containing the current item.

**Example:**
```json
"$requiredIf parent.status('ACTIVE')": ["code"]
```
↳ In a nested object: `code` required when the parent's `status` equals "ACTIVE"

---

## 8. Registries

**Nomenclature** — reusable value lists:
```json
"$nomenclature": { "STATUS": "ACTIVE,INACTIVE,PENDING" }
```
Usage: `($STATUS)`

**Format** — reusable regex patterns:
```json
"$format": { "Phone": "^\\+[0-9]{10,15}$" }
```
Usage: `~$Phone~`

---

## 9. Built-in Formats

| Format | Description |
|--------|-------------|
| `$Date` | ISO 8601 date (semantic validation) |
| `$DateTime` | ISO 8601 datetime (semantic validation) |
| `$Time` | RFC 3339 time |
| `$Email` | Email address |
| `$Uri` | URI with scheme |
| `$Uuid` | UUID v1-v5 |
| `$Ipv4` | IPv4 address |
| `$Ipv6` | IPv6 address |
| `$Hostname` | DNS hostname |

---

## 10. Type Guards

Test the runtime type of a field in conditions. Useful when a field can have different types and you need different validation rules for each.

| Guard | Matches |
|-------|---------|
| `_String_` | string |
| `_Integer_` | integer (no decimals) |
| `_Number_` | number (int or decimal) |
| `_Boolean_` | boolean |
| `_Null_` | null |
| `_Object_` | object |
| `_EmptyList_`, `_ListOfString_`, `_ListOfInteger_`, `_ListOfNumber_`, `_ListOfBoolean_`, `_ListOfObject_`, `_ListOfNull_` | array by element type |

**Example:**
```json
"$appliedIf data": {
  "(_String_)":{"data|{5,50}": "Hello"},
  "(_Integer_)":{"data|(1..200)": 100}
}
```
↳ `data` validated as string (5-50 chars) or integer (1-200) depending on its actual type

---

## 11. Polymorphism

For fields that accept different structures. For conditional variants, prefer `$appliedIf` (§6). Use `$oneOf`/`$anyOf` when variants are unrelated to other field values.

| Modifier | Meaning |
|----------|---------|
| `$oneOf` | Value must match exactly one variant |
| `$anyOf` | Value must match at least one variant (default) |

**Example:**
```json
"payment|@ $oneOf": [
  { "type|@ ('card')": "card", "number|@": "1234..." },
  { "type|@ ('paypal')": "paypal", "email|@": "a@b.c" }
]
```
↳ `payment` expects ONE object matching either card or paypal variant (not an array)

---

## 12. Comments

Prefix attribute name with `//` to ignore it and its entire subtree:

```json
"//user|@": { "id": 1, "name": "ignored" }
```
↳ The entire `user` object and its children are ignored by the parser

---

## 13. Full Document Structure

Complete schema with all optional metadata:

```json
{
  "$okylineVersion": "1.4.0",
  "$version": "1.0.0",
  "$title": "My Schema",
  "$description": "Schema description",
  "$id": "my.schema",
  "$additionalProperties": false,
  "$nomenclature": { ... },
  "$format": { ... },
  "$oky": { ... }
}
```

| Key | Required | Purpose |
|-----|----------|---------|
| `$oky` | **Yes** | Schema definition |
| `$okylineVersion` | No | Spec version (`"1.4.0"`) |
| `$version` | No | Schema version |
| `$title` | No | Schema title |
| `$description` | No | Schema description |
| `$id` | No | Schema identifier |
| `$additionalProperties` | No | Allow unknown fields (default: `false`) |
| `$nullAsAbsentIfUndeclared` | No | Treat null on non-`?` fields as absent (default: `false`) |
| `$nomenclature` | No | Reusable value lists |
| `$format` | No | Reusable regex patterns |

See **Okyline Language specification annexes** for: `$compute` (C), `$defs` (D), `$field` (F)

---

*Okyline® is a registered trademark of Akwatype.*
