---
name: thatopen-impl-clipping-plans
description: >
  Use when adding clipping planes, floor plan views, or cross-section
  visualization to a ThatOpen BIM viewer.
  Prevents using deprecated v2 Plans/ClipEdges patterns.
  Covers Clipper (create, delete, drag), ClipStyler (styles, createFromView,
  createFromClipping), View system, Plan camera mode, edge visualization,
  programmatic plane creation.
  Keywords: clipper, clipping, plane, section, floor plan, clip edges, clip styler, view, cut, cross section, slice model, cut through building, show floor plan.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Clipping Planes & Floor Plans

## Overview

Clipping planes slice through 3D BIM geometry to reveal internal structure.
Floor plan views combine clipping planes with styled edge visualization and
a top-down camera. In ThatOpen v3, three components work together:

| Component | Package | Role |
|-----------|---------|------|
| `Clipper` | `@thatopen/components` | Creates and manages clipping planes |
| `ClipStyler` | `@thatopen/components-front` | Adds styled edge/fill visualization to clips |
| `Views` | `@thatopen/components` | Manages named section views with camera sync |

**v2 to v3 BREAKING CHANGE**: The `Plans` component and standalone `ClipEdges`
component from v2 are removed. ALWAYS use `ClipStyler` + `Views` in v3.
See the migration section at the bottom of this file.

## Prerequisites

- World with `OrthoPerspectiveCamera` and renderer (see `thatopen-impl-viewer`)
- Model loaded via `FragmentsManager` (see `thatopen-syntax-ifc-loading`)
- For styled edges: `@thatopen/components-front` and `LineMaterial` from
  `three/examples/jsm/lines/LineMaterial.js`

```typescript
import * as OBC from "@thatopen/components";
import * as OBF from "@thatopen/components-front";
import * as THREE from "three";
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial.js";
```

---

## 1. Clipper: Creating Clipping Planes

The `Clipper` component creates and manages `SimplePlane` instances that clip
the scene geometry. It lives in `@thatopen/components` (works in browser and
Node.js).

### Setup

```typescript
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;
```

### Interactive Creation (Raycast-Based)

Create a clipping plane where the user clicks on geometry:

```typescript
container.ondblclick = () => {
  if (clipper.enabled) {
    clipper.create(world);
  }
};
```

`create(world)` casts a ray from the current mouse position, finds the
intersection point and surface normal, and places a draggable plane there.
Returns `Promise<SimplePlane | null>` (null if no intersection).

### Programmatic Creation

Create a plane at an exact position and orientation:

```typescript
const normal = new THREE.Vector3(0, -1, 0);  // pointing down
const point = new THREE.Vector3(0, 3.5, 0);   // at y=3.5m
const planeId = clipper.createFromNormalAndCoplanarPoint(world, normal, point);
```

Returns the `string` ID of the created plane. Use this ID with
`ClipStyler.createFromClipping()` to add edge visualization.

### Deletion

```typescript
// Delete plane under mouse cursor (interactive)
await clipper.delete(world);

// Delete a specific plane by ID
await clipper.delete(world, planeId);

// Delete ALL planes
clipper.deleteAll();

// Delete only planes of a specific type
clipper.deleteAll(new Set(["floor-plans"]));
```

### Keyboard Shortcut Pattern

```typescript
window.onkeydown = (event) => {
  if (event.code === "Delete" || event.code === "Backspace") {
    clipper.delete(world);
  }
};
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enabled` | `boolean` | `false` | Enables/disables the clipper |
| `visible` | `boolean` | `true` | Shows/hides all plane helpers |
| `size` | `number` | `5` | Geometric size of plane helpers |
| `material` | `MeshBasicMaterial` | — | Material for plane helpers |
| `orthogonalY` | `boolean` | `false` | Forces planes orthogonal to Y |
| `toleranceOrthogonalY` | `number` | `0.7` | Threshold for Y orthogonality |
| `autoScalePlanes` | `boolean` | `true` | Scale planes based on camera distance |

### Events

Key events (see `references/methods.md` for full list):

| Event | Payload | When |
|-------|---------|------|
| `onAfterCreate` | `SimplePlane` | After plane is created |
| `onBeforeDrag` | `SimplePlane` | Drag operation starts |
| `onAfterDrag` | `SimplePlane` | Drag operation ends |
| `onAfterDelete` | `SimplePlane` | After plane is deleted |

```typescript
clipper.onAfterCreate.add((plane) => {
  console.log("Created plane:", plane.three.normal, plane.three.constant);
});
```

### SimplePlane Instance

Each plane in `clipper.list` is a `SimplePlane` with properties: `three`
(THREE.Plane), `origin`, `normal`, `enabled`, `visible`, `size`, `type`,
`controls` (TransformControls). Key method:
`setFromNormalAndCoplanarPoint(normal, point)` to reposition.
See `references/methods.md` for full API.

---

## 2. ClipStyler: Styled Edge Visualization

`ClipStyler` adds colored edge outlines and fill materials to clipping planes.
It lives in `@thatopen/components-front` (browser-only).

### Setup

```typescript
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;
```

ALWAYS set `.world` before creating any styled edges.

### Defining Styles

Styles are named combinations of line material and fill material:

```typescript
clipStyler.styles.set("ArchBlue", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
  fillsMaterial: new THREE.MeshBasicMaterial({
    color: 0xadd8e6,
    side: THREE.DoubleSide,
  }),
});

clipStyler.styles.set("StructRed", {
  linesMaterial: new LineMaterial({ color: 0xff0000, linewidth: 3 }),
  fillsMaterial: new THREE.MeshBasicMaterial({
    color: 0xffcccc,
    side: THREE.DoubleSide,
  }),
});
```

ALWAYS define styles BEFORE calling any `create*` method.

### Creating Styled Edges from a Clipping Plane

Link edge visualization to an existing Clipper plane by its ID:

```typescript
// Listen for new clipping planes and auto-style them
clipper.list.onItemSet.add(({ key }) => {
  clipStyler.createFromClipping(key, {
    items: {
      All: { style: "ArchBlue" },
    },
  });
});
```

### Creating Styled Edges from a View

Link edge visualization to a View (camera-synced section):

```typescript
const views = components.get(OBC.Views);
const sectionView = views.create(
  new THREE.Vector3(0, -1, 0),
  new THREE.Vector3(0, 1.5, 0),
);

clipStyler.createFromView(sectionView, {
  items: {
    Walls: {
      style: "ArchBlue",
      data: { "BuildingElement.Class": ["IFCWALL", "IFCWALLSTANDARDCASE"] },
    },
    Structure: {
      style: "StructRed",
      data: { "BuildingElement.Class": ["IFCSLAB", "IFCCOLUMN"] },
    },
  },
});
```

### Creating Styled Edges from a Raw Plane

```typescript
const rawPlane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 3.5);
const edges = clipStyler.create(rawPlane, {
  items: { All: { style: "ArchBlue" } },
});
```

### ClipEdgesCreationConfig

```typescript
interface ClipEdgesCreationConfig {
  link?: boolean;                              // Auto-sync with source plane
  id?: string;                                 // Custom identifier
  world?: OBC.World;                           // Override default world
  items?: Record<string, ClipEdgesItemStyle>;  // Named item groups
}

interface ClipEdgesItemStyle {
  style: string;                           // Style name from clipStyler.styles
  data?: OBC.ClassifierIntersectionInput;  // Filter by classification
}
```

### Visibility Control

```typescript
clipStyler.visible = false;  // Hide all styled edges
clipStyler.visible = true;   // Show all styled edges
```

See `references/methods.md` for full ClipEdges instance API.

---

## 3. Views: Named Section Views

The `Views` component manages named orthogonal section views. Each view has
a primary clipping plane, a far clipping plane, and a camera configuration.

### Setup

```typescript
const views = components.get(OBC.Views);
views.world = world;
```

### Creating Views

```typescript
// From normal and point
const floorView = views.create(
  new THREE.Vector3(0, -1, 0),   // normal pointing down
  new THREE.Vector3(0, 1.5, 0),  // coplanar point at y=1.5m
  { id: "floor-1" },
);

// From a THREE.Plane
const plane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 1.5);
const sectionView = views.createFromPlane(plane, { id: "section-A" });

// From IFC building storeys (auto-creates one view per storey)
const storeyViews = await views.createFromIfcStoreys({
  offset: 0.25,  // 25cm above slab
});

// Elevation views (front, back, left, right)
const elevationViews = await views.createElevations({
  combine: true,  // merge all models into one bounding box
});
```

### Opening and Closing Views

```typescript
// Open a view — switches camera to plan mode, enables clipping
views.open("floor-1");

// Close the active view — restores default camera
views.close("floor-1");

// Close any open view
views.close();

// Check if any view is open
if (views.hasOpenViews) { /* ... */ }
```

### View Instance

Key properties: `id`, `plane` (THREE.Plane), `farPlane`, `camera`
(OrthoPerspectiveCamera), `open` (boolean), `range` (distance between
front/back planes, default 15), `planesEnabled`, `helpersVisible`.
Methods: `update()`, `flip()`, `dispose()`.
See `references/methods.md` for full View API.

---

## 4. Complete Floor Plan Workflow

This is the canonical pattern for creating a floor plan view with styled edges.

```typescript
import * as OBC from "@thatopen/components";
import * as OBF from "@thatopen/components-front";
import * as THREE from "three";
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial.js";

// --- Assume world, components, and model are already set up ---

// Step 1: Configure ClipStyler with styles
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

clipStyler.styles.set("Default", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
  fillsMaterial: new THREE.MeshBasicMaterial({
    color: 0xf0f0f0,
    side: THREE.DoubleSide,
  }),
});

// Step 2: Create views from IFC storeys
const views = components.get(OBC.Views);
views.world = world;

const storeyViews = await views.createFromIfcStoreys({
  offset: 0.25,
});

// Step 3: Add styled edges to each storey view
for (const [id, view] of views.list) {
  clipStyler.createFromView(view, {
    items: {
      All: { style: "Default" },
    },
  });
}

// Step 4: Open a floor plan view (activates clipping + plan camera)
views.open("ground-floor");

// Step 5: Set orthographic projection for true 2D plan
await world.camera.projection.set("Orthographic");
```

### Returning to 3D

```typescript
// Close the view (restores camera)
views.close();

// Switch back to perspective
await world.camera.projection.set("Perspective");

// Return to orbit navigation
world.camera.set("Orbit");
```

---

## 5. Programmatic Section Workflow

For a quick section cut without the View system:

```typescript
// Step 1: Enable clipper
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;

// Step 2: Create plane programmatically
const planeId = clipper.createFromNormalAndCoplanarPoint(
  world,
  new THREE.Vector3(0, -1, 0),
  new THREE.Vector3(0, 3.0, 0),
);

// Step 3: Add styled edges
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

clipStyler.styles.set("Section", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
});

clipStyler.createFromClipping(planeId, {
  items: { All: { style: "Section" } },
});

// Step 4: Hide the plane helper (keep the cut visible)
clipper.visible = false;
```

---

## 6. Per-Category Styling

Use multiple styles and `ClipEdgesItemStyle.data` to apply different edge
colors per IFC element type. The `data` field uses
`ClassifierIntersectionInput` (same format as `Classifier.find()`).
See `references/examples.md` example 6 for full code.

---

## 7. v2 to v3 Migration

| v2 Pattern | v3 Replacement | Notes |
|------------|----------------|-------|
| `Plans` component | `Views` + `ClipStyler.createFromView()` | Complete redesign |
| `ClipEdges` component (standalone) | `ClipStyler.create/createFromClipping()` | Now managed by ClipStyler |
| `plans.create(planConfig)` | `views.create(normal, point)` | Different API shape |
| `plans.goTo(planId)` | `views.open(viewId)` | Renamed |
| `plans.exitPlanView()` | `views.close()` | Renamed |
| Manual edge geometry | `ClipEdgesCreationConfig` | Declarative config |
| `clipEdges.visible = true` | `clipStyler.visible = true` | Managed through ClipStyler |

NEVER use `Plans` or standalone `ClipEdges` imports — they do not exist in v3.
ALWAYS use `Views` + `ClipStyler` for floor plan and section visualization.

---

## Critical Rules

1. **ALWAYS** set `clipStyler.world = world` before creating styled edges.
2. **ALWAYS** define styles in `clipStyler.styles` before calling `create*`.
3. **NEVER** use `Plans` or standalone `ClipEdges` from v2 — they are removed.
4. **ALWAYS** use `ClipStyler` for edge visualization in v3.
5. **ALWAYS** call `clipper.enabled = true` before creating planes.
6. **ALWAYS** use `views.open(id)` to activate a view — it handles camera
   switching and plane activation automatically.
7. **NEVER** manually position the camera for floor plans when using Views —
   `views.open()` handles this.
8. **ALWAYS** dispose clipping resources when done: `clipper.deleteAll()` and
   `clipStyler.dispose()`.
9. **ALWAYS** use `LineMaterial` from `three/examples/jsm/lines/LineMaterial.js`
   for edge line styles — regular `LineBasicMaterial` does not support linewidth.
10. **NEVER** forget to await `projection.set()` — it returns a Promise.

## Reference Files

- [references/methods.md](references/methods.md) — Full API signatures for
  Clipper, ClipStyler, ClipEdges, Views, View, SimplePlane
- [references/examples.md](references/examples.md) — Basic clipping, floor
  plan, styled edges, programmatic sections
- [references/anti-patterns.md](references/anti-patterns.md) — v2 patterns,
  missing ClipStyler setup, common mistakes

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch
  (`packages/core/src/core/Clipper/`, `packages/front/src/core/ClipStyler/`,
  `packages/core/src/core/Views/`)
- Examples: `Clipper/example.ts`, `ClipStyler/example.ts`
- Research: `docs/research/vooronderzoek-thatopen.md` Sections 2.6, 2.8, 6
