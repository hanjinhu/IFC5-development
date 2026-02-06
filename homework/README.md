# Homework Submission - IFC5 Implementers Assembly

**Author**: Hanjin Hu  
**Date**: February 6, 2026  
**Assignment**: Add a light fixture layer to the HelloWall model

## File

- `hello-wall-add-light-fixture.ifcx` - Layer file adding a ceiling-mounted light fixture

## What Was Added

This layer adds a simple ceiling-mounted light fixture to the My_Storey level of the HelloWall model.

### Light Fixture Details:
- **Type**: IfcLightFixture
- **Name**: Ceiling Light
- **Geometry**: Simple cubic shape (0.4m × 0.4m × 0.2m extruded solid)
- **Position**: Located at coordinates (2.5, 0.0, 2.5) at ceiling height
- **Properties**:
  - Power Rating: 60W
  - Manufacturer: Generic Lighting Co.
  - Light Color: Warm White

## How to Use

This layer file is designed to compose with the base HelloWall model:
1. Load the base model: `examples/Hello Wall/hello-wall.ifcx`
2. Apply this layer: `homework/hello-wall-add-light-fixture.ifcx`
3. The light fixture will appear as a child of My_Storey alongside the existing Wall and My_Space

## Technical Details

The layer file:
- References the My_Storey object (GUID: `44af358b-3160-4063-8a89-a868335ff3b5`)
- Adds a new child named "LightFixture" with a unique GUID
- Defines custom properties through a schema definition
- Includes complete geometry definition using IFC shape representation
