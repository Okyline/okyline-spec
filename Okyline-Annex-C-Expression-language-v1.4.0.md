---
description: Okyline Expression Language specification - a pure and deterministic language for computed validations, cross-field calculations, and conditional business logic.
---

# Annex C — Okyline Expression Language (Normative)

**Version:** 1.4.0
**Date:** April 2026
**Status:** Draft

 License: This annex is part of the Open Okyline Language Specification and is subject to the same license terms (CC
  BY-SA 4.0). See the Core Specification for full license details.

This annex defines the **Okyline Expression Language**, a pure and deterministic language for expressing computed validations. It enables business invariants such as cross-field calculations (e.g., `total == subtotal * (1 + taxRate)`) and conditional logic based on field values. Expressions are declared in a `$compute` block and referenced in field constraints using the `(%ExpressionName)` syntax.


---

## Relation to the Core Specification

This annex defines the **Okyline Expression Language**, used inside `$compute` blocks and computed field constraints.

It specifies:

- the **declaration** of computed expressions (`$compute` block)
- the **syntax** for referencing expressions in field constraints (`(%Name)`)
- the **grammar**, **operators**, and **evaluation semantics** of expressions

Implementations claiming Okyline 1.4.0 **Level 3** conformance (computed business invariants) **MUST** implement the rules described in this annex.

The Expression Language is an **integral component** of Okyline, not a separate language. It extends the core validation model with computation and conditional logic while remaining **pure, deterministic, and side-effect-free**.

Although this annex is part of the Okyline 1.4.0 Draft specification, the expression language defined here is considered stable for practical use.

---

### Non-normative note

The Expression Language can be used independently of validation, for example in documentation tools or template generators, but only the behaviors defined in this annex are considered part of the Okyline 1.4.0 normative model.


---

## C.1 Overview

The Okyline Expression Language defines the grammar and semantics used in `$compute` blocks and computed constraints.

This annex is **normative**, meaning its contents define the expected behavior of conforming implementations of Okyline 1.4.0.

Expressions are pure, deterministic, and evaluated within the context of the object where they appear. All functions are **null-safe** and **side-effect free**.

---

## C.2 Computed Expression Block — `$compute`

Okyline reserves the special block `$compute` as a container for named expressions. It is declared at the **root level**, alongside `$oky`.

### C.2.1 Syntax

```json
{
  "$oky": {
    ...
  },
  "$compute": {
    "ExpressionName": "expression body"
  }
}
```

**Example:**

```json
{
  "$oky": {
    "invoice": {
      "subtotal": 100.0,
      "taxRate": 0.2,
      "total|(%ValidTotal)": 120.0
    }
  },
  "$compute": {
    "TaxAmount": "subtotal * taxRate",
    "ValidTotal": "total == subtotal + %TaxAmount"
  }
}
```

### C.2.2 Normative Rules

- `$compute` is declared at the **root level**, not inside `$oky`.
- `$compute` is **optional**; when present, it MUST be a JSON object.
- Keys are expression names; values are expression strings.
- Expression names MUST be valid identifiers:
  - Start with a letter (a-z, A-Z)
  - Contain only letters, digits (0-9), and underscores (_)
- Circular dependencies MUST be detected at schema load time and cause a parsing error.
- Expressions are evaluated lazily when referenced.

---

## C.3 Usage in Field Constraints — `(%Name)`

To validate a field using a computed expression, reference it with the `(%ExpressionName)` syntax in the field's constraint declaration.

### C.3.1 Syntax

```
"fieldName|(%ExpressionName)": exampleValue
```

**Example:**

```json
{
  "$oky": {
    "order": {
      "quantity|(>0)": 5,
      "unitPrice|(>0)": 20.0,
      "total|(%CheckTotal)": 100.0
    }
  },
  "$compute": {
    "CheckTotal": "total == quantity * unitPrice"
  }
}
```

### C.3.2 Validation Semantics

When a field references a computed expression:

- The expression is evaluated in the field's parent object context.
- The result determines validation outcome:

| Result                          | Validation                                                  |
| ------------------------------- | ----------------------------------------------------------- |
| `true`                          | Passes                                                      |
| `false`                         | Fails — `Compute` error                                     |
| Non-boolean (including `null`)  | Fails — `Compute` error ("*expression* has to be boolean")  |
| Evaluation exception            | Fails — `Compute` error (exception message)                 |

Error reporting SHOULD include:

- Field path (e.g., `order.total`)
- Expression name (e.g., `CheckTotal`)
- Failure detail (sub-expression that failed, or type violation)

### C.3.3 Combining with Other Constraints

`(%Name)` can be combined with other constraint types:

```json
"email|@ (%ValidEmail) ~^[^@]+@[^@]+$~|Contact email": "user@example.com"
```

The field `email` must satisfy all constraints:

1. Required (`@`)
2. Pass the `ValidEmail` expression (`(%ValidEmail)`)
3. Match the regex pattern (`~...~`)

**Constraint type exclusivity:**

Since each constraint type can appear only once per field, `(%Name)` **cannot** be combined with other parenthetical constraints like `(>0)` or `(<100)`. Both use the `(...)` syntax and occupy the same constraint slot.

```json
// INVALID - two parenthetical constraints
"total|@ (>0) (%ValidTotal)": 120.0

// VALID - numeric check must be inside the expression
"total|@ (%ValidTotal)": 120.0
```

If you need both a range check and a computed validation, include the range check within the computed expression itself:

```json
"$compute": {
  "ValidTotal": "total > 0 && total == subtotal * (1 + taxRate)"
}
```

**Constraint evaluation order:**

1. Type validation (inferred from example)
2. Standard constraints (`@`, `{...}`, `~...~`)
3. Computed expression validation (`(%Name)`)

### C.3.4 Collection Fields — Container and Element Compute

`(%Name)` can be applied to collection fields (scalar lists, object lists, and maps). When combined with the `->` separator (Core §5.2.2), it distinguishes **container-level** and **element-level** evaluation:

```
"field|@ [size] (%ContainerCheck) -> (%ElementCheck)": [10, 20]
```

- **Before `->`**: evaluated once on the collection as a whole. `it` references the collection.
- **After `->`**: evaluated once per element. `it` references the current element value.

The separation is **strict**: container constraints are never applied to individual elements, and element constraints are never applied to the collection. Either side is optional.

Container errors are reported on the collection path (e.g., `'scores'`); element errors on the element path (e.g., `'scores[0]'`).

**Example:**

```json
"scores|@ [*] (%CheckTotal) -> (%CheckElement)": [10, 20, 35]
```
```json
"CheckTotal": "100 == sum(scores)",
"CheckElement": "it >= 10"
```

---

## C.4 Evaluation Context

Expressions are evaluated in the context of a JSON object. All fields of the current object are accessible by name.

### C.4.1 Object Context

For a field constraint `(%ExpressionName)`, the expression is evaluated in the context of the **parent object** containing that field.

```json
{
  "$oky": {
    "invoice": {
      "subtotal": 100.0,
      "taxRate": 0.2,
      "total|(%ValidTotal)": 120.0
    }
  },
  "$compute": {
    "ValidTotal": "total == subtotal * (1 + taxRate)"
  }
}
```

In this example, `ValidTotal` is evaluated with access to:

- `subtotal` → 100.0
- `taxRate` → 0.2
- `total` → 120.0

**Field resolution rules:**

- Fields are accessed by name directly: `subtotal`, `taxRate`
- Nested field access uses dot notation: `address.city`
- Missing fields resolve to `null`
- Accessing a field on a null value returns `null` (null-safe navigation)

### C.4.2 Collection Context (Aggregations)

When using aggregation functions (`sum`, `average`, `min`, `max`, etc.), the second parameter expression is evaluated **for each element** in the collection.

```json
{
  "$oky": {
    "order": {
      "items|[1,100]": [
        {"qty": 2, "price": 10.0},
        {"qty": 3, "price": 15.0}
      ],
      "total|(%CheckTotal)": 65.0
    }
  },
  "$compute": {
    "LineTotal": "qty * price",
    "CheckTotal": "total == sum(items, %LineTotal)"
  }
}
```

In this example:

- `LineTotal` is evaluated in the context of **each item** (has access to `qty` and `price`)
- `CheckTotal` is evaluated in the context of **order** (has access to `items` and `total`)
- `sum(items, %LineTotal)` iterates over `items`, evaluates `%LineTotal` for each, and returns the sum

### C.4.3 Path Expressions

Expressions support **path expressions** to access fields beyond the immediate validation context.

Path expression syntax and resolution rules are defined in **§6.3.20 Field Path Expressions** of the core specification.

| Prefix | Resolves From |
|--------|---------------|
| *(none)* | Current context |
| `this.` | Current context (explicit) |
| `parent.` | Parent object |
| `root.` | Document root |
| `origin.` | Origin element of the aggregation (lambda only, §C.4.5) |
| `prev.` | Previous iteration element (lambda only, §C.4.5) |
| `next.` | Next iteration element (lambda only, §C.4.5) |
| `first.` | First collection element (lambda only, §C.4.5) |
| `last.` | Last collection element (lambda only, §C.4.5) |

**Example:**

```json
{
  "$compute": {
    "ValidDiscount": "discount <= parent.maxDiscount"
  }
}
```

### C.4.4 The `it` Variable

When an expression is evaluated as a **field constraint** (`(%Name)`), the variable `it` references the value of the field being validated.

**Example:**

```json
{
  "$compute": {
    "IsPositive": "it > 0"
  },
  "$oky": {
    "quantity|(%IsPositive)": 10,
    "price|(%IsPositive)": 99.99
  }
}
```

**Scope:** `it` is defined in the constraint expression and propagates to any referenced expression (`%Name`). In contexts where `it` is not defined (aggregations, standalone evaluations), referencing `it` resolves to `null`.

### C.4.5 List Iteration Context

When an expression is evaluated inside the lambda of an aggregation function (`countIf`, `exists`, `notExists`, `filter`, `sum`, etc.), the following context variables are available in addition to the implicit current element:

| Variable | Resolves to | Null when |
|----------|-------------|-----------|
| `origin` | The item whose validation triggered the aggregation call | Never (always the calling item) |
| `prev` | The element immediately before the current iteration element | Current element is first |
| `next` | The element immediately after the current iteration element | Current element is last |
| `first` | The first element of the iterated collection | Collection is empty |
| `last` | The last element of the iterated collection | Collection is empty |

All variables support dotted navigation: `origin.amount`, `prev.date`, `first.id`, `last.status`.

These variables are **only defined** inside aggregation lambdas. Outside this context, they resolve to `null`.

**Positional predicates:**

In addition to the navigation variables above, the following predicates are available inside aggregation lambdas:

| Predicate | Returns `true` when |
|-----------|---------------------|
| `isOrigin` | Current iteration element is the origin element |
| `isFirst` | Current iteration element is the first in the collection |
| `isLast` | Current iteration element is the last in the collection |

These predicates are **only defined** inside aggregation lambdas. Outside this context, they resolve to `null`.

**Implicit element access vs. `origin`:**

Inside an aggregation lambda, unqualified field names (`amount`, `status`) refer to the **current iteration element**. `origin` refers to the **element whose field constraint triggered the aggregation**. These are distinct objects.

**Example:**

```json
{
  "$compute": {
    "IsUnique": "countIf(parent.items, id == origin.id) == 1"
  },
  "$oky": {
    "items|[*]": [{
      "id|@ (%IsUnique)": "A001"
    }]
  }
}
```

When validating `items[0].id`, the expression iterates over all items. For each iteration element, `id` is the element's id and `origin.id` is `items[0].id`. The count equals 1 only if the id is unique.

---

## C.5 Grammar

### C.5.1 Simplified EBNF

```
expression       ::= conditional
conditional      ::= logical_or ( "?" expression ":" expression )?
logical_or       ::= logical_and ( "||" logical_and )*
logical_and      ::= equality ( "&&" equality )*
equality         ::= comparison ( ("==" | "!=" | "===" | "!==") comparison )*
comparison       ::= addition ( (">" | "<" | ">=" | "<=") addition )*
addition         ::= multiplication ( ("+" | "-") multiplication )*
multiplication   ::= null_coalescing ( ("*" | "/") null_coalescing )*
null_coalescing  ::= unary ( "??" unary )*
unary            ::= ("!" | "-") unary | primary
primary          ::= literal | identifier | compute_ref | function_call | "(" expression ")"
compute_ref      ::= "%" identifier
function_call    ::= identifier "(" [ arguments ] ")"
arguments        ::= expression ( "," expression )*
literal          ::= number | string | boolean | null
identifier       ::= letter ( letter | digit | "_" )*
```

---

## C.6 Operators

| Operator | Type | Description | Example | Null Behavior |
|----------|------|-------------|---------|---------------|
| `??` | Null coalescing | Returns left if non-null, otherwise right | `price ?? 0` → 0 | Short-circuits; high precedence |
| `+` | Arithmetic | Addition | `2 + 3` → 5 | Null propagates — Exception: string concatenation treats null as `""` |
| `-` | Arithmetic | Subtraction | `5 - 2` → 3 | Null propagates |
| `*` | Arithmetic | Multiplication | `3 * 2` → 6 | Null propagates |
| `/` | Arithmetic | Division | `6 / 2` → 3.0 | Null propagates; division by zero → null |
| `>` `<` `>=` `<=` | Comparison | Numeric/lexicographic | `age > 18` | Returns null if either operand is null |
| `==` `!=` | Equality | Value equality | `"A" == "A"` | `null == null` → true; `null == x` → false |
| `===` `!==` | Strict equality | IEEE-754 bit-exact | `1.0 === 1.0` | Strict comparison (NaN !== NaN) |
| `&&` | Logical | Conjunction | `a && b` | null → false |
| `||` | Logical | Disjunction | `a || b` | null → false |
| `!` | Logical | Negation | `!true` → false | `!null` → true |
| `? :` | Ternary | Conditional | `x > 10 ? "hi" : "lo"` | Condition null → false |

In case of any discrepancy, the operator precedence defined in the grammar (§C.5) **MUST** take precedence.

### C.6.1 Null Coalescing Operator

| Operator | Type | Description | Example |
|----------|------|-------------|---------|
| `??` | Null coalescing | Returns the left operand if it is non-null; otherwise returns the right operand (short-circuit evaluation). | `price ?? 0` → 0 (if price is null) |

**Precedence:**
The ?? operator has high precedence, binding tighter than both comparison operators and arithmetic operators.

**Short-circuit evaluation:**
The right operand is **not evaluated** if the left operand is non-null.

**Examples:**

```js
// If price is null, use default of 10
finalPrice = price ?? 10

// Chain multiple coalescings
value = a ?? b ?? c ?? 0  // First non-null value or 0

// In arithmetic context
total = (price ?? 0) * (quantity ?? 1)
```

---

## C.7 Date Functions

| Function | Description | Example |
|----------|-------------|---------|
| `date(dateString, pattern?)` | Parses a string into a LocalDate (default pattern `yyyy-MM-dd`). | `date("2024-03-15")` → LocalDate(2024-03-15) |
| `formatDate(date, pattern?)` | Formats a LocalDate to string (`yyyy-MM-dd` default). | `formatDate(date("2024-03-15"), "dd/MM/yy")` → `"15/03/24"` |
| `today()` | Returns the current system date. | `today()` → 2025-11-05 |
| `daysBetween(start, end)` | Number of days difference (`end - start`). | `daysBetween("2024-03-15","2024-03-18")` → 3 |
| `plusDays(date, days)` | Adds days to a date. | `plusDays("2024-02-28",1)` → `"2024-02-29"` |
| `minusDays(date, days)` | Subtracts days. | `minusDays("2024-03-01",1)` → `"2024-02-29"` |
| `plusMonths(date, months)` | Adds months. | `plusMonths("2024-01-31",1)` → `"2024-02-29"` |
| `minusMonths(date, months)` | Subtracts months. | `minusMonths("2024-03-31",1)` → `"2024-02-29"` |
| `plusYears(date, years)` | Adds years. | `plusYears("2023-03-15",1)` → `"2024-03-15"` |
| `minusYears(date, years)` | Subtracts years. | `minusYears("2024-03-15",1)` → `"2023-03-15"` |
| `isWeekend(date)` | True if Saturday or Sunday. | `isWeekend("2024-03-16")` → true |
| `isLeapYear(date)` | True if leap year. | `isLeapYear("2024-03-15")` → true |
| `year(date)` | Extracts year. | `year("2024-03-15")` → 2024 |
| `month(date)` | Extracts month. | `month("2024-03-15")` → 3 |
| `day(date)` | Extracts day of month. | `day("2024-03-15")` → 15 |
| `dayOfWeek(date)` | Day of week (MON=1, SUN=7). | `dayOfWeek("2024-03-15")` → 5 |
| `dayOfYear(date)` | Day of year (1–366). | `dayOfYear("2024-03-15")` → 75 |
| `weekOfYear(date)` | ISO week number (1–53). | `weekOfYear("2024-01-01")` → 1 |
| `quarter(date)` | Quarter (1–4). | `quarter("2024-09-15")` → 3 |
| `semester(date)` | Semester (1–2). | `semester("2024-09-15")` → 2 |
| `before(date1, date2)` | True if `date1` is before `date2`. | `before("2024-01-01","2024-12-31")` → true |
| `after(date1, date2)` | True if `date1` is after `date2`. | `after("2024-12-31","2024-01-01")` → true |
| `equals(date1, date2)` | True if same date. | `equals("2024-03-15","2024-03-15")` → true |

---

## C.8 String Functions

| Function | Description | Example |
|----------|-------------|---------|
| `isNull(v)` | True if `v` is `null`. | `isNull(null)` → true |
| `isNullOrEmpty(s)` | True if `s` is `null` or empty. | `isNullOrEmpty("")` → true |
| `isEmpty(s)` | True if string length is 0. | `isEmpty("")` → true |
| `substring(s,start,len)` | Returns substring from start. | `substring("Hello",1,3)` → `"ell"` |
| `substringBefore(s,delim)` | Part before first occurrence. | `substringBefore("a:b:c",":")` → `"a"` |
| `substringAfter(s,delim)` | Part after first occurrence. | `substringAfter("a:b:c",":")` → `"b:c"` |
| `substringBeforeLast(s,delim)` | Part before last occurrence. | `substringBeforeLast("a:b:c",":")` → `"a:b"` |
| `substringAfterLast(s,delim)` | Part after last occurrence. | `substringAfterLast("a.b.txt",".")` → `"txt"` |
| `replace(s,target,repl)` | Replace all occurrences. | `replace("foo bar foo","foo","baz")` → `"baz bar baz"` |
| `replaceFirst(s,target,repl)` | Replace first occurrence. | `replaceFirst("foo bar foo","foo","baz")` → `"baz bar foo"` |
| `replaceLast(s,target,repl)` | Replace last occurrence. | `replaceLast("foo bar foo","foo","baz")` → `"foo bar baz"` |
| `trim(s)` | Removes leading/trailing spaces. | `trim("  hi  ")` → `"hi"` |
| `ltrim(s)` | Removes leading spaces. | `ltrim("  hi  ")` → `"hi  "` |
| `rtrim(s)` | Removes trailing spaces. | `rtrim("  hi  ")` → `"  hi"` |
| `length(s)` | String length (null → 0). | `length("hey")` → 3 |
| `startsWith(s,prefix)` | Checks prefix. | `startsWith("hello","he")` → true |
| `endsWith(s,suffix)` | Checks suffix. | `endsWith("hello","lo")` → true |
| `contains(s,search)` | True if substring found. | `contains("banana","an")` → true |
| `removePrefix(s,prefix)` | Removes prefix if present. | `removePrefix("Hello","He")` → `"llo"` |
| `removeSuffix(s,suffix)` | Removes suffix if present. | `removeSuffix("Hello","lo")` → `"Hel"` |
| `removeRange(s,start,len)` | Removes `len` characters from `start`. | `removeRange("Hello",1,3)` → `"Ho"` |
| `toUpperCase(s)` | Converts to upper case. | `toUpperCase("Hi")` → `"HI"` |
| `toLowerCase(s)` | Converts to lower case. | `toLowerCase("Hi")` → `"hi"` |
| `capitalize(s)` | Uppercases first character. | `capitalize("hello")` → `"Hello"` |
| `decapitalize(s)` | Lowercases first character. | `decapitalize("Hello")` → `"hello"` |
| `padStart(s,len,ch)` | Left pads string. | `padStart("7",3,"0")` → `"007"` |
| `padEnd(s,len,ch)` | Right pads string. | `padEnd("7",3,"0")` → `"700"` |
| `repeat(times,ch)` | Repeats char. | `repeat(5,"*")` → `"*****"` |
| `indexOf(s,sub)` | First index of substring. | `indexOf("abracadabra","bra")` → 1 |
| `indexOfFirst(s,sub)` | Alias for `indexOf`. | `indexOfFirst("abracadabra","bra")` → 1 |
| `indexOfLast(s,sub)` | Last index of substring. | `indexOfLast("abracadabra","bra")` → 8 |

### C.8.1 String Index Handling

String functions use **defensive index handling** to prevent runtime errors and ensure deterministic behavior.

#### Rules

- Negative start indices are **clamped to 0**.
- Negative lengths are treated as **0** (the result is an empty string).
- A start index greater than the string length returns **an empty string**.
- End indices beyond the string length are **clamped to length()**.
- Functions **never throw exceptions** for out-of-bounds access.
- When applicable, functions return safe default values (e.g., empty string `""`, index `-1`, etc.).

#### Examples

```js
substring("Hello", -10, 3)   → "Hel"    // negative start clamped to 0
substring("Hello", 1, -5)    → ""       // negative length treated as 0
substring("Hello", 100, 5)   → ""       // start beyond length → empty
substring("Hello", 1, 999)   → "ello"   // end clamped to string length
substring("Hello", 2, 0)     → ""       // zero length → empty
indexOf("abc", "z")          → -1       // not found
padStart("Hi", 1, "x")       → "Hi"     // already long enough (no truncation)
```

#### Notes

- The third parameter of `substring(s, start, length)` is interpreted as a **length**, not an end index.
- Negative lengths are **not** treated as "counting from the end" — they are simply clamped to zero.
- This ensures that all string operations remain **null-safe**, **exception-free**, and **consistent** across implementations.

---

## C.9 Numeric Functions

| Function | Description | Example |
|----------|-------------|---------|
| `abs(x)` | Absolute value. | `abs(-5)` → 5 |
| `sqrt(x)` | Square root. | `sqrt(9)` → 3.0 |
| `floor(x,scale?)` | Rounds down (optional decimals). | `floor(3.1415,2)` → 3.14 |
| `ceil(x,scale?)` | Rounds up. | `ceil(3.1415,2)` → 3.15 |
| `round(x,scale?,mode?)` | Round with mode (`HALF_UP` default). | `round(3.5,0,"HALF_EVEN")` → 4.0 |
| `mod(a,b)` | Remainder of division. | `mod(10,3)` → 1 |
| `pow(base,exp)` | Power function. | `pow(2,3)` → 8.0 |
| `log(x)` | Natural logarithm. | `log(2.71828)` → 1.0 |
| `log10(x)` | Base-10 logarithm. | `log10(1000)` → 3.0 |
| `toInt(v)` | Converts to integer. | `toInt(3.7)` → 4 |
| `toNum(v)` | Converts to number. | `toNum("42")` → 42.0 |
| `toStr(v)` | Converts to string. | `toStr(42)` → `"42"` |

---

## C.10 Aggregation & Utility Functions

Aggregation functions operate on **collections** (arrays or maps). When the collection contains **objects**, a second parameter specifies the expression to evaluate on each element. When the collection contains **scalar values**, the aggregation operates directly on the element values and no second parameter is required.

| Function | Object collection | Scalar collection | Description |
|----------|-------------------|-------------------|-------------|
| `sum` | `sum(collection, expr)` | `sum(collection)` | Sum of values. |
| `average` | `average(collection, expr)` | `average(collection)` | Arithmetic mean of values. |
| `min` | `min(collection, expr)` | `min(collection)` | Minimum value. |
| `max` | `max(collection, expr)` | `max(collection)` | Maximum value. |
| `count` | `count(collection)` | `count(collection)` | Count of non-null elements. |
| `countAll` | `countAll(collection)` | `countAll(collection)` | Count of all elements (including null). |
| `countIf` | `countIf(collection, expr)` | — | Count of elements where expression is true. |
| `exists` | `exists(collection, expr)` | — | True if at least one element satisfies the expression. |
| `notExists` | `notExists(collection, expr)` | — | True if no element satisfies the expression. |
| `sumIf` | `sumIf(collection, predicate, expr)` | — | Sum of `expr` for elements where `predicate` is true. |
| `map` | `map(collection, expr)` | — | Returns a list of `expr` evaluated for each element. |
| `filter` | `filter(collection, expr)` | — | Returns elements where `expr` is true. |

The first argument can be a JSON **array** or a JSON **object used as a map** (`[*:*]`). For maps, values are iterated; keys are ignored for aggregation purposes.

**Note:** The `%identifier` syntax is not a function but a **reference operator** for accessing compute expressions. See [C.10.1 Compute Reference Syntax](#c101-compute-reference-syntax) for details.

### C.10.1 Membership Function — `in`

The `in` function tests whether a value belongs to a set of allowed values.

**Forms:**

| Form | Description | Example |
|------|-------------|---------|
| `in(value, 'A', 'B', 'C')` | Inline literal list | `in(status, 'DRAFT', 'SENT')` |
| `in(value, '$NomenclatureName')` | Nomenclature lookup | `in(status, '$INVOICE_STATUS')` |
| `in(value, listField)` | JSON array field | `in(code, allowedCodes)` |

**Semantics:**

- Returns `true` if the value is found in the target set, `false` otherwise.
- `null` value → `false`.
- When the first argument is itself a list, every element of that list must be present in the target (containsAll semantics). An empty list → `true`.
- Comparison uses the same equality rules as `==` (numeric precision handling, cross-type string comparison).

### C.10.2 Compute Reference Syntax

#### Referencing Compute Expressions: `%identifier`

The `%identifier` syntax allows expressions to reference other computed expressions defined in the same `$compute` block.

**Syntax:**

```
%<identifier>
```

Where `<identifier>` is the name of a compute expression defined in the same `$compute` block.

**Semantics:**

- `%ComputeName` evaluates the expression named `ComputeName` from the current `$compute` block
- The `%` prefix distinguishes compute references from field accesses
- Circular dependencies are detected at schema load time and result in a validation error
- References are resolved in the context where the compute expression is evaluated

**Examples:**

```json
{
  "$compute": {
    "BasePrice": "1000",
    "Tax": "%BasePrice * 0.2",
    "Total": "%BasePrice + %Tax"
  }
}
```

In this example:

- `Tax` references `BasePrice` using `%BasePrice`
- `Total` references both `BasePrice` and `Tax` using `%BasePrice` and `%Tax`
- All references are resolved when the expressions are evaluated

**Aggregation Functions:**

When using aggregation functions (such as `sum`, `map`, `filter`), the second parameter can be:

- A **field reference** (direct access to a field in each collection element)
- An **inline expression** (evaluated for each element)
- A **compute reference** (using `%identifier` to reference a compute expression)

```json
{
  "$compute": {
    "LineTotal": "netAmount * (1 + vat)",
    "SubTotal": "sum(items, netAmount)",
    "TotalWithFees": "sum(items, netAmount * 1.2)",
    "InvoiceTotal": "sum(lines, %LineTotal)"
  }
}
```

**Benefits:**

- Clear visual distinction between field access (`fieldName`) and compute references (`%ComputeName`)
- No ambiguity in aggregation functions
- Consistent with validation constraint syntax (`field|(%ComputeName)`)
- Readable and concise expressions
- Reduces visual noise in complex compute chains

**Constraints:**

- `%` must be followed immediately by a valid identifier
- The referenced identifier must exist in the current `$compute` block
- Circular references result in a validation error at schema load time
- Compute references are resolved before evaluation begins

**Implementation Note:**

The parser transforms `%identifier` into an internal representation at parse time. The exact internal mechanism is implementation-defined, but the observable behavior must match the semantics described above.

---

## C.11 Evaluation Rules

- Expressions are evaluated in the context of the enclosing object (fields accessible by name).
- Missing fields resolve to `null`.
- Type coercion **MUST NOT** be implicit except where explicitly defined by helper functions (`toNum`, `toStr`, etc.).
- Division by zero returns `null`.
- Comparisons with `null` evaluate to `false` unless explicitly tested.
- Implementations **MUST** guarantee deterministic results for the same input.
- Functions **MUST NOT** have side effects.
- Execution errors **SHOULD** return a structured error (`COMPUTE_ERROR`, `INVALID_ARGUMENT`, etc.).

---

### C.11.1 Numeric Precision

The Okyline Expression Language guarantees **exact decimal arithmetic** suitable for financial calculations.

**Guaranteed behaviors:**

- Decimal operations are **exact** within 6 decimal places
- No IEEE 754 floating-point artifacts: `0.1 + 0.2 = 0.3` (exactly)
- Financial calculations produce predictable, auditable results
- Chained operations preserve precision: `(0.1 + 0.2) * 3 = 0.9` (exactly)

**Examples:**
```js
0.1 + 0.2           → 0.3       // exact, not 0.30000000000000004
19.99 * 3           → 59.97     // exact
100.00 / 3          → 33.333333 // rounded to 6 decimals
1000 * 0.196        → 196       // exact
```

**Rationale:**

Financial and business applications require deterministic decimal arithmetic. Implementations **MUST NOT** rely on binary floating-point (IEEE 754) for intermediate calculations, as this produces rounding artifacts unacceptable in financial contexts.

**Manual Rounding Control:**

When finer control over rounding is required, use the `round(x, scale?, mode?)` function to explicitly specify the number of decimal places and rounding mode.

```js
// Default: 6 decimals, HALF_UP
total * vatRate                        → 196.000000

// Explicit rounding to 2 decimals for display
round(total * vatRate, 2)              → 196.00

// Banker's rounding (HALF_EVEN) for financial compliance
round(amount, 2, "HALF_EVEN")          → 2.50

// Round down for conservative estimates
round(estimate, 0, "DOWN")             → 99
```

Available rounding modes: `HALF_UP` (default), `HALF_DOWN`, `HALF_EVEN`, `UP`, `DOWN`, `CEILING`, `FLOOR`.

---

### C.11.2 Type Promotion

Arithmetic operations follow standard type promotion rules, preserving the most precise type from the operands.

**Rules:**

| Operation | Result Type |
|-----------|-------------|
| `int + int` | `int` |
| `int + decimal` | `decimal` |
| `decimal + decimal` | `decimal` |
| `int * int` | `int` |
| `int * decimal` | `decimal` |
| `int / int` | `decimal` (division always returns decimal) |

**Examples:**
```js
5 + 7           → 12       // int + int → int
5 + 7.0         → 12.0     // int + decimal → decimal
5.0 + 7.0       → 12.0     // decimal + decimal → decimal
6 / 2           → 3.0      // division always returns decimal
6 * 2           → 12       // int * int → int
6 * 2.0         → 12.0     // int * decimal → decimal
```

**Rationale:**

This behavior mirrors standard programming language conventions (Java, Python, SQL) and ensures predictable results. Integer operations remain integers when possible, preserving the semantic meaning of whole numbers.

---

## C.12 Null Handling Rules

### C.12.1 Arithmetic Operations (Null Propagation)

Arithmetic operations (`+`, `-`, `*`, `/`) follow **three-valued logic** (similar to SQL semantics):

- If any operand is null, the result is null (null propagates).
- **Exception:** String concatenation with `+` treats null as the empty string `""`.

**Examples:**

```js
10 + null     → null
null * 5      → null
null / 2      → null
100 - null    → null

// String concatenation exception
"Hello" + null     → "Hello"
null + " World"    → " World"
```

### C.12.2 Comparison Operations

Comparison operators (`<`, `<=`, `>`, `>=`) return **null** if either operand is null.

```js
10 > null      → null
null <= 5      → null
null > null    → null
```

### C.12.3 Equality Operations

Equality operators have specific null handling:

- `null == null` → `true`
- `null != null` → `false`
- `null == <any-value>` → `false`
- `null != <any-value>` → `true`

**Note:**
For numeric equality (`==`), values are compared with **implicit rounding to 6 decimal places** using **HALF_UP** rounding mode.

### C.12.4 Logical Operations

Boolean operations require boolean operands:

- `null` is treated as `false` in boolean contexts.
- Non-boolean values are also treated as `false`.

```js
null && true   → false
null || true   → true
!null          → true
```

### C.12.5 Function Arguments

Functions handle null arguments according to their specific semantics:

- **String functions:** Most treat null as empty string `""`.
- **Numeric functions:** Most propagate null (return null if input is null).
- **Date functions:** Null dates typically return null.
- See each function's documentation for specific behavior.

### C.12.6 Field Resolution

- Missing fields resolve to null.
- Accessing a field on a null value returns null (null-safe navigation).

```js
user.address.city  → null  // if user or address is null
```

---

### Rationale for Null Propagation

The null propagation semantics for arithmetic operations follow **SQL/database conventions**, where operations involving unknown values (null) produce unknown results (null).
This prevents silent calculation errors and makes null handling **explicit** via the `??` operator.

---

## C.13 Summary

### Declaration and Usage

| Feature | Syntax | Description |
|---------|--------|-------------|
| Declaration | `"$compute": {...}` | Root-level block for named expressions |
| Field constraint | `field\|(%Name)` | Validate field using expression |
| Compute reference | `%Name` | Reference another expression in `$compute` |
| Field access | `fieldName` | Access field in current context |
| Nested access | `obj.field` | Null-safe nested field access |
| Parent access | `parent.field` | Access field in parent object (§6.3.20) |
| Root access | `root.field` | Access field at document root (§6.3.20) |
| Field value | `it` | Value of field being validated (§C.4.4) |
| Origin element | `origin` | Element whose validation triggered the aggregation (§C.4.5) |
| Iteration navigation | `prev`, `next`, `first`, `last` | Positional access within aggregation lambda (§C.4.5) |
| Positional predicates | `isOrigin`, `isFirst`, `isLast` | Boolean predicates within aggregation lambda (§C.4.5) |
| Null coalescing | `a ?? b` | Return `b` if `a` is null |

### Validation Results

| Expression Result | Validation |
|-------------------|------------|
| `true` | Pass |
| `false` | Fail — `COMPUTE_VALIDATION_FAILED` |
| `null` | Fail — `COMPUTE_TYPE_ERROR` |
| Non-boolean | Fail — `COMPUTE_TYPE_ERROR` |

### Operator Categories

| Category | Operators |
|----------|-----------|
| Arithmetic | `+`, `-`, `*`, `/` |
| Comparison | `>`, `<`, `>=`, `<=` |
| Equality | `==`, `!=`, `===`, `!==` |
| Logical | `&&`, `\|\|`, `!` |
| Null coalescing | `??` |
| Ternary | `? :` |

### Function Categories

| Category | Functions |
|----------|-----------|
| Date | `date`, `today`, `daysBetween`, `plusDays`, `minusDays`, `plusMonths`, `minusMonths`, `plusYears`, `minusYears`, `formatDate`, `isWeekend`, `isLeapYear`, `year`, `month`, `day`, `dayOfWeek`, `dayOfYear`, `weekOfYear`, `quarter`, `semester`, `before`, `after`, `equals` |
| String | `length`, `substring`, `substringBefore`, `substringAfter`, `substringBeforeLast`, `substringAfterLast`, `replace`, `replaceFirst`, `replaceLast`, `trim`, `ltrim`, `rtrim`, `startsWith`, `endsWith`, `contains`, `removePrefix`, `removeSuffix`, `removeRange`, `toUpperCase`, `toLowerCase`, `capitalize`, `decapitalize`, `padStart`, `padEnd`, `repeat`, `indexOf`, `indexOfFirst`, `indexOfLast`, `isEmpty`, `isNullOrEmpty`, `isNull` |
| Numeric | `abs`, `sqrt`, `floor`, `ceil`, `round`, `mod`, `pow`, `log`, `log10`, `toInt`, `toNum`, `toStr` |
| Aggregation | `sum`, `average`, `min`, `max`, `count`, `countAll`, `countIf`, `sumIf`, `exists`, `notExists`, `map`, `filter` |
| Membership | `in` |

### Null Propagation

| Context | Behavior |
|---------|----------|
| Arithmetic (`+`, `-`, `*`, `/`) | Null propagates |
| String concatenation (`+`) | Null treated as `""` |
| Comparisons (`<`, `>`, etc.) | Returns null |
| Equality (`==`) | `null == null` → true |
| Logical (`&&`, `\|\|`) | Null treated as false |
| Division by zero | Returns null |

---

## C.14 Complete Example

```json
{
  "$okylineVersion": "1.4.0",
  "$version": "1.0.0",
  "$title": "Invoice Schema with Computed Validation",

  "$oky": {
    "invoice": {
      "invoiceId|@ ~$InvoiceId~": "INV-2025-0001",
      "issueDate|@ ~$Date~": "2025-06-15",
      "dueDate|@ ~$Date~ (%DueDateAfterIssue)": "2025-07-15",

      "customer": {
        "name|@ {2,100}": "Acme Corporation",
        "email|@ ~$Email~": "billing@acme.com"
      },

      "lines|@ [1,100]": [
        {
          "description|@ {2,200}": "Consulting services",
          "quantity|@ (1..1000)": 10,
          "unitPrice|@ (>0)": 150.00,
          "lineTotal|@ (%CheckLineTotal)": 1500.00
        },
        {
          "description|@ {2,200}": "Travel expenses",
          "quantity|@ (1..1000)": 1,
          "unitPrice|@ (>0)": 350.00,
          "lineTotal|@ (%CheckLineTotal)": 350.00
        }
      ],

      "subtotal|@ (%CheckSubtotal)": 1850.00,
      "taxRate|@ (0..0.5)": 0.20,
      "taxAmount|@ (%CheckTaxAmount)": 370.00,
      "total|@ (%CheckTotal)": 2220.00,

      "amountPaid|(>=0)": 0.00,
      "balance|(%CheckBalance)": 2220.00,

      "status|@ ('DRAFT','SENT','PAID','OVERDUE')": "DRAFT",

      "$requiredIf status('PAID')": ["paymentDate"],
      "paymentDate|~$Date~": "2025-07-10"
    }
  },

  "$compute": {
    "CheckLineTotal": "lineTotal == round(quantity * unitPrice, 2)",
    "CheckSubtotal": "subtotal == sum(lines, lineTotal)",
    "CheckTaxAmount": "taxAmount == round(subtotal * taxRate, 2)",
    "CheckTotal": "total == subtotal + taxAmount",
    "CheckBalance": "balance == total - (amountPaid ?? 0)",
    "DueDateAfterIssue": "daysBetween(issueDate, dueDate) >= 0"
  },

  "$format": {
    "InvoiceId": "^INV-[0-9]{4}-[0-9]{4}$"
  },

  "$nomenclature": {
    "INVOICE_STATUS": "DRAFT,SENT,PAID,OVERDUE,CANCELLED"
  }
}
```

### Explanation

1. **Line-level validation:** Each `lineTotal` must equal `quantity × unitPrice` (rounded to 2 decimals)
2. **Aggregation:** `subtotal` must equal the sum of all `lineTotal` values
3. **Tax calculation:** `taxAmount` validated against `subtotal × taxRate`
4. **Total validation:** `total` must equal `subtotal + taxAmount`
5. **Balance with null safety:** Uses `??` to default `amountPaid` to 0 if null
6. **Date validation:** `dueDate` must be on or after `issueDate`
7. **Conditional requirement:** `paymentDate` is required only when `status` is `'PAID'`

### Valid Instance

```json
{
  "invoice": {
    "invoiceId": "INV-2025-0042",
    "issueDate": "2025-06-15",
    "dueDate": "2025-07-15",
    "customer": {
      "name": "Tech Solutions Ltd",
      "email": "accounts@techsolutions.com"
    },
    "lines": [
      {
        "description": "Software development",
        "quantity": 40,
        "unitPrice": 125.00,
        "lineTotal": 5000.00
      }
    ],
    "subtotal": 5000.00,
    "taxRate": 0.20,
    "taxAmount": 1000.00,
    "total": 6000.00,
    "amountPaid": 3000.00,
    "balance": 3000.00,
    "status": "SENT"
  }
}
```

---

**End of Annex C — Okyline Expression Language (Normative)**
