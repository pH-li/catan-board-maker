<!--
Title must be a Conventional Commit, e.g. "feat(geometry): add gabled roof preset".
It becomes the squash commit on main, and this description becomes its body.
-->

## What

<!-- What does this change? Keep it short. -->

## Why

<!-- The problem or issue this solves. -->

Closes #

## How it was tested

<!-- Commands run, manual steps, and for geometry: printed or measured results. -->

## Screenshots

<!-- For UI changes. Delete if not applicable. -->

## Checklist

- [ ] `pnpm lint`, `pnpm typecheck` and `pnpm test` pass locally
- [ ] Tests cover the change (bug fixes include a test that fails without the fix)
- [ ] README / ARCHITECTURE.md updated if behaviour, structure or a design decision changed
- [ ] Diagram `.mmd` edits have re-rendered `.png` files
- [ ] No new `any`, warnings or unexplained lint disables

## Security

<!-- Answer each one. See ARCHITECTURE.md → Security. -->

- [ ] New or changed inputs are validated with a Zod schema **with limits** (or: no new inputs)
- [ ] No user-influenced value reaches a file path, shell command, query or HTML
- [ ] New ports or services are bound to `127.0.0.1` or internal only (or: none added)
- [ ] New dependencies are needed, maintained, and checked for install scripts (or: none added)
- [ ] ARCHITECTURE.md → Security updated if the attack surface changed
