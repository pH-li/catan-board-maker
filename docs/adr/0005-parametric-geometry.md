# 0005. Build pieces as parametric models in millimeters, using manifold-3d

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Every piece must match the real component dimensions (for example a terrain hex of 79 × 89 mm, a number chit of Ø25 mm, a settlement of 14 × 10 × 12 mm) and come out as a **watertight** mesh that slicers accept. Users customize pieces through silhouettes, embossed or engraved text and icons, heightmap reliefs, edges and tolerances. They don't sculpt freely.

Cutting and joining shapes (engraving text, hollowing undersides) needs **CSG** (constructive solid geometry) booleans that reliably produce valid meshes.

## Decision

1. **Parametric models.** Each component has a pure builder function, `build<Piece>(params, detail) → Mesh`. A design is just JSON parameters.
2. **Millimeters are the only unit.** Every dimension comes from `packages/specs` (the single source of truth). Geometry code contains no magic numbers.
3. **manifold-3d** (WebAssembly) performs all booleans and guarantees **manifold** (watertight) output. three.js helpers (`ExtrudeGeometry`, `SVGLoader`, `TextGeometry`) create the input shapes.
4. **Level of detail:** `detail: 'preview' | 'full'`. Preview lowers the resolution of things like heightmap grids and curve segments, for the board and live editing. Export always uses `full`.
5. **Validation before export:** the mesh must be manifold, with a minimum wall thickness of 0.8 mm (a warning, with the offending piece identified).
6. Output is **binary STL**.

**Out of scope:** uploading or merging custom CAD/STL models.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Free-form mesh editor | Maximum creative freedom | Huge scope; easy to produce unprintable meshes | Not the product; parametric keeps outputs printable |
| Importing user CAD/STL models | Flexibility | Validation, security and scaling problems; out of scope by decision | Explicitly excluded |
| three-bvh-csg for booleans | Pure JS, fast | Doesn't guarantee manifold output | Printability requires watertight meshes |
| OpenSCAD/CadQuery on the server | Powerful CAD kernels | Native binaries, no browser preview parity | Breaks [0001](0001-typescript-everywhere.md) |

## Consequences

**Positive**
- Designs are tiny JSON documents that are easy to store, validate, share, diff and test.
- Builders are **pure functions**: the same input always gives the same output, which makes them trivially testable and cacheable.
- Dimension tests can assert each bounding box within ±0.05 mm of spec.

**Negative / trade-offs**
- The creativity users get is bounded by the options we build.
- WASM initialization adds startup time, so it's loaded once per worker.

## Related

- [0001 TypeScript everywhere](0001-typescript-everywhere.md), [0015 Testing](0015-testing-strategy.md)
