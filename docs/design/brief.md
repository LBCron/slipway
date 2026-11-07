# Design brief: Slipway docs theme

Status: final draft for maintainer approval (issue #1082). Every mockup, layout, chrome, and token task follows this file.

Slipway is a documentation theme for developers. The goal is to be the first theme people install for a docs site: the place where reading code, finding an API, and moving through hundreds of pages feel fast and calm. Evergreen's docs (about 600 pages) are the first site and the benchmark.

The look is quiet and exact. The quality is in the details (type, spacing, code, keyboard, search), not in a theme or a mascot.

## Principles

1. **Code first.** Code blocks and signatures are the most important elements after prose. They are highlighted at build time, never wrapped, and easy to copy.
2. **Fast to find.** Search opens with `/`, results show the page and the symbol, and the sidebar always shows where you are. Any page is reachable in two keystrokes.
3. **Keyboard complete.** Everything has a shortcut and every shortcut is shown on the page: `/` search, `j` and `k` between sections, `c` copy the focused code block, `t` theme, `m` copy the page as Markdown.
4. **Reference-grade pages.** Function and option pages have a signature block, a parameter table with types and defaults, and an example in every supported language.
5. **Calm color, dark by default.** The site opens in night mode (Kanagawa) for every first visit. A visitor can switch to day (Nord) or to "system", and the choice is remembered. Two palettes only, one per mode. One accent, used for the current item, links, and focus. Never decoration.
6. **Quiet proof.** Trust data (examples tested, drift, source file and line) is present but small: a chip near the title and a detail panel, not a banner.
7. **Logical, not left/right.** All spacing and placement use logical properties so right-to-left works by changing `dir`.
8. **Accessible and light.** WCAG 2.2 AA contrast in both modes, visible focus, and the budgets from the 0.3.0 gates (under 40 KB CSS and 30 KB JavaScript per page).

## Not a theme with a story

No nautical, mascot, or decorative concept. No serif body, no cream paper, no neon accent on dark (a volt-yellow accent was tried and rejected as tiring to the eye), no gradients or glass.

## References: what to take

We take patterns, not assets. Nothing is copied: no screenshots, icons, fonts, or code.

| Site | Take | Skip |
|---|---|---|
| Next.js docs | Three-column page, pager cards, breadcrumb, active-section marking | Animated home page |
| Tailwind docs | Type scale and vertical rhythm, hairline tables, inline code chips | Dense top bar |
| Stripe docs | Language tabs that remember the choice, code header with copy, line marks | Two-column prose/code split as the default |
| Supabase docs | Card grid for the home page, top section tabs that swap the sidebar, command palette | Per-product colored icons |
| Starlight | Full right-to-left mirroring from `dir`, native-name language picker | Nothing: it is the bilingual benchmark |
| Rust std docs | Fast search by symbol, signature-first pages | Dense typography |

## Color

Dark mode uses the **Kanagawa** palette family (night). Light mode uses the **Nord** palette family (snow storm). Both are MIT-licensed palettes; we use their values as a starting point and may adjust them in the token tasks (#1089, #1090) to pass contrast checks.

### Night (Kanagawa)

| Token | Value | Use |
|---|---|---|
| `bg` | `#1f1f28` | Page |
| `surface` | `#2a2a37` | Code panels, inputs |
| `overlay` | `#363646` | Active nav item, callouts, code header |
| `highlight` | `#54546d` | Hover, borders, selection |
| `muted` | `#727169` | Captions, comments |
| `subtle` | `#c8c093` | Secondary text |
| `text` | `#dcd7ba` | Body text |
| `accent` | `#7e9cd8` | Links, focus, current item, primary button |
| `accent-text` | `#7fb4ca` | Accent used as text |
| `keyword` | `#957fb8` | Syntax |
| `function` | `#7e9cd8` | Syntax |
| `string` | `#98bb6c` | Syntax |
| `number` | `#ffa066` | Syntax |
| `type` | `#7aa89f` | Syntax |
| `variable` | `#e6c384` | Syntax |

### Day (Nord)

| Token | Value | Use |
|---|---|---|
| `bg` | `#eceff4` | Page |
| `surface` | `#e5e9f0` | Code panels, inputs |
| `overlay` | `#d8dee9` | Active nav item, callouts, code header |
| `highlight` | `#c5cddc` | Hover, borders, selection |
| `muted` | `#8892a6` | Captions, comments |
| `subtle` | `#4c566a` | Secondary text |
| `text` | `#2e3440` | Body text |
| `accent` | `#5e81ac` | Fills: primary button, focus ring |
| `accent-text` | `#4c6f9a` | Accent used as text |
| `keyword` | `#5e81ac` | Syntax |
| `function` | `#3f7f8c` | Syntax |
| `string` | `#6c8a52` | Syntax |
| `number` | `#94638a` | Syntax |
| `type` | `#4f8a88` | Syntax |
| `variable` | `#b07d2b` | Syntax |

Rules:
- The accent is in the blue family in both modes, so the brand reads the same.
- Text on a filled accent uses `bg` in night and white in day.
- Status colors (note, tip, warning, danger) reuse the syntax hues, always with an icon and a label, never color alone.
- Code blocks follow the page mode (dark in night, light in day), so the page is one surface.
- `muted` is for captions and comments only; body and UI text use `subtle` or `text`.
- Borders are `highlight` mixed 55% into `bg`.

## Type

| Use | Font | License |
|---|---|---|
| Body and UI | **Geist** (variable) | OFL |
| Code | **Geist Mono** | OFL |
| Persian and Arabic | **Vazirmatn** (variable) | OFL |

Self-hosted WOFF2, one file per family, subset by script, `font-display: swap`, with a size-adjusted system fallback so layout does not jump.
- Base 16 px, line height 1.65, line length 65 to 75 characters.
- Scale ratio 1.2 from 16 px. Page title 2rem, h2 1.5rem, h3 1.25rem, tight tracking on titles.
- Code at 0.875em, line height 1.8, ligatures off.
- Persian and Arabic line height about 1.9.

## Layout

| Part | Size |
|---|---|
| Sidebar | 17 rem |
| Article | 46 rem max |
| On this page | 14 rem |
| Page max | 90 rem |
| Header | 3.5 rem |

- Below 64 rem the on-this-page column folds into a list at the top of the article. Below 48 rem the sidebar becomes a drawer.
- 4 px spacing base. Radius 6 px for controls, 10 px for code blocks and cards. Shadows only on popovers.
- Wide tables may break out of the article column up to the page max.

## Components

- **Header:** logo, section tabs (swap the sidebar tree), search field showing `/`, language select, theme switch, repository link.
- **Command palette:** opens with `/` or `Ctrl K`. Groups results as API, Guides, and Actions. Actions include copy page as Markdown.
- **Sidebar:** grouped tree, current page marked with an overlay fill and accent text.
- **Code block:** header row with language tabs and copy; tabs remember the choice across the site; line numbers; marked lines with an accent bar plus a non-color cue; footer chip when the example is tested.
- **Hoverable symbols:** in signatures, names and types show a popover with the type and one line of meaning, with a link to the full page.
- **Parameter table:** name, type, default, required mark, one-line meaning. Hairline rules.
- **Callouts:** overlay fill, icon, label, and a 3 px accent bar on the inline-start edge.
- **Trust chip:** `4/4 examples tested` near the title; detail panel lists examples, drift, source file and line, and agent actions (copy context pack, suggest an edit).
- **Pager and edit link** at the end of every page.

## Anti-goals

- Any themed concept (nautical, terminal cosplay, retro).
- A third color palette, or per-section accent colors.
- Serif body text, cream or paper backgrounds, decorative display type, gradients, glass effects.
- Color as the only signal for state, status, or diff.
- Marketing sections, testimonials, pricing, and blog chrome.
- Client JavaScript for anything that can be HTML or CSS.

## What the mockups must prove

The mockups (#1083 home, #1084 guide, #1085 reference, #1086 mobile) each show night, day, and right-to-left, use only the colors, fonts, and widths above, and include the command palette and a hover popover. The approval task (#1087) records any change to this brief.

## Decisions and open questions

Decided: the default mode is night, not the system setting.

Decided: right-to-left pages use Geist for Latin text and Vazirmatn for Persian text, with code in Geist Mono. The mockups (#1086 and the right-to-left views) confirm the pairing looks balanced on mixed lines.
Decided: wide tables break out to the page width. Where there is no room (small screens, or a table wider than the page), the table scrolls horizontally inside the article, with the first column sticky and a clear edge shadow.
