# Pascal Editor — Semantic Codebase Guide

*A conceptual map of the codebase from first principles to full end-to-end flow.*

---

## What this product is

Pascal is a **parametric 3D architectural editor** that runs in the browser. Users draw floor plans, add walls, doors, windows, slabs, staircases, roofs, furnishings, and zones, then see them rendered in real-time as a navigable 3D building. The same renderer works in a read-only viewer mode (no editor controls) for sharing and presentation.

The codebase is a TypeScript/React monorepo built with Bun and Turborepo. The renderer is React Three Fiber (R3F) over Three.js with a WebGPU renderer. State management is Zustand. Schemas are Zod.

---

## Mental model in one paragraph

Every element in a scene is a **node** — a plain, serialisable Zod-validated object stored in a flat dictionary. Nodes form a tree via `parentId` / `children` references (site → building → level → wall → door, etc.). When a node changes, it is added to a **dirty set**. On each animation frame, **systems** (pure Three.js components) read that set, update the live Three.js objects, and clear it. **Renderers** only mount geometry and wire pointer events — they never run logic. **Tools** in the editor translate user gestures into node mutations. The three-layer rule — core owns data, viewer owns rendering, editor owns interaction — enforces that the viewer works standalone without any editor code.

---

## Package map

```
packages/core     Pure logic — no Three.js, no UI
  schemas         Zod node types
  store           useScene (scene graph + CRUD + undo)
  systems         Wall mitering, stair opening sync, space detection
  events          Typed event bus (mitt)
  hooks           sceneRegistry, useSpatialQuery, useLiveTransforms

packages/viewer   3D canvas — no editor concepts
  <Viewer>        R3F Canvas, WebGPU renderer, lights, post-processing
  renderers/      One React component per node type (mount geometry)
  systems/        18 frame-loop components (update geometry each frame)
  store           useViewer (selection path, camera mode, display state)
  hooks           useNodeEvents (pointer → emitter)

packages/editor   Reusable editor UI
  <Editor>        Top-level shell: sidebar, viewer canvas, overlays
  store           useEditor (phase, mode, active tool, paint state)
  panels          Property inspector, catalog, scene tree
  tools           Registered tool components (injected into viewer)

apps/editor       Standalone Next.js app
  Composes packages/editor + viewer + core
  API routes, scene persistence, auth
```

Dependency direction: `apps/editor → packages/editor → packages/viewer → packages/core`. Nothing flows the other way.

---

## Domain model — the node tree

Every scene is a tree of typed nodes. The tree is stored flat in `useScene.nodes` (a `Record<id, AnyNode>`). Parent-child relationships are maintained in both directions: every node has `parentId` and every parent keeps a `children: string[]` array.

### Valid hierarchy

```
Site
└── Building
    └── Level
        ├── Wall
        │   ├── Door
        │   ├── Window
        │   └── Item  (wall-mounted fixture)
        ├── Fence
        ├── Column
        ├── Slab
        ├── Ceiling
        │   └── Item  (ceiling-mounted fixture)
        ├── Roof
        │   └── RoofSegment
        ├── Stair
        │   └── StairSegment
        ├── Zone      (named polygon region)
        ├── Item      (floor furniture / appliance)
        │   └── Item  (items stacked on surfaces)
        ├── Guide     (reference image for tracing)
        ├── Scan      (point cloud / 3D scan)
        └── Spawn     (VR player start point)
```

A `Site` also accepts top-level `Item` nodes (site furniture).

### Node type catalogue

| Type | Role | Key fields |
|---|---|---|
| `site` | Root container, defines property boundary | `polygon` (2D points), `children` |
| `building` | Groups stories; positioned in site space | `position`, `rotation`, `children` |
| `level` | One floor / storey | `level` (integer index), `children` |
| `wall` | Vertical divider; supports openings | `start`, `end` (2D), `thickness`, `height`, `curveOffset`, `interiorMaterial`, `exteriorMaterial`, `frontSide`, `backSide` |
| `door` | Parametric door / opening in wall | `doorType`, `width`, `height`, `operationState`, `segments`, `hingesSide`, `swingDirection`, `swingAngle`, and ~35 more fields |
| `window` | Parametric window / opening in wall | `windowType`, `width`, `height`, `operationState`, `columnRatios`, `rowRatios`, `openingShape`, and ~25 more fields |
| `slab` | Floor platform | `polygon`, `holes`, `elevation`, `autoFromWalls` |
| `ceiling` | Overhead plane | `polygon`, `holes`, `height`, `autoFromWalls` |
| `roof` | Groups roof segments; holds roof-level materials | `position`, `rotation`, `topMaterial`, `edgeMaterial`, `wallMaterial` |
| `roof-segment` | Single roof module | `roofType` (gable/hip/shed/…), `width`, `depth`, `wallHeight`, `roofHeight`, `overhang` |
| `stair` | Multi-segment staircase | `stairType` (straight/curved/spiral), `totalRise`, `stepCount`, `fromLevelId`, `toLevelId`, `slabOpeningMode` |
| `stair-segment` | Single flight or landing | `segmentType`, `width`, `length`, `height`, `stepCount`, `attachmentSide` |
| `column` | Parametric pillar with ~60 style parameters | `style`, `crossSection`, `height`, `radius`, `baseStyle`, `capitalStyle`, `shaftProfile`, … |
| `fence` | Decorative boundary segment | `start`, `end`, `height`, `thickness`, `style` (slat/rail/privacy) |
| `zone` | Named polygon region (rooms, labels) | `name`, `polygon`, `color` |
| `item` | Furniture, fixture, prop (GLB model) | `position`, `rotation`, `scale`, `asset` (src, dimensions, attachTo, interactive controls/effects), `children` |
| `guide` | Reference image for tracing | `url`, `position`, `scale`, `opacity`, `scaleReference` |
| `scan` | 3D point cloud or scan mesh | `url`, `position`, `scale`, `opacity` |
| `spawn` | VR/game player start location | `position`, `rotation` |

### BaseNode — fields on every node

```ts
{
  object:   'node'               // always
  id:       '<type>_<nanoid16>'  // e.g. "wall_4f8a2b..."
  type:     '<type>'             // discriminator
  name?:    string
  parentId: string | null
  visible:  boolean              // default true
  camera?:  CameraSchema
  metadata: Record<string, unknown>
}
```

IDs are generated by `objectId(prefix)` and `generateId(prefix)` in `packages/core/src/schema/base.ts`. Never hardcode them.

---

## The scene store (`useScene`)

**File:** `packages/core/src/store/use-scene.ts`

The single source of truth for all scene data. It is a Zustand store with Zundo temporal middleware for undo/redo.

### State shape

```ts
{
  nodes:       Record<AnyNodeId, AnyNode>   // flat dictionary of all nodes
  rootNodeIds: AnyNodeId[]                  // top-level nodes (sites)
  dirtyNodes:  Set<AnyNodeId>               // nodes queued for system update
  collections: Record<CollectionId, Collection>
  readOnly:    boolean
}
```

### Core actions

| Action | What it does |
|---|---|
| `createNode(node, parentId?)` | Adds node to `nodes`, appends id to parent's `children`, marks node + parent dirty |
| `updateNode(id, patch)` | Merges patch into node, handles reparenting, marks dirty |
| `deleteNode(id)` | Recursively collects descendants, removes from parent's children, wall-merges collinear neighbours, marks dirty |
| `markDirty(id)` / `clearDirty(id)` | Add / remove from `dirtyNodes` set |
| `setScene(nodes, rootIds)` | Load a serialised scene: runs migrations then validates (removes orphans and unreachable nodes) |
| `createCollection` / `addToCollection` etc. | Manage named groups of nodes |

### Undo/redo

Zundo wraps the store. Every `updateNode` / `createNode` / `deleteNode` call is captured as a state snapshot. `useScene.temporal.getState()` gives `{ pastStates, futureStates }`. On undo/redo, the store diffs the restored snapshot against current state and automatically re-marks changed nodes as dirty so systems re-run.

### Schema migrations

`setScene` runs migrations before loading:
- Old single `material` on walls → split to `interiorMaterial` / `exteriorMaterial`
- Old single `material` on stairs → `railingMaterial`, `treadMaterial`, `sideMaterial`
- Old roof format (no children) → wrapped in `roof` + `roof-segment`
- Orphan / unreachable node cleanup

---

## The rendering pipeline

Three.js objects are created and updated through a two-component pattern: **Renderer** (mounts, wires events) + **System** (updates geometry each frame).

### Renderer

One `.tsx` file per node type in `packages/viewer/src/components/renderers/<type>/`.

What a renderer does:
1. Reads its node from `useScene`
2. Creates a `<group>` at the node's position
3. Calls `useRegistry(node.id, 'type', ref)` — registers the live `THREE.Object3D` in `sceneRegistry`
4. Spreads `useNodeEvents(node, 'type')` onto the clickable mesh — this wires pointer events to the event bus
5. Renders child node IDs via `<NodeRenderer nodeId={childId} />`

What a renderer must NOT do:
- Run geometry generation or heavy computation (that belongs in the system)
- Import from `apps/editor` or reference editor state
- Manage selection directly

**Dispatch** — `NodeRenderer` (`packages/viewer/src/components/renderers/node-renderer.tsx`) reads a node from the store and switches on `node.type` to render the right renderer:

```tsx
{node.type === 'wall'  && <WallRenderer node={node} />}
{node.type === 'item'  && <ItemRenderer node={node} />}
// …one branch per type
```

### System

One directory per node type in `packages/viewer/src/systems/<type>/`. Systems are React components that return `null` and run geometry updates in `useFrame`.

```tsx
export const WallSystem = () => {
  const dirtyNodes = useScene(s => s.dirtyNodes)
  const clearDirty  = useScene(s => s.clearDirty)

  useFrame(() => {
    if (dirtyNodes.size === 0) return
    dirtyNodes.forEach(id => {
      const node = useScene.getState().nodes[id]
      if (node?.type !== 'wall') return
      const mesh = sceneRegistry.nodes.get(id)
      if (!mesh) return
      // recompute geometry, CSG cuts, materials…
      clearDirty(id)
    })
  }, 2)   // execution order 2

  return null
}
```

Systems only process nodes in `dirtyNodes` — they are never exhaustive per-frame.

### Systems in the viewer

All 18 systems are mounted inside `<Viewer>` before any renderers:

| System | Responsibility |
|---|---|
| `LevelSystem` | Positions levels vertically (stacked / exploded / solo modes) |
| `WallSystem` | CSG boolean cuts for doors/windows, mitering at junctions, material assignment |
| `SlabSystem` | Triangulates floor polygon, applies hole cutouts |
| `CeilingSystem` | Same as slab but overhead |
| `RoofSystem` | Generates hip/gable/shed/mansard geometry from segment params |
| `StairSystem` | Treads + risers + railings for straight/curved/spiral stairs |
| `DoorSystem` | Parametric door leaf + frame, glass panels, handles |
| `WindowSystem` | Parametric frame + panes + sill |
| `FenceSystem` | Posts + rails + slats along a curve |
| `ItemSystem` | Elevation snapping (slab height), wall-side Z offset |
| `ItemLightSystem` | Point light pool for interactive lamp effects |
| `ZoneSystem` | Zone fill polygons on `ZONE_LAYER` |
| `GuideSystem` | Reference image plane |
| `ScanSystem` | Point cloud mesh |
| `DoorAnimationSystem` | Lerps door `swingAngle` / `operationState` over time |
| `WindowAnimationSystem` | Lerps window `operationState` |
| `LevelSystem` | vertical level positioning |
| `InteractiveSystem` | Responds to item interactive control state changes |

### `sceneRegistry`

**File:** `packages/core/src/hooks/scene-registry/scene-registry.ts`

A global mutable map that bridges the Zustand world (node IDs) and the Three.js world (live `Object3D` instances):

```ts
sceneRegistry.nodes              // Map<id, THREE.Object3D>
sceneRegistry.byType.wall        // Set<string> — all wall IDs currently rendered
sceneRegistry.nodes.get(wallId)  // → THREE.Object3D (O(1))
```

Renderers register via `useRegistry(id, type, ref)` on mount and auto-deregister on unmount. Systems and tools use `sceneRegistry` for direct mesh access without subscribing to React re-renders.

---

## The event system

**File:** `packages/core/src/events/bus.ts`

A typed mitt emitter. Every event has a namespaced key.

### Node events

Format: `<type>:<suffix>` — e.g. `wall:click`, `item:enter`, `door:double-click`.

Suffixes: `click`, `move`, `enter`, `leave`, `pointerdown`, `pointerup`, `context-menu`, `double-click`.

`useNodeEvents(node, type)` (in `packages/viewer/src/hooks/use-node-events.ts`) returns R3F event handlers (`onPointerDown`, `onPointerUp`, etc.) that translate R3F pointer events into typed `NodeEvent` payloads and emit them on the bus.

### Grid events

Format: `grid:<suffix>` — raycasted against the ground plane when the pointer hits empty space. Used by placement tools to get world-space coordinates for new node creation.

### NodeEvent payload

```ts
{
  node:          AnyNode
  position:      [x, y, z]       // world space
  localPosition: [x, y, z]       // building-local space
  normal?:       [x, y, z]
  stopPropagation: () => void
  nativeEvent:   ThreeEvent<PointerEvent>
}
```

### Rules

- **Renderers only emit**, never listen.
- **Tools and selection managers listen**, never emit directly to `emitter`.
- Always clean up with the same function reference: `emitter.off('wall:click', fn)`.

---

## The viewer store (`useViewer`)

**File:** `packages/viewer/src/store/use-viewer.ts`

Presentation state that `<Viewer>` owns. Never contains editor-specific concepts.

| Field | Type | Purpose |
|---|---|---|
| `selection` | `{ buildingId, levelId, zoneId, selectedIds[] }` | Hierarchical selection path |
| `previewSelection` | same shape | Hover preview |
| `hoveredObjectId` | `string \| null` | Cursor highlight |
| `cameraMode` | `'perspective' \| 'orthographic'` | Camera projection |
| `levelMode` | `'stacked' \| 'exploded' \| 'solo' \| 'manual'` | How storeys are arranged |
| `wallMode` | `'up' \| 'cutaway' \| 'down'` | Wall display height |
| `theme` | `'light' \| 'dark'` | Viewer colour theme |
| `unit` | `'metric' \| 'imperial'` | Measurement display |
| `showScans` | `boolean` | Point cloud visibility |
| `showGuides` | `boolean` | Reference image visibility |
| `showGrid` | `boolean` | Ground grid visibility |
| `cameraDragging` | `boolean` | Suppresses pointer events during orbit |
| `outliner` | `{ selectedObjects, hoveredObjects }` | Live `THREE.Object3D[]` arrays for post-processing outline passes |

Persisted fields (localStorage, per-project): `cameraMode`, `theme`, `unit`, `levelMode`, `wallMode`, project-specific preferences.

---

## The editor store (`useEditor`)

**File:** `packages/editor/src/store/use-editor.tsx`

Editor-only interaction state. Never leaks into `packages/viewer`.

### Phase system

The editor has three top-level phases that gate what the user can interact with:

| Phase | What is selectable / buildable |
|---|---|
| `site` | Site boundary, buildings |
| `structure` | Walls, slabs, ceilings, roofs, stairs, zones, columns, fences |
| `furnish` | Items (furniture, fixtures, appliances) |

### Mode system

Within a phase, a mode describes the current interaction intent:

| Mode | Meaning |
|---|---|
| `select` | Default — click to select, properties in sidebar |
| `build` | Active tool placement (drawing a wall, placing an item, etc.) |
| `edit` | Editing geometry of a selected node (wall endpoint, polygon vertex, etc.) |
| `delete` | Click to delete |
| `material-paint` | Click a surface to apply the active material |

### Active tool

The tool is a string key that maps to a registered tool component (e.g. `'wall'`, `'door'`, `'item'`, `'property-line'`). `ToolManager` (in the editor's viewer children) mounts the active tool component.

### Key fields

```ts
{
  phase:          'site' | 'structure' | 'furnish'
  mode:           'select' | 'build' | 'edit' | 'delete' | 'material-paint'
  activeTool:     string | null
  viewMode:       '3d' | '2d' | 'split'
  activePanel:    string | null           // sidebar panel id
  moving:         boolean                 // node drag in progress
  paintMaterial:  MaterialPreset | null
  paintTarget:    AnyNodeId | null
}
```

---

## Selection — two managers

**Viewer SelectionManager** (`packages/viewer/src/components/viewer/selection-manager.tsx`)
- Listens to `node:click` events on the bus
- Implements hierarchical drill-down: click a wall when building not selected → select building; click again → select level; click again → select wall
- Stores result in `useViewer.selection`
- Syncs `useViewer.outliner` for post-processing outline

**Editor SelectionManager** (`packages/editor/src/components/…/selection-manager.tsx`)
- Overrides the viewer's manager when the editor is active
- Phase-aware: in `furnish` phase, clicking a wall auto-switches to `structure` phase then selects the wall
- Handles multi-select (Ctrl/Meta), box-select, double-click drill-in

The editor's manager replaces the viewer's via a prop — the viewer is always passive. The selection path in `useViewer.selection` remains the single source of truth.

---

## Tools — how user gestures become scene mutations

Tools live exclusively in `apps/editor` (or `packages/editor` for reusable ones). They are React components that return `null` — pure side-effect logic.

### Lifecycle

1. User picks a tool in the toolbar → `useEditor.setActiveTool('wall')` + `setMode('build')`
2. `ToolManager` mounts `<WallTool />` (or whichever tool)
3. Tool registers listeners on `emitter` for `grid:click`, `grid:move`, `wall:click`, etc.
4. On pointer move: tool may use `useLiveTransforms` to show a preview mesh (ephemeral, not undo-captured)
5. On commit (e.g. click to place): tool calls `useScene.getState().createNode(node, parentId)` — undo-captured
6. On tool deactivate / unmount: tool removes all emitter listeners, clears live transforms

### The live-drag exception

During an active drag, a tool may directly mutate `sceneRegistry.nodes.get(id)` in-place to move a mesh without React re-renders. This is allowed **only** when the same transform is mirrored in `useLiveTransforms`. On drag end, `updateNode` commits the final position to `useScene`.

### Spatial queries

Before creating or placing a node, tools validate placement:

```ts
const { canPlaceOnFloor, canPlaceOnWall, canPlaceOnCeiling } = useSpatialQuery()

const { valid, conflictIds } = canPlaceOnFloor(levelId, position, dimensions, rotation, [item.id])
const { valid, adjustedY }   = canPlaceOnWall(levelId, wallId, localX, localY, dims, 'wall')
```

Always pass `[item.id]` as `ignoreIds` when checking placement of an existing node being moved. Always use `adjustedY` from `canPlaceOnWall` for the vertical snap position.

---

## End-to-end flow: drawing a wall

1. User clicks "Wall" in the toolbar
   → `useEditor.setActiveTool('wall')`, `setMode('build')`
   → `ToolManager` mounts `<WallTool />`

2. User moves mouse over canvas
   → R3F fires pointer event on the invisible ground plane
   → `useNodeEvents` emits `grid:move` with world position
   → `WallTool` listener updates `useLiveTransforms` with preview wall endpoints
   → A preview mesh appears (ephemeral, not in `useScene`)

3. User clicks to set start point, then end point
   → `WallTool` listener for `grid:click` fires
   → Tool calls `WallNode.parse({ start, end, parentId: levelId })`
   → Tool calls `useScene.getState().createNode(wall, levelId)`
   → `createNode` adds wall to `nodes`, appends id to level's `children`, marks wall + level dirty

4. Next animation frame
   → `WallSystem.useFrame` fires, finds wall in `dirtyNodes`
   → Fetches miter data for adjacent walls, builds 3D geometry, applies CSG (none yet — no doors)
   → Writes geometry to the `THREE.Mesh` held in `sceneRegistry.nodes.get(wallId)`
   → Calls `clearDirty(wallId)`

5. React reconciliation (separate from frame loop)
   → `useScene(s => s.nodes[levelId])` subscription fires on the Level renderer
   → Level renderer re-renders its `children` list, finds the new wall ID
   → `<NodeRenderer nodeId={wallId} />` mounts → `<WallRenderer node={wall} />`
   → Renderer creates a `<group>` ref, calls `useRegistry(wall.id, 'wall', ref)`
   → `sceneRegistry.nodes.set(wallId, groupRef.current)`

6. Subsequent frame
   → `WallSystem` re-runs (wall still in `dirtyNodes` until geometry is applied and `clearDirty` called)
   → Wall mesh now exists in registry; system writes final geometry

7. User clicks the wall later
   → R3F fires `onPointerUp` on the wall mesh
   → `useNodeEvents` emits `wall:click` on the bus
   → `SelectionManager` listener catches it, calls `useViewer.setSelection({ …, selectedIds: [wallId] })`
   → Outliner gets the live `Object3D`, post-processing outline renders around it

---

## End-to-end flow: placing a door on a wall

1. User selects a wall, activates "Door" tool
   → `useEditor.setActiveTool('door')`, `setMode('build')`

2. User moves mouse over the wall
   → R3F fires `onPointerMove` on wall mesh
   → `useNodeEvents` emits `wall:move` with `localPosition` (wall-local coords)
   → `DoorTool` computes snap position along wall, updates `useLiveTransforms` preview

3. User clicks wall
   → `DoorTool` receives `wall:click` event
   → Validates placement: `canPlaceOnWall(levelId, wallId, localX, localY, dims, 'wall')`
   → Calls `DoorNode.parse({ wallId, position, width, height })` then `createNode(door, wallId)`
   → Door added under wall in node tree; wall + door marked dirty

4. Next frame — `WallSystem` processes dirty wall
   → Finds door in `wall.children`, computes door cutout geometry
   → CSG-subtracts door opening from wall mesh
   → Writes updated wall geometry to registry

5. React reconciliation
   → `WallRenderer` re-renders children, finds new door ID
   → `<DoorRenderer node={door} />` mounts
   → `DoorSystem` processes dirty door → builds parametric door leaf geometry

---

## Material system

Materials are referenced on node schemas as string presets (`materialPreset: 'brick'`) or as inline `MaterialSchema` objects. The viewer resolves presets via `getMaterialPresetByRef(preset)` from the material catalog in `packages/viewer/src/lib/materials.ts`.

Shared singleton materials (`baseMaterial`, `glassMaterial`) are reused across all renderers to minimise GPU draw calls. WebGPU `MeshStandardNodeMaterial` is used throughout for TSL shader node compatibility.

---

## Packages in detail

### `packages/core` — what it exports

- All node schemas and their TypeScript types
- `useScene` — scene graph store
- `useLiveTransforms` — ephemeral drag store  
- `useInteractive` — door/window animation state
- `sceneRegistry` + `useRegistry`
- `emitter` + all event types
- `useSpatialQuery`, `spatialGridManager`
- Wall utilities: `getWallThickness`, `getWallPlanFootprint`, `calculateLevelMiters`
- Stair utilities: `syncAutoStairOpenings`
- Graph utilities: `cloneSceneGraph`, `forkSceneGraph`, `cloneLevelSubtree`
- History control: `pauseSceneHistory`, `resumeSceneHistory`, `clearSceneHistory`

### `packages/viewer` — what it exports

- `<Viewer>` component — accepts `children` (editor injects tools/systems here)
- `useViewer` store
- `SCENE_LAYER`, `EDITOR_LAYER`, `ZONE_LAYER` constants
- `useNodeEvents` hook
- Material helpers

### `packages/editor` — what it exports

- `<Editor>` component — full editing shell
- `useEditor` store
- Sidebar panels (properties, catalog, scene tree, materials)
- `SceneLoader` — loads and saves scenes
- Command palette hooks, audio/preset helpers

---

## Three.js layer assignments

| Layer | Value | Owner | Used for |
|---|---|---|---|
| `SCENE_LAYER` | 0 | `packages/viewer` | All scene geometry (walls, items, slabs…) |
| `EDITOR_LAYER` | 1 | `apps/editor` | Editor helpers: grid, tool previews, cursor sphere, measurement guides |
| `ZONE_LAYER` | 2 | `packages/viewer` | Semi-transparent zone fills composited separately |

Import constants from `@pascal-app/viewer`. Never hardcode the integer values.

---

## Key invariants to internalise

1. **Flat + bidirectional**: `nodes` is a flat dict; `parentId` and `children` are always kept in sync by `createNode` / `deleteNode` actions.

2. **Dirty → system → clear**: Nothing updates Three.js objects directly except systems. When any code wants a mesh to update, it calls `markDirty(id)` or `useScene.getState().dirtyNodes.add(id)`. Systems clear dirty flags after processing.

3. **Renderers never compute**: A renderer's job is to mount geometry and register with `sceneRegistry`. All computation (CSG, extrusion, triangulation) happens in systems.

4. **Viewer is standalone**: Nothing in `packages/viewer` knows about tools, phases, modes, or editor UI. Editor features are injected as `children` of `<Viewer>`.

5. **Events flow one way**: renderers → emitter → tools/selection managers. Never the reverse.

6. **Tools write to useScene (undo) or useLiveTransforms (ephemeral)**: Never both for the same value, never directly to `useViewer` or `useEditor` for geometry state.

7. **sceneRegistry is live, not reactive**: It is a plain `Map`, not Zustand. Code that reads from it gets the current Three.js object but does not subscribe to changes.

8. **Undo diffs itself**: When Zundo restores a past snapshot, `useScene` compares it to the current `nodes` dict and marks all changed nodes dirty automatically.
