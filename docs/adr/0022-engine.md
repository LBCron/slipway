# 0022. The engine is TypeScript on unified, rendered with Preact, Rust-ready

**Status:** Accepted (2026-11-06).

## Context

[ADR 0021](0021-docs-first.md) replaces Jekyll with Slipway's own engine in `packages/engine`. About thirty engine tasks in `0.3.0` (the Engine core and Engine outputs epics in [ROADMAP.md](../../ROADMAP.md)) will be built in parallel, many by agents that each see one issue. Without one written architecture, each task would choose its own language setup, parser, template system, and output rules, and the pieces would not fit.

The engine has to deliver the promises that ADR 0021 carries over from the Jekyll theme, now in one build with no classic-Pages path:

- **Exact source positions.** Source maps ([ADR 0007](0007-source-maps.md)) and the "Edit with your agent" button need the file and line of every block, not only of headings. Liquid could never give them.
- **Pages read without JavaScript.** Every page works with scripts off, and `assets/js/` stayed under 10 KB; the engine keeps both rules.
- **Proof.** A build that agents change must be checkable: the same input has to give the same output, so a diff in the output means a change in the input.
- **Speed later, not now.** Evergreen's docs are about 600 pages. A pure TypeScript build is fast enough for that, but parsing and highlighting are the hot paths if sites grow, and the API must not have to change to speed them up.

The repository already runs on Node: the tools, the doctor, the MCP server, and the tests are `.mjs` files with no build step, and CI runs only `node` and `npm`.

## Decision

1. **Language: TypeScript run by Node 24, with no compile step.** Engine source is `.ts` and Node runs it directly through type stripping. Only erasable syntax is allowed: no `enum`, `namespace`, parameter properties, or decorators; type-only imports use `import type`; relative imports name the `.ts` file. `tsconfig.json` sets `noEmit`, `erasableSyntaxOnly`, `verbatimModuleSyntax`, `allowImportingTsExtensions`, and `strict`, and `tsc --noEmit` is part of `npm run check`. Because type stripping does not transform JSX, components are written with Preact's `h()` and no `.tsx` files. Existing `.mjs` files stay as they are and gain `// @ts-check` over time; nothing is rewritten only to change its extension.
2. **Pipeline.** A build runs these stages in order, each a function from typed input to typed output (the types are frozen in `packages/engine/src/contracts/`):
   1. **Config:** load and validate the site config (ADR 0024).
   2. **Discover:** list content files, data files, assets, and app-page folders.
   3. **Front matter:** parse each file's front matter, keeping the line of every key.
   4. **Markdown:** parse the body with remark into an mdast tree in which every node keeps its position.
   5. **HTML:** turn mdast into a hast tree with rehype, carrying the positions over; directives, code fences, links, and headings are handled here as tree transforms.
   6. **Site model:** assemble pages, URLs, navigation, languages, and data into one immutable model, and check links and anchors against it.
   7. **Render:** render each page with Preact's `renderToString` from the theme's layouts and components. Preact runs only on the server; no client framework ships.
   8. **Outputs:** write HTML, Markdown twins, `sourcemap.json`, `llms.txt`, the search index, CSS, islands, and assets.

   Stages talk only through the contract types, so a stage can be replaced (by a cache, a plugin, or Rust; rule 4) without changing the others.
3. **Client code is islands.** A component that needs the browser ships a small plain ES module beside it. The engine adds a deferred `<script type="module">` for that module only on pages that use the component, and bundles nothing else. There is no hydration framework, no virtual DOM in the browser, and no global client runtime. Every page reads and navigates with JavaScript off; controls a script enhances start `hidden`.
4. **Rust later, behind the same API.** Hot paths (Markdown parsing, syntax highlighting) may move to Rust through NAPI (napi-rs) after `0.3.0`. A Rust stage implements the same TypeScript contract, must produce byte-identical output to the TypeScript stage on the test fixtures, and is optional: the TypeScript stage stays as the fallback when no prebuilt binary exists for a platform. Nothing in `0.3.0` depends on Rust.
5. **Determinism.** The same input gives byte-identical output. Files are discovered and written in sorted order (by path, comparing code points); object keys in JSON outputs are written in a fixed order; nothing reads the clock, the environment, the machine, or random values unless the config asks for it (a build date comes from the config or the commit, never from `Date.now()`); IDs and hashes come from content; library versions are pinned by the lockfile; line endings are LF. A test builds the fixture twice into different folders and compares every byte.
6. **Source positions on every block.** Every block element in the HTML output (headings, paragraphs, lists and items, tables, code blocks, block quotes, directives, and component roots) carries `data-source="<path>:<line>"`, the source file relative to the site root and the 1-based line where the block starts. These positions come from the mdast and front-matter positions in stages 3 to 5 and feed `sourcemap.json` (format unchanged from ADR 0007) and the "Edit with your agent" button. A transform that creates a node without a position copies the position of the node it came from.
7. **Libraries.** Runtime dependencies of the engine, each with the reason it was chosen and the main alternative that was rejected:

   | Library | Why | Rejected alternative |
   | --- | --- | --- |
   | `unified` | One plugin pipeline over syntax trees (mdast, hast) with positions on every node, so stages are tree transforms and positions survive end to end. | `markdown-it`: a token stream with line maps only on some block tokens and no tree to transform. |
   | `remark-parse`, `remark-gfm` | CommonMark plus the GitHub extensions docs use (tables, task lists, footnotes, strikethrough, autolinks), so pages render the way they do on GitHub. | `marked`: fast, but keeps no source positions and is not CommonMark-exact. |
   | `remark-rehype`, `rehype-*` | Turns Markdown into an HTML tree that keeps positions, so `data-source`, heading IDs, and link rewrites are transforms on hast, not regexes on strings. | Post-processing HTML strings with `parse5` or `cheerio`: the positions of the Markdown source are lost by then. |
   | `shiki` | Highlighting at build time with the TextMate grammars and themes VS Code uses: accurate, no client JavaScript, light and dark from one pass, and output as hast. | `highlight.js` and Prism: regex grammars that are less accurate and are usually run in the browser. |
   | `pagefind` | A static search index built from the output, split into chunks the browser loads on demand, with no server and support for many languages, including Persian. | Lunr or FlexSearch: the whole index is shipped to every reader; Algolia DocSearch: a hosted service the site depends on. |
   | `lightningcss` | Bundles, minifies, and lowers CSS (nesting, logical properties) for the browsers the theme targets, deterministically, in one dependency. | PostCSS with `postcss-nesting`, `autoprefixer`, and `cssnano`: several packages and more configuration for the same result. |
   | `preact` | Typed components for layouts and chrome, with a small API and no client framework required. | React: a larger server renderer for no gain when nothing hydrates; string templates (Liquid, Nunjucks): untyped, and Liquid is what ADR 0021 removes. |
   | `preact-render-to-string` | Synchronous `renderToString` for Preact components on the server, with stable output for the same props. | `react-dom/server`: comes only with React. |

   `shiki`, `pagefind`, and `lightningcss` ship prebuilt binaries or WebAssembly; that is acceptable because they are pinned and none is needed to read the source. Adding any other runtime dependency to the engine needs a reason in its pull request.

## Consequences

- Every engine task builds a stage of the pipeline in rule 2 against the contract types, in TypeScript under the rules of rule 1. A task that needs a different stage order or a new stage changes this ADR first.
- The engine requires Node 24; `foundation.engine-package` raises `engines` and the Node version in CI when it creates the workspace.
- Without JSX, layouts and components read as nested `h()` calls. That is the price of no compile step; a compile step can be added later without changing the components' output.
- Output carries one `data-source` attribute per block, which makes HTML slightly larger. The attribute is the contract that source maps and agent editing depend on, so it is not optional in `0.3.0`.
- Determinism is a test, not a guideline: a change that makes two builds of the fixture differ fails `npm run check`.
- The engine knows nothing about the theme (ADR 0023): the libraries above are the engine's; the theme supplies layouts, components, CSS, and islands through the registries and brings no runtime dependency of its own without a reason in its pull request.
- A Rust stage is a later, optional speed-up. Until one exists, contributors need only Node and npm.
