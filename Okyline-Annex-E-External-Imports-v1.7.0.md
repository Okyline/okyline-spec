---
description: Annex E of the Okyline specification - external schema imports, cross-schema composition, identifiers, and version management.
---

# Annex E - External Imports and Versioning (Normative)

**Version:** 1.7.0
**Date:** May 2026
**Status:** Draft

 License: This annex is part of the Open Okyline Language Specification and is subject to the same license terms (CC
  BY-SA 4.0). See the Core Specification for full license details.

This annex extends Annex D to enable **cross-schema composition** in enterprise environments. It defines schema identity (`$id`, `$version`), versioned dependencies (`$deps`), and external imports (`$import`). Organizations can publish schemas to a registry and import definitions from other contracts while maintaining strict version control and compatibility guarantees.

---

## Relation to the Core Specification

This annex **extends Annex D** by adding support for:

- schema identity and versioning (`$id`, `$version`)
- versioned dependencies (`$deps`)
- external definition imports (`$import`)
- registry-based resolution

Annex D defines the core mechanisms for structural composition (`$ref`, `$defs`, `$override`, `$remove`). This annex enables those same mechanisms to operate **across schema boundaries** through versioned imports.

Implementations MAY support Annex D without Annex E, providing internal composition only. Full Okyline 1.2.0 conformance requires both annexes.

---

## E.1 Overview

In enterprise environments, schemas often need to:

- share common definitions across multiple contracts
- evolve independently while maintaining compatibility
- declare explicit dependencies with version constraints

Okyline addresses these needs through a **two-layer architecture**:

1. **Annex D (internal)** - `$defs` + `$ref` for composition within a single document
2. **Annex E (external)** - `$import` + `$deps` for importing definitions from other schemas

External definitions, once imported via `$import`, behave **exactly like internal definitions** in `$defs`. The composition mechanisms (`$ref`, `$override`, `$remove`) work identically regardless of origin.

---

## E.2 Schema Identity - `$id`, `$version`, and `$state`

### E.2.1 Contract Identity

Each publishable Okyline schema is identified by a pair `($id, $version)` that defines a unique contract in the registry.

The format and validation rules for `$id` and `$version` are defined in the **Core Specification [§7.3 Optional Metadata Keys]**. These fields can be used as simple metadata for traceability even without this annex.

**Example:**

```json
{
  "$id": "sales.orders",
  "$version": "1.3.4",
  "$oky": { }
}
```

### E.2.2 When `$id` and `$version` Become Required

While `$id` and `$version` are optional in the Core specification, they become **required** when using external imports:

| Field | Required when |
|-------|---------------|
| `$id` | Schema is published to a registry, or referenced by other schemas via `$import` |
| `$version` | Schema is published to a registry, or uses external dependencies via `$deps` |

### E.2.3 Pre-release Version Resolution

When resolving dependencies, pre-release versions follow semver ordering:

```
2.0.0-alpha < 2.0.0-alpha.1 < 2.0.0-beta < 2.0.0-rc.1 < 2.0.0
```

**Registry behavior:**

- Stable versions are preferred over pre-release versions
- Build metadata (`+build.123`) is accepted but ignored for registry resolution (it does not affect version ordering)
- See [§E.6 Registry Resolution](#e6-registry-resolution) for the complete resolution algorithm

### E.2.4 Lifecycle State - `$state`

The optional `$state` field declares the lifecycle state of the contract. It
carries the publisher's intent regarding whether the schema is still being
shaped or has been frozen for consumption.

**Allowed values:**

| Value | Semantics |
|-------|-----------|
| `"DRAFT"` | The schema is work-in-progress and MAY evolve without prior notice, even at the same `$version`. Consumers MUST consider its content unstable. |
| `"FINAL"` | The schema is frozen. Once a `(id, version)` pair is published as `FINAL`, its content MUST NOT change. Any modification requires a new `$version`. |

**Default:**

If `$state` is absent, the schema MUST be treated as `"DRAFT"`. Implementations
MAY enforce explicit `$state` declaration at publication boundaries (e.g. in a
registry workflow), but the language parser itself MUST accept the absence and
apply the default.

**Validation:**

Any value other than `"DRAFT"` or `"FINAL"` MUST cause schema loading to fail.

**Example:**

```json
{
  "$id": "sales.orders",
  "$version": "1.0.0",
  "$state": "FINAL",
  "$oky": { }
}
```

> **Note:** The lifecycle state has no effect on schema validation itself; it
> informs registry, packaging, and dependency-binding tooling. The
> registry-level consequences of `DRAFT` and `FINAL` (immutability guarantees,
> transitive constraints) are out of scope for this annex.

### E.2.5 Validation Entry Points - `$entries`

The optional `$entries` block declares additional validation entry points
within a single schema. Each entry maps a public name to a local type
definition; consumers can request validation against a specific entry rather
than only against the root `$oky` payload.

**Syntax:**

```json
{
  "$entries": {
    "AnimalCreate": "&Animal",
    "OrderCreate":  "&Order"
  },
  "$defs": {
    "Animal": { "name|@ {2,50}": "Rex", "age|@ (0..50)": 3 },
    "Order":  { "id|@ ~^ORD-[0-9]+$~": "ORD-1" }
  },
  "$oky": { "label|@ {1,100}": "Pet store API" }
}
```

**Normative Rules:**

- `$entries` is **optional**. Its absence does not change validation behavior:
  `$oky` remains the default validation target.
- Each key MUST match `^[a-zA-Z][a-zA-Z0-9_]*$`.
- Each value MUST be a string of the form `&Name` where `Name` is a simple
  identifier. The external-reference form `&schemaId.Name` is **not** allowed
  here; import via `$import` first.
- `Name` MUST resolve to a type definition available in the schema's unified
  symbol table (local `$defs` or imported `$import` alias). If it does not,
  schema loading MUST fail.
- Validation against an unknown entry name MUST surface a runtime error.

Validating a JSON payload against `"AnimalCreate"` enforces the `Animal`
contract; validating without an entry name targets `$oky`.

> **Note:** `$entries` is a runtime concern; it does not affect what other
> schemas can import. See §E.4.5 Visibility for export control.

---

## E.3 Dependency Declaration - `$deps`

The `$deps` block declares **external dependencies** with version constraints.

### E.3.1 Syntax

```json
{
  "$id": "orders",
  "$version": "2.1.0",

  "$oky": { },

  "$deps": {
    "common": "~1.3.0",
    "crm": "^2.0.0",
    "billing": "1.5.0"
  }
}
```

### E.3.2 Version Constraint Syntax

Okyline supports three constraint modes, following semver conventions:

| Syntax | Name | Matches | Example |
|--------|------|---------|---------|
| `1.3.4` | Exact | Only this version | `1.3.4` only |
| `~1.3.0` | Tilde (patches) | Same MAJOR.MINOR, any PATCH | `1.3.0`, `1.3.1`, `1.3.17`... but not `1.4.0` |
| `^1.3.0` | Caret (compatible) | Same MAJOR, any MINOR/PATCH | `1.3.0`, `1.4.0`, `1.9.5`... but not `2.0.0` |

The choice of constraint mode is a governance decision left to each organization. The specification does not mandate or recommend a particular strategy.

### E.3.3 Normative Rules

- `$deps` is **required** when the schema uses at least one external definition via `$import`
- If the schema does not use any external definitions, `$deps` **SHOULD** be omitted
- When present, `$deps` MUST:
    - be a non-empty object
    - have keys corresponding to external schema identifiers (the `$id` of other contracts)
    - have values that are valid version constraints (exact, tilde, or caret)

#### Dependency Presence Rule (Normative)

`$deps` MUST be present whenever the schema contains at least one
non-empty `$import` import block, whether inside `$defs`,
`$nomenclature`, `$format`, or `$compute`. A schema that declares
`$import` imports without a corresponding `$deps` section MUST be rejected
at load time.

If no `$import` import block is present (or all are empty), `$deps` SHOULD
be omitted, but a `$deps` entry that is not referenced by any `$import`
import is tolerated. Hygiene of unused dependency declarations is a
governance concern that implementations MAY surface through separate
linting tooling, not through schema rejection. This aligns with the
convention adopted by mainstream package managers (npm, Maven, Cargo).
---

## E.4 External Definition Imports - `$import`

The `$import` block creates **local aliases** for definitions from external schemas.

### E.4.1 Syntax

```json
{
  "$id": "orders",
  "$version": "2.1.0",

  "$oky": {
    "order": {
      "shippingAddress | $ref @": "&Address",
      "customerEmail | $ref @": "&Email"
    }
  },

  "$defs": {
    "$import": {
      "Address": "&common.Address",
      "Email": "&common.Email"
    }
  },

  "$deps": {
    "common": "~1.3.0"
  }
}
```

### E.4.2 Semantics

- `$import` maps **local aliases** to external definitions
- The value `"&common.Address"` means:
    - `common` is the `$id` of another Okyline schema
    - `Address` is the name of a definition in `common`'s `$defs`
    - the version of `common` is resolved from `$deps["common"]` by the registry

Once imported, aliases in `$import` behave **exactly like entries in `$defs`**:

- Referenced via `&Name` syntax (not `&schemaId.Name`)
- Used with `$ref` for property-level or object-level composition
- Subject to `$override` and `$remove` for adaptation

### E.4.3 Normative Rules

- `$import` is **optional**
- If `$import` is non-empty, `$deps` MUST be present and non-empty
- For each entry `"Alias": "&schemaId.Name"` in `$import`:
    - `schemaId` MUST appear as a key in `$deps`
    - the schema resolved by the registry for (`schemaId`, `$deps[schemaId]`) MUST exist
    - that schema MUST define `Name` in its `$defs`
- A name MUST NOT appear in both `$defs` and `$import` - if such a collision occurs, the schema MUST be rejected
- Inside `$oky` and `$defs`, references MUST use the local alias `&Name`, not `&schemaId.Name`

#### Import Kind Restriction (Normative)

`$import` MUST be declared inside one of the four sub-blocks; each sub-block
restricts what kinds of definitions can be imported through it:

- A `$import` block declared inside `$defs` MUST import only type and object definitions.
- A `$import` block declared inside `$nomenclature` MUST import only nomenclatures.
- A `$import` block declared inside `$format` MUST import only formats.
- A `$import` block declared inside `$compute` MUST import only computed expressions.

Any `$import` entry that attempts to import an element of a different kind
than its enclosing block MUST cause schema loading to fail.

A `$import` block declared at the root of the schema document (not inside one
of the four sub-blocks) MUST cause schema loading to fail.

#### Implementation Dependency (Normative)

Support for external imports via `$import` requires support for `$ref`
resolution as defined in Annex D.

An implementation that does not implement `$ref` MUST reject any schema that
contains at least one non-empty `$import` import block.


### E.4.4 Aliasing for Disambiguation

When multiple external schemas define a definition with the same name, `$import` aliases resolve the ambiguity:

```json
{
  "$id": "shipping",
  "$version": "1.0.0",

  "$oky": {
    "Shipment": {
      "destination | $ref @": "&Address",
      "legacyOrigin | $ref ?": "&LegacyAddress"
    }
  },

  "$defs": {
    "$import": {
      "Address": "&common.Address",
      "LegacyAddress": "&legacy.Address"
    }
  },

  "$deps": {
    "common": "~1.0.0",
    "legacy": "~2.0.0"
  }
}
```

Both `common` and `legacy` define `Address`, but the local aliases `Address` and `LegacyAddress` are unambiguous within this schema.

### E.4.5 Visibility - `$public` and `$private`

Each sub-block (`$defs`, `$nomenclature`, `$format`, `$compute`) MAY declare
a visibility rule that controls which of its entries are exposable to
external imports via `$import`. The two directives are mutually exclusive
within a single sub-block.

**Syntax:**

```json
{
  "$defs": {
    "Animal":  { "name|@": "Rex" },
    "Helper":  { "x|@": "v" },
    "$private": ["Helper"]
  },
  "$nomenclature": {
    "STATUS":  "DRAFT,FINAL",
    "$public": ["STATUS"]
  },
  "$format": {
    "OrderId": "^ORD-[0-9]+$",
    "$private": []
  }
}
```

**Semantics:**

| Directive present in a sub-block | Effect |
|---|---|
| Neither `$public` nor `$private` | All entries are exposed (default - backward-compatible). |
| `$public` declared | Strict allow-list: only listed entries are exposed. Adding a new entry leaves it private by construction. |
| `$private` declared | Deny-list: every entry is exposed except the listed ones. |
| Both declared | **Error** - mutually exclusive. |

The listed names are bare (no sigil); the enclosing sub-block determines the
kind unambiguously.

**Normative Rules:**

- Each sub-block MAY declare **at most one** of `$public` or `$private`.
  Declaring both within the same sub-block MUST cause schema loading to fail.
- The directive value MUST be an array of strings.
- `$public: []` is accepted and locks the entire sub-block (nothing exposed).
- `$private: []` MUST be rejected as ambiguous - equivalent to absence.
- Each listed name MUST exist in the sub-block (as a local entry or as an
  imported `$import` alias). If it does not, schema loading MUST fail.

**Effect on imports:**

An external `$import` import that references a hidden name (entry present in
the source sub-block but not exposed) MUST fail with the same error as if
the name did not exist. The source schema MUST NOT disclose whether the
name is genuinely absent or merely hidden.

**Example:**

```json
{
  "$id": "common",
  "$version": "1.0.0",
  "$defs": {
    "Address":  { "city|@": "Paris" },
    "Internal": { "x|@": "v" },
    "$private": ["Internal"]
  },
  "$oky": { "f|@": "v" }
}
```

A consumer schema MAY import `&common.Address` but MUST NOT import
`&common.Internal`; attempting the latter is treated as an unknown name.

> **Note:** Visibility is a barrier to **import**, not a runtime gate. A
> type that is exposed but uses a hidden nomenclature/format/compute/type
> internally continues to resolve those internal references at validation
> time - the source's own scope is preserved (§E.8.5).

### E.4.6 Re-export through Alias Chains

A `$import` alias is itself a valid import target. An external import
`&schemaId.Name` succeeds whether `Name` is a local entry of the target
sub-block OR an exposed `$import` alias of that target. This applies
symmetrically to all four kinds (types, nomenclatures, formats, computes).

Re-export is subject to the same visibility rules as direct exposure
(§E.4.5). Cycles in alias chains MUST cause schema loading to fail.

**Example:** an intermediate lib `region` re-exports `common.Address` as
`Addr`; a consumer `app` imports `&region.Addr` and resolves transparently
to `common.Address`.

---

## E.5 External Reference Syntax

### E.5.1 In `$import` Only

External references use the syntax:

```
&schemaId.DefinitionName
```

**Examples:**

```json
"$defs": {
  "$import": {
    "Address": "&common.Address",
    "Customer": "&crm.Customer",
    "Invoice": "&fr.billing.Invoice"
  }
}
```

**Structure:**

- `&` - reference prefix
- `schemaId` - the `$id` of the target schema (before the `.`)
- `DefinitionName` - the definition name within that schema's `$defs` (after the `.`)

### E.5.2 Restriction

External reference syntax (`&schemaId.Name`) is **only valid in `$import`**.

Inside `$oky` and `$defs`, all references MUST use local names:

```json
// ✅ Correct - local alias
"address | $ref": "&Address"

// ❌ Invalid - external syntax not allowed in $oky
"address | $ref": "&common.Address"
```

This restriction ensures that:

- all external dependencies are explicitly declared in `$import`
- the schema body remains independent of external naming
- refactoring external imports only affects `$import`, not the entire schema

---

## E.6 Registry Resolution

### E.6.1 Registry Role

The reference implementation maintains a **schema registry** that maps `($id, $version)` to Okyline schema documents.

### E.6.2 Resolution Algorithm

Constraint matching follows **strict semver ordering**. A pre-release
version is, by semver, strictly less than its stable counterpart at the
same `{major, minor, patch}` (e.g. `1.3.0-rc.1 < 1.3.0`). Consequently,
`~`, `^`, and exact constraints **do not match pre-releases of the base
{major, minor, patch}** unless the constraint itself names a pre-release
(see E.6.4 for the explicit opt-in form). This aligns Okyline with the
mainstream package-manager convention (npm, Cargo, Maven).

When resolving a dependency, the resolver performs a two-pass selection:

1. **Pass 1 - Stable preferred.** Among matching versions, selects the
   **highest stable version** (no pre-release suffix). If found, this is
   the resolved version regardless of consumer stability.
2. **Pass 2 - Pre-release fallback (gated).** Only if no stable version
   matches AND the consumer is itself a pre-release (the **F.5 stability
   contagion rule**, §E.6.3) OR the constraint explicitly names a
   pre-release, the resolver selects the **highest pre-release version**
   matching the constraint.

If neither pass yields a candidate, the resolution fails and the schema MUST
be rejected.

### E.6.3 Pre-release Fallback (F.5 Stability Contagion)

A schema whose own `$version` carries a pre-release suffix is itself
unstable. To support development workflows where consumer and dependency
schemas evolve in parallel pre-release cycles, an unstable consumer is
allowed to fall back to a matching pre-release dependency (Pass 2 above).
A stable consumer is **not** so allowed: it sees only stable matches,
unless the constraint itself opts in to pre-releases.

The resolution table below illustrates both consumer modes:

| Constraint | Available in registry | Stable consumer | Pre-release consumer |
|------------|-----------------------|-----------------|----------------------|
| `"2.0.0"` | `2.0.0` | `2.0.0` | `2.0.0` |
| `"~1.3.0"` | `1.3.0`, `1.3.1`, `1.3.2-beta` | `1.3.1` | `1.3.1` |
| `"~1.3.0"` | `1.3.0`, `1.3.5-rc.1` | `1.3.0` | `1.3.0` |
| `"~1.3.0"` | `1.3.5-rc.1`, `1.3.5-rc.2` | *no match* | `1.3.5-rc.2` |
| `"~1.3.0"` | `1.3.0-rc.1`, `1.3.0-rc.2` | *no match* | *no match* |
| `"^2.0.0"` | `2.0.0`, `2.1.0`, `2.2.0-rc.1` | `2.1.0` | `2.1.0` |

Note the last two rows: pre-releases of the **constraint base patch**
(e.g. `1.3.0-rc.1` for `~1.3.0`) are **not** matched even by an unstable
consumer, because they rank below the base in strict semver order. To
allow them, the consumer must opt in explicitly via the constraint syntax
described in §E.6.4.

#### Relation to Mainstream Package Managers

Okyline's resolution combines two regimes:

- **For pre-releases of the constraint base patch** (e.g. `1.3.0-rc.1`
  matching `~1.3.0`): Okyline is **strict**, aligned with npm, Cargo and
  Maven. The consumer must opt in explicitly via a pre-release suffix
  in the constraint (§E.6.4).
- **For pre-releases of higher patches** (e.g. `1.3.5-rc.1` matching
  `~1.3.0`): Okyline is **more permissive than npm/Cargo** when the
  consumer is itself a pre-release. Such pre-releases are picked up
  automatically as a fallback (Pass 2) without requiring a flag or an
  explicit pre-release suffix in the constraint. Mainstream package
  managers would require an opt-in (e.g. npm's `--include-prereleases`).

This combination is intentional: it preserves predictable resolution for
released schemas (a stable consumer never accidentally pulls in a
pre-release dep), while keeping in-flight development workflows
ergonomic (an unstable consumer can iterate against the latest
pre-release patches without continuously rewriting its `$deps`).

### E.6.4 Explicit Pre-release Opt-in

To target a base patch that has not yet released as stable, the consumer
writes the constraint with a pre-release suffix:

```
"$deps": { "lib": "~1.3.0-rc.0" }
```

Such a constraint matches:

- any pre-release of the base patch with a suffix `>= rc.0` per semver
  ordering (e.g. `1.3.0-rc.1`, `1.3.0-rc.2`),
- the eventual stable `1.3.0`,
- and any higher patch in the `1.3.x` train.

This makes the intent explicit (the consumer knows it is targeting an
unreleased version) and removes the ambiguity that would otherwise come
with implicit pre-release fallback for stable-base constraints.

### E.6.5 Resolution Guarantees

- Consumers always receive the most recent compatible version available
- Transition from pre-release to stable is automatic and transparent
- The resolved version is determined at schema load time
- All references MUST be resolvable at schema load time - unresolved references MUST cause schema loading to fail

---

## E.7 Interaction with Annex D

### E.7.1 Unified Namespace

Once imported via `$import`, external definitions join the same namespace as `$defs` entries:

```
Available definitions = $defs ∪ $import
```

All Annex D mechanisms work identically on both:

| Mechanism | Internal (`$defs`) | External (`$import`) |
|-----------|-------------------|---------------------|
| `$ref` (property-level) | ✓ | ✓ |
| `$ref` (object-level) | ✓ | ✓ |
| `$override` | ✓ | ✓ |
| `$remove` | ✓ | ✓ |

### E.7.2 Name Collision Rule

A definition name MUST NOT appear as both a local entry of `$defs` and an
alias inside `$defs.$import`:

```json
// ❌ Invalid - collision
{
  "$oky": { },
  "$defs": {
    "Address": { "city|@": "Paris" },
    "$import": {
      "Address": "&common.Address"
    }
  }
}
```

If a collision occurs, the schema MUST be rejected at load time.

### E.7.3 Transitive Dependencies

External schemas may themselves have dependencies. The registry is responsible for resolving the full dependency graph.

**Rules:**

- Circular dependencies between schemas are **forbidden**. Implementations using a
  manual registry - where consumers must register all transitive dependencies before
  themselves - prevent cycles by construction (the registration order itself enforces
  a topological dependency graph). Dynamic registries that load schemas on demand
  (filesystem, HTTP, etc.) MUST detect cycles at load time.
- Multiple versions of the same `$id` MAY coexist in the resolved dependency graph,
  provided each consumer's `$deps` constraint resolves unambiguously to a single
  version. This supports incremental migration scenarios where a consumer schema and
  its transitive dependencies pin different versions of a shared library.
- The resolution strategy (e.g., flat vs. nested) is implementation-defined

---

# E.8 Resolution Scope Rules (All Root Blocks)

This section defines the **global resolution scope rules** for Okyline.  
These rules apply uniformly to **all reusable elements** defined or imported by a schema:

- type and object definitions (`$defs`, `$import`)
- nomenclatures (`$nomenclature`)
- formats (`$format`)
- computed expressions (`$compute`)

The goal is to guarantee **deterministic behavior**, **strict version isolation**,  
and **predictable inheritance semantics**, even in the presence of external imports  
and object-level inclusion.

---

## E.8.1 Owner Scope of a Definition

Every reusable element in Okyline belongs to exactly one **owner contract scope**:

```
(scope) = ($id, $version)
```

- A **local element** belongs to the scope of the current contract.
- An **imported element** belongs to the scope of the external contract
  resolved via `$deps`.

This rule applies uniformly to all element kinds:

- `$defs`
- `$nomenclature`
- `$format`
- `$compute`

The owner scope of an element **never changes implicitly**.

External owner scopes are introduced only through explicit `$import` imports
(`$import` inside `$defs`, `$nomenclature`, `$format`, or `$compute`).
`$deps` declarations do not create visibility by themselves.


---

## E.8.2 Global Namespace Rules for Type Definitions

Type and object definitions participate in a **single global namespace per contract**.

Accordingly:

- Importing **type or object definitions** via `$import` is only valid
  inside the **`$defs` block** (`$defs.$import`). A `$import` block declared
  at the root of the schema document MUST cause schema loading to fail (E.4.3).
- `$defs.$import` contributes to the global type namespace:

```
Available types = $defs (local entries) ∪ $defs.$import (imported aliases)
```

Any `$import` declared inside `$nomenclature`, `$format`, or `$compute`:

- imports **only symbols of that specific kind**,
- does **not** introduce type or object definitions,
- does **not** affect the global type namespace.

---

## E.8.3 Reference Resolution Principle

Any reference to a reusable name:

- `&TypeName`
- `($NOMENCLATURE)`
- `~$FORMAT~`
- `(%COMPUTE)`

is always resolved **within the owner scope of the element that contains the reference**.

References are:

- **never resolved globally**, and
- **never rebound implicitly** to the consuming contract.

---

## E.8.4 Resolution in Local Definitions

For any element declared in the current contract:

```json
"$nomenclature": {
  "COUNTRY": "FR,DE,ES"
}
```

or:

```json
"$defs": {
  "Address": {
    "country|@ ($COUNTRY)": "FR"
  }
}
```

All references inside the element are resolved in the  
**current contract’s owner scope**.

---

## E.8.5 Resolution in Imported Definitions (`$import`)

When a definition is imported via `$defs.$import`:

```json
"$deps": { "crm": "2.0.1" },
"$defs": { "$import": { "Customer": "&crm.Customer" } }
```

the imported definition:

- belongs to scope `crm@2.0.1`,
- retains its original semantics,
- resolves **all its internal references**
  (`$nomenclature`, `$format`, `$compute`, nested `$defs`)
  strictly within the `crm@2.0.1` scope.

The consuming contract **does not alter** this behavior.

---

## E.8.6 Object-Level Inclusion (`$ref`) and Field Injection

When object-level inclusion is used:

```json
"Order": {
  "$ref": "&Customer",
  "orderId|@": "ORD-1"
}
```

the implementation performs **field injection**.

### Normative Rule

> **Injected fields retain their original owner scope.**

Consequences:

- A field originating from `crm@2.0.1`
  continues to resolve its nomenclatures, formats and computes
  in the `crm@2.0.1` scope.
- Local elements with identical names in the consuming contract
  do **not** override or affect injected fields.

This guarantees that importing a type preserves the exact behavior
defined and tested in its original contract version.

---

## E.8.7 Overrides and Scope Rebinding

`$override` is the **only mechanism** that changes resolution scope.

Example:

```json
"country | $override @ ($COUNTRY)": "FR"
```

### Semantics

- The inherited field definition is fully replaced.
- The overridden field becomes **local** to the current contract.
- All references inside the overridden definition
  are resolved in the **current contract scope**.

---

## E.8.8 Local Additions

Any field added locally (not inherited) is:

- owned by the current contract scope,
- resolved using the current contract’s
  `$nomenclature`, `$format`, and `$compute`.

---

## E.8.9 Imported Nomenclatures, Formats and Computes

When a contract imports shared resources explicitly:

```json
"$deps": { "geo": "1.4.0" },
"$nomenclature": {
  "$import": { "COUNTRY": "&geo.COUNTRY" }
}
```

the imported name:

- becomes a **local alias**,
- but its **owner scope remains** `geo@1.4.0`.

Thus:

```json
"country|@ ($COUNTRY)": "FR"
```

resolves against `geo.COUNTRY`, not a local definition.

The same rule applies to `$format.$import` and `$compute.$import`.

**Numeric precision of imported computes.** When a compute is imported via
`$compute.$import` (or inherited through a field of an imported type), the
validator evaluates it with the **source contract's** `$decimalScale` - not
the consumer's. The library author chose that precision for a domain reason
(often financial); silently inheriting the consumer's scale would cause
incorrect rounding.

**Opt-in.** This scale switch happens only when the source contract has
explicitly declared `$decimalScale`. A contract that does not declare one
is treated as "no opinion" and inherits the active scale of the validation.

**Example.** A tax library at `$decimalScale: 6` defines
`round(amount * 0.1965, 6)`. When imported by a consumer at
`$decimalScale: 2`, the multiplication still runs at scale 6, preserving the
4 fractional digits needed for accurate fiscal rounding before final
cent-level display.

---

## E.8.10 No Implicit Cross-Scope Rebinding

The following behaviors are **explicitly forbidden**:

- resolving an imported element’s references
  against the consuming contract’s nomenclatures, formats or computes,
- implicitly aligning identically named elements
  across different contracts,
- mixing resolution scopes during inclusion.

Any such behavior would break determinism and testability.

---

## E.8.11 Summary of Scope Rules

| Situation | Resolution Scope |
|---------|------------------|
| Local definition | Current contract |
| Imported type definition (`$defs.$import`) | Source contract |
| Injected field via `$ref` | Source contract |
| Overridden field (`$override`) | Current contract |
| Local field addition | Current contract |
| Imported nomenclature / format / compute | Source contract |
| `$decimalScale` for imported compute | Source contract if defined |

## E.9 Summary

| Feature | Description |
|---------|-------------|
| `$id` | Schema identifier for registry publication and external references |
| `$version` | Schema version following semver format |
| `$state` | Lifecycle state of the contract (`"DRAFT"` or `"FINAL"`, default `"DRAFT"`) |
| `$entries` | Additional validation entry points (public name → local type reference) |
| `$deps` | Declares external dependencies with version constraints (`~`, `^`, exact) |
| `$import` | Creates local aliases for external definitions |
| `$public` / `$private` | Per-sub-block visibility - controls what is exposable to `$import` import |
| `&schemaId.Name` | External reference syntax (only in `$import`) |
| `&Name` | Local reference syntax (in `$oky`, `$defs`) - resolves in `$defs` ∪ `$import` |
| Re-export | A `$import` alias is a valid import target - intermediate libs can re-export |
| Registry resolution | Highest compatible version, with pre-release fallback |
| Transitive dependencies | Resolved by registry; cycles forbidden; multiple versions of the same `$id` MAY coexist |

### Conformance Levels

| Level | Annexes | Capabilities |
|-------|---------|--------------|
| **Core** | D only | Internal composition (`$defs`, `$ref`, `$override`, `$remove`) |
| **Full** | D + E | Core + external imports and versioning |

---

## E.10 Complete Example

### E.10.1 Shared Schema: `common` v1.2.0

```json
{
  "$id": "common",
  "$version": "1.2.0",
  "$title": "Common Definitions",

  "$oky": { },

  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com",

    "DateTime|~$DateTime~": "2025-01-01T00:00:00Z",

    "Address": {
      "street|@ {5,100}": "123 Main Street",
      "city|@ {2,50}": "Paris",
      "postalCode|@ {5,10}": "75001",
      "country|@ {2}": "FR"
    },

    "Auditable": {
      "createdAt|@ $ref": "&DateTime",
      "updatedAt|@ $ref": "&DateTime"
    }
  }
}
```

### E.10.2 Consumer Schema: `ecommerce` v1.0.0

```json
{
  "$id": "ecommerce",
  "$version": "1.0.0",
  "$title": "E-Commerce Order Schema",

  "$oky": {
    "order": {
      "$ref": "&Auditable",
      "orderId|@ # ~$OrderId~": "ORD-12345678",
      "customerEmail | $ref @": "&Email",
      "status|@ ($ORDER_STATUS)": "PENDING",
      "items | $ref @ [1,100]": ["&OrderItem"],
      "shippingAddress | $ref @": "&Address",
      "billingAddress | $ref": "&Address",
      "total|@ (%OrderTotal)": 100.00,

      "$requiredIf status('SHIPPED','DELIVERED')": ["trackingNumber"],
      "trackingNumber|{10,50}": "TRACK123456"
    }
  },

  "$defs": {
    "$import": {
      "Email": "&common.Email",
      "Address": "&common.Address",
      "Auditable": "&common.Auditable"
    },
    "OrderItem": {
      "sku|@ # {5,20}": "SKU-12345",
      "name|@ {2,200}": "Product Name",
      "quantity|@ (1..1000)": 1,
      "unitPrice|@ (>0)": 10.00
    }
  },

  "$format": {
    "OrderId": "^ORD-[0-9]{8}$"
  },

  "$compute": {
    "ItemTotal": "unitPrice * quantity",
    "OrderTotal": "sum(items, %ItemTotal)"
  },

  "$nomenclature": {
    "ORDER_STATUS": "PENDING,CONFIRMED,SHIPPED,DELIVERED,CANCELLED"
  },

  "$deps": {
    "common": "~1.2.0"
  }
}
```

### E.10.3 Resolution Flow

1. Registry resolves `common: ~1.2.0` → finds `common@1.2.3` (highest patch)
2. `$import` aliases are bound:
   - `&Email` → `common@1.2.3.$defs.Email`
   - `&Address` → `common@1.2.3.$defs.Address`
   - `&Auditable` → `common@1.2.3.$defs.Auditable`
3. Schema body uses `&Email`, `&Address`, `&Auditable` as if they were local
4. All Annex D composition rules apply normally

---

**End of Annex E - External Imports and Versioning (Normative)**
