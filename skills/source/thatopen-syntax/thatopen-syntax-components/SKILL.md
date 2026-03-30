---
name: thatopen-syntax-components
description: >
  Use when accessing ThatOpen components, implementing custom components,
  or working with the component lifecycle and event system.
  Prevents direct instantiation of components instead of using get().
  Covers components.get() singleton pattern, component registration, lifecycle
  interface implementation, Event system, DataMap, DataSet reactive collections.
  Keywords: components, get, singleton, uuid, lifecycle, event, datamap, dataset, disposable, updateable, configurable, component api, how to use components, register component.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# ThatOpen Component Syntax

## Purpose

This skill covers the **syntax and usage patterns** for ThatOpen's component
system. It provides exact method signatures, lifecycle interface contracts,
event system API, and reactive collection patterns. For architectural overview
and design rationale, see `thatopen-core-architecture`.

**Version**: @thatopen/components 3.3.x

## The components.get() Pattern

ALWAYS obtain component instances through the singleton registry:

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();

// ALWAYS use get() — lazy singleton creation
const worlds = components.get(OBC.Worlds);
const ifcLoader = components.get(OBC.IfcLoader);
const fragments = components.get(OBC.FragmentsManager);
```

**How it works:**
1. First call: `get()` invokes `new ComponentClass(components)` internally.
2. The constructor calls `components.add(ComponentClass.uuid, this)` to
   register the instance.
3. Subsequent calls: `get()` returns the cached instance from `components.list`.

NEVER call `new OBC.Worlds(components)` or any component constructor directly.
This bypasses the registry and creates untracked duplicate instances.

## Components Container API

```typescript
class Components implements Disposable {
  readonly list: DataMap<string, Component>;
  enabled: boolean;
  onDisposed: Event<void>;
  onInit: Event<undefined>;
  static release: string;

  get<U extends Component>(Ctor: new (c: Components) => U): U;
  add(uuid: string, instance: Component): void;
  init(): void;
  dispose(): void;
}
```

### init()

ALWAYS call `components.init()` after setting up your world. This starts
the `requestAnimationFrame` loop using `THREE.Clock` for delta time.
Without it, nothing renders and no `Updateable` components receive updates.

### dispose()

ALWAYS call `components.dispose()` on cleanup. This iterates all registered
components and disposes each one. `FragmentsManager` is ALWAYS disposed last
to prevent dangling references. After `dispose()`, NEVER use any component
references — they are invalidated.

## Component Base Class

Every ThatOpen component extends this hierarchy:

```typescript
abstract class Base {
  constructor(public components: Components) {}

  // Runtime interface detection (duck-typing, NOT instanceof)
  isDisposeable(): this is Disposable;
  isUpdateable(): this is Updateable;
  isConfigurable(): this is Configurable<any, any>;
  isResizeable(): this is Resizeable;
  isHideable(): this is Hideable;
  isSerializable(): this is Serializable<any>;
}

abstract class Component extends Base {
  static readonly uuid: string;   // MUST be unique across all components
  abstract enabled: boolean;      // MUST be implemented by every component
}
```

## Creating Custom Components

Every custom component MUST follow this exact pattern:

```typescript
import * as OBC from "@thatopen/components";

class MyTool extends OBC.Component implements OBC.Disposable {
  // 1. Static UUID — MUST be unique, use a real UUID generator
  static readonly uuid = "a1b2c3d4-e5f6-7890-abcd-ef1234567890" as const;

  // 2. Enabled flag — required by Component
  enabled = true;

  // 3. Disposable event — required by Disposable interface
  onDisposed = new OBC.Event<void>();

  constructor(components: OBC.Components) {
    super(components);
    // 4. Self-register — ALWAYS call add() in the constructor
    components.add(MyTool.uuid, this);
  }

  dispose(): void {
    this.enabled = false;
    // 5. Trigger disposal event, then reset to clear handlers
    this.onDisposed.trigger();
    this.onDisposed.reset();
  }
}

// 6. Access via get() — NEVER via new
const myTool = components.get(MyTool);
```

**Checklist for custom components:**
- Static `uuid` with `as const` assertion
- `enabled` property initialized
- Constructor calls `super(components)` then `components.add()`
- Implements `Disposable` if it holds any resources
- `dispose()` sets `enabled = false`, triggers `onDisposed`, resets events

## Lifecycle Interfaces

Components opt into behaviors by implementing interfaces. This is a mixin
pattern — no deep inheritance hierarchies.

### Disposable

ALWAYS implement if the component holds resources (event handlers, GPU
buffers, DOM references, subscriptions).

```typescript
interface Disposable {
  dispose(): void;
  onDisposed: Event<void>;
}
```

**Implementation pattern:**
```typescript
dispose(): void {
  this.enabled = false;
  // Clean up resources: remove DOM elements, clear maps, etc.
  this.someMap.clear();
  this.someDomElement?.remove();
  // Trigger event, then reset ALL events owned by this component
  this.onDisposed.trigger();
  this.onDisposed.reset();
  this.onSomeEvent.reset();
}
```

### Updateable

Implement when the component needs per-frame updates. The `Components`
animation loop calls `update(delta)` on EVERY enabled `Updateable`
component each frame.

```typescript
interface Updateable {
  update(delta?: number): void;
  onBeforeUpdate: Event<any>;
  onAfterUpdate: Event<any>;
}
```

**Implementation pattern:**
```typescript
class AnimationController extends OBC.Component
  implements OBC.Updateable, OBC.Disposable {

  static readonly uuid = "..." as const;
  enabled = true;

  onBeforeUpdate = new OBC.Event<void>();
  onAfterUpdate = new OBC.Event<void>();
  onDisposed = new OBC.Event<void>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(AnimationController.uuid, this);
  }

  // Called automatically every frame when enabled=true
  update(delta?: number): void {
    this.onBeforeUpdate.trigger();
    // Per-frame logic using delta (seconds since last frame)
    this.onAfterUpdate.trigger();
  }

  dispose(): void {
    this.enabled = false;
    this.onDisposed.trigger();
    this.onDisposed.reset();
    this.onBeforeUpdate.reset();
    this.onAfterUpdate.reset();
  }
}
```

### Configurable

Implement when the component requires deferred or async initialization.
ALWAYS call `setup()` before using a `Configurable` component.

```typescript
interface Configurable<TConfig, TPartialConfig> {
  setup(config?: TPartialConfig): void;
  config: TConfig;
  isSetup: boolean;
  onSetup: Event<any>;
}
```

**Implementation pattern:**
```typescript
class MyConfigurable extends OBC.Component
  implements OBC.Configurable<MyConfig, Partial<MyConfig>>, OBC.Disposable {

  static readonly uuid = "..." as const;
  enabled = true;
  isSetup = false;

  config: MyConfig = { /* defaults */ };
  onSetup = new OBC.Event<MyConfigurable>();
  onDisposed = new OBC.Event<void>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(MyConfigurable.uuid, this);
  }

  setup(config?: Partial<MyConfig>): void {
    if (config) {
      this.config = { ...this.config, ...config };
    }
    // Perform initialization...
    this.isSetup = true;
    this.onSetup.trigger(this);
  }

  dispose(): void {
    this.enabled = false;
    this.isSetup = false;
    this.onDisposed.trigger();
    this.onDisposed.reset();
    this.onSetup.reset();
  }
}
```

ALWAYS check `isSetup` before calling methods that depend on configuration:

```typescript
const myComp = components.get(MyConfigurable);
if (!myComp.isSetup) {
  await myComp.setup({ /* config */ });
}
```

### Other Interfaces

| Interface       | Contract                                               |
|-----------------|--------------------------------------------------------|
| `Resizeable`    | `resize(size?)`, `getSize()`, `onResize: Event`       |
| `Hideable`      | `visible: boolean`                                     |
| `Createable`    | `create()`, `delete()`, `endCreation()`, `cancelCreation()` |
| `Serializable`  | `import(data)`, `export(): data`                       |

## Event\<T\> System

ThatOpen uses a custom pub/sub event class throughout the entire API.

```typescript
class Event<T> {
  enabled: boolean;          // default: true

  add(handler: (data: T) => void): void;     // subscribe
  remove(handler: (data: T) => void): void;  // unsubscribe
  trigger(data?: T): void;                   // fire event
  reset(): void;                             // remove ALL handlers
}
```

### Usage Rules

1. **ALWAYS store handler references** when you need to remove them later.
   Anonymous arrow functions cannot be removed.

2. **ALWAYS call `reset()` on all owned events** in your `dispose()` method
   to prevent memory leaks.

3. **Use `enabled = false`** to temporarily suppress an event without
   removing handlers. Set back to `true` to resume.

4. **NEVER assume event ordering** — handlers fire in registration order,
   but do not depend on this for correctness.

### Quick Reference

```typescript
// Store reference — NEVER use anonymous functions if you need to remove
const onLoaded = (model: OBC.FragmentsModel) => { /* ... */ };
fragments.onFragmentsLoaded.add(onLoaded);      // subscribe
fragments.onFragmentsLoaded.remove(onLoaded);    // unsubscribe
fragments.onFragmentsLoaded.enabled = false;     // suppress temporarily
fragments.onFragmentsLoaded.enabled = true;      // re-enable
fragments.onFragmentsLoaded.reset();             // remove ALL handlers
```

See [references/examples.md](references/examples.md) for detailed patterns.

## DataMap\<K, V\>: Reactive Map

Extends the standard `Map` with event hooks. Used throughout ThatOpen
for observable collections (e.g., `Components.list`, `Worlds.list`,
`Classifier.list`).

```typescript
class DataMap<K, V> extends Map<K, V> {
  onItemSet: Event<{ key: K; value: V }>;
  onItemUpdated: Event<{ key: K; value: V }>;
  onItemDeleted: Event<{ key: K }>;
  onCleared: Event<void>;
}
```

All standard `Map` methods work (`get`, `set`, `delete`, `has`, `forEach`,
`entries`, `keys`, `values`, `size`). The events fire automatically when
the corresponding operations occur.

### Reacting to DataMap Changes

```typescript
const worlds = components.get(OBC.Worlds);

// React when a new world is added
worlds.list.onItemSet.add(({ key, value }) => {
  console.log(`World added: ${key}`);
});

// React when a world is updated
worlds.list.onItemUpdated.add(({ key, value }) => {
  console.log(`World updated: ${key}`);
});

// React when a world is removed
worlds.list.onItemDeleted.add(({ key }) => {
  console.log(`World removed: ${key}`);
});

// React when all worlds are cleared
worlds.list.onCleared.add(() => {
  console.log("All worlds cleared");
});
```

## DataSet\<T\>: Reactive Set

Extends the standard `Set` with event hooks. Used for collections like
`world.meshes`, measurement lists, and style sets.

```typescript
class DataSet<T> extends Set<T> {
  onItemAdded: Event<T>;
  onItemDeleted: Event<T>;
  onCleared: Event<void>;
}
```

All standard `Set` methods work (`add`, `delete`, `has`, `forEach`,
`entries`, `values`, `size`). Events fire automatically.

### Reacting to DataSet Changes

```typescript
// Track when meshes are added to a world
world.meshes.onItemAdded.add((mesh) => {
  console.log("Mesh added:", mesh.name);
});

world.meshes.onItemDeleted.add((mesh) => {
  console.log("Mesh removed:", mesh.name);
});
```

## Components Lifecycle Flow

```
new Components()
    |
    v
components.get(X)  ──>  new X(components)  ──>  components.add(uuid, instance)
    |                         |
    v                         v
[setup() if Configurable]   registered in components.list
    |
    v
components.init()  ──>  starts requestAnimationFrame loop
    |                         |
    v                         v
update(delta) called    on ALL enabled Updateable components each frame
    |
    v
components.dispose()  ──>  dispose() on ALL components
                           FragmentsManager disposed LAST
```

## Runtime Interface Detection

Use the `is*()` methods on `Base` for runtime type checking:

```typescript
const component = components.list.get(someUuid);

if (component?.isDisposeable()) {
  component.dispose(); // TypeScript narrows type to Disposable
}

if (component?.isUpdateable()) {
  component.update(0.016); // TypeScript narrows type to Updateable
}

if (component?.isConfigurable()) {
  if (!component.isSetup) {
    component.setup();
  }
}
```

These use duck-typing (checking for method existence), NOT `instanceof`.

## Critical Rules

1. **ALWAYS** use `components.get(ComponentClass)` to obtain instances.
   NEVER use `new ComponentClass(components)` directly.
2. **ALWAYS** implement `Disposable` if your component holds any resources.
3. **ALWAYS** call `components.add(uuid, this)` in your component constructor.
4. **ALWAYS** define a static `uuid` with `as const` on custom components.
5. **ALWAYS** call `reset()` on all owned events in `dispose()`.
6. **ALWAYS** store event handler references for later removal.
7. **ALWAYS** call `setup()` on `Configurable` components before using them.
8. **ALWAYS** call `components.init()` after world setup.
9. **NEVER** use components after `components.dispose()` has been called.
10. **NEVER** use anonymous functions as event handlers if you need to remove
    them later.

## Reference Files

- [references/methods.md](references/methods.md) — Complete API signatures
  for Event, DataMap, DataSet, lifecycle interfaces, and Base/Component
- [references/examples.md](references/examples.md) — Custom component,
  lifecycle implementation, and event patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Direct
  instantiation, missing disposal, event leaks, and other common mistakes

## Source Verification

All API signatures verified against:
- GitHub: `ThatOpen/engine_components` main branch (packages/core/src/)
- npm: `@thatopen/components@3.3.3`
- Research: `docs/research/vooronderzoek-thatopen.md`
