# @slipway/theme

Slipway's docs theme: layouts, chrome, components, design tokens, and styles ([ADR 0023](../../docs/adr/0023-engine-theme-boundary.md)). The engine reads it only by listing the folders below; adding a unit never edits a shared index file.

The theme may `import type` from `packages/engine/src/contracts/` and imports nothing else from the engine. Preact is a peer dependency, with `jsx: react-jsx` and `jsxImportSource: preact` in `tsconfig.json`.

## Folders

| Folder | One unit is | Guide |
| --- | --- | --- |
| `layouts/` | `<name>.ts` | [layouts/README.md](layouts/README.md) |
| `chrome/` | `<name>/` | [chrome/README.md](chrome/README.md) |
| `components/` | `<name>/` | [components/README.md](components/README.md) |
| `tokens/` | `<group>.json` | [tokens/README.md](tokens/README.md) |
| `styles/` | `<name>.css` | [styles/README.md](styles/README.md) |

## Commands

From the repository root, `npm run check` runs the tests and the type check with everything else. For this package alone:

- `npm test --workspace @slipway/theme`
- `npm run typecheck --workspace @slipway/theme`

## Layout

- `src/index.ts`: exports the theme manifest (`name` and `folders`) and nothing else.
- `test/`: `*.test.ts` files for `node --test`.
