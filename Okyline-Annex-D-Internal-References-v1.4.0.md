---
description: Internal schema references in Okyline - mechanism for composing and reusing schema fragments with $defs, $ref, $override, and $remove directives.
---

# Annex D: Internal Schema References (Normative)

**Version:** 1.4.0
**Date:** April 2026
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

For versioned imports and external schema references, see **Annex E — External Imports and Versioning**.

---

## D.1 Overview

Okyline supports **schema references** to promote reuse, modularity, and consistency within a schema document.

References allow a field or an object definition to reuse an existing schema fragment identified by a logical name stored in `$defs`.

Okyline distinguishes two complementary use cases:

1. **Property-level reference** — a single field whose type is taken from a definition (`field | $ref`).
2. **Object-level reference** — an object that includes all fields of another definition and may add, override or remove fields.

In Okyline, `$ref` is an **inclusion mechanism**: a referenced schema is treated as a base that can be extended, explicitly overridden or partially removed. This differs intentionally from JSON Schema, where `$ref` behaves as a total substitution.

---

## D.2 Definition Repository — `$defs`

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
      "email | $ref @": "&Email",
      "score | $ref": "&Percentage"
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

## D.4 Property-Level References — `field | $ref`

### D.4.1 Syntax

A field can use a definition as its **type** via the `$ref` constraint suffix:

```json
{
  "$oky": {
    "person": {
      "address | $ref": "&Address",
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
- The `$ref` constraint indicates that the **type and value constraints** of the field are taken from the target definition.
- The field behaves as if the referenced schema had been written inline at this location.

A property-level `$ref` MAY target:

- a scalar definition (e.g. a number, string, boolean),
- an object definition,
- an array definition.

### D.4.3 Lists of Referenced Elements

If the value associated with `field | $ref` is a **single-element array containing a reference string**, the field is interpreted as a list whose elements use the referenced schema:

```json
{
  "$oky": {
    "company": {
      "addresses | $ref": ["&Address"]
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

List size constraints can be combined:

```json
"addresses | $ref [1,10]": ["&Address"]
```

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
      "primaryEmail | $ref @": "&Email",
      "backupEmail | $ref ?": "&Email"
    }
  },
  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com"
  }
}
```

- `~$Email~` and `{5,100}` → included from `Email`
- `@` vs `?` → defined locally per usage

### D.4.5 Example Values

Example values are **included** from the referenced definition. They can be modified via `$override` if needed.

---

## D.5 Object-Level References — Structural Composition

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

Object-level `$ref` is a **structural composition**: it merges the fields of one or more templates into the current object.

A definition eligible for inclusion MUST contain only fields (structure). Definitions containing conditional rules (`$requiredIf`, `$forbiddenIf`, `$appliedIf`, etc.) or `$compute` expressions **cannot be included** via object-level `$ref`.

If a definition contains conditional rules or computes, using it as an object-level `$ref` target MUST cause a schema parsing error.

### D.5.4 Field Collision Rules

When a schema includes a template via `$ref`:

- If the including schema defines a field **with the same name** as a field in the template, a **collision** occurs.
- Collision is **forbidden** by default:
    - **Not using `$override`** → the schema MUST be rejected.
    - **Using `$override`** → explicitly allowed (see section D.7).

This rule enforces explicit intent when adapting included fields.

### D.5.5 Multiple Inclusions (Composite Templates)

An object schema can include **multiple templates** by providing an array to `$ref`:

```json
{
  "$oky": {
    "Article": {
      "$ref": ["&Auditable", "&Deletable"],
      "title|@ {1,200}": "Mon article"
    }
  },
  "$defs": {
    "Auditable": {
      "createdAt|@ ~$DateTime~": "2025-01-01T00:00:00Z",
      "updatedAt|@ ~$DateTime~": "2025-01-01T00:00:00Z"
    },
    "Deletable": {
      "deletedAt|? ~$DateTime~": "2025-01-01T00:00:00Z",
      "isDeleted|@": false
    }
  }
}
```

**Semantics:**

- All fields from all templates are injected into the current object.
- Templates are applied in order (left to right).
- In case of field name collision between templates, the schema **MUST be rejected** unless `$keep` is used to resolve the conflict (see D.5.6).
- Fields declared alongside `$ref` take precedence over all templates.

### D.5.6 Collision Between Templates

If two templates define a field with the same name and neither `$remove` nor `$keep` is used:

- The schema **MUST be rejected**.

To resolve:

- Use `$keep` to specify which template's field to retain (see D.5.7).
- Use `$remove` to exclude the field from all templates, then re-add locally.
- Or ensure no collision exists by design.

### D.5.7 `$keep` — Resolving Inclusion Conflicts

When multiple inclusions cause field name collisions, `$keep` specifies which template's version of a field to retain.

#### Syntax

```json
{
  "$oky": {
    "Combined": {
      "$ref": ["&A", "&B"],
      "$keep": ["&B.config"]
    }
  },
  "$defs": {
    "A": {
      "config|@|ConfigName": "Config-name-A",
      "name|@": "A"
    },
    "B": {
      "config|@|Config num": 25,
      "status|@": "active"
    }
  }
}
```

#### Semantics

- `$keep` is an array of strings in the format `&TemplateName.fieldName`.
- Each entry specifies that for the given `fieldName`, the version from `TemplateName` should be retained, and versions from other templates should be discarded.
- If a collision occurs on a field not listed in `$keep`, the schema MUST be rejected.
- `$keep` entries referencing non-existent templates or fields MUST cause a schema parsing error.

#### Rules

- `$keep` is only valid when using multiple inclusions (`$ref` as array).
- `$keep` resolves collisions silently — no error is raised for fields listed in `$keep`.
- `$keep` and `$remove` can be used together: `$keep` resolves which version to use, `$remove` can then exclude it entirely if needed.

### D.5.8 Cycles

- **Object-level cycles** (A includes B which includes A) are **forbidden** and MUST be detected at schema load time.
- **Property-level recursion** (A has a property of type A) is **allowed**.

---

## D.6 Template Adaptation — `$remove`

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
- Each field name in `$remove`:
    - MUST exist in at least one of the included templates.
    - If a field does not exist in any template, the schema MUST be rejected.
- `$remove` applies to **all** included templates: if multiple `$ref` targets define the same field, `$remove` excludes it regardless of origin.

### D.6.4 Resolving Multiple Inclusion Conflicts

When using multiple inclusions, `$remove` excludes fields from **all** templates. This allows resolving field collisions by removing the conflicting field and re-adding it locally:

```json
{
  "$oky": {
    "Article": {
      "$ref": ["&Auditable", "&Deletable"],
      "$remove": ["updatedAt"],
      "updatedAt|@ ~$DateTime~": "2025-06-15T10:30:00Z",
      "title|@": "Mon article"
    }
  },
  "$defs": {
    "Auditable": {
      "createdAt|@ ~$DateTime~": "2025-01-01T00:00:00Z",
      "updatedAt|@ ~$DateTime~": "2025-01-01T00:00:00Z"
    },
    "Deletable": {
      "deletedAt|~$DateTime~": "2025-01-01T00:00:00Z",
      "updatedAt|@ ~$DateTime~": "2025-01-01T00:00:00Z"
    }
  }
}
```

**Processing order:**

1. Collect `$remove` fields → `{updatedAt}`
2. Inject `Auditable`, skip `updatedAt` → `{createdAt}`
3. Inject `Deletable`, skip `updatedAt` → `{createdAt, deletedAt}`
4. Local additions → `{createdAt, deletedAt, updatedAt, title}`

No collision because `$remove` excludes the field from all injections.

---

## D.7 Template Adaptation — `$override`

Okyline provides the `$override` constraint to **redefine a field** from an included template.

### D.7.1 Syntax

```json
{
  "$oky": {
    "Employee": {
      "$ref": "&Person",
      "name | $override ? {10,20}": "Jean Dupont",
      "salary|@ (>=0)": 3000
    }
  },
  "$defs": {
    "Person": {
      "name|@ {1,50}": "John",
      "age|@ (0..150)": 42
    }
  }
}
```

### D.7.2 Semantics

- `$ref` injects all fields from the `Person` template.
- `name | $override ...` indicates that:
    - the included definition of `name` is **replaced**,
    - `Employee.name` is defined **entirely** by the local constraints after `$override`.
- Other included fields (e.g. `age`) remain unchanged.
- Additional local fields (e.g. `salary`) are simply added.

Effective `Employee` schema:

```json
{
  "name|? {10,20}": "Jean Dupont",
  "age|@ (0..150)": 42,
  "salary|@ (>=0)": 3000
}
```

### D.7.3 Rules

- `X | $override ...` MUST refer to a field `X` that exists in a template referenced by `$ref` (after removals).
  Otherwise, the schema MUST be rejected.
- When `$override` is present, the included definition for that field is completely replaced by the local one.
- `$override` MAY be combined with any standard field constraints (`@`, list bounds, formats, etc.), exactly like a normal field definition.
- Without `$override`, attempting to redefine an included field name is an error (see D.5.4).
- `$override` applies to **value constraints only**. Structural constraints are always defined locally anyway.

---

## D.8 Order of Application

For an object that uses `$ref`, `$remove`, `$override` and local fields, the effective schema is computed conceptually in the following order:

### D.8.1 Single Inclusion

1. **Template injection** — Resolve the `$ref` to a template and inject all fields.
2. **Removals** — Apply all `$remove` directives.
3. **Overrides** — Apply all `$override` directives.
4. **Local additions** — Add remaining locally-declared fields.

### D.8.2 Multiple Inclusions

For `"$ref": ["&A", "&B", "&C"]`:

1. **Inject A** → Apply removes → current field set
2. **Inject B** → Apply removes → current field set (collision = error unless `$keep`)
3. **Inject C** → Apply removes → current field set (collision = error unless `$keep`)
4. **Apply overrides** on final included set
5. **Local additions** (collision = error unless `$override`)

The `$remove` directives act as a persistent filter, removing the named field after each injection.

### D.8.3 Error Conditions

| Situation | Error |
|-----------|-------|
| `$remove` targets non-existent field | Schema rejected |
| `$override` targets non-existent field (after removes) | Schema rejected |
| Local field collides with included (no override) | Schema rejected |
| Two templates define same field (not removed, no `$keep`) | Schema rejected |
| Object-level cycle detected | Schema rejected |

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
| `field \| $ref` | Reuse a template as the type of a field (scalar, object, or array) |
| `field \| $ref`: `["&Name"]` | Array whose elements use the referenced template |
| Object-level `$ref` | **Include** all fields from template(s) — structural composition |
| `"$ref": ["&A", "&B"]` | Multiple inclusions (collision = error unless `$keep` or `$remove`) |
| `$keep` | Select which template's field to retain on collision |
| `$override` | **Adapt** an included field — replace its definition |
| `$remove` | **Adapt** by excluding an included field |
| Structural constraints | `@`, `?`, `[...]`, `!`, `%`, labels — local, at usage |
| Value constraints | `#`, `{...}`, `(...)`, `~...~`, etc. — included, adapt via `$override` |
| Conditional directives | Included, not modifiable |
| Object-level cycles | Forbidden (detected at load time) |
| Property-level recursion | Allowed |

> **Note:** For external references and versioned imports, see **Annex E — External Imports and Versioning**.

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
      "customerEmail | $ref @": "&Email",
      "status|@ ($ORDER_STATUS)": "PENDING",
      "items | $ref @ [1,100]": ["&OrderItem"],
      "shippingAddress | $ref @": "&Address",
      "billingAddress | $ref": "&Address",
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

**End of Annex D — Internal Schema References (Normative)**
