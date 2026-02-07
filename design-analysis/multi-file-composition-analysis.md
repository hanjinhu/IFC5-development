# Multi-File Composition Design Analysis

## Overview

The IFCX format's GUID-based graph structure enables powerful multi-file composition capabilities, allowing models to be split across multiple files and overlaid together based on GUID relationships. This document analyzes the advantages and potential issues of this design pattern.

## Evidence in Repository

The Hello Wall examples demonstrate this pattern:

- **Base Model** ([hello-wall-1st-part.ifcx](../examples/Hello Wall/hello-wall-1st-part.ifcx)): Defines spatial hierarchy and wall geometry
- **Property Overlays**:
  - [hello-wall-add-fire-rating-30.ifcx](../examples/Hello Wall/hello-wall-add-fire-rating-30.ifcx): Adds `FireRating: "R30"`
  - [hello-wall-add-fire-rating-60.ifcx](../examples/Hello Wall/hello-wall-add-fire-rating-60.ifcx): Adds `FireRating: "R60"`
- **Extended Model** ([hello-wall-2nd-part.ifcx](../examples/Hello Wall/hello-wall-2nd-part.ifcx)): Adds space boundaries and analytical data

All reference the same wall entity: `93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b`

---

## Key Advantages

### 1. Modularity
Separate disciplines (architecture, MEP, structural, energy analysis) can work on different aspects in isolated files without conflicts.

### 2. Versioning & Change Tracking
Easy to track what changed by comparing small overlay files rather than monolithic models.

### 3. Conditional Loading
Applications can load only relevant data (e.g., skip fire ratings for geometric viewers, skip detailed geometry for schedulers).

### 4. Incremental Updates
Add properties or relationships without modifying original files, preserving source data integrity.

### 5. Multi-Tenant Collaboration
Different organizations can contribute data to the same model without accessing each other's proprietary files.

### 6. Fine-Grained Access Control
Sensitive data (cost, security ratings) can be in separate files with restricted access.

---

## Potential Issues

### Issue #1: Merge Conflict Resolution ✅ RESOLVED

**Problem:**
```json
// FileA.ifcx adds:
{"path": "93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b", "attributes": {"FireRating": "R30"}}

// FileB.ifcx adds:
{"path": "93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b", "attributes": {"FireRating": "R60"}}
```

When multiple files add conflicting values to the same attribute on the same entity, which value should be used?

**Official Resolution:**
**The IFC5 schema developers have defined that "last value wins"** - the most recently loaded/merged file takes precedence.

**Implications:**
- Load order matters and must be deterministic
- Intentional overrides are supported (e.g., project-specific values override defaults)
- Silent overwrites possible without warnings (design trade-off for simplicity)
- Predictable behavior across all compliant tools

**Best Practices:**
- Document intended load order in manifest files
- Use naming conventions to indicate override intent (e.g., `base-model.ifcx`, `overrides.ifcx`)
- Implement optional logging/auditing to track what values were replaced
- Consider import order carefully when composing models

---

### Issue #2: Reference Integrity

**Problem:**
```json
// FileA.ifcx creates:
{"path": "wall-guid", "children": {"Window": "window-guid"}}

// FileB.ifcx references window-guid but window definition is missing or deleted
```

Cross-file references can break if target entities are not available.

**Consequences:**
- Dangling references to non-existent entities
- Orphaned entities with no parent
- Application crashes or undefined rendering behavior
- Hard to detect broken links until post-merge validation

**Mitigation Strategies:**
- Comprehensive post-merge validation tools
- Dependency graph validation before import
- Soft references with graceful degradation
- Required vs. optional reference annotations
- Manifest files declaring all required entities

---

### Issue #3: Circular Dependencies

**Problem:**
```json
// FileA.ifcx:
{"imports": ["FileB.ifcx"]}

// FileB.ifcx:
{"imports": ["FileA.ifcx"]}
```

Import chains can create circular dependencies.

**Consequences:**
- Infinite import loops
- Tooling hangs or crashes
- Ambiguous merge order
- Memory exhaustion

**Mitigation Strategies:**
- Topological sort of import graph with cycle detection
- Maximum import depth limits
- Explicit error reporting for circular imports
- Import once semantics with caching

---

### Issue #4: Schema Version Conflicts

**Problem:**
```json
// FileA uses:
"bsi::ifc::v5a::prop::IsExternal": true

// FileB uses:
"bsi::ifc::v5b::prop::IsExternal": "YES"
```

Same entity with attributes from incompatible schema versions.

**Consequences:**
- Property name collisions with different semantics
- Type mismatches (boolean vs. string)
- Validation failures
- Data corruption during merge

**Mitigation Strategies:**
- Schema versioning compatibility matrix
- Automatic migration/translation layers
- Reject incompatible schema mixing
- Per-attribute schema version tagging
- Semantic versioning with backward compatibility rules

---

### Issue #5: GUID Collision Risk

**Problem:**
Though astronomically unlikely with proper UUID generation, two independent parties might generate identical GUIDs for different entities.

**Consequences:**
- Accidentally merging unrelated entities
- Data corruption
- Namespace pollution in distributed scenarios
- Security vulnerabilities in malicious scenarios

**Mitigation Strategies:**
- Organizational GUID prefixes or namespacing
- GUID registry/allocation service for critical projects
- Collision detection during merge
- Cryptographic verification for high-stakes projects
- Use UUID v4 or v7 with proper entropy sources

---

### Issue #6: Performance and Loading Order

**Problem:**
```json
// Geometry in File10.ifcx, but classification in File1.ifcx
// What gets loaded first? How long is the entity incomplete?
```

Data for single entity scattered across many files with arbitrary load order.

**Consequences:**
- Viewers display incomplete/incorrect data during progressive loading
- Memory overhead from maintaining partial entity states
- Complex caching and state management
- User confusion from temporarily invalid states
- Performance degradation from multiple file seeks

**Mitigation Strategies:**
- Parallel loading with deferred rendering
- Loading priority hints in file metadata
- Smart buffering and batch processing
- Progressive enhancement pattern (show basic, then enrich)
- Dependency-aware load orchestration

---

### Issue #7: Deletion Semantics

**Problem:**
How do you delete an entity or property that was added in another file?

**Consequences:**
- No clear way to remove inherited/imported data
- Can only add, never subtract
- Workarounds like override-with-null are ambiguous
- Partial deletion (remove one attribute but keep others) undefined

**Mitigation Strategies:**
- Explicit tombstone markers: `{"path": "guid", "delete": true}`
- Attribute-level deletion: `{"path": "guid", "attributes": {"prop": null}}`
- Last-file-wins convention with clear documentation
- Deletion log/manifest files
- Immutable append-only model with validity timestamps

---

### Issue #8: Validation Timing

**Problem:**
When should validation occur? Per-file or post-merge?

**Scenario:**
```json
// FileA.ifcx has incomplete entity (missing required properties)
// FileB.ifcx completes it
// FileA alone would fail validation
```

**Consequences:**
- Per-file validation: Incomplete entities always fail (false negatives)
- Post-merge only: Late error detection, poor developer experience
- Unclear which file is "wrong" when validation fails
- Tooling confusion about valid states

**Mitigation Strategies:**
- Two-phase validation: relaxed per-file + strict post-merge
- Required vs. optional property annotations
- Partial entity markers indicating completion expected elsewhere
- Validation profiles (strict/lenient modes)
- Clear error messages indicating which file + which merge caused failure

---

### Issue #9: Discoverability

**Problem:**
Given one IFCX file, how do you know what other files are needed for complete model?

**Consequences:**
- Missing critical data without realizing it
- Incomplete models appear valid
- Manual tracking of related files error-prone
- Version mismatches between related files
- Onboarding friction for new collaborators

**Mitigation Strategies:**
- Explicit `imports` array (already exists)
- Manifest/index files listing all components
- Metadata describing optional vs. required dependencies
- Tooling to discover and suggest related files
- Naming conventions for file relationships
- Project-level configuration files

---

### Issue #10: Ambiguous Merge Semantics for Collections

**Problem:**
```json
// FileA.ifcx:
{"path": "wall", "children": {"Window": "w1"}}

// FileB.ifcx:
{"path": "wall", "children": {"Window": "w2"}}
```

Should "Window" be `w1`, `w2`, or both? Different semantics needed for different cases.

**Consequences:**
- Named children might replace when concatenation intended
- Arrays might concatenate when replacement intended
- Inconsistent behavior across different property types
- Tool-specific interpretations leading to incompatibility

**Mitigation Strategies:**
- Clear documentation of merge behavior per schema
- Different operators: `children_add`, `children_replace`, `children_merge`
- Arrays concatenate by default, named references replace by default
- Explicit merge strategies in schema definitions
- Validation warnings on ambiguous cases

---

## Recommended Solutions Framework

### 1. Specification Level
- **Formal Merge Semantics**: Document exact behavior for conflicts, collections, deletions
- **Schema Versioning Policy**: Semantic versioning with compatibility rules
- **Standard Operations**: Define add/update/delete/replace primitives
- **Validation Levels**: Specify when and what to validate

### 2. File Format Enhancements
- **Merge Metadata**: `"mergeStrategy": "last-wins" | "error-on-conflict" | "merge-deep"`
- **Precedence Hints**: `"priority": 100` for conflict resolution
- **Change Operations**: `"operation": "add" | "update" | "delete"`
- **Dependency Declarations**: Enhanced `imports` with version constraints

### 3. Tooling Requirements
- **Import Graph Validator**: Detect cycles, missing dependencies
- **Conflict Detector**: Warn about attribute conflicts pre-merge
- **Reference Checker**: Validate GUID references across files
- **Merge Simulator**: Preview merge results before committing
- **Diff Tools**: Show what each overlay contributes

### 4. Best Practices
- **Manifest Files**: Central registry of component files for projects
- **Naming Conventions**: Clear file naming indicating purpose/discipline
- **GUID Namespacing**: Organizational prefixes for large collaborations
- **Minimal Overlays**: Keep overlay files focused and atomic
- **Documentation**: README files explaining file relationships

---

## Conclusion

The GUID-based multi-file composition pattern is a powerful feature that enables flexible, modular collaboration at scale. However, it requires:

1. **Clear specifications** for merge behavior and conflict resolution
2. **Robust tooling** for validation, dependency management, and merge operations
3. **Strong conventions** for project organization and file management
4. **Comprehensive testing** of edge cases and failure modes

With proper implementation of these solutions, the compositional design can transform BIM collaboration while maintaining data integrity and consistency.

---

**Document Version:** 1.0  
**Date:** February 7, 2026  
**Author:** Design Analysis  
**Status:** Draft for Discussion
