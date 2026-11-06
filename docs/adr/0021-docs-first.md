# 0021. Slipway is a docs-first theme on its own engine

**Status:** Accepted (2026-11-06). Supersedes, in whole or in part, ADRs 0001, 0003, 0004, 0006, 0007, 0010, 0012, 0015, 0016, 0017, and 0020 (listed under Consequences).

## Context

[ADR 0001](0001-why-slipway.md) started Slipway as a Jekyll-first theme for any kind of site, and [ADR 0011](0011-standalone-and-the-spec.md) added an agent-editable site spec with a checker. By `v0.2.0` the repository had grown in every direction at once:

- **The design reads as a blog.** The editorial type system ([ADR 0004](0004-fonts.md): a display serif for headings, a serif for body text) and the bronze-and-rose palettes suit essays and posts. On reference pages, long guides, and code, readers expect a sans-serif body, a tight measure, and a strong code style, the way well-known docs sites look.
- **The only real site is a docs site.** Evergreen's documentation (about 600 pages) is the one production site built on Slipway. The fifteen starters (academic, blog, newsletter, podcast, portfolio, resume, and others), the blog layouts, and the marketing blocks (hero, pricing, testimonial, logos, stats) have no site that uses them, so nothing tests them against real content.
- **The roadmap had no center.** The old roadmap had more than 45 epics across ten milestones: niche starters, migrations from other themes, ten generator adapters, a CMS, social cards, and the spec. Work spread thin, and no epic made the one real site better.
- **Jekyll limits what a docs theme can do.** Classic GitHub Pages runs no plugins, so source maps, Markdown twins, social images, and diagrams each needed two paths (a committed stub or map for classic Pages, a Ruby plugin for Actions mode; ADRs 0006, 0007, 0015, 0016). Liquid cannot give exact source positions, a build cannot check itself, and the theme cannot be reused outside Jekyll. Every feature paid for two runtimes, Ruby and Node, in CI and in the tests.

## Decision

1. **Docs first.** Slipway is a documentation theme. Its job is docs that people find easy to read and that agents can edit safely, with proof. Evergreen's docs site is the first site and the benchmark: a feature is in scope when Evergreen's docs need it.
2. **Its own engine.** A TypeScript engine replaces Jekyll. The engine and the theme are two products in one repository (`packages/engine`, `packages/theme`): the engine knows nothing about Slipway's names, colors, or layouts, so others can build on it. Its architecture is ADR 0022, the boundary ADR 0023, and the site config ADR 0024.
3. **Theme, then standard.** First the theme reaches the acceptable level in [ROADMAP.md](../../ROADMAP.md) (design approved, quality gates green, source maps, twins, `llms.txt`, and `AGENTS.md` still working) and Evergreen moves to it. Work on the agent-editable standard (the spec, the doctor on other generators, adapters) comes after `0.3.0`.
4. **Everything that is not docs is removed or frozen.** Blog and publishing layouts, non-docs starters and examples, and marketing blocks are removed. Generator adapters are frozen, not removed: they wait for the standard phase.
5. **Jekyll is removed in `0.3.0`.** No Jekyll layouts, includes, plugins, Liquid, Gemfiles, or Ruby remain in the repository or in CI. The last Jekyll version stays reachable on branch `legacy/jekyll` and tag `v0.2.0`, so sites pinned to it keep building.

## Consequences

- **Removed in `0.3.0`:**
  - Starters other than `docs` (academic, bilingual, blog, course, garden, handbook, links, newsletter, nonprofit, podcast, portfolio, project, resume, tool) and examples other than `docs` and `product-docs`.
  - Blog and publishing layouts: posts, archives, tags, authors, feeds, series, related posts, and bylines.
  - Marketing blocks: hero, pricing, testimonial, logos, stats, people, and the like. Blocks that docs need (callout, steps, code group, file tree, terminal, endpoint) are rebuilt as theme components.
  - The Jekyll theme itself: `_layouts/`, `_includes/`, `_plugins/`, `assets/`, `Gemfile`, `Gemfile.lock`, `slipway-theme.gemspec`, and `.ruby-version`.
- **Frozen:** `adapters/` (Docusaurus, Eleventy, Hexo, Hugo, mdBook, MkDocs, Quarto, Starlight, VitePress, Zola) stays in the repository untouched until the standard phase. They get no new features until then. [ADR 0013](0013-astro-adapter.md) is frozen with them, not superseded.
- **`remote_theme` stops being the install path.** A new site is created with `create-slipway` and built with the engine; `remote_theme: arminazimi/slipway@v0.2.0` works only for sites that stay on the last Jekyll release. The `slipway-theme` gem gets no release after `0.2.x`. `UPGRADING.md` gets a migration from `_config.yml` to the new config.
- **What carries over:** the promises, not the mechanisms. Source maps, Markdown twins, `llms.txt`, `AGENTS.md`, per-page social images, right-to-left as a first-class direction, and no JavaScript needed to read a page all stay; the engine produces them in one build with no classic-Pages path.
- **Superseded ADRs** (their files are not edited here; each keeps its text as history):
  - [0001](0001-why-slipway.md), in part: decision 2 (Jekyll first) and the niche starters and theme migrations in decision 3. Decisions 4 and 5 stand.
  - [0003](0003-content-model.md), in whole: the content model is no longer the Jekyll file layout; ADR 0024 defines the site config and how `_config.yml` maps to it.
  - [0004](0004-fonts.md), in whole: the editorial type system (display serif and serif body) gives way to a docs type system with a sans-serif body and a monospaced code font, chosen in the design epic. Vazirmatn for Persian and self-hosting stay.
  - [0006](0006-markdown-twins.md), in part: the classic-Pages stubs and the Actions plugin. The engine writes twins; the `direction` key and rule 6 (every twin is advertised) stay.
  - [0007](0007-source-maps.md), in part: the three Jekyll levels (Liquid page map, committed map, Ruby plugin). The engine writes exact positions at build; the `sourcemap.json` format stays.
  - [0010](0010-ui-strings-and-language-urls.md), in part: the Liquid mechanism (`_includes/i18n-builtin.html`, `t.html`, `_data/i18n/`). Language folders, direction from `lang`, and the link-only switcher stay.
  - [0012](0012-public-api.md), in part: the public surface (`_config.yml` keys, layouts, include names, `remote_theme` and gem pins) ends with `0.2.x`. The release and deprecation rules stay and apply to the engine's config and the theme's public items.
  - [0015](0015-og-images.md), in part: the pure-Ruby plugin and the classic-Pages fallback. The `image` front-matter key stays.
  - [0016](0016-diagrams.md), in part: the classic-Pages and Actions-mode split. Build-time rendering, kept sources, and text alternatives stay.
  - [0017](0017-threat-model.md), in part: the sections on the gem and `remote_theme` describe `v0.2.0` only; the engine and theme packages need their own entries.
  - [0020](0020-social-cards.md), in part: classic Pages (rule 5) and the Actions build step (rule 6). The template and its rules stay.
- **Not affected:** [0002](0002-webmcp.md), [0005](0005-brand-mark-and-palette.md), [0011](0011-standalone-and-the-spec.md) (its spec moves to the standard phase), [0014](0014-evergreen.md), [0018](0018-cms.md), and [0019](0019-api-model.md). [0008](0008-theme-palette.md) stands until the design epic's color tokens replace it.
- Sites on `v0.2.0` get no new features. Evergreen stays on `v0.2.0` until the engine builds all of its pages with every old URL working.
- The roadmap is one milestone, `v0.3.0`, with twelve epics. Work that does not serve docs waits for a later release or is dropped.
