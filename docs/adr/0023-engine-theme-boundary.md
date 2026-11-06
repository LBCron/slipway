# 0023. The engine/theme boundary, one folder per unit, and registries that discover units

**Status:** Accepted (2026-11-06).

## Context

[ADR 0021](0021-docs-first.md) splits Slipway into two products in one repository: an engine (`packages/engine`) that knows nothing about Slipway, and a theme (`packages/theme`) that is Slipway's look. [ADR 0022](0022-engine.md) fixes the engine's language, pipeline, and libraries. Neither says where a given piece of code lives, or how a new piece is wired in.

That matters because of how `0.3.0` is built. [ROADMAP.md](../../ROADMAP.md) plans about 200 tasks, and up to 20 run at once, most of them by agents that each see one issue. Each pull request has to merge in any order without conflicting with the others. Two habits break that:

- **Shared index files.** When every new output, plugin, layout, or component must be added to a list in one file (an `index.ts` that imports them all, an array in the pipeline, a map in the theme), every task edits that file and every pair of tasks conflicts.
- **A leaky boundary.** When the engine imports a theme component, hard-codes a layout name, or picks a color, the two products cannot change separately, someone else's theme cannot replace Slipway's, and a theme task has to edit engine files.

## Decision

1. **Who owns what.**
   - The **engine** owns the site config, the content model, Markdown processing, navigation, the render host (the code that calls the theme's layouts and components and turns them into HTML), outputs, diagnostics, and the CLI.
   - The **theme** owns layouts, chrome (header, sidebar, footer, table of contents, and the other parts around the content), components, design tokens, styles, and client islands ([ADR 0022](0022-engine.md), rule 3).

   A task that does not fit one side changes this ADR first.
2. **One folder or file per unit.** Each unit has its own path, named after the unit:

   | Unit | Path |
   | --- | --- |
   | Output | `packages/engine/src/outputs/<name>.ts` |
   | Markdown plugin | `packages/engine/src/markdown/plugins/<name>.ts` |
   | Layout | `packages/theme/layouts/<name>.ts` |
   | Chrome part | `packages/theme/chrome/<name>/` |
   | Component | `packages/theme/components/<name>/` |
   | Token group | `packages/theme/tokens/<group>.json` |

   A chrome part or component folder holds `index.ts` (the Preact component, written with `h()`), `style.css`, an optional `client.ts` (its island), and its tests. Layouts are `.ts`, not `.tsx`: ADR 0022 rule 1 runs TypeScript through Node's type stripping, which does not transform JSX. Names are lowercase kebab-case and unique within their kind; the path is the name, so two tasks cannot pick the same one without a conflict Git shows.
3. **Registries discover units by folder.** The engine finds outputs and Markdown plugins by listing its own folders, and finds layouts, chrome, components, and tokens by listing the theme's folders, in sorted order (by path, comparing code points; ADR 0022 rule 5). Each unit exports what its contract type requires; the registry checks the export and reports a diagnostic naming the file when it is missing or wrong. Adding a unit never edits a shared index file, the pipeline, or another unit. Removing one is deleting its file or folder.
4. **The engine knows nothing about the theme.**
   - Nothing under `packages/engine/` imports from `packages/theme/`, by relative path or by package name. The theme is handed to the engine at build time (a folder path from the config or the CLI), and the engine reads it only through the registries in rule 3.
   - The engine contains no Slipway name, color, font, or layout name. Defaults it needs (a fallback layout, a diagnostic's wording) are neutral, and anything a theme can choose is chosen by the theme.
   - The theme may import the engine's contract types (`import type` from `packages/engine/src/contracts/`) and nothing else from the engine.
   - The `quality.boundary` gate enforces the first two points in `npm run check`: it fails on an import from the engine into the theme and on a Slipway name in engine source.
5. **Contracts are frozen.** The types in `packages/engine/src/contracts/` (content model, config, plugin, output, render, and registry types) are written once in the foundation epic. After that they change only through an ADR that names the change and the units it affects. A task that needs a contract change stops and asks for that ADR instead of editing the types.

## Consequences

- Up to 20 pull requests can merge in any order: two tasks touch the same file only when their briefs say so, and then the roadmap chains them by a dependency.
- `foundation.theme-package` creates the empty folders of rule 2, and `engine.registry` writes the discovery of rule 3. Until those land, no unit can be added.
- Units cannot depend on their order in a list. Where order matters (navigation, the order outputs are written), it comes from the contract (a field on the unit) or from the sorted path, never from where a unit was added.
- Discovery costs a directory listing per build. It is cheap next to parsing and rendering, and it makes the engine and the theme data to each other rather than code.
- Another theme can replace Slipway's by providing the same folders, and the engine can be used without Slipway. Both are tested only by `quality.boundary` in `0.3.0`; a second theme is not planned.
- Changing a contract is slower on purpose: it costs an ADR. That is the price of letting many tasks build against the same types at once.
