---
name: thatopen-errors-loading
description: >
  Use when IFC loading fails, WASM initialization errors occur, or
  fragment worker initialization problems arise.
  Prevents common loading failures by documenting exact error patterns.
  Covers WASM init failures, IFC parse errors, WASM path misconfiguration,
  web-ifc version mismatches, worker initialization failures, missing IFC
  classes, silent failures, COORDINATE_TO_ORIGIN issues.
  Keywords: error, loading, wasm, ifc, parse, failed, worker, init, setup, path, version, mismatch, crash, fix, model won't load, IFC loading failed, wasm not found, blank screen.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x / web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Loading Errors: Diagnosis & Recovery

## Overview

This skill covers every failure mode that occurs when loading IFC files or initializing the ThatOpen fragment pipeline. Each error pattern maps to a specific root cause and a concrete fix.

Use this skill when:
- An IFC file fails to load or parse
- WASM initialization throws errors
- The viewer shows nothing after loading
- Fragment worker errors appear in the console
- Elements are missing from the loaded model

---

## Critical Warnings

1. **ALWAYS** call `await ifcLoader.setup()` before `ifcLoader.load()` — skipping setup causes silent null reference failures.
2. **ALWAYS** call `fragmentsManager.init(workerURL)` before any fragment operation — the worker is required for all data operations.
3. **ALWAYS** match the WASM file version to the installed `web-ifc` npm package version — mismatches cause `RuntimeError: unreachable`.
4. **NEVER** pass a `string` or `File` object to `ifcLoader.load()` — it requires `Uint8Array`.
5. **ALWAYS** include a trailing slash in WASM paths — `"/wasm/"` not `"/wasm"`.
6. **ALWAYS** serve `.wasm` files with MIME type `application/wasm` — incorrect MIME types cause `CompileError`.
7. **ALWAYS** configure COOP/COEP headers for multi-threaded WASM — `SharedArrayBuffer` requires cross-origin isolation.
8. **NEVER** assume all IFC entity types load by default — check `excludedCategories` if elements are missing.

---

## Error Message to Fix Mapping

| Error Message / Symptom | Root Cause | Fix | Section |
|---|---|---|---|
| `RuntimeError: unreachable` | WASM version mismatch | Match WASM files to npm version | E-01 |
| `CompileError: WebAssembly.instantiate()` | Corrupt/wrong WASM file or MIME type | Serve `.wasm` as `application/wasm` | E-02 |
| `404 Not Found` on `web-ifc.wasm` | Wrong WASM path | Fix path, add trailing slash | E-03 |
| `Cannot read properties of undefined` on load | `setup()` not called | ALWAYS call `await ifcLoader.setup()` first | E-04 |
| `FragmentsManager not initialized` | Worker not initialized | Call `fragmentsManager.init(workerURL)` | E-05 |
| Model loads but nothing renders | Missing `components.init()` or scene add | Call `components.init()`, add `model.object` to scene | E-06 |
| Elements missing from model | Default `excludedCategories` | Add missing classes via `onIfcImporterInitialized` | E-07 |
| `TypeError: data is not Uint8Array` | Wrong data type passed to `load()` | Convert to `new Uint8Array(buffer)` | E-08 |
| Geometry at wrong position / z-fighting | Large world coordinates | Use `COORDINATE_TO_ORIGIN: true` | E-09 |
| `SharedArrayBuffer is not defined` | Missing COOP/COEP headers | Configure server headers for cross-origin isolation | E-10 |
| Worker script 404 / CORS error | Wrong worker URL or CORS policy | Fix worker URL path, configure CORS headers | E-11 |
| Boolean operation hangs / infinite loop | Complex geometry timeout | Set `BOOL_ABORT_THRESHOLD` | E-12 |
| `Invalid IFC file` or empty model | Corrupt file or wrong encoding | Validate IFC header, ensure binary fetch | E-13 |
| Tab crashes on large model | WASM memory exhaustion | Set `MEMORY_LIMIT`, use streaming | E-14 |

---

## E-01: WASM Version Mismatch

**Error:** `RuntimeError: unreachable` or `CompileError` during WASM instantiation.

**Cause:** The `.wasm` binary file version does not match the installed `web-ifc` npm package version. The WASM binary and JavaScript API are tightly coupled — a mismatch causes undefined behavior.

**Diagnostic:**
1. Check installed version: `npm ls web-ifc`
2. Check WASM path in code — does the version in the CDN URL match?
3. If using local files, were they copied from the correct `node_modules/web-ifc/` version?

**Fix:**
```typescript
// Option A: Use autoSetWasm (recommended)
await ifcLoader.setup(); // autoSetWasm: true resolves correct version automatically

// Option B: Match CDN version to installed package
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/", // MUST match npm ls web-ifc
    absolute: true,
  },
});
```

**Rule:** NEVER hardcode a web-ifc version without verifying it matches the installed npm package. ALWAYS prefer `autoSetWasm: true` unless you have a specific reason for manual configuration.

---

## E-02: WASM MIME Type Error

**Error:** `CompileError: WebAssembly.instantiate(): expected magic word` or similar compilation failure.

**Cause:** The web server serves `.wasm` files with the wrong MIME type (e.g., `application/octet-stream` or `text/html` for a 404 page). Browsers require `application/wasm`.

**Diagnostic:**
1. Open DevTools Network tab
2. Find the `web-ifc.wasm` request
3. Check the `Content-Type` response header

**Fix (server configuration):**

| Server | Configuration |
|---|---|
| Nginx | `types { application/wasm wasm; }` |
| Apache | `AddType application/wasm .wasm` |
| Express | `express.static(dir, { setHeaders: (res, path) => { if (path.endsWith('.wasm')) res.type('application/wasm'); } })` |
| Vite | Handled automatically |
| Webpack | Use `file-loader` or `copy-webpack-plugin` with correct MIME |

**Rule:** ALWAYS verify the server serves `.wasm` files with `Content-Type: application/wasm`.

---

## E-03: WASM Path Misconfiguration

**Error:** `404 Not Found` on `web-ifc.wasm` or `web-ifc-mt.wasm`.

**Cause:** The WASM path does not resolve to the directory containing the WASM files. Common mistakes: missing trailing slash, wrong relative path, files not copied to public directory.

**Diagnostic checklist:**
1. Does the path end with `/`?
2. Does the directory actually contain `web-ifc.wasm`?
3. Is `absolute: true` set when using a full URL?
4. For local paths: are the files in the build output / public directory?

**Fix:**
```typescript
// WRONG: missing trailing slash
wasm: { path: "https://unpkg.com/web-ifc@0.0.77", absolute: true }

// WRONG: relative path with absolute: true
wasm: { path: "/static/wasm/", absolute: true }

// CORRECT: CDN with trailing slash
wasm: { path: "https://unpkg.com/web-ifc@0.0.77/", absolute: true }

// CORRECT: local path (relative to document base)
wasm: { path: "/static/wasm/", absolute: false }
```

**Rule:** ALWAYS include a trailing slash in WASM paths. The runtime appends `web-ifc.wasm` directly to this string.

---

## E-04: setup() Not Called Before load()

**Error:** `Cannot read properties of undefined`, silent failure, or WASM not initialized.

**Cause:** `IfcLoader` implements the `Configurable` interface. WASM initialization happens inside `setup()`, not in the constructor or `components.get()`.

**Fix:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();  // ALWAYS await before load()
const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS call `await ifcLoader.setup()` before calling `load()`. This is the single most common loading failure.

---

## E-05: FragmentsManager Worker Not Initialized

**Error:** `FragmentsManager not initialized`, model fails to process, or worker-related errors in console.

**Cause:** The fragment system requires a Web Worker for processing IFC-to-Fragments conversion. Without `init()`, the worker is not available.

**Fix:**
```typescript
const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init(workerURL); // ALWAYS call before any IFC loading

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
```

**Diagnostic:** If `workerURL` is wrong, you will see a 404 or CORS error. See E-11 for worker URL issues.

**Rule:** ALWAYS initialize FragmentsManager with a valid worker URL before any loading operations.

---

## E-06: Silent Render Failure

**Error:** No error messages, but the viewer shows nothing (black screen or empty viewport).

**Cause (check in order):**
1. `components.init()` not called — the render loop never starts
2. `model.object` not added to the scene
3. Camera not pointing at the model
4. Container element has zero dimensions (0px height)

**Fix:**
```typescript
// 1. Start render loop
components.init();

// 2. Add model to scene
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);

// 3. Frame the camera on the model
world.camera.fit(world.scene.three.children);

// 4. Ensure container has dimensions
// CSS: #viewer { width: 100%; height: 100vh; }
```

**Diagnostic checklist:**
- [ ] Is `components.init()` called?
- [ ] Is `model.object` added to `world.scene.three`?
- [ ] Does the container element have non-zero `offsetWidth` and `offsetHeight`?
- [ ] Is the camera positioned to see the model?

**Rule:** ALWAYS call `components.init()` and ALWAYS add `model.object` to the scene after loading.

---

## E-07: Missing IFC Elements

**Error:** Model loads successfully but certain element types (spaces, openings, furnishing) are absent.

**Cause:** The `IfcImporter` has a default `classes.elements` set that excludes some IFC categories for performance. Common exclusions: `IFCSPACE`, `IFCOPENINGELEMENT`, `IFCFLOWSEGMENT`, `IFCFLOWFITTING`.

**Fix:**
```typescript
ifcLoader.onIfcImporterInitialized.add((importer) => {
  // Add missing categories
  importer.classes.elements.add(WEBIFC.IFCSPACE);
  importer.classes.elements.add(WEBIFC.IFCOPENINGELEMENT);
  importer.classes.elements.add(WEBIFC.IFCFLOWSEGMENT);
});
```

**Diagnostic:** Before loading, log available types after loading with `ifcApi.GetAllTypesOfModel(modelID)` to verify which IFC types exist in the file.

**Rule:** ALWAYS check the default element class set when elements are missing. NEVER assume all IFC entity types are imported by default.

---

## E-08: Wrong Data Type for load()

**Error:** `TypeError`, `Invalid IFC file`, or empty model.

**Cause:** `load()` requires `Uint8Array`. Passing `ArrayBuffer`, `string`, `Blob`, or `File` directly causes failures.

**Fix:**
```typescript
// From fetch
const buffer = await response.arrayBuffer();
const data = new Uint8Array(buffer);

// From File input
const data = new Uint8Array(await file.arrayBuffer());

// From base64
const binary = atob(base64String);
const data = new Uint8Array(binary.length);
for (let i = 0; i < binary.length; i++) data[i] = binary.charCodeAt(i);
```

**Rule:** ALWAYS convert to `Uint8Array` before passing to `load()`.

---

## E-09: Large Coordinate / Floating-Point Issues

**Error:** Model appears but geometry is distorted, flickering (z-fighting), or positioned far from origin causing camera issues.

**Cause:** IFC models often use real-world coordinates (e.g., UTM: x=500000, y=6000000). WebGL uses 32-bit floats — at these magnitudes, precision is ~1 meter, causing visible artifacts.

**Fix:**
```typescript
await ifcLoader.setup();
ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
const model = await ifcLoader.load(data, true, "Building");
```

**For multi-model coordination**, use the coordination matrix instead:
```typescript
// Load at origin, then apply coordination manually
const model = await ifcLoader.load(data, true, "Building");
// FragmentsManager handles coordination when coordinate=true
```

**Rule:** ALWAYS use `COORDINATE_TO_ORIGIN: true` for models with large world coordinates. For multi-model federation, use the `coordinate` parameter in `load()`.

---

## E-10: SharedArrayBuffer Not Available

**Error:** `SharedArrayBuffer is not defined`, multi-threaded WASM fails, or falls back to single-threaded mode silently.

**Cause:** `SharedArrayBuffer` requires cross-origin isolation via HTTP headers. Without these headers, browsers disable `SharedArrayBuffer` for security.

**Required HTTP headers:**
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**Fix per environment:**

| Environment | Configuration |
|---|---|
| Vite | `server: { headers: { 'Cross-Origin-Opener-Policy': 'same-origin', 'Cross-Origin-Embedder-Policy': 'require-corp' } }` |
| Webpack devServer | Same headers in `devServer.headers` |
| Nginx | `add_header Cross-Origin-Opener-Policy same-origin; add_header Cross-Origin-Embedder-Policy require-corp;` |
| Vercel | `vercel.json` headers configuration |

**Warning:** Enabling these headers means all cross-origin resources (images, scripts, iframes) MUST have appropriate CORS headers or `crossorigin` attributes. This can break third-party integrations.

**Rule:** ALWAYS configure COOP/COEP headers for production deployments that need multi-threaded WASM. If you cannot set these headers, web-ifc falls back to single-threaded mode — functional but slower.

---

## E-11: Worker Initialization Failures

**Error:** 404 on worker script, `DOMException: Failed to construct 'Worker'`, or CORS errors loading worker.

**Cause:** The worker URL passed to `fragmentsManager.init()` does not resolve or is blocked by CORS policy.

**Diagnostic:**
1. Open DevTools Network tab — is the worker script request returning 200?
2. Is the worker URL correct relative to the document origin?
3. For cross-origin workers, is the server setting `Access-Control-Allow-Origin`?

**Fix:**
```typescript
// Local worker file (most common)
fragmentsManager.init("/workers/fragment-worker.mjs");

// CDN worker (requires CORS)
fragmentsManager.init("https://cdn.example.com/workers/fragment-worker.mjs");
```

**Rule:** ALWAYS verify the worker URL resolves to a valid JavaScript module. NEVER use a worker URL from a different origin without CORS headers.

---

## E-12: Boolean Operation Timeout

**Error:** Loading hangs indefinitely on certain IFC files, browser tab becomes unresponsive.

**Cause:** Complex CSG (Constructive Solid Geometry) boolean operations in the IFC file cause web-ifc to enter long-running computations. This is a known limitation with certain modeling software exports.

**Fix:**
```typescript
await ifcLoader.setup();
ifcLoader.settings.webIfc.BOOL_ABORT_THRESHOLD = 5000; // Abort after 5 seconds
ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;       // Faster but less accurate
```

**Rule:** ALWAYS set `BOOL_ABORT_THRESHOLD` when loading untrusted or unknown IFC files. A threshold of 5000-10000ms prevents infinite hangs while allowing most boolean operations to complete.

---

## E-13: Corrupt or Invalid IFC File

**Error:** `Invalid IFC file`, empty model, or parser crash.

**Diagnostic checklist:**
1. Open the file in a text editor — does it start with `ISO-10303-21;`?
2. Was the file fetched correctly? Check response status and content length.
3. Was `response.text()` used instead of `response.arrayBuffer()`? Text decoding corrupts binary IFC data in some encodings.
4. Is the file an IFC-XML (`.ifcXML`) file? web-ifc only supports STEP-encoded IFC.

**Fix:**
```typescript
// ALWAYS use arrayBuffer, never text()
const response = await fetch(url);
if (!response.ok) throw new Error(`Fetch failed: ${response.status}`);
const buffer = await response.arrayBuffer();
if (buffer.byteLength === 0) throw new Error("Empty IFC file");
const data = new Uint8Array(buffer);
```

**Rule:** ALWAYS validate the response before loading. NEVER use `response.text()` for IFC files — ALWAYS use `response.arrayBuffer()`.

---

## E-14: WASM Memory Exhaustion

**Error:** Browser tab crashes, `Out of memory`, or `RuntimeError: memory access out of bounds`.

**Cause:** Very large IFC models (500MB+) or multiple models loaded simultaneously exhaust the WASM linear memory.

**Fix:**
```typescript
// Set memory limit
ifcLoader.settings.webIfc.MEMORY_LIMIT = 2147483648; // 2GB

// Use streaming for large models instead of full load
// Filter unnecessary IFC classes
ifcLoader.onIfcImporterInitialized.add((importer) => {
  importer.classes.elements.delete(WEBIFC.IFCFURNISHINGELEMENT);
  importer.classes.elements.delete(WEBIFC.IFCSPACE);
});

// Dispose models when no longer needed
fragmentsManager.dispose();
```

**Rule:** ALWAYS filter IFC classes to reduce memory when loading large models. ALWAYS dispose models that are no longer needed.

---

## Diagnostic Checklist: Loading Failures

When an IFC file fails to load, work through this checklist in order:

1. **Console errors?** Check the browser console for specific error messages. Match to the table above.
2. **WASM initialized?** Is `setup()` called and awaited before `load()`?
3. **Worker initialized?** Is `fragmentsManager.init(workerURL)` called before loading?
4. **WASM path correct?** Does the path end with `/`? Do the files exist at that path?
5. **Version match?** Does the WASM file version match `npm ls web-ifc`?
6. **Data type correct?** Is the IFC data a `Uint8Array`?
7. **File valid?** Does it start with `ISO-10303-21;`? Is the response status 200?
8. **Render loop active?** Is `components.init()` called?
9. **Model in scene?** Is `model.object` added to `world.scene.three`?
10. **Camera framed?** Can the camera see the model bounds?
11. **Container sized?** Does the container have non-zero dimensions?
12. **CORS/headers?** Are COOP/COEP set for multi-threaded? Is the worker URL accessible?

---

## Error Recovery Patterns

### Graceful Fallback
```typescript
try {
  const model = await ifcLoader.load(data, true, name);
  world.scene.three.add(model.object);
} catch (error) {
  if (error.message.includes("unreachable")) {
    console.error("WASM version mismatch — check web-ifc version");
  } else if (error.message.includes("magic word")) {
    console.error("WASM MIME type error — configure server for application/wasm");
  } else {
    console.error("IFC loading failed:", error.message);
  }
}
```

### Retry with Relaxed Settings
```typescript
async function loadWithFallback(data: Uint8Array, name: string) {
  try {
    return await ifcLoader.load(data, true, name);
  } catch (firstError) {
    // Retry with relaxed boolean settings
    ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;
    ifcLoader.settings.webIfc.BOOL_ABORT_THRESHOLD = 3000;
    ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
    return await ifcLoader.load(data, true, name);
  }
}
```

---

## Relationship to Other Skills

| Skill | Relationship |
|---|---|
| `thatopen-syntax-ifc-loading` | Normal loading workflow. Use this errors skill when loading fails. |
| `thatopen-core-web-ifc` | Low-level WASM engine. Error patterns E-01 through E-03 trace to web-ifc initialization. |
| `thatopen-core-fragments` | Fragment system. Error E-05 traces to FragmentsManager initialization. |
| `thatopen-errors-performance` | Performance-related issues (memory, speed). Complementary to E-12 and E-14. |

---

## References

- `references/methods.md` — Error types, messages, and fix patterns
- `references/examples.md` — Working fix examples: WASM, path, version, worker
- `references/anti-patterns.md` — Common mistakes that cause loading failures

## Sources

- API docs: https://docs.thatopen.com/api/@thatopen/components/classes/IfcLoader
- web-ifc: https://github.com/ThatOpen/engine_web-ifc
- GitHub issues: https://github.com/ThatOpen/engine_components/issues
