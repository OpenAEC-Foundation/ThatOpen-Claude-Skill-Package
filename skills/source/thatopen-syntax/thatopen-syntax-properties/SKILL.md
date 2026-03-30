---
name: thatopen-syntax-properties
description: >
  Use when querying IFC properties, classifying model elements, or
  extracting data from loaded ThatOpen fragments.
  Prevents inefficient property queries and missing classifications.
  Covers Classifier (byCategory, byIfcBuildingStorey, byModel), find()
  queries, FragmentsManager.getData(), ItemsFinder, IFC relationships.
  Keywords: classifier, properties, ifc, category, storey, find, getdata, spatial structure, property set, quantity set, classification, get properties, read pset, filter by type.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Property Queries and Classification

## Overview

ThatOpen provides two complementary systems for organizing and extracting
data from loaded BIM models:

1. **Classifier** — Groups fragment items by IFC category, building storey,
   model, or custom criteria. Produces `ModelIdMap` results for downstream
   operations (hiding, highlighting, data extraction).
2. **FragmentsManager.getData()** — Extracts IFC property data (property
   sets, quantity sets, type information, relationships) for targeted items.
3. **ItemsFinder** — Searches items across models using query parameters
   with category, attribute, and relationship filters.

All three operate on the fragment layer, not raw web-ifc. For direct
web-ifc property access (GetLine, GetPropertySets), see
`thatopen-core-web-ifc`.

**Pipeline:**
```
Loaded FragmentsModel
  ├─ Classifier.byCategory()      → classification groups
  ├─ Classifier.byIfcBuildingStorey() → storey groups
  ├─ Classifier.find({...})        → ModelIdMap (intersection)
  ├─ ItemsFinder.getItems(queries) → ModelIdMap (filtered)
  └─ FragmentsManager.getData(items) → IFC property data
```

## Critical Warnings

1. **ALWAYS classify models AFTER loading.** Classification methods read
   data from loaded FragmentsModels. Calling `byCategory()` before any
   model is loaded produces empty groups.

2. **ALWAYS call `FragmentsManager.init(workerURL)` before getData().**
   Property extraction runs in the web worker. Without initialization,
   `getData()` fails silently or throws.

3. **NEVER assume classification groups persist across model loads.**
   When a model is disposed and reloaded, ALWAYS re-run classification
   methods to rebuild groups.

4. **ALWAYS use `find()` to get a ModelIdMap from classifications.**
   NEVER manually iterate `classifier.list` to build item maps — `find()`
   handles intersection logic correctly across multiple classifications.

5. **NEVER query properties for thousands of items at once without
   pagination.** `getData()` runs in the worker but still serializes
   results back to the main thread. Large result sets cause jank.

## Classifier

### Purpose

The Classifier organizes fragment items into named classification groups.
Each classification (e.g., "Categories", "Storeys", "Models") contains
named groups (e.g., "IFCWALL", "Ground Floor", "Arch Model") that map
to sets of element IDs via `ModelIdMap`.

### Data Structure

```typescript
classifier.list: DataMap<string, DataMap<string, ClassificationGroupData>>
//                │                  │                │
//                classification     group name       group data (items + query)
//                name
```

**ClassificationGroupData** contains:
- A `get()` method returning `Promise<ModelIdMap>` — the items in the group
- Optional query configuration for dynamic groups
- Item storage maps keyed by model ID

### Getting the Classifier

```typescript
import * as OBC from "@thatopen/components";

const classifier = components.get(OBC.Classifier);
```

### Built-in Classification Methods

#### `byCategory(config?): Promise<void>`

Groups all items by their IFC entity type (e.g., IFCWALL, IFCSLAB,
IFCDOOR). Default classification name: `"Categories"`.

```typescript
await classifier.byCategory();
// classifier.list now has "Categories" with groups like "IFCWALL", "IFCSLAB"
```

#### `byIfcBuildingStorey(config?): Promise<void>`

Groups items by the building storey they belong to, using the
`ContainsElements` IFC relationship. Default classification name:
`"Storeys"`.

```typescript
await classifier.byIfcBuildingStorey();
// classifier.list now has "Storeys" with groups like "Ground Floor", "Level 1"
```

#### `byModel(config?): Promise<void>`

Groups items by their parent FragmentsModel. Default classification
name: `"Models"`.

```typescript
await classifier.byModel();
// classifier.list now has "Models" with groups per loaded model
```

#### AddClassificationConfig

All three methods accept an optional config:

```typescript
interface AddClassificationConfig {
  classificationName?: string;  // Override default name
  modelIds?: RegExp[];          // Filter: only classify matching models
}
```

Example with custom name and model filter:

```typescript
await classifier.byCategory({
  classificationName: "Element Types",
  modelIds: [/arch/i]  // Only classify models whose ID matches "arch"
});
```

### Querying Classifications with find()

`find()` accepts an object where keys are classification names and values
are arrays of group names. It returns the **intersection** of all
specified groups as a `ModelIdMap`.

```typescript
// Type: ClassifierIntersectionInput
// Keys = classification names, Values = arrays of group names
const items = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: ["Ground Floor"]
});
// Returns: ModelIdMap of walls on the ground floor only
```

**Intersection logic:** When multiple classifications are specified,
`find()` returns only items that appear in ALL specified groups. This
enables powerful cross-classification queries.

### Custom Groups

#### getGroupData(classification, group): ClassificationGroupData

Retrieves or creates a group. Use this to build custom classifications.

```typescript
const groupData = classifier.getGroupData("Fire Rating", "REI 120");
```

#### addGroupItems(classification, group, items): void

Adds items to a specific group within a classification.

```typescript
const wallItems: ModelIdMap = {
  [model.modelId]: new Set([42, 43, 44])
};
classifier.addGroupItems("Custom", "Selected Walls", wallItems);
```

#### removeItems(modelIdMap, config?): void

Removes items from the classifier. Without config, removes from ALL
classifications.

```typescript
// Remove specific items from all classifications
classifier.removeItems(itemsToRemove);

// Remove with config (RemoveClassifierItemsConfig)
classifier.removeItems(itemsToRemove, { /* config */ });
```

#### setGroupQuery(classification, group, query): void

Assigns a dynamic query to a group. When `get()` is called on the group,
the query runs via ItemsFinder to produce fresh results.

```typescript
classifier.setGroupQuery("Custom", "First Floor Walls", {
  name: "First Floor Walls"  // References a named ItemsFinder query
});
```

### Aggregation Methods

#### aggregateItems(classification, query, config?): Promise<void>

Groups items per classification using a query. Each item matching the
query is placed into a group based on the aggregation callback.

#### aggregateItemRelations(classification, query, relation, config?): Promise<void>

Groups items by IFC relationships. Used internally by
`byIfcBuildingStorey()` to group elements by their containing storey
via the `ContainsElements` relation.

```typescript
await classifier.aggregateItemRelations(
  "Storeys",
  { categories: [/BUILDINGSTOREY/] },
  "ContainsElements"
);
```

## FragmentsManager.getData()

### Purpose

Extracts IFC property data for targeted items. Runs in the web worker
for performance. Returns structured property information including
property sets, quantity sets, type data, and relationship data.

### Signature

```typescript
getData(
  items: ModelIdMap,
  config?: Partial<ItemsDataConfig>
): Promise<Record<string, ItemData[]>>
```

### Parameters

- **items** — `ModelIdMap` specifying which elements to query
- **config** — Optional `Partial<ItemsDataConfig>` controlling what data
  to retrieve

### Return Value

Returns `Record<string, ItemData[]>`:
- Keys are model UUID strings
- Values are arrays of `ItemData` objects, one per queried element

### ItemData

Each `ItemData` object represents the IFC data for one element. It
contains the element's properties organized by their IFC structure:
property sets, type information, attributes, and relationships.

### Basic Usage

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Target specific elements
const items: ModelIdMap = {
  [model.modelId]: new Set([42, 43])
};

// Extract all available property data
const data = await fragments.getData(items);

for (const [modelId, itemDataArray] of Object.entries(data)) {
  for (const itemData of itemDataArray) {
    console.log("Element data:", itemData);
  }
}
```

### Combined with Classifier

The most common pattern: classify first, then extract properties for
the classified items.

```typescript
const classifier = components.get(OBC.Classifier);
const fragments = components.get(OBC.FragmentsManager);

// Classify
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

// Find walls on the ground floor
const wallItems = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: ["Ground Floor"]
});

// Extract properties for those walls
const wallData = await fragments.getData(wallItems);
```

## ItemsFinder

### Purpose

ItemsFinder searches and filters items across loaded models using
structured query parameters. It supports filtering by IFC category,
element attributes, and IFC relationships. Queries can be saved,
named, and reused.

### Getting ItemsFinder

```typescript
const finder = components.get(OBC.ItemsFinder);
```

### Creating Named Queries

```typescript
const query = finder.create("First Floor Walls", [
  {
    categories: [/WALL/],
    relation: {
      name: "ContainedInStructure",
      query: {
        categories: [/STOREY/],
        attributes: {
          queries: [{ name: /Name/, value: /01/ }]
        }
      }
    }
  }
]);
```

### Query Parameters (ItemsQueryParams)

```typescript
interface ItemsQueryParams {
  categories?: RegExp[];      // Filter by IFC category (regex match)
  relation?: {                // Filter by IFC relationship
    name: string;             // Relationship name (e.g., "ContainedInStructure")
    query: ItemsQueryParams;  // Nested query for related items
  };
  attributes?: {              // Filter by element attributes
    queries: Array<{
      name: RegExp;           // Attribute name pattern
      value: RegExp;          // Attribute value pattern
    }>;
  };
}
```

### Executing Queries

```typescript
// Execute with getItems()
const items = await finder.getItems(
  [{ categories: [/WALL/, /SLAB/] }],
  {
    modelIds: [/arch/i],             // Optional: filter models by ID pattern
    aggregation: "union"             // "union" or "intersection"
  }
);
// Returns: ModelIdMap
```

### Auto-Generate Category Queries

```typescript
// Create queries for all geometric categories in loaded models
const categoryNames = await finder.addFromCategories();
// Returns: string[] of category names found (e.g., ["IFCWALL", "IFCSLAB"])
```

### Serialization

```typescript
// Export queries for storage
const serialized = finder.export();

// Import previously saved queries
await finder.import(serialized);
```

## IFC Relationships in Fragment Context

The fragment system preserves key IFC relationships that can be queried
through Classifier and ItemsFinder:

| Relationship | IFC Entity | Fragment Usage |
|---|---|---|
| Spatial containment | IfcRelContainedInSpatialStructure | `byIfcBuildingStorey()`, `ContainedInStructure` relation queries |
| Aggregation | IfcRelAggregates | Spatial hierarchy traversal |
| Property assignment | IfcRelDefinesByProperties | `getData()` returns property sets |
| Type assignment | IfcRelDefinesByType | `getData()` returns type information |
| Material assignment | IfcRelAssociatesMaterial | Available through `getData()` |

### Spatial Structure via Classification

The spatial hierarchy (Project > Site > Building > Storey > Space) is
accessible through the Classifier's storey classification:

```typescript
await classifier.byIfcBuildingStorey();

// Access storey groups
const storeyGroups = classifier.list.get("Storeys");
if (storeyGroups) {
  for (const [storeyName, groupData] of storeyGroups) {
    const items = await groupData.get();
    console.log(`${storeyName}: ${Object.keys(items).length} models`);
  }
}
```

### Property Sets and Quantity Sets

IFC models contain two types of element metadata accessible via
`getData()`:

1. **Property Sets (Pset_)** — Named groups of properties assigned via
   `IfcRelDefinesByProperties`. Common examples: `Pset_WallCommon`,
   `Pset_DoorCommon`. Contains `IfcPropertySingleValue` entries with
   name-value pairs.

2. **Quantity Sets (Qto_)** — Physical measurements assigned via the same
   relationship. Common examples: `Qto_WallBaseQuantities`,
   `Qto_SpaceBaseQuantities`. Contains typed quantities (length, area,
   volume, weight, count).

For detailed IFC property set and quantity set schemas, see the
`ifc-bim-standards` skill.

## Workflow: Classify, Query, Extract

The standard workflow for working with IFC properties in ThatOpen:

```
1. Load model          → FragmentsModel in scene
2. Classify            → Classifier.byCategory(), byIfcBuildingStorey()
3. Query               → Classifier.find() or ItemsFinder.getItems()
4. Extract properties  → FragmentsManager.getData(items)
5. Use the data        → Display in UI, export, validate
```

**ALWAYS follow this order.** Classifying before loading produces empty
groups. Extracting properties without targeting specific items wastes
worker bandwidth.

## Quick Reference

| Task | Method |
|---|---|
| Classify by IFC type | `classifier.byCategory()` |
| Classify by storey | `classifier.byIfcBuildingStorey()` |
| Classify by model | `classifier.byModel()` |
| Find items across classifications | `classifier.find({ Classification: ["Group"] })` |
| Add items to custom group | `classifier.addGroupItems(class, group, items)` |
| Set dynamic query on group | `classifier.setGroupQuery(class, group, query)` |
| Remove items from classifier | `classifier.removeItems(items)` |
| Create named search query | `finder.create(name, queries)` |
| Search items by query | `finder.getItems(queries, config?)` |
| Auto-create category queries | `finder.addFromCategories()` |
| Extract IFC properties | `fragments.getData(items)` |

## Related Skills

- `thatopen-core-fragments` — FragmentsManager, ModelIdMap, worker setup
- `thatopen-core-web-ifc` — Raw web-ifc property access (GetLine, GetPropertySets)
- `thatopen-core-architecture` — Component system, world setup
- `thatopen-syntax-ifc-loading` — IfcLoader configuration, WASM setup
- `thatopen-impl-viewer` — Full viewer setup including classification UI

## References

- [references/methods.md](references/methods.md) — Classifier API, getData config, ItemsFinder
- [references/examples.md](references/examples.md) — Classify, find, getData, spatial queries
- [references/anti-patterns.md](references/anti-patterns.md) — Wrong query patterns, missing classification
