---
description: Complete specification of the Okyline language - an open JSON-based schema language for validating JSON documents by example. Covers syntax, types, constraints, and semantics.
---

# Okyline Language Specification

**Version:** 1.4.0
**Date:** April 2026
**Status:** Draft

**Okyline** is a **declarative language** designed to describe the **structure** and **constraints** of **JSON documents** in a **lightweight** and **readable** manner. It enriches **JSON examples** with **inline constraints**, enabling **data validation** while keeping schemas **human-friendly**.

**Okyline** was designed from the outset to support **structural validation**, **conditional logic**, and **computed business invariants** within JSON document definitions.

---

## Preamble & License

**Okyline® is a registered trademark of Akwatype**. The **Open Okyline Language Specification** is licensed under the
**Creative Commons Attribution–ShareAlike 4.0 International License (CC BY-SA 4.0).**


**Modified versions of this specification must be clearly identified as such and must not be presented as official or endorsed by Akwatype.**
Use of the Okyline® name must not create confusion regarding the origin, affiliation, or approval of the document.

You are free to:

- **Share** - copy and redistribute the material in any medium or format.
- **Adapt** - remix, transform, and build upon the material for any purpose, even commercially.

Under the following terms:

- **Attribution** - You must give appropriate credit, provide a link to the license, and indicate whether changes were made.
- **ShareAlike** - Any derivative works must be distributed under the same license.

Full license text:
**https://creativecommons.org/licenses/by-sa/4.0/**

---

## Reference Implementation

A free version of **Okyline Studio Free**, implementing this specification
(including live documentation, JSON Schema generation, and real-time validation),
is available online:

**https://community.studio.okyline.io**

**Okyline Studio Free** fully supports:
- Okyline Core language
- Annex C - Expression Language
- Annex D - Internal Schema References
- Annex F - Virtual Fields

---

## Specification Status

This document is published as a Draft of the Okyline 1.4.0 specification.
The term "Draft" refers solely to the ongoing formalization of the specification text.

Except for Annex E, Okyline is considered stable for practical use and is implemented by the reference implementation.
Annex E - External Imports and Versioning - is at an advanced stage and will be published in the coming months, following final compatibility and consistency validation.

During the Draft phase, the reference implementation serves as the authoritative behavioral reference in case of ambiguity.


 ## Progressive Adoption Levels (Non-Normative)

Okyline is designed as a **progressively adoptable language**.
Not all features are required for all use cases, and implementations or users may intentionally restrict themselves to a subset of the language.

The following adoption levels are **informative only**.
They do not introduce any additional requirements and do not alter the normative rules defined in the core specification or annexes.

Each level represents a coherent and sufficient usage scope.
Moving to a higher level is a **deliberate design choice**, not an obligation.

### Adoption Levels Overview

| Level | Scope | Specification Coverage | Typical Usage                            |
|------:|-------|------------------------|------------------------------------------|
|     1 | Structural and field validation | Core | CRUD payloads, simple APIs               |
|     2 | Conditional logic | Core | Structural business variants             |
|     3 | Computed business invariants | Annex C | Business coherence rules                 |
|     4 | Internal schema composition and reuse | Annex D | Complex contract structuring             |
|     5 | Virtual fields for conditional logic | Annex F | Derived values in conditions             |
|     6 | Platform governance and versioning | Annex E | Shared schema ecosystems (*in progress*) |


This progressive approach enables a smooth and incremental adoption path, starting from
a simple JSON example that already validates the structure and inferred types of a payload,
and extending gradually to enterprise-scale usage involving schema composition,
versioning, and governance.

---

## Conformance

### Conformance by Annex (Normative)

Conformance to the Okyline specification is defined **per annex**.
An implementation MUST explicitly declare which normative annexes it supports.

If a schema uses any feature defined in a normative annex that the implementation
does not support, the implementation MUST reject the schema as **unsupported**.

This mechanism allows implementations to support coherent subsets of the language
while preserving strict and predictable validation semantics.

### Annex Dependency Rules (Normative)

Conformance to **Annex E - External Imports and Versioning** REQUIRES conformance
to **Annex D - Internal Schema References**.

An implementation MUST NOT claim conformance to Annex E unless it fully implements
Annex D.

Conformance to **Annex F - Virtual Fields** REQUIRES conformance
to **Annex C - Expression Language**.

An implementation MUST NOT claim conformance to Annex F unless it fully implements
Annex C.

---

## 1. Introduction

### 1.1 What is Okyline?

Okyline is a declarative language designed to describe the structure and constraints of JSON documents in a lightweight, readable way. It enriches simple JSON examples with inline constraints, making validation easier while keeping schemas human-friendly.

**Purpose:** Validate JSON data by example.

**Design Philosophy:** Start with an example JSON document, then add constraints directly to field names.

### 1.2 Why Okyline?

Modern data contracts and design-first approaches often struggle under the weight of overly complex schema languages.
Okyline takes a lighter path.

The best way to describe data is to show examples.
The best way to validate data is to define rules.

Okyline lets you do both, in one place, with one syntax.

- Easy to write and read
- Progressive: start with examples, add constraints step by step
- Transpilable to JSON Schema when needed

### 1.3 Key Principles

1. **Example-driven:** Schemas are valid JSON documents with real example values
2. **Type inference:** Field types are automatically inferred from example values
3. **Inline constraints:** Validation rules are expressed as suffixes on field names
4. **Human-readable:** Schemas are self-documenting and easy to understand

### 1.4 Minimal Example

```json
{
  "$oky": {
    "name|@ {2,100}|User name": "Julie",
    "status|@ ('ACTIVE','INACTIVE')|User status":"ACTIVE",
    "$appliedIf status('ACTIVE')": {
      "nbrDaysOfActivities|@ (1..22)|Number of days of activities": 22
    }
  }
}
```
**Breakdown:**
- `name|@ {2,100}` → required string, length between 2 and 100
- `status|@ ('ACTIVE','INACTIVE')` → required enumeration with two allowed values
- `$appliedIf status('ACTIVE')` → applies the nested block only when status is "ACTIVE"
- `nbrDaysOfActivities|@ (1..22)` → required integer between 1 and 22

**Equivalent schema in JSON Schema**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema",
  "x-oky-generated-from": "okyline",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "title": "User name",
      "examples": ["Julie"],
      "minLength": 2,
      "maxLength": 100
    },
    "status": {
      "type": "string",
      "title": "User status",
      "examples": ["ACTIVE"],
      "enum": [
        "ACTIVE", "INACTIVE"
      ]
    }
  },
  "required": [
    "name", "status"
  ],
  "allOf": [{
    "if": {
      "properties": {
        "status": {
          "enum": ["ACTIVE"]
        }
      },
      "required": ["status"]
    },
    "then": {
      "properties": {
        "nbrDaysOfActivities": {
          "type": "integer",
          "title": "Number of days of activities",
          "examples": [22],
          "minimum": 1,
          "maximum": 22
        }
      },
      "required": ["nbrDaysOfActivities"]
    }
  }]
}
```

---


## 2. Quick Start

### 2.1 Minimal Example

The fastest way to understand Okyline is by example:

```json
{
  "$oky": {
    "username|@ {3,20}": "alice",
    "email|@ ~$Email~": "alice@example.com",
    "age|@ (18..120)": 25,
    "role|('USER','ADMIN')": "USER",
    "verified|?": true
  }
}
```

**Reading this:**
- `username|@ {3,20}` → Required string, 3-20 characters
- `email|@ ~$Email~` → Required, email format
- `age|@ (18..120)` → Required integer, 18-120
- `role|(...)` → Enum: USER or ADMIN
- `verified|?` → nullable

### 2.2 Core Constraint Symbols

| Symbol | Type | Meaning | Example |
|--------|------|---------|---------|
| `@` | All | Required | `"name|@": "value"` |
| `?` | All | nullable | `"field|?": "value"` |
| `{min,max}` | String | Length | `"name|{2,50}": "Alice"` |
| `(min..max)` | Number | Range | `"age|(18..65)": 30` |
| `('a','b')` | All | Enum | `"status|('ACTIVE','INACTIVE')"` |
| `~format~` | String | Format/regex | `"email|~$Email~"` |
| `[min,max]` | Array | Size | `"items|[1,10]": [...]` |
| `->` | Array/Map | Element constraint | `"tags|[*] -> {2,10}"` |
| `!` | Array | Uniqueness | `"codes|[*]!": ["A","B"]` |
| `#` | Scalar | Key field | `"id|#": 123` |

**For complete details:** See [Section 5: Constraint Reference](#5-constraint-reference)

## 3. Type System

### 3.1 Type Inference

Okyline does **not** require explicit type declarations. The field type is automatically inferred from the example value provided.

**Principle:** The example value determines the type that will be validated.

### 3.2 Recognized Types

| Type | Example Value | Description |
|------|---------------|-------------|
| **String** | `"Alice"` | Unicode text |
| **Integer** | `42` | Whole numbers |
| **Number** | `3.14` | Floating-point numbers |
| **Boolean** | `true` or `false` | Boolean values |
| **Array** | `["a", "b"]` | Ordered lists |
| **Object** | `{"key": "value"}` | Nested structures |

### 3.3 Type Inference Rules

#### Rule 1: Example value determines type

```json
"street": "Yellow tree avenue"  // → String
"age": 42                        // → Integer
"price": 35.5                    // → Number
"active": true                   // → Boolean
"tags": ["eco", "garden"]        // → Array[String]
"address": {"city": "Paris"}     // → Object
```

**Note:** JSON discards trailing zeros (e.g., `78.00` becomes `78`). To preserve decimal inference, you can write `"78.00"` as a string - Okyline automatically converts it to Number. See [§6.4](#64-type-inference-modifiers).

#### Rule 2: Arrays infer element type from first item

```json
"scores": [10, 20, 30]           // → Array[Integer]
"prices": [10.5, 20.99]          // → Array[Number]
"labels": ["A", "B", "C"]        // → Array[String]
```

**Important:** All elements in an array must conform to the inferred type.

#### Rule 3: Empty arrays are prohibited

Arrays in schema examples **MUST** contain at least one element for type inference.

```json
// ❌ INVALID - cannot infer type
"tags": []

// ✅ VALID
"tags|[0,10]": ["example"]
```

#### Rule 4: Null values are prohibited as examples

`null` cannot be used as an example value. To indicate that a field can be null, use the `?` constraint.

```json
// ❌ INVALID
"middleName": null

// ✅ VALID - field can be null
"middleName|?": "John"
```

### 3.4 Type Coercion

Okyline does **not** perform type coercion during validation. A value must exactly match the inferred type.

```json
"age": 42  // Integer expected

// Validation results:
42     → ✅ Valid
"42"   → ❌ Invalid (string, not integer)
42.0   → ❌ Invalid (number, not integer)
```

---

## 4. Field Syntax

### 4.1 General Syntax

Every field in an Okyline schema follows this pattern:

```
"fieldName | constraints | label": exampleValue
```

**Components:**
1. **fieldName** (required): The JSON field name
2. **constraints** (optional): Pipe-separated constraints
3. **label** (optional): The JSON field label


### 4.2 Field Name

The base identifier for the field in JSON documents.

**Rules:**
- Must be a valid JSON string
- No special restrictions beyond JSON requirements

### 4.3 Constraints

Constraints applied to the field, separated from the field name by `|`.

**General form:** `fieldName|constraint1 constraint2 constraint3`

**Example:**
```json
"email|@ ~$Email~": "user@example.com"
```
- `@` → required field
- `~$Email~` → must match email format

**Whitespace policy:** Spaces are allowed **everywhere** for readability. All the following forms are equivalent:
```json
// Compact form (no spaces)
"email|@ ~$Email~": "user@example.com"

// Spaced form (readable)
"email| @ ~$Email~": "user@example.com"

// Fully spaced (maximum readability)
"email | @ ~$Email~ ": "user@example.com"

// Complex example - all equivalent:
"tags|@ [1,10]->{2,20}!": ["eco", "bio"]
"tags|@ [1,10] -> {2,20} !": ["eco", "bio"]
"tags | @ [1, 10] -> {2, 20} !": ["eco", "bio"]
"tags | @ [ 1 , 10 ] -> { 2 , 20 } ! ": ["eco", "bio"]
   ```

### 4.4 Label

An optional human-readable description placed after the constraints.

**Syntax:** `fieldName|constraints|Description text`

**Example:**
```json
"status|@ ('ACTIVE','INACTIVE')|User account status": "ACTIVE"
```
#### Normative Rule

- A label **MUST NOT** contain the vertical bar character `|` (U+007C) **after JSON decoding**.

**Purpose:**
- Self-documenting schemas
- UI generation
- Developer guidance

### 4.5 Comments

Okyline supports inline comments by prefixing attribute names with `//`.

#### 4.5.1 Syntax

A JSON key starting with `//` (U+002F U+002F) is treated as a **comment** and ignored during parsing.

**Syntax:** `"//anyText": anyValue`

#### 4.5.2 Scope

When an attribute is commented out, **the entire subtree** rooted at that attribute is ignored, including the attribute value and all nested children.

#### 4.5.3 Applicable Contexts

Comments are recognized in all structural blocks:

| Block | Example |
|-------|---------|
| `$oky` | `"//user\|@": {...}` |
| `$nomenclature` | `"//COLORS": "RED,GREEN"` |
| `$format` | `"//Phone": "^\\+[0-9]+$"` |
| `$defs` | `"//Address": {...}` |
| `$compute` | `"//total": "a + b"` |

#### 4.5.4 Normative Rules

1. **Prefix matching:** A key is a comment if and only if it starts with exactly `//` after JSON decoding.

2. **Subtree exclusion:** The commented attribute and its entire value subtree MUST be ignored by the parser. No validation, type inference, or constraint processing occurs on commented content.

3. **No nesting effect:** Comments do not propagate. Uncommenting a parent does not automatically uncomment its children.

4. **JSON validity:** Commented entries MUST remain valid JSON. The value associated with a comment key MUST be a valid JSON value.

### 4.6 Example Value

The value provided determines the expected type and serves as documentation.

**Rules:**
- MUST be a valid JSON value
- MUST NOT be `null` (use `?` constraint instead)
- For arrays: MUST contain at least one element
- Should represent a realistic example

---

## 5. Constraint Reference

### 5.1 Scalar Field Constraints

These constraints apply to string, integer, number, and boolean fields.

#### 5.1.1 `@` - Required Field

Indicates that the field MUST be present in validated documents.

**Applies to:** All types

**Example:**
```json
"name|@": "Alice"
```

**Validation:**
- `{"name": "Bob"}` → ✅ Valid
- `{}` → ❌ Invalid (missing required field)

#### 5.1.2 `?` - Nullable Field

Indicates that the field can contain `null` values.

**Applies to:** All types

**Example:**
```json
"middleName|?": "Marie"
```

**Validation:**
- `{"middleName": "John"}` → ✅ Valid
- `{"middleName": null}` → ✅ Valid
- `{}` → ✅ Valid (field is optional)

**Note:** A field can be both required and nullable using `@?` - the field must be present but can be null.

#### 5.1.3 `{...}` - String Length

Restricts the character length of string values measured in Unicode code points

**Applies to:** String only

**Syntax variants:**
- `{max}` - Maximum length
- `{min,max}` - Minimum and maximum length

**Examples:**
```json
"username|{3,10}": "Alice"   // min 3, max 10 characters
"city|{50}": "Paris"          // max 50 characters (no minimum)
"code|{5,5}": "ABC12"         // exactly 5 characters
```

**Validation:**
- Length is measured in Unicode code points
- Empty string has length 0
- Minimum defaults to 0 if not specified

#### 5.1.4 `(...)` - Value Constraints

Restricts values to specific sets, ranges, or conditions.

**Applies to:** Strings, integers, numbers

##### Discrete Values (Enumeration)

List specific allowed values separated by commas.

**Syntax:** `('value1','value2','value3')`

**Example:**
```json
"status|('ACTIVE','INACTIVE','PENDING')": "ACTIVE"
```

**Validation:**
- `{"status": "ACTIVE"}` → ✅ Valid
- `{"status": "DELETED"}` → ❌ Invalid

##### Numeric Ranges

Define inclusive ranges using `..` notation.

**Syntax:** `(min..max)`

**Example:**
```json
"age|(18..120)": 30
"price|(0..1000)": 49.99
```

**Validation:**
- Boundaries are **inclusive**: `(1..10)` accepts both 1 and 10
- Works with integers and numbers

##### Numeric Comparisons

Use comparison operators for one-sided constraints.

**Operators:** `>`, `<`, `>=`, `<=`

**Examples:**
```json
"quantity|(>0)": 5           // strictly greater than 0
"discount|(<=50)": 20        // less than or equal to 50
"score|(>=10)": 85           // greater than or equal to 10
```

##### Lexicographic Ranges

String ranges using alphabetical ordering.

**Syntax:** `('start'..'end')`

**Example:**
```json
"letter|('A'..'Z')": "B"     // A through Z (single uppercase letter)
"grade|('A'..'F')": "C"      // Letter grades
```

**Note:** Comparisons use Unicode lexicographic ordering.

##### Combined Constraints

Combine multiple constraints with commas.

**Examples:**
```json
"value|(1,2..5,>10)": 12     // equals 1, OR between 2-5, OR greater than 10
"code|('A','B',100..200)": "A"
```

**Validation logic:** Value must satisfy **at least one** of the listed constraints (logical OR).

##### Reference to Nomenclature

Use values from a predefined registry (see §6.1).

**Syntax:** `($REGISTRY_NAME)`

**Example:**
```json
{
  "$oky": {
    "favoriteColor|($COLORS)": "RED"
  },
  "$nomenclature": {
    "COLORS": "RED,GREEN,BLUE,YELLOW"
  }
}
```

#### 5.1.5 `~...~` - Format Validation

Validates strings against regular expressions or semantic formats.

**Applies to:** String only

##### Inline Regex

Embed a regular expression directly in the constraint.

**Regex flavor:** Okyline uses **ECMA-262** (JavaScript) regular expression syntax.

**Syntax:** `~pattern~`

**Examples:**
```json
"postalCode|~^[0-9]{5}$~": "75001"
"phoneNumber|~^\\+33[0-9]{9}$~": "+33612345678"
"sku|~^[A-Z]{2}-\\d{4}$~": "AB-1234"
```

**Important:**
- Backslashes must be escaped in JSON strings (`\d` becomes `\\d`)
- Regex delimiters (`/`) are **not** used; the tilde `~` serves as delimiter
- Flags are not supported in inline patterns (use anchors `^` `$` instead)

**ECMA-262 reference:** https://262.ecma-international.org/13.0/#sec-regexp-regular-expression-objects

##### Named Formats

Reference predefined patterns from `$format` block or use built-in formats.

**Syntax:** `~$FormatName~`

**Example:**
```json
{
  "$oky": {
    "zipCode|~$PostalCode~": "75001"
  },
  "$format": {
    "PostalCode": "^[0-9]{5}$"
  }
}
```

> **Non-normative - Implementation note (regex)**
>
> Okyline specifies ECMA-262 regex syntax and match semantics.
> Operational concerns such as regex timeouts, backtracking limits, input size limits, or sandboxing are **out of scope** for this specification and are left to implementations.
> Implementations **MAY** apply safeguards (e.g., timeouts or complexity limits). If a safeguard prevents evaluation, the validator **SHOULD** report a clear execution error rather than altering match semantics.

##### Built-in Formats

Okyline provides standard formats that can be used without declaration.

**Semantic validation** (validates logical consistency):

| Format | Description | Validation | Example |
  |--------|-------------|------------|---------|
| `$Date` | ISO 8601 date | Validates leap years, month lengths, day ranges | `"2025-05-30"` |
| `$DateTime` | ISO 8601 datetime | Validates date/time consistency, timezone offsets | `"2025-05-30T14:30:00Z"` |

**Syntactic validation with structural constraints**:

| Format      | Description     | Validation                                             | Example                           |
  |-------------|-----------------|--------------------------------------------------------|-----------------------------------|
| `$Time`     | ISO 8601 time   | RFC 3339 syntax (HH:MM:SS + optional fractions/offset) | `"14:30:00"`,`"14:30:00.123Z"`    |
| `$Uri`      | URI with scheme | RFC 3986 syntax + port range validation (1-65535)      | `"https://example.com:8080/path"` |
| `$Ipv4`     | IPv4 address    | Dotted-decimal notation, octets 0-255                  | `"192.168.1.1"`                   |
| `$Ipv6`     | IPv6 address    | RFC 4291 syntax + single `::` compression              | `"2001:db8::1"`                   |
| `$Hostname` | Hostname        | RFC 1034 labels + total length ≤ 255 chars             | `"example.com"`                   |

**Syntactic validation** (pattern matching):

| Format | Description | Example |
  |--------|-------------|---------|
| `$Email` | Email address | `"user@example.com"` |
| `$Uuid` | UUID (versions 1-5) | `"550e8400-e29b-41d4-a716-446655440000"` |

**Overriding built-in formats:**

All built-in formats, including semantic formats, **can be overridden** in the `$format` block to accommodate different validation rules or regional formats.

**Example - Override `$Date` for European format:**
```json
{
  "$oky": {
    "birthDate|~$Date~": "15/05/90",
    "eventDate|~$Date~": "31/12/25"
  },
  "$format": {
    "Date": "^(0[1-9]|[12]\\d|3[01])/(0[1-9]|1[0-2])/\\d{2}$"
  }
}
```

**Warning:** When overriding semantic formats with regex, you lose semantic validation. The pattern `31/02/25` would be syntactically valid but semantically incorrect (February 31 doesn't exist).

**Example - Override `$Email` for stricter validation:**
```json
{
  "$format": {
    "Email": "^[a-zA-Z0-9._%+-]+@ [a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
  },
  "$oky": {
    "email|~$Email~": "user@example.com"
  }
}
```

**Validation examples:**
```json
// Default $Date - Semantic validation (ISO 8601)
"date|~$Date~": "2024-02-29"  // ✅ Valid (2024 is leap year)
"date|~$Date~": "2025-02-29"  // ❌ Invalid (2025 is not leap year)
"date|~$Date~": "2025-13-01"  // ❌ Invalid (month 13 doesn't exist)

// Custom $Date - Regex validation (DD/MM/YY)
{
  "$format": {
    "Date": "^(0[1-9]|[12]\\d|3[01])/(0[1-9]|1[0-2])/\\d{2}$"
  },
  "$oky": {
    "date|~$Date~": "29/02/25"  // ✅ Syntactically valid (but semantically wrong!)
  }
}
```
#### 5.1.6 `#` - Key Field

Marks a field as an identifier within an object. Used for enforcing uniqueness in lists of objects.

**Applies to:** Scalars inside objects

**Example:**
```json
{
  "users|@ [1,10] -> !": [
    {
      "id|#": "user001",
      "name": "Alice"
    },
    {
      "id|#": "user002",
      "name": "Bob"
    }
  ]
}
```

**Behavior:**
- Key fields are used to determine object uniqueness in lists
- Multiple fields can be marked as keys (composite key)
- See §5.2.3 for uniqueness validation

#### 5.1.7 `%` - Default Value

Indicates that the example value is also the default value. This is **informational only** and does not affect validation.

**Applies to:** All types

**Example:**
```json
"country|%": "France",
"theme|%('light','dark')": "light"
```

**Use cases:**
- UI form generators
- Documentation
- Code generation tools

### 5.2 List Constraints

These constraints apply to array fields.

#### 5.2.1 `[...]` - List Size

Defines minimum and maximum number of elements in a list.

**Syntax variants:**
- `[max]` - Maximum size only
- `[min,max]` - Minimum and maximum
- `[min,*]` - Minimum only (no maximum)
- `[*]` - Any size (no constraints)

**Examples:**
```json
"tags|[1,5]": ["eco", "garden"]          // 1 to 5 items
"codes|[10,*]": ["A","B"]                // at least 10 items
"letters|[5]": ["A","B"]                 // max 5 items
"items|[*]": ["x"]                       // any number of items
```

**Empty Lists & Type Inference**

Array examples **MUST NOT** be empty, regardless of their size constraints.

**Validation:**
- Empty lists are valid if minimum is 0 or `[*]`
- Boundaries are inclusive

#### 5.2.2 `->` - Element/Value Constraints

Applies constraints to each element in a collection (list or map).

**Syntax:**
- Arrays: `[size] -> constraints`
- Maps: `[key:size] -> constraints`

**Examples:**
```json
// Each tag must be 2-10 characters and unique
"tags|@ [1,5] -> {2,10}!": ["eco", "garden"]

// Each score must be between 0-100
"scores|[*] -> (0..100)": [85, 92, 78]

// Each email must match format
"contacts|[1,10] -> ~$Email~": ["a@example.com", "b@example.com"]
```

**Combination with object constraints:**
```json
"products|[1,50] -> !": [
  {
    "sku|#": "ABC123",
    "name|@ {2,100}": "Product Name",
    "price|@ (0..10000)": 29.99
  }
]
```

#### 5.2.3 `!` - Uniqueness

Enforces that all elements in a list are unique.

**Applies to:** Arrays

**Behavior:**
- For scalar lists: uniqueness by value
- For object lists: uniqueness by composite key formed from key fields (`#`)

**Examples**

**Scalar uniqueness:**
```json
"codes|[1,10] -> !": ["A001", "B002", "C003"]
```

- ✅ Valid: `["A", "B", "C"]`
- ❌ Invalid: `["A", "B", "A"]` (duplicate "A")

---

**Object-based Uniqueness**

When applied to a list of objects, the `!` modifier enforces uniqueness based on one or more fields marked with `#`.

**Requirements:**
- At least one key field (`#`) **MUST** be declared
- If no key fields are declared, validation **MUST fail**
- An object missing **all** declared key fields **MUST** cause a validation error
- An object missing **some** key fields is valid; only present ones contribute to the composite key

**Example:**
```json
"records|[*] -> !": [
  {"type|#": "A", "code|#": "001", "label": "First"},
  {"type|#": "A", "code|#": "002", "label": "Second"},
  {"type|#": "B", "code|#": "001", "label": "Third"}
]
```

- ✅ Valid: all composite keys (`A-001`, `A-002`, `B-001`) are unique
- ❌ Invalid: two objects with the same composite key

---

**Composite Key Construction**

When multiple fields are marked as key fields (modifier `#`), Okyline constructs a composite key to verify uniqueness of objects in a list.

**Algorithm:**

The composite key is formed by concatenating the values of fields marked with `#`, in their **order of declaration** in the schema, separated by a hyphen (`-`).

**Rules:**

1. **Separator:** `-` (U+002D HYPHEN-MINUS)
2. **Encoding:** Each value is URL-encoded according to RFC 3986 (UTF-8)
3. **Number normalization:** Trailing zeros are removed (`1.0` → `1`, `123.000` → `123`)
4. **Booleans:** Converted to `true` or `false` (lowercase)
5. **Null/absent values:** Ignored (do not contribute to the key)
6. **Non-scalar fields:** Ignored (only scalar values are accepted as keys)

**Example 1: Two string fields**

Schema:
```json
"items|[*] -> !": [
  {
    "country|#": "FR",
    "code|#": "75001"
  }
]
```

Object: `{ "country": "FR", "code": "75001" }`
Composite key: `"FR-75001"`

---

**Example 2: String + number**

Schema:
```json
"sessions|[*] -> !": [
  {
    "userId|#": 42,
    "sessionId|#": "abc-123"
  }
]
```

Object: `{ "userId": 42, "sessionId": "abc-123" }`
Composite key: `"42-abc%2D123"` (hyphen in "abc-123" is URL-encoded)

---

**Example 3: Missing value**

Schema:
```json
"addresses|[*] -> !": [
  {
    "country|#": "FR",
    "region|#?": "IDF",
    "code|#": "75001"
  }
]
```

Object: `{ "country": "FR", "code": "75001" }` (region absent)
Composite key: `"FR-75001"` (region ignored because absent)

---

**Example 4: Number normalization**

Schema:
```json
"products|[*] -> !": [
  {
    "sku|#": "ABC",
    "version|#": 1.0
  }
]
```

Object 1: `{ "sku": "ABC", "version": 1.0 }`
Object 2: `{ "sku": "ABC", "version": 1 }`
Both have composite key: `"ABC-1"` → ❌ Duplicate!

---

**Example 5: Boolean values**

Schema:
```json
"flags|[*] -> !": [
  {
    "name|#": "feature",
    "enabled|#": true
  }
]
```

Object: `{ "name": "feature", "enabled": true }`
Composite key: `"feature-true"`

---

**Example 6: URL encoding**

Schema:
```json
"paths|[*] -> !": [
  {
    "path|#": "/api/v1",
    "method|#": "GET"
  }
]
```

Object: `{ "path": "/api/v1", "method": "GET" }`
Composite key: `"%2Fapi%2Fv1-GET"` (slashes are URL-encoded)

---

**Error Handling**

| Situation | Behavior |
|-----------|----------|
| No key fields declared | ❌ Validation error if uniqueness (`!`) is requested |
| All key fields absent/null in an object | ❌ Validation error (uniqueness not verifiable) |
| Duplicate composite keys | ❌ Validation error: `NOT_UNIQUE` |
| Non-scalar key field (object/array) | Field is ignored in key construction |

**Example - Error cases:**
```json
// ❌ No key fields declared
"items|[*] -> !": [
  {"name": "A"},
  {"name": "B"}
]
// Error: Cannot verify uniqueness without key fields

// ❌ All key fields absent
"items|[*] -> !": [
  {"id|#": 1, "name": "A"},
  {"name": "B"}  // id missing
]
// Error: Object missing all key fields

// ❌ Duplicate keys
"items|[*] -> !": [
  {"id|#": 1, "name": "A"},
  {"id|#": 1, "name": "B"}
]
// Error: Duplicate key "1"
```

---

**Design Rationale**

✅ **Explicit semantics** via mandatory key field declaration
✅ **Superior performance** through O(n) hash-based checking
✅ **Business alignment** by validating identity, not structure
✅ **Clear errors** when uniqueness cannot be established
✅ **No collision ambiguity** via RFC 3986 URL encoding

This design makes Okyline schemas self-documenting and production-ready for high-performance validation scenarios.

### 5.3 Map Constraints

Maps are objects used as dictionaries with dynamic keys.

#### 5.3.1 `[key_constraint:size_constraint]` - Map Constraints

**Syntax:** `[key_pattern:max_entries]`

**Variants:**
- `[*:max]` - Any string key, maximum entries
- `[~pattern~:max]` - Keys matching regex, maximum entries
- `[~pattern~:*]` - Keys matching regex, any number of entries

**Examples:**

**Free keys:**
```json
"translations|[*:5]": {
  "en": "Hello",
  "fr": "Bonjour",
  "es": "Hola"
}
```
- Any string keys allowed
- Maximum 5 entries

**Pattern-constrained keys:**
```json
"products|[~^SKU-\\d{5}$~:*]": {
  "SKU-12345": {
    "name|@": "Product A",
    "price|@ (0..1000)": 29.99
  },
  "SKU-67890": {
    "name|@": "Product B",
    "price|@ (0..1000)": 49.99
  }
}
```
- Keys must match pattern `SKU-` followed by 5 digits
- Unlimited number of entries

**Language codes:**
```json
"labels|[~^[a-z]{2}(-[A-Z]{2})?$~:10] -> {1,100}": {
  "en": "Label",
  "fr": "Étiquette",
  "en-US": "Label (US)"
}
```
- Keys match ISO language codes
- Maximum 10 translations
- Each value 1-100 characters

### 5.4 Polymorphism Constraints

#### 5.4.1 `$oneOf` - Exclusive Match

The value must match **exactly one** of the provided schema examples.

A candidate schema **matches** if validating the instance against that schema produces **no validation errors**.

Matching is evaluated independently for each candidate schema.

**Applies to:** Objects and arrays of objects

**Example:**
```json
"payment|@ $oneOf": [
  {
    "type|@ ('card')": "card",
    "cardNumber|@ {16}": "1234567812345678",
    "cvv|@ {3}": "123"
  },
  {
    "type|@ ('paypal')": "paypal",
    "email|@ ~$Email~": "user@example.com"
  },
  {
    "type|@ ('bank')": "bank",
    "iban|@ {15,34}": "FR7630006000011234567890189"
  }
]
```

**Validation:**
- Document must match one and only one schema
- Matching is typically determined by a discriminator field (here: `type`)

#### 5.4.2 `$anyOf` - Non-Exclusive Match

The value must match **at least one** of the provided schema examples.

A candidate schema **matches** if validating the instance against that schema produces **no validation errors**.

**Applies to:** Objects and arrays of objects

**Default behavior:** When multiple object examples are provided without an explicit `$oneOf` or `$anyOf` modifier, `$anyOf` semantics apply implicitly. This applies uniformly to collections and to single-value fields using `$obj` (see §6.4.3).

**Example with explicit `$anyOf`:**
```json
"notification|$anyOf": [
  {"email|~$Email~": "user@example.com"},
  {"sms|~^\\+[0-9]{10,15}$~": "+33612345678"}
]
```

**Example with implicit `$anyOf` (no modifier):**
```json
"telecom|[*]": [
  {"system|@ ('phone')": "phone", "value|@ {1,100}": "+61355556473"},
  {"system|@ ('email')": "email", "value|@ ~$Email~": "user@example.com"}
]
```

**Validation:**
- The instance must match at least one of the candidate schemas
- When no explicit modifier is specified, `$anyOf` is the implicit default for any field with multiple object examples

### 5.5 Combination Rules

#### Rule 1: One constraint per type

Only **one** constraint of the same type is allowed per field.

**Invalid:**
```json
// ❌ Two length constraints
"name|{10,50}{5,20}": "Alice"

// ❌ Two value constraints
"age|(0..100)(18..65)": 30
```

**Valid:**
```json
// ✅ Multiple constraints in single constraint
"age|(0..18,65..100)": 75

// ✅ Different constraint types
"name|@ {2,50}": "Alice"
```

#### Rule 2: Different constraint types can be combined

```json
// ✅ Required + length + pattern
"email|@ {5,100}~$Email~": "user@example.com"

// ✅ Required + range + label
"age|@ (18..120)|User age in years": 30

// ✅ List size + element constraints + uniqueness
"tags|[1,5] -> {2,20}!": ["eco", "bio"]
```

#### Rule 3: Order matters for readability (not for parsing)

While order doesn't affect validation, the recommended order is:
1. Existence (`@`, `?`)
2. Value constraints (`(...)`, `{...}`, `~...~`)


**Recommended:**
```json
"email|@ {5,100}~$Email~|User email address": "user@example.com"
```

---

## 6. Advanced Features

### 6.1 Reusable Value Registries - `$nomenclature`

Define centralized lists of allowed values that can be reused across multiple fields.

**Purpose:**
- Centralize enum definitions
- Improve maintainability
- Ensure consistency

#### 6.1.1 Declaration

Declared at the root level alongside `$oky`.

**Syntax:**
```json
{
  "$nomenclature": {
    "REGISTRY_NAME": "value1,value2,value3"
  }
}
```

**Example:**
```json
{
  "$nomenclature": {
    "STATUS": "DRAFT,VALIDATED,REJECTED,ACTIVE,INACTIVE,ARCHIVED",
    "COUNTRIES": "FRA,DEU,ESP,USA,GBR",
    "UNITS": "kg,m,cm,L,°C"
  }
}
```

**Rules:**
- Keys are uppercase identifiers
- Values are comma-separated strings
- No quotes needed around individual items

#### 6.1.2 Usage

Reference a nomenclature using `($NAME)` syntax in value constraints.

**Example:**
```json
{
  "$nomenclature": {
    "COLORS": "RED,GREEN,BLUE,YELLOW"
  },
  "$oky": {
    "product": {
      "name|@": "T-Shirt",
      "color|@ ($COLORS)": "RED",
      "secondaryColor|($COLORS)": "BLUE"
    }
  }
}
```

**Validation:**
- `{"color": "RED"}` → ✅ Valid
- `{"color": "PURPLE"}` → ❌ Invalid

### 6.2 Reusable Formats - `$format`

Define named regular expressions for reuse across the schema.

#### 6.2.1 Declaration

Declared at root level alongside `$oky`.

**Syntax:**
```json
{
  "$format": {
    "PatternName": "^regular_expression$"
  }
}
```

**Example:**
```json
{
  "$format": {
    "PostalCode": "^[0-9]{5}$",
    "PhoneNumber": "^\\+33[0-9]{9}$",
    "Sku": "^SKU-[0-9]{5}$",
    "IsoDate": "^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\\d|3[01])$"
  }
}
```

**Important:** Remember to escape backslashes in JSON (`\d` → `\\d`).

#### 6.2.2 Usage

Reference with `~$PatternName~` syntax.

**Example:**
```json
{
  "$format": {
    "Code": "^[A-Z]{2}-\\d{4}$"
  },
  "$oky": {
    "session": {
      "code|@ ~$Code~": "SR-0012",
      "backupCode|~$Code~": "AB-9999"
    }
  }
}
```

#### 6.2.3 Format Resolution

When `~$Name~` is encountered, Okyline resolves in this order:

1. **Custom pattern:** Check `$format[Name]`
2. **Built-in format:** Check if `Name` is a recognized built-in (see §5.1.5)

**Override built-ins:**
```json
{
  "$format": {
    "Date": "^[0-9]{2}/[0-9]{2}/[0-9]{4}$"
  },
  "$oky": {
    "eventDate|~$Date~": "25/12/2025"
  }
}
```
- Custom `$Date` pattern overrides the built-in ISO date format


### 6.3 Presence & Conditional Directives

Apply structural changes based on field values or existence.

#### 6.3.1 `$requiredIf` - Conditional Required Fields

Require fields when a condition is met.

**Syntax:** `"$requiredIf condition": ["field1", "field2"]`

**Example:**
```json
{
  "$oky": {
    "person": {
      "age": 17,
      "parentConsent": true,
      "$requiredIf age(<18)": ["parentConsent"]
    }
  }
}
```
- If age < 18, then `parentConsent` must be present

#### 6.3.2 `$requiredIfNot` - Required If Condition Not Met

Require fields when a condition is NOT met.

**Syntax:** `"$requiredIfNot condition": ["field1", "field2"]`

**Example:**
```json
{
  "$oky": {
    "person": {
      "age": 25,
      "idCard": "AB123456",
      "$requiredIfNot age(<18)": ["idCard"]
    }
  }
}
```
- If age is NOT < 18 (i.e., >= 18), then `idCard` must be present

#### 6.3.3 `$forbiddenIf` - Conditional Forbidden Fields

Forbid fields when a condition is met.

**Syntax:** `"$forbiddenIf condition": ["field1", "field2"]`

**Example:**
```json
{
  "$oky": {
    "account": {
      "status": "CLOSED",
      "lastLogin": "2025-01-15",
      "$forbiddenIf status('CLOSED')": ["lastLogin"]
    }
  }
}
```
- If status is "CLOSED", `lastLogin` must not be present

#### 6.3.4 `$forbiddenIfNot` - Forbidden If Condition Not Met

Forbid fields when a condition is NOT met.

**Syntax:** `"$forbiddenIfNot condition": ["field1", "field2"]`

**Example:**
```json
{
  "$oky": {
    "account": {
      "status": "ACTIVE",
      "closureReason": "Moved abroad",
      "$forbiddenIfNot status('CLOSED')": ["closureReason"]
    }
  }
}
```
- If status is NOT "CLOSED", then `closureReason` must not be present

#### 6.3.5 `$appliedIf` - Conditional Structure

Add fields dynamically based on a condition.

**Syntax variants:**

**Simple if/else:**
```json
"$appliedIf condition": {
  "field1|@": value,
  "$else": {
    "field2|@": value
  }
}
```

**Example:**
```json
{
  "$oky": {
    "employee": {
      "status": "ACTIVE",
      "$appliedIf status('ACTIVE')": {
        "workDays|@ (1..22)": 20
      }
    }
  }
}
```

```json
{
  "$oky": {
    "employee": {
      "status": "ACTIVE",
      "$appliedIf status('ACTIVE')": {
        "workDays|@ (1..22)": 20,
        "$else": {
          "reason|@": "On leave"
        }
      }
    }
  }
}
```


**Switch-case:**
```json
"$appliedIf fieldName": {
  "('value1')": { "field1|@": value },
  "('value2')": { "field2|@": value },
  "$else": { "field3|@": value },
  "$notExist": { "field4|@": value }
}
```

**Example:**
```json
{
  "$oky": {
    "employee": {
      "status": "ACTIVE",
      "$appliedIf status": {
        "('ACTIVE')": {
          "workDays|@ (1..22)": 20
        },
        "('INACTIVE')": {
          "reason|@": "On leave"
        },
        "$else": {
          "note|@": "Status unknown"
        }
      }
    }
  }
}
```

**Special branches in switch-case:**

| Branch | Triggers when |
|--------|---------------|
| `$else` | The trigger field exists but its value does not match any listed branch |
| `$notExist` | The trigger field is **absent** from the validated data |

`$else` and `$notExist` are mutually independent: if the trigger field is absent, only `$notExist` applies (not `$else`). If the trigger field is present but matches no branch, only `$else` applies. Both can be declared in the same switch-case block.

#### 6.3.6 `$requiredIfExist` - Existence-Based Required

Require fields if another field exists.

**Example:**
```json
{
  "$oky": {
    "contact": {
      "firstName": "John",
      "$requiredIfExist firstName": ["lastName"]
    }
  }
}
```

#### 6.3.7 `$requiredIfNotExist` - Require If Field Does Not Exist

Require fields if another field does NOT exist.

**Example:**
```json
{
  "$oky": {
    "contact": {
      "$requiredIfNotExist email": ["phone"]
    }
  }
}
```
- If `email` does not exist, then `phone` must be present

#### 6.3.8 `$forbiddenIfExist` - Existence-Based Forbidden

Forbid fields if another field exists.

**Example:**
```json
{
  "$oky": {
    "product": {
      "archived": true,
      "$forbiddenIfExist archived": ["active"]
    }
  }
}
```

#### 6.3.9 `$forbiddenIfNotExist` - Forbid If Field Does Not Exist

Forbid fields if another field does NOT exist.

**Example:**
```json
{
  "$oky": {
    "product": {
      "$forbiddenIfNotExist sku": ["internalCode"]
    }
  }
}
```
- If `sku` does not exist, then `internalCode` must not be present

#### 6.3.10 `$appliedIfExist` - Existence-Based Structure

Add fields if another field exists.

**Example:**
```json
{
  "$oky": {
    "order": {
      "tracking": "ABC123",
      "$appliedIfExist tracking": {
        "carrier|@": "DHL",
        "estimatedDelivery|@ ~$Date~": "2025-12-25"
      }
    }
  }
}
```

#### 6.3.11 `$appliedIfNotExist` - Add Fields If Field Does Not Exist

Add fields dynamically if another field does NOT exist.

**Example:**
```json
{
  "$oky": {
    "contact": {
      "$appliedIfNotExist email": {
        "phone|@": "+33612345678",
        "phoneVerified|@": true
      }
    }
  }
}
```
- If `email` does not exist, then `phone` and `phoneVerified` fields are required

#### 6.3.12 `$required` — Unconditional Required Fields

Forces the presence of listed fields in the current object.

**Syntax:** `"$required": ["field1", "field2"]`

The value MUST be a non-empty array of strings. Each string identifies a field name that MUST be present in the validated data. This directive is valid in any object node, including `$appliedIf`, `$else`, and switch-case payloads. A field already marked `@` is accepted but redundant.

#### 6.3.13 `$forbidden` — Unconditional Forbidden Fields

Forces the exclusion of listed fields from the current object.

**Syntax:** `"$forbidden": ["field1", "field2"]`

The value MUST be a non-empty array of strings. Each string identifies a field name that MUST NOT be present in the validated data. This directive is valid in any object node, including `$appliedIf`, `$else`, and switch-case payloads.

#### 6.3.14 `$atLeastOne` — At Least One Required

Requires that at least one field from a given group is present.

**Syntax:** `"$atLeastOne": ["field1", "field2", ...]`

The value MUST be a non-empty array of at least two strings. At least one of the listed fields MUST be present in the validated data. To declare multiple independent groups in the same object, append a unique suffix: `$atLeastOne_contact`, `$atLeastOne_address`, etc. The suffix is semantically ignored and serves only to produce a unique JSON key.

**Example:**
```json
{
  "$oky": {
    "contact": {
      "email": "alice@example.com",
      "phone": "+33612345678",
      "$atLeastOne": ["email", "phone"]
    }
  }
}
```
- At least one of `email` or `phone` must be present.

#### 6.3.15 `$mutuallyExclusive` — Mutually Exclusive Fields

Ensures that at most one field from a given group is present.

**Syntax:** `"$mutuallyExclusive": ["field1", "field2", ...]`

The value MUST be a non-empty array of at least two strings. At most one of the listed fields MAY be present; having two or more is a validation error. The suffix mechanism (`$mutuallyExclusive_xxx`) applies as described in §6.3.14.

**Example:**
```json
{
  "$oky": {
    "payment": {
      "cardNumber": "4111111111111111",
      "iban": "FR7630006000011234567890189",
      "$mutuallyExclusive": ["cardNumber", "iban"]
    }
  }
}
```
- `cardNumber` and `iban` cannot both be present.

#### 6.3.16 `$exactlyOne` — Exactly One Required

Requires that exactly one field from a given group is present.

**Syntax:** `"$exactlyOne": ["field1", "field2", ...]`

The value MUST be a non-empty array of at least two strings. Exactly one of the listed fields MUST be present. Zero or more than one is a validation error. Semantically equivalent to combining `$atLeastOne` and `$mutuallyExclusive` on the same group. The suffix mechanism (`$exactlyOne_xxx`) applies as described in §6.3.14.

**Example:**
```json
{
  "$oky": {
    "auth": {
      "password": "secret",
      "oauthToken": "tok_abc123",
      "$exactlyOne": ["password", "oauthToken"]
    }
  }
}
```
- Exactly one of `password` or `oauthToken` must be present.

#### 6.3.17 `$allOrNone` — All Or None

Requires that either all fields in a group are present or none of them.

**Syntax:** `"$allOrNone": ["field1", "field2", ...]`

The value MUST be a non-empty array of at least two strings. Either all listed fields MUST be present, or none of them. Partial presence is a validation error. The suffix mechanism (`$allOrNone_xxx`) applies as described in §6.3.14.

**Example:**
```json
{
  "$oky": {
    "address": {
      "street": "123 Main St",
      "city": "Paris",
      "zip": "75001",
      "$allOrNone": ["street", "city", "zip"]
    }
  }
}
```
- All three fields must be present together, or none of them.

#### 6.3.18 Type Guards in Conditions

Type Guards allow checking the **runtime type** of a field value in conditional expressions.

**Syntax:** `fieldName(_TypeGuard_)`

**Important:** Type Guards can **only** be used in condition expressions (`$appliedIf`, `$requiredIf`, `$forbiddenIf`), not as field value constraints.

##### Available Type Guards

| Type Guard | Description |
|------------|-------------|
| `_Null_` | Value is null |
| `_Boolean_` | Value is a boolean |
| `_String_` | Value is a string |
| `_Integer_` | Value is an integer (no fractional part) |
| `_Number_` | Value is a number (integer or decimal) |
| `_Object_` | Value is an object |
| `_EmptyList_` | Value is an empty array |
| `_ListOfNull_` | Value is an array containing only null values |
| `_ListOfBoolean_` | Value is an array of booleans |
| `_ListOfString_` | Value is an array of strings |
| `_ListOfInteger_` | Value is an array of integers |
| `_ListOfNumber_` | Value is an array of numbers |
| `_ListOfObject_` | Value is an array of objects |

##### Examples

**Conditional structure based on type:**
```json
{
  "$oky": {
    "data": "example",
    "$appliedIf data(_String_)": {
      "length": 7,
      "$else": {
        "type": "non-string"
      }
    }
  }
}
```

**Require field based on array type:**
```json
{
  "$oky": {
    "items": [1, 2, 3],
    "$requiredIf items(_ListOfInteger_)": ["sum"]
  }
}
```

**Multiple type guards (OR logic):**
```json
{
  "$oky": {
    "value": null,
    "$appliedIf value(_String_,_Null_)": {
      "isTextOrEmpty": true
    }
  }
}
```

##### Notes

- `_Integer_` is stricter than `_Number_`: `3.0` matches `_Number_` but NOT `_Integer_`
- `_Number_` includes integers: `42` matches both `_Integer_` and `_Number_`
- `_ListOfXXX_` guards ignore null elements within the array for type inference
- `_EmptyList_` only matches arrays with zero elements
- Multiple guards are combined with comma and use OR logic

#### 6.3.19 Null Literal in Conditions

The `null` literal can be used in condition triggers to test if a field's value is null.

**Syntax:** `fieldName(null)`

**Important:** The `null` literal is **only** allowed in condition expressions (`$appliedIf`, `$requiredIf`, `$forbiddenIf` and their variants), not as field value constraints. Use the `?` modifier to indicate field nullability.

##### Difference from Type Guard `_Null_`

| Syntax | Meaning | Can mix with other values? |
|--------|---------|---------------------------|
| `field(_Null_)` | Type guard: checks if value **is** null | No (type guards cannot mix with value constraints) |
| `field(null)` | Value literal: matches null value | Yes |

The `null` literal allows combining null with other discrete values, which is not possible with the `_Null_` type guard.

##### Examples

**Require fallback when value is null:**
```json
{
  "$oky": {
    "item": {
      "value|?": "example",
      "fallback|?": "default",
      "$requiredIf value(null)": ["fallback"]
    }
  }
}
```

**Require field when value is NOT null:**
```json
{
  "$oky": {
    "item": {
      "config|?": "settings",
      "configLabel|?": "active",
      "$requiredIfNot config(null)": ["configLabel"]
    }
  }
}
```

**Mix null with other values (OR logic):**
```json
{
  "$oky": {
    "order": {
      "status|?": "ACTIVE",
      "reason|?": "explanation",
      "$requiredIf status('CANCELLED', 'INACTIVE', null)": ["reason"]
    }
  }
}
```
In this example, `reason` is required when `status` is `'CANCELLED'`, `'INACTIVE'`, or `null`.

##### Rules

- The `null` literal can be mixed with other value types (strings, numbers, booleans) in the same condition
- Multiple values are combined with comma and use OR logic: `field('A', 'B', null)` matches if value is `'A'` OR `'B'` OR `null`
- Using `null` in field value constraints (e.g., `"field|@ (null)": ...`) produces a parsing error

#### 6.3.20 Field Path Expressions

Conditional directives support **path expressions** to reference fields at any level of the document structure.

##### Applicability

Path expressions are supported in:
- **Trigger conditions** of all conditional directives (`$requiredIf`, `$forbiddenIf`, `$appliedIf`, and their `Exist`/`Not`/`NotExist` variants)
- **Target field lists** of `$requiredIf` and `$forbiddenIf` directives

##### Validation Context

The **validation context** is the object currently being validated in the schema tree. When validating:
- A root-level object: context is the root object
- A nested object: context is that nested object
- An array element: context is the element itself (not the array)

##### Nested Paths

Use dot notation to navigate into child objects from the current context.

**Syntax:** `field.subfield.subsubfield`

**Example:**
```json
{
  "$oky": {
    "company": {
      "info": {
        "type": "CORP"
      },
      "$appliedIf info.type('CORP')": {
        "registrationNumber|@": "RC-123456"
      }
    }
  }
}
```
- Context is `company`; `info.type` navigates to `company.info.type`

##### Parent Context

Use `parent` to reference the **parent object** of the current validation context.

**Syntax:** `parent`, `parent.field`, `parent.field.subfield`

**Example:**
```json
{
  "$oky": {
    "order": {
      "type": "WHOLESALE",
      "items": [{
        "name": "Widget",
        "$appliedIf parent.type('WHOLESALE')": {
          "bulkDiscount|@": 15
        }
      }]
    }
  }
}
```
- Context is the item `{"name": "Widget", ...}`
- `parent` refers to `order` (arrays are transparent in the parent chain)
- `parent.type` resolves to `"WHOLESALE"`

**Multi-level:** Use `parent.parent` to access grandparent, `parent.parent.parent` for great-grandparent, etc.

##### Root Context

Use `root` to reference the **document root object**, regardless of nesting depth.

**Syntax:** `root`, `root.field`, `root.field.subfield`

**Example:**
```json
{
  "$oky": {
    "config": {
      "strictMode": true
    },
    "data": {
      "items": [{
        "value": "test",
        "$appliedIf root.config.strictMode(true)": {
          "validatedBy|@": "admin"
        }
      }]
    }
  }
}
```

##### Explicit Current Context

Use `this` to explicitly reference the current validation context. This is equivalent to omitting the prefix but allows disambiguation when a field name collides with a reserved prefix.

**Syntax:** `this`, `this.field`

**Example:**
```json
{
  "$oky": {
    "node": {
      "parent": "value",
      "$appliedIf this.parent('value')": {
        "note|@": "Field named parent"
      }
    }
  }
}
```
- `this.parent` refers to the field named `parent`, not the parent context

##### Target Field Paths

Path expressions in target field lists reference fields relative to the current context.

**Example:**
```json
{
  "$oky": {
    "user": {
      "isPremium": true,
      "profile": {
        "displayName?": "str"
      },
      "$requiredIf isPremium(true)": ["profile.displayName"]
    }
  }
}
```
- When `isPremium` is `true`, the nested field `profile.displayName` must be present

##### Resolution Summary

| Prefix | Resolves From |
|--------|---------------|
| *(none)* | Current validation context |
| `this.` | Current validation context (explicit) |
| `parent.` | Parent object of current context |
| `root.` | Document root |

##### Normative Rules

1. **Parent chain:** Arrays are transparent in the parent chain. `parent` always refers to the nearest ancestor **object**, skipping any intermediate array containers.

2. **Parent at root:** Using `parent` when the current context is the root object evaluates to **not found** (condition is false).

3. **Reserved prefixes:** The keywords `parent`, `root`, and `this` are reserved as scope prefixes. A path segment matching these keywords at the start of an expression is interpreted as a scope reference, not a field name. To reference a field literally named `parent`, `root`, or `this`, use the `this.` prefix (e.g., `this.parent`, `this.root`).

4. **Path syntax:** A path expression MUST conform to one of the following forms:

   | Form | Example | Description |
   |------|---------|-------------|
   | `fieldPath` | `info.type` | From current context (implicit) |
   | `this.fieldPath` | `this.parent` | From current context (explicit) |
   | `parent[.parent]*.fieldPath` | `parent.parent.status` | From ancestor context |
   | `root.fieldPath` | `root.config.mode` | From document root |

   Where `fieldPath` is one or more field names separated by `.`. Field names MUST start with a letter or underscore and contain only alphanumeric characters and underscores.

   The prefixes `root`, `parent`, and `this` are mutually exclusive starting points and MUST NOT be combined (e.g., `parent.root.field` is invalid).

   Malformed paths (empty segments, leading/trailing dots, invalid characters) MUST be rejected as schema parsing errors.

5. **Case sensitivity:** Path resolution is case-sensitive. The reserved prefixes `parent`, `root`, and `this` MUST be lowercase. `Parent`, `ROOT`, or `This` are treated as field names, not scope references.

6. **Missing paths:** If any segment of a path does not exist or is not an object (except for the final segment), the path is considered **not found**:
   - A trigger condition on a non-existent path evaluates to **false**
   - A target field path that does not exist is treated as **missing** (for `$requiredIf`) or **absent** (for `$forbiddenIf`)

7. **Array traversal restriction:** Paths navigate object structures only. Array index notation (e.g., `items[0].name`) is **not supported**. Attempting to navigate through an array value (other than via the implicit array element context) results in **not found**.

---

### 6.4 Type Inference Modifiers

Okyline provides modifiers that override default type inference behavior.
These are useful when the example value structure differs from the intended schema type.

#### 6.4.1 Decimal String Inference (Default Behavior)

JSON numeric serialization discards trailing zeros from decimal values (e.g., `78.00` becomes `78`). To preserve decimal precision in examples, Okyline allows writing decimal values as strings.

##### Normative Rule (Default)

When an example value is a **string** representing a decimal numeric literal (containing a decimal point `.`):
- The value is **automatically** interpreted as a **Number**
- The numeric value is obtained by decimal parsing
- The inferred type is **Number**, never Integer
- This rule applies regardless of the fractional value

##### Examples (Default Behavior)

```json
"amount": "78.00"    // → Number (automatically converted to 78.0)
"price": "5.0"       // → Number (automatically converted to 5.0)
"rate": "0.125"      // → Number (automatically converted to 0.125)
"code": "78"         // → String (no decimal point, no conversion)
"value": 78          // → Integer
"value": 78.5        // → Number
```

#### 6.4.2 `$str` - Force String Type

Use `$str` to **prevent** the automatic decimal-to-number conversion and force the field to remain a String type.

**Syntax:** `fieldName|$str`

**Use case:** When a field looks like a decimal but should validate as a String (e.g., version numbers, codes).

##### Examples

```json
"version|$str": "1.0"        // → String (not converted to Number)
"productCode|$str": "78.00"  // → String (not converted to Number)
```

##### Normative Rules

1. `$str` prevents the automatic decimal string conversion
2. The field type remains **String**
3. String constraints like `{min,max}` apply to string length


#### 6.4.3 `$obj` - Single Value from Array Example

When an example value is an array but the field should accept a **single value** (not an array), use the `$obj` modifier. This allows providing multiple example values for documentation while defining a non-array field.

**Syntax:** `fieldName|$obj`

**Applies to:** All types (scalars and objects)

##### Scalar Fields

```json
"street|@ $obj {5,100}|Street address": ["123 Maple Street", "456 Oak Avenue"]
```

**Interpretation:**
- Without `$obj`: Field type would be `Array[String]`
- With `$obj`: Field type is `String`
- The constraint `{5,100}` applies to string length
- Each array element is validated as a valid string example

##### Object Fields

```json
"address|@ $obj": [
  {"city": "Paris", "zip": "75001"},
  {"city": "London", "zip": "SW1A 1AA"}
]
```

**Interpretation:**
- Without `$obj`: Field type would be `Array[Object]`
- With `$obj`: Field type is `Object`
- `$obj` is purely a **type inference modifier** — it prevents array inference from the example format
- When multiple object examples are provided, `$anyOf` semantics apply implicitly (see §5.4.2): the instance must match at least one of the examples. Use `$oneOf` to require an exclusive match.

##### Polymorphic Objects with `$oneOf`

A key use case is defining polymorphic fields where different object structures are valid:

```json
"payment|@ $oneOf $obj": [
  {
    "type|@ ('card')": "card",
    "number|@ {16,16}": "4111111111111111",
    "expiry|@ ~^(0[1-9]|1[0-2])/\\d{2}$~": "12/25"
  },
  {
    "type|@ ('paypal')": "paypal",
    "email|@ ~$Email~": "user@example.com"
  },
  {
    "type|@ ('transfer')": "transfer",
    "iban|@ {15,34}": "FR7630006000011234567890189"
  }
]
```

**Interpretation:**
- The `payment` field expects **one** object (not an array)
- `$oneOf` indicates the object must match exactly one of the variants
- Each array element defines a valid variant structure
- All variants are validated during schema parsing

##### Normative Rules

1. When `$obj` is present and the example value is an array:
   - The field type MUST be inferred from the **first element** (scalar type or object)
   - The field accepts a **single value**, not an array
   - `$obj` affects **only type inference**; it does not alter validation semantics. Multiple object examples follow `$anyOf` semantics by default (see §5.4.2), or `$oneOf` if explicitly specified.

2. When `$obj` is present and the example value is NOT an array:
   - The modifier has no effect; standard inference applies

3. An empty array with `$obj` is an error:
   - At least one example element is required for type inference

##### Summary Table

| Example | Without `$obj` | With `$obj` |
|---------|----------------|-------------|
| `["Alice", "Bob"]` | `Array[String]` | `String` |
| `[42, 100]` | `Array[Integer]` | `Integer` |
| `[{"a":1}, {"b":2}]` | `Array[Object]` | `Object` |


#### 6.4.4 Summary: Type Inference Modifiers

| Modifier | Purpose | Example Value | Inferred Type |
|----------|---------|---------------|---------------|
| *(none)* | Auto decimal conversion | `"78.00"` | `Number` |
| `$str` | Force string | `"78.00"` | `String` |
| *(none)* | Standard | `["a","b"]` | `Array[String]` |
| `$obj` | Force single value | `["a","b"]` | `String` |
| *(none)* | Standard | `[{...},{...}]` | `Array[Object]` |
| `$obj` | Force single value | `[{...},{...}]` | `Object` |

---

## 7. Document Structure

### 7.1 Root Structure

An Okyline document is a JSON object that MUST contain at least the `$oky` key.

**Minimal valid document:**
```json
{
  "$oky": {
    "message": "Hello"
  }
}
```

### 7.2 Mandatory Key

#### `$oky`

The central element containing the schema definition. This key is **required**.

**Type:** Object
**Content:** Field definitions with optional constraints

### 7.3 Optional Metadata Keys

#### `$okylineVersion`

Specifies the version of the Okyline specification used.

**Type:** String
**Format:** Semantic versioning (e.g., `"1.0"`, `"1.2.3"`)
**Default:** `"1.0"`

#### `$version`

Version of the schema itself (not the Okyline language).
**Optional** for standalone schemas; **required** for registry publication and external references (`$deps`, `$xDefs`)

**Type:** String
**Example:** `"1.2.3"`

#### `$title`

Human-readable title for the schema.

**Type:** String
**Example:** `"User Profile Schema"`

#### `$description`

Description of what the schema validates.

**Type:** String
**Example:** `"Schema for user profile data including personal information and preferences"`

#### `$additionalProperties`

Controls whether unknown fields are allowed in validated documents.

**Type:** Boolean
**Default:** `false` (unknown attributes are not allowed)

#### `$nullAsAbsentIfUndeclared`

Controls whether null values on non-nullable fields are treated as absent.

**Type:** Boolean
**Default:** `false` (null on a non-nullable field is a type error)

#### `$id`

Unique identifier for the schema within a registry. The identifier may include a namespace using dot notation.

**Type:** String
**Format:** `[namespace.]name` where:
- `namespace` (optional): Logical grouping, can be hierarchical (e.g., `sales`, `fr.company.team`)
- `name`: Schema name (part after the last `.`)

**Validation Rules:**
- Must not start or end with `.`
- Must not contain consecutive dots (`..`)
- Each segment must start with a letter and contain only alphanumeric characters and underscores
- Pattern: `^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)*$`

**Optional** for standalone schemas; **required** for registry publication and external references (`$deps`, `$xDefs`)
**Examples:**
- `"common"` - schema `common` in default namespace
- `"sales.orders"` - schema `orders` in namespace `sales`
- `"fr.company.billing.invoices"` - schema `invoices` in namespace `fr.company.billing`

### 7.4 Complete Example

```json
{
  "$okylineVersion": "1.0",
  "$version": "1.2.3",
  "$title": "User Profile",
  "$description": "Schema for user profiles with contact information",
  "$additionalProperties": false,
  "$oky": {
    "user": {
      "id|@": 123,
      "name|@ {2,50}": "Alice",
      "email|~$Email~": "alice@example.com"
    }
  }
}
```

### §7.3.5 `$additionalProperties`

#### Description
Controls whether unknown (undeclared) fields are allowed in validated documents.

#### Scope and Inheritance

- `$additionalProperties` **MAY** be defined at the **root level** of the Okyline document.
  When defined at the root, it applies **globally** to the entire JSON structure.

- `$additionalProperties` **MAY** also be defined **inside an object** within `$oky`.
  When defined inside an object, the rule **applies only to that specific object**,
  and **does not propagate recursively** to its child objects.

- Child objects **inherit** the global (root) rule **only if** they do not redefine it locally.

#### Normative Rules
- By default, if not specified, `$additionalProperties` is considered **false** (unknown fields are not allowed).
- A local `$additionalProperties` **overrides** the global rule **for that object only**.
- The setting **is not recursive** - it is **not inherited** by nested objects inside the one where it is defined.

#### Examples

**Global setting only:**
```json
{
  "$additionalProperties": false,
  "$oky": {
    "user": {
      "name|@": "Alice",
      "age|@": 30
    }
  }
}
```
→ Unknown fields anywhere are rejected.

---

**Local override (non-recursive):**
```json
{
  "$additionalProperties": false,
  "$oky": {
    "user": {
      "$additionalProperties": true,
      "name|@": "Alice",
      "address": {
        "street|@": "Main St"
      }
    }
  }
}
```

✅ `user` may include extra fields (e.g., `"nickname"`)
❌ `user.address` may **not** include unknown fields (no recursive propagation)

---

**Summary Table**

| Level | Applies To | Inherited by Children | Default |
|--------|-------------|----------------------|----------|
| Root | Entire document | ✅ (unless overridden) | `false` |
| Object (local) | That object only | ❌ (not recursive) | - |


---

### §7.3.6 `$nullAsAbsentIfUndeclared`

#### Description
Controls whether `null` values on non-nullable fields are treated as if the field were absent rather than producing a type error.

This directive addresses legacy systems that emit `null` instead of omitting optional fields.

#### Scope

- `$nullAsAbsentIfUndeclared` MAY be defined at the **root level** of the Okyline document.
- It applies **globally** to the entire JSON structure.
- There is **no local override** — the setting is uniform across the document.

#### Normative Rules

- By default, if not specified, `$nullAsAbsentIfUndeclared` is considered **false** (null on a non-nullable field is a type error).
- When **true**, a `null` value on a non-nullable field (without `?`) is treated as **absent**:
  - If the field is required (`@` or `$required`), a **missing field** error is reported instead of a type error.
  - If the field is optional, no error is reported.
- **Nullable fields (`?`) are never affected**: `null` on a `?` field remains a valid, intentional value regardless of this setting.
- This rule applies uniformly to **all presence checks**, including: required fields (`@`), `$required`, `$forbidden`, `$requiredIf*`, `$forbiddenIf*`, `$appliedIf*` (triggers and payloads), `$atLeastOne`, `$mutuallyExclusive`, `$exactlyOne`, `$allOrNone`.

#### Examples

**Strict mode (default):**
```json
{
  "$oky": {
    "user": {
      "name|@ {1,100}": "Alice",
      "age": 30
    }
  }
}
```
→ `{"name": null, "age": 25}` ❌ (null on non-nullable required field → type error)
→ `{"name": "Bob", "age": null}` ❌ (null on non-nullable field → type error)

---

**Tolerant mode:**
```json
{
  "$nullAsAbsentIfUndeclared": true,
  "$oky": {
    "user": {
      "name|@ {1,100}": "Alice",
      "age": 30,
      "nickname|?": "Al"
    }
  }
}
```
→ `{"name": null, "age": 25}` ❌ (null treated as absent → missing required field)
→ `{"name": "Bob", "age": null}` ✅ (null treated as absent → optional field, no error)
→ `{"name": "Bob", "nickname": null}` ✅ (nullable field → null is valid, unaffected by directive)

---

## 8. Validation Rules

### 8.1 Type Validation

Values must match the type inferred from the example.

**Valid:**
```json
// Schema: "age": 42 (Integer)
{"age": 30}  ✅
```

**Invalid:**
```json
{"age": "30"}    ❌ (string, not integer)
{"age": 30.5}    ❌ (number, not integer)
{"age": null}    ❌ (null, not integer - use ? constraint for nullable)
```

### 8.2 Required Field Validation

Fields marked with `@` must be present.

**Valid:**
```json
// Schema: "name|@": "Alice"
{"name": "Bob"}  ✅
```

**Invalid:**
```json
{}  ❌ (missing required field 'name')
```

### 8.3 Nullable Field Validation

Fields marked with `?` can be null or absent.

**Valid:**
```json
// Schema: "middleName|?": "John"
{"middleName": "Marie"}  ✅
{"middleName": null}     ✅
{}                       ✅
```

> **Note:** When `$nullAsAbsentIfUndeclared` is `true` (see §7.3.6), a `null` value on a field **without** `?` is treated as absent instead of producing a type error. Fields marked with `?` are never affected by this setting — `null` remains a valid value for them.

### 8.4 String Length Validation

Strings must satisfy length constraints.

**Valid:**
```json
// Schema: "username|{3,10}": "alice"
{"username": "bob"}       ✅ (length 3)
{"username": "alexander"} ✅ (length 9)
```

**Invalid:**
```json
{"username": "jo"}        ❌ (length 2, minimum is 3)
{"username": "verylongusername"} ❌ (length > 10)
```

### 8.5 Value Constraint Validation

Values must satisfy range, enum, or comparison constraints.

**Valid:**
```json
// Schema: "age|(18..65)": 30
{"age": 18}  ✅
{"age": 42}  ✅
{"age": 65}  ✅
```

**Invalid:**
```json
{"age": 17}  ❌ (below minimum)
{"age": 66}  ❌ (above maximum)
```

### 8.6 List Size Validation

Arrays must satisfy size constraints.

**Valid:**
```json
// Schema: "tags|[1,5]": ["eco"]
{"tags": ["a"]}               ✅
{"tags": ["a","b","c"]}       ✅
{"tags": ["a","b","c","d","e"]} ✅
```

**Invalid:**
```json
{"tags": []}                  ❌ (empty, minimum is 1)
{"tags": ["a","b","c","d","e","f"]} ❌ (6 items, maximum is 5)
```

### 8.7 Uniqueness Validation

Lists marked with `!` must have unique elements.

**Scalar uniqueness:**
```json
// Schema: "codes|[1,5] -> !": ["A"]

// Valid:
{"codes": ["A", "B", "C"]}  ✅

// Invalid:
{"codes": ["A", "B", "A"]}  ❌ (duplicate "A")
```

**Object uniqueness (by key field):**
```json
// Schema:
"users|[*] -> !": [
  {"id|#": "u1", "name": "Alice"}
]

// Valid:
{"users": [
  {"id": "u1", "name": "Alice"},
  {"id": "u2", "name": "Bob"}
]} ✅

// Invalid:
{"users": [
  {"id": "u1", "name": "Alice"},
  {"id": "u1", "name": "Charlie"}
]} ❌ (duplicate key "u1")
```

### 8.8 Regex Pattern Validation

Strings must match the specified pattern.

**Valid:**
```json
// Schema: "code|~^[A-Z]{2}-\\d{4}$~": "AB-1234"
{"code": "XY-9999"}  ✅
```

**Invalid:**
```json
{"code": "ab-1234"}  ❌ (lowercase letters)
{"code": "A-1234"}   ❌ (only one letter)
{"code": "AB-123"}   ❌ (only 3 digits)
```

### 8.9 Additional Properties

By default, unknown fields are **not allowed** unless `$additionalProperties: true`.

**Schema:**
```json
{
  "$additionalProperties": false,
  "$oky": {
    "user": {
      "name|@": "Alice"
    }
  }
}
```

**Valid:**
```json
{"user": {"name": "Bob"}}  ✅
```

**Invalid:**
```json
{"user": {"name": "Bob", "age": 30}}  ❌ (unknown field 'age')
```

### 8.10 Validation Error Messages

Implementations should provide clear error messages including:
- Field path (e.g., `user.address.zipCode`)
- Constraint violated (e.g., "string length", "required field")
- Expected vs. actual value
- Constraint details (e.g., "expected length 5-10, got 3")

---

## 9. Complete Examples

### 9.1 Minimal Example

```json
{
  "$oky": {
    "message|@ {1,100}": "Hello, Okyline!"
  }
}
```

**Validates:**
- Required string field named "message"
- Length between 1 and 100 characters

### 9.2 User Profile

```json
{
  "$okylineVersion": "1.4.0",
  "$version": "1.0.0",
  "$title": "User Profile",
  "$description": "Schema for user account information",
  "$oky": {
    "user": {
      "id|@ (>0)|User identifier": 12345,
      "username|@ {3,20}|Unique username": "alice_dev",
      "email|@ ~$Email~|Email address": "alice@example.com",
      "firstName|@ {1,50}|First name": "Alice",
      "lastName|@ {1,50}|Last name": "Smith",
      "dateOfBirth|~$Date~|Date of birth": "1990-05-15",
      "isActive|@|Account status": true,
      "roles|[1,5] -> ('admin','user','guest')!|User roles": ["user"],
      "preferences|?|User preferences": {
        "theme|('light','dark')": "light",
        "language|('en','fr','es')": "en"
      }
    }
  }
}
```

### 9.3 E-commerce Order

```json
{
  "$version": "2.1.0",
  "$id": "E-ORDER-001",
  "$title": "Order Schema",
  "$description": "Schema for e-commerce orders",
  "$oky": {
    "order": {
      "orderId|@ #~$OrderId~|Unique order identifier": "ORD-12345678",
      "customerId|@ (>0)|Customer ID": 42,
      "orderDate|@ ~$DateTime~|Order timestamp": "2025-01-15T10:30:00Z",
      "status|@ ($ORDER_STATUS)|Order status": "PENDING",
      "items|@ [1,100] -> !|Order items": [
        {
          "sku|@ #~$Sku~|Product SKU": "SKU-ABC12345",
          "name|@ {2,200}|Product name": "Wireless Mouse",
          "quantity|@ (1..1000)|Quantity": 2,
          "vat|(0.05,0.1,0.15,0.2)": 0.2,
          "unitPrice|@ (>0)|Unit price": 50.0
        }
      ],
      "shippingAddress|@|Shipping address": {
        "street|@ {5,100}": "123 Main Street",
        "city|@ {2,50}": "Paris",
        "postalCode|@ {5,10}": "75001",
        "country|@ {2}": "FR"
      },
      "paymentMethod|@ ($PAYMENT_METHOD)|Payment method": "CARD",
      "total|@ (>0)|Total gross amount": 120.00,
      "$requiredIf status('SHIPPED','DELIVERED')": ["trackingNumber"],
      "trackingNumber|{10,50}": "TRACK123456789",
      "$appliedIf paymentMethod": {
        "('CARD')": {
          "cardLastFour|@ {4}|Last 4 digits": "1234"
        },
        "('PAYPAL')": {
          "paypalEmail|@ ~$Email~|PayPal email": "user@example.com"
        },
        "('BANK_TRANSFER')": {
          "bankReference|@ {10,50}|Bank reference": "REF1234567890"
        }
      }
    }
  },
  "$nomenclature": {
    "ORDER_STATUS": "PENDING,CONFIRMED,SHIPPED,DELIVERED,CANCELLED",
    "PAYMENT_METHOD": "CARD,PAYPAL,BANK_TRANSFER"
  },
  "$format": {
    "OrderId": "^ORD-[0-9]{8}$",
    "Sku": "^SKU-[A-Z]{3}[0-9]{5}$"
  }
}
```

### 9.4 Garden Management

```json
{
  "$okylineVersion": "1.4.0",
  "$version": "1.0.5",
  "$title": "Shared Vegetable Garden",
  "$description": "Schema for shared vegetable garden crop monitoring",
  "$nomenclature": {
    "VEGGIES": "Carrot,Tomato,Lettuce,Zucchini,Pepper,Cucumber",
    "METHODS": "Permaculture,Direct_sowing,Greenhouse,Container",
    "WEATHER": "SUNNY,CLOUDY,RAINY,WINDY"
  },
  "$format": {
    "SessionCode": "^[A-Z]{2}-\\d{4}$",
    "IsoDate": "^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\\d|3[01])$"
  },
  "$oky": {
    "gardener|@|Gardener information": {
      "id|@ #(>0)|Gardener identifier": 101,
      "name|@ {2,100}|Gardener name": "Julie Martin",
      "email|{6,100}~$Email~|Contact email": "julie@eco-garden.org",
      "status|@ ('ACTIVE','INACTIVE')|Membership status": "ACTIVE"
    },
    "session|@|Gardening session": {
      "code|@ #~$SessionCode~|Unique session code": "SR-0012",
      "date|@ ~$IsoDate~|Session date": "2025-04-10",
      "participantNames|[1,12] -> {2,50}|Participant names": ["Paul", "Léa", "Marc"],
      "surface|@ (1..10,>20)|Planted area in m²": 25,
      "weather|@ ($WEATHER)|Weather conditions": "SUNNY",
      "notes|?{0,500}|Session notes": "Great session with new participants",
      "plants|@ [1,10] -> !|Plants grown": [
        {
          "type|@ #($VEGGIES)|Vegetable type": "Carrot",
          "method|@ [1,2] -> ($METHODS)!|Growing methods": ["Permaculture"],
          "plantedAt|@ ~$IsoDate~|Planting date": "2025-03-20",
          "quantity|(1..1000)|Number of plants": 50
        },
        {
          "type|@ #($VEGGIES)|Vegetable type": "Tomato",
          "method|@ [1,2] -> ($METHODS)!|Growing methods": ["Greenhouse"],
          "plantedAt|@ ~$IsoDate~|Planting date": "2025-04-01",
          "quantity|(1..1000)|Number of plants": 30
        }
      ]
    }
  }
}
```

---

## 10. Appendix: Quick Reference

### 10.1 Constraint Cheat Sheet
Examples in this table illustrate values in validated JSON instance.

| Constraint | Applies To | Meaning | Example |
|--------|------------|---------|---------|
| `@` | All | Required field | `"name|@": "Alice"` |
| `?` | All | Nullable field | `"middle|?": null` |
| `#` | Scalars | Key field (for uniqueness) | `"id|#": 123` |
| `{...}` | String | Length constraint | `"name|{2,50}": "Alice"` |
| `(...)` | Scalars | Value constraints | `"age|(18..65)": 30` |
| `~...~` | String | Regex/format | `"email|~$Email~": "a@b.c"` |
| `[...]` | Array | Size constraint | `"tags|[1,5]": ["a"]` |
| `->` | Array/Map | Element/value constraints | `"tags|[*] -> {2,10}": ["eco"]` |
| `!` | Array | Uniqueness | `"codes|[*] -> !": ["A","B"]` |
| `[...:...]` | Object (map) | Map constraints | `"data|[*:10]": {...}` |
| `%` | All | Default value (informational) | `"theme|%": "light"` |
| `$oneOf` | Object/Array | Match exactly one | `"pay|$oneOf": [...]` |
| `$anyOf` | Object/Array | Match at least one | `"notif|$anyOf": [...]` |

### 10.2 Built-in Formats

| Format | Description | Example |
|--------|-------------|---------|
| `$Date` | ISO date (YYYY-MM-DD) | `"2025-05-30"` |
| `$DateTime` | ISO 8601 datetime | `"2025-05-30T14:30:00Z"` |
| `$Time` | Time (HH:mm or HH:mm:ss) | `"23:59:00"` |
| `$Email` | Email address | `"user@example.com"` |
| `$Uri` | URI with scheme | `"https://example.com"` |
| `$Ipv4` | IPv4 address | `"192.168.1.1"` |
| `$Ipv6` | IPv6 address | `"2001:db8::1"` |
| `$Uuid` | UUID v1-v5 | `"f47ac10b-58cc-..."` |
| `$Hostname` | DNS hostname | `"api.example.com"` |

### 10.3 Special Keys

| Key | Location | Purpose |
|-----|----------|---------|
| `$oky` | Root | **Required**. Contains the schema definition |
| `$okylineVersion` | Root | Okyline spec version (e.g., `"1.0"`) |
| `$version` | Root | Schema version |
| `$title` | Root | Schema title |
| `$description` | Root | Schema description |
| `$additionalProperties` | Root | Allow unknown fields (default: `false`) |
| `$nullAsAbsentIfUndeclared` | Root | Treat null on non-`?` fields as absent (default: `false`) |
| `$nomenclature` | Root | Reusable value registries |
| `$format` | Root | Reusable regex patterns |
| `//...` | Any block | Comment (attribute and subtree ignored) |

### 10.4 Presence & Conditional Directives

| Directive              | Location | Purpose                                          |
|------------------------|----------|--------------------------------------------------|
| `$requiredIf`          | Object | Conditional required fields (if condition)       |
| `$requiredIfNot`       | Object | Required if condition NOT met                    |
| `$requiredIfExist`     | Object | Required if another field exists                 |
| `$requiredIfNotExist`  | Object | Required if field does NOT exist                 |
| `$forbiddenIf`         | Object | Conditional forbidden fields (if condition)      |
| `$forbiddenIfNot`      | Object | Forbidden if condition NOT met                   |
| `$forbiddenIfExist`    | Object | Forbidden if another field exists                |
| `$forbiddenIfNotExist` | Object | Forbidden if field does NOT exist                |
| `$appliedIf`           | Object | Conditional structure (if/switch condition)      |
| `$appliedIfExist`      | Object | Conditional structure (if field exists)          |
| `$appliedIfNotExist`   | Object | Conditional structure (if field does NOT exists) |
| `$required`            | Object | Unconditional required fields                    |
| `$forbidden`           | Object | Unconditional forbidden fields                   |
| `$atLeastOne`          | Object | At least one field from group must be present    |
| `$mutuallyExclusive`   | Object | At most one field from group may be present      |
| `$exactlyOne`          | Object | Exactly one field from group must be present     |
| `$allOrNone`           | Object | All fields in group or none of them              |

> **Note:** Conditions can use value expressions (ranges, comparisons, discrete values), **Type Guards** (`_String_`, `_Integer_`, `_ListOfObject_`, etc.), **Null Literal** (`null`), or **Path Expressions** (`info.type`, `parent.status`, `root.config.mode`). See §6.3.18 for Type Guards, §6.3.19 for Null Literal, and §6.3.20 for Path Expressions.

### 10.5 Type Inference Table

| Example Value | Inferred Type |
|---------------|---------------|
| `"text"` | String |
| `42` | Integer |
| `3.14` | Number |
| `true` / `false` | Boolean |
| `["a", "b"]` | Array[String] |
| `[1, 2, 3]` | Array[Integer] |
| `{"key": "value"}` | Object |

### 10.6 Common Patterns

**Required email:**
```json
"email|@ ~$Email~": "user@example.com"
```

**Optional phone number with pattern:**
```json
"phone|~^\\+[0-9]{10,15}$~": "+33612345678"
```

**List of unique strings, 1-10 items, 2-20 chars each:**
```json
"tags|@ [1,10] -> {2,20}!": ["eco", "bio"]
```

**Enum from nomenclature:**
```json
{
  "$oky": {
    "status|@ ($STATUS)": "ACTIVE"
  },
  "$nomenclature": {
    "STATUS": "ACTIVE,INACTIVE,PENDING"
  }
}
```

**Age between 18 and 120:**
```json
"age|@ (18..120)": 30
```

**Map with pattern-constrained keys:**
```json
"translations|[~^[a-z]{2}$~:10] -> {1,100}": {
  "en": "Hello",
  "fr": "Bonjour"
}
```

**Polymorphic payment with oneOf:**
```json
"payment|@ $oneOf": [
  {"type|@ ('card')": "card", "number|@ {16}": "1234567812345678"},
  {"type|@ ('paypal')": "paypal", "email|@ ~$Email~": "user@example.com"}
]
```

---

## Document Information

**Specification Version:** 1.4.0
**Date:** April 2026
**Status:** Draft
**License:** CC BY-SA 4.0
**Copyright:** © Akwatype - 2025-2026

**Changelog:**
- **v1.4.0 (2026-04):** Added unconditional directives `$required` and `$forbidden` (§6.3.12, §6.3.13); added structural group directives `$atLeastOne`, `$mutuallyExclusive`, `$exactlyOne`, `$allOrNone` (§6.3.14–§6.3.17); added `$nullAsAbsentIfUndeclared` directive (§7.3.6); added `$notExist` branch in `$appliedIf` switch-case (§6.3.5); `$str` support on list item constraints. Expression Language (Annex C): added list context navigation (`origin`, `prev`, `next`, `first`, `last`) and positional predicates (`isOrigin`, `isFirst`, `isLast`); added membership function `in()`; added aggregation functions `sumIf`, `map`, `filter`; added date functions `dayOfWeek`, `dayOfYear`, `weekOfYear`, `quarter`, `semester`, `before`, `after`, `equals`; added string functions `substringBeforeLast`, `substringAfterLast`, `replaceFirst`, `replaceLast`, `ltrim`, `rtrim`, `removePrefix`, `removeSuffix`, `removeRange`, `indexOfFirst`, `isNull`; short-circuit evaluation for `&&`/`||`; updated EBNF grammar with revised operator precedence
- **v1.2.0 (2026-01):** Split Annex D into D (internal references) and E (external imports/versioning); terminology update (inclusion/composition instead of inheritance); moved computed expressions to Annex C; added Field Path Expressions in conditional directives (§6.3.20); added Null Literal in condition triggers (§6.3.19); added Annex F for Virtual Fields; added Comments syntax (§4.5)
- **v1.1.0 (2025-12):** Draft update (normative clarifications, additionalProperties behavior, compute semantics, and example fixes)
- **v1.0 (2025-11):** Initial specification release

**Contributing:**
Feedback and contributions to this specification are welcome. Please ensure any derivative works are shared under the same CC BY-SA 4.0 license.

**Contact:**
For questions or suggestions regarding this specification, please contact Akwatype at:
pierre-michel.bret@akwatype.io

---

Okyline® and Akwatype® are registered trademarks of Akwatype.

---

*End of Okyline Language Specification v1.4.0*

> **Note:** Annexes A (Conformance) and B (Terminology) will be published in the Final version of the specification.

---

# Annex C - Okyline Expression Language (Normative)

This annex defines the **Okyline Expression Language**, a pure and deterministic language for expressing computed validations. It enables business invariants such as cross-field calculations (e.g., `total == subtotal * (1 + taxRate)`) and conditional logic based on field values. Expressions are declared in a `$compute` block and referenced in field constraints using the `(%ExpressionName)` syntax.

Defined in the external file "Okyline-Annex-C-Expression-language-v1.4.0.md"

---

# Annex D - Internal Schema References (Normative)

This annex defines the **internal reference mechanism** for composing and reusing schema fragments within a single document. It introduces `$defs` for declaring reusable templates, `$ref` for including them, and `$override`/`$remove` for adapting included structures. This enables DRY (Don't Repeat Yourself) schema design while maintaining full control over structural composition.

Defined in the external file "Okyline-Annex-D-Internal-References-v1.4.0.md"

---

# Annex E - External Imports and Versioning (Normative)

This annex extends Annex D to enable **cross-schema composition** in enterprise environments. It defines schema identity (`$id`, `$version`), versioned dependencies (`$deps`), and external imports (`$xDefs`). Organizations can publish schemas to a registry and import definitions from other contracts while maintaining strict version control and compatibility guarantees.

Defined in the external file "Okyline-Annex-E-External-Imports-v1.4.0.md"

---

# Annex F - Virtual Fields (Normative)

This annex defines **virtual fields** (`$field`), a mechanism to declare computed values that exist only during validation and can be used in conditional directives. Virtual fields enable scenarios where conditional rules depend on derived values not present in the validated data, such as computed tiers, classifications, or flags.

Virtual fields are computed from `$compute` expressions, scoped to the declaring object, evaluated sequentially in declaration order, and can form chains where one field depends on another. They can be used as triggers in value-based conditional directives but MUST NOT be used in existence-based directives (`$requiredIfExist`, etc.).

Defined in the external file "Okyline-Annex-F-Virtual-Fields-v1.4.0.md"


*End of Okyline Annex Language Specification v1.4.0*
