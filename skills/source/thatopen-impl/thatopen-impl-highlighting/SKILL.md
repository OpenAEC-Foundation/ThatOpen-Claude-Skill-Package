---
name: thatopen-impl-highlighting
description: >
  Use when implementing element selection, hover effects, outline
  visualization, or converting fragments to meshes in a ThatOpen viewer.
  Prevents missing Highlighter setup and style configuration errors.
  Covers Highlighter (click selection with styles), Hoverer (hover animation),
  Outliner (post-processing outlines), Mesher (fragment-to-mesh), FastModelPicker
  (GPU picking), multi-select, zoom-to-selection, autoToggle.
  Keywords: highlighter, selection, hover, outline, pick, highlight, multi-select, color, style, mesher, fast model picker, click to select, hover effect, selected element color.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components-front 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Highlighting & Selection

## Overview

This skill covers element highlighting, selection, hover effects, outline
rendering, and fragment-to-mesh conversion in a ThatOpen BIM viewer. The
five components covered are:

| Component | Package | Purpose |
|---|---|---|
| Highlighter | `@thatopen/components-front` | Click-based element selection with color styles |
| Hoverer | `@thatopen/components-front` | Fade animation on hover |
| Outliner | `@thatopen/components-front` | Post-processing outline effects |
| Mesher | `@thatopen/components-front` | Convert fragment selections to THREE.Mesh |
| FastModelPicker | `@thatopen/components` | GPU-based element picking |

**Version**: @thatopen/components-front 3.3.x
**Prerequisites**: thatopen-impl-viewer (world setup, PostproductionRenderer)

## MaterialDefinition Type

The core type for highlight styles:

```typescript
type MaterialDefinition = {
  color: THREE.Color;
  opacity: number;
};
```

All Highlighter styles are either a `MaterialDefinition` object or `null`
(meaning transparent/no-color selection tracking only).

## Highlighter

### Setup

ALWAYS call `setup()` before using any Highlighter functionality. Without
`setup()`, the Highlighter has no world reference and all operations fail
silently or throw.

```typescript
import * as OBCF from "@thatopen/components-front";

const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });
```

### HighlighterConfig

| Property | Type | Default | Description |
|---|---|---|---|
| `world` | `World` | required | World to pick elements from |
| `selectName` | `string` | `"select"` | Default selection style name |
| `selectionColor` | `THREE.Color` | `#BCF124` | Default selection color |
| `autoHighlightOnClick` | `boolean` | `true` | Auto-highlight clicked elements |
| `selectEnabled` | `boolean` | `true` | Enable/disable selection globally |
| `autoUpdateFragments` | `boolean` | `true` | Auto-update fragment visuals |
| `selectMaterialDefinition` | `MaterialDefinition` | yellow | Default style material |

```typescript
highlighter.setup({
  world,
  selectName: "select",
  selectionColor: new THREE.Color("#BCF124"),
  autoHighlightOnClick: true,
});
```

### Styles

Styles define how highlighted elements appear. Each style has a name and a
`MaterialDefinition` (or `null` for invisible tracking).

```typescript
// Add a custom highlight style
highlighter.styles.set("warning", {
  color: new THREE.Color("#FF0000"),
  opacity: 0.5,
});

// Null style — tracks selection without visual change
highlighter.styles.set("invisible", null);
```

- ALWAYS define styles AFTER calling `setup()`.
- The default `"select"` style is created automatically by `setup()`.
- Style names are arbitrary strings. Use descriptive names.

### Highlight (Click Selection)

```typescript
// Highlight whatever the mouse is pointing at with the "select" style
await highlighter.highlight("select");

// Highlight without removing previous selection
await highlighter.highlight("select", false);

// Highlight and zoom to selection
await highlighter.highlight("select", true, true);

// Highlight with exclusion list
const exclude: ModelIdMap = new Map();
await highlighter.highlight("select", true, false, exclude);
```

Method signature:
```typescript
highlight(
  name: string,
  removePrevious?: boolean,   // default: true
  zoomToSelection?: boolean,  // default: false
  exclude?: ModelIdMap         // elements to skip
): Promise<void>;
```

### HighlightByID (Programmatic Selection)

Select specific elements by their model/element IDs without requiring a
mouse event:

```typescript
const items: ModelIdMap = new Map();
items.set(modelId, new Set([elementId1, elementId2]));

await highlighter.highlightByID("select", items);

// Keep previous selection
await highlighter.highlightByID("select", items, false);

// Highlight and zoom
await highlighter.highlightByID("select", items, true, true);
```

Method signature:
```typescript
highlightByID(
  name: string,
  modelIdMap: ModelIdMap,
  removePrevious?: boolean,   // default: true
  zoomToSelection?: boolean,  // default: false
  exclude?: ModelIdMap
): Promise<void>;
```

### Reading the Selection

The `selection` property holds the current state for every style:

```typescript
// Get current "select" selection
const selected: ModelIdMap = highlighter.selection["select"];

// Iterate selected elements
for (const [modelId, elementIds] of selected) {
  console.log(`Model ${modelId}: ${elementIds.size} elements selected`);
}
```

### Clear Selection

```typescript
// Clear a specific style
await highlighter.clear("select");

// Clear all styles
await highlighter.clear();

// Clear with a filter (keep specific elements)
await highlighter.clear("select", filterModelIdMap);
```

### Multi-Select

The `multiple` property controls multi-selection behavior:

```typescript
// No multi-select (default) — each click replaces previous selection
highlighter.multiple = "none";

// Multi-select with Shift key
highlighter.multiple = "shiftKey";

// Multi-select with Ctrl key
highlighter.multiple = "ctrlKey";
```

- When `multiple` is `"shiftKey"` or `"ctrlKey"`, holding the specified
  modifier key while clicking adds to the selection instead of replacing it.
- ALWAYS set `multiple` AFTER calling `setup()`.

### Zoom to Selection

```typescript
// Enable zoom-to-selection globally
highlighter.zoomToSelection = true;

// Or per-call
await highlighter.highlight("select", true, true);
```

When `zoomToSelection` is `true`, the camera frames the highlighted elements
after each highlight operation.

### AutoToggle

Styles listed in `autoToggle` automatically deselect when the same element
is clicked again (toggle behavior):

```typescript
highlighter.autoToggle.add("select");

// First click on element A → selects A
// Second click on element A → deselects A
```

- ALWAYS add style names to `autoToggle` AFTER calling `setup()`.
- AutoToggle works per-element, not per-click. Clicking a different element
  still replaces the selection (unless `multiple` is also set).

### Selectable (Per-Style Filtering)

Restrict which elements can be highlighted for a given style:

```typescript
const allowedElements: ModelIdMap = new Map();
allowedElements.set(modelId, new Set([101, 102, 103]));

highlighter.selectable["select"] = allowedElements;
```

- When `selectable` is set for a style, ONLY elements in that ModelIdMap
  can be highlighted with that style. All other elements are ignored.
- If `selectable` is not set for a style, all elements are selectable.

### Update Colors

After changing a style's MaterialDefinition, call `updateColors()` to apply
the visual change immediately:

```typescript
highlighter.styles.set("select", {
  color: new THREE.Color("#0000FF"),
  opacity: 0.8,
});
await highlighter.updateColors();
```

## Hoverer

Hoverer provides a fade animation effect when the mouse hovers over
elements. It is a separate component from Highlighter.

```typescript
const hoverer = components.get(OBCF.Hoverer);
```

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `duration` | `number` | — | Fade animation duration (ms) |
| `delay` | `number` | — | Delay before hover triggers (ms) |
| `animation` | `string` | — | Animation type |

### Events

```typescript
hoverer.onHoverStarted.add((data) => {
  console.log("Hover started:", data);
});

hoverer.onHoverEnded.add(() => {
  console.log("Hover ended");
});
```

- Hoverer works independently of Highlighter. You can use both
  simultaneously: Hoverer for visual feedback, Highlighter for selection.
- NEVER rely on Hoverer for selection state. Use Highlighter for that.

## Outliner

Outliner renders post-processing outlines around highlighted elements. It
requires `PostproductionRenderer` (not `SimpleRenderer`).

### Setup

```typescript
import * as OBCF from "@thatopen/components-front";

// MUST use PostproductionRenderer — Outliner has no effect with SimpleRenderer
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

// ... world setup ...
world.renderer.postproduction.enabled = true;

const outliner = components.get(OBCF.Outliner);
```

### Styles

Outliner styles reference Highlighter style names. When elements are
highlighted with a style that the Outliner tracks, outlines appear.

```typescript
// Outliner tracks Highlighter style names
outliner.styles.add("select");
```

### Outline Appearance

| Property | Type | Default | Description |
|---|---|---|---|
| `color` | `THREE.Color` | — | Outline stroke color |
| `thickness` | `number` | — | Outline stroke width (px) |
| `fillColor` | `THREE.Color` | — | Interior fill color |
| `fillOpacity` | `number` | — | Interior fill opacity (0-1) |

```typescript
outliner.color = new THREE.Color("#FF6600");
outliner.thickness = 2;
outliner.fillColor = new THREE.Color("#FF6600");
outliner.fillOpacity = 0.1;
```

### Manual Item Management

```typescript
const items: ModelIdMap = new Map();
items.set(modelId, new Set([elementId]));

outliner.addItems(items);
outliner.removeItems(items);
outliner.clean(); // Remove all outlined items
```

## Mesher

Mesher converts fragment selections into standard `THREE.Mesh` objects.
This is useful for custom rendering, export, or manipulation outside the
fragment system.

```typescript
const mesher = components.get(OBCF.Mesher);

const items: ModelIdMap = new Map();
items.set(modelId, new Set([elementId1, elementId2]));

const meshes: THREE.Mesh[] = mesher.get(items);
```

### Method Signature

```typescript
get(modelIdMap: ModelIdMap, config?: MesherConfig): THREE.Mesh[];
```

- Returns standard `THREE.Mesh` objects with geometry and material.
- The returned meshes are NOT managed by the fragment system. You MUST
  manually add them to a scene and dispose them when done.
- Useful for: custom shaders, physics engines, export to glTF, snapshots.

## FastModelPicker

FastModelPicker provides GPU-based element picking as an alternative to
raycasting. It renders element IDs to an off-screen buffer and reads back
the pixel under the cursor. This is significantly faster than raycasting
for large models.

```typescript
const picker = components.get(OBC.FastModelPicker);
```

- FastModelPicker is in `@thatopen/components` (not components-front).
- It is new in v3 and replaces manual raycasting for performance-critical
  selection scenarios.
- For most use cases, the built-in raycasting via Highlighter is sufficient.
  Use FastModelPicker only when raycasting becomes a bottleneck with very
  large models (100k+ elements).

## Complete Click-to-Select Workflow

```typescript
import * as THREE from "three";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

// 1. Viewer setup (see thatopen-impl-viewer)
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
world.renderer.postproduction.enabled = true;

// 2. Setup Highlighter — MUST call setup() before any other operation
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// 3. Configure multi-select
highlighter.multiple = "shiftKey";

// 4. Enable zoom-to-selection
highlighter.zoomToSelection = true;

// 5. Add custom styles
highlighter.styles.set("warning", {
  color: new THREE.Color("#FF0000"),
  opacity: 0.5,
});

// 6. Setup Outliner for post-processing outlines
const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");
outliner.color = new THREE.Color("#BCF124");
outliner.thickness = 2;

// 7. Setup Hoverer for hover feedback
const hoverer = components.get(OBCF.Hoverer);

// 8. Start render loop
components.init();

// 9. Load a model (see thatopen-syntax-ifc-loading)
// ... model loading code ...

// 10. Programmatic selection
const items: ModelIdMap = new Map();
items.set(modelId, new Set([elementId]));
await highlighter.highlightByID("select", items);

// 11. Clear selection
await highlighter.clear("select");
```

## Critical Rules

1. **ALWAYS** call `highlighter.setup({ world })` before using any
   Highlighter method. Without it, highlight operations fail.
2. **ALWAYS** use `PostproductionRenderer` when using Outliner. Outlines
   are a post-processing effect and have no effect with SimpleRenderer.
3. **ALWAYS** enable postproduction (`world.renderer.postproduction.enabled
   = true`) before Outliner will render.
4. **ALWAYS** dispose meshes returned by `Mesher.get()` manually. They are
   not tracked by the fragment system.
5. **NEVER** set `multiple`, `autoToggle`, or `selectable` before calling
   `setup()`. The Highlighter is not initialized until `setup()` completes.
6. **NEVER** assume Hoverer and Highlighter share state. They are
   independent components with separate tracking.
7. **NEVER** use FastModelPicker without measuring whether raycasting is
   actually a bottleneck. The built-in raycasting is sufficient for most
   models.

## Reference Files

- [references/methods.md](references/methods.md) — Highlighter, Hoverer,
  Outliner, Mesher, FastModelPicker API signatures
- [references/examples.md](references/examples.md) — Click select, hover,
  outline, multi-select, custom styles
- [references/anti-patterns.md](references/anti-patterns.md) — Missing
  setup, style conflicts, performance issues

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch
- npm: `@thatopen/components-front@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md` (Sections 4.2-4.5)
- Skill: `thatopen-impl-viewer` (PostproductionRenderer context)
