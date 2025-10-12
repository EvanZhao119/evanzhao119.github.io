---
layout: post
title: "Stable Names & Task Contracts with JSON Schema: Prevent Breaking Changes and Ship Faster"
date: 2025-10-12
categories: ai
published: true
description: "Learn how Stable Names and JSON Schema–based Task Contracts keep APIs backward compatible, prevent breaking changes, and let teams evolve services safely."
tags: [Stable Name, Task Contract, JSON Schema, API Stability, Backward Compatibility, Microservices]
---

# Stable Names + Task Contracts: Keep Callers Stable While You Evolve Internals

_A practical guide with a tiny JSON Schema, validation tips, and evolution rules—using `report.generate` as the example._


## 1 Background and Problem
In fast-moving teams, services evolve often. If callers bind to ad-hoc fields or specific implementations, upgrades break downstream systems.

**The fix:** To give each reusable capability a **Stable Name** (e.g., `report.generate`) and define a **Task Contract** with **JSON Schema**. The stable name is the unchanging entry point; the contract is the machine-checkable definition of inputs/outputs/errors. Internals can change freely as long as they continue to honor the contract.

---

## 2 Core Ideas
- **Stable Name**: A permanent entry point (e.g., `report.generate`). Implementations, versions, or engines can change without breaking the interface callers use.
- **Task Contract (JSON Schema)**: A single source of truth describing input/output/error shape, required fields, enums, defaults, and constraints.
- **Backward-Compatible Evolution**: Prefer additive changes (new optional fields, relaxed constraints). Use versioning (e.g., `v2`) for breaking changes and provide a migration window.

---

## 3 Minimal Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "report.generate",
  "type": "object",
  "properties": {
    "period": { "type": "string", "pattern": "^\\d{4}-\\d{2}$" },
    "sourceIds": { "type": "array", "minItems": 1, "items": { "type": "string" } },
    "outputFormat": { "type": "string", "enum": ["pdf", "docx", "md"], "default": "pdf" }
  },
  "required": ["period", "sourceIds"]
}
```
- **Object shape**: The request must be a JSON object.
- **`period` (required)**: Must match `YYYY-MM` (e.g., `2025-09`) to lock reporting granularity.
- **`sourceIds` (required)**: A non-empty array of stable source IDs.
- **`outputFormat` (optional)**: One of `pdf | docx | md`. The default is `pdf`.
- **Examples**
  - Valid: `{"period":"2025-09","sourceIds":["sales"],"outputFormat":"pdf"}`
  - Invalid: `{"period":"2025/09","sourceIds":[]}` (wrong date format; empty source list)
- **Extension hooks (when needed)**: Add `additionalProperties:false` to prevent drift, and use `$defs.Success / $defs.Error` to standardize responses for output validation.

---

## 4 Why This Works
- **No downstream breakage**: As long as the implementation respects the schema, callers don’t change.
- **Early incompatibility detection**: Schema validation and schema-diff checks catch breaking changes in dev, CI, or at the gateway before they hit production.
- **Consistent error semantics**: Standard error codes.
- **Lower integration cost**: The schema serves as documentation and powers type/SDK/mock generation.

---

## 5 Compatibility and Evolution Rules
**Backward-compatible (allowed)**
- Add **optional** fields.
- **Extend** enums (e.g., later add `"html"` to `outputFormat`).
- Relax constraints (e.g., `patterns`, `minItems`, etc.).
- Add non-breaking output fields (e.g., `warnings`, `bytes`, `checksum`).

**Potentially breaking (version it)**
- Remove fields or make optional fields required.
- Shrink enums or tighten constraints.
- Change a field’s meaning.
- Restructure key objects.

---

**Takeaway:** Pin the entry point with a **Stable Name** and pin the boundaries with a **Task Contract**. Validate in dev/CI/gateway. Evolve additively by default, version when necessary. You’ll move fast internally without toppling your callers.