---
name: thatopen-impl-federation
description: >
  Use when loading multiple IFC models, coordinating model positions,
  controlling per-model visibility, or working with multi-model scenarios.
  Prevents coordinate misalignment and visibility management errors.
  Covers multi-model loading, coordination matrix, Hider (set/isolate/toggle),
  BoundingBoxer (fit, camera orientation), per-model visibility, cross-model
  queries via ModelIdMap.
  Keywords: federation, multi-model, coordination, hider, visibility, isolate, bounding box, alignment, model, hide, show, load multiple models, combine IFC files, show hide elements.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Model Federation

## Overview

Model federation is the practice of loading multiple IFC models into a single
3D scene, aligning them to a shared coordinate system, and controlling their
visibility independently. In ThatOpen Engine, federation is not a single
component but a workflow that combines FragmentsManager (loading, alignment),
Hider (visibility), BoundingBoxer (spatial queries), and Classifier (per-model
grouping).

**Core principle:** Each model loaded via IfcLoader produces an independent
FragmentsModel. These models may originate from different authoring tools with
different world origins. ALWAYS apply coordinate alignment before interacting
with multi-model scenes.

**Federation workflow:**
```
Load Model A (IfcLoader) → Set as base coordination model
Load Model B (IfcLoader) → Align to base via applyBaseCoordinateSystem()
Load Model C (IfcLoader) → Align to base via applyBaseCoordinateSystem()
        │
        ├─ Classify by model → Classifier.byModel()
        ├─ Control visibility → Hider.set() / isolate() / toggle()
        ├─ Compute bounds    → BoundingBoxer.addFromModels()
        └─ Query cross-model → Classifier.find() with ModelIdMap
```

## Critical Warnings

1. **ALWAYS call `applyBaseCoordinateSystem()` for every model after the
   first.** Without coordination, models will appear scattered across 3D space
   at their original authoring origins. This is the most common federation bug.

2. **ALWAYS set `baseCoordinationModel` and `baseCoordinationMatrix` before
   loading additional models.** The base defines the shared origin. All
   subsequent models are transformed relative to it.

3. **NEVER assume models share the same coordinate origin.** Even models from
   the same project may use different site placement or project base points.

4. **ALWAYS use ModelIdMap for cross-model operations.** A local element ID
   is only unique within its model. Use `Record<string, Set<number>>` to
   address elements across models unambiguously.

5. **ALWAYS dispose individual models via `FragmentsManager.disposeModel()`
   when removing them from a federated scene.** Do not rely on full
   `components.dispose()` for selective unloading.

## Multi-Model Loading

### Sequential Loading Pattern

Load models one at a time through IfcLoader. The first model sets the
coordination base; subsequent models are aligned to it.

```typescript
import * as OBC from "@thatopen/components";
import * as THREE from "three";

const components = new OBC.Components();
const fragments = components.get(OBC.FragmentsManager);
const ifcLoader = components.get(OBC.IfcLoader);

// Initialize worker and loader
fragments.init(workerURL);
await ifcLoader.setup();

// Load first model — becomes the base
const architecturalBytes = new Uint8Array(/* ... */);
const archModel = await ifcLoader.load(architecturalBytes, true, "Architectural");

// Set coordination base from first model
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;

// Load second model — coordinate=true triggers alignment
const structuralBytes = new Uint8Array(/* ... */);
const structModel = await ifcLoader.load(structuralBytes, true, "Structural");

// Manually align if needed (automatic when coordinate=true in load)
fragments.applyBaseCoordinateSystem(structModel, structModel.coordinationMatrix);

// Load third model
const mepBytes = new Uint8Array(/* ... */);
const mepModel = await ifcLoader.load(mepBytes, true, "MEP");
fragments.applyBaseCoordinateSystem(mepModel, mepModel.coordinationMatrix);
```

### Coordination Matrix

The coordination matrix is a 4x4 transformation matrix stored in each IFC
file's header. It encodes the model's position, rotation, and scale relative
to a world origin.

```typescript
// FragmentsManager coordination properties
fragments.baseCoordinationModel: string;       // model ID of the reference model
fragments.baseCoordinationMatrix: THREE.Matrix4; // matrix of the reference model

// Apply alignment to any THREE.Object3D
fragments.applyBaseCoordinateSystem(
  object: THREE.Object3D,            // the object to transform
  originalMatrix?: THREE.Matrix4     // its original coordination matrix
): THREE.Matrix4;                    // returns the applied transformation
```

**How alignment works:** The method computes the inverse of the base matrix,
multiplies it with the object's original matrix, and applies the result. This
effectively re-parents the object into the base model's coordinate space.

### Adding Models to the World

After loading and aligning, add models to the world scene:

```typescript
const world = worlds.create();
// ... scene, camera, renderer setup ...

// Models are automatically added to the scene on load
// To verify:
for (const [id, model] of fragments.list) {
  console.log(`Model ${id}: ${model.name}`);
}
```

## Hider: Visibility Control

The Hider component controls element visibility at the fragment level. It
operates on ModelIdMap, making it ideal for per-model and cross-model
visibility management.

### API

```typescript
class Hider extends Component {
  // Show or hide elements
  set(visible: boolean, modelIdMap?: ModelIdMap): void;

  // Show ONLY the specified elements, hide everything else
  isolate(modelIdMap: ModelIdMap): void;

  // Toggle visibility of specified elements
  toggle(modelIdMap: ModelIdMap): void;

  // Get current visibility state
  getVisibilityMap(state: boolean, modelIds?: string[]): Map<string, Set<number>>;
}
```

### Per-Model Visibility

```typescript
const hider = components.get(OBC.Hider);
const classifier = components.get(OBC.Classifier);

// Classify models
classifier.byModel();

// Get ModelIdMap for a specific model
const structuralItems = await classifier.find({
  models: ["Structural"]
});

// Hide the structural model
hider.set(false, structuralItems);

// Show it again
hider.set(true, structuralItems);
```

### Isolate a Single Model

```typescript
// Show ONLY the MEP model, hide everything else
const mepItems = await classifier.find({
  models: ["MEP"]
});
hider.isolate(mepItems);
```

### Toggle Visibility

```typescript
// Toggle architectural model on/off
const archItems = await classifier.find({
  models: ["Architectural"]
});
hider.toggle(archItems);
```

### Hide by Category Across Models

```typescript
// Classify by IFC category
classifier.byCategory();

// Hide all walls across ALL models
const allWalls = await classifier.find({
  categories: ["IFCWALL"]
});
hider.set(false, allWalls);
```

### Get Visibility State

```typescript
// Get all currently visible items
const visibleItems = hider.getVisibilityMap(true);

// Get visible items for specific models only
const visibleInArch = hider.getVisibilityMap(true, [archModel.modelId]);

// Get all currently hidden items
const hiddenItems = hider.getVisibilityMap(false);
```

### Show All (Reset Visibility)

```typescript
// Show everything — call set(true) with no ModelIdMap
hider.set(true);
```

## BoundingBoxer: Spatial Queries

BoundingBoxer computes axis-aligned bounding boxes for elements or entire
models. Use it to fit the camera to selections, compute model extents, or
orient the camera for specific views.

### API

```typescript
class BoundingBoxer extends Component {
  // Add items to the bounding box computation
  addFromModelIdMap(items: ModelIdMap): void;
  addFromModels(modelIds?: string[]): void;

  // Get the computed bounding box
  get(): THREE.Box3;

  // Get center point of items
  getCenter(modelIdMap: ModelIdMap): THREE.Vector3;

  // Get camera position and target for a given orientation
  getCameraOrientation(
    orientation: "front" | "back" | "left" | "right" | "top" | "bottom",
    offsetFactor?: number
  ): { position: THREE.Vector3; target: THREE.Vector3 };
}
```

### Fit Camera to All Models

```typescript
const boxer = components.get(OBC.BoundingBoxer);

// Add all loaded models
boxer.addFromModels();

// Get the unified bounding box
const box = boxer.get();

// Fit camera to encompass all models
const camera = world.camera as OBC.OrthoPerspectiveCamera;
camera.fit([box]);
```

### Fit Camera to a Selection

```typescript
// Compute bounding box for specific items
const selectedItems: ModelIdMap = {
  [archModel.modelId]: new Set([101, 102, 103]),
  [structModel.modelId]: new Set([201, 202])
};

boxer.addFromModelIdMap(selectedItems);
const selectionBox = boxer.get();
camera.fit([selectionBox]);
```

### Camera Orientation for Views

```typescript
// Get camera position for a front view of the model
boxer.addFromModels();
const frontView = boxer.getCameraOrientation("front", 1.5);
// frontView.position → THREE.Vector3 (camera position)
// frontView.target   → THREE.Vector3 (look-at point)

// Apply to camera
camera.controls.setLookAt(
  frontView.position.x, frontView.position.y, frontView.position.z,
  frontView.target.x, frontView.target.y, frontView.target.z,
  true // animate
);
```

### Get Center of Items

```typescript
const center = boxer.getCenter(selectedItems);
// center → THREE.Vector3 at the centroid of the selected elements
```

## Cross-Model Queries

### Classifier.byModel

Group all loaded elements by their source model. This is the foundation for
per-model operations.

```typescript
const classifier = components.get(OBC.Classifier);

// Create model-based classification
classifier.byModel();

// The classifier.list now contains a "models" group
// Each entry maps a model name to its ModelIdMap
```

### Classifier.find with Filters

```typescript
// Find items by model name
const archItems = await classifier.find({ models: ["Architectural"] });

// Find items by category within a specific model
const archWalls = await classifier.find({
  models: ["Architectural"],
  categories: ["IFCWALL"]
});

// Find items by storey
const groundFloor = await classifier.find({
  storeys: ["Ground Floor"]
});
```

## Model Lifecycle

ALWAYS use `coordinate=true` in `ifcLoader.load()` for federated models.
Track loads via `fragments.onFragmentsLoaded` to auto-classify new models.
Dispose individual models with `fragments.disposeModel(modelId)`. Clean up
everything with `components.dispose()`.

## Complete Federation Workflow

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// 1. Setup
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

const fragments = components.get(OBC.FragmentsManager);
fragments.init(workerURL);
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

const hider = components.get(OBC.Hider);
const classifier = components.get(OBC.Classifier);
const boxer = components.get(OBC.BoundingBoxer);

// 2. Load and coordinate models
const archModel = await ifcLoader.load(archBytes, true, "Architectural");
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;

const structModel = await ifcLoader.load(structBytes, true, "Structural");
const mepModel = await ifcLoader.load(mepBytes, true, "MEP");

// 3. Classify
classifier.byModel();
classifier.byCategory();
classifier.byIfcBuildingStorey();

// 4. Fit camera to all models
boxer.addFromModels();
const camera = world.camera as OBC.OrthoPerspectiveCamera;
camera.fit([boxer.get()]);

// 5. Per-model visibility controls (e.g., UI toggle)
async function toggleModel(modelName: string) {
  const items = await classifier.find({ models: [modelName] });
  hider.toggle(items);
}

// 6. Isolate a discipline
async function isolateDiscipline(modelName: string) {
  const items = await classifier.find({ models: [modelName] });
  hider.isolate(items);
}

// 7. Show all
function showAll() {
  hider.set(true);
}

// 8. Cleanup
function removeModel(modelId: string) {
  fragments.disposeModel(modelId);
}
```

## Quick Reference

| Task | Method |
|---|---|
| Set coordination base | `fragments.baseCoordinationModel = id` + `fragments.baseCoordinationMatrix = matrix` |
| Align model to base | `fragments.applyBaseCoordinateSystem(object, matrix)` |
| Hide elements | `hider.set(false, modelIdMap)` |
| Show elements | `hider.set(true, modelIdMap)` |
| Show all | `hider.set(true)` |
| Isolate elements | `hider.isolate(modelIdMap)` |
| Toggle visibility | `hider.toggle(modelIdMap)` |
| Get visibility state | `hider.getVisibilityMap(true/false, modelIds?)` |
| Bounding box from models | `boxer.addFromModels(modelIds?)` |
| Bounding box from items | `boxer.addFromModelIdMap(items)` |
| Get box | `boxer.get()` |
| Get center | `boxer.getCenter(items)` |
| Camera orientation | `boxer.getCameraOrientation(direction, offset?)` |
| Classify by model | `classifier.byModel()` |
| Find by model | `classifier.find({ models: [name] })` |
| Dispose single model | `fragments.disposeModel(modelId)` |

## Related Skills

- `thatopen-core-fragments` — FragmentsManager, ModelIdMap, worker initialization
- `thatopen-core-architecture` — Component system, world setup, lifecycle
- `thatopen-syntax-ifc-loading` — IfcLoader configuration and WASM setup
- `thatopen-impl-selection` — Highlighter for visual selection in federated scenes
- `thatopen-impl-viewer` — Full viewer setup with fragments initialization

## References

- [references/methods.md](references/methods.md) — Hider, BoundingBoxer, coordination matrix APIs
- [references/examples.md](references/examples.md) — Multi-model load, isolate, visibility, bounding box
- [references/anti-patterns.md](references/anti-patterns.md) — Missing coordination, wrong isolation
