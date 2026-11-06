# 0024. The site config is `slipway.yml`, and every `_config.yml` key has a place in it or is dropped

**Status:** Accepted (2026-11-06). Replaces the config half of [ADR 0003](0003-content-model.md), as [ADR 0021](0021-docs-first.md) announced.

## Context

[ADR 0021](0021-docs-first.md) replaces Jekyll with Slipway's own engine, and [ADR 0022](0022-engine.md) makes loading the config the first stage of every build. Until now a site was configured in Jekyll's `_config.yml`: Jekyll's own keys (`baseurl`, `plugins`, `defaults`, `include`), plugin keys (`remote_theme`), and Slipway's keys under `slipway:` ([site/config.md](../../site/config.md), generated from `schema/slipway-config.schema.json`). Without Jekyll, most of that file means nothing, and the engine needs a file of its own.

Three things constrain the new file:

- **Evergreen has to move without rewriting content.** Its docs (about 600 pages) set a base path, rely on `jekyll-optional-front-matter` (pages without front matter), `jekyll-readme-index` (a folder's `README.md` is its index), and `jekyll-redirect-from` (`redirect_from` and `redirect_to`), give every page `layout: doc` through `defaults`, copy `.well-known` through `include`, and set a logo and a body font under `slipway:`. Every one of these needs a home, or the move rewrites hundreds of files.
- **The engine knows nothing about the theme** ([ADR 0023](0023-engine-theme-boundary.md), rule 4). Logo, tokens, header style, and analytics are the theme's, so the engine cannot validate them or give them meaning; it can only carry them.
- **Agents read and edit the config.** A closed schema catches typos with a line number, and settings that live in data (not code) can be read without running anything.

## Decision

1. **One file, `slipway.yml`, at the site root.** The site root is the folder that holds `slipway.yml`; every path in it is relative to that folder. `slipway build [dir]` reads `<dir>/slipway.yml`, and `--config <file>` picks another file. Without a config file the engine builds with the defaults below and the folder name as the title. When a `_config.yml` sits there and no `slipway.yml` does, the engine reports a warning that names the migration tool (`evergreen.migrate-config`).
   - **`slipway.config.ts` is reserved for code**, and only for code: it default-exports `{ plugins: [...] }` for the plugin API (`engine.plugins`). It cannot set any key of `slipway.yml`, so every setting stays readable without running the site's code. Until the plugin API lands, a `slipway.config.ts` is an error diagnostic, not silently ignored.
   - The file name is the one Slipway name the engine source contains: it is a single constant in `packages/engine/src/config/`, and `quality.boundary` allows that constant and nothing else (ADR 0023, rule 4).
2. **The keys.** The top level is closed: an unknown key is an error that points at its line. Every key is optional.

   | Key | Type | Default | What it does |
   | --- | --- | --- | --- |
   | `title` | string | the folder name | Site name, in the header, the page title, `llms.txt`, and JSON-LD. |
   | `description` | string |  | One-sentence summary of the site, for the home page's meta description and `llms.txt`. |
   | `url` | string |  | Scheme and host of the published site, without a trailing slash, such as `https://arminazimi.github.io`. Needed for the sitemap, canonical links, and social images; without it they are relative or left out, with a warning. |
   | `base` | string | `""` | Path the site lives under, such as `/evergreen`, starting with `/` and without a trailing slash; `""` at a domain root. Every URL the engine writes carries it. |
   | `lang` | string | `en` | Default language of every page, as a BCP 47 tag. A page's `lang` wins. |
   | `dir` | `ltr` or `rtl` | from `lang` | Default text direction. Unset, it comes from `lang` (`rtl` for `fa`, `ar`, `he`, `ur`, and the other right-to-left languages). A page's `dir` wins. |
   | `languages` | list of `{code, name, prefix}` | `[]` | The site's languages in switcher order, with the same three keys as in 0.2 ([ADR 0010](0010-ui-strings-and-language-urls.md)). Unset for a one-language site. |
   | `source` | string | `.` | Folder that holds the content, relative to the site root. Files outside it are not pages. |
   | `include` | list of strings | `[]` | Paths under `source` that discovery would skip (names starting with `.` or `_`) but should copy or build, such as `.well-known`. |
   | `exclude` | list of strings | `[]` | Paths under `source` that are neither pages nor copied. `node_modules`, the output folder, and `slipway.yml` are always excluded. |
   | `navigation` | string | `_data/navigation.yml` | Navigation file, relative to the site root. When it does not exist, navigation is built from folders and `order` (`engine.nav-folders`). |
   | `defaults` | list of `{path, values}` | `[]` | Front matter applied to every page under `path` (a folder relative to `source`; `""` for all) unless the page sets the key itself. Later entries win over earlier ones; a page always wins. |
   | `repository` | object |  | Where the source lives, for "Edit this page" links and the "Edit with your agent" prompt. `url`: the repository's web URL, such as `https://github.com/arminazimi/evergreen`. `branch`: default `main`. `dir`: the site root's folder inside the repository, such as `docs`; default `""`. |
   | `theme` | object |  | `name`: the theme package or folder (default `@slipway/theme`). `options`: an object the engine passes to the theme unread; the theme validates it against its own schema and reports errors with their line through the engine's diagnostics. |
   | `search` | `false` or object | `{}` | `false` builds no search index. As an object: `text`, the most characters of a page's body the index keeps (default `300`; titles and headings are always indexed). Pages leave the index with `search: false` in front matter. |
   | `redirects` | map of path to path or URL | `{}` | Redirect pages: each key is an old path under `base`, each value a path on the site or an `https` URL. Merged with `redirect_from` and `redirect_to` in front matter; the same old path claimed twice, or a redirect over a real page, is an error. |
   | `outputs` | map of name to boolean | every output on | Turns off an output by its name in `packages/engine/src/outputs/` (ADR 0023, rule 2), such as `twins: false` or `llms: false`. HTML pages and assets cannot be turned off. An unknown name is an error. |

   The schema (`engine.config-schema`) describes exactly these keys, and `SiteConfig` in `packages/engine/src/contracts/` is the same shape. Adding a key later is a minor change; renaming or removing one follows the deprecation rules of [ADR 0012](0012-public-api.md), which ADR 0021 keeps for the engine's config.
3. **Front matter the engine reads.** The engine parses all front matter, gives these keys their meaning, and passes every key, known or not, to the layout. Unknown keys are kept and never rejected, as in 0.2.

   | Key | What the engine does with it |
   | --- | --- |
   | `title` | Page title. Without it, the first `h1`; without that, the file name (`engine.title`). |
   | `description` | Meta description, `llms.txt` entry, and search excerpt. |
   | `layout` | Name of the theme layout that renders the page. Unset, it comes from `defaults`, then the theme's default layout. An unknown name is an error. |
   | `permalink` | Output URL, a path starting with `/`. Unset, the URL comes from the file path: `guide/install.md` and `guide/install/index.md` both give `/guide/install/`. |
   | `redirect_from` | Old paths (a string or a list) that redirect to this page. |
   | `redirect_to` | A path or URL this page redirects to; the page's body is not rendered. Read so Evergreen's redirect pages move unchanged. |
   | `lang` | Page language, a BCP 47 tag; defaults to the site's. |
   | `dir` | `ltr` or `rtl`; defaults to the direction of the page's `lang`. |
   | `order` | Sort order among siblings in navigation built from folders. Pages without it follow, sorted by title. |
   | `status` | `experimental`, `beta`, `deprecated`, `proposed`, `accepted`, `rejected`, or `superseded`. The engine checks the value and puts it in the site model; the theme decides how to show it. |
   | `agent` | Instructions for agents editing this page: `edit` (`false` for a generated page that must be changed at its source), `source` (the file to edit instead), and `notes` (a short instruction). The engine writes them into the page's twin and `sourcemap.json`, and the doctor and the MCP server follow `edit: false`. |

   Three keys from 0.2 keep their meaning because ADR 0021 carries their promise over: `image` (the page's social image, [ADR 0015](0015-og-images.md)), `translation_key` (links translations, ADR 0010), and `search: false`. `published: false` keeps a page out of the build. `nav_order` and `direction` are read as deprecated aliases of `order` and `dir` through `0.3.x`, with a warning naming the file and line; the migration tool (`evergreen.migrate-content`) renames them.
4. **Compatibility with 0.2 content.**
   - `_data/navigation.yml` keeps its shape (`schema/navigation.schema.json`): top-level lists such as `main` and `docs`, each item a link (`title`, `url`, optional `match`) or a group (`title`, `children`). URLs in it are written without `base`, as in 0.2, and the engine adds it.
   - A page without front matter still builds: it gets its title from the first heading (rule 3), its layout from `defaults`, and its URL from its path.
   - A folder's `README.md` is its index page when the folder has no `index.md` (what `jekyll-readme-index` did), so `docs/rules/README.md` is `/rules/`. With both, `index.md` wins and the engine warns.
   - `.md` and `.html` files are pages; `_data/*.yml` and `_data/*.json` are data; every other file under `source` is copied as is. Folders starting with `_` or `.` are skipped unless `include` names them.
5. **Every 0.2 config key maps to a new key or is dropped.** This table covers every key in [site/config.md](../../site/config.md) and the Jekyll keys Slipway sites set. The migration tool (`evergreen.migrate-config`) applies it and lists each dropped key it finds. `theme.options.*` keys are the theme's: their final names are set by the theme's schema in the design epic, and the theme reports an unknown one like any other error.

   | `_config.yml` key (0.2) | `slipway.yml` key (0.3) | Note |
   | --- | --- | --- |
   | `title` | `title` |  |
   | `description` | `description` |  |
   | `url` | `url` |  |
   | `baseurl` | `base` | Same value: leading `/`, no trailing slash. |
   | `lang` | `lang` |  |
   | `direction` | `dir` | Same name as the front matter key. |
   | `author` | dropped | Bylines and feeds leave with the blog layouts (ADR 0021). JSON-LD names the site `title` as the publisher. |
   | `slipway` | dropped as a group | Its keys move to the rows below; the engine has no `slipway:` namespace (ADR 0023, rule 4). |
   | `slipway.source_dir` | `repository.dir` | It always meant the site's folder in the repository, for edit links. |
   | `slipway.languages` | `languages` | The engine owns languages. |
   | `slipway.languages[].code` | `languages[].code` |  |
   | `slipway.languages[].name` | `languages[].name` |  |
   | `slipway.languages[].prefix` | `languages[].prefix` |  |
   | `slipway.search` | `search` |  |
   | `slipway.search.text` | `search.text` |  |
   | `slipway.header` | `theme.options.header` |  |
   | `slipway.fonts` | `theme.options.fonts` |  |
   | `slipway.logo` | `theme.options.logo` |  |
   | `slipway.logo.light` | `theme.options.logo.light` |  |
   | `slipway.logo.dark` | `theme.options.logo.dark` |  |
   | `slipway.logo.favicon_svg` | `theme.options.logo.favicon_svg` |  |
   | `slipway.logo.touch_icon` | `theme.options.logo.touch_icon` |  |
   | `slipway.tokens` | `theme.options.tokens` |  |
   | `slipway.tokens.accent` | `theme.options.tokens.accent` |  |
   | `slipway.tokens.canvas` | `theme.options.tokens.canvas` |  |
   | `slipway.tokens.text` | `theme.options.tokens.text` |  |
   | `slipway.tokens.radius` | `theme.options.tokens.radius` |  |
   | `slipway.tokens.font_body` | `theme.options.tokens.font_body` |  |
   | `slipway.tokens.font_display` | dropped | The display serif leaves with the editorial type system (ADR 0021, superseding ADR 0004). |
   | `slipway.analytics` | `theme.options.analytics` | The script is chrome the theme renders. |
   | `slipway.analytics.goatcounter` | `theme.options.analytics.goatcounter` |  |
   | `slipway.analytics.plausible` | `theme.options.analytics.plausible` |  |
   | `slipway.analytics.umami` | `theme.options.analytics.umami` |  |
   | `slipway.analytics.umami.url` | `theme.options.analytics.umami.url` |  |
   | `slipway.analytics.umami.website_id` | `theme.options.analytics.umami.website_id` |  |
   | `slipway.persian` | `theme.options.persian` |  |
   | `slipway.persian.digits` | `theme.options.persian.digits` | Digits in section numbers, reading time, and dates on doc pages. |
   | `slipway.persian.jalali` | dropped | It dated posts only; posts are removed (ADR 0021). |
   | `slipway.arabic` | `theme.options.arabic` |  |
   | `slipway.arabic.digits` | `theme.options.arabic.digits` | As `persian.digits`. |
   | `slipway.arabic.hijri` | dropped | It dated posts only. |
   | `slipway.comments` | dropped | Comments were on posts only; docs pages stay comment-free. |
   | `slipway.comments.repo` | dropped |  |
   | `slipway.comments.repoId` | dropped |  |
   | `slipway.comments.category` | dropped |  |
   | `slipway.comments.categoryId` | dropped |  |
   | `slipway.comments.mapping` | dropped |  |
   | `slipway.comments.lang` | dropped |  |
   | `slipway.comments.light` | dropped |  |
   | `slipway.comments.dark` | dropped |  |
   | `slipway.social` | dropped | It fed the links layout, which is removed. Header links are the theme's to design. |
   | `slipway.social[].icon` | dropped |  |
   | `slipway.social[].title` | dropped |  |
   | `slipway.social[].url` | dropped |  |
   | `theme`, `remote_theme` | `theme.name` | Only when it named another theme; `slipway-theme` and `arminazimi/slipway` become the default. |
   | `plugins` | dropped | Jekyll plugins. What Slipway sites used them for is built in: optional front matter and README indexes (rule 4), redirects (rule 2), the sitemap, and SEO tags. |
   | `defaults` | `defaults` | `scope.path` becomes `path`; `scope.type` is dropped (there are no collections). `values` is unchanged. |
   | `include` | `include` |  |
   | `exclude` | `exclude` |  |
   | `permalink` | dropped | URLs come from the file path (rule 3), which is what `pretty` gave. A page that needs another URL sets `permalink`, or the old one gets a `redirects` entry. |
   | `collections` | dropped | Docs are folders; `_posts` and other collections leave with the blog. |
   | `future` | dropped | It published future-dated posts. |
   | `github` | `repository.url` | Only the repository URL is kept. |
   | `markdown`, `kramdown`, `highlighter` | dropped | The engine's Markdown and highlighting are fixed by ADR 0022. |

## Consequences

- `foundation.contracts` writes `SiteConfig` with exactly the keys of rule 2, and `engine.config-schema` and `engine.config-load` build on it; changing a key after that costs a new ADR (ADR 0023, rule 5).
- Evergreen moves with a new `slipway.yml` and no content edits: `baseurl` becomes `base`, `defaults` and `include` keep their values, `source_dir` becomes `repository.dir`, and its pages without front matter, `README.md` indexes, and redirect pages build as they are. URL parity (`evergreen.url-parity`) proves it.
- The engine validates only its own keys. A typo under `theme.options` is caught by the theme, so a theme that ships no schema gets no checking there; Slipway's theme must ship one.
- `outputs` keys are the names of the output units, checked by the loader against the output registry, so adding an output (one file, ADR 0023) adds its switch without editing the config schema.
- Sites gain one migration step: `_config.yml` to `slipway.yml`, done by the migration tool and described in `UPGRADING.md`. Front matter needs none in `0.3.x`, thanks to the aliases in rule 3; the aliases may be removed in a later minor release under ADR 0012's deprecation rules.
- `site/config.md` and the Skills' config reference are regenerated from the new schema (`site.config-ref`, `tooling.skills-content`); until then they describe 0.2.
