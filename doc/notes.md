# Notes for IFC 5

## Notes

- The customdata node in the schema is not like the other industrial standards or schemas such as USD schema. It is a custom schema defined by users or developers.

### Space boundary

- In IFC4, space boundary is a relationship. In IFC5, space boundary is defined as an object. One advantage is that you can add attributes to it.

**Actual Hierarchy in hello-wall.ifcx:**

```
My_Storey (44af358b-3160-4063-8a89-a868335ff3b5)
├── My_Space (e3035b71-bd9f-4cdc-86fd-b56e2f4605b6) [IfcSpace]
│   ├── Body (geometry of the space)
│   ├── Boundary_Wall (c8ecbf4c-e37a-4489-9133-15163b8a904e) [Space Boundary Object]
│   │   ├── Body (boundary surface geometry)
│   │   └── Attributes:
│   │       └── bsi::ifc::spaceBoundary:
│   │           ├── relatedelement: ref → Wall (93791d5d...)
│   │           └── relatingspace: ref → My_Space (e3035b71...)
│   ├── Boundary_Window (cc40ef6e-950e-4b79-a802-4cc710294e3a) [Space Boundary Object]
│   │   ├── Body (boundary surface geometry)
│   │   └── Attributes:
│   │       └── bsi::ifc::spaceBoundary:
│   │           ├── relatedelement: ref → Window (2c2d549f...)
│   │           └── relatingspace: ref → My_Space
│   └── Boundary_Window_001 (7f2a3284-bf01-4b0a-85e4-e48d92ba56dd) [Space Boundary Object]
│       └── (similar structure)
└── Wall (93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b) [IfcWall]
    ├── Body (wall geometry with window openings)
    ├── Window (2c2d549f...)
    └── Window_001 (592504dc...)
```

**Key Points:**
1. **My_Space is NOT a space boundary** - it's an IfcSpace object
2. **Space boundary objects** are separate entities (Boundary_Wall, Boundary_Window, etc.) that are **children of the space**
3. **Wall and Space are siblings** - both children of My_Storey, not hierarchically nested
4. Each space boundary object:
   - Has its own GUID and is a first-class object
   - Contains a `Body` child with the boundary surface geometry
   - Has a `bsi::ifc::spaceBoundary` attribute that **references** both:
     - `relatedelement`: The building element (wall/window) being bounded
     - `relatingspace`: The space being bounded
5. **Advantages of object-based approach:**
   - Can add attributes directly to the boundary (geometry, properties, metadata)
   - Space boundary has its own identity and lifecycle
   - Can have child elements (like Body geometry)
   - More flexible than IFC4's pure relationship approach

### Material layer set

- In principal, they don't exist in IFC5 because everything is defined as an object. See double-wall.ifcx file for example.

### Inheritance for Typical Floors

- The [add-2nd-storey.ifcx](../examples/Hello Wall/advanced/add-2nd-storey.ifcx) demonstrates **inheritance-based floor repetition**
- **Pattern**: Create 2nd storey that inherits ALL content from 1st storey, then override only the vertical position
- **Result**: 2000+ lines of storey definition inherited via single `inherits` statement
- **Benefits**:
  - Massive file size reduction (87.5% for 10-storey building)
  - Single source of truth for typical floor design
  - Changes to 1st storey automatically propagate to all upper floors
  - Enables variation by override (e.g., penthouses with custom windows)
- **Real-world application**: Typical floors in office towers, hotels, residential buildings
- See detailed analysis in [inheritance-vs-reference-analysis.md](./inheritance-vs-reference-analysis.md#real-world-example-typical-floor-repetition)

---

## Issues

- I can't find the schema file which said to be located at ifc5.technical.buildingsmart.org/schemas/bsi/ifc/v5a/schema.json

---

## Path Concept: IFC4 vs. IFC5

**One of the biggest differentiations between IFC4 and IFC5 is the concept of `path`.**

### Two Ways to Reference Objects in IFC5

From the schema definition (`ifcx.tsp`):
```typescript
@pattern("</[A-Za-z0-9_/.:]+>")
scalar path extends string;
```

The pattern allows **slashes (`/`)**, enabling two distinct addressing modes:

#### 1. **GUID Path** (Direct Reference)

Traditional GUID-based addressing:
```json
{
  "path": "592504dc-469a-44d6-9ae8-c801b591679b",  // Window_001
  "inherits": {
    "windowType": "25503984-6605-43a1-8597-eae657ff5bea"
  }
}
```

- **What**: Direct reference by globally unique identifier
- **Pros**: Globally unique, persistent across versions
- **Cons**: Not human-readable, no context
- **Same as IFC4**: Yes, traditional approach

#### 2. **Hierarchical Path** (Navigation Path)

Navigate through the spatial/compositional hierarchy:
```json
{
  "path": "ab143723-f7b1-5368-b106-55896e88d768/My_Project/My_Site/My_Building/My_Storey/Wall/Window_001",
  "attributes": {
    "bsi::ifc::v5a::prop::isexternal": false
  }
}
```

- **What**: Path-based navigation through named relationships
- **Pros**: Human-readable, shows context, self-documenting
- **Cons**: Can break if hierarchy changes
- **New in IFC5**: This is the revolutionary part!

### The Two Paths Reference THE SAME Object

**GUID path:**
```
592504dc-469a-44d6-9ae8-c801b591679b
```

**Hierarchical path:**
```
ab143723-f7b1-5368-b106-55896e88d768/My_Project/My_Site/My_Building/My_Storey/Wall/Window_001
```

Both identify **Window_001 in the wall** - just different addressing schemes!

### Example: just-1-property.ifcx

The [just-1-property.ifcx](../examples/Hello Wall/advanced/just-1-property.ifcx) demonstrates adding a single attribute using hierarchical path:

```json
{
  "data": [
    {
      "path": "ab143723.../My_Project/My_Site/My_Building/My_Storey/Wall/Window_001",
      "attributes": {
        "bsi::ifc::v5a::prop::isexternal": false
      }
    }
  ]
}
```

This adds the `isexternal` property to Window_001 by **navigating the hierarchy**.

### Path Resolution Rules

1. **GUID-only path**: `"592504dc-469a-44d6-9ae8-c801b591679b"` → Direct lookup
2. **Hierarchical path**: `"root/child1/child2"` → Navigate relationships
3. **Mixed path**: Can inherit from hierarchical path: `"inherits": {"detail": "windowType/Frame/Detail"}`

### IFC4 vs. IFC5 Comparison

| Aspect | IFC4 | IFC5 |
|--------|------|------|
| **Object Identity** | GUID only | GUID or Hierarchical Path |
| **Addressing** | `#123=IfcWindow('GUID',...)` | `"path": "Root/.../Window"` |
| **Navigation** | Must traverse relationships | Built into path string |
| **Human Readability** | Very poor | Excellent with paths |
| **Context** | Lost without traversal | Embedded in path |
| **Flexibility** | Single addressing mode | Multiple addressing modes |

### Why This Matters

**Hierarchical paths enable:**

1. **Self-Documenting Files**: Path shows exactly where object lives
   ```json
   "path": "Building-A/Floor-2/Apartment-201/LivingRoom/Window"
   ```

2. **Easier Authoring**: No need to look up GUIDs
   ```json
   // Add property to specific window by location
   {"path": "MyBuilding/Floor1/Wall/Window", "attributes": {...}}
   ```

3. **Path-Based Inheritance**: Inherit from nested elements
   ```json
   {"inherits": {"frame": "WindowType/Frame/DetailA"}}
   ```

4. **Query-Like Operations**: Address multiple objects
   ```json
   // Conceptually could support patterns like:
   "path": "Building/*/Wall/Window"  // All windows in all floors
   ```

5. **Overlay Files**: External files can target specific locations
   ```json
   // Energy analysis file adds properties to specific room
   {"path": "Project/Site/Building/Floor2/Room201", "attributes": {...}}
   ```

### Best Practices

**Use GUID paths when:**
- Object identity must be globally unique and persistent
- Cross-file references that must survive hierarchy changes
- Type definitions and materials (don't belong to spatial hierarchy)

**Use Hierarchical paths when:**
- Human authoring/editing files
- Adding properties to spatially located objects
- Documentation and communication
- Inheriting from nested sub-components

### Technical Implementation

The schema allows both because `path` accepts:
- **Alphanumeric + hyphens**: GUIDs like `592504dc-469a-44d6-...`
- **Forward slashes**: Hierarchy like `Root/Child/GrandChild`
- **Colons and dots**: Namespaces like `org.bsi:building/floor`

**Pattern**: `@pattern("</[A-Za-z0-9_/.:]+>")`

This flexibility is **revolutionary for BIM** - combining the robustness of GUIDs with the usability of hierarchical addressing!

