# 0004. Use react-three-fiber and drei for 3D, including a 3D board

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Two screens render 3D:

1. **Piece designer:** one piece at a time, updating live as settings change.
2. **Board builder:** the full board in 3D (19 hexes, 18 chits, 9 ports, plus optional player pieces). Users click and hover to select and place parts.

**three.js** is the standard WebGL library for the browser. The question is how React and three.js work together.

## Decision

Use **react-three-fiber (R3F)**, which renders three.js scenes as React components, plus **drei**, its helper library.

- **Picking:** R3F's built-in pointer events (`onClick`, `onPointerOver`), which do raycasting for us.
- **Repeated parts:** drei's `<Instances>`, which draws many copies in one call.
- **Camera:** `<OrbitControls>` with limits (no going below the table, a zoom range), plus a top-down button for precise placement.
- **Helpers:** `<Grid>` for a 1 mm grid, and HTML labels for dimensions.
- **Level of detail:** the board uses **preview** meshes; the designer and export use **full** detail ([0005](0005-parametric-geometry.md)).
- Mesh generation runs in a **Web Worker**. The viewer only receives finished geometry buffers.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Plain three.js inside `useEffect` | No extra layer; learn three.js directly | React state and the scene graph must be synced by hand; manual cleanup risks GPU memory leaks; picking written by hand | Our viewer is driven by lots of React state |
| A 2D (SVG) board builder | Simpler, light, easy on mobile | Doesn't show the actual printed pieces | Lily wants a 3D board |
| Babylon.js | Full engine with an editor | Smaller React integration; heavier | R3F's ecosystem fits better |

## Consequences

**Positive**
- The scene updates declaratively from state ("change the color and the piece turns red").
- Automatic disposal of GPU resources when components unmount.
- Picking, controls and instancing come with little code.

**Negative / trade-offs**
- An extra abstraction. Advanced cases still need three.js knowledge.
- Performance needs care: keep per-frame work out of React renders, and use refs and `useFrame` for animation.

**Follow-ups**
- Performance budget for the board view: 60 fps on a mid-range laptop. Measure it in Phase 4.

## Related

- [0003 React](0003-react-vite.md), [0005 Parametric geometry](0005-parametric-geometry.md), [0011 State](0011-state-management.md)
