# 0011. Split frontend state: Zustand for the project, TanStack Query for server state, useState for UI

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The frontend has three different kinds of state:

| Kind | Examples | Needs |
|---|---|---|
| **Project state** | every piece's design parameters, the board layout, uploaded image references | shared by many components; frequent updates (slider drags); **undo/redo**; **auto-save** |
| **Server state** | export job status and progress | fetching, caching, loading and error states, **polling** |
| **UI state** | which dialog is open, hover state | local to one component |

Putting all of it in one React Context would re-render every consumer on every change. Next to a 3D canvas, that means dropped frames while dragging sliders.

## Decision

- **Project state → Zustand**, organized as slices (`pieces`, `board`, `assets`):
  - components read it through **selectors**, so they re-render only when their slice of data changes
  - **Immer** middleware for readable immutable updates
  - **zundo** for undo/redo history
  - **persist** middleware with an IndexedDB storage adapter for auto-save, with a schema version for migrations
- **Server state → TanStack Query**: export status uses `refetchInterval`, which stops once the job completes or fails. Mutations handle uploads and export creation.
- **UI state → `useState`/`useReducer`** in the component that owns it.
- **React Context** only for rarely changing values (theme).

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Context + `useReducer` for everything | No libraries | Broad re-renders; manual persistence and undo | Performance near the 3D canvas |
| Redux Toolkit | Structured, great DevTools, widely used | More boilerplate | Zustand is simpler at our size |
| Jotai | Very fine-grained atoms | A different mental model, less common | Zustand fits a document-like project model |

## Consequences

**Positive**
- Slider drags re-render only the affected viewer and controls.
- Undo/redo and auto-save are almost free.
- Server concerns (polling, retries, caching) stay out of the project store.

**Negative / trade-offs**
- Two state libraries to learn, each with a clear job.
- The persisted project schema needs versioned migrations when it changes.

## Related

- [0003 React](0003-react-vite.md), [0004 3D](0004-react-three-fiber.md)
