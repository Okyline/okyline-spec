---
description: External imports and versioning in Okyline - cross-schema composition with schema identity, versioned dependencies, and registry-based imports.
---

# Annex E: External Imports and Versioning - (*in progress*)

**Version:** 1.4.0
**Date:** April 2026
**Status:** Draft

 License: This annex is part of the Open Okyline Language Specification and is subject to the same license terms (CC
  BY-SA 4.0). See the Core Specification for full license details.

This annex extends Annex D to enable **cross-schema composition** in enterprise environments. It defines schema identity (`$id`, `$version`), versioned dependencies (`$deps`), and external imports (`$xDefs`). Organizations can publish schemas to a registry and import definitions from other contracts while maintaining strict version control and compatibility guarantees.

---

## Relation to the Core Specification

This annex **extends Annex D** by adding support for:

- schema identity and versioning (`$id`, `$version`)
- versioned dependencies (`$deps`)
- external definition imports (`$xDefs`)
- registry-based resolution

Annex D defines the core mechanisms for structural composition (`$ref`, `$defs`, `$override`, `$remove`). This annex enables those same mechanisms to operate **across schema boundaries** through versioned imports.

Implementations MAY support Annex D without Annex E, providing internal composition only.
Since Annex E—External Imports and Version Management—has not yet been finalized, full compliance with Okyline 1.4.0 does not require Annex E.

---

## Overview

In enterprise environments, schemas often need to:

- share common definitions across multiple contracts
- evolve independently while maintaining compatibility
- declare explicit dependencies with version constraints

Okyline addresses these needs through a **two-layer architecture**:

1. **Annex D (internal)** — `$defs` + `$ref` for composition within a single document
2. **Annex E (external)** — `$xDefs` + `$deps` for importing definitions from other schemas

External definitions, once imported via `$xDefs`, behave **exactly like internal definitions** in `$defs`. The composition mechanisms (`$ref`, `$override`, `$remove`) work identically regardless of origin.

---

## Specification Status

Annex E — External Imports and Versioning — is at an advanced stage. Its publication is expected in the coming months, following final compatibility and consistency validation.

---

**End of Annex E — External Imports and Versioning (Normative)**
