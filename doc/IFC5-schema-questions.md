# IFC5 Schema Questions for Developers

Questions about the IFC5 IFCX multi-file composition model that require clarification.

---

## Question 1: Loading Order Determinism

Given "last value wins" merge strategy, different loading orders produce different results:

```json
// fire-rating.ifcx
{"path": "wall-guid", "attributes": {"FireRating": "R60"}}

// overrides.ifcx
{"path": "wall-guid", "attributes": {"FireRating": "R30"}}
```

Load order `[fire-rating, overrides]` → R30  
Load order `[overrides, fire-rating]` → R60

**Questions:**

1. How should loading order be specified and enforced (manifest file, import chains, metadata)?
2. What is the standard way to package and distribute multi-file models to ensure consistency?
3. Should there be deterministic tie-breakers beyond load order (timestamps, file names, priority levels)?

---

## Question 2: Deletion Semantics

How do you delete entities or attributes added in another file?

**Scenarios:**

```json
// base.ifcx
{"path": "wall-guid", "attributes": {"cost::EstimatedPrice": 5000}}
{"path": "wall-guid", "children": {"Window1": "w1", "Window2": "w2"}}

// update.ifcx - Need to:
// 1. Remove the cost attribute entirely
// 2. Remove Window2 from children
// 3. Delete a temporary scaffold entity
```

**Questions:**

1. Is deletion supported? If so, how is it represented in the JSON structure (e.g., `"delete": true`, null values, special paths)?
2. How should cascading deletions work (parent deleted → what happens to children and references)?
3. Are operations atomic? What's the order: deletes first, then adds?

---

## Question 3: Multi-File Validation

Validation changes from single-file to multi-file composition:

**Scenario:**

```json
// geometry.ifcx - Has geometry, missing required IFC properties
{"path": "wall-guid", "attributes": {"usd::usdgeom::mesh": {...}}}

// properties.ifcx - Completes the entity
{"path": "wall-guid", "attributes": {"bsi::ifc::class": {"code": "IfcWall"}}}
```

Validating `geometry.ifcx` alone → FAIL (missing required properties)  
Validating `properties.ifcx` alone → FAIL (no geometry)  
Validating merged result → PASS

**Questions:**

1. What is the scope of validation: individual files, merged result, or both?
2. Can files be marked as "partial" or "overlay" to bypass standalone validation?
3. How does IDS work with distributed data across multiple files?
4. Should there be a "validation manifest" declaring which files to load and in what order?

---

## Question 4: Custom Schema Definition Conflicts

What happens when two files define a custom schema with the same name but different rules?

**Scenario:**

```json
// fileA.ifcx
"schemas": {
  "customdata": {
    "value": {
      "dataType": "Object",
      "objectRestrictions": {
        "values": {
          "originalStepInstance": {"dataType": "String", "optional": true},
          "cost": {"dataType": "Number", "optional": false}  // Required number
        }
      }
    }
  }
}

// fileB.ifcx
"schemas": {
  "customdata": {
    "value": {
      "dataType": "Object",
      "objectRestrictions": {
        "values": {
          "originalStepInstance": {"dataType": "Number", "optional": true},  // Different type!
          "scheduleCode": {"dataType": "String", "optional": true}  // Different fields!
        }
      }
    }
  }
}
```

**Questions:**

1. How are schema definitions merged when multiple files define the same schema name? Are schemas merged (union of all fields) or does "last wins" apply to entire schema definitions?
2. What happens when one schema marks a field as required but another marks it optional?
3. Should schema definitions include versioning or namespacing to avoid conflicts?
