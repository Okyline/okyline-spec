# Okyline Language Specification

![Spec Status](https://img.shields.io/badge/spec-Draft_1.7.0-blue)
![Spec License](https://img.shields.io/badge/spec_license-CC--BY--SA%204.0-green)
![Maintained by](https://img.shields.io/badge/maintained_by-Akwatype-orange)

![Okyline Studio Free](https://img.shields.io/badge/Okyline_Studio_Free-online-black)

---

## Overview

**Okyline is a declarative, example-driven language for describing and validating JSON data**. 

An Okyline schema is a real example of data, enriched with built-in constraints. 
It remains readable while serving as an enforceable contract. 
A single Okyline contract expresses both the data structure 
(types, presence, enumerations, lengths, models, key uniqueness) 
and its cross-field business invariants 
(consistency, conditional requirements, exact decimal arithmetic, and ordering constraints) 
through a pure and deterministic expression language. 
It also defines version management and cross-schema composition 
(semantic versions, dependencies, imports, visibility), 
so that contracts can be shared and evolved across teams.

This repository hosts the **official Okyline Language Specification** (version 1.7.0, May 2026).

---

# 1. Okyline Language Specification

These are the **official reference documents** for Okyline 1.7.0:

- **Core Specification** - [Okyline-Core-Language-Specification-v1.7.0.md](./Okyline-Core-Language-Specification-v1.7.0.md)
- **Annex C - Expression Language** - [Okyline-Annex-C-Expression-language-v1.7.0.md](./Okyline-Annex-C-Expression-language-v1.7.0.md)
- **Annex D - Internal Schema References** - [Okyline-Annex-D-Internal-References-v1.7.0.md](./Okyline-Annex-D-Internal-References-v1.7.0.md)
- **Annex E - External Imports and Versioning** - [Okyline-Annex-E-External-Imports-v1.7.0.md](./Okyline-Annex-E-External-Imports-v1.7.0.md)
- **Annex F - Virtual Fields** - [Okyline-Annex-F-Virtual-Fields-v1.7.0.md](./Okyline-Annex-F-Virtual-Fields-v1.7.0.md)

> Quick references and user guides are companion (non-normative) material, published on the documentation hub: https://community.okyline.design-hub.okyline.io/

### License (Specification)

The Okyline Language Specification is published under:

**Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**  
https://creativecommons.org/licenses/by-sa/4.0/

---

# 2. Okyline Studio Free

![Okyline Studio Free](https://img.shields.io/badge/Okyline_Studio_Free-online-black)

**Okyline Studio Free** fully implements the Okyline 1.7.0 specification and provides:

- Visual Okyline schema editor
- Instant documentation generation
- Automatic JSON Schema transpilation
- Interactive JSON validation against a selected Okyline schema

**Try it online:**  
https://community.studio.okyline.io/

All free Okyline resources (guides, documentation, studio) are available at:  
https://community.okyline.design-hub.okyline.io/

---

# Contact

https://www.akwatype.io  
pierre-michel.bret@akwatype.io

© 2025-2026 Akwatype - Okyline® and Akwatype® are registered trademarks of Akwatype.
