---
description: Internal schema references in Okyline - mechanism for composing and reusing schema fragments with $defs, $ref, $override, and $remove directives.
---

# Annex D - Internal Schema References (Normative)

**Version:** 1.7.0
**Date:** May 2026
**Status:** Draft

 License: This annex is part of the Open Okyline Language Specification and is subject to the same license terms (CC
  BY-SA 4.0). See the Core Specification for full license details.


This annex defines the **internal reference mechanism** for composing and reusing schema fragments within a single document. It introduces `$defs` for declaring reusable templates, `$ref` for including them, and `$override`/`$remove` for adapting included structures. This enables DRY (Don't Repeat Yourself) schema design while maintaining full control over structural composition.

---

## Relation to the Core Specification

This annex defines the semantics of:

- internal reusable definitions (`$defs`)
- schema references (`$ref`)
- the `$override` / `$remove` mechanisms

It specifies how Okyline schemas can be composed and extended using internal definitions, promoting reuse and consistency within a single document.

For versioned imports and external schema references, see **Annex E - External Imports and Versioning**.

---

## D.1 Overview

Okyline supports **schema references** to promote reuse, modularity, and consistency within a schema document.

References allow a field or an object definition to reuse an existing schema fragment identified by a logical name stored in `$defs`.

Okyline distinguishes two complementary use cases:

1. **Property-level reference** - a single field whose value of the form `&Name` (or `["&Name"]` for a list) qualifies the field as having the type of the referenced definition.
2. **Object-level reference** - an object that includes all fields of another definition and may add, override or remove fields. Declared via the special key `$ref` inside the object. Object-level reference targets **exactly one** template.

In Okyline, `$ref` is an **inclusion mechanism**: a referenced schema is treated as a base that can be extended, explicitly overridden or partially removed. This differs intentionally from JSON Schema, where `$ref` behaves as a total substitution.

---

## D.2 Definition Repository - `$defs`

Okyline reserves the special block `$defs` as a container for reusable schema fragments. It is declared at the **root level**, alongside `$oky`.

### D.2.1 Syntax

```json
{
  "$oky": {
    "person": {
      "$ref": "&Address",
      "name|@ {2,50}": "Dupond"
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

### D.2.2 Normative Rules

- `$defs` is declared at the **root level**, not inside `$oky`.
- `$defs` is **optional** but, when present, MUST contain a map of named schemas.
- Entries under `$defs` are **not** interpreted as JSON properties of validated instances; they are reusable definitions only.
- Any schema inside `$defs` MAY be targeted by `$ref` using its name.
- References are limited to the **first level** of `$defs`. Nested paths within `$defs` entries are not addressable.

### D.2.3 Scalar Definitions

`$defs` supports both object schemas and scalar type definitions:

```json
{
  "$oky": {
    "user": {
      "email | @": "&Email",
      "score": "&Percentage"
    }
  },
  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com",
    "Percentage|(0..100)": 50,
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

Scalar definitions follow standard Okyline syntax: the key contains the name and constraints, the value is the example.

### D.2.4 Collection Definitions

The key of a `$defs` entry accepts the **same constraint grammar as a field key** (Core §4 Field Syntax, §5 Constraint Reference), with the exception of `@` (presence) and `#` (key-field role) which are usage/role concerns that belong at the use site, not at the type.

A `$defs` entry can therefore name a collection (list or map, including its leaf constraints via `->`) and be chained through `&Name` references, enabling map-of-map, list-of-list, list-of-map, and map-of-list at any depth.

```json
{
  "$oky": {
    "kits|@ [*]": ["&Kit"]
  },
  "$defs": {
    "Kit|[*:*]":            { "k": "&Variant" },
    "Variant|[*] -> (>=0)": [0]
  }
}
```

`kits` is `List<Map<String, List<int>>>`:
- `Kit` is a map whose values are `Variant`
- `Variant` is a list of non-negative integers

---

## D.3 Reference Syntax

Within a document, definitions are referenced via:

```
&Name
```

Where `Name` is defined in `$defs`.

**Examples:**

```
&Address
&Email
&Person
```

**Rules:**

- `&` denotes the current document's definition namespace.
- The name following `&` refers to an entry in `$defs`.
- Resolution is **case-sensitive**.
- If `Name` does not exist in `$defs`, the schema MUST be rejected.

> **Note:** For external references to definitions in other schemas, see Annex E.

---

## D.4 Property-Level References

### D.4.1 Syntax

A field uses a definition as its **type** when its value is a reference of the form `&Name`:

```json
{
  "$oky": {
    "person": {
      "address": "&Address",
      "name|@ {2,50}": "Dupond"
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

### D.4.2 Semantics

- The part before the pipe (`address`) is the JSON property name.
- A field value of the form `&Name` indicates that the **type and value constraints** of the field are taken from the target definition.
- The field behaves as if the referenced schema had been written inline at this location.

A property-level reference MAY target:

- a scalar definition (e.g. a number, string, boolean),
- an object definition,
- an array definition.

### D.4.3 Lists and Maps of Referenced Elements

If the value of a field is a **single-element array containing a reference string**, the field is interpreted as a list whose elements use the referenced schema:

```json
{
  "$oky": {
    "company": {
      "addresses": ["&Address"]
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

**Semantics:**

- `addresses` is an array field.
- Each element in `addresses` must validate against `&Address`.

List size constraints can be combined as standard structural constraints on the field name:

```json
"addresses|[1,10]": ["&Address"]
```

A **map** field works the same way: a single-entry object whose value is a
reference string declares a map whose values use the referenced schema.

```json
"statsByRegion|[*:*]": { "region-1": "&Stat" }
```

- `statsByRegion` is a map field.
- Each value must validate against `&Stat`; the keys follow the map's key rule.

### D.4.4 Constraint Categories

Okyline distinguishes two categories of constraints with different behaviors:

#### Structural Constraints (contextual, defined at usage)

These constraints depend on the **context of use** and are NOT included from the referenced definition:

| Constraint | Description |
|------------|-------------|
| `@` | Required |
| `?` | Nullable |
| `[min,max]` | List size |
| `!` | Uniqueness in list |
| `%` | Default value |
| Label | Field description |

The same `Address` can be required in `Order` but optional in `UserProfile`.

#### Value Constraints (intrinsic, included)

These constraints define the **contract of the type itself** and ARE included from the referenced definition. They can only be modified via `$override`:

| Constraint | Description |
|------------|-------------|
| `#` | Key field(s) for object identity |
| `{min,max}` | String length |
| `(min..max)` | Numeric range |
| `('A','B')` | Enumeration |
| `~pattern~` | Regex/format |
| `(%Compute)` | Computed validation |

**Example:**

```json
{
  "$oky": {
    "user": {
      "primaryEmail|@": "&Email",
      "backupEmail|?": "&Email"
    }
  },
  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com"
  }
}
```

- `~$Email~` and `{5,100}` → included from `Email`
- `@` vs `?` → defined locally per usage

#### Special case - collection definitions

When a `$defs` entry *is* a collection type (a named list or map), its collection-level constraints are part of the type rather than contextual: the cardinality `[min,max]`, the uniqueness `!`, and the per-element constraints introduced by `->` (including element nullability `?`) are **intrinsic** and travel with every `&Name` reference. This is required because a chained reference has no field on which to attach the inner collection's structure - it can only live on the definition.

```json
{
  "$oky": { "matrix|@ [*]": ["&Row"] },
  "$defs": { "Row|[*] -> (>=0)": [0] }
}
```

`matrix` is `List<List<int>>`: the outer list's size sits on the `matrix` field, but the inner list's `[*]` and `-> (>=0)` can only live on `Row` - no field carries them, so they are intrinsic to the `Row` type.

(`@`, and `#` on the definition key, remain forbidden - see D.2.4.)

### D.4.5 Example Values

Example values are **included** from the referenced definition. They can be modified via `$override` if needed.

### D.4.6 Backward-Compatible and Opt-Out Modifiers

- `field | $ref`: explicit reference modifier. Accepted for backward compatibility with versions of Okyline prior to 1.7.0 (where it was required). It is now redundant when the value already starts with `&`, and can be omitted.
- `field | $str`: opt-out for the rare case where the literal string `"&..."` is the intended value of a string field rather than a reference. With `| $str`, a value such as `"&Address"` is treated as a plain string example, not a reference.

### D.4.7 Polymorphic References

A list or map field whose example contains more than one `&Name`
(optionally mixed with inline values) declares a polymorphic field -
each element / value matches any of the declared variants.

```json
"events|@ [*]":       ["&Login", "&Logout"]
"indicators|@ [*:*]": { "k1": "&Counter", "k2": "&Gauge" }
```

All variants of a polymorphic field MUST share the same kind: either all
scalar or all object. Mixing a scalar definition and an object definition in
the same field MUST cause schema loading to fail.

`| $str` on a list field disables `&Name` promotion (strings stay
literal). Supported on lists only.

---

## D.5 Object-Level References - Structural Composition

### D.5.1 Basic Inclusion

An object schema can **include** another definition as a **template** using a top-level `$ref` field:

```json
{
  "$oky": {
    "Person": {
      "$ref": "&Address",
      "name|@ {2,50}": "Dupond"
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

**Semantics:**

- `$ref` designates a **template** to include.
- All fields defined in the template are **injected** into the current object schema.
- Fields declared next to `$ref` (here `name`) are treated as **additional fields**, provided there is no name collision.

The effective `Person` schema is conceptually equivalent to:

```json
{
  "street|@ {2,100}": "12 rue du Saule",
  "city|@ {2,100}": "Lyon",
  "name|@ {2,50}": "Dupond"
}
```

### D.5.2 Target Type

For object-level inclusion:

- Object-level `$ref` **MUST** target an **object definition**.
- Targeting non-object definitions (scalar definitions) at object level is invalid and MUST cause a schema parsing error.

For property-level `$ref` (section D.4), the target MAY be scalar, object or array.

### D.5.3 Structural Composition Rules

Object-level `$ref` is a **structural composition**: it merges the fields of a template into the current object. Each object-level `$ref` targets exactly one template.

When a template contains conditional rules (`$requiredIf`, `$forbiddenIf`, `$appliedIf`, etc.) or `$compute` expressions, these stateful elements are injected into the including object alongside the template's fields. To preserve the semantic integrity of these included elements, the following restriction applies:

- **`$remove` not permitted**: if a template contains conditional rules or `$compute` expressions, the including schema MUST NOT use `$remove`. Combining `$remove` with such a template MUST cause a schema parsing error.

This restriction prevents broken references (removing a field that an included conditional rule depends on, which would produce an unsatisfiable schema).

Templates containing only fields (no conditional rules, no `$compute`) are unaffected by this restriction.

### D.5.4 Field Collision Rules

When a schema includes a template via `$ref`:

- If the including schema defines a field with the same name as a field in the template, a **collision** occurs.
- A collision MUST be resolved explicitly with `$override` or `$amend` (see D.7). Otherwise, the schema MUST be rejected.

### D.5.5 Single Inclusion

The value of `$ref` MUST be a single reference string (e.g. `"&Address"`). An object-level `$ref` targets exactly one template.

### D.5.6 Cycles and Recursion

Whether a cycle is legal depends on **how** the reference is used:

- **Object-level inclusion cycles are forbidden.** An object-level `$ref`
  (D.5.1) merges the target's fields into the including object. A cycle formed
  purely by object-level inclusion - A includes B which includes A (including
  the self-case A includes A) - would expand infinitely and MUST be detected
  and rejected at schema load time.
- **Property-level recursion is allowed.** A `$ref` used as the **type of a
  property**, or as the **element or value template of a list or map**, is
  composition, not inclusion: the recursion terminates whenever the property is
  absent or the collection is empty. This holds whether the reference is written
  in string form (`"&Type"`) or object form (`{"$ref": "&Type"}`).

---

## D.6 Template Adaptation - `$remove`

Okyline provides the `$remove` directive to **exclude fields** from included templates.

### D.6.1 Syntax

```json
{
  "$oky": {
    "AnonymousPerson": {
      "$ref": "&Person",
      "$remove": ["email", "ssn"]
    }
  },
  "$defs": {
    "Person": {
      "name|@ {1,50}": "John",
      "age|@ (0..150)": 42,
      "email|@ ~$Email~": "john@example.com",
      "ssn|@": "123-45-6789"
    }
  }
}
```

### D.6.2 Semantics

- `$ref` injects all fields from the `Person` template.
- `$remove` specifies fields to **exclude** from the effective schema.

Effective `AnonymousPerson` schema:

```json
{
  "name|@ {1,50}": "John",
  "age|@ (0..150)": 42
}
```

### D.6.3 Rules

- `$remove` MUST be an array of field names (strings).
- Each field name in `$remove` MUST exist in the referenced template. If a field does not exist, the schema MUST be rejected.
- `$remove` MUST NOT be used when `$ref` targets a template containing conditional rules or `$compute` expressions. See D.5.3 for the rationale.

---

## D.7 Field Adaptation - `$override` and `$amend`

Okyline provides two directives to adapt a field inherited from a template: `$override` replaces the field entirely, `$amend` replaces only the constraint blocks specified by the adapter. Both produce a new effective field by merging the base field (from the referenced template) with the adapter.

### D.7.1 Syntax

Both directives appear as a suffix after the pipe in a field declaration, at the same position as standard field constraints:

```
"fieldName | $override <constraints>": <exampleValue>
"fieldName | $amend <constraints>": <exampleValue>
```

`$override` and `$amend` MUST NOT appear on the same field declaration.

### D.7.2 Invariants (both directives)

The following aspects of the base field MUST be preserved by both `$override` and `$amend`:

- field type (scalar type, `obj`, `list`, `map`)
- collection nature
- `$ref` target (when the base field is itself a reference to a definition)

Any attempt to change one of these via `$override` or `$amend` MUST cause a schema parsing error.

### D.7.3 Merge Semantics

Let the *base* be the inherited field and the *adapter* be the `$override` or `$amend` declaration. The effective field is computed block-by-block:

- **`$override`**: for each constraint block (regex, string length, numeric range, enum, compute, label, example, list bounds, presence flag, nullable flag, unique flag, key flag), the effective value is the adapter's value. Blocks not specified by the adapter are **removed** from the effective field.
- **`$amend`**: for each constraint block, the effective value is the adapter's value if specified, otherwise the base's value. Blocks not specified by the adapter are **inherited** from the base. For primitive presence flags (`@`, `?`, `!`, `#`), `$amend` applies a logical OR: the flag is present in the effective field if it is present in either the base or the adapter. As a consequence, `$amend` cannot remove a flag from the base - `$override` must be used for that.

After the merge, the effective field carries exactly one block of each type, consistent with the single-constraint-per-type invariant defined in the Core Specification.

### D.7.4 Rules

- `X | $override ...` and `X | $amend ...` MUST target a field `X` that exists at the point of application, namely:
    - in a template referenced by `$ref` at the same object level, after applying `$remove`, or
    - at the parent object level of an `$appliedIf` branch (see Core §6.3.5).
  Otherwise, the schema MUST be rejected.
- Both directives MAY be combined with any standard field constraint, exactly like a normal field declaration.
- Without either directive, redefining an inherited or parent-level field name is a collision error.

### D.7.5 Example

```json
{
  "$oky": {
    "Employee": {
      "$ref": "&Person",
      "name | $amend @": "John Doe",
      "salary|@ (>=0)": 3000
    }
  },
  "$defs": {
    "Person": {
      "name|? {1,50}": "John",
      "age|@ (0..150)": 42
    }
  }
}
```

Effective `Employee` schema (after merge of `name` via `$amend @`):

```json
{
  "name|@? {1,50}": "John Doe",
  "age|@ (0..150)": 42,
  "salary|@ (>=0)": 3000
}
```

The `@` flag is added by `$amend`; the `?` flag, the string length `{1,50}` and the example are kept from the base. Using `$override @` instead would yield `name|@`, erasing `{1,50}`, `?` and the base example.

---

## D.8 Order of Application

For an object that uses `$ref`, `$remove`, `$override`, `$amend` and local fields, the effective schema is computed conceptually in the following order:

1. **Template injection** - Resolve the single `$ref` to its template and inject all fields.
2. **Removals** - Apply all `$remove` directives.
3. **Adaptations** - Apply all `$override` and `$amend` directives via the field merge defined in D.7.3.
4. **Local additions** - Add remaining locally-declared fields.

### D.8.1 Error Conditions

| Situation | Error |
|-----------|-------|
| `$ref` value is not a single reference string | Schema rejected |
| `$remove` targets non-existent field | Schema rejected |
| `$override` or `$amend` targets non-existent field (after removes) | Schema rejected |
| `$override` and `$amend` both specified on the same field declaration | Schema rejected |
| `$override` or `$amend` attempts to change field type, collection nature or `$ref` target | Schema rejected |
| Local field collides with included (no `$override` or `$amend`) | Schema rejected |
| Object-level cycle detected | Schema rejected |
| `$remove` used with a `$ref` targeting a template containing conditional rules or `$compute` | Schema rejected |

---

## D.9 Interaction with Conditional Directives

`$ref`, `$override` and `$remove` apply only to **structural fields** of an object.

A *structural field* is a field that is part of the object's schema after:

1. applying object-level `$ref` inclusion,
2. applying all `$remove` directives,
3. applying all `$override` directives,
4. adding local fields declared at the same level.

Fields that exist **only inside `$appliedIf` (or related conditional) branches** are **not** considered structural fields of the base object.

### D.9.1 Rules

- `$override` and `$remove` MAY only target structural fields included from the template.
- If `$override` or `$remove` targets a field that does not exist structurally in the referenced definition, the Okyline schema **MUST** be rejected.
- Conditional directives (`$requiredIf*`, `$forbiddenIf*`, `$appliedIf*`) declared in a referenced definition are **included** by the schema that uses `$ref`.
- Included conditional directives **cannot** be modified via `$override` or `$remove` (they apply to structural fields, not the directives themselves).
- Inclusion of conditional directives from a template is subject to the `$remove` restriction defined in D.5.3.

### D.9.2 Validation at Load Time

During schema loading, conditional directives are interpreted **after** structural resolution is complete:

- The field used in the condition (e.g. `status`, `paymentMethod`) **MUST** exist as a structural field of the object after `$ref`/`$remove`/`$override`.
- Every field name mentioned in `$requiredIf*` / `$forbiddenIf*` **MUST** either:
    - be a structural field of the object, or
    - be introduced by the conditional directive itself (e.g. inside a `$appliedIf` branch).

If a conditional directive references a field that does not exist according to these rules, the schema **MUST** be rejected.

---

## D.10 Summary

| Feature | Description |
|---------|-------------|
| `$defs` | Repository for reusable templates (root level, first level only) |
| `&Name` | Reference syntax (resolves in `$defs`) |
| `"field": "&Name"` | Property-level reference: field uses the referenced template as its type (scalar, object, or array) |
| `"field": ["&Name"]` | Array whose elements use the referenced template |
| `field \| $ref` | Backward-compatible explicit modifier (redundant since 1.7.0) |
| `field \| $str` | Opt-out: treat `"&..."` value as a literal string, not a reference |
| Object-level `$ref` | **Include** all fields from a single template - structural composition (single inclusion only) |
| `$override` | **Adapt** an included field - replace its definition block-by-block, unspecified blocks are removed (see D.7) |
| `$amend` | **Adapt** an included field - replace only specified blocks, unspecified blocks are inherited from the base (see D.7) |
| `$remove` | **Adapt** by excluding an included field |
| Structural constraints | `@`, `?`, `[...]`, `!`, `%`, labels - local, at usage |
| Value constraints | `#`, `{...}`, `(...)`, `~...~`, etc. - included, adapt via `$override` or `$amend` |
| Invariants preserved by `$override`/`$amend` | field type, collection nature, `$ref` target |
| Conditional directives / `$compute` | Included when `$ref` targets a template containing them; `$remove` is not permitted in that case (see D.5.3); not modifiable via `$override`/`$remove` |
| Object-level cycles | Forbidden (detected at load time) |
| Property-level recursion | Allowed |

> **Note:** For external references and versioned imports, see **Annex E - External Imports and Versioning**.

---

## D.11 Complete Example

```json
{
  "$okylineVersion": "1.4.0",
  "$version": "1.0.0",
  "$title": "Order Schema with Internal References",

  "$oky": {
    "order": {
      "$ref": "&Auditable",
      "orderId|@ # ~$OrderId~": "ORD-12345678",
      "customerEmail|@": "&Email",
      "status|@ ($ORDER_STATUS)": "PENDING",
      "items|@ [1,100]": ["&OrderItem"],
      "shippingAddress|@": "&Address",
      "billingAddress": "&Address",
      "total|@ (%OrderTotal)": 100.50,

      "$requiredIf status('SHIPPED','DELIVERED')": ["trackingNumber"],
      "trackingNumber|{10,50}": "TRACK123456"
    }
  },

  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com",

    "Auditable": {
      "createdAt|@ ~$DateTime~": "2025-01-01T00:00:00Z",
      "updatedAt|@ ~$DateTime~": "2025-01-01T00:00:00Z"
    },

    "Address": {
      "street|@ {5,100}": "123 Main Street",
      "city|@ {2,50}": "Paris",
      "postalCode|@ {5,10}": "75001",
      "country|@ {2}": "FR"
    },

    "OrderItem": {
      "sku|@ # {5,20}": "SKU-12345",
      "name|@ {2,200}": "Product Name",
      "quantity|@ (1..1000)": 1,
      "unitPrice|@ (>0)": 100.50
    }
  },

  "$format": {
    "OrderId": "^ORD-[0-9]{8}$"
  },

  "$compute": {
    "ItemTotal": "unitPrice * quantity",
    "OrderTotal": "total == sum(items, %ItemTotal)"
  },

  "$nomenclature": {
    "ORDER_STATUS": "PENDING,CONFIRMED,SHIPPED,DELIVERED,CANCELLED",
    "PAYMENT_METHOD": "CARD,PAYPAL,BANK_TRANSFER"
  }
}
```

---
## Changelog

### 1.7.0
- **Property-level references auto-promote on `&Name` value.** A field whose value is `&Name` (or `["&Name"]` for a list) is now interpreted as a reference without requiring a `| $ref` modifier (D.4.1, D.4.2, D.4.3). The `&` sigil already qualifies the intent, mirroring the convention used for decimal strings (e.g. `"1.00"`).
- **`| $ref` retained for backward compatibility**, but redundant when the value already starts with `&` (D.4.6).
- **`| $str` opt-out** introduced to treat `"&..."` as a literal string for the rare case where the string is the intended value, not a reference (D.4.6).
- **Polymorphic references (D.4.7).** A list or map field whose example contains more than one `&Name` (optionally mixed with inline values) declares a polymorphic field - each element / value matches any of the declared variants. `| $str` disables the promotion on lists; not supported on maps.
- **Collection definitions (D.2.4).** The key of a `$defs` entry now accepts the same constraint grammar as a field key (excluding `@` and `#`), allowing a definition to name a collection type (list or map, including leaf constraints) and chain it through `&Name` references. Enables map-of-map, list-of-list, list-of-map, map-of-list at any depth.
- Examples and summary table (D.10, D.11) updated to use the value-based form. Object-level `$ref` (used as a key inside an object for template inclusion) is unchanged.

### 1.6.0
- **Multiple inclusion removed**: object-level `$ref` now targets exactly one template. Array values for `$ref`, and the `$keep` directive, are no longer part of the language. D.5.5, D.5.6, D.5.7 and D.8.2 of the 1.4.0 draft are removed.
- D.5.3 relaxed: object-level `$ref` may now include a template that contains conditional rules or `$compute` expressions, provided no `$remove` directive is applied. Such stateful elements are injected into the including object together with the template's fields.
- D.6.3 and D.8.1 updated with the corresponding `$remove` restriction for stateful templates.
- D.9.1 harmonized with D.5.3: conditional directives declared in a template are effectively included in the schema that uses `$ref`.
- D.7 rewritten to introduce `$amend` alongside `$override` with unified merge semantics: both preserve field type, collection nature and `$ref` target. `$override` erases base blocks not respecified, `$amend` inherits them. Both directives apply uniformly in `$ref` object-level inclusion and in `$appliedIf` branches (see Core §6.3.5).

**End of Annex D - Internal Schema References (Normative)**
