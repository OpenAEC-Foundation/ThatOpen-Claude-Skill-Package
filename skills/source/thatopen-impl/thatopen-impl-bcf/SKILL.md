---
name: thatopen-impl-bcf
description: >
  Use when implementing BCF (BIM Collaboration Format) issue tracking,
  viewpoint management, or IDS validation in a ThatOpen viewer.
  Prevents BCF version mismatches and missing topic configuration.
  Covers BCFTopics (create, load, export), BCF v2.1 and v3.0 support,
  Viewpoints, topic configuration, IDSSpecifications overview.
  Keywords: bcf, topic, viewpoint, issue, collaboration, ids, export, import, bim collaboration format, snapshot, share issues, track problems, report clash.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen BCF & Viewpoints

## Overview

This skill covers BIM Collaboration Format (BCF) support in
`@thatopen/components`: the `BCFTopics` component for issue tracking
and the `Viewpoints` component for 3D scene state capture. BCF enables
structured communication about BIM model issues between different tools.

**Version**: @thatopen/components 3.3.x
**Prerequisites**: thatopen-impl-viewer (world setup), thatopen-core-architecture
**Dependencies**: jszip (BCF zip), fast-xml-parser (BCF XML)

## Component Overview

| Component | Package | UUID | Purpose |
|---|---|---|---|
| `BCFTopics` | `@thatopen/components` | `de977976-e4f6-4e4f-a01a-204727839802` | Issue tracking, BCF import/export |
| `Viewpoints` | `@thatopen/components` | `ee867824-a796-408d-8aa0-4e5962a83c66` | 3D camera state capture, snapshots |
| `IDSSpecifications` | `@thatopen/components` | — | IDS validation (separate module) |

## BCFTopics

### Setup

ALWAYS call `setup()` with configuration before creating or importing topics:

```typescript
import * as OBC from "@thatopen/components";

const bcfTopics = components.get(OBC.BCFTopics);
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  types: new Set(["Issue", "Request", "Comment"]),
  statuses: new Set(["Active", "Resolved", "Closed"]),
  priorities: new Set(["Critical", "Major", "Normal", "Minor"]),
  labels: new Set(["Architecture", "Structure", "MEP"]),
  stages: new Set(["Design", "Construction", "Handover"]),
  users: new Set(["user@example.com", "reviewer@example.com"]),
});
```

### BCFTopicsConfig

| Property | Type | Default | Description |
|---|---|---|---|
| `version` | `"2.1" \| "3"` | `""` | BCF version for export |
| `author` | `string` | `""` | User email for topic/comment creation |
| `types` | `Set<string>` | empty | Allowed topic types |
| `statuses` | `Set<string>` | empty | Allowed topic statuses |
| `priorities` | `Set<string>` | empty | Allowed topic priorities |
| `labels` | `Set<string>` | empty | Allowed topic labels |
| `stages` | `Set<string>` | empty | Allowed topic stages |
| `users` | `Set<string>` | empty | Allowed user emails |
| `strict` | `boolean` | `false` | Enforce extensions validation |
| `includeSelectionTag` | `boolean` | `false` | Include AuthoringSoftwareId in viewpoints |
| `updateExtensionsOnImport` | `boolean` | `false` | Auto-update extensions after import |
| `includeAllExtensionsOnExport` | `boolean` | `false` | Export all found extensions |
| `fallbackVersionOnImport` | `BCFVersion \| null` | `null` | Default version if missing in BCF file |
| `ignoreIncompleteTopicsOnImport` | `boolean` | `false` | Skip topics missing required fields |
| `exportCustomDataAsLabels` | `boolean` | `false` | Export customData as labels |

### Strict Mode

When `strict: true`, all Topic property setters validate against the
configured extensions. Setting `topic.type = "Unknown"` throws if
`"Unknown"` is not in `config.types`. When `strict: false` (default),
any value is accepted.

### Properties

| Property | Type | Description |
|---|---|---|
| `list` | `DataMap<string, Topic>` | All topics indexed by GUID |
| `documents` | `DataMap<string, DocumentReference>` | Internal/external document references |
| `enabled` | `boolean` | Component enabled state |
| `isSetup` | `boolean` | Whether `setup()` has been called |

### Events

| Event | Payload | Trigger |
|---|---|---|
| `onSetup` | — | After `setup()` completes |
| `onBCFImported` | `Topic[]` | After `load()` imports topics |
| `onDisposed` | — | After `dispose()` |

### Methods

| Method | Returns | Description |
|---|---|---|
| `setup(config?)` | `void` | Initialize with configuration |
| `create(data?)` | `Topic` | Create a new topic |
| `load(data)` | `Promise<{viewpoints, topics}>` | Import BCF zip data |
| `export(topics?)` | `Promise<Blob>` | Export topics to BCF zip |
| `updateExtensions()` | `void` | Sync config sets with current topics |
| `updateViewpointReferences()` | `void` | Remove stale viewpoint references |
| `dispose()` | `void` | Clean up all resources |

### Computed Getters

| Getter | Returns | Description |
|---|---|---|
| `usedTypes` | `Set<string>` | All types currently in use |
| `usedStatuses` | `Set<string>` | All statuses currently in use |
| `usedPriorities` | `Set<string>` | All priorities currently in use |
| `usedStages` | `Set<string>` | All stages currently in use |
| `usedUsers` | `Set<string>` | All users from authors and comments |
| `usedLabels` | `Set<string>` | All labels across topics |

## Topic

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `guid` | `string` | auto-generated | Unique identifier |
| `title` | `string` | `"BCF Topic"` | Topic title |
| `type` | `string` | `"Issue"` | Topic type (validated in strict mode) |
| `status` | `string` | `"Active"` | Topic status (validated in strict mode) |
| `priority` | `string?` | — | Priority (validated in strict mode) |
| `stage` | `string?` | — | Project stage (validated in strict mode) |
| `assignedTo` | `string?` | — | Assigned user email |
| `description` | `string?` | — | Topic description |
| `labels` | `Set<string>` | empty | Topic labels/tags |
| `dueDate` | `Date?` | — | Due date |
| `index` | `number?` | — | Display ordering index |
| `creationDate` | `Date` | auto-set | Creation timestamp |
| `creationAuthor` | `string` | from config | Author email |
| `modifiedDate` | `Date?` | — | Last modification timestamp |
| `modifiedAuthor` | `string?` | — | Last modifier email |
| `customData` | `Record<string, any>` | `{}` | Arbitrary metadata |

### Topic References (stored as GUIDs)

| Property | Type | Description |
|---|---|---|
| `viewpoints` | `DataSet<string>` | Associated viewpoint GUIDs |
| `relatedTopics` | `DataSet<string>` | Related topic GUIDs (no self-reference) |
| `comments` | `DataMap<string, Comment>` | Comments on this topic |
| `documentReferences` | `DataSet<string>` | Document reference GUIDs |

### Topic Methods

| Method | Returns | Description |
|---|---|---|
| `set(data)` | `Topic` | Bulk update properties (skips GUID) |
| `createComment(text, viewpoint?)` | `Comment` | Create a comment on this topic |
| `toJSON()` | `BCFApiTopic` | Serialize to API format |
| `serialize()` | `string` | Generate BCF XML markup |

### set() vs Direct Assignment

Direct property assignment updates internally without triggering events.
Use `set()` to broadcast changes for reactive UI updates:

```typescript
// Silent update — no events fired
topic.title = "Updated Title";

// Reactive update — listeners notified
topic.set({ title: "Updated Title", status: "Resolved" });
```

## Comment

| Property | Type | Description |
|---|---|---|
| `guid` | `string` | Unique identifier |
| `date` | `Date` | Creation timestamp |
| `author` | `string` | From config at creation time |
| `comment` | `string` | Text (setter updates modifiedDate/modifiedAuthor) |
| `viewpoint` | `string?` | Associated viewpoint GUID |
| `modifiedDate` | `Date?` | Auto-set on comment text change |
| `modifiedAuthor` | `string?` | Auto-set from config on change |

## Document References

Two types of document references stored in `bcfTopics.documents`:

```typescript
// Internal (embedded in BCF zip)
{ type: "internal", fileName: "report.pdf", data: Uint8Array, description?: string }

// External (URL reference)
{ type: "external", url: "https://...", description?: string }
```

## Viewpoints

### Setup

```typescript
const viewpoints = components.get(OBC.Viewpoints);
viewpoints.world = world; // REQUIRED — default world for viewpoint creation
```

### Properties

| Property | Type | Description |
|---|---|---|
| `list` | `DataMap<string, Viewpoint>` | All viewpoints indexed by GUID |
| `snapshots` | `DataMap<string, Uint8Array>` | Binary snapshot data |
| `world` | `World \| null` | Default world for creation |
| `enabled` | `boolean` | Defaults to `true` |

### Methods

| Method | Returns | Description |
|---|---|---|
| `create(data?)` | `Viewpoint` | Create a viewpoint (optionally from BCFViewpoint data) |
| `getSnapshotExtension(name)` | `string` | Detect snapshot format from header bytes |
| `dispose()` | `void` | Clean up resources |

### Viewpoint Instance

Each Viewpoint captures a complete 3D scene state:

| Property | Type | Description |
|---|---|---|
| `guid` | `string` | Unique identifier |
| `title` | `string?` | Viewpoint name |
| `camera` | camera data | Perspective or orthogonal camera settings |
| `defaultVisibility` | `boolean` | Base visibility state |
| `selectionComponents` | `DataSet<string>` | Component GUIDs to highlight |
| `exceptionComponents` | `DataSet<string>` | Visibility override GUIDs |
| `componentColors` | `DataMap<string, string[]>` | Hex color to GUID array |
| `clippingPlanes` | `DataSet<string>` | Enabled clipping plane IDs |
| `spacesVisible` | `boolean` | Show IfcSpace elements |
| `spaceBoundariesVisible` | `boolean` | Show space boundaries |
| `openingsVisible` | `boolean` | Show IfcOpeningElement |
| `snapshot` | `string?` | Snapshot reference ID |
| `customData` | `Record<string, any>` | Arbitrary metadata |

### Viewpoint Methods

| Method | Returns | Description |
|---|---|---|
| `updateCamera(takeSnapshot?)` | `void` | Sync from current world camera |
| `go(config?)` | `Promise<void>` | Apply viewpoint to world |
| `takeSnapshot()` | `void` | Capture canvas to snapshots map |
| `applyVisibility()` | `void` | Enforce visibility/exceptions |
| `setColorizationState(state)` | `void` | Apply/reset component colors |
| `updateClippingPlanes()` | `void` | Sync from Clipper component |
| `toJSON()` | `BCFViewpoint` | Serialize to BCF data |
| `serialize(version)` | `string` | Generate BCF XML (v2.1 or v3.0) |

## BCF Version Differences

| Feature | BCF 2.1 | BCF 3.0 |
|---|---|---|
| XML schema | `bcf/2.1` namespace | `bcf/3.0` namespace |
| Viewpoint references | `<Viewpoints>` element | `<Viewpoints>` element |
| Document references | In markup XML | In markup XML |
| Labels | `<Labels>` element | `<Labels>` element |
| Related topics | `<RelatedTopics>` | `<RelatedTopics>` |
| Extensions file | `bcf.extensions` | `bcf.extensions` |
| Config `version` value | `"2.1"` | `"3"` |

ALWAYS set `config.version` before exporting. The serialization format
of viewpoints and markup XML differs between versions.

## BCF Import/Export Workflow

### Export

```typescript
// Export all topics
const blob = await bcfTopics.export();

// Export specific topics
const selectedTopics = [...bcfTopics.list.values()].filter(t => t.status === "Active");
const blob = await bcfTopics.export(selectedTopics);

// Download in browser
const url = URL.createObjectURL(blob);
const link = document.createElement("a");
link.href = url;
link.download = "issues.bcf";
link.click();
URL.revokeObjectURL(url);
```

### Import

```typescript
// From file input
const input = document.createElement("input");
input.type = "file";
input.accept = ".bcf,.bcfzip";
input.addEventListener("change", async () => {
  const file = input.files?.[0];
  if (!file) return;
  const data = new Uint8Array(await file.arrayBuffer());
  const { topics, viewpoints } = await bcfTopics.load(data);
  console.log(`Imported ${topics.length} topics, ${viewpoints.length} viewpoints`);
});
input.click();
```

### Linking Viewpoints to Topics

ALWAYS link viewpoints to topics using GUIDs, not object references:

```typescript
const viewpoint = viewpoints.create();
viewpoint.title = "Clash at Level 2";
await viewpoint.updateCamera();
viewpoint.takeSnapshot();

const topic = bcfTopics.create({
  title: "Steel beam clashes with duct",
  type: "Issue",
  priority: "Critical",
});
topic.viewpoints.add(viewpoint.guid);
```

### Auto-linking on Creation

```typescript
viewpoints.list.onItemSet.add(({ value: vp }) => {
  const topic = bcfTopics.create();
  topic.viewpoints.add(vp.guid);
});
```

## IDSSpecifications Overview

The `IDSSpecifications` component in `@thatopen/components` provides
Information Delivery Specification (IDS) validation. IDS is a buildingSMART
standard for specifying information requirements on BIM models.

This component is exported from the `openbim` module alongside BCFTopics.
For detailed IDS usage, refer to the ThatOpen documentation.

## Complete Setup Pattern

```typescript
import * as OBC from "@thatopen/components";

// 1. Get components
const bcfTopics = components.get(OBC.BCFTopics);
const viewpoints = components.get(OBC.Viewpoints);

// 2. Configure BCFTopics (REQUIRED before create/load/export)
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  types: new Set(["Issue", "Request", "Comment"]),
  statuses: new Set(["Active", "Resolved", "Closed"]),
  priorities: new Set(["Critical", "Major", "Normal", "Minor"]),
  labels: new Set(["Architecture", "Structure", "MEP"]),
  stages: new Set(["Design", "Construction"]),
  users: new Set(["user@example.com"]),
  strict: false,
});

// 3. Set viewpoints world (REQUIRED before creating viewpoints)
viewpoints.world = world;

// 4. Create topic with viewpoint
const viewpoint = viewpoints.create();
await viewpoint.updateCamera();
viewpoint.takeSnapshot();

const topic = bcfTopics.create({
  title: "Coordination issue at grid A-3",
  type: "Issue",
  priority: "Major",
  assignedTo: "user@example.com",
  description: "Steel column interferes with HVAC duct routing",
});
topic.viewpoints.add(viewpoint.guid);
topic.createComment("Please review and propose resolution");
```

## Critical Rules

1. **ALWAYS** call `bcfTopics.setup()` before creating, loading, or
   exporting topics. Without setup, the author field is empty and
   extensions are not configured.
2. **ALWAYS** set `viewpoints.world` before creating viewpoints. Camera
   capture and snapshot require a valid world reference.
3. **ALWAYS** set `config.version` to `"2.1"` or `"3"` before exporting.
   An empty version string produces invalid BCF output.
4. **ALWAYS** link viewpoints to topics via `topic.viewpoints.add(guid)`,
   not by storing object references. GUIDs prevent memory leaks.
5. **ALWAYS** call `updateCamera()` after creating a viewpoint to capture
   the current camera state. New viewpoints have no camera data by default.
6. **NEVER** skip `setup()` and rely on defaults. All config sets start
   empty, meaning strict mode would reject every value.
7. **NEVER** assume imported BCF files specify a version. Use
   `fallbackVersionOnImport` to handle version-less files.
8. **NEVER** mix BCF version strings: use `"2.1"` or `"3"` (not `"3.0"`).
9. **NEVER** store Topic or Viewpoint object references in external data
   structures. Use GUIDs from `topic.guid` / `viewpoint.guid` and look
   up via `bcfTopics.list.get(guid)` / `viewpoints.list.get(guid)`.
10. **NEVER** forget to call `dispose()` or `components.dispose()` on
    cleanup. BCFTopics and Viewpoints hold DataMaps that must be freed.

## Reference Files

- [references/methods.md](references/methods.md) — BCFTopics, Viewpoints,
  Topic, Comment full API reference
- [references/examples.md](references/examples.md) — Create topic, import/
  export BCF, viewpoints integration examples
- [references/anti-patterns.md](references/anti-patterns.md) — Wrong BCF
  version, missing config, memory leaks

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch
  (`packages/core/src/openbim/BCFTopics/`, `packages/core/src/core/Viewpoints/`)
- npm: `@thatopen/components@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md` (Section 7)
