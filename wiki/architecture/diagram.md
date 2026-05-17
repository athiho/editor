# Architecture Diagram

*Visual maps of the three-layer architecture, data-flow, and node-type lifecycle.*

Applies to: `packages/**` `apps/editor/**`

---

## Layer dependency graph

Arrows mean "imports from". `packages/core` sits at the base — consumed by every layer above it. Three.js is a peer dependency of `core` (type-only) and a real dependency of `viewer` and the editor app.

```mermaid
graph TD
  APP[apps/editor]
  PKG_ED[packages/editor]
  PKG_VW[packages/viewer]
  PKG_CO[packages/core]
  THREE[three.js / react-three-fiber]

  APP --> PKG_ED
  APP --> PKG_VW
  APP --> PKG_CO
  PKG_ED --> PKG_VW
  PKG_ED --> PKG_CO
  PKG_VW --> PKG_CO
  PKG_CO -.->|peerDep — types only| THREE
  PKG_VW --> THREE
  APP --> THREE
```

**Forbidden arrows (layer violations):**

| From | To | Rule |
|---|---|---|
| `packages/core` | `packages/viewer` | core must not know about Three.js rendering |
| `packages/viewer` | `apps/editor` | viewer must stay editor-agnostic |
| `packages/core` | `apps/editor` | same — core is a standalone library |

See `wiki/architecture/layers.md` and `wiki/architecture/viewer-isolation.md`.

---

## Data-flow — the frame loop

A single mutation travels through four stages on each animation frame.

```mermaid
sequenceDiagram
  participant Tool as Tool<br/>(apps/editor)
  participant Store as useScene<br/>(packages/core)
  participant System as Viewer System<br/>(packages/viewer)
  participant Reg as sceneRegistry<br/>(packages/core)
  participant Renderer as Renderer<br/>(packages/viewer)
  participant Bus as Event Bus / emitter<br/>(packages/core)

  Note over Tool,Store: Committed mutation (undo-captured)
  Tool->>Store: updateNode(id, patch)
  Store->>Store: dirtyNodes.add(id)

  Note over System,Reg: useFrame — runs every frame
  System->>Store: reads dirtyNodes
  System->>Reg: sceneRegistry.nodes.get(id)
  System->>Reg: mutate Object3D in-place
  System->>Store: clearDirty(id)

  Note over Renderer,Bus: Pointer events — R3F → emitter
  Renderer->>Bus: emitter.emit('wall:click', NodeEvent)
  Bus->>Tool: listener callback (e.g. selection, tool action)
```

**Key invariants:**

- Renderers only **emit**, never listen to the event bus.
- Systems only **read and mutate** scene registry; they never emit events or mutate `useScene` state.
- Tools only **write** to `useScene` (committed) or `useLiveTransforms` (ephemeral drag). They never import from `packages/viewer`.

---

## Node-type lifecycle — adding a new 3D object

Every new node type requires changes in all three layers, in this order:

```mermaid
flowchart LR
  S1["① Schema\npackages/core/src/schema/nodes/<type>.ts"]
  S2["② Register in AnyNode union\npackages/core/src/schema/types.ts"]
  S3["③ Event types\npackages/core/src/events/bus.ts"]
  S4["④ sceneRegistry.byType\npackages/core/src/hooks/scene-registry/scene-registry.ts"]
  S5["⑤ Renderer\npackages/viewer/src/components/renderers/<type>/<type>-renderer.tsx"]
  S6["⑥ Viewer System\npackages/viewer/src/systems/<type>/<type>-system.tsx"]
  S7["⑦ NodeRenderer dispatch\npackages/viewer/src/components/renderers/node-renderer.tsx"]
  S8["⑧ useNodeEvents NodeConfig\npackages/viewer/src/hooks/use-node-events.ts"]
  S9["⑨ Editor Tool (optional)\napps/editor/components/tools/<type>-tool.tsx"]

  S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8 --> S9
```

Use `.agents/skills/build-3d-feature/SKILL.md` for the executable step-by-step checklist.

---

## Store ownership

Three Zustand stores, one per layer. No cross-store coupling inside store actions.

```mermaid
graph LR
  subgraph core["packages/core"]
    SC[useScene\nscene graph · undo/redo · dirtyNodes]
    LT[useLiveTransforms\nephemeral drag state]
    SI[useInteractive\nanimation & control state]
  end
  subgraph viewer["packages/viewer"]
    VW[useViewer\nselection path · camera mode\nlevel mode · theme · display toggles]
  end
  subgraph editor["apps/editor (private)"]
    ED[useEditor\nactive tool · phase · edit mode\npanel states · paint preview]
  end

  core --- viewer
  viewer --- editor
```

**Rule:** Any state that would be needed by the read-only `/viewer` route belongs in `useViewer` or `useScene`. State that is only needed while editing belongs in `useEditor`.

---

## Three.js layer assignments

Three rendering layers control what each camera sees.

| Constant | Value | Owned by | Used for |
|---|---|---|---|
| `SCENE_LAYER` | 0 | `packages/viewer` | All regular scene geometry (walls, items, slabs…) |
| `EDITOR_LAYER` | 1 | `apps/editor` | Editor helpers, grid, tool previews, cursor sphere |
| `ZONE_LAYER` | 2 | `packages/viewer` | Semi-transparent zone fills — composited separately |

Import constants from `@pascal-app/viewer`; never hardcode layer numbers.

See `wiki/architecture/layers.md`.
