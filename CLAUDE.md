# CLAUDE.md

This file guides Claude Code (claude.ai/code) when working in this repository.

## What this is

**peak** is a fork of [Pico CSS](https://github.com/picocss/pico) v2.1.1, the final upstream release. Upstream is archived. Pico is a minimal, class-light CSS framework that styles semantic HTML (`<article>`, `<nav>`, `<dialog>`, `<details>`, `role="group"`, `aria-busy`, `data-tooltip`, …) directly, with automatic light/dark mode.

The repo still uses the upstream identity in `package.json` (`@picocss/pico`), `composer.json`, `README.md`, `.github/`, the build log prefix `[@picocss/pico]`, and `scss/helpers/_copyright.scss`. Upstream asks forks to use their own name, because the Pico name and logo are not MIT-licensed. Rename these only when asked.

There is no JavaScript runtime code, no test suite, and no CI. The product is the CSS in `css/`.

## Commands

Uses Yarn v1 (`yarn.lock`) and Node (built with Node 22).

```bash
yarn install --frozen-lockfile
yarn build      # lint (prettier + declaration sort) → sass → themes → autoprefix → minify
yarn dev        # nodemon: rebuild on any scss/ change
yarn lint       # prettier --write + css-declaration-sorter on scss/ (rewrites files in place)
```

What `yarn build` runs, in order (see `package.json`):
1. `lint:prettier`: Prettier on `scss/**/*.scss` (`printWidth: 100`).
2. `lint:sort-scss`: PostCSS with `scss/postcss.config.js` sorts declarations in **SMACSS order**, rewriting the source files.
3. `build:css`: `sass scss/:css/` compiles each non-partial entry file into `css/<name>.css` (expanded).
4. `build:themes`: `node scripts/build-themes` writes temp entry files to `.pico/` (gitignored) and compiles 19 colors × 6 variants into `css/pico[.variant].<color>.css` (compressed).
5. `build:autoprefix`: PostCSS with `css/postcss.config.js` (autoprefixer, `browserslist: defaults`) on the non-minified CSS.
6. `build:minify`: clean-css writes a `.min.css` next to each file.

**Gotcha:** `scripts/build-themes.js` calls `process.stdout.clearLine()`, which only exists on a TTY. In a non-interactive shell (like Claude's Bash tool) `yarn build` crashes at step 4 with `TypeError: process.stdout.clearLine is not a function`, leaving `css/` half-built (unprefixed). Run it under a pseudo-terminal instead:

```bash
script -qec "yarn build" /dev/null
```

The build is deterministic (`caniuse-lite` is pinned). A clean build of an unchanged tree produces no `git diff`, so use the diff to check your change.

## Repository layout

- `scss/`: source of truth. **Never edit `css/` by hand.** It is generated and committed, so rebuild and commit it along with SCSS changes.
- `css/`: about 240 generated files (base, variant and theme builds, each with `.min.css`), plus `css/postcss.config.js` (autoprefixer config, the only hand-written file here).
- `scripts/build-themes.js`: generates the per-color theme builds.

### SCSS entry points (non-underscore files in `scss/`)

Each one is `@use "index" with (...)` using a different config:

| Entry | Config | Output |
|---|---|---|
| `pico.scss` | defaults (azure) | `css/pico.css` |
| `pico.classless.scss` | `$enable-semantic-container: true`, `$enable-classes: false` | `css/pico.classless.css` |
| `pico.fluid.classless.scss` | above + `$enable-viewport: false` | `css/pico.fluid.classless.css` |
| `pico.conditional.scss` | `$parent-selector: ".pico"` | `css/pico.conditional.css` |
| `pico.colors.scss` | standalone color utility classes and vars (`scss/colors/utilities/`) | `css/pico.colors.css` |

`scripts/build-themes.js` builds the same variants (plus `pico.classless.conditional` and `pico.fluid.classless.conditional`) for every color **except azure**, the default, which the plain files above already cover. If you add a variant or color, update both the entry files and the `versions`/`themeColors` arrays in that script.

### Module structure

- `scss/_index.scss`: `@forward`s settings and `@use`s every module in cascade order: theme → layout → content → forms → components → utilities. Order matters.
- `scss/_settings.scss`: all `!default` config: `$theme-color`, `$css-var-prefix` (`--pico-`), `$parent-selector`, feature flags (`$enable-semantic-container`, `$enable-viewport`, `$enable-responsive-spacings`, `$enable-responsive-typography`, `$enable-classes`, `$enable-transitions`, `$enable-important`), the `$breakpoints` map (sm…xxl: breakpoint, viewport width, root font size), and the `$modules` map that switches each module on or off.
- `scss/colors/_index.scss`: the raw palette, 20 families × shades 50–950 (`$azure-550`, …), plus `$black`/`$white`.
- `scss/themes/default/`:
  - `_styles.scss`: global `:root, :host` custom properties (typography, spacing, radii, transitions, and module-specific vars inside `@if map.get($modules, …)` guards).
  - `_light.scss` / `_dark.scss`: `@mixin theme` blocks that set every color variable for that scheme.
  - `_theme-colors.scss`: `$color-mappings`, the primary-color tokens per theme color and per scheme, read with `theme-colors.get("primary", "light")`.
  - `_schemes.scss`: wires the schemes. Light is the default. Dark applies through `prefers-color-scheme: dark` when there is no `data-theme`, or when forced with `[data-theme="dark"]`.
- `scss/helpers/_functions.scss`: `display-rgb()`, `shadow()` (7-layer box-shadow).
- `scss/layout|content|forms|components|utilities/`: one partial per element group. The comment next to each `@use` in `_index.scss` lists the elements and selectors it styles.

## Conventions for writing SCSS

Each module follows this pattern (see `scss/components/_card.scss`):

```scss
@use "sass:map";
@use "../settings" as *;

@if map.get($modules, "components/card") {
  /**
   * Card (<article>)
   */
  #{$parent-selector} article {
    padding: var(#{$css-var-prefix}block-spacing-vertical) ...;
    background: var(#{$css-var-prefix}card-background-color);
  }
}
```

- Wrap the whole module in `@if map.get($modules, "<path>")`. A new module also needs an entry in `$modules` (`_settings.scss`) and a `@use` in `_index.scss`.
- Prefix every element selector with `#{$parent-selector}` so the `.pico` conditional build scopes correctly. `:root`/`:host` are never wrapped.
- Never hardcode `--pico-`. Always write `#{$css-var-prefix}name` for both declaring and reading custom properties.
- Put values in CSS custom properties: define defaults in `themes/default/_styles.scss` and colors in `_light.scss`/`_dark.scss`, then use `var(...)` in modules. Guard new vars with the same `$modules` check as the module that uses them.
- Put class-based selectors (`.container`, `.grid`, `.dropdown`, `.secondary`, `.contrast`, `.outline`, …) behind `@if $enable-classes` (often combined with the module check) so the classless builds drop them. Put `!important` behind `$enable-important` and transitions behind `$enable-transitions`.
- Use the Sass module system only (`@use`/`@forward`, `sass:map`, `sass:color`, `sass:string`). Upstream already removed deprecated global functions and color functions such as `map-get` and `lighten` to silence Dart Sass warnings, so don't reintroduce them.
- Do not hand-sort declarations. `yarn lint` reorders them (SMACSS) and reformats, so expect it to rewrite your edits.
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `refactor:`, `chore: build`, …), matching upstream history.

## Known upstream quirks

These bugs are inherited from upstream. Keep them in mind; fix them only when asked, because fixing them changes the generated CSS:
- `scss/utilities/_reduce-motion.scss:4`: `enable-transitions and enable-important` has no `$`, so it reads as truthy strings and the flags are ignored.
- `scss/content/_table.scss:52`: same problem, `@if enable-classes` has no `$`.
- `scripts/build-themes.js`: the TTY-only `clearLine`/`cursorTo` calls described above.
