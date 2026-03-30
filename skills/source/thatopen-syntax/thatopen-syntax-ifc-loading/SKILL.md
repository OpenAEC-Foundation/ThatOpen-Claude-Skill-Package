---
name: thatopen-syntax-ifc-loading
description: >
  Use when loading IFC files into a ThatOpen viewer, configuring the IFC
  loader, or choosing between IfcLoader and IfcImporter approaches.
  Prevents WASM configuration failures and version mismatches.
  Covers IfcLoader setup, WASM path config, load() method,
  IfcFragmentSettings, IfcImporter (fragments-level), default IFC imports.
  Keywords: ifc, loading, ifcloader, wasm, setup, load, ifcimporter, fragments, web-ifc, model, uint8array, load IFC file, open model, import IFC, fetch and load.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x / web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Loading: IfcLoader & IfcImporter

## Overview

IFC loading in ThatOpen converts IFC files into the internal Fragments binary format for high-performance 3D rendering. There are two approaches:

1. **IfcLoader** (high-level, `@thatopen/components`) — The standard component for loading IFC files in a ThatOpen application. Wraps web-ifc and handles WASM setup, conversion to fragments, and scene integration.
2. **IfcImporter** (low-level, `@thatopen/fragments`) — Direct fragment conversion without the component framework. Useful for server-side processing, Web Workers, or custom pipelines.

> The time-consuming part is the IFC-to-Fragments conversion, not the actual fragment loading. ALWAYS convert once and store the Fragments binary for fast reloading.

---

## Critical Rules

1. **ALWAYS call `setup()` before `load()`** — IfcLoader implements `Configurable`. Calling `load()` before `setup()` causes WASM initialization failures.
2. **ALWAYS match WASM path version to installed web-ifc version** — A version mismatch between the WASM files and the `web-ifc` npm package causes silent parsing failures or crashes.
3. **NEVER hardcode a web-ifc version in the WASM path** without checking `package.json` — the installed version may differ.
4. **ALWAYS initialize FragmentsManager before loading IFC** — Call `fragmentsManager.init(workerURL)` before any IFC loading to enable fragment processing.
5. **ALWAYS pass IFC data as `Uint8Array`** — The `load()` method does not accept `ArrayBuffer`, `string`, or `File` objects directly.
6. **NEVER skip disposal** — Call `components.dispose()` on teardown. IfcLoader holds a `webIfc: IfcAPI` instance that allocates WASM heap memory.

---

## IfcLoader Component

### Setup

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();

// Create world (scene, camera, renderer)
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// Initialize FragmentsManager with worker
const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init(workerURL);

// Get and configure IfcLoader
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();  // uses autoSetWasm (default: true)
```

### WASM Configuration

Two approaches for WASM path configuration:

**Automatic (default):**
```typescript
await ifcLoader.setup();  // autoSetWasm: true (default)
// Automatically resolves WASM path from the web-ifc package
```

**Manual (CDN or custom path):**
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/",
    absolute: true,
  },
});
```

**Manual (local files):**
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "/static/wasm/",
    absolute: false,
  },
});
```

When using a local path, the directory MUST contain `web-ifc.wasm` (and `web-ifc-mt.wasm` for multi-threaded mode). Copy these from `node_modules/web-ifc/`.

### Loading IFC Files

```typescript
// Fetch IFC file as ArrayBuffer, convert to Uint8Array
const response = await fetch("/models/building.ifc");
const buffer = await response.arrayBuffer();
const data = new Uint8Array(buffer);

// Load into the viewer
const model = await ifcLoader.load(data, true, "MyBuilding");

// Add to scene
world.scene.three.add(model.object);
```

### `load()` Parameters

| Parameter | Type | Description |
|---|---|---|
| `data` | `Uint8Array` | IFC file contents as byte array |
| `coordinate` | `boolean` | Apply coordination matrix to align with other models |
| `name` | `string` | Display name for the model |
| `config` | `object` (optional) | Advanced options: `instanceCallback`, `processData`, `userData` |

- Set `coordinate: true` when loading multiple models that must align spatially.
- Set `coordinate: false` for single-model scenarios or when manual positioning is needed.

### `readIfcFile()`: Low-Level Access

```typescript
const modelID = await ifcLoader.readIfcFile(data);
// Now access raw web-ifc API:
const line = ifcLoader.webIfc.GetLine(modelID, expressID);
ifcLoader.cleanUp();  // ALWAYS clean up after raw access
```

Use `readIfcFile()` when you need raw web-ifc queries without full fragment conversion.

---

## IfcFragmentSettings

The `settings` property on IfcLoader controls loading behavior.

| Property | Type | Default | Description |
|---|---|---|---|
| `autoSetWasm` | `boolean` | `true` | Auto-resolve WASM path from web-ifc package |
| `wasm.path` | `string` | — | Directory containing WASM files |
| `wasm.absolute` | `boolean` | — | Whether `path` is an absolute URL |
| `wasm.logLevel` | `LogLevel` | — | web-ifc logging verbosity |
| `webIfc` | `LoaderSettings` | — | web-ifc LoaderSettings passed to `OpenModel()` |
| `customLocateFileHandler` | `LocateFileHandlerFn \| null` | `null` | Custom WASM file locator function |

### web-ifc LoaderSettings (via `settings.webIfc`)

| Setting | Type | Default | Description |
|---|---|---|---|
| `COORDINATE_TO_ORIGIN` | `boolean` | `false` | Translate model geometry to origin |
| `USE_FAST_BOOLS` | `boolean` | `false` | Faster but less accurate boolean operations |
| `CIRCLE_SEGMENTS_LOW` | `number` | — | Tessellation for small curves |
| `CIRCLE_SEGMENTS_MEDIUM` | `number` | — | Tessellation for medium curves |
| `CIRCLE_SEGMENTS_HIGH` | `number` | — | Tessellation for large curves |
| `BOOL_ABORT_THRESHOLD` | `number` | — | Timeout (ms) for boolean operations |
| `MEMORY_LIMIT` | `number` | — | WASM memory limit in bytes |

Use `COORDINATE_TO_ORIGIN: true` for models with large real-world coordinates to prevent floating-point precision issues.

### Configuring Settings After Setup

```typescript
await ifcLoader.setup();

// Modify web-ifc settings before loading
ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;
```

---

## Events

| Event | Type | When |
|---|---|---|
| `onSetup` | `Event<void>` | After `setup()` completes |
| `onIfcStartedLoading` | `Event<void>` | When IFC file parsing begins |
| `onIfcImporterInitialized` | `Event<IfcImporter>` | When the internal IfcImporter is created |
| `onDisposed` | `Event<string>` | When the component is disposed |

### Customizing Default Imports

Use `onIfcImporterInitialized` to modify which IFC entity types are converted:

```typescript
ifcLoader.onIfcImporterInitialized.add((importer) => {
  // Add a category that is not imported by default
  importer.classes.elements.add(WEBIFC.IFCSPACE);

  // Remove a category to save memory
  importer.classes.elements.delete(WEBIFC.IFCFURNISHINGELEMENT);
});
```

By default, IfcLoader imports common building elements (walls, slabs, beams, columns, doors, windows, roofs, stairs, etc.) but excludes some categories like `IFCSPACE` and `IFCOPENINGELEMENT` for performance. Use this event to customize the import set.

---

## IfcImporter (Low-Level, @thatopen/fragments)

IfcImporter operates at the fragments level without requiring the component framework. Use it for server-side conversion, Web Worker pipelines, or when you need fine-grained control.

```typescript
import { IfcImporter } from "@thatopen/fragments";

const importer = new IfcImporter();

// Configure WASM
importer.wasm.path = "https://unpkg.com/web-ifc@0.0.77/";
importer.wasm.absolute = true;

// Process IFC file → Fragments binary
const fragmentsData = await importer.process(ifcData);  // Uint8Array in, Uint8Array out
```

### IfcImporter Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `wasm` | `{ path, absolute }` | — | WASM configuration |
| `classes.elements` | `Set<number>` | common building types | Physical IFC elements to import |
| `classes.abstract` | `Set<number>` | materials, properties | Abstract types (properties, materials, classifications) |
| `relations` | `Map<number, object>` | key relations | Entity relationship mappings |
| `distanceThreshold` | `number \| null` | `100000` | Max distance from origin; items beyond are skipped |
| `includeRelationNames` | `boolean` | `false` | Include relation names in output |
| `includeUniqueAttributes` | `boolean` | `false` | Include unique attributes in output |
| `replaceSiteElevation` | `boolean` | `true` | Replace site elevation with absolute meters |
| `replaceStoreyElevation` | `boolean` | `true` | Replace storey elevation with absolute meters |
| `attributesToExclude` | `Set<string>` | — | Attributes filtered from serialization |

### IfcImporter Methods

| Method | Description |
|---|---|
| `process(data: Uint8Array)` | Convert IFC to Fragments binary (Uint8Array) |
| `addAllAttributes()` | Include all IFC attributes (increases output size) |
| `addAllRelations()` | Include all IFC relations (increases output size) |

---

## Fragment Conversion Workflow

The recommended production pattern: convert once, store fragments, reload fast.

```typescript
// Step 1: Convert IFC → FragmentsModel (slow, one-time)
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);

// Step 2: Export to Fragments binary (fast serialization)
const fragmentsData = model.export();  // Uint8Array

// Step 3: Store the binary (IndexedDB, server, file system)
await saveToStorage("building.frg", fragmentsData);

// Step 4: Reload from Fragments binary (fast, skips IFC parsing)
const stored = await loadFromStorage("building.frg");
const reloaded = fragmentsManager.load(stored);
world.scene.three.add(reloaded.object);
```

This workflow avoids re-parsing IFC on every page load. The Fragments binary format loads orders of magnitude faster than IFC conversion.

---

## Relationship to Other Skills

| Skill | Relationship |
|---|---|
| `thatopen-core-web-ifc` | Low-level WASM engine beneath IfcLoader. Use when you need raw `IfcAPI` access. |
| `thatopen-core-architecture` | Component framework that IfcLoader plugs into (`components.get()`). |
| `thatopen-core-fragments` | Fragment system that stores and renders loaded models. |
| `thatopen-syntax-properties` | Querying IFC properties after loading. |
| `thatopen-syntax-streaming` | Streaming large IFC files (alternative to full `load()`). |

---

## References

- `references/methods.md` — Complete IfcLoader and IfcImporter API signatures
- `references/examples.md` — Working examples: basic load, CDN WASM, IfcImporter, fragment workflow
- `references/anti-patterns.md` — Common failures: WASM misconfig, version mismatch, missing setup

## Sources

- API docs: https://docs.thatopen.com/api/@thatopen/components/classes/IfcLoader
- Tutorial: https://docs.thatopen.com/Tutorials/Components/Core/IfcLoader
- GitHub: https://github.com/ThatOpen/engine_components
- IfcImporter: https://docs.thatopen.com/api/@thatopen/fragments/classes/IfcImporter
