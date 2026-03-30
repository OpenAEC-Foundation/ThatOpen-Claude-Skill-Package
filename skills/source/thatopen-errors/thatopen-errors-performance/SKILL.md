---
name: thatopen-errors-performance
description: >
  Use when experiencing memory leaks, slow rendering, browser tab crashes,
  or performance issues with large BIM models.
  Prevents memory leaks from undisposed resources and main thread blocking.
  Covers disposal patterns, memory management, worker usage, BVH raycasting,
  polygon offset z-fighting, fragment optimization, large model strategies.
  Keywords: performance, memory, leak, dispose, slow, crash, large model, worker, optimization, z-fighting, disposal, gpu, browser, laggy, low FPS, browser freezes, out of memory.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Performance and Memory Management

## Overview

BIM models are large. A typical building model contains millions of triangles,
thousands of IFC elements, and dozens of material definitions. Without proper
memory management, a single model can consume gigabytes of browser memory and
crash the tab. This skill covers every performance-critical pattern for
ThatOpen applications: disposal chains, memory budgets, worker offloading,
GPU optimization, and strategies for handling models that exceed normal limits.

## Critical Rules

1. **ALWAYS dispose resources when done.** Every `FragmentsModel`, `THREE.Mesh`,
   `THREE.Material`, and `THREE.Texture` holds GPU memory. The browser garbage
   collector does NOT free GPU resources — you must call `.dispose()` explicitly.

2. **ALWAYS dispose in the correct order.** `components.dispose()` handles
   ordering internally (FragmentsManager last). When disposing manually,
   ALWAYS dispose dependents before their dependencies.

3. **NEVER forget to dispose Three.js objects created outside components.**
   Custom meshes, materials, geometries, and textures added to the scene are
   YOUR responsibility. `components.dispose()` only cleans component-managed
   resources.

4. **ALWAYS initialize FragmentsManager with a worker URL.** The worker
   offloads raycasting, data queries, and model loading from the main thread.
   Without it, large model operations freeze the UI.

5. **NEVER load raw IFC repeatedly.** Convert IFC to fragments once, store the
   binary, reload from fragments. IFC parsing is 10-100x slower than fragment
   loading.

## Disposal Patterns

### Full Application Disposal

When tearing down the entire viewer (route change, component unmount):

```typescript
// This is the ONLY call needed for full cleanup.
// components.dispose() walks all registered components and disposes them.
// It disposes FragmentsManager LAST (it depends on other components).
components.dispose();
```

**What `components.dispose()` does internally:**
1. Sets `components.enabled = false` (stops the render loop)
2. Iterates all components in `components.list`
3. Calls `.dispose()` on each `Disposable` component
4. Disposes `FragmentsManager` last (ensures dependent components clean up first)
5. Fires `components.onDisposed` event

### Per-Model Disposal

When removing a single model while the viewer stays active:

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Dispose a specific model by its ID
fragments.disposeModel(modelId);
```

**What `disposeModel()` does:**
1. Fires `onBeforeDispose` event with the model
2. Removes all fragments (InstancedMesh instances) from the scene
3. Disposes GPU buffers (geometry, materials) for each fragment
4. Terminates worker state for that model
5. Removes the model from `fragments.list`

### Three.js Manual Disposal

For custom Three.js objects you create yourself:

```typescript
// Geometry — ALWAYS dispose
geometry.dispose();

// Material — ALWAYS dispose, including its maps
material.dispose();
if (material.map) material.map.dispose();
if (material.normalMap) material.normalMap.dispose();
if (material.aoMap) material.aoMap.dispose();
if (material.envMap) material.envMap.dispose();

// Render target — if you create custom render targets
renderTarget.dispose();

// Remove from scene BEFORE disposing
scene.remove(mesh);
mesh.geometry.dispose();
mesh.material.dispose();
```

### Disposal Checklist

Use this checklist when implementing cleanup logic:

- [ ] `components.dispose()` called on application teardown
- [ ] `fragments.disposeModel(id)` called for individual model removal
- [ ] Custom geometries disposed via `geometry.dispose()`
- [ ] Custom materials disposed via `material.dispose()`
- [ ] Texture maps disposed (map, normalMap, aoMap, envMap, etc.)
- [ ] Custom meshes removed from scene before disposal
- [ ] Event listeners removed (`removeEventListener`)
- [ ] `requestAnimationFrame` loops cancelled
- [ ] DOM references cleared (container elements)

## Memory Leak Detection

### Symptoms of Memory Leaks

| Symptom | Likely Cause |
|---|---|
| Tab crashes after loading/unloading models | Undisposed fragment models |
| Memory grows on every model load cycle | Missing `disposeModel()` calls |
| GPU process crashes | Undisposed geometries or textures |
| Gradual slowdown over time | Accumulating event listeners or meshes |
| Memory never decreases after unload | Three.js objects not disposed |

### DevTools Memory Profiling

```
1. Open Chrome DevTools > Memory tab
2. Take heap snapshot BEFORE loading a model
3. Load the model — note memory increase
4. Dispose the model
5. Force garbage collection (trash can icon)
6. Take heap snapshot AFTER disposal
7. Compare: search for "BufferGeometry", "Material", "Texture"
8. Any remaining instances = leak
```

**Key objects to watch in heap snapshots:**
- `BufferGeometry` — GPU geometry buffers
- `InstancedBufferAttribute` — per-instance transforms
- `MeshLambertMaterial` / `MeshBasicMaterial` — materials
- `Texture` / `DataTexture` — texture data
- `WebGLProgram` — compiled shaders

### Memory Budget Guidelines

| Model Size | Elements | Approx. Memory | Strategy |
|---|---|---|---|
| Small | < 10,000 | < 200 MB | Full load, no special handling |
| Medium | 10,000 - 100,000 | 200 - 800 MB | Full load, filter IFC classes |
| Large | 100,000 - 500,000 | 800 MB - 2 GB | Filter aggressively, ALWAYS use streaming |
| Very Large | > 500,000 | > 2 GB | Fragment streaming mandatory, federate |

**Browser memory limits:**
- Chrome: ~4 GB per tab (64-bit), ~1 GB (32-bit)
- Firefox: ~4 GB per tab
- Safari: ~3 GB per tab (more aggressive about killing tabs)
- Mobile browsers: ~1-2 GB total

## Worker Thread Usage

### Why Workers Matter

Fragment operations that run in the worker (off main thread):
- FlatBuffers deserialization (model loading)
- Raycast intersection calculations
- Property data extraction (`getData`)
- Position and bounding box calculations
- GUID-to-ID mapping lookups

Without the worker, ALL of these block the main thread. For a 100K-element
model, a single `getData` call can freeze the UI for several seconds.

### Worker Initialization

```typescript
const fragments = components.get(OBC.FragmentsManager);

// ALWAYS init with worker URL matching your installed fragments version
fragments.init(
  "https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs"
);
```

**ALWAYS match the worker URL to your installed `@thatopen/fragments` version.**
A version mismatch between the main thread library and worker script causes
deserialization failures and silent data corruption.

### Main Thread vs Worker Thread

| Operation | Thread | Blocking? |
|---|---|---|
| Model loading (FlatBuffers) | Worker | No |
| Raycasting | Worker | No |
| getData / getPositions / getBBoxes | Worker | No |
| GUID mapping | Worker | No |
| Scene graph updates | Main | Yes (fast) |
| Highlight / resetHighlight | Main | Yes (fast) |
| Coordinate alignment | Main | Yes (fast) |
| Three.js rendering | Main | Yes (per frame) |

## BVH Raycasting

ThatOpen uses `three-mesh-bvh` for accelerated raycasting. This is
auto-patched onto `THREE.BufferGeometry` in the `Components` constructor.

**No manual setup needed.** When you call `components.get(OBC.Components)` or
create a `new Components()`, the BVH extension is automatically applied.

**Performance impact:** Without BVH, raycasting a 100K-triangle model tests
every triangle (O(n)). With BVH, it traverses a bounding volume hierarchy
(O(log n)) — typically 100-1000x faster.

**NEVER remove or override the BVH patch.** If you import Three.js separately,
ensure the BVH patch is applied before creating geometries.

## Polygon Offset (Z-Fighting Prevention)

When two surfaces overlap at the same depth (e.g., a floor slab and a floor
finish), the GPU cannot determine which is in front. This causes z-fighting:
flickering pixels alternating between the two surfaces.

### Solution: Polygon Offset

```typescript
// Apply polygon offset to materials that overlap other surfaces
material.polygonOffset = true;
material.polygonOffsetFactor = 1;
material.polygonOffsetUnits = 1;
```

**When to use polygon offset:**
- Highlight overlays on existing geometry
- Section fill materials on clipping planes
- Floor finish layers on structural slabs
- Any custom overlay material

**ALWAYS apply polygon offset to highlight and overlay materials.** ThatOpen's
built-in highlight system handles this automatically, but custom materials
you create need manual polygon offset.

## Large Model Strategies

### Strategy 1: IFC Class Filtering

Reduce memory by loading only the IFC classes you need:

```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// The default import configuration already excludes some classes
// for memory efficiency. You can further restrict it.
// Only configure the classes you actually need in your application.
```

**Default excluded classes** (not imported unless explicitly added):
Some geometry-heavy classes like `IFCOPENINGELEMENT` and `IFCSPACE` are
excluded by default to save memory. Check `ifcLoader.settings` for the
current default import list.

### Strategy 2: Convert Once, Load Fragments

```typescript
// FIRST TIME: Convert IFC (slow, memory-intensive)
const model = await ifcLoader.load(ifcBytes, true, "Building");

// STORE: Export the FragmentsModel binary to IndexedDB or server

// SUBSEQUENT LOADS: Skip IFC entirely (10-100x faster)
// Load the pre-converted .frag binary through FragmentsManager
```

This is the single most impactful performance optimization. IFC parsing
requires WASM, schema interpretation, and geometry tessellation. Fragment
loading is just FlatBuffers deserialization — nearly instant.

### Strategy 3: Model Federation

Instead of loading one massive model, split into discipline-specific models:

| Discipline | Typical Size | Load Priority |
|---|---|---|
| Architecture | Large | First (base model) |
| Structure | Medium | On demand |
| MEP / HVAC | Large | On demand |
| Site | Small | Optional |

```typescript
// Load the base architecture model first
const archModel = await loadFragments("architecture.frag");
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;

// Load additional disciplines on demand
const structModel = await loadFragments("structure.frag");
fragments.applyBaseCoordinateSystem(structModel, structModel.coordinationMatrix);
```

**ALWAYS set the base coordination model from the first loaded model.**
Subsequent models must be aligned to this base.

### Strategy 4: Selective Loading and Disposal

Load models as users navigate, dispose when they leave the area:

```typescript
// User opens floor 3 → load floor 3 model
const floor3 = await loadFragments("floor3.frag");

// User navigates to floor 7 → dispose floor 3, load floor 7
fragments.disposeModel(floor3.modelId);
const floor7 = await loadFragments("floor7.frag");
```

## Performance Optimization Checklist

### Before Loading
- [ ] Worker initialized with correct version URL
- [ ] Using pre-converted fragments (not raw IFC for repeat loads)
- [ ] IFC class filtering configured to exclude unnecessary types
- [ ] Memory budget estimated for target model size

### During Runtime
- [ ] No unnecessary re-renders (check `renderer.update()` frequency)
- [ ] Raycasting uses BVH (automatic, but verify no override)
- [ ] Polygon offset applied to overlay materials
- [ ] Heavy operations (getData, getPositions) called sparingly

### On Cleanup
- [ ] `components.dispose()` called on teardown
- [ ] Individual models disposed when no longer needed
- [ ] Custom Three.js objects disposed manually
- [ ] Event listeners removed
- [ ] DOM references cleared

## Common Performance Problems

### Problem: Browser Tab Crashes

**Cause:** Memory exhaustion from loading too-large models.

**Solution:**
1. Check model element count before loading
2. Filter IFC classes to reduce geometry
3. Use model federation (split by discipline/floor)
4. Monitor `performance.memory.usedJSHeapSize` (Chrome only)

### Problem: UI Freezes During Operations

**Cause:** Heavy operations running on main thread without worker.

**Solution:**
1. Verify worker is initialized: `fragments.initialized === true`
2. Use async APIs: `await fragments.getData(items)` (runs in worker)
3. Batch operations instead of per-element calls

### Problem: Gradual Slowdown

**Cause:** Accumulating undisposed resources across load/unload cycles.

**Solution:**
1. Profile with DevTools Memory tab
2. Ensure `disposeModel()` is called before loading replacement
3. Check for custom Three.js objects not being disposed
4. Verify event listeners are properly removed

### Problem: Z-Fighting Flickering

**Cause:** Overlapping surfaces at same depth without polygon offset.

**Solution:**
1. Apply `polygonOffset = true` to overlay materials
2. Use `polygonOffsetFactor = 1` and `polygonOffsetUnits = 1`
3. ThatOpen highlights handle this automatically — check custom materials

## Related Skills

- `thatopen-core-fragments` — Fragment system, worker init, model lifecycle
- `thatopen-core-architecture` — Component system, disposal via components
- `thatopen-syntax-ifc-loading` — IfcLoader settings, class filtering
- `thatopen-errors-loading` — Loading failures, WASM issues, error recovery

## References

- [references/methods.md](references/methods.md) — Disposal methods, memory APIs, performance-related APIs
- [references/examples.md](references/examples.md) — Proper disposal, worker setup, memory monitoring patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Memory leaks, missing disposal, main thread blocking
