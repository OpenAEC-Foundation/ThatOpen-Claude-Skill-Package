---
name: thatopen-syntax-ui
description: >
  Use when building BIM user interfaces with ThatOpen UI components,
  creating panels, toolbars, or property displays.
  Prevents missing BUI.Manager.init() and incorrect component usage.
  Covers @thatopen/ui web components (bim-panel, bim-toolbar, bim-table,
  bim-grid, bim-button, inputs), CSS theming, @thatopen/ui-obc functional
  components, Lit web component patterns.
  Keywords: ui, panel, toolbar, table, grid, button, bim-panel, lit, web components, theming, css custom properties, ui-obc, create sidebar, add toolbar, custom BIM interface.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/ui 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen UI Component Syntax

## Purpose

This skill covers the **syntax and usage patterns** for ThatOpen's UI component
library (`@thatopen/ui`) and its BIM-connected counterpart (`@thatopen/ui-obc`).
It provides component catalogs, property references, layout patterns, CSS theming,
and event handling. For underlying Lit web component patterns, see `lit-bim-ui`.

**Version**: @thatopen/ui 3.3.x / @thatopen/ui-obc 3.3.x

## Mandatory Initialization

ALWAYS call `BUI.Manager.init()` before using ANY `bim-*` element in your
application. This registers all custom elements and injects global styles.

```typescript
import * as BUI from "@thatopen/ui";

// ALWAYS call this once at application startup
BUI.Manager.init();
```

Without this call, `bim-*` tags render as unknown HTML elements with no
styling or behavior. The browser silently ignores unregistered custom elements.

### Manager API

```typescript
class Manager {
  // Register all bim-* custom elements and inject global styles
  static init(
    querySelectorElements?: string,  // optional CSS selector for animation targets
    animateOnLoad?: boolean          // default: true — entrance animations
  ): void;

  // Switch between dark and light theme
  static toggleTheme(animate?: boolean): void;

  // Preload icon collections (Iconify)
  static preloadIcons(collections: string[], log?: boolean): Promise<void>;

  // Configuration
  static config: ManagerConfig;
}

interface ManagerConfig {
  sectionLabelOnVerticalToolbar: boolean;  // default: false
  internalComponentNameAttribute: string;
}
```

## Package Architecture

| Package | Role | Dependencies |
|---|---|---|
| `@thatopen/ui` | Presentational web components | Standalone (Lit-based) |
| `@thatopen/ui-obc` | Functional BIM components | `@thatopen/ui` + `@thatopen/components` |

`@thatopen/ui` provides **generic UI elements** (panels, toolbars, tables, inputs).
These have NO dependency on the ThatOpen engine — they work as standalone web
components in any HTML page.

`@thatopen/ui-obc` provides **pre-wired BIM components** that connect UI elements
to `@thatopen/components` engine functionality (model trees, property tables,
IFC load buttons). These ALWAYS require a `Components` instance.

## Core Component Catalog

All components use the `bim-` prefix and are registered by `BUI.Manager.init()`.

### Layout Components

| Element | Description | Key Properties |
|---|---|---|
| `bim-grid` | CSS Grid layout container | `layout`, `layouts`, `floating`, `resizeableAreas` |
| `bim-panel` | Collapsible side panel | `name`, `label`, `icon`, `hidden`, `headerHidden` |
| `bim-panel-section` | Section within a panel | `label`, `icon`, `collapsed`, `fixed` |
| `bim-toolbar` | Horizontal/vertical toolbar | `vertical`, `labelsHidden`, `hidden` |
| `bim-toolbar-section` | Group within a toolbar | `label`, `icon`, `vertical` |
| `bim-toolbar-group` | Nested button group | `vertical` |
| `bim-viewport` | 3D rendering container | Wraps canvas for renderer |
| `bim-tabs` | Tab container | Active tab management |
| `bim-tab` | Individual tab | `label`, `icon` |

### Interactive Components

| Element | Description | Key Properties |
|---|---|---|
| `bim-button` | Action button | `label`, `icon`, `disabled`, `active`, `vertical` |
| `bim-checkbox` | Boolean toggle | `label`, `checked`, `inverted` |
| `bim-text-input` | Text entry | `label`, `value`, `placeholder` |
| `bim-number-input` | Numeric entry | `label`, `value`, `min`, `max`, `step`, `suffix` |
| `bim-color-input` | Color picker | `label`, `color`, `opacity` |
| `bim-dropdown` | Select menu | `label`, `multiple`, `required` |
| `bim-option` | Dropdown option | `label`, `value`, `checked`, `icon` |
| `bim-selector` | Radio-style selector | `label`, `multiple` |

### Display Components

| Element | Description | Key Properties |
|---|---|---|
| `bim-label` | Text with optional icon | `icon`, `img` |
| `bim-icon` | Iconify icon display | `icon` |
| `bim-table` | Data table with hierarchy | `data`, `columns`, `expanded`, `selectableRows` |
| `bim-chart` | Chart visualization | Chart data binding |
| `bim-tooltip` | Hover tooltip | Text content |
| `bim-context-menu` | Right-click menu | Menu items |

## Grid Layout System

The `bim-grid` component provides CSS Grid-based layouts with named areas.
This is the PRIMARY way to compose a full BIM application layout.

```typescript
const grid = document.createElement("bim-grid");

// Define named layouts
grid.layouts = {
  main: {
    template: `
      "toolbar toolbar" auto
      "panel viewport" 1fr
      / 320px 1fr
    `,
    elements: {
      toolbar: toolbarElement,
      panel: panelElement,
      viewport: viewportElement,
    },
  },
  fullscreen: {
    template: `"viewport" 1fr / 1fr`,
    elements: { viewport: viewportElement },
  },
};

// Activate a layout
grid.layout = "main";
```

### Layout Template Syntax

The `template` string maps directly to CSS `grid-template`. Format:
```
"area1 area2" rowSize
"area3 area4" rowSize
/ col1Size col2Size
```

### Resizable Areas

Enable user-resizable grid tracks:
```typescript
grid.resizeableAreas = true;
// Optionally exclude areas from resizing
grid.areasResizeExceptions = ["toolbar"];
```

## bim-panel Pattern

Panels contain sections. Sections contain form elements or custom content.

```html
<bim-panel name="Properties" icon="settings">
  <bim-panel-section label="General" icon="info" fixed>
    <bim-text-input label="Name" value="Wall-001"></bim-text-input>
    <bim-number-input label="Height" value="3.0" suffix="m"></bim-number-input>
    <bim-checkbox label="Structural" checked></bim-checkbox>
  </bim-panel-section>

  <bim-panel-section label="Materials" icon="palette" collapsed>
    <bim-color-input label="Color" color="#ff6600"></bim-color-input>
    <bim-dropdown label="Material">
      <bim-option label="Concrete" checked></bim-option>
      <bim-option label="Steel"></bim-option>
      <bim-option label="Wood"></bim-option>
    </bim-dropdown>
  </bim-panel-section>
</bim-panel>
```

### Panel Value Collection

The panel aggregates values from its child form elements:

```typescript
const panel = document.querySelector("bim-panel");
panel.addEventListener("change", () => {
  const values = panel.value;
  // { "Name": "Wall-001", "Height": 3.0, "Structural": true, ... }
});
```

### Panel Activation Button

Every panel creates an `activationButton` that toggles its visibility:

```typescript
const panel = document.querySelector("bim-panel");
const toggleBtn = panel.activationButton;
// Place this button in a toolbar to show/hide the panel
toolbar.appendChild(toggleBtn);
```

## bim-toolbar Pattern

Toolbars group buttons into sections. Sections can nest toolbar-groups.

```html
<bim-toolbar>
  <bim-toolbar-section label="Navigation">
    <bim-button icon="open_with" label="Orbit"
      @click=${() => camera.set("Orbit")}></bim-button>
    <bim-button icon="visibility" label="First Person"
      @click=${() => camera.set("FirstPerson")}></bim-button>
  </bim-toolbar-section>

  <bim-toolbar-section label="Tools">
    <bim-toolbar-group>
      <bim-button icon="straighten" label="Measure"></bim-button>
      <bim-button icon="content_cut" label="Clip"></bim-button>
    </bim-toolbar-group>
  </bim-toolbar-section>
</bim-toolbar>
```

### Vertical Toolbar

```html
<bim-toolbar vertical>
  <!-- labelsHidden is auto-set when vertical=true -->
  <bim-toolbar-section label="Tools">
    <bim-button icon="select_all" label="Select"></bim-button>
  </bim-toolbar-section>
</bim-toolbar>
```

When `vertical=true`, section labels are hidden by default. Override with:
```typescript
BUI.Manager.config.sectionLabelOnVerticalToolbar = true;
```

## bim-table Pattern

Tables display hierarchical data with filtering, grouping, and selection.

```typescript
const table = document.createElement("bim-table");

table.columns = [
  { name: "Name", width: "minmax(200px, 1fr)" },
  { name: "Type", width: "120px" },
  { name: "Level", width: "100px" },
];

table.data = [
  {
    data: { Name: "Wall-001", Type: "IfcWall", Level: "Level 1" },
    children: [
      { data: { Name: "Opening-001", Type: "IfcOpeningElement", Level: "Level 1" } },
    ],
  },
  {
    data: { Name: "Slab-001", Type: "IfcSlab", Level: "Level 1" },
  },
];

// Enable row selection
table.selectableRows = true;
table.addEventListener("dataselected", (e) => {
  const row = e.detail.data;
});
```

### Table Filtering

```typescript
// Simple text search
table.queryString = "wall";

// Column-specific query
table.queryString = 'Type="IfcWall" & Level="Level 1"';
```

### Table Export

```typescript
table.downloadData("export", "csv");  // or "tsv"
```

## CSS Theming System

All `bim-*` components use CSS custom properties with the `--bim-ui_` prefix.
Override at any level in the DOM tree. Key variable groups:

- **Backgrounds**: `--bim-ui_bg-base`, `--bim-ui_bg-contrast-{10,20,40,60,80,100}`
- **Accent**: `--bim-ui_accent-base`, `--bim-ui_accent-contrast`
- **Sizing**: `--bim-ui_size-{base,2xs,xs,sm,md,lg,xl}`
- **Typography**: `--bim-ui_font-family`

See [references/methods.md](references/methods.md) for full variable list with defaults.

```css
:root {
  --bim-ui_bg-base: #0d1117;
  --bim-ui_accent-base: #238636;
  --bim-ui_font-family: "JetBrains Mono", monospace;
}
```

### Theme Switching

```typescript
BUI.Manager.toggleTheme();       // instant switch
BUI.Manager.toggleTheme(true);   // animated overlay transition
```

This toggles `bim-ui-dark` / `bim-ui-light` CSS classes on the `<html>` element.

## Event Handling

All `bim-*` components dispatch standard DOM events. Use `@event` syntax in
Lit templates or `addEventListener` in vanilla JS.

### Common Events

| Component | Event | Detail |
|---|---|---|
| `bim-button` | `click` | Standard MouseEvent |
| `bim-checkbox` | `change` | `{ checked: boolean }` |
| `bim-text-input` | `input` | `{ value: string }` |
| `bim-number-input` | `change` | `{ value: number }` |
| `bim-color-input` | `change` | `{ color: string, opacity: number }` |
| `bim-dropdown` | `change` | Selected options |
| `bim-panel` | `change` | Aggregated form values |
| `bim-panel` | `hiddenchange` | Hidden state toggled |
| `bim-toolbar` | `hiddenchange` | Hidden state toggled |
| `bim-table` | `dataselected` | `{ data: row }` |
| `bim-table` | `datadeselected` | `{ data: row }` |
| `bim-grid` | `layoutchange` | Layout name changed |

### Vanilla JS Event Handling

```typescript
const btn = document.querySelector("bim-button");
btn.addEventListener("click", () => {
  console.log("Button clicked");
});

const checkbox = document.querySelector("bim-checkbox");
checkbox.addEventListener("change", (e) => {
  console.log("Checked:", checkbox.checked);
});
```

## @thatopen/ui-obc: Functional BIM Components

These are pre-built UI component factories that wire `@thatopen/ui` elements
to `@thatopen/components` engine functionality. ALWAYS import from
`@thatopen/ui-obc` separately.

```typescript
import * as BUI from "@thatopen/ui";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as CUI from "@thatopen/ui-obc";

BUI.Manager.init();
```

### Available Functional Components

| Category | Components | Description |
|---|---|---|
| Buttons | `loadIfc`, `loadFrag` | IFC/Fragment file load buttons |
| Tables | `spatialTree`, `itemsData`, `modelsList` | Model data tables |
| Tables | `viewpointsList`, `topicsList`, `commentsList` | BCF data tables |
| Sections | `topicComments`, `topicInformation` | BCF topic panels |
| Sections | `topicRelations`, `topicViewpoints` | BCF relation panels |
| Forms | Form builders | Configuration forms |
| Charts | Chart builders | Data visualization |

### Usage Pattern

ui-obc components are factory functions that return configured HTML elements:

```typescript
// Create a spatial tree table wired to the engine
const [tree, updateTree] = CUI.tables.spatialTree({
  components,
  models: [],
});

// Create IFC load button
const [loadBtn, updateLoadBtn] = CUI.buttons.loadIfc({ components });

// Append to DOM
panel.appendChild(tree);
toolbar.appendChild(loadBtn);
```

The factory pattern returns a tuple: `[element, updateFunction]`. Call
the update function when engine state changes to refresh the UI.

## Slot-Based Composition

ThatOpen UI uses the Web Components slot system. Parent components define
slots; child components fill them. ALWAYS nest components correctly:

| Parent | Accepts (default slot) |
|---|---|
| `bim-panel` | `bim-panel-section` |
| `bim-panel-section` | Any form elements or content |
| `bim-toolbar` | `bim-toolbar-section` |
| `bim-toolbar-section` | `bim-button`, `bim-toolbar-group` |
| `bim-toolbar-group` | `bim-button` |
| `bim-dropdown` | `bim-option` |
| `bim-selector` | `bim-option` |
| `bim-tabs` | `bim-tab` |

## Complete BIM Application Layout

See [references/examples.md](references/examples.md) E-001 for the full
implementation. The essential pattern:

```typescript
BUI.Manager.init();
// ... engine setup, create world ...

const viewport = document.createElement("bim-viewport");
// ... renderer uses viewport as container ...

const grid = document.createElement("bim-grid");
grid.layouts = {
  main: {
    template: `"toolbar toolbar" auto "panel viewport" 1fr / 320px 1fr`,
    elements: { toolbar, panel, viewport },
  },
};
grid.layout = "main";
document.body.appendChild(grid);
```

## Critical Rules

1. **ALWAYS** call `BUI.Manager.init()` before using any `bim-*` element.
2. **ALWAYS** use the `bim-` prefix for element tag names — these are the
   registered custom element names. NEVER use unprefixed names.
3. **ALWAYS** set `grid.layouts` before setting `grid.layout` — the layout
   name must exist in the layouts object.
4. **ALWAYS** import `@thatopen/ui-obc` separately from `@thatopen/ui` —
   they are different packages with different dependencies.
5. **NEVER** use `bim-*` elements without `BUI.Manager.init()` — they will
   render as empty unknown elements with no functionality.
6. **NEVER** set properties on `bim-*` elements before they are defined —
   call `Manager.init()` first or wait for `customElements.whenDefined()`.
7. **NEVER** style `bim-*` internals directly — use CSS custom properties
   (`--bim-ui_*`) for theming. Shadow DOM encapsulates internal styles.

## Reference Files

- [references/methods.md](references/methods.md) — BUI.Manager API, complete
  component property catalog, CSS custom properties
- [references/examples.md](references/examples.md) — Panel layout, toolbar,
  table, property panel, and grid examples
- [references/anti-patterns.md](references/anti-patterns.md) — Missing init,
  wrong element names, direct style manipulation

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_ui-components` main branch (packages/core/src/)
- npm: `@thatopen/ui@3.3.3`, `@thatopen/ui-obc@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md` (Section 8)
