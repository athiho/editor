---
name: build-3d-feature
description: Step-by-step workflow for adding a new 3D node type to the Pascal editor — from Zod schema in core, through viewer renderer and system, to an optional editor tool. Use when the user asks to add a new object type, a new 3D element, or a new placeable node.
allowed-tools: Bash(git *) Bash(bun *) Read Grep Glob Edit Write
---

Adding a new 3D node type requires nine coordinated changes across three packages. Work through each step in order; the TypeScript compiler catches most wiring mistakes as you go.

The architecture diagram at `wiki/architecture/diagram.md` shows how all the pieces connect. Read it first if you are unfamiliar with the codebase.

## 0. Gather context

Before writing code, decide:

- **Node name** — use `kebab-case` (e.g. `my-object`). This becomes the schema `type` discriminator and the file/directory name everywhere.
- **Parent** — which node type will own this node? (Level, Wall, Slab, etc.)
- **Geometry source** — procedural (build in the system) or GLB model (load with `useGLTF`)?
- **Placement surface** — floor, wall, ceiling, or free-floating?

Read the relevant wiki pages before writing code:

```bash
# Always read these four:
cat wiki/architecture/node-schemas.md
cat wiki/architecture/renderers.md
cat wiki/architecture/systems.md
cat wiki/architecture/events.md

# Read these when they apply:
cat wiki/architecture/spatial-queries.md   # if the node is user-placed
cat wiki/architecture/tools.md             # if you are adding an editor tool
cat wiki/architecture/viewer-isolation.md  # if you touch packages/viewer
```

---

## 1. Define the node schema (`packages/core`)

Create `packages/core/src/schema/nodes/<type>.ts`.

```ts
import { z } from 'zod'
import { BaseNode, nodeType, objectId } from '../base'

export const MyObjectNode = BaseNode.extend({
  id: objectId('my-object'),       // generates "my-object_<nanoid>" by default
  type: nodeType('my-object'),     // literal discriminator
  position: z.tuple([z.number(), z.number(), z.number()]).default([0, 0, 0]),
  // add your domain fields here
}).describe('My object node — one-line description of what it represents')

export type MyObjectNode = z.infer<typeof MyObjectNode>
```

Rules:
- Always use `objectId('<type>')` and `nodeType('<type>')` — never hardcode IDs or type strings.
- Keep the schema in `packages/core`. No Three.js, no UI imports.
- Call `.parse({})` to create instances — it auto-generates the ID and applies defaults.

---

## 2. Register in the `AnyNode` union (`packages/core`)

Open `packages/core/src/schema/types.ts`.

```ts
// Add import at the top
import { MyObjectNode } from './nodes/my-object'

// Add to the discriminated union array
export const AnyNode = z.discriminatedUnion('type', [
  // …existing entries…
  MyObjectNode,   // ← add here
])
```

Run `bun run typecheck` in the repo root — TypeScript will now tell you everywhere a new `case` is needed.

---

## 3. Register event types (`packages/core`)

Open `packages/core/src/events/bus.ts`.

```ts
// 1. Import your node type
import type { MyObjectNode } from '../schema'

// 2. Add an event alias
export type MyObjectEvent = NodeEvent<MyObjectNode>

// 3. Wire into EditorEvents (add to the intersection near the bottom of the file)
type EditorEvents = GridEvents &
  // …existing entries…
  NodeEvents<'my-object', MyObjectEvent> &
  // …rest…
```

---

## 4. Register in `sceneRegistry.byType` (`packages/core`)

Open `packages/core/src/hooks/scene-registry/scene-registry.ts`.

```ts
export const sceneRegistry = {
  nodes: new Map<string, THREE.Object3D>(),
  byType: {
    // …existing entries…
    'my-object': new Set<string>(),   // ← add here
  },
  // …
}
```

The type parameter of `useRegistry` is now `keyof typeof sceneRegistry.byType`, so TypeScript will accept `'my-object'` once this entry exists.

---

## 5. Create the renderer (`packages/viewer`)

Create `packages/viewer/src/components/renderers/my-object/my-object-renderer.tsx`.

```tsx
import { type MyObjectNode, useRegistry } from '@pascal-app/core'
import { useRef } from 'react'
import type { Group } from 'three'
import { useNodeEvents } from '../../../hooks/use-node-events'

export const MyObjectRenderer = ({ node }: { node: MyObjectNode }) => {
  const ref = useRef<Group>(null!)
  useRegistry(node.id, 'my-object', ref)    // registers id → Object3D

  const handlers = useNodeEvents(node, 'my-object')

  return (
    <group position={node.position} ref={ref} visible={node.visible}>
      {/* Placeholder geometry — geometry generation belongs in the system */}
      <mesh {...handlers}>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="#cccccc" />
      </mesh>
    </group>
  )
}
```

Renderer rules:
- Call `useRegistry` — this is what makes `sceneRegistry.nodes.get(id)` work everywhere.
- Spread `useNodeEvents` handlers onto the `<mesh>` that should be clickable.
- **Do not** run geometry generation here. Geometry belongs in the viewer system (step 6).
- **Do not** import from `apps/editor`.

---

## 6. Wire `useNodeEvents` NodeConfig (`packages/viewer`)

Open `packages/viewer/src/hooks/use-node-events.ts` and add your type to the `NodeConfig` map:

```ts
import type { MyObjectEvent, MyObjectNode } from '@pascal-app/core'

type NodeConfig = {
  // …existing entries…
  'my-object': { node: MyObjectNode; event: MyObjectEvent }
}
```

---

## 7. Create the viewer system (`packages/viewer`)

If your node requires geometry updates or position corrections each frame (procedural geometry, elevation snapping, cutouts…), create `packages/viewer/src/systems/my-object/my-object-system.tsx`.

```tsx
import { type AnyNodeId, type MyObjectNode, sceneRegistry, useScene } from '@pascal-app/core'
import { useFrame } from '@react-three/fiber'
import type * as THREE from 'three'

export const MyObjectSystem = () => {
  const dirtyNodes = useScene((state) => state.dirtyNodes)
  const clearDirty = useScene((state) => state.clearDirty)

  useFrame(() => {
    if (dirtyNodes.size === 0) return
    const nodes = useScene.getState().nodes

    dirtyNodes.forEach((id) => {
      const node = nodes[id]
      if (!node || node.type !== 'my-object') return

      const mesh = sceneRegistry.nodes.get(id) as THREE.Object3D
      if (!mesh) return

      // Compute and apply geometry or position updates here
      // Example: mesh.position.y = computeElevation(node as MyObjectNode)

      clearDirty(id as AnyNodeId)
    })
  }, 2)   // execution order 2 — runs after physics, before render

  return null
}
```

Rules:
- Process **only** dirty nodes — never iterate all nodes every frame.
- Call `clearDirty(id)` after handling each node.
- Return `null` — systems produce no JSX output.
- **Do not** contain UI state, business logic, or imports from `apps/editor`.

Mount the system inside `<Viewer>` (or in the viewer's systems slot) so it runs inside the R3F canvas context:

```tsx
<Viewer>
  <MyObjectSystem />
</Viewer>
```

---

## 8. Add a case to `NodeRenderer` (`packages/viewer`)

Open `packages/viewer/src/components/renderers/node-renderer.tsx` and wire your renderer:

```tsx
import { MyObjectRenderer } from './my-object/my-object-renderer'

// Inside the NodeRenderer component return block:
{node.type === 'my-object' && <MyObjectRenderer node={node} />}
```

Run `bun run typecheck` again — exhaustiveness errors from the `AnyNode` union will appear here if anything is missing.

---

## 9. Build an editor tool (optional, `apps/editor`)

Only needed if users must interactively place or edit this node type. Tools live in `apps/editor/components/tools/`.

Skeleton for a placement tool:

```tsx
import { MyObjectNode, useSpatialQuery, useScene } from '@pascal-app/core'
import { useLiveTransforms } from '@pascal-app/core'
import type { GridEvent } from '@pascal-app/core'
import { emitter } from '@pascal-app/core'
import { useEffect, useRef } from 'react'

export const MyObjectTool = () => {
  const { canPlaceOnFloor } = useSpatialQuery()
  const pendingRef = useRef<MyObjectNode | null>(null)

  useEffect(() => {
    const onGridMove = (e: GridEvent) => {
      const dims: [number, number, number] = [1, 1, 1]
      const rot: [number, number, number] = [0, 0, 0]
      const { valid } = canPlaceOnFloor(levelId, e.localPosition, dims, rot)
      // Update preview material / position using useLiveTransforms
    }

    const onGridClick = (e: GridEvent) => {
      const node = MyObjectNode.parse({ position: e.localPosition })
      const { createNode } = useScene.getState()
      createNode(node, levelId)    // committed — captured for undo
    }

    emitter.on('grid:move', onGridMove)
    emitter.on('grid:click', onGridClick)
    return () => {
      emitter.off('grid:move', onGridMove)
      emitter.off('grid:click', onGridClick)
    }
  }, [canPlaceOnFloor])

  return null
}
```

Tool rules:
- Use `useScene.getState().createNode` for committed mutations (captured for undo).
- Use `useLiveTransforms` for ephemeral drag state (not undo-captured).
- Direct `sceneRegistry` mesh mutations are allowed **only** during an active drag, and only if mirrored in `useLiveTransforms`.
- **Never** import from `@pascal-app/viewer`.
- Clean up all `emitter.on` listeners in the `useEffect` return.

---

## 10. Verify

```bash
# Type-check the whole monorepo
bun run typecheck

# Lint
bun run lint

# Start dev server and test in the browser
bun dev
```

Checklist before opening a PR:

- [ ] Schema defined in `packages/core/src/schema/nodes/<type>.ts`
- [ ] `AnyNode` union updated in `packages/core/src/schema/types.ts`
- [ ] Event alias + `EditorEvents` entry added in `packages/core/src/events/bus.ts`
- [ ] `sceneRegistry.byType['<type>']` entry added
- [ ] Renderer created in `packages/viewer/src/components/renderers/<type>/`
- [ ] `useNodeEvents` `NodeConfig` entry added
- [ ] Viewer system created (if geometry updates needed)
- [ ] `NodeRenderer` case added
- [ ] Editor tool created (if user placement needed)
- [ ] `bun run typecheck` passes with no errors
- [ ] Tested in browser: node appears, is selectable, survives undo/redo
