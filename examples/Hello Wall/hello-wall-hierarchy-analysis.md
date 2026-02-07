# Hello Wall IFCX File Structure Analysis

## Overview

This document provides a comprehensive analysis of the `hello-wall.ifcx` file structure, explaining the hierarchy, entities, and decomposition patterns used in this IFC5 example.

## File Structure Fundamentals

### Graph-Based Data Model

The IFCX format uses a **graph-based structure** where:
- Each node is uniquely identified by a **GUID** (stored in the `path` property)
- Nodes define **named relationships** to children via the `children` object
- Relationship names are **semantically meaningful** (e.g., "Body", "Window", "My_Project")
- A single GUID can appear **multiple times** in the data array:
  - Once to define its children
  - Additional times to add different sets of attributes
  - This allows modular composition of entity properties

### Entities vs Properties

**Entities** (also called **nodes**):
- Are separate objects with their own GUIDs
- Referenced in the `children` section
- Can have their own geometry, attributes, and children
- Example: `"Window": "2c2d549f-f9fe-4e22-8590-562fda81a690"`

**Properties** (also called **attributes**):
- Stored in the `attributes` section
- Attached directly to an entity
- Include metadata, classifications, geometry, and visual properties
- Example: `"bsi::ifc::prop::Height": 3.0`

---

## Complete Spatial Hierarchy

```
ROOT: ab143723-f7b1-5368-b106-55896e88d768
 │
 └─ My_Project → 14adb22b-d474-48a2-8e8f-6d4c067c1953 (IfcProject)
     │
     └─ My_Site → e0834921-e095-40f0-8874-3c6bd1ec699e (IfcSite)
         │
         └─ My_Building → e84dc79e-fe9d-4781-9f4b-54dd435cca91 (IfcBuilding)
             │
             └─ My_Storey → 44af358b-3160-4063-8a89-a868335ff3b5 (IfcBuildingStorey)
                 │
                 ├─ My_Space → e3035b71-bd9f-4cdc-86fd-b56e2f4605b6 (IfcSpace)
                 │   ├─ Body → 15289df1-5ae0-5fc9-8399-19de1fbb87a0
                 │   │   └─ [Mesh geometry defining space volume]
                 │   ├─ Boundary_Wall → c8ecbf4c-e37a-4489-9133-15163b8a904e
                 │   │   ├─ Body → 911155b7-f688-51ee-8e3e-b97475be2452
                 │   │   └─ [Space boundary relationship to Wall]
                 │   ├─ Boundary_Window → cc40ef6e-950e-4b79-a802-4cc710294e3a
                 │   │   ├─ Body → f3aa8991-7481-5d6b-8dae-f98c372f4a98
                 │   │   └─ [Space boundary relationship to Window #1]
                 │   └─ Boundary_Window_001 → 7f2a3284-bf01-4b0a-85e4-e48d92ba56dd
                 │       ├─ Body → 126781f3-6465-5c39-9216-81cc636eeb39
                 │       └─ [Space boundary relationship to Window #2]
                 │
                 └─ Wall → 93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b (IfcWall)
                     ├─ Body → 634f90c3-831e-5f29-a9b2-fa69b207821e
                     │   └─ [Mesh geometry with two window openings]
                     ├─ Axis → 8407e490-ceaa-56e5-96df-2351d9110668
                     │   └─ [Curve from (0,0,0) to (10,0,0)]
                     ├─ Directrix → 9d1fce89-e179-5076-9a3b-1b40eef3524b
                     │   └─ [Curve from (0,0,0) to (0,0,3)]
                     ├─ Basis → 367f7f7d-b4c5-50fe-829a-6bbced170dd1
                     │   └─ [Mesh defining wall base surface]
                     ├─ Window → 2c2d549f-f9fe-4e22-8590-562fda81a690 (IfcWindow)
                     │   └─ [Inherits from windowType: 25503984-6605-43a1-8597-eae657ff5bea]
                     └─ Window_001 → 592504dc-469a-44d6-9ae8-c801b591679b (IfcWindow)
                         └─ [Inherits from windowType: 25503984-6605-43a1-8597-eae657ff5bea]
```

---

## Entity Details

### Root Entity
- **GUID**: `ab143723-f7b1-5368-b106-55896e88d768`
- **Purpose**: Top-level container for the project
- **Children**: Links to "My_Project"

### Project Entity (IfcProject)
- **GUID**: `14adb22b-d474-48a2-8e8f-6d4c067c1953`
- **IFC Class**: IfcProject
- **Name**: "My Project"
- **Children**: Links to "My_Site"

### Site Entity (IfcSite)
- **GUID**: `e0834921-e095-40f0-8874-3c6bd1ec699e`
- **IFC Class**: IfcSite
- **Name**: "My Site"
- **Children**: Links to "My_Building"

### Building Entity (IfcBuilding)
- **GUID**: `e84dc79e-fe9d-4781-9f4b-54dd435cca91`
- **IFC Class**: IfcBuilding
- **Name**: "My Building"
- **Children**: Links to "My_Storey"

### Building Storey Entity (IfcBuildingStorey)
- **GUID**: `44af358b-3160-4063-8a89-a868335ff3b5`
- **IFC Class**: IfcBuildingStorey
- **Name**: "My Storey"
- **Children**: Links to "My_Space" and "Wall"

### Space Entity (IfcSpace)
- **GUID**: `e3035b71-bd9f-4cdc-86fd-b56e2f4605b6`
- **IFC Class**: IfcSpace
- **Name**: "My Space"
- **Volume**: 120.0 m³
- **Height**: 3.0 m
- **Visual Properties**:
  - Diffuse color: RGB(0.6, 0.7, 0.8)
  - Opacity: 0.3 (semi-transparent)
- **Transform**: Translated by 0.1 units in Y-axis
- **Children**: Body, Boundary_Wall, Boundary_Window, Boundary_Window_001

### Wall Entity (IfcWall)
- **GUID**: `93791d5d-5beb-437b-b8ec-2f1f0ba4bf3b`
- **IFC Class**: IfcWall
- **Name**: "Wall"
- **Volume**: 2.784 m³
- **Height**: 3.0 m
- **Properties**:
  - IsExternal: true
  - Material: Inherits from material entity (CONCRETE)
  - Classification: NLSFB 21.21
- **Children**: Body, Axis, Directrix, Basis, Window, Window_001

### Window Instances
Both windows inherit from the same window type.

**Window #1**:
- **GUID**: `2c2d549f-f9fe-4e22-8590-562fda81a690`
- **IFC Class**: IfcWindow
- **IsExternal**: true
- **Transform**: Positioned at (1.767, 0, 1)
- **Inherits**: windowType → `25503984-6605-43a1-8597-eae657ff5bea`

**Window #2**:
- **GUID**: `592504dc-469a-44d6-9ae8-c801b591679b`
- **IFC Class**: IfcWindow
- **IsExternal**: true
- **Transform**: Positioned at different location
- **Inherits**: windowType → `25503984-6605-43a1-8597-eae657ff5bea`

---

## Window Type Decomposition Pattern

### Window Type Entity
- **GUID**: `25503984-6605-43a1-8597-eae657ff5bea`
- **IFC Class**: IfcWindow (used as a type/template)
- **Type Name**: "WT01"
- **Volume**: 0.026 m³
- **Height**: 1.2 m

### The Three Decomposed Components

The window type is **decomposed** into three separate **entities** (not properties):

```
IfcWindowType: 25503984-6605-43a1-8597-eae657ff5bea
 │
 ├─ Void → 8fada721-cff8-590b-8d0b-9300b5fe8e18
 │   ├─ Type: Entity (separate node with its own GUID)
 │   ├─ Geometry: Mesh (0.9m × 1.2m × 1.2m box)
 │   ├─ Purpose: Defines the opening/void space for the window
 │   └─ Visibility: Invisible (not rendered visually)
 │
 ├─ Frame → 08f06095-3f32-55b9-a353-61c9aca5cc4d
 │   ├─ Type: Entity (separate node with its own GUID)
 │   ├─ Geometry: Mesh (complex frame structure)
 │   ├─ Purpose: Physical window frame component
 │   └─ Material: WOOD (referenced from material entity)
 │
 └─ Glazing → 5ad6f475-c04c-5628-8b9d-75d0bab0c0e5
     ├─ Type: Entity (separate node with its own GUID)
     ├─ Geometry: Mesh (0.7m × 1.0m × 0.01m glass pane)
     └─ Purpose: Transparent glass/glazing component
```

### Why These Are Entities, Not Properties

1. **Each has its own GUID**: Independent identity in the graph
2. **Each has its own geometry**: Separate mesh definitions with vertices and face indices
3. **Each appears in `children` section**: Referenced as child entities, not stored in `attributes`
4. **Each can have its own attributes**: Materials, visibility, transformations

### The Inheritance Mechanism

When a window **instance** (like `2c2d549f-f9fe-4e22-8590-562fda81a690`) inherits from the window type:

```json
{
  "path": "2c2d549f-f9fe-4e22-8590-562fda81a690",
  "inherits": {
    "windowType": "25503984-6605-43a1-8597-eae657ff5bea"
  }
}
```

It inherits:
- The entire decomposition structure (Void, Frame, Glazing)
- All three geometric components
- Properties defined on the type
- The relationship names and structure

The instance can then override or add:
- Its own position (`usd::xformop::transform`)
- Instance-specific properties
- Additional children or relationships

---

## Key Architectural Patterns

### 1. Decomposition Pattern
Complex objects (like windows, walls) are broken down into constituent parts, each represented as a separate entity with its own geometry and properties.

### 2. Inheritance Pattern
Type definitions (templates) are created once and reused by multiple instances, promoting consistency and reducing redundancy.

### 3. Named Relationships
Parent-child relationships have semantic names ("Body", "Axis", "Window") rather than generic array indices, making the structure self-documenting.

### 4. Multi-Aspect Attribution
A single entity can have multiple attribute blocks added incrementally, allowing separation of concerns (geometry, classification, properties, etc.).

### 5. Space Boundaries
Explicit boundary entities connect spaces to their bounding elements (walls, windows), enabling energy analysis and other simulations.

---

## Material Entities

### Concrete Material
- **GUID**: `7a187a90-3dcf-58cc-b3a6-51a9a407c55a`
- **Code**: CONCRETE
- **URI**: https://identifier.buildingsmart.org/uri/fish/midas-materials/26/class/CONCRETE
- **Visual Properties**:
  - Diffuse color: RGB(0.5, 0.5, 0.5)
  - Opacity: 1.0 (fully opaque)
- **Used by**: Wall entity (via inheritance)

### Wood Material
- **GUID**: `4549bada-a37e-5044-bb70-456516cca5a8`
- **Code**: WOOD
- **URI**: https://identifier.buildingsmart.org/uri/fish/midas-materials/26/class/WOOD
- **Visual Properties**:
  - Diffuse color: RGB values for wood appearance
- **Used by**: Window frame component

---

## Geometry Types

The file uses several USD geometry primitives:

1. **`usd::usdgeom::mesh`**: Polygonal mesh with face vertex indices and points
2. **`usd::usdgeom::basiscurves`**: Line segments defined by points
3. **`usd::xformop`**: Transformation matrices for positioning and orientation
4. **`usd::usdgeom::visibility`**: Visibility control (visible/invisible)

---

## Summary

The `hello-wall.ifcx` file demonstrates:

- **Spatial hierarchy**: Root → Project → Site → Building → Storey → Space/Wall
- **Geometric decomposition**: Windows broken into Void, Frame, and Glazing entities
- **Type-instance pattern**: One window type reused by two window instances
- **Material inheritance**: Wall and window frame reference material definitions
- **Space boundaries**: Explicit relationships between spaces and enclosing elements
- **Multi-representation**: Entities can have body geometry plus axis, directrix, and basis curves

This structure enables rich semantic modeling while maintaining clear separation between spatial structure, geometric representation, material properties, and relationships.
