---
name: thatopen-impl-measurements
description: >
  Use when adding measurement tools to a ThatOpen BIM viewer, including
  distance, area, volume, and angle measurements.
  Prevents measurement lifecycle errors and missing snapping configuration.
  Covers LengthMeasurement, AreaMeasurement, VolumeMeasurement,
  AngleMeasurement, Measurement base class, snapping, units, modes,
  valueFormatter, measurement lifecycle (create/end/cancel/delete).
  Keywords: measurement, length, area, volume, angle, distance, snap, dimension, measure, units, picker, measure distance, how far apart, calculate area in viewer.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components-front 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Measurements

## Overview

This skill covers the measurement system in `@thatopen/components-front`:
LengthMeasurement, AreaMeasurement, VolumeMeasurement, and AngleMeasurement.
All four extend the abstract `Measurement<T, U>` base class, which provides
shared visual elements, snapping, units, disposal, and event handling.

**Version**: @thatopen/components-front 3.3.x
**Prerequisites**: thatopen-impl-viewer (world setup), thatopen-core-architecture

## Measurement Type Comparison

| Type | Class | Modes | Unit Type | Visual Elements |
|---|---|---|---|---|
| Distance | `LengthMeasurement` | `"free"`, `"edge"` | mm, cm, m, km | DimensionLine + Mark labels |
| Area | `AreaMeasurement` | `"free"`, `"square"`, `"face"` | mm2, cm2, m2, km2 | MeasureFill + DimensionLine + Mark |
| Volume | `VolumeMeasurement` | `"free"` | mm3, cm3, m3, km3 | MeasureVolume + DimensionLine + Mark |
| Angle | `AngleMeasurement` | `"free"` | deg, rad | Arc geometry + DimensionLine + Mark |

## Measurement Base Class

All measurement classes extend:

```typescript
abstract class Measurement<
  T extends Record<string, any>,
  U extends keyof MeasureToUnitMap
> extends OBC.Component implements OBC.Createable, OBC.Hideable, OBC.Disposable
```

### Shared Data Collections

| Collection | Type | Purpose |
|---|---|---|
| `list` | `DataSet<T>` | All measurement elements of that type |
| `lines` | `DataSet<DimensionLine>` | Visual dimension lines |
| `fills` | `DataSet<MeasureFill>` | Visual area fill meshes |
| `labels` | `DataSet<Mark>` | HTML label overlays |
| `volumes` | `DataSet<MeasureVolume>` | Visual volume meshes |

### Shared Configuration Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `false` | Activates measurement interaction |
| `visible` | `boolean` | `true` | Shows/hides all visual elements |
| `units` | `MeasureToUnitMap[U]` | per type | Unit for display |
| `rounding` | `number` | `2` | Decimal precision |
| `color` | `THREE.Color` | blue | Color of all visual elements |
| `delay` | `number` | `300` | Pointer stop detection delay (ms) |
| `world` | `OBC.World` | — | World reference (REQUIRED) |

### Material Properties

| Property | Type | Default |
|---|---|---|
| `linesMaterial` | `THREE.LineBasicMaterial` | Blue, depthTest false |
| `fillsMaterial` | `THREE.MeshLambertMaterial` | Green, double-sided, 30% opacity |
| `volumesMaterial` | `THREE.MeshLambertMaterial` | Green, double-sided, 30% opacity |
| `linesEndpointElement` | `HTMLElement` | Blue rounded div (dimension mark) |

### Static Members

```typescript
// Custom value formatter — applies to ALL measurement instances
Measurement.valueFormatter = (value: number) => `${value.toFixed(1)} m`;
```

ALWAYS set `Measurement.valueFormatter` before enabling measurements if you
need custom label formatting. It is a static property shared across all
measurement types.

### Events

| Event | Payload | Trigger |
|---|---|---|
| `onPointerStop` | — | Pointer stopped moving for `delay` ms |
| `onPointerMove` | — | Pointer moved |
| `onStateChanged` | `MeasurementStateChange[]` | Mode, color, units, rounding, visibility, or enabled changed |
| `onEnabledChange` | `boolean` | Enabled toggled |
| `onVisibilityChange` | `boolean` | Visibility toggled |
| `onDisposed` | — | Measurement disposed |

### MeasurementStateChange Values

```typescript
type MeasurementStateChange =
  | "mode" | "color" | "units" | "rounding" | "visibility" | "enabled";
```

### Unit Types

```typescript
type MeasureToUnitMap = {
  length: "mm" | "cm" | "m" | "km";
  area: "mm2" | "cm2" | "m2" | "km2";
  volume: "mm3" | "cm3" | "m3" | "km3";
  angle: "deg" | "rad";
};
```

Default units: length = `"m"`, area = `"m2"`, volume = `"m3"`, angle = `"deg"`.

## Snapping System

Snapping is provided by the `GraphicVertexPicker` inside the base class.
Configure snapping via these properties on any measurement instance:

| Property | Type | Description |
|---|---|---|
| `snappings` | Snapping class array | Which snap types to use: `LINE`, `POINT`, `FACE` |
| `snapDistance` | `number` | Maximum distance for snapping (world units) |
| `pickerSize` | `number` | Visual size of the snap marker (pixels, default 6) |
| `pickerMode` | `GraphicVertexPickerMode` | `DEFAULT` (with snapping) or `SYNCHRONOUS` |

### Snapping Classes

| Class | Visual Indicator | Snap Behavior |
|---|---|---|
| `LINE` | Gray border, square | Snaps to nearest edge |
| `POINT` | Red border, square | Snaps to nearest vertex |
| `FACE` | Purple border, circle | Snaps to face surface |

ALWAYS configure `snappings` before enabling measurements. The default
snapping array includes LINE, POINT, and FACE.

## Measurement Lifecycle

All measurement types follow the same creation lifecycle via the
`Createable` interface:

```
1. Set enabled = true        → activates vertex picker and pointer events
2. User clicks → create()    → places first point / adds point
3. User clicks → create()    → places additional points (area/volume/angle)
4. endCreation()             → finalizes measurement, adds to list
5. cancelCreation()          → discards in-progress measurement
6. delete()                  → removes measurement under cursor via raycasting
7. Set enabled = false       → deactivates interaction
```

- `create()` is called on each click. For LengthMeasurement it takes two
  clicks (start + end). For AreaMeasurement it takes 3+ clicks. For
  AngleMeasurement it takes exactly 3 clicks. For VolumeMeasurement,
  points define the volume boundary.
- `endCreation()` finalizes the current measurement. For AreaMeasurement,
  a minimum of 3 points is required.
- `cancelCreation()` discards any in-progress measurement without saving.
- `delete()` uses raycasting against measurement bounding boxes to find
  and remove the measurement under the cursor.

## LengthMeasurement

Measures distance between two points. Supports free placement and edge
snapping.

```typescript
import * as OBCF from "@thatopen/components-front";

const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.enabled = true;
lengths.mode = "free"; // or "edge"
```

**UUID**: `"2f9bcacf-18a9-4be6-a293-e898eae64ea1"`

### Modes

| Mode | Behavior |
|---|---|
| `"free"` | Click any two points to measure distance between them |
| `"edge"` | Snap to geometry edges; measures edge length |

### Workflow

1. Set `enabled = true` and `mode`
2. User clicks first point → `create()` initializes preview line
3. Preview line follows cursor with snapping
4. User clicks second point → `create()` → `endCreation()` finalizes
5. DimensionLine with label appears showing the distance

## AreaMeasurement

Measures area defined by a polygon of 3+ points.

```typescript
const areas = components.get(OBCF.AreaMeasurement);
areas.world = world;
areas.enabled = true;
areas.mode = "free"; // or "square" or "face"
```

**UUID**: `"09b78c1f-0ff1-4630-a818-ceda3d878c75"`

### Modes

| Mode | Behavior |
|---|---|
| `"free"` | Click 3+ points to define a polygon area |
| `"square"` | Click 2 points to define a rectangular area |
| `"face"` | Click a face to measure its area directly |

### Extra Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `pickTolerance` | `number` | `0.1` | Precision for point selection |
| `tolerance` | `number` | `0.005` | Margin for point inclusion in area |

### Workflow

1. Set `enabled = true` and `mode`
2. User clicks points → `create()` adds each point
3. Preview fill updates showing the polygon
4. Call `endCreation()` when polygon is complete (min 3 points)
5. MeasureFill with perimeter DimensionLines and label appears

## VolumeMeasurement

Measures volume defined by a 3D boundary.

```typescript
const volumes = components.get(OBCF.VolumeMeasurement);
volumes.world = world;
volumes.enabled = true;
volumes.mode = "free";
```

**UUID**: `"01f885ab-ec4e-4e6c-a853-9dfc0d6766ed"`

### Modes

| Mode | Behavior |
|---|---|
| `"free"` | Define volume boundary by selecting geometry faces |

### Extra Events

| Event | Payload | Trigger |
|---|---|---|
| `onPreviewInitialized` | `MeasureVolume` | Preview volume object created |

### Workflow

1. Set `enabled = true`
2. `initPreview()` creates a MeasureVolume preview (fires `onPreviewInitialized`)
3. User clicks faces → `create()` adds selected items to preview
4. `endCreation()` clones preview volume into `list`
5. `cancelCreation()` disposes preview without saving

## AngleMeasurement

Measures the angle between three points (vertex at second point).

```typescript
const angles = components.get(OBCF.AngleMeasurement);
angles.world = world;
angles.enabled = true;
angles.mode = "free";
```

**UUID**: `"2c88a142-2378-422e-b26a-bb2710841813"`

### Modes

| Mode | Behavior |
|---|---|
| `"free"` | Click three points to measure the angle at the middle point |

### Constants

| Constant | Value | Purpose |
|---|---|---|
| `ARC_SEGMENTS` | `32` | Number of segments in the arc visualization |
| `ARC_RADIUS_FACTOR` | `0.3` | Arc radius relative to shortest arm |
| `LABEL_OFFSET_FACTOR` | `1.4` | Label distance from arc center |

### Workflow

1. Set `enabled = true`
2. User clicks first point → `create()` (click 1)
3. User clicks second point (vertex) → `create()` (click 2)
4. User clicks third point → `create()` (click 3) → `endCreation()`
5. Arc visualization with angle label appears

## Setup Pattern (All Types)

ALWAYS follow this pattern when setting up any measurement type:

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

// 1. Get the measurement component
const lengths = components.get(OBCF.LengthMeasurement);

// 2. Assign world (REQUIRED before enabling)
lengths.world = world;

// 3. Configure snapping (optional — defaults include LINE, POINT, FACE)
lengths.snapDistance = 0.5;
lengths.pickerSize = 8;

// 4. Configure units and rounding (optional)
lengths.units = "m";
lengths.rounding = 2;

// 5. Configure color (optional)
lengths.color = new THREE.Color(0xff0000);

// 6. Set mode
lengths.mode = "free";

// 7. Enable (ALWAYS last — this activates pointer events)
lengths.enabled = true;
```

NEVER enable a measurement before assigning `world`. The vertex picker
requires a valid world reference to perform raycasting.

## Toggling Between Measurement Types

ALWAYS disable the current measurement before enabling another:

```typescript
// Switch from length to area
lengths.enabled = false;
areas.enabled = true;
```

NEVER have multiple measurement types enabled simultaneously. They share
pointer events and will conflict.

## Deleting Measurements

```typescript
// Delete measurement under cursor (interactive)
lengths.delete();

// Delete all measurements of a type
lengths.list.clear(); // Clears all length measurements
```

## Disposal

ALWAYS dispose measurement components when no longer needed:

```typescript
// Dispose a specific measurement type
lengths.dispose();

// Or let components.dispose() handle all cleanup
components.dispose();
```

`dispose()` clears the vertex picker, all DataSets (list, lines, fills,
labels, volumes), and disposes all materials. NEVER use a measurement
instance after calling `dispose()`.

## Clipping Plane Integration

Measurements support clipping plane visibility:

```typescript
lengths.applyPlanesVisibility(planes);
```

This updates all visual elements (lines, fills, volumes) to respect the
provided clipping planes.

## Critical Rules

1. **ALWAYS** assign `world` before setting `enabled = true`.
2. **ALWAYS** disable one measurement type before enabling another.
3. **ALWAYS** call `dispose()` or `components.dispose()` on cleanup.
4. **ALWAYS** call `endCreation()` to finalize a measurement. Without it,
   the measurement stays in preview state.
5. **ALWAYS** call `cancelCreation()` to discard an in-progress measurement
   cleanly. Do not just disable — this leaks preview elements.
6. **NEVER** enable multiple measurement types simultaneously.
7. **NEVER** use measurement instances after `dispose()`.
8. **NEVER** forget to set `mode` before enabling — the default mode may
   not match the desired interaction.
9. **NEVER** set `Measurement.valueFormatter` after measurements are
   already created — existing labels will not update retroactively.
10. **NEVER** skip snapping configuration when precision matters — default
    snap distance may be too large or too small for your model scale.

## Reference Files

- [references/methods.md](references/methods.md) — Measurement base class
  API, LengthMeasurement, AreaMeasurement, VolumeMeasurement,
  AngleMeasurement full method reference
- [references/examples.md](references/examples.md) — Setup and usage
  examples for each measurement type
- [references/anti-patterns.md](references/anti-patterns.md) — Missing
  dispose, snapping issues, lifecycle errors

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch (`packages/front/src/measurement/`)
- npm: `@thatopen/components-front@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md` (Section 5)
