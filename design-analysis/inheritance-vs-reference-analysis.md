# IFC 5 Inheritance vs. Reference: USD Alignment Analysis

## Executive Summary

IFC 5 uses an `inherits` mechanism in its IFCX JSON format that is **inspired by USD but not strictly following USD composition arc semantics**. While the Python converter (`ifc4-to-usda.py`) correctly uses USD's native `GetInherits().AddInherit()` composition arc, the IFCX JSON schema implements a **custom inheritance pattern** that serves IFC 5's specific needs.

## USD Composition Arcs Primer

USD (Universal Scene Description) defines several **composition arcs** with different strength levels:

| Composition Arc | Strength | Purpose | Use Case |
|----------------|----------|---------|----------|
| **Local Opinions** | Strongest | Direct values on prim | Override everything |
| **Variant Selection** | Very Strong | Choose between variants | Configuration selection |
| **References** | Strong | Include external content | Asset reuse, file composition |
| **Payload** | Strong | Lazy-loadable references | Performance optimization |
| **Inherits** | Weak | Class-based inheritance | Type/instance pattern |
| **Specializes** | Weakest | Variation from base | Specialized versions |

### References vs. Inherits in USD

**References (`references`):**
- Bring external content into a **new namespace**
- Strong composition - referenced data appears "as authored"
- Used for **file-level composition** and **asset reuse**
- Cannot be easily overridden by the referencing prim
- Example: Including a whole building model in a site

**Inherits (`inherits`):**
- Establish **class-like relationships**
- Weaker composition - local opinions can override
- Used for **type-instance patterns**
- Supports polymorphism and templating
- Example: Window instances inheriting from IfcWindowType

---

## IFC 5 Implements Inheritance

### 1. IFCX JSON Schema Definition

From `ifcx.tsp`:
```typescript
@pattern("</[A-Za-z0-9_/.:]+>")
scalar path extends string;

model IfcxNode {
    path: path;                          // GUID or Hierarchical path
    children?: Record<string | null>;    // Named child relationships
    inherits?: Record<string | null>;    // Inheritance relationships
    attributes?: Record<unknown>;        // Properties and data
}
```

**Key Innovation:** The `path` pattern allows **two addressing modes**:
- **GUID**: `"592504dc-469a-44d6-9ae8-c801b591679b"` (traditional)
- **Hierarchical**: `"Root/My_Project/Site/Building/Floor/Wall/Window"` (revolutionary!)

This dual-addressing is **one of the biggest differentiations from IFC4**. See [notes.md](./notes.md#path-concept-ifc4-vs-ifc5) for detailed comparison.

### 2. Actual Usage in hello-wall.ifcx

**Type Definition (IfcWindowType):**
```json
{
  "path": "25503984-6605-43a1-8597-eae657ff5bea",
  "children": {
    "Void": "8fada721-cff8-590b-8d0b-9300b5fe8e18",
    "Frame": "08f06095-3f32-55b9-a353-61c9aca5cc4d",
    "Glazing": "5ad6f475-c04c-5628-8b9d-75d0bab0c0e5"
  },
  "attributes": {
    "bsi::ifc::class": {"code": "IfcWindow", ...},
    "bsi::ifc::prop::Volume": 0.026,
    "bsi::ifc::prop::Height": 1.2
  }
}
```

**Instance Inheriting from Type:**
```json
{
  "path": "2c2d549f-f9fe-4e22-8590-562fda81a690",
  "inherits": {
    "windowType": "25503984-6605-43a1-8597-eae657ff5bea"
  },
  "attributes": {
    "usd::xformop": {
      "transform": [[1,0,0,0], [0,1,0,0], [0,0,1,0], [1.767,0,1,1]]
    }
  }
}
```

**Material Inheritance:**
```json
{
  "path": "93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b",  // Wall
  "inherits": {
    "material": "7a187a90-3dcf-58cc-b3a6-51a9a407c55a"  // Concrete material
  }
}
```

### 3. Composition Implementation

From `compose.ts`:
```typescript
Object.values(input.inherits).forEach((inheritPath) => {
    let classNode = ComposeNodeFromPath(GetHead(inheritPath), nodes);
    let subnode = GetChildNodeWithPath(classNode, GetTail(inheritPath));
    
    // Copy children from inherited node
    subnode.children.forEach((child, childName) => {
        node.children.set(childName, child);
    })
    
    // Copy attributes from inherited node
    for (let [attrID, attr] of subnode.attributes) {
        node.attributes.set(attrID, attr);
    }
});
```

**Key behaviors:**
- Inherited children are **copied** to the instance
- Inherited attributes are **copied** to the instance  
- Later attributes **override** earlier ones (last-wins)
- Supports path-based inheritance: `"inherit1": "class/child/subchild"`

### 4. USD Conversion

From `ifc4-to-usda.py`:
```python
# When converting IFCX to USD, uses actual USD inherits:
for ty in types[el]:
    xf.GetPrim().GetInherits().AddInherit(ty)

# Creates class prims for types:
xf = stage.CreateClassPrim(Sdf.Path(path_str))

# Instances inherit from class:
stage.DefinePrim(instance_path).GetInherits().AddInherit(class_path)
```

---

## Comparison: IFC 5 vs. USD

| Aspect | IFC 5 IFCX | USD Inherits | Alignment? |
|--------|------------|--------------|------------|
| **Semantics** | Type-instance pattern | Class-based inheritance | ✅ **Aligned** |
| **Strength** | Configurable (last-wins) | Weaker than references | ⚠️ **Different** |
| **Override Behavior** | Later definitions win | Local opinions override | ✅ **Similar** |
| **Child Propagation** | Yes (children copied) | Yes (namespace inherited) | ✅ **Aligned** |
| **Attribute Propagation** | Yes (attributes copied) | Yes (properties inherited) | ✅ **Aligned** |
| **Path-based Inheritance** | Yes (`class/a/b/c`) | No (only prim-level) | ❌ **Extension** |
| **Multi-inheritance** | Yes (multiple inherits) | Yes (multiple inherits) | ✅ **Aligned** |
| **Named Inheritance** | Yes (`"windowType": guid`) | No (unnamed list) | ❌ **Extension** |

---

## Key Differences from USD

### 1. Named Inheritance Relationships

**IFC 5:**
```json
{
  "inherits": {
    "windowType": "type-guid",
    "material": "material-guid"
  }
}
```

**USD:**
```python
# Unnamed list of inheritance targets
prim.GetInherits().AddInherit(path1)
prim.GetInherits().AddInherit(path2)
```

**Analysis:** IFC 5 adds **semantic naming** to inheritance relationships, making it clear WHY something is inherited. This is an **intentional extension** beyond USD.

### 2. Path-Based Inheritance

**IFC 5 allows inheriting from sub-paths:**
```json
{
  "inherits": {
    "detail": "windowType/Frame/Detail"
  }
}
```

This inherits properties from a **nested child** of the type, not just the type itself.

**USD does not support this** - you can only inherit from top-level prims.

**Analysis:** This is a **powerful extension** that supports more granular reuse patterns.

### 3. Inheritance vs. Children

IFC 5 makes a **clear distinction**:

- **`children`**: Spatial/compositional relationships (part-of, contains)
  - Example: Wall → Window (wall contains windows)
  - Example: Space → Body (space has geometry)

- **`inherits`**: Type-instance and material relationships
  - Example: Window instance → WindowType (is-a relationship)
  - Example: Wall → Concrete material (has-material relationship)

**USD also makes this distinction**, but uses different mechanisms:
- Children: Scene graph hierarchy
- Inherits: Class inheritance
- **References**: File composition and asset inclusion

---

## Should Material Be Inheritance or Reference?

### Current Implementation (Inheritance)

```json
{
  "path": "93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b",  // Wall
  "inherits": {
    "material": "7a187a90-3dcf-58cc-b3a6-51a9a407c55a"
  }
}
```

### USD Standard Approach (Material Binding)

In USD, materials are typically handled via **material binding**, not inheritance:

```python
# USD standard pattern:
UsdShade.MaterialBindingAPI.Apply(prim)
material = UsdShade.Material.Define(stage, '/Materials/Concrete')
UsdShade.MaterialBindingAPI(prim).Bind(material)
```

### Analysis

| Approach | Pros | Cons |
|----------|------|------|
| **Inheritance** (current) | - Simple and consistent<br>- Reuses existing mechanism<br>- Material properties propagate like other attributes | - Not how USD typically does it<br>- Mixing different semantic relationships<br>- Material is not really "is-a" |
| **Reference** | - Stronger relationship<br>- Clearer intent<br>- More aligned with asset management | - Adds another relationship type<br>- More complex composition<br>- Need reference strength rules |
| **Dedicated Binding** | - Most aligned with USD<br>- Clear semantics<br>- Industry standard | - Another mechanism to learn<br>- More schema complexity |

**Recommendation:** The current **inheritance approach is acceptable** for IFC 5 alpha, but consider adding a dedicated **material binding relationship** in future versions to better align with USD and industry standards.

---

## Verdict: Is IFC 5 Following USD Standards?

### ✅ Correctly Aligned

1. **Type-Instance Pattern**: Using inheritance for type→instance is correct USD usage
2. **Attribute Overriding**: Local opinions override inherited values - matches USD
3. **Class Prims**: Using class-like definitions for types - correct USD pattern
4. **Composition Strength**: Inherits weaker than direct attributes - matches USD intent
5. **USD Export**: The Python converter correctly uses `GetInherits().AddInherit()`

### ❌ Intentional Extensions Beyond USD

1. **Named Inheritance**: Adding semantic names ("windowType", "material") 
2. **Path-Based Inheritance**: Inheriting from nested children (`type/a/b`)
3. **JSON Representation**: Custom JSON schema vs. USD's native formats (.usda, .usdc)

### ⚠️ Potential Misalignments

1. **Material as Inheritance**: Should consider dedicated material binding
2. **Missing Reference Arc**: No equivalent to USD's `references` for file composition
   - Currently using `imports` + `children`, which is different
3. **No Payload Concept**: No lazy-loading mechanism like USD payloads

---

## Recommendations

### For IFC 5 Development

1. **Keep Current Approach**: The inheritance mechanism works well for IFC 5's needs
2. **Document as Extension**: Clearly state that IFC 5 extends USD concepts while maintaining compatibility
3. **Consider Adding**:
   - Explicit `references` relationship for strong composition (complement `inherits`)
   - Material binding relationship type (future version)
   - Payload-like mechanism for performance (future version)

### Best Practices

When deciding **inheritance vs. children**:

| Relationship | Use |
|--------------|-----|
| **`inherits`** | Type/instance, material assignment, shared properties |
| **`children`** | Spatial containment, part decomposition, geometry components |
| **`ref` (future)** | File composition, external asset inclusion, strong dependencies |

### Documentation Needs

1. Explain that IFC 5 uses **USD-inspired** inheritance, not pure USD
2. Document the **named inheritance** extension
3. Clarify **path-based inheritance** semantics
4. Provide guidance on when to use `inherits` vs. `children`

---

## Conclusion

**IFC 5's inheritance mechanism is fundamentally aligned with USD's philosophy** of using weak inheritance for type-instance patterns. The implementation **correctly maps to USD's native `inherits` composition arc** during export.

However, IFC 5 makes **intentional, valuable extensions**:
- Named inheritance relationships (better semantics)
- Path-based inheritance (more granular reuse)
- Simplified JSON representation (better for BIM workflows)

These extensions **do not violate USD principles** - they enhance them for IFC 5's specific domain needs. The approach is **pragmatic and appropriate** for an alpha-stage schema that needs to serve the BIM industry while maintaining interoperability with USD-based tools.

**The current approach should be maintained**, with future consideration for adding dedicated material binding and reference relationships to enable even richer composition patterns.

---

## Real-World Example: Typical Floor Repetition

### Case Study: add-2nd-storey.ifcx

The [add-2nd-storey.ifcx](../examples/Hello Wall/advanced/add-2nd-storey.ifcx) file provides a **textbook example** of IFC 5's inheritance power for typical floor modeling.

#### The Problem

In real buildings, floors often repeat the same layout:
- Same walls, windows, doors
- Same spaces and boundaries
- Same materials and properties
- Different vertical position

**Traditional approach:** Copy all geometry and properties for each floor
- Results in massive file sizes
- Difficult to maintain consistency
- Changes must be synchronized across all copies
- Error-prone

#### The IFC 5 Solution

Using inheritance, the entire 2nd storey is defined in **just 3 nodes**:

```json
{
  "data": [
    // Step 1: Add 2nd storey to building hierarchy
    {
      "path": "e84dc79e-fe9d-4781-9f4b-54dd435cca91",  // My_Building
      "children": {
        "My_2nd-Storey": "44af358b-3160-4063-8a89-a868335ff2nd"
      }
    },
    
    // Step 2: Inherit ALL content from 1st storey
    {
      "path": "44af358b-3160-4063-8a89-a868335ff2nd",
      "inherits": {
        "original-storey-inherit": "44af358b-3160-4063-8a89-a868335ff3b5"
      }
    },
    
    // Step 3: Override ONLY the vertical position
    {
      "path": "44af358b-3160-4063-8a89-a868335ff2nd",
      "attributes": {
        "usd::xformop": {
          "transform": [
            [1, 0, 0, 0],
            [0, 1, 0, 0],
            [0, 0, 1, 0],
            [0, 0, 3, 1]    // 3 meters vertical offset
          ]
        }
      }
    }
  ]
}
```

#### What Gets Inherited Automatically

The 2nd storey inherits the **complete composition** from the 1st storey:

```
Original My_Storey (44af358b-3160-4063-8a89-a868335ff3b5)
├── My_Space (e3035b71-bd9f-4cdc-86fd-b56e2f4605b6)
│   ├── Body (space geometry)
│   ├── Boundary_Wall (space boundary object)
│   ├── Boundary_Window (space boundary object)
│   └── Boundary_Window_001 (space boundary object)
└── Wall (93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b)
    ├── Body (wall geometry with openings)
    ├── Axis, Directrix, Basis
    ├── Window (2c2d549f-f9fe-4e22-8590-562fda81a690)
    │   └── inherits from WindowType (with Void, Frame, Glazing)
    └── Window_001 (592504dc-469a-44d6-9ae8-c801b591679b)
        └── inherits from WindowType

All of this (~2000 lines of JSON) is inherited via 1 line:
"inherits": {"original-storey-inherit": "44af358b-...ff3b5"}
```

#### The Results

| Metric | Without Inheritance | With Inheritance |
|--------|---------------------|------------------|
| **File Size** | ~2,000 lines × 2 storeys = 4,000 lines | 50 lines total |
| **Maintainability** | Change wall → Update 2 places | Change wall → Auto-updates both |
| **Consistency** | Manual synchronization required | Guaranteed identical |
| **Design Changes** | High risk of errors | Single source of truth |
| **Performance** | Duplicate geometry loading | Shared geometry reference |

#### Scalability

This pattern scales brilliantly:

**10-storey building:**
```json
// First storey: Full definition
{"path": "storey-1", "children": {...}}  // 2000 lines

// Storeys 2-10: Inherit + offset
{"path": "storey-2", "inherits": {"typical": "storey-1"}}
{"path": "storey-2", "attributes": {"transform": [0,0,3,1]}}

{"path": "storey-3", "inherits": {"typical": "storey-1"}}
{"path": "storey-3", "attributes": {"transform": [0,0,6,1]}}

// ... etc. Each additional storey: ~50 lines instead of 2000
```

**Space savings:** From 20,000 lines → ~2,500 lines (87.5% reduction!)

#### Advanced Patterns Enabled

**Variation by Override:**
```json
// Penthouse with different window
{
  "path": "storey-penthouse",
  "inherits": {"typical": "storey-1"}
},
{
  "path": "storey-penthouse/Wall/Window",
  "inherits": {"windowType": "luxury-window-type"}  // Override specific window
}
```

**Building Typologies:**
```json
// Building A: Full definition
{"path": "building-A", "children": {...}}

// Buildings B, C, D: Same design, different locations
{"path": "building-B", "inherits": {"design": "building-A"}}
{"path": "building-B", "attributes": {"location": [100, 0, 0]}}
```

#### Alignment with USD

This usage **perfectly demonstrates** USD's inherit composition arc:

1. **Class-like Templates**: 1st storey acts as a "class"
2. **Instancing**: 2nd storey is an "instance" with overrides
3. **Weak Composition**: Transform override wins over inherited position
4. **Namespace Propagation**: All children come along automatically
5. **Efficient Memory**: Shared geometry and attributes

#### Business Value

For the BIM industry:

- **Reduces file sizes** by 80-90% for typical buildings
- **Accelerates design** - define once, replicate infinitely
- **Ensures consistency** - single source of truth
- **Simplifies coordination** - changes propagate automatically
- **Enables parametric design** - vary overrides for custom floors
- **Supports modular construction** - reuse building components across projects

#### Conclusion

The `add-2nd-storey.ifcx` example demonstrates why IFC 5's inheritance model is **revolutionary for BIM**:

- Simple to author (3 nodes)
- Powerful composition (2000+ lines inherited)
- Industry-standard pattern (USD-aligned)
- Real-world applicability (typical floors)
- Scalable to large projects (skyscrapers, campuses)

This is **not just a technical feature** - it's a **fundamental paradigm shift** in how BIM data can be authored, managed, and exchanged. The inheritance model transforms IFC from a verbose, redundant format into an efficient, maintainable, composable data structure.

---

**Document Version:** 1.1  
**Date:** February 8, 2026  
**Author:** Technical Analysis  
**Status:** For Review and Discussion
