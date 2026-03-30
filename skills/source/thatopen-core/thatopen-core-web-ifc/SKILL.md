---
name: thatopen-core-web-ifc
description: >
  Use when working with web-ifc directly for IFC parsing, querying IFC
  entities, extracting geometry, or understanding the WASM engine beneath
  ThatOpen components.
  Prevents WASM initialization failures and memory leaks from unclosed models.
  Covers IfcAPI, SetWasmPath, Init, OpenModel, GetLine, GetFlatMesh,
  StreamAllMeshes, properties helper, LoaderSettings, schema detection.
  Keywords: web-ifc, wasm, ifc, parser, ifcapi, geometry, properties, spatial structure, express id, flatmesh, parse IFC in browser, read IFC JavaScript, IFC web.
license: MIT
compatibility: "Designed for Claude Code. Requires web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# web-ifc Engine: Direct WASM IFC Parser

## Overview

web-ifc is the WASM-powered IFC parsing engine beneath the ThatOpen component stack. It reads and writes IFC files at native speed in browser and Node.js environments. The central class is `IfcAPI`.

- **Package**: `web-ifc` (npm)
- **Source**: https://github.com/ThatOpen/engine_web-ifc
- **Environments**: Browser, Node.js (single-threaded and multi-threaded WASM)
- **License**: MPL-2.0

> When using ThatOpen components (`@thatopen/components`), you rarely call web-ifc directly — the `IfcLoader` component wraps it. Use this skill when you need low-level IFC access, custom geometry extraction, or bulk property queries outside the component framework.

---

## Critical Warnings

1. **ALWAYS call `Init()` before any other method** (except `SetWasmPath`). Every method silently fails or throws without WASM initialization.
2. **ALWAYS call `CloseModel(modelID)` when done** — each open model holds significant WASM heap memory. Forgetting this causes memory leaks that crash browser tabs.
3. **ALWAYS use `.size()` and `.get(i)` for `Vector<T>` access** — NEVER use array indexing (`[]`). WASM vectors are not JavaScript arrays.
4. **ALWAYS call `Dispose()` when the IfcAPI instance is no longer needed** — this releases the entire WASM module.
5. **NEVER create multiple `IfcAPI` instances** — one instance handles multiple models. Extra instances waste memory by loading duplicate WASM modules.
6. **Vertex format is 6 floats per vertex**: `[x, y, z, nx, ny, nz]` — position followed by normal. ALWAYS account for this interleaved layout when extracting geometry.
7. **All 4x4 matrices are column-major** (16 floats) — directly compatible with `THREE.Matrix4.fromArray()`.

---

## Quick Start

```typescript
import * as WebIFC from "web-ifc";

const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath("/wasm/");       // MUST be called before Init()
await ifcApi.Init();

const data = new Uint8Array(buffer); // from fetch or fs.readFile
const modelID = ifcApi.OpenModel(data);

// Query walls
const wallIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
for (let i = 0; i < wallIDs.size(); i++) {
  const wall = ifcApi.GetLine(modelID, wallIDs.get(i));
  console.log(wall.Name?.value);
}

// Get geometry
const mesh = ifcApi.GetFlatMesh(modelID, wallIDs.get(0));

// Get coordination matrix
const matrix = ifcApi.GetCoordinationMatrix(modelID);

ifcApi.CloseModel(modelID); // ALWAYS free memory
```

---

## Initialization

### `SetWasmPath(path: string, absolute?: boolean): void`

Sets the directory containing WASM files. MUST be called **before** `Init()`. The directory MUST contain `web-ifc.wasm` (and `web-ifc-mt.wasm` for multi-threaded mode).

```typescript
ifcApi.SetWasmPath("/static/wasm/");                         // relative
ifcApi.SetWasmPath("https://cdn.example.com/wasm/", true);   // absolute URL
```

NEVER hardcode a version in the WASM path — ALWAYS match the installed `web-ifc` npm version.

### `Init(customLocateFileHandler?, forceSingleThread?): Promise<void>`

Initializes the WASM module. MUST be awaited before calling any other API method.

```typescript
await ifcApi.Init();                        // default (auto-detect threading)
await ifcApi.Init(undefined, true);         // force single-threaded
await ifcApi.Init((path, prefix) => "/custom/" + path); // custom file locator
```

### `Dispose(): void`

Releases the entire WASM module and all resources. Call when the IfcAPI instance is no longer needed.

---

## Model Lifecycle

| Method | Signature | Purpose |
|---|---|---|
| `OpenModel` | `(data: Uint8Array, settings?: LoaderSettings) => number` | Load IFC from buffer, returns modelID |
| `OpenModels` | `(dataSets: Uint8Array[], settings?) => number[]` | Load multiple IFC files at once |
| `OpenModelFromCallback` | `(callback: ModelLoadCallback, settings?) => number` | Stream-load without full buffer in memory |
| `CreateModel` | `(model: NewIfcModel, settings?) => number` | Create empty IFC model |
| `SaveModel` | `(modelID: number) => Uint8Array` | Serialize model to IFC bytes |
| `CloseModel` | `(modelID: number) => void` | Free all WASM memory for this model |
| `IsModelOpen` | `(modelID: number) => boolean` | Check if model is open |

### LoaderSettings

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;    // translate model to origin (recommended)
  USE_FAST_BOOLS?: boolean;          // faster but less accurate boolean ops
  CIRCLE_SEGMENTS_LOW?: number;      // tessellation for small curves
  CIRCLE_SEGMENTS_MEDIUM?: number;   // tessellation for medium curves
  CIRCLE_SEGMENTS_HIGH?: number;     // tessellation for large curves
  BOOL_ABORT_THRESHOLD?: number;     // timeout (ms) for boolean operations
  MEMORY_LIMIT?: number;             // WASM memory limit in bytes
}
```

ALWAYS use `COORDINATE_TO_ORIGIN: true` for models with large world coordinates — prevents floating-point precision issues in rendering.

---

## Data Queries

### Core Query Methods

| Method | Returns | Purpose |
|---|---|---|
| `GetAllLines(modelID)` | `Vector<number>` | All expressIDs in the model |
| `GetLineIDsWithType(modelID, type, includeInherited?)` | `Vector<number>` | ExpressIDs by IFC type |
| `GetLine(modelID, expressID, flatten?, inverse?, inversePropKey?)` | `any` | Single entity by expressID |
| `GetLines(modelID, expressIDs, flatten?, inverse?, inversePropKey?)` | `any[]` | Batch entity retrieval |
| `GetRawLineData(modelID, expressID)` | `RawLineData` | Raw unparsed data (faster) |
| `GetLineType(modelID, expressID)` | `number` | IFC type code only |
| `GetMaxExpressID(modelID)` | `number` | Highest expressID |
| `GetNextExpressID(modelID, expressID)` | `number` | Next valid expressID |

### `GetLine` Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `flatten` | `boolean` | `false` | Recursively resolve all references inline |
| `inverse` | `boolean` | `false` | Include inverse relationships |
| `inversePropKey` | `string?` | `null` | Filter inverse props to specific key |

NEVER use `flatten: true` on large models without limiting scope — it recursively resolves every reference and causes severe performance degradation.

### Schema & Type Information

| Method | Returns | Purpose |
|---|---|---|
| `GetModelSchema(modelID)` | `string` | Schema version ("IFC2X3", "IFC4", "IFC4X3") |
| `GetAllTypesOfModel(modelID)` | `IfcType[]` | All IFC types present in model |
| `GetHeaderLine(modelID, headerType)` | `any` | IFC header info |

---

## Geometry Extraction

### Single-Element Geometry

```typescript
const flatMesh = ifcApi.GetFlatMesh(modelID, expressID);

for (let i = 0; i < flatMesh.geometries.size(); i++) {
  const pg = flatMesh.geometries.get(i);
  const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
  const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
  const indices = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

  // verts: Float32Array — 6 floats per vertex [x, y, z, nx, ny, nz]
  // indices: Uint32Array — triangle indices
  // pg.color: { x, y, z, w } — RGBA (w = alpha)
  // pg.flatTransformation: number[] — 4x4 column-major transform matrix
}
```

### Geometry Streaming (Memory-Efficient)

ALWAYS prefer streaming over `LoadAllGeometry` for models with more than a few hundred elements.

| Method | Purpose |
|---|---|
| `StreamAllMeshes(modelID, callback)` | Stream ALL meshable entities |
| `StreamAllMeshesWithTypes(modelID, types[], callback)` | Stream filtered by IFC types |
| `StreamMeshes(modelID, expressIDs[], callback)` | Stream specific elements |
| `LoadAllGeometry(modelID)` | Load all at once (AVOID for large models) |

Callback signature: `(mesh: FlatMesh, index: number, total: number) => void`

### Coordination & Transforms

| Method | Purpose |
|---|---|
| `GetCoordinationMatrix(modelID)` | 4x4 matrix for multi-model alignment |
| `SetGeometryTransformation(modelID, matrix)` | Apply global transform to all geometry output |
| `GetWorldTransformMatrix(modelID, placementExpressId)` | Transform for specific placement |

---

## Properties Helper

The `ifcApi.properties` object provides high-level async methods for property queries.

| Method | Returns | Purpose |
|---|---|---|
| `getItemProperties(modelID, id, recursive?, inverse?)` | `Promise<any>` | All properties for an element |
| `getPropertySets(modelID, elementID?, recursive?)` | `Promise<any[]>` | Property sets (Psets) |
| `getTypeProperties(modelID, elementID?, recursive?)` | `Promise<any[]>` | Type object properties |
| `getMaterialsProperties(modelID, elementID?, recursive?)` | `Promise<any[]>` | Material definitions |
| `getSpatialStructure(modelID, includeProperties?)` | `Promise<Node>` | Full spatial hierarchy tree |

The spatial structure returns a tree: `{ expressID, type, children: [...] }`.

---

## GUID Utilities

| Method | Purpose |
|---|---|
| `GetExpressIdFromGuid(modelID, guid)` | Convert IFC GUID string to expressID |
| `GetGuidFromExpressId(modelID, expressID)` | Convert expressID to IFC GUID string |
| `CreateIfcGuidToExpressIdMapping(modelID)` | Pre-build mapping for faster lookups |

ALWAYS call `CreateIfcGuidToExpressIdMapping` first if performing many GUID lookups — it builds an index that makes subsequent calls faster.

---

## Writing Data

| Method | Purpose |
|---|---|
| `WriteLine(modelID, lineObject)` | Write or update a single entity |
| `WriteLines(modelID, lineObjects[])` | Batch write |
| `DeleteLine(modelID, expressID)` | Remove entity from model |
| `CreateIfcEntity(modelID, type, ...args)` | Create new IFC entity |

After modifications, use `SaveModel(modelID)` to serialize back to IFC format.

---

## Logging

```typescript
import { LogLevel } from "web-ifc";
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_OFF);    // silent (recommended for production)
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_ERROR);  // errors only
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_DEBUG);  // verbose debugging
```

---

## Key Types

See `references/methods.md` for complete type definitions.

| Type | Description |
|---|---|
| `FlatMesh` | Triangulated geometry result: `{ expressID, geometries: Vector<PlacedGeometry> }` |
| `PlacedGeometry` | Single geometry piece: color (RGBA), transform (4x4), geometryExpressID |
| `IfcGeometry` | Raw WASM geometry: `GetVertexData/Size()`, `GetIndexData/Size()` |
| `Vector<T>` | WASM vector: access via `.size()` and `.get(i)` ONLY |
| `RawLineData` | Unparsed entity: `{ ID, type, arguments }` |
| `LoaderSettings` | Model loading configuration |
| `IfcType` | Type descriptor: `{ typeID, typeName }` |

---

## Common IFC Type Constants

```typescript
import {
  // Structural
  IFCWALL, IFCWALLSTANDARDCASE, IFCSLAB, IFCBEAM, IFCCOLUMN,
  // Openings
  IFCDOOR, IFCWINDOW, IFCOPENINGELEMENT,
  // Building elements
  IFCROOF, IFCSTAIR, IFCFURNISHINGELEMENT,
  // Spatial hierarchy
  IFCPROJECT, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY, IFCSPACE,
  // Properties & relations
  IFCPROPERTYSET, IFCPROPERTYSINGLEVALUE,
  IFCRELDEFINESBYPROPERTIES, IFCRELCONTAINEDINSPATIALSTRUCTURE,
  IFCRELAGGREGATES, IFCRELVOIDSELEMENT,
} from "web-ifc";
```

---

## References

- `references/methods.md` — Complete IfcAPI method signatures and type definitions
- `references/examples.md` — Full working examples: init, load, query, geometry, properties
- `references/anti-patterns.md` — Common failures: WASM init, memory leaks, Vector access

## Sources

- GitHub: https://github.com/ThatOpen/engine_web-ifc
- Docs: https://thatopen.github.io/engine_web-ifc/docs/
- npm: https://www.npmjs.com/package/web-ifc
