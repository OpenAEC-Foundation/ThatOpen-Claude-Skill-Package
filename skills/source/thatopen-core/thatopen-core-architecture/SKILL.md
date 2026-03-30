---
name: thatopen-core-architecture
description: >
  Use when creating a ThatOpen BIM application, understanding the component
  system, or reasoning about the ThatOpen architecture.
  Prevents direct component instantiation instead of using components.get().
  Covers Components container, Component base class, lifecycle interfaces,
  World system, package split (core vs front), Event system, DataMap/DataSet.
  Keywords: thatopen, components, world, scene, renderer, camera, lifecycle, disposable, updateable, configurable, bim, architecture, how ThatOpen works, component system, project structure.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Core Architecture

## Overview

ThatOpen engine_components is a component-based BIM/IFC viewer built on
Three.js. It converts IFC models into optimized Fragment geometry (instanced
meshes) and provides tools for visualization, selection, measurement, and
classification.

**Version**: @thatopen/components 3.3.x
**License**: MIT

## Dependency Chain

```
@thatopen/ui-obc
  └─ @thatopen/components-front       (browser-only)
       └─ @thatopen/components         (core, browser + Node.js)
            ├─ @thatopen/fragments     (FlatBuffers, web workers)
            │    ├─ web-ifc            (WASM IFC parser)
            │    └─ three              (3D rendering, >=0.175)
            ├─ three-mesh-bvh          (accelerated raycasting)
            ├─ camera-controls         (orbit/pan/zoom, >=3.1.2)
            ├─ jszip                   (BCF import/export)
            └─ fast-xml-parser         (BCF XML)
```

## Components Container (Singleton Registry)

`Components` is the top-level container. It manages all component instances
and the animation loop.

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
```

**Key behaviors:**
- ALWAYS use `components.get(ComponentClass)` to obtain a component instance.
  NEVER instantiate a component with `new` directly.
- `get<U>()` creates the instance on first call (lazy singleton pattern).
  Subsequent calls return the same instance.
- Each component registers itself by its static `uuid` in the constructor.
- `init()` starts the `requestAnimationFrame` loop using `THREE.Clock` for
  delta time. ALWAYS call `init()` after setting up your world.
- `dispose()` iterates all components and disposes them. `FragmentsManager`
  is ALWAYS disposed last to prevent dangling references.
- BVH acceleration is auto-patched onto `THREE.BufferGeometry` in the
  `Components` constructor. No manual BVH setup is needed.

See [references/methods.md](references/methods.md) for full API signatures.

## Component Base Class

Every ThatOpen component extends `Component`, which extends `Base`:

```typescript
abstract class Base {
  constructor(public components: Components) {}
  isDisposeable(): this is Disposable;
  isUpdateable(): this is Updateable;
  isConfigurable(): this is Configurable<any, any>;
  isResizeable(): this is Resizeable;
  isHideable(): this is Hideable;
  isSerializable(): this is Serializable<any>;
}

abstract class Component extends Base {
  abstract enabled: boolean;
  static readonly uuid: string;
}
```

**Key points:**
- Every component MUST have a static `uuid` string.
- Every component MUST have an `enabled` property.
- The `Base` class provides runtime interface detection via `is*()` methods.
  These use duck-typing, not `instanceof` checks.
- Components self-register in their constructor by calling
  `components.add(MyComponent.uuid, this)`.

## Lifecycle Interfaces

Components opt into behaviors by implementing interfaces (mixin pattern).
This replaces deep inheritance hierarchies.

| Interface       | Methods / Properties                                            | Purpose              |
|-----------------|-----------------------------------------------------------------|----------------------|
| `Disposable`    | `dispose()`, `onDisposed: Event<void>`                          | Cleanup resources    |
| `Updateable`    | `update(delta?)`, `onBeforeUpdate`, `onAfterUpdate`             | Per-frame updates    |
| `Configurable`  | `setup(config?)`, `config`, `isSetup`, `onSetup`                | Deferred init        |
| `Resizeable`    | `resize()`, `getSize()`, `onResize`                             | Viewport resizing    |
| `Hideable`      | `visible: boolean`                                              | Show/hide            |
| `Createable`    | `create()`, `delete()`, `endCreation()`, `cancelCreation()`     | Interactive creation |
| `Serializable`  | `import()`, `export()`                                          | Persistence          |

**Lifecycle flow:**

```
Constructor → [setup()] → enabled=true → update() loop → dispose()
                ↑                           ↑
          Configurable only          Updateable only
```

- The `Components` animation loop calls `update(delta)` on EVERY enabled
  `Updateable` component each frame.
- `Configurable` components ALWAYS require an explicit `setup()` call before
  they function. Check `isSetup` to verify.
- ALWAYS call `dispose()` on cleanup. Failing to dispose causes memory leaks
  that crash browser tabs with large BIM models.

## World System

A `World` connects Scene + Camera + Renderer into a single viewport.

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

**Abstraction layers:**

| Abstract         | Wraps                 | Access underlying Three.js |
|------------------|-----------------------|----------------------------|
| `BaseScene`      | `THREE.Scene`         | `.three`                   |
| `BaseCamera`     | `THREE.Camera`        | `.three`                   |
| `BaseRenderer`   | `THREE.WebGLRenderer` | `.three`                   |
| `SimpleWorld`    | Scene+Camera+Renderer | `.scene` / `.camera` / `.renderer` |

- ALWAYS access the underlying Three.js object via the `.three` property
  when you need direct Three.js manipulation.
- `Worlds` is itself a `Component` that implements `Updateable` and
  `Disposable`. It manages a `DataMap<string, World>` of all worlds.
- When a world is disposed via `worlds.delete(world)`, its scene, camera,
  and renderer are ALSO disposed automatically.

## Package Split: Core vs Front

### @thatopen/components (core)
Works in **both browser and Node.js**:
- Components container, event system, lifecycle interfaces
- World management (Worlds, SimpleWorld, SimpleScene, SimpleRenderer)
- Camera (OrthoPerspectiveCamera with camera-controls)
- Fragment loading (IfcLoader, FragmentsManager)
- Fragment manipulation (Hider, BoundingBoxer, Classifier, ItemsFinder)
- Raycasting, clipping planes (Clipper), grids
- OpenBIM standards (BCFTopics, IDSSpecifications)

### @thatopen/components-front (front)
**Browser-only** (requires DOM and WebGL context):
- PostproductionRenderer (AO, edge detection, outlines, SMAA)
- Highlighter (selection visualization with color maps)
- Hoverer (fade animation hover effect) — NEW in v3
- Outliner (geometry outline rendering)
- Marker (2D screen-space annotations with clustering)
- Measurements (length, area, angle, volume)
- ClipStyler (section/plan edge visualization) — replaces v2 Plans/ClipEdges
- Mesher (convert fragments to regular THREE.Mesh) — NEW in v3
- FastModelPicker (GPU-based element picking) — NEW in v3

NEVER import from `@thatopen/components-front` in Node.js code. It will fail.

## Event System

ThatOpen uses a custom pub/sub `Event<T>` class throughout:

```typescript
class Event<T> {
  add(handler: (data: T) => void): void;
  remove(handler: (data: T) => void): void;
  trigger(data?: T): void;
  reset(): void;
  enabled: boolean;
}
```

- Events are used for lifecycle hooks (`onDisposed`, `onSetup`), state
  changes (`onBeforeUpdate`, `onAfterUpdate`), and domain events
  (`onFragmentsLoaded`).
- ALWAYS remove event handlers when disposing custom code to prevent leaks.
- Set `event.enabled = false` to temporarily suppress an event.

## Reactive Collections: DataMap and DataSet

ThatOpen provides reactive wrappers around `Map` and `Set`:

- **DataMap<K, V>** — extends `Map` with `onItemSet`, `onItemUpdated`,
  `onItemDeleted`, `onCleared` events.
- **DataSet<T>** — extends `Set` with `onItemAdded`, `onItemDeleted`,
  `onCleared` events.

These are used throughout the API (e.g., `Components.list`, `Worlds.list`,
`Classifier.list`, `Highlighter.styles`). ALWAYS use event hooks on these
collections when you need to react to changes.

## ModelIdMap

The universal data structure for targeting specific items across models:

```typescript
type ModelIdMap = Record<string, Set<number>>;
// Keys: model IDs (strings)
// Values: sets of local element IDs (numbers)
```

Used by: Hider, Classifier, FragmentsManager, BoundingBoxer, Highlighter,
and all selection/query operations.

## Critical Warnings

1. **NEVER** instantiate components directly with `new`. ALWAYS use
   `components.get(ComponentClass)`.
2. **NEVER** forget to call `components.dispose()` on cleanup. Memory leaks
   from undisposed BIM models crash browser tabs.
3. **NEVER** import `@thatopen/components-front` in Node.js environments.
4. **NEVER** skip `components.init()` — without it, the render loop does
   not start and nothing renders.
5. **NEVER** use deprecated packages: `web-ifc-three`, `web-ifc-viewer`,
   or `openbim-components`. Use `@thatopen/components` 3.x.
6. **NEVER** use v2 APIs (`Plans`, `ClipEdges`) — use `ClipStyler` + `View`
   in v3.
7. **ALWAYS** call `setup()` on `Configurable` components before using them
   (e.g., `IfcLoader`, `SimpleScene`, `Highlighter`).
8. **ALWAYS** initialize `FragmentsManager` with a worker URL before loading
   models: `fragments.init(workerURL)`.

## Key Design Decisions

1. **Singleton per Components instance** — `components.get(X)` ALWAYS
   returns the same instance for a given component class.
2. **Mixin interfaces over inheritance** — Components opt into behaviors
   (Disposable, Updateable, etc.) rather than using deep class hierarchies.
3. **Worker offloading** — Fragment operations run in web workers to keep
   the main thread responsive.
4. **BVH raycasting** — three-mesh-bvh is auto-patched. No manual setup
   is needed for fast raycasting on large models.
5. **Configurable deferred setup** — Components with complex initialization
   implement `Configurable` and require explicit `setup()` calls.

## Reference Files

- [references/methods.md](references/methods.md) — Full API signatures for
  Components, Component, Base, World, Worlds, Event, DataMap, DataSet
- [references/examples.md](references/examples.md) — World setup, component
  registration, lifecycle patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Common
  mistakes and what NOT to do

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch (packages/core/src/)
- npm: `@thatopen/components@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md`
