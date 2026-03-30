---
name: thatopen-agents-model-analyzer
description: >
  Use when analyzing an IFC model's contents, generating reports on element
  counts, properties, or spatial structure.
  Prevents inefficient model queries and incomplete analysis.
  Covers IFC model analysis: spatial structure extraction, property set
  enumeration, element counting by type, classification reports, model
  validation, data export, quality checks.
  Keywords: analyze, analysis, report, model, ifc, properties, count, spatial, structure, validate, quality, inventory, how many walls, list all elements, model summary, element statistics.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Model Analyzer: Agent Workflow

## Purpose

This is an **agent skill** — it defines a guided analysis workflow for
extracting structured information from loaded IFC models. Use it to produce
model summaries, element inventories, property reports, spatial structure
maps, classification breakdowns, and quality validation results.

The workflow combines ThatOpen's Classifier, FragmentsManager.getData(),
ItemsFinder, and direct web-ifc queries to build a complete picture of a
model's contents.

## Prerequisites

Before starting any analysis workflow, verify these conditions:

1. **Model is loaded** — A `FragmentsModel` exists in
   `components.get(OBC.FragmentsManager).list`. NEVER attempt analysis on
   an unloaded model.
2. **FragmentsManager is initialized** — `fragments.init(workerURL)` has
   been called. ALWAYS verify this before calling `getData()`.
3. **web-ifc is accessible** — For low-level queries, `IfcLoader.webIfc`
   provides the `IfcAPI` instance. NEVER create a second `IfcAPI`.

## Critical Rules

1. **ALWAYS classify before querying.** Run `Classifier.byCategory()` and
   `Classifier.byIfcBuildingStorey()` before any analysis step that depends
   on classification groups.

2. **NEVER query properties for all elements at once.** ALWAYS paginate
   `getData()` calls — batch by type or storey to avoid main-thread jank.

3. **ALWAYS use `ModelIdMap` as the interchange format** between analysis
   steps. Every Classifier result, ItemsFinder result, and getData input
   uses `ModelIdMap` (`Map<string, Set<number>>`).

4. **ALWAYS dispose analysis resources when done.** If you created temporary
   classifications or groups, clean them up.

5. **NEVER assume property sets exist.** Not all IFC models have complete
   property data. ALWAYS handle missing Psets gracefully.

6. **ALWAYS detect the IFC schema version first.** Use
   `ifcApi.GetModelSchema(modelID)` — behavior differs between IFC2X3,
   IFC4, and IFC4X3.

---

## Analysis Workflow: Step by Step

### Phase 1: Model Identification

Collect basic model metadata before deeper analysis.

```typescript
import * as OBC from "@thatopen/components";

const fragments = components.get(OBC.FragmentsManager);
const loader = components.get(OBC.IfcLoader);
const ifcApi = loader.webIfc;

// Step 1a: List loaded models
for (const [modelId, model] of fragments.list) {
  console.log(`Model: ${modelId}`);
}

// Step 1b: Get schema version (requires the web-ifc modelID)
const schema = ifcApi.GetModelSchema(modelID);
// Returns: "IFC2X3" | "IFC4" | "IFC4X3"

// Step 1c: Get all IFC types present in model
const allTypes = ifcApi.GetAllTypesOfModel(modelID);
// Returns: Array<{ typeID: number, typeName: string }>
```

**Decision point:** If `allTypes` returns fewer than expected types, the
model may have been loaded with filtered IFC classes. Check IfcLoader
settings.

### Phase 2: Classification

Build the classification index for all subsequent queries.

```typescript
const classifier = components.get(OBC.Classifier);

// ALWAYS run both — they are independent and fast
await classifier.byCategory();
await classifier.byIfcBuildingStorey();
await classifier.byModel();
```

**Decision point:** After classification, inspect
`classifier.list.get("Categories")` — if it contains fewer groups than
expected from Phase 1's type list, some elements may lack geometry (spatial
elements like IfcProject are not classified by category).

### Phase 3: Element Inventory

Count elements by IFC type using classification groups.

```typescript
// Method A: Via Classifier (fragment-level, includes only geometric elements)
const categories = classifier.list.get("Categories");
if (categories) {
  const inventory: Record<string, number> = {};
  for (const [categoryName, groupData] of categories) {
    const items = await groupData.get();
    let count = 0;
    for (const [, ids] of Object.entries(items)) {
      count += (ids as Set<number>).size;
    }
    inventory[categoryName] = count;
  }
  console.log("Element inventory:", inventory);
}

// Method B: Via web-ifc (includes ALL entities, not just geometric)
import { IFCWALL, IFCSLAB, IFCDOOR, IFCWINDOW, IFCBEAM, IFCCOLUMN,
         IFCROOF, IFCSTAIR, IFCFURNISHINGELEMENT } from "web-ifc";

const typesToCount = [
  { type: IFCWALL, name: "Walls" },
  { type: IFCSLAB, name: "Slabs" },
  { type: IFCDOOR, name: "Doors" },
  { type: IFCWINDOW, name: "Windows" },
  { type: IFCBEAM, name: "Beams" },
  { type: IFCCOLUMN, name: "Columns" },
  { type: IFCROOF, name: "Roofs" },
  { type: IFCSTAIR, name: "Stairs" },
  { type: IFCFURNISHINGELEMENT, name: "Furniture" },
];

for (const { type, name } of typesToCount) {
  const ids = ifcApi.GetLineIDsWithType(modelID, type);
  console.log(`${name}: ${ids.size()}`);
}
```

**Decision point:** Choose Method A for visual/geometric element counts.
Choose Method B for complete IFC entity counts (includes non-geometric
entities). For a full report, use both and note the difference.

### Phase 4: Spatial Structure

Extract the project hierarchy.

```typescript
// Method A: Via web-ifc properties helper (complete tree)
const spatialTree = await ifcApi.properties.getSpatialStructure(modelID);
// Returns: { expressID, type, children: [...] }

function printTree(node: any, indent = 0) {
  const prefix = "  ".repeat(indent);
  console.log(`${prefix}${node.type} [#${node.expressID}]`);
  if (node.children) {
    for (const child of node.children) {
      printTree(child, indent + 1);
    }
  }
}
printTree(spatialTree);

// Method B: Via Classifier storey groups (element-to-storey mapping)
const storeys = classifier.list.get("Storeys");
if (storeys) {
  for (const [storeyName, groupData] of storeys) {
    const items = await groupData.get();
    let elementCount = 0;
    for (const ids of Object.values(items)) {
      elementCount += (ids as Set<number>).size;
    }
    console.log(`${storeyName}: ${elementCount} elements`);
  }
}
```

**Decision point:** Method A gives the full IFC hierarchy tree (Project >
Site > Building > Storey > Space). Method B gives only storey-level grouping
with element counts. Use Method A for structural reports, Method B for
per-storey analysis.

### Phase 5: Property Analysis

Extract and enumerate property sets for targeted elements.

```typescript
// Step 5a: Pick a target group (e.g., all walls)
const wallItems = await classifier.find({
  Categories: ["IFCWALL"]
});

// Step 5b: Extract property data (paginated)
const wallData = await fragments.getData(wallItems);

// Step 5c: Enumerate property sets
for (const [modelId, itemDataArray] of Object.entries(wallData)) {
  for (const itemData of itemDataArray) {
    console.log("Element:", itemData);
    // itemData contains property sets, type info, attributes
  }
}

// Step 5d: For detailed property sets via web-ifc
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < Math.min(wallIDs.size(), 5); i++) {
  const psets = await ifcApi.properties.getPropertySets(
    modelID, wallIDs.get(i), false
  );
  console.log(`Wall #${wallIDs.get(i)} property sets:`, psets);
}
```

**ALWAYS limit property queries.** In Step 5d, the `Math.min(... , 5)`
pattern demonstrates sampling. For full reports, iterate in batches.

### Phase 6: Cross-Classification Analysis

Combine classifications for targeted analysis.

```typescript
// Example: Walls on the ground floor
const groundFloorWalls = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: ["Ground Floor"]
});

// Example: All structural elements on Level 1
const structuralLevel1 = await classifier.find({
  Categories: ["IFCWALL", "IFCSLAB", "IFCBEAM", "IFCCOLUMN"],
  Storeys: ["Level 1"]
});

// Extract properties for the cross-classified items
const structData = await fragments.getData(structuralLevel1);
```

### Phase 7: Validation Checks

Run quality checks on the model.

```typescript
// Check 1: Orphaned elements (not in any storey)
const allCategoryItems = await classifier.find({ Categories: ["IFCWALL"] });
const storeyWalls = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: Array.from(storeys?.keys() ?? [])
});

// Compare: items in allCategoryItems but not in storeyWalls are orphaned

// Check 2: Elements without property sets
for (let i = 0; i < wallIDs.size(); i++) {
  const psets = await ifcApi.properties.getPropertySets(
    modelID, wallIDs.get(i), false
  );
  if (!psets || psets.length === 0) {
    console.warn(`Wall #${wallIDs.get(i)} has no property sets`);
  }
}

// Check 3: Missing spatial hierarchy levels
import { IFCPROJECT, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY } from "web-ifc";

const requiredTypes = [
  { type: IFCPROJECT, name: "IfcProject" },
  { type: IFCSITE, name: "IfcSite" },
  { type: IFCBUILDING, name: "IfcBuilding" },
  { type: IFCBUILDINGSTOREY, name: "IfcBuildingStorey" },
];

for (const { type, name } of requiredTypes) {
  const ids = ifcApi.GetLineIDsWithType(modelID, type);
  if (ids.size() === 0) {
    console.warn(`Missing required spatial element: ${name}`);
  }
}
```

### Phase 8: Report Generation

Compile findings into structured output.

```
ALWAYS use this output format for analysis reports:

=== IFC MODEL ANALYSIS REPORT ===

Model: {filename}
Schema: {IFC2X3 | IFC4 | IFC4X3}
Total IFC entity types: {count}

--- ELEMENT INVENTORY ---
| IFC Type         | Count |
|------------------|-------|
| IFCWALL          | {n}   |
| IFCSLAB          | {n}   |
| ...              | ...   |
| TOTAL            | {sum} |

--- SPATIAL STRUCTURE ---
IfcProject: {name}
  IfcSite: {name}
    IfcBuilding: {name}
      IfcBuildingStorey: {name} ({n} elements)
      IfcBuildingStorey: {name} ({n} elements)
      ...

--- PROPERTY SETS ---
| Property Set Name    | Occurrence Count |
|----------------------|------------------|
| Pset_WallCommon      | {n}              |
| ...                  | ...              |

--- VALIDATION ---
[PASS/WARN] Spatial hierarchy completeness
[PASS/WARN] Elements with property sets: {n}/{total} ({%})
[PASS/WARN] Elements assigned to storeys: {n}/{total} ({%})

--- NOTES ---
{Any observations, anomalies, or recommendations}

=== END REPORT ===
```

---

## Decision Tree

Use this to determine which analysis path to follow:

```
User wants to analyze a model
├─ "What's in this model?" → Phase 1 + 2 + 3 (inventory)
├─ "Show me the structure" → Phase 1 + 2 + 4 (spatial)
├─ "What properties do X have?" → Phase 1 + 2 + 5 (properties)
├─ "How many X on floor Y?" → Phase 1 + 2 + 6 (cross-classification)
├─ "Is this model valid?" → Phase 1 + 2 + 7 (validation)
└─ "Full report" → All phases, output Phase 8 format
```

---

## Performance Guidelines

1. **Batch property queries by type.** Query all walls, then all slabs —
   NEVER query one element at a time in a loop without batching.

2. **Use web-ifc `GetLineIDsWithType` for counting.** It returns a
   `Vector<number>` with a `.size()` method — NEVER load full entity data
   just to count elements.

3. **Limit `getData()` result sets.** For models with 10,000+ elements,
   ALWAYS filter via Classifier first. NEVER pass the entire model to
   `getData()`.

4. **Cache classification results.** `classifier.byCategory()` reads the
   entire model — call it once and reuse `classifier.list` across analysis
   steps.

5. **Use `GetRawLineData` for statistics.** When you only need type and ID
   (not full properties), `GetRawLineData` is faster than `GetLine`.

---

## Quick Reference

| Analysis Task | Primary API | Fallback API |
|---|---|---|
| Schema version | `ifcApi.GetModelSchema()` | Header line query |
| Type inventory | `ifcApi.GetAllTypesOfModel()` | `GetLineIDsWithType` per type |
| Element count by type | `classifier.list.get("Categories")` | `GetLineIDsWithType` |
| Spatial tree | `ifcApi.properties.getSpatialStructure()` | Classifier storeys |
| Storey element counts | `classifier.list.get("Storeys")` | Spatial tree traversal |
| Property sets | `fragments.getData(items)` | `ifcApi.properties.getPropertySets()` |
| Cross-classification | `classifier.find({...})` | `ItemsFinder.getItems()` |
| Orphan detection | Compare category vs storey sets | Spatial tree analysis |

## Related Skills

- `thatopen-syntax-properties` — Classifier API, getData, ItemsFinder details
- `thatopen-core-web-ifc` — Raw web-ifc query methods
- `thatopen-core-fragments` — FragmentsManager, ModelIdMap, worker setup
- `thatopen-syntax-ifc-loading` — IfcLoader, model loading prerequisites

## References

- [references/methods.md](references/methods.md) — Analysis APIs, classification queries, data extraction
- [references/examples.md](references/examples.md) — Model summary, property report, element inventory
- [references/anti-patterns.md](references/anti-patterns.md) — Inefficient queries, missing classification
