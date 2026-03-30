---
name: thatopen-impl-viewer
description: >
  Use when setting up a ThatOpen BIM viewer from scratch, creating worlds,
  configuring renderers, or initializing the 3D viewport.
  Prevents missing init() calls and incorrect world setup order.
  Covers complete viewer setup: Worlds.create(), SimpleScene, SimpleRenderer,
  PostproductionRenderer, OrthoPerspectiveCamera, Grids, components.init(),
  container binding, dispose lifecycle.
  Keywords: viewer, world, scene, renderer, camera, grid, setup, init, dispose, postproduction, viewport, container, webgl, create viewer, embed viewer, viewer not rendering, blank canvas.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Viewer Setup

## Overview

This skill covers the complete setup of a ThatOpen BIM viewer: creating a
World (scene + camera + renderer), configuring the container element,
starting the render loop, and proper cleanup. It includes both the basic
SimpleRenderer path and the advanced PostproductionRenderer path with
ambient occlusion and edge detection.

**Version**: @thatopen/components 3.3.x, @thatopen/components-front 3.3.x
**Prerequisite**: thatopen-core-architecture (component system, lifecycle)

## Setup Order

ALWAYS follow this exact order when creating a viewer:

```
1. new Components()
2. components.get(Worlds)
3. worlds.create<Scene, Camera, Renderer>()
4. world.scene = new SimpleScene(components)
5. world.scene.setup()
6. world.renderer = new SimpleRenderer(components, container)
      — OR new PostproductionRenderer(components, container)
7. world.camera = new OrthoPerspectiveCamera(components)
8. components.get(Grids).create(world)          [optional]
9. components.init()                            [REQUIRED]
```

NEVER reorder these steps. The scene MUST be assigned before the renderer.
The camera MUST be assigned last. `components.init()` MUST be the final
call after all world setup is complete.

## Container Element

The HTML container element is the DOM element where the WebGL canvas is
rendered. It MUST satisfy these requirements:

- MUST be a block-level element (typically `<div>`).
- MUST have explicit width and height via CSS. Zero-size containers
  produce a black or invisible viewport.
- MUST exist in the DOM before passing it to the renderer constructor.
- MUST NOT be `display: none` at construction time.

```html
<div id="viewer" style="width: 100%; height: 100vh;"></div>
```

```typescript
const container = document.getElementById("viewer") as HTMLDivElement;
// ALWAYS verify the container exists before creating the renderer
if (!container) throw new Error("Viewer container not found");
```

The renderer creates a `<canvas>` element inside the container and manages
its lifecycle. NEVER manually create or manipulate this canvas.

## World Creation

A World connects a Scene, Camera, and Renderer into a single viewport.
Use the `Worlds` component to create worlds.

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);

// Type parameters specify the concrete implementations
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();
```

- ALWAYS use `components.get(OBC.Worlds)` to access the Worlds manager.
  NEVER instantiate `Worlds` directly.
- The type parameters are for TypeScript type narrowing only. You MUST
  still assign the concrete instances manually (see next sections).
- Multiple worlds are supported. Each world gets its own scene, camera,
  and renderer. All worlds update in the same `components.init()` loop.

## Scene Setup

`SimpleScene` wraps a `THREE.Scene` and provides default lighting and
background setup via the `Configurable` interface.

```typescript
world.scene = new OBC.SimpleScene(components);
world.scene.setup(); // Creates default directional + ambient lights
```

- `SimpleScene` implements `Configurable`. ALWAYS call `setup()` after
  assigning it to get default lights and background.
- Access the underlying Three.js scene: `world.scene.three`.
- To customize lighting, modify the Three.js scene after `setup()`:

```typescript
world.scene.setup();
const scene = world.scene.three;
scene.background = new THREE.Color(0x202932);
```

- To add custom Three.js objects:

```typescript
const mesh = new THREE.Mesh(geometry, material);
world.scene.three.add(mesh);
world.meshes.add(mesh); // Track for raycasting
```

## Renderer: SimpleRenderer

`SimpleRenderer` wraps `THREE.WebGLRenderer`. It handles canvas creation,
resizing, and per-frame rendering.

```typescript
const container = document.getElementById("viewer") as HTMLDivElement;
world.renderer = new OBC.SimpleRenderer(components, container);
```

- The renderer creates a WebGL canvas inside the container element.
- It implements `Resizeable` and auto-resizes via `ResizeObserver`.
- Access the underlying Three.js renderer: `world.renderer.three`.
- To customize WebGL parameters:

```typescript
world.renderer = new OBC.SimpleRenderer(components, container, {
  antialias: true,
  alpha: true,
});
```

## Renderer: PostproductionRenderer

`PostproductionRenderer` extends the renderer with post-processing effects:
ambient occlusion (AO), edge detection, outline rendering, and SMAA
anti-aliasing. It is in `@thatopen/components-front` (browser-only).

```typescript
import * as OBCF from "@thatopen/components-front";

const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

world.renderer = new OBCF.PostproductionRenderer(components, container);
```

### Enabling Post-Processing

Post-processing is disabled by default. ALWAYS enable it explicitly:

```typescript
world.renderer.postproduction.enabled = true;
```

### Post-Processing Features

Access features via `world.renderer.postproduction`:

| Feature | Property | Default | Effect |
|---|---|---|---|
| Ambient Occlusion | `ao` | enabled | Darkens crevices and edges |
| Edge Detection | `customEdges` | enabled | Draws outlines on geometry |
| SMAA | Part of pipeline | enabled | Anti-aliasing |
| Gloss Pass | `glossEnabled` | false | Reflective surface pass |

```typescript
const pp = world.renderer.postproduction;
pp.enabled = true;
pp.ao = true;           // ambient occlusion
pp.customEdges = true;  // edge detection outlines
```

### PostproductionRenderer + Grid Interaction

When using `PostproductionRenderer`, the grid MUST be excluded from
post-processing to avoid visual artifacts:

```typescript
const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);
```

ALWAYS exclude the grid mesh from post-processing. Failing to do so
causes the grid to render with incorrect edges and AO artifacts.

## Camera: OrthoPerspectiveCamera

`OrthoPerspectiveCamera` provides dual perspective/orthographic cameras
with orbit, first-person, and plan navigation modes.

```typescript
world.camera = new OBC.OrthoPerspectiveCamera(components);
```

- ALWAYS assign the camera after the scene and renderer.
- It wraps `camera-controls` for orbit/pan/zoom interaction.
- Access the underlying cameras:
  - `world.camera.threePersp` — `THREE.PerspectiveCamera`
  - `world.camera.threeOrtho` — `THREE.OrthographicCamera`
  - `world.camera.three` — whichever is currently active

### Navigation Modes

```typescript
const camera = world.camera as OBC.OrthoPerspectiveCamera;
camera.set("Orbit");       // Default: orbit around target
camera.set("FirstPerson"); // WASD-style movement
camera.set("Plan");        // Orthographic top-down view
```

### Framing Objects

```typescript
// Frame all meshes in the world
world.camera.fit(world.meshes);

// Frame with offset (1.5 = 50% padding)
world.camera.fit(world.meshes, 1.5);
```

## Grid

`Grids` creates an infinite ground grid for spatial reference.

```typescript
const grids = components.get(OBC.Grids);
const grid = grids.create(world);
```

- ALWAYS create the grid BEFORE calling `components.init()`.
- The grid auto-fades based on camera distance.
- Access the underlying Three.js mesh: `grid.three`.

## Starting the Render Loop

```typescript
components.init();
```

- ALWAYS call `components.init()` AFTER all world setup is complete.
- This starts the `requestAnimationFrame` loop.
- All `Updateable` components (renderer, camera, etc.) receive
  `update(delta)` calls each frame.
- NEVER call `init()` before assigning scene, renderer, and camera.
  The update loop will attempt to render an incomplete world.
- Calling `init()` multiple times is safe but unnecessary.

## Disposal

```typescript
components.dispose();
```

- ALWAYS call `components.dispose()` when the viewer is no longer needed.
- This disposes ALL components, worlds, renderers, scenes, and cameras.
- `FragmentsManager` is disposed last to prevent dangling references.
- After calling `dispose()`, NEVER use any component or world references.
- In frameworks (React, Vue, Angular), call dispose in the unmount/destroy
  lifecycle hook.

### Framework Integration Example (React)

```typescript
useEffect(() => {
  const components = new OBC.Components();
  // ... setup world ...
  components.init();

  return () => {
    components.dispose();
  };
}, []);
```

## Bundler Configuration (Vite)

ThatOpen requires specific Vite configuration for WASM files, web workers,
and cross-origin isolation headers (needed for `SharedArrayBuffer`).

```typescript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    {
      name: "coop-coep",
      configureServer(server) {
        server.middlewares.use((_req, res, next) => {
          res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
          res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
          next();
        });
      },
    },
  ],
  optimizeDeps: {
    exclude: ["web-ifc"],
  },
  worker: {
    format: "es",
  },
});
```

### COOP/COEP Headers

- `Cross-Origin-Opener-Policy: same-origin` and
  `Cross-Origin-Embedder-Policy: require-corp` are REQUIRED for
  `SharedArrayBuffer`, which web-ifc uses internally.
- Without these headers, WASM operations may fail silently or throw.
- For production, configure these headers on your web server or CDN.

### WASM Files

- web-ifc WASM files (`web-ifc.wasm`, `web-ifc-mt.wasm`) MUST be
  accessible at runtime.
- `IfcLoader.setup()` with default settings auto-resolves WASM paths.
- For custom paths, copy WASM files to your `public/` directory and
  configure `IfcLoader` accordingly.

### Worker Files

- Fragment processing uses web workers for off-main-thread performance.
- ALWAYS initialize `FragmentsManager` with a worker URL:
  `fragments.init(workerURL)`.
- Vite handles worker bundling via `worker: { format: "es" }`.

## Minimal Viewer (Copy-Paste Ready)

```typescript
import * as OBC from "@thatopen/components";

// Container: must have width and height
const container = document.getElementById("viewer") as HTMLDivElement;

// 1. Components container
const components = new OBC.Components();

// 2. Create world
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();

// 3. Scene with default lights
world.scene = new OBC.SimpleScene(components);
world.scene.setup();

// 4. Renderer bound to container
world.renderer = new OBC.SimpleRenderer(components, container);

// 5. Camera with orbit controls
world.camera = new OBC.OrthoPerspectiveCamera(components);

// 6. Grid (optional)
components.get(OBC.Grids).create(world);

// 7. Start render loop
components.init();

// 8. Cleanup when done
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

## PostproductionRenderer Viewer (Copy-Paste Ready)

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

const container = document.getElementById("viewer") as HTMLDivElement;

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// Enable post-processing
world.renderer.postproduction.enabled = true;

// Grid — exclude from post-processing
const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);

components.init();

window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

## Critical Rules

1. **ALWAYS** call `components.init()` after world setup. Without it,
   nothing renders.
2. **ALWAYS** call `components.dispose()` on cleanup. BIM models consume
   hundreds of MB of GPU memory.
3. **ALWAYS** call `world.scene.setup()` after assigning the scene.
   Without it, there are no lights.
4. **ALWAYS** exclude the grid from PostproductionRenderer.
5. **ALWAYS** ensure the container has non-zero dimensions before
   creating the renderer.
6. **NEVER** reorder the setup sequence: scene first, then renderer,
   then camera.
7. **NEVER** use component references after calling `components.dispose()`.
8. **NEVER** import `@thatopen/components-front` in Node.js environments.
9. **NEVER** create the renderer with a container that has `display: none`.
10. **NEVER** call `components.init()` before assigning all three world
    components (scene, renderer, camera).

## Reference Files

- [references/methods.md](references/methods.md) — World setup methods,
  renderer options, camera configuration
- [references/examples.md](references/examples.md) — Minimal viewer,
  PostproductionRenderer, full application setup
- [references/anti-patterns.md](references/anti-patterns.md) — Missing
  init, wrong dispose order, WebGL context issues

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch
- npm: `@thatopen/components@3.3.3`, `@thatopen/components-front@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md`
- Skill: `thatopen-core-architecture` (SKILL.md, references/)
