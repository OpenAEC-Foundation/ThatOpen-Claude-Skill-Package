---
name: thatopen-core-fragments
description: >
  Use when working with ThatOpen's fragment system for BIM model storage,
  loading, querying, or understanding the optimized geometry pipeline.
  Prevents worker initialization failures and memory leaks from undisposed models.
  Covers FragmentsManager, FragmentsModel, ModelIdMap, worker initialization,
  coordinate alignment, raycast, getData, GUID mapping.
  Keywords: fragments, fragmentsmanager, modelidmap, worker, instanced mesh, flatbuffers, bim model, load, dispose, raycast, how models are stored, fragment format, model data structure.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/fragments 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Fragment System

## Overview

The fragment system is ThatOpen's optimized geometry pipeline for BIM models.
It converts IFC files into a GPU-friendly binary format built on FlatBuffers
and THREE.InstancedMesh, enabling fast rendering of large models with millions
of elements. The `@thatopen/fragments` package provides the binary format,
worker architecture, and core operations. `FragmentsManager` in
`@thatopen/components` orchestrates model lifecycle, raycasting, data queries,
and coordinate alignment.

**Pipeline:**
```
IFC File → web-ifc (WASM) → IfcLoader → FragmentsModel (binary .frag)
                                              │
                                    ┌─────────┴─────────┐
                                    │   Fragment[]       │
                                    │  (InstancedMesh)   │
                                    └────────────────────┘
```

**Convert once, reload fast:** ALWAYS convert IFC to fragments once via
IfcLoader, then store the binary `.frag` file. Subsequent loads skip WASM
parsing entirely and load the pre-converted fragment binary directly.

## Critical Warnings

1. **ALWAYS call `FragmentsManager.init(workerURL)` before ANY fragment
   operation.** The worker handles raycasting, data queries, and model loading
   off the main thread. Omitting this causes silent failures or crashes.

2. **ALWAYS dispose models** via `FragmentsManager.disposeModel(modelId)` or
   `components.dispose()` when done. Fragment models hold GPU buffers
   (InstancedMesh geometry, textures) and worker state. Undisposed models
   cause memory leaks that crash browser tabs.

3. **NEVER skip coordinate alignment in multi-model scenarios.** Models from
   different origins will appear scattered in 3D space. Use
   `applyBaseCoordinateSystem()` to align them to a common origin.

4. **ALWAYS match the worker.mjs URL to your installed `@thatopen/fragments`
   version.** A version mismatch between the main-thread library and the
   worker script causes deserialization failures.

## Core Concepts

### Fragment Binary Format

Fragments use Google FlatBuffers for zero-copy binary serialization:

| Layer | Content |
|---|---|
| `FragmentGroup` (root) | Coordination matrix, IFC metadata, array of Fragments |
| `Fragment` | ID, type (Mesh/InstancedMesh/Point/Line), geometry, transforms, colors |
| `Geometry` | Position/normal/index arrays, groups, bounding box |
| `Transform` | 4x4 matrix, local ID, express ID per instance |

File identifier: `FRAG`. Dependencies: `flatbuffers`, `pako` (compression),
`earcut` (triangulation).

**Key advantage:** Geometry arrays (`position`, `index`) map directly to GPU
buffers as zero-copy typed array views. No deserialization step.

### GPU Instancing

Each Fragment wraps a `THREE.InstancedMesh`. Identical geometries (e.g., all
doors of the same type) share one GPU geometry buffer with per-instance
transform matrices. This reduces draw calls from thousands to dozens.

### FragmentsModel

A `FragmentsModel` represents one loaded BIM model. It contains:
- An array of `Fragment` objects (instanced meshes)
- The coordination matrix (world positioning)
- IFC metadata and property data
- GUID-to-localID mappings

Access loaded models via `FragmentsManager.list: Map<string, FragmentsModel>`.

## FragmentsManager API

```typescript
class FragmentsManager extends Component implements Disposable {
  // State
  list: Map<string, FragmentsModel>;
  initialized: boolean;
  baseCoordinationModel: string;
  baseCoordinationMatrix: THREE.Matrix4;

  // Initialization — MUST call before any operation
  init(workerURL: string, options?): void;

  // Raycasting
  raycast(config: {
    camera: THREE.Camera,
    mouse: THREE.Vector2,
    dom: HTMLElement,
    snappingClasses?: number[]
  }): Promise<Result | undefined>;

  // Visual operations
  highlight(style: MaterialDefinition, items?: ModelIdMap): Promise<void>;
  resetHighlight(items?: ModelIdMap): Promise<void>;

  // Data queries
  getData(items: ModelIdMap, config?): Promise<Record<string, ItemData[]>>;
  getPositions(items: ModelIdMap): Promise<THREE.Vector3[]>;
  getBBoxes(items: ModelIdMap): Promise<THREE.Box3[]>;

  // GUID mapping
  guidsToModelIdMap(guids: Iterable<string>): Promise<ModelIdMap>;
  modelIdMapToGuids(modelIdMap: ModelIdMap): Promise<string[]>;

  // Coordinate alignment
  applyBaseCoordinateSystem(object: THREE.Object3D, originalMatrix?: THREE.Matrix4): THREE.Matrix4;

  // Disposal
  disposeModel(modelId: string): void;
}
```

**Events:**
- `onFragmentsLoaded: Event<FragmentsModel>` — fires after a model is loaded
- `onBeforeDispose: Event<FragmentsModel>` — fires before model disposal
- `onDisposed: Event<void>` — fires after FragmentsManager itself is disposed

## ModelIdMap

The universal data structure for targeting items across models:

```typescript
type ModelIdMap = Record<string, Set<number>>;
// Keys: model UUID strings
// Values: sets of local element IDs (numbers)
```

Used by: FragmentsManager, Hider, Classifier, BoundingBoxer, Highlighter,
and every component that operates on specific BIM elements.

**ALWAYS use ModelIdMap** to reference items. NEVER reference items by
expressID alone — expressIDs are only unique within a single model.

## Worker Architecture

FragmentsManager offloads heavy operations (raycasting, data extraction,
model loading) to a dedicated web worker.

```typescript
// Worker initialization — ALWAYS do this first
const fragments = components.get(OBC.FragmentsManager);
fragments.init("https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs");
```

**What runs in the worker:**
- FlatBuffers deserialization
- Raycast intersection tests
- Property data extraction (getData)
- Position and bounding box calculations
- GUID-to-ID mapping

**What stays on the main thread:**
- THREE.InstancedMesh creation and scene graph management
- Highlight/resetHighlight (GPU material swaps)
- Coordinate alignment (matrix multiplication)

## Coordinate Alignment

When loading multiple models, each may have a different world origin stored
in its coordination matrix.

```typescript
// Set the first loaded model as the base
fragments.baseCoordinationModel = firstModel.modelId;
fragments.baseCoordinationMatrix = firstModel.coordinationMatrix;

// Align subsequent models
fragments.applyBaseCoordinateSystem(secondModel, secondModel.coordinationMatrix);
```

**NEVER skip this step for multi-model federation.** Models will appear at
wrong positions without alignment.

## Data Operations

### getData: Extract IFC Properties

```typescript
const items: ModelIdMap = { [model.modelId]: new Set([42, 43, 44]) };
const data = await fragments.getData(items);
// Returns: Record<string, ItemData[]>
// Keys are model IDs, values are arrays of property data per element
```

### getPositions: Get 3D Coordinates

```typescript
const positions = await fragments.getPositions(items);
// Returns: THREE.Vector3[] — center positions of targeted elements
```

### getBBoxes: Get Bounding Boxes

```typescript
const boxes = await fragments.getBBoxes(items);
// Returns: THREE.Box3[] — axis-aligned bounding boxes
```

## GUID Mapping

Convert between IFC GlobalId (GUID) strings and ModelIdMap:

```typescript
// GUIDs → ModelIdMap (for targeting elements by GUID)
const items = await fragments.guidsToModelIdMap(["2O2Fr$t4X7Zf8NOew3FLOH"]);

// ModelIdMap → GUIDs (for exporting selections)
const guids = await fragments.modelIdMapToGuids(items);
```

## Highlight and Raycast

### Raycasting

```typescript
const result = await fragments.raycast({
  camera: world.camera.three,
  mouse: new THREE.Vector2(normalizedX, normalizedY),
  dom: renderer.three.domElement,
  snappingClasses: [IFCWALL, IFCSLAB] // optional: restrict hit targets
});

if (result) {
  console.log(result.modelId, result.localId, result.point);
}
```

### Highlighting

```typescript
// Define a highlight style (material definition)
const style: MaterialDefinition = {
  color: new THREE.Color("#BCF124"),
  opacity: 0.6
};

// Highlight specific items
await fragments.highlight(style, items);

// Reset to original appearance
await fragments.resetHighlight(items);
```

## IFC-to-Fragment Pipeline

The recommended workflow for production:

1. **First time:** Convert IFC via IfcLoader, export binary
2. **Subsequent loads:** Load the binary directly (10-100x faster)

```typescript
// Step 1: Convert IFC to FragmentsModel
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
const model = await ifcLoader.load(ifcBytes, true, "MyBuilding");

// Step 2: Export as binary for storage
// The FragmentsModel binary can be persisted (IndexedDB, server, etc.)

// Step 3: On next load, skip IFC parsing entirely
// Load the pre-converted fragment binary through FragmentsManager
```

## Dependencies

| Package | Purpose |
|---|---|
| `@thatopen/fragments` | Core fragment engine, FlatBuffers format, worker |
| `flatbuffers` | Binary serialization (zero-copy reads) |
| `pako` | Compression/decompression of fragment binaries |
| `earcut` | Polygon triangulation for 2D profiles |
| `three` (>=0.175) | 3D rendering, InstancedMesh, scene graph |
| `web-ifc` (>=0.0.74) | IFC parsing (used by IfcLoader, not fragments directly) |

## Quick Reference

| Task | Method |
|---|---|
| Initialize worker | `fragments.init(workerURL)` |
| Get loaded models | `fragments.list` |
| Raycast scene | `fragments.raycast({camera, mouse, dom})` |
| Get element properties | `fragments.getData(items)` |
| Get element positions | `fragments.getPositions(items)` |
| Get bounding boxes | `fragments.getBBoxes(items)` |
| GUID to items | `fragments.guidsToModelIdMap(guids)` |
| Items to GUIDs | `fragments.modelIdMapToGuids(items)` |
| Highlight elements | `fragments.highlight(style, items)` |
| Reset highlights | `fragments.resetHighlight(items)` |
| Align coordinates | `fragments.applyBaseCoordinateSystem(obj, matrix)` |
| Dispose a model | `fragments.disposeModel(modelId)` |
| Dispose everything | `components.dispose()` |

## Related Skills

- `thatopen-core-architecture` — Component system, world setup, lifecycle
- `thatopen-syntax-ifc-loading` — IfcLoader configuration and WASM setup
- `thatopen-syntax-properties` — Deep property extraction from IFC data
- `thatopen-impl-viewer` — Full viewer setup including fragments initialization

## References

- [references/methods.md](references/methods.md) — Complete FragmentsManager API, FragmentsModel, ModelIdMap details
- [references/examples.md](references/examples.md) — Init worker, load model, getData, raycast, coordinate alignment examples
- [references/anti-patterns.md](references/anti-patterns.md) — Missing worker init, disposal failures, common mistakes
