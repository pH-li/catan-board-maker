# 0012. Use Tailwind CSS and shadcn/ui for styling and components

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The UI needs many controls: sliders, selects, tabs, dialogs, toggles, tooltips, color pickers and progress bars. Building these correctly is hard, mostly because of **accessibility**: keyboard navigation, focus management and screen-reader labels (ARIA). We want accessible behaviour out of the box, a consistent look, and full control over styling.

## Decision

- **Tailwind CSS** for styling: utility classes built on a shared design scale for spacing, color and type.
- **shadcn/ui** for components. Its components are built on **Radix** primitives (headless, accessible behaviour) and copied into `apps/web/src/components/ui/`, so **we own and can edit the code**.
- Design tokens (colors, radii) are defined as CSS variables, with light and dark themes.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Plain CSS / CSS Modules | No tooling; learn CSS deeply | Every accessible control built by hand | Accessibility risk and time |
| MUI | Complete library | Harder to restyle; heavier; distinctive look | Less control |
| Mantine | Complete, includes a color picker | A dependency we can't edit; its own styling system | Prefer owning component code |

## Consequences

**Positive**
- Accessible controls by default.
- Consistent visual language through the design scale.
- No lock-in to a component library's release cycle.

**Negative / trade-offs**
- Markup carries long class lists, which we extract into components when they repeat.
- We maintain the copied components ourselves, including applying upstream fixes.

## Related

- [0003 React](0003-react-vite.md)
