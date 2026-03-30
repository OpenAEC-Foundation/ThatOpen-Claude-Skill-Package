---
name: thatopen-agents-viewer-builder
description: >
  Use when scaffolding a complete ThatOpen BIM viewer application from
  scratch, including project setup, package installation, and working code.
  Prevents incomplete project setup and missing dependencies.
  Covers end-to-end viewer creation: Vite project setup, npm dependencies,
  HTML container, TypeScript viewer code, IFC loading, highlighting,
  toolbar and panel UI, complete working application.
  Keywords: scaffold, project, setup, viewer, create, new, app, vite, template, starter, boilerplate, complete, how to start, new IFC viewer project, quick setup, getting started.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Viewer Builder Agent

## Purpose

This is an **agent skill** that provides step-by-step instructions for
scaffolding a complete ThatOpen BIM viewer application from scratch. It
guides code generation, not just documentation. When a user asks to create
a new BIM viewer, follow these steps exactly.

**Version**: @thatopen/components 3.3.x, @thatopen/components-front 3.3.x
**Prerequisites**: Node.js 18+, npm

## When to Use This Skill

- User asks to "create a BIM viewer"
- User asks to "scaffold a ThatOpen app"
- User asks to "set up a new IFC viewer project"
- User needs a complete working application from zero

## Step-by-Step Build Instructions

### Step 1: Initialize Vite Project

Create a new directory and initialize with Vite + TypeScript:

```bash
npm create vite@latest <project-name> -- --template vanilla-ts
cd <project-name>
```

ALWAYS use the `vanilla-ts` template. NEVER use React/Vue templates unless
the user explicitly requests a framework.

### Step 2: Install Dependencies

Install ALL required packages with exact compatible versions:

```bash
npm install @thatopen/components@^3.3.3 \
  @thatopen/components-front@^3.3.3 \
  @thatopen/fragments@^3.3.6 \
  @thatopen/ui@^3.3.3 \
  @thatopen/ui-obc@^3.3.3 \
  three@^0.175.0 \
  web-ifc@^0.0.77

npm install -D typescript@^5.4.0 vite@^5.4.0
```

ALWAYS install `@thatopen/fragments` explicitly even though it is a
dependency of `@thatopen/components` — peer dependency resolution varies
across npm versions.

ALWAYS install `three` and `web-ifc` explicitly — they are peer
dependencies, not bundled.

### Step 3: Create Vite Configuration

Create `vite.config.ts` with COOP/COEP headers and WASM handling:

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    {
      name: "coop-coep-headers",
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

**COOP/COEP headers** are REQUIRED. Without them, `SharedArrayBuffer` is
unavailable and web-ifc WASM operations fail silently or throw.

**`optimizeDeps.exclude: ["web-ifc"]`** is REQUIRED. Vite's pre-bundling
breaks WASM module loading.

**`worker.format: "es"`** is REQUIRED for fragment worker compatibility.

### Step 4: Create HTML Entry Point

Replace `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>BIM Viewer</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { width: 100%; height: 100%; overflow: hidden; }
  </style>
</head>
<body>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

NEVER add a static `<div id="viewer">` — the `bim-grid` and `bim-viewport`
custom elements handle layout. The body MUST have full width/height.

### Step 5: Create TypeScript Viewer Code

Replace `src/main.ts` with the complete viewer setup. The code follows
this exact order:

1. **Initialize UI Manager** — `BUI.Manager.init()`
2. **Create Components** — `new OBC.Components()`
3. **Create World** — scene, renderer, camera (in that order)
4. **Enable post-processing** — postproduction renderer
5. **Create grid** — exclude from post-processing
6. **Initialize FragmentsManager** — worker URL
7. **Setup IfcLoader** — WASM configuration
8. **Setup Highlighter** — click-to-select
9. **Start render loop** — `components.init()`
10. **Build UI** — toolbar, panel, grid layout
11. **Wire events** — file input, buttons

See [references/examples.md](references/examples.md) for the complete
copy-pasteable implementation.

### Step 6: Delete Boilerplate Files

Remove Vite template files that are not needed:

```bash
rm -f src/counter.ts src/style.css src/typescript.svg public/vite.svg
```

### Step 7: Run and Verify

```bash
npm run dev
```

Verify in the browser:
- 3D viewport with grid is visible
- Orbit controls work (click + drag)
- IFC file can be loaded via file input button
- Clicking an element highlights it in yellow
- Properties panel shows element data

## Setup Order Rules

ALWAYS follow this exact initialization order:

```
BUI.Manager.init()
  |
  v
new Components()
  |
  v
worlds.create() -> world.scene -> world.renderer -> world.camera
  |
  v
Grids.create(world) + exclude from postproduction
  |
  v
FragmentsManager.init(workerURL)
  |
  v
IfcLoader.setup()
  |
  v
Highlighter.setup({ world })
  |
  v
components.init()
  |
  v
Build UI and append to DOM
```

NEVER call `components.init()` before all components are configured.
NEVER call `ifcLoader.load()` before `ifcLoader.setup()` completes.
NEVER use `bim-*` elements before `BUI.Manager.init()`.
NEVER assign camera before scene and renderer.

## Package Version Matrix

| Package | Version | Purpose |
|---|---|---|
| `@thatopen/components` | ^3.3.3 | Core engine, Components, Worlds, IfcLoader |
| `@thatopen/components-front` | ^3.3.3 | PostproductionRenderer, Highlighter |
| `@thatopen/fragments` | ^3.3.6 | Fragment binary format, workers |
| `@thatopen/ui` | ^3.3.3 | UI web components (bim-panel, bim-toolbar) |
| `@thatopen/ui-obc` | ^3.3.3 | Pre-wired BIM UI components |
| `three` | ^0.175.0 | 3D rendering engine |
| `web-ifc` | ^0.0.77 | IFC WASM parser |
| `typescript` | ^5.4.0 | TypeScript compiler |
| `vite` | ^5.4.0 | Build tool and dev server |

## Highlighter Configuration

The Highlighter provides click-to-select functionality:

```typescript
import * as OBCF from "@thatopen/components-front";

const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });
```

- `setup({ world })` is REQUIRED — it binds to the world's renderer and
  camera for raycasting.
- Default selection color is yellow (#BCF124).
- `highlighter.selection` contains the current selection as a
  `{ [styleName]: ModelIdMap }` object.
- Use `highlighter.clear()` to deselect all.
- Set `highlighter.zoomToSelection = true` to auto-frame selected elements.

## File Input Pattern for IFC Loading

```typescript
const fileInput = document.createElement("input");
fileInput.type = "file";
fileInput.accept = ".ifc";

fileInput.addEventListener("change", async () => {
  const file = fileInput.files?.[0];
  if (!file) return;

  const buffer = await file.arrayBuffer();
  const data = new Uint8Array(buffer);

  const model = await ifcLoader.load(data, true, file.name);
  world.camera.fit(world.meshes);
});
```

ALWAYS convert `File` to `Uint8Array` before passing to `ifcLoader.load()`.
ALWAYS call `world.camera.fit(world.meshes)` after loading to frame the
model in view.

## UI Layout Pattern

Use `bim-grid` with named layouts for the application shell:

```typescript
const grid = document.createElement("bim-grid");
grid.layouts = {
  main: {
    template: `
      "toolbar toolbar" auto
      "panel viewport" 1fr
      / 320px 1fr
    `,
    elements: { toolbar, panel, viewport },
  },
};
grid.layout = "main";
document.body.appendChild(grid);
```

ALWAYS define `grid.layouts` before setting `grid.layout`.
ALWAYS append the grid to `document.body` as the root element.

## Disposal Pattern

```typescript
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

ALWAYS register disposal on `beforeunload`. For frameworks, use the
component lifecycle (React `useEffect` cleanup, Vue `onUnmounted`,
Angular `ngOnDestroy`).

## Critical Rules

1. **ALWAYS** install all packages from the version matrix — missing peer
   dependencies cause silent failures.
2. **ALWAYS** include COOP/COEP headers in Vite config — WASM needs them.
3. **ALWAYS** exclude `web-ifc` from `optimizeDeps` — pre-bundling breaks WASM.
4. **ALWAYS** call `BUI.Manager.init()` before creating any `bim-*` elements.
5. **ALWAYS** follow the setup order: scene -> renderer -> camera.
6. **ALWAYS** call `components.init()` after all setup is complete.
7. **ALWAYS** dispose components on teardown.
8. **NEVER** use `@thatopen/components-front` in Node.js environments.
9. **NEVER** skip `ifcLoader.setup()` — WASM initialization will fail.
10. **NEVER** pass raw `File` or `ArrayBuffer` to `ifcLoader.load()` —
    it requires `Uint8Array`.

## Reference Files

- [references/methods.md](references/methods.md) — Setup checklist,
  package versions, configuration reference
- [references/examples.md](references/examples.md) — Complete working
  application: HTML + TypeScript + Vite config
- [references/anti-patterns.md](references/anti-patterns.md) — Common
  scaffolding mistakes and how to avoid them

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch
- npm: `@thatopen/components@3.3.3`, `@thatopen/components-front@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md`
- Skills: `thatopen-impl-viewer`, `thatopen-syntax-ifc-loading`,
  `thatopen-syntax-ui`
