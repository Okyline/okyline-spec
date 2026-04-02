---
description: Virtual fields in Okyline - computed values that exist only during validation for conditional rules based on derived classifications, tiers, or flags.
---

# Annex F: Virtual Fields (Normative)

**Version:** 1.4.0
**Date:** April 2026
**Status:** Draft

 License: This annex is part of the Open Okyline Language Specification and is subject to the same license terms (CC
  BY-SA 4.0). See the Core Specification for full license details.

This annex defines **virtual fields** (`$field`), a mechanism to declare computed values that exist only during validation and can be used in conditional directives. Virtual fields enable scenarios where conditional rules depend on derived values not present in the validated data, such as computed tiers, classifications, or flags.

---

## Relation to the Core Specification

This annex defines the semantics of **virtual fields** (`$field`), a mechanism to declare computed values that exist only during validation and can be used in conditional directives.

Virtual fields **require** the Expression Language defined in **Annex C**. Implementations claiming support for virtual fields MUST implement Annex C (Level 3 conformance).

Virtual fields are distinct from:

- **`$compute` expressions** (Annex C) — reusable expressions for field constraints
- **`$defs` definitions** (Annex D) — reusable schema fragments

While `$compute` expressions validate field values, virtual fields **derive new values** from existing data for use in conditional logic.

---

## F.1 Overview

Virtual fields address scenarios where conditional rules depend on **derived values** not present in the validated data:

- A discount tier computed from order total
- A customer classification derived from multiple attributes
- A date category (past, present, future) computed from a date field

Virtual fields are:

- **Computed** — their value is the result of an expression evaluation
- **Virtual** — they do not correspond to JSON properties in the validated data
- **Scoped** — they exist only within the object where they are declared
- **Immutable** — their value is computed once per validation context
- **Chainable** — they can reference other virtual fields declared before them

---

## F.2 Declaration Syntax

### F.2.1 Basic Syntax

Virtual fields are declared using the `$field` directive within an object schema. Each virtual field is declared as a separate key-value pair:

```json
{
  "$oky": {
    "order": {
      "$field discountTier": "%CalculateTier",
      "total": 1500.00,
      "$appliedIf discountTier('GOLD')": {
        "loyaltyBonus|@": 50.00
      }
    }
  },
  "$compute": {
    "CalculateTier": "total >= 1000 ? 'GOLD' : 'STANDARD'"
  }
}
```

The syntax is:
```
"$field <variableName>": "%<ComputeExpressionName>"
```

### F.2.2 Normative Rules

- Each `$field` declaration is a separate JSON key starting with `$field ` followed by the variable name.
- The value MUST be a reference to a `$compute` expression, using the `%Name` syntax.
- Virtual field names MUST be valid identifiers (start with letter or underscore, contain only alphanumeric and underscore).
- Virtual field names MUST NOT collide with actual field names in the same object.
- Virtual field names MUST NOT collide with field names in `$appliedIf` payloads.
- Virtual field names MUST NOT be used in `$requiredIf` or `$forbiddenIf` field lists (they are computed, not data fields).
- Virtual fields MUST NOT reference other virtual fields declared **after** them (forward reference prohibition).
- `$field` directives MUST be declared at object level, not inside `$appliedIf` payloads.

---

## F.3 Evaluation Semantics

### F.3.1 Evaluation Context

Virtual field expressions (via their `$compute` reference) are evaluated in the **same context** as other expressions at that object level:

- Access to sibling data fields by name
- Access to ancestor contexts via `parent`, `root` (see §6.3.20)
- Access to `$compute` expressions via `%Name` syntax
- Access to **previously declared virtual fields** in the same object

### F.3.2 Evaluation Order

Virtual fields are evaluated **sequentially in declaration order**:

1. Parse object structure
2. For each `$field` in declaration order:
   a. Evaluate the referenced `$compute` expression
   b. The expression can reference previously evaluated virtual fields
   c. Store the result for use by subsequent virtual fields and conditional directives
3. Process conditional directives (`$requiredIf`, `$forbiddenIf`, `$appliedIf`, etc.)
4. Validate field constraints

**Forward Reference Prohibition:** A `$compute` expression referenced by a `$field` MUST NOT use another virtual field declared later in the same object. This is detected at schema parsing time and produces a schema error.

### F.3.3 Chained Virtual Fields

Virtual fields can form **chains** where one field depends on another. Each evaluated virtual field becomes immediately available to subsequent virtual field expressions.

```json
{
  "$oky": {
    "order": {
      "$field subtotal": "%ComputeSubtotal",
      "$field tax": "%ComputeTax",
      "$field total": "%ComputeTotal",
      "price": 150,
      "quantity": 5,
      "$appliedIf total(> 500)": {
        "bulkDiscount|@": "APPLIED"
      }
    }
  },
  "$compute": {
    "ComputeSubtotal": "price * quantity",
    "ComputeTax": "subtotal * 0.2",
    "ComputeTotal": "subtotal + tax"
  }
}
```
In this example:
- `subtotal` references `price` and `quantity`, evaluates to 750
- `tax` references `subtotal`, evaluates to 150
- `total` references `subtotal` and `tax`, evaluates to 900
- The condition `total(> 500)` triggers

**Invalid example (forward reference):**

```json
{
  "$oky": {
    "order": {
      "$field total": "%UsesLater",
      "$field subtotal": "%GetSubtotal",
      "price": 100,
      "quantity": 5
    }
  },
  "$compute": {
    "UsesLater": "subtotal + tax",
    "GetSubtotal": "price * quantity"
  }
}
```

This produces a **schema error**: `$field 'total' references 'subtotal' which is declared later`.

### F.3.4 Result Types

Virtual field expressions MAY return any type:

| Result Type | Condition Usage |
|-------------|-----------------|
| String | `$appliedIf fieldName('value')` |
| Number | `$appliedIf fieldName(>100)` |
| Boolean | `$appliedIf fieldName(true)` |
| Null | `$appliedIf fieldName(null)` |

If evaluation produces `null`, the virtual field's value is `null`. Use the `null` literal in conditions to test for this case (see F.4.2).

### F.3.5 Error Handling

If a virtual field expression fails to evaluate:

- The virtual field is treated as `null` (not existing)
- Validation continues with remaining rules
- Dependent virtual fields in the chain will receive `null` for the failed field

---

## F.4 Usage in Conditional Directives

### F.4.1 Trigger Syntax

Virtual fields are referenced in condition triggers by their name (without any prefix):

```
fieldName
```

This is the same syntax as for regular data fields. The validator resolves the name first against virtual fields, then against data fields.

### F.4.2 Supported Directives

Virtual fields can be used as triggers in **value-based** conditional directives:

| Directive | Example |
|-----------|---------|
| `$requiredIf` | `"$requiredIf tier('PREMIUM')": ["supportEmail"]` |
| `$requiredIfNot` | `"$requiredIfNot active(true)": ["reason"]` |
| `$forbiddenIf` | `"$forbiddenIf category('RESTRICTED')": ["publicUrl"]` |
| `$forbiddenIfNot` | `"$forbiddenIfNot enabled(true)": ["legacyMode"]` |
| `$appliedIf` | `"$appliedIf type('SPECIAL')": { ... }` |
| `$appliedIf` (switch) | See F.4.3 |

**Restriction:** Virtual fields MUST NOT be used as triggers in existence-based directives (`$requiredIfExist`, `$forbiddenIfExist`, `$appliedIfExist` and their negated variants). These directives test whether a field is present in the JSON data, which is a concept that does not apply to computed values. Using a virtual field as trigger in an IfExist directive produces a schema parsing error.

To test if a virtual field's value is null, use the `null` literal in a value-based condition:

```json
{
  "$requiredIf computedField(null)": ["fallback"],
  "$requiredIfNot computedField(null)": ["derived"]
}
```

### F.4.3 Switch Directive

Virtual fields can be used with the switch form of `$appliedIf`:

```json
{
  "$oky": {
    "order": {
      "$field tier": "%ComputeTier",
      "amount": 500.00,
      "$appliedIf tier": {
        "('GOLD')": {
          "discount|@": 0.15
        },
        "('SILVER')": {
          "discount|@": 0.10
        },
        "('BRONZE')": {
          "discount|@": 0.05
        }
      }
    }
  },
  "$compute": {
    "ComputeTier": "amount >= 1000 ? 'GOLD' : amount >= 500 ? 'SILVER' : 'BRONZE'"
  }
}
```

---

## F.5 Scope Rules

### F.5.1 Visibility

Virtual fields are visible only within the object where they are declared:

```json
{
  "$oky": {
    "order": {
      "$field orderTier": "%GetTier",
      "total": 1500,
      "items|[*]": [{
        "$appliedIf orderTier('HIGH')": { }
      }]
    }
  },
  "$compute": {
    "GetTier": "total >= 1000 ? 'HIGH' : 'LOW'"
  }
}
```

In this example, `orderTier` is **not accessible** from within `items` array elements. Each array element has its own validation context.

### F.5.2 No Inheritance

Virtual fields are **not inherited** through `$ref` inclusion:

```json
{
  "$oky": {
    "derived": {
      "$ref": "&Base",
      "$appliedIf computed(>15)": { }
    }
  },
  "$defs": {
    "Base": {
      "$field computed": "%DoubleValue",
      "value": 10
    }
  },
  "$compute": {
    "DoubleValue": "value * 2"
  }
}
```

The `computed` virtual field from `Base` is **not available** in `derived`. Virtual fields must be redeclared if needed.

### F.5.3 Nested Objects

Each nested object has its own independent scope for virtual fields:

```json
{
  "$oky": {
    "parent": {
      "$field category": "%GetParentType",
      "$field isPremium": "%IsPremium",
      "parentType": "STANDARD",
      "$appliedIf isPremium(true)": {
        "parentFeature|@": "premium"
      },
      "child": {
        "$field category": "%GetChildType",
        "$field isPremium": "%IsPremium",
        "childType": "PREMIUM",
        "$appliedIf isPremium(true)": {
          "childFeature|@": "premium"
        }
      }
    }
  },
  "$compute": {
    "GetParentType": "parentType",
    "GetChildType": "childType",
    "IsPremium": "category == 'PREMIUM'"
  }
}
```

In this example:
- Parent's `category` = "STANDARD", `isPremium` = false → `parentFeature` not required
- Child's `category` = "PREMIUM", `isPremium` = true → `childFeature` required

The same virtual field name (`category`, `isPremium`) can be used in both scopes without conflict.

---

## F.6 Interaction with `$compute`

### F.6.1 Mandatory Use of `$compute`

Virtual field values MUST reference `$compute` expressions. Inline expressions are not supported:

**Valid:**
```json
{
  "$oky": {
    "order": {
      "$field tier": "%CalculateTier",
      "total": 1500.00
    }
  },
  "$compute": {
    "CalculateTier": "total >= 1000 ? 'GOLD' : 'STANDARD'"
  }
}
```

**Invalid:**
```json
{
  "$oky": {
    "order": {
      "$field tier": "total >= 1000 ? 'GOLD' : 'STANDARD'",
      "total": 1500.00
    }
  }
}
```

### F.6.2 Distinction

| Feature | `$compute` | `$field` |
|---------|-----------|----------|
| Declaration | Root level | Object level |
| Value | Expression string | Reference to `$compute` (`%Name`) |
| Purpose | Reusable expressions | Derived values for conditions |
| Usage in constraints | `(%Name)` in field constraints | Name in condition triggers |
| Scope | Global within schema | Local to declaring object |
| Chaining | N/A | Can reference earlier virtual fields |

---

## F.7 JSON Schema Transpilation

### F.7.1 Approach

Virtual fields do not exist in the JSON data, so they cannot be directly represented in JSON Schema. The transpiler handles virtual fields by:

1. Adding an `x-oky-virtualField-<directive>` extension in the `if` block
2. Using `"not": {}` to make the condition always false (JSON Schema cannot evaluate compute expressions)
3. Preserving the `then` block for documentation purposes

### F.7.2 Extension Format

```json
{
  "if": {
    "x-oky-virtualField-appliedIf": {
      "virtualField": "tier",
      "compute": "CalculateTier",
      "condition": "('GOLD')"
    },
    "not": {}
  },
  "then": {
    "properties": { ... },
    "required": [ ... ]
  }
}
```

### F.7.3 Limitations

Virtual field conditions cannot be validated by JSON Schema validators. The transpiled schema:

- Documents the intent via `x-oky-` extensions
- Always evaluates conditions as false (conservative approach)
- The Okyline schema remains the authoritative specification for full validation

---

## F.8 Summary

| Feature | Description |
|---------|-------------|
| Syntax | `"$field <name>": "%<ComputeName>"` |
| Expression | Must reference a `$compute` expression |
| Trigger syntax | `fieldName` in value-based conditional directives |
| IfExist restriction | Virtual fields MUST NOT be used in IfExist directives |
| Scope | Local to declaring object; not inherited |
| Chaining | Can reference previously declared virtual fields |
| Forward reference | Referencing a later-declared virtual field is a schema error |
| Null handling | Use `fieldName(null)` to test for null values |
| Evaluation order | Sequential in declaration order, before conditional directives |

---

## F.9 Complete Example

```json
{
  "$okylineVersion": "1.4.0",
  "$version": "1.0.0",
  "$title": "Order Schema with Virtual Fields",

  "$oky": {
    "order": {
      "customerId|@ {5,20}": "CUST-12345",
      "orderDate|@ ~$Date~": "2025-06-15",
      "items|@ [1,100]": [{
        "sku|@": "SKU-001",
        "quantity|@ (1..1000)": 2,
        "unitPrice|@ (>0)": 49.99
      }],
      "subtotal|@ (%CheckSubtotal)": 99.98,
      "shippingCost|@ (>=0)": 5.00,
      "total|@ (%CheckTotal)": 104.98,

      "$field orderTotal": "%GetTotal",
      "$field itemCount": "%GetItemCount",
      "$field orderTier": "%ComputeTier",
      "$field isHighValue": "%IsHighValue",
      "$field isBulkOrder": "%IsBulkOrder",

      "$appliedIf orderTier": {
        "('PREMIUM')": {
          "priorityShipping|@": true,
          "loyaltyPoints|@ (>=0)": 500
        },
        "('STANDARD')": {
          "loyaltyPoints|@ (>=0)": 100
        },
        "('BASIC')": {
          "loyaltyPoints?": 0
        }
      },

      "$requiredIf isHighValue(true)": ["approvalCode"],
      "approvalCode|? {10,20}": "APPROVE-12345",

      "$appliedIf isBulkOrder(true)": {
        "bulkOrderNote|@": "Bulk order processing"
      }
    }
  },

  "$compute": {
    "GetTotal": "total",
    "GetItemCount": "count(items)",
    "ComputeTier": "orderTotal >= 500 ? 'PREMIUM' : orderTotal >= 100 ? 'STANDARD' : 'BASIC'",
    "IsHighValue": "orderTotal >= 1000",
    "IsBulkOrder": "itemCount > 10",
    "LineTotal": "quantity * unitPrice",
    "CheckSubtotal": "subtotal == sum(items, %LineTotal)",
    "CheckTotal": "total == subtotal + shippingCost"
  }
}
```

### Explanation

1. **Virtual field `orderTotal`** — Captures the total for use in other computations
2. **Virtual field `itemCount`** — Counts items in the order
3. **Virtual field `orderTier`** — Computes tier based on `orderTotal` (chained reference)
4. **Virtual field `isHighValue`** — Boolean flag based on `orderTotal` (chained reference)
5. **Virtual field `isBulkOrder`** — Boolean flag based on `itemCount` (chained reference)
6. **`$appliedIf orderTier`** — Applies different schemas based on computed tier
7. **`$requiredIf isHighValue(true)`** — Requires approval code for high-value orders
8. **`$appliedIf isBulkOrder(true)`** — Adds bulk order note for large orders

---

**End of Annex F — Virtual Fields (Normative)**
