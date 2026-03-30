---
name: thatopen-impl-navigation
description: >
  Use when configuring camera navigation modes, switching between orbit and
  first-person views, or implementing plan view for floor plans.
  Prevents incorrect projection usage and navigation mode conflicts.
  Covers OrthoPerspectiveCamera modes (Orbit, FirstPerson, Plan), projection
  switching, fit() framing, custom navigation modes, camera controls,
  frustum settings, user input toggling.
  Keywords: navigation, camera, orbit, first person, plan, projection, perspective, orthographic, fit, controls, zoom, pan, rotate, fly to, look at, camera position, walk through model.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Navigation: OrthoPerspectiveCamera

## Overview

`OrthoPerspectiveCamera` extends `SimpleCamera` and provides three built-in
navigation modes (Orbit, FirstPerson, Plan) with switchable perspective and
orthographic projections. It wraps the `camera-controls` library and manages
dual Three.js cameras (`PerspectiveCamera` + `OrthographicCamera`).

**Package**: `@thatopen/components` (core — works in browser and Node.js)
**Dependency**: `camera-controls` >= 3.1.2

## Prerequisites

- World must be created with `OrthoPerspectiveCamera` as the camera type.
  See `thatopen-impl-viewer` for full viewer setup.
- `components.init()` MUST be called after camera assignment for the render
  loop to start.

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

components.init();
```

## Navigation Modes

Three built-in modes control how the user interacts with the 3D viewport.
Switch modes with `camera.set(mode)`.

### Mode Comparison Table

| Property          | Orbit               | FirstPerson          | Plan                 |
|-------------------|----------------------|----------------------|----------------------|
| ID                | `"Orbit"`            | `"FirstPerson"`      | `"Plan"`             |
| Default           | Yes (active at start)| No                   | No                   |
| Rotation          | Full orbit around target | Look around from position | Disabled (locked)  |
| Pan               | Truck (right-drag)   | Truck (high speed)   | Truck (left-drag)    |
| Zoom              | Dolly (scroll)       | Dolly (scroll)       | Zoom (scroll)        |
| Projection        | Any                  | Perspective ONLY     | Any (ortho preferred)|
| Use case          | 3D model inspection  | Walkthroughs         | Floor plan views     |

### Orbit Mode (Default)

Standard 3D orbit navigation. Rotates around a target point.

**Camera-controls settings applied:**
- `minDistance`: 1
- `maxDistance`: 300
- `truckSpeed`: 2
- Orbit target calculated from camera direction and distance

```typescript
world.camera.set("Orbit");
```

### FirstPerson Mode

FPS-style navigation. Camera stays at a fixed position; user looks around
and moves through the scene.

**Camera-controls settings applied:**
- `minDistance`: 1, `maxDistance`: 1, `distance`: 1
- `truckSpeed`: 50
- Mouse wheel mapped to dolly action
- Two-finger touch mapped to zoom-truck

**CRITICAL**: FirstPerson mode ALWAYS requires perspective projection. It
CANNOT run with orthographic projection. ALWAYS validate before switching:

```typescript
const { current } = world.camera.projection;
if (current === "Orthographic") {
  // Switch to perspective first, or skip FirstPerson
  await world.camera.projection.set("Perspective");
}
world.camera.set("FirstPerson");
```

If you call `set("FirstPerson")` while in orthographic mode, the camera
falls back to Orbit mode instead.

### Plan Mode

2D floor plan navigation. Rotation is disabled; only pan and zoom work.
Ideal for architectural floor plans combined with clipping planes.

**Camera-controls settings applied:**
- Azimuth rotation speed: 0 (disabled)
- Polar rotation speed: 0 (disabled)
- Left mouse mapped to TRUCK (pan)
- Single-touch mapped to TOUCH_TRUCK
- Dual-touch mapped to TOUCH_ZOOM

When Plan mode is deactivated, original rotation speeds and mouse/touch
mappings are restored.

```typescript
// Switch to Plan mode for floor plan viewing
world.camera.set("Plan");

// Combine with orthographic projection for true floor plans
await world.camera.projection.set("Orthographic");
```

### Camera Controls Interaction Table

| Input               | Orbit           | FirstPerson     | Plan            |
|---------------------|-----------------|-----------------|-----------------|
| Left mouse drag     | Rotate          | Look around     | Pan             |
| Right mouse drag    | Pan (truck)     | Pan (truck)     | Pan (truck)     |
| Scroll wheel        | Zoom (dolly)    | Move (dolly)    | Zoom            |
| Single-finger touch | Rotate          | Look around     | Pan             |
| Two-finger touch    | Zoom + Pan      | Zoom + Truck    | Zoom            |

## Projection System

`ProjectionManager` handles switching between perspective and orthographic
cameras. Access it via `camera.projection`.

### Switching Projection

```typescript
// Set to orthographic
await world.camera.projection.set("Orthographic");

// Set to perspective
await world.camera.projection.set("Perspective");

// Toggle between the two
await world.camera.projection.toggle();

// Read current projection
const current: OBC.CameraProjection = world.camera.projection.current;
// Returns "Perspective" or "Orthographic"
```

### Projection Changed Event

Listen for projection changes to update UI or other components:

```typescript
world.camera.projection.onChanged.add(() => {
  const projection = world.camera.projection.current;
  console.log("Projection changed to:", projection);

  // Common pattern: toggle grid fade based on projection
  grid.fade = projection === "Perspective";
});
```

`onChanged` fires after the projection switch completes. The event emits
the new active `THREE.Camera` instance.

### Projection Compatibility Matrix

| Projection     | Orbit | FirstPerson | Plan  |
|----------------|-------|-------------|-------|
| Perspective    | Yes   | Yes         | Yes   |
| Orthographic   | Yes   | NO          | Yes   |

NEVER combine FirstPerson mode with Orthographic projection. ALWAYS
validate before allowing user selection of mode/projection combinations.

## Dual Camera Access

OrthoPerspectiveCamera maintains two Three.js camera instances:

```typescript
// Perspective camera (THREE.PerspectiveCamera)
const perspCam = world.camera.threePersp;

// Orthographic camera (THREE.OrthographicCamera)
const orthoCam = world.camera.threeOrtho;

// Active camera (whichever is current based on projection)
const activeCam = world.camera.three;
```

- `threePersp` and `threeOrtho` ALWAYS exist, regardless of current mode.
- `camera.three` returns whichever camera is currently active.
- The frustum size for orthographic projection is fixed at 50.
- Aspect ratio is automatically updated when the renderer resizes.

## Fit / Frame Objects

`fit()` positions the camera to frame a set of meshes in the viewport.

```typescript
// Fit specific meshes with default offset (1.5)
await world.camera.fit(meshes);

// Fit with custom offset (larger = more padding)
await world.camera.fit(meshes, 2.0);
```

**Parameters:**
- `meshes`: `Iterable<THREE.Mesh>` — the meshes to frame
- `offset`: `number` (default `1.5`) — padding multiplier around the
  bounding box. Values > 1 add space around the objects.

There is also a convenience method `fitToItems()` that fits to all loaded
fragment models:

```typescript
// Frame all loaded models
world.camera.fitToItems();
```

## User Input Control

Temporarily disable or enable all camera controls:

```typescript
// Disable all user camera input
world.camera.setUserInput(false);

// Re-enable user camera input
world.camera.setUserInput(true);
```

When disabled, `setUserInput(false)` stores the current mouse button
mappings and then nullifies them. When re-enabled, mappings are restored.

Use this to lock the camera during animations, automated camera movements,
or guided tours.

## Custom Navigation Modes

Register a custom navigation mode that follows the `NavigationMode`
interface:

```typescript
interface NavigationMode {
  id: string;        // Unique mode identifier
  enabled: boolean;  // Whether this mode is currently active
  set(active: boolean): void;  // Called when mode is activated/deactivated
}
```

Register and use:

```typescript
const flyMode: OBC.NavigationMode = {
  id: "Fly",
  enabled: false,
  set(active: boolean) {
    this.enabled = active;
    if (active) {
      // Configure camera-controls for fly behavior
      const controls = world.camera.controls;
      controls.minDistance = 0;
      controls.maxDistance = Infinity;
      controls.truckSpeed = 10;
    }
  },
};

world.camera.addCustomNavigationMode(flyMode);
world.camera.set("Fly");
```

Custom modes are stored in the internal `_navigationModes` map alongside
the built-in Orbit, FirstPerson, and Plan modes.

## Camera Controls (Low-Level)

Direct access to the underlying `camera-controls` instance:

```typescript
const controls = world.camera.controls;

// Set camera position and look-at target
await controls.setLookAt(x, y, z, targetX, targetY, targetZ);

// Smooth transition
await controls.setLookAt(x, y, z, tx, ty, tz, true); // animated

// Listen for camera updates (e.g., to update fragment LOD)
controls.addEventListener("update", () => {
  fragments.core.update();
});
```

ALWAYS use `controls.setLookAt()` for programmatic camera positioning.
NEVER set `camera.three.position` directly — camera-controls will
overwrite it on the next frame.

## Floor Plan Setup Pattern

Complete pattern for floor plan viewing with Plan mode:

```typescript
// 1. Switch to Plan mode (disables rotation)
world.camera.set("Plan");

// 2. Switch to orthographic projection
await world.camera.projection.set("Orthographic");

// 3. Position camera looking straight down
await world.camera.controls.setLookAt(
  centerX, height, centerZ,  // camera position (above the floor)
  centerX, 0, centerZ,       // look-at target (floor level)
);

// 4. Disable grid fade for orthographic
grid.fade = false;

// 5. Add clipping plane at floor level (see thatopen-impl-clipping-plans)
// const clipper = components.get(OBC.Clipper);
// clipper.createFromNormalAndCoplanarPoint(world, normal, point);
```

## Critical Rules

1. **NEVER** combine FirstPerson mode with Orthographic projection. The
   camera silently falls back to Orbit mode, causing confusing behavior.
2. **ALWAYS** validate mode/projection compatibility before switching.
3. **NEVER** set `camera.three.position` directly. ALWAYS use
   `camera.controls.setLookAt()` or `camera.controls.moveTo()`.
4. **ALWAYS** call `projection.set()` with `await` — it returns a Promise.
5. **NEVER** forget to listen to `projection.onChanged` when your UI or
   components depend on the current projection type.
6. **ALWAYS** use `setUserInput(false)` during programmatic camera
   animations to prevent user interference.
7. **ALWAYS** call `camera.dispose()` on cleanup (handled automatically
   when the world is disposed via `worlds.delete(world)`).
8. **NEVER** instantiate OrthoPerspectiveCamera with `new` and forget to
   assign it to `world.camera`. The camera needs a world context.

## Reference Files

- [references/methods.md](references/methods.md) — Full API signatures for
  OrthoPerspectiveCamera, ProjectionManager, NavigationMode interface
- [references/examples.md](references/examples.md) — Mode switching,
  fit patterns, custom mode, plan view setup
- [references/anti-patterns.md](references/anti-patterns.md) — Common
  mistakes with navigation and projection

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch
  (`packages/core/src/core/OrthoPerspectiveCamera/`)
- Example: `OrthoPerspectiveCamera/example.ts`
- Research: `docs/research/vooronderzoek-thatopen.md` Section 2.5
