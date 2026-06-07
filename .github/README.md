# zen-wabi

![wabi](./assets/wabi.png)

> **Matugen-driven dynamic theme for [Zen Browser](https://zen-browser.app/) — wallpaper-aware, per-site, hot-reloadable.**

`zen-wabi` turns the matugen palette generator into a live theming engine
for Zen Browser. The same wallpaper-switcher event that re-tints your
terminal, status bar, and launcher also re-tints every tab you have open —
including per-site overrides for sites like GitHub that have their own
design system.

The repo ships three layers:

1. **Browser chrome** — Zen's own UI (`userChrome.css` + `userContent.css`).
2. **Per-site userstyles** — content CSS injected into matching hostnames
   via Zen Boosts' `customCSS` field. Currently: `github.com` and
   subdomains.
3. **Universal tint** — every visited http(s) domain gets a Zen Boost
   that drives Zen's C++ color-boost layer from the matugen accent.

Both layers read `--matugen-*` CSS variables, so a single palette change
fades the entire browser from one look to another.

---

## Table of contents

- [zen-wabi](#zen-wabi)
  - [Table of contents](#table-of-contents)
  - [How it works](#how-it-works)
    - [Why a JSWindowActor?](#why-a-jswindowactor)
  - [Features](#features)
  - [Requirements](#requirements)
  - [Installation](#installation)
    - [1. Clone](#1-clone)
    - [2. Deploy templates](#2-deploy-templates)
    - [3. Install fx-autoconfig](#3-install-fx-autoconfig)
    - [4. Enable the experimental actor runtime](#4-enable-the-experimental-actor-runtime)
    - [5. Restart Zen](#5-restart-zen)
    - [6. Trigger a wallpaper change](#6-trigger-a-wallpaper-change)
  - [Usage](#usage)
    - [Switching themes](#switching-themes)
    - [Per-site toggling](#per-site-toggling)
    - [Per-profile](#per-profile)
    - [Logs](#logs)
  - [Repository layout](#repository-layout)
  - [Risks \& limitations](#risks--limitations)
    - [Hard risks](#hard-risks)
    - [Soft limitations](#soft-limitations)
    - [Known visual nits](#known-visual-nits)
  - [Roadmap / TODOs](#roadmap--todos)
    - [Short term](#short-term)
    - [Medium term](#medium-term)
    - [Long term](#long-term)
    - [Done (recent)](#done-recent)
  - [Contributing](#contributing)
    - [Quick start](#quick-start)
    - [Adding a new site](#adding-a-new-site)
    - [Theming rules \& conventions](#theming-rules--conventions)
    - [Debugging tips](#debugging-tips)
  - [License](#license)

---

## How it works

```
┌──────────────────┐    JSON     ┌──────────────────┐
│  wallpaper       │────────────▶│  theme_switcher  │
│  switcher (QML)  │             │  (Rust binary)   │
└──────────────────┘             └────────┬─────────┘
                                           │ renders templates
                                           ▼
                           ┌──────────────────────────────┐
                           │  ~/.config/zen/<profile>/    │
                           │  chrome/                     │
                           │    matugen-vars.json         │
                           │    matugen-userstyles.css    │
                           │    matugen-userstyles-       │
                           │         github.css           │
                           └────────┬─────────────────────┘
                                    │ mtime watcher
                                    ▼
                           ┌──────────────────────────────┐
                           │  matugen-bridge.uc.js        │
                           │  (fx-autoconfig chrome side) │
                           │   • sets prefs (8 vars)      │
                           │   • updates :root in chrome  │
                           │   • broadcasts to actors     │
                           │   • syncs per-site CSS into  │
                           │     Zen Boosts (customCSS)   │
                           │   • creates universal Zen    │
                           │     Boosts for visited       │
                           │     http(s) domains          │
                           │   • syncs workspace gradient │
                           │     or HSL fallback          │
                           └────────┬──────────┬──────────┘
                                    │          │
                            actor   │          │ Zen Boosts API
                          messages  │          ▼
                                    │  ┌──────────────────────┐
                                    │  │  ZenBoostsParent     │
                                    │  │   • reads customCSS  │
                                    │  │   • registers as     │
                                    │  │     AGENT_SHEET      │
                                    │  │   • drives C++       │
                                    │  │     color-boost      │
                                    │  └────────┬─────────────┘
                                    │           │
                          ┌─────────▼──┐    ┌──▼──────────────┐
                          │ Matugen    │    │ ZenBoostsChild  │
                          │ Child      │    │  (per process)  │
                          │ • :root    │    │  • applies CSS  │
                          │ • userCSS  │    │  • applies tint │
                          └────────────┘    └─────────────────┘
```
┌──────────────────┐    JSON     ┌──────────────────┐
│  wallpaper       │────────────▶│  theme_switcher  │
│  switcher (QML)  │             │  (Rust binary)   │
└──────────────────┘             └────────┬─────────┘
                                          │ renders templates
                                          ▼
                          ┌──────────────────────────────┐
                          │  ~/.config/zen/<profile>/    │
                          │  chrome/                     │
                          │    matugen-vars.json         │
                          │    matugen-userstyles.css    │
                          │    matugen-userstyles-       │
                          │         github.css           │
                          └────────┬─────────────────────┘
                                   │ mtime watcher
                                   ▼
                          ┌──────────────────────────────┐
                          │  matugen-bridge.uc.js        │
                          │  (fx-autoconfig chrome side) │
                          │   • sets prefs (8 vars)      │
                          │   • updates :root in chrome  │
                          │   • broadcasts to actors     │
                          └────────┬─────────────────────┘
                                   │ JSWindowActor messages
                                   ▼
                          ┌──────────────────────────────┐
                          │  MatugenChild.sys.mjs        │
                          │  (per content process)       │
                          │   • re-injects userstyles    │
                          │   • updates :root on doc     │
                          │   • MutationObserver for     │
                          │     lazy / SPA-loaded nodes  │
                          └──────────────────────────────┘
```

The `theme_switcher` binary is the only piece this repo **does not**
ship — it is built from your wallpaper switcher's source tree and is
expected to know how to read the palette JSON and call
`render_template()`. A reference implementation lives in the
[dotfiles](https://github.com/parazeeknova/doty) used to develop this
project; see the [Contributing](https://github.com/parazeeknova/zen-wabi/blob/main/docs/ADDING-A-SITE.md) section for the
template-rendering contract.

### Why a JSWindowActor?

Firefox's `userContent.css` is unreliable in Zen ≥ 1.20 — it is gated
behind a pref that often resets, and `@-moz-document` rules only work
inside `userContent.css`, not inside stylesheets dynamically injected
into the content process. The actor approach:

- injects CSS as a real `<style>` element in the content document,
- lets us match hostnames from the parent side (no hardcoded URL
  prefixes in the CSS itself),
- gets a clean re-injection point on theme change with no page reload,
- survives Fission (multi-process) — each content process has its own
  `MatugenChild`.

For per-site CSS injection we now **prefer Zen Boosts' `customCSS`
field** over the actor — Zen handles the `AGENT_SHEET` registration
and Fission correctly out of the box. The actor is kept for the
`:root` variable push (which Zen Boosts don't expose a public API
for) and as a fallback for per-site content.

### Why Zen Boosts for the universal tint?

Zen's C++ color-boost layer is what actually tints pixel content on
a page (think: an HSL hue shift applied as a paint pass). It reads
from the active Zen Boost's dot-picker knobs (HSL) or from
`workspace.theme.gradientColors[primary].c` when `autoTheme: true`.
By creating one boost per visited domain, we get this tint for free
on every site — no per-site CSS template needed.

---

## Features

- **Hot reload, no restart.** Change wallpapers and every open tab
  fades from the old palette to the new one in 0.35s. No reload, no
  flicker.
- **Universal Zen Boost tint (quick path).** Every visited http(s)
  domain auto-gets a Zen Boost that drives Zen's C++ color-boost
  layer from your matugen accent. Zero config — just visit a site.
  Hot-swaps on every wallpaper change.
- **Per-site overrides (detailed path).** The bridge ships
  `matugen-userstyles-<site>.css` per supported site. Contents are
  pushed into the site's Zen Boost `customCSS` field (registered
  as `AGENT_SHEET`). Currently: `github.com` and subdomains.
  **GitHub coverage is functional but a work in progress** — most
  pages look right, but specific elements (Pinned cards on the
  dashboard, some dropdowns, niche admin pages) still leak the
  default Primer styling. Pull requests welcome.
- **No hardcoded CSS class names** for Primer/React components. Every
  `prc-ModuleName-HASH` selector is written as `[class*="prc-ModuleName"]`
  so GitHub's nightly deploys don't break the theme.
- **Smooth color transitions.** 0.35s ease on `background-color`,
  `color`, `border-color`, `fill`, `stroke`, and `box-shadow` for every
  element. Looks like a real desktop theme, not a flash-cut.
- **No "card look" leak.** Surfaces blend to the page background
  (`--matugen-bg`); only true content cards (pinned items, file tree
  rows, code viewer) get the lighter `--matugen-bg-dark` tint.
- **Catppuccin-faithful accents.** The 26 Catppuccin colors are mapped
  to 8 matugen variables; the danger color stays a neutral rose that
  reads correctly on both warm and cool palettes.
- **Per-profile isolation.** Each Zen profile under
  `~/.config/zen/<profile>/chrome/` is independent. You can theme one
  profile for work and another for personal without cross-talk.
- **Workspaces-aware.** With Zen Workspaces enabled, the bridge
  pushes the accent into your active workspace's gradient palette
  and fires `zen-space-gradient-update` so the C++ layer re-tints.
  With Workspaces disabled, it falls back to writing the accent as
  HSL (`dotAngleDeg`/`saturation`/`brightness`) directly on each
  universal boost.

---

## Requirements

| Tool         | Version  | Notes                                    |
| ------------ | -------- | ---------------------------------------- |
| Zen Browser  | ≥ 1.20   | Earlier builds lack Fission-safe actors. |
| matugen      | latest   | Generates the JSON palette.              |
| A matugen-   | any      | Hyprland, Quickshell, river, sway, etc.  |
| aware WM     |          |                                          |
| fx-autoconfig| latest   | Provides the `JSWindowActor` runtime.    |
| Rust (build) | stable   | Only needed if rebuilding `theme_switcher`. |

You also need a wallpaper that goes through matugen — the color
extraction is the palette source. If your WM doesn't run matugen yet,
see [Adding a new site](#adding-a-new-site) for the JSON contract and
adapt any external palette source.

---

## Installation

### 1. Clone

```sh
git clone https://github.com/parazeeknova/zen-wabi.git ~/Repository/zen-wabi
```

### 2. Deploy templates

The repo ships `*.template` files. Either:

- copy them into your existing `~/.config/zen/` and let your
  `theme_switcher` render them on next wallpaper change, **or**
- symlink them so template edits are live:

  ```sh
  ln -sf ~/Repository/zen-wabi/userChrome.css.template \
         ~/.config/zen/userChrome.css.template
  ln -sf ~/Repository/zen-wabi/userContent.css.template \
         ~/.config/zen/userContent.css.template
  ln -sf ~/Repository/zen-wabi/userContent.github.template \
         ~/.config/zen/userContent.github.template
  ln -sf ~/Repository/zen-wabi/user.js.template \
         ~/.config/zen/user.js.template
  ```

### 3. Install fx-autoconfig

The `fx-autoconfig/` directory in this repo is a **drop-in** for the
`fx-autoconfig` Zen distribution layout:

```sh
# merge with the rest of your fx-autoconfig config
cp -r fx-autoconfig/* /opt/zen-browser-bin/
```

If you use a different fx-autoconfig layout, just merge the contents
of `fx-autoconfig/profile/chrome/JS/Matugen/` and the bridge file into
your existing `chrome/JS/` directory.

### 4. Enable the experimental actor runtime

Add to `~/.config/zen/<profile>/user.js`:

```js
userChromeJS.experimental.enabled = true;
```

(The `user.js.template` already does this — merge it into your
existing `user.js` if you have one.)

### 5. Restart Zen

```sh
pkill -9 zen-bin
/opt/zen-browser-bin/zen-bin &
```

Tail the bridge log to confirm boot:

```sh
tail -f ~/.config/zen/<profile>/chrome/matugen-bridge.log
```

You should see:

```
[matugen-bridge] [INFO] SCRIPT TOP  version 1.4
[matugen-bridge] [INFO] Registered Matugen JSWindowActor
[matugen-bridge] [INFO] Watching: .../matugen-userstyles*.css
```

### 6. Trigger a wallpaper change

Switch your wallpaper through whatever mechanism your WM exposes
(Hyprland `hyprpaper`, `swaybg`, `wpaperd`, `wallutils`, etc.). The
switcher should call:

```sh
theme_switcher wallpaper /path/to/wallpaper.jpg
```

Every open GitHub tab will fade from the old palette to the new one
over ~0.35s.

---

## Usage

### Switching themes

Theme changes are driven by the wallpaper event — there is no manual
"rebuild CSS" step. If you want to test a palette without changing
wallpaper:

```sh
theme_switcher palette /path/to/matugen-vars.json
```

The JSON must have these keys:

```json
{
  "accent":      "#fcb974",
  "bg":          "#19120c",
  "bg_dark":     "#261e18",
  "bg_light":    "#50453a",
  "fg":          "#eee0d5",
  "fg_light":    "#cdb89e",
  "secondary":   "#3a3027",
  "tertiary":    "#5a4c40"
}
```

### Per-site toggling

To disable a site theme without removing the file, rename it to drop
the `.css` extension — the bridge only loads `matugen-userstyles*.css`.

### Per-profile

The bridge auto-detects the running profile by reading its own
location at startup. Two Zen profiles running side-by-side are
fully isolated: each has its own `matugen-vars.json` watcher and
its own set of injected styles.

### Logs

```sh
tail -f ~/.config/zen/<profile>/chrome/matugen-bridge.log
```

The log is overwritten on every restart (the bridge opens it with
`O_TRUNC`). The actor child re-forwards its log lines to the parent
via `sendSyncMessage`, so this single file is the only place you need
to look.

---

## Repository layout

```
zen-wabi/
├── README.md                        ← you are here
├── .gitignore
├── userChrome.css.template          ← Zen chrome (9 sections)
├── userContent.css.template         ← global :root vars + about:*
├── userContent.github.template      ← github.com / *.github.com
├── user.js.template                 ← required Zen prefs
├── fx-autoconfig/
│   ├── program/
│   │   ├── config.js                ← fx-autoconfig entry
│   │   └── defaults/pref/
│   │       └── config-prefs.js
│   └── profile/chrome/
│       ├── utils/                   ← fx-autoconfig runtime
│       │   ├── boot.sys.mjs
│       │   ├── chrome.manifest
│       │   ├── fs.sys.mjs
│       │   ├── module_loader.mjs
│       │   ├── uc_api.sys.mjs
│       │   └── utils.sys.mjs
│       └── JS/
│           ├── matugen-bridge.uc.js ← theme_switcher ↔ actors
│           └── Matugen/
│               ├── MatugenParent.sys.mjs
│               └── MatugenChild.sys.mjs
├── docs/                            ← per-site how-tos (see Contributing)
│   ├── ADDING-A-SITE.md
│   ├── THEMING-RULES.md
│   └── DEBUGGING.md
└── sites/                           ← per-site userstyles (planned)
    ├── github.userstyles.template
    └── ...
```

---

## Risks & limitations

> **Read this section before reporting an issue.** Most "the theme
> broke" reports are one of these.

### Hard risks

- **fx-autoconfig required.** Without it, the bridge script never
  loads and the actor never registers. Zen 1.20 ships without
  autoconfig enabled by default — install it from
  [https://github.com/MrOtherGuy/fx-autoconfig](https://github.com/MrOtherGuy/fx-autoconfig).
- **`userChromeJS.experimental.enabled = true` must be set.** The
  JSWindowActor runtime is gated behind this pref. If it's missing,
  the bridge logs `ChromeUtils.registerWindowActor is not a function`.
- **CSS variables don't cross document boundaries.** Chrome vars
  (`:root` in `userChrome.css`) are not visible in content documents.
  That's why we re-set `--matugen-*` on `html[data-color-mode]` in
  `userContent.github.template` — a second copy of the variables.
- **`@-moz-document` is ignored in injected `<style>` elements.** It
  only works in `userContent.css`. Per-site stylesheets are filtered
  by hostname **on the parent side** before the actor ever sees them.
- **Single `export class` per ESM file.** Defining `MatugenChild`
  twice in the same file (e.g. once at the top, once in an alternate
  import path) throws `SyntaxError: Identifier 'MatugenChild' has
  already been declared` at MODULE LOAD — not at first use.
- **CSS in `userContent.css` is unreliable on Zen 1.20.1b.** The
  pref exists but doesn't always load. We compensate by injecting
  the same rules via the actor.

### Soft limitations

- **GitHub coverage is a work in progress.** The template is
  3700+ lines and covers most pages, but specific elements still
  leak the default Primer styling — dashboard Pinned cards, some
  dropdown menus, niche admin pages. The universal Zen Boost tint
  handles these cases gracefully (it tints *every* site), but the
  explicit per-site CSS is under active finetuning. PRs welcome.
- **GitHub Primer hashes can rotate.** Today the rule is
  `[class*="prc-Button-ButtonBase"]`. If GitHub renames the
  module, the wildcard still matches (it falls through to the
  prefix), but a renamed module needs a new rule.
- **Universal Zen Boost creates a boost per domain.** Every http(s)
  domain you visit gets a boost entry in `zen-boosts.json`. Over
  months of browsing this can grow to hundreds of entries. Zen
  handles this fine (the JSON is re-read on every boost lookup)
  but if you're a privacy-sensitive user, prune the file
  periodically.
- **Universal tint is hue-only.** It does not restyle surfaces,
  borders, or text contrast. It changes the hue of whatever the
  page already shows. For sites where the default is white-on-white
  in light mode, the tint won't fix readability — use a per-site
  template for those.
- **No form control theming in `userChrome.css`.** `<input
  type="checkbox">` etc. would leak styling to every website. The
  GitHub ruleset themes the in-content checkboxes via
  `[class*="prc-Checkbox-Checkbox"]` selectors instead.
- **Two Zen profiles = two bridges.** Each profile has its own
  `chrome/` and its own actor registration. Switching wallpapers
  only refreshes the active profile's tabs.
- **Smooth transitions have a small perf cost.** `transition: *` on
  `*` is ~0.5% extra paint time on slow GPUs. Negligible on modern
  hardware, noticeable on a 2015 ThinkPad.
- **No wayland-specific features.** This is a CSS / JS layer; it
  has nothing to do with Wayland. It will work in any Zen session.

### Known visual nits

- The "Public" repo badge is the `--color-btn-primary-bg` accent on
  `--color-fg-on-emphasis` dark text. On very low-contrast palettes
  the badge may be hard to read. Fix in
  `userContent.github.template:Label--secondary` rule.
- GitHub's search filter input has a hidden measurement mirror div
  (`aria-hidden="true"`) that GitHub uses to size the input. We push
  it off-screen with `position: absolute; top: -9999px; visibility:
  hidden`. If you remove that rule, the input will show duplicate
  text.

---

## Roadmap / TODOs

### Short term

- [ ] **More sites.** Add userstyles for: YouTube, Reddit, Mastodon
      instances, Hacker News, Twitter/X, Gmail, Google Docs, Notion,
      Linear, Figma. The GitHub file is the reference implementation.
- [ ] **Light-palette first-class support.** Right now everything
      assumes `--matugen-bg` is the darkest. On a true light palette
      that breaks — `bg` is the lightest, `bg-dark` is the medium
      surface. The CSS uses `color-mix(in srgb, ...)` in a few
      places to handle this, but the rule order needs auditing.
- [ ] **CSS variable parity check.** Confirm that the chrome
      variables and the content variables don't drift. Right now
      they're duplicated in `userChrome.css.template` and
      `userContent.github.template`; a single source of truth would
      be better.
- [ ] **Auto-extract PrimeReact / Material / Chakra tokens.** Add
      a small CLI that walks a site, finds the CSS vars it actually
      uses, and emits a starter template.

### Medium term

- [ ] **Per-site config files** with per-site settings (toggle
      animations, force dark mode for the page, override the danger
      color, etc.). Move the hostname-match table out of the
      bridge into a JSON config.
- [ ] **Hover-state color extraction.** Generate accent-tinted
      hover states automatically from the base accent, instead of
      hand-writing `color-mix(in srgb, accent 88%, fg 12%)` for
      every button.
- [ ] **Wider Zen coverage.** Sidebery workspace indicator, Zen
      Glance, Zen Split View, Zen Picture-in-Picture, Zen Modals.
      These are all `userChrome.css` work.
- [ ] **A "dev mode" that pulls the live matugen JSON from the
      current wallpaper and re-applies every N seconds** so you can
      see the theme work without actually changing wallpaper.

### Long term

- [ ] **Native theming API.** Replace the `theme_switcher` shell-out
      with a D-Bus signal that the bridge listens for. Eliminates
      the file-watcher round-trip and works in non-Linux WMs.
- [ ] **Theme presets** that ship as `.json` files — "Catppuccin
      Mocha", "Gruvbox", "Nord", "Rose Pine", etc. — selectable
      independent of the wallpaper. Useful for screenshots and
      accessibility audits.
- [ ] **A webview-mode theme picker** — a Zen sidebar panel that
      lets you switch palettes without leaving the page you're
      theming.
- [ ] **A Figma plugin** that exports a site's design tokens into
      the matugen JSON format.

### Done (recent)

- [x] Matugen CSS variables on `:root` for both chrome and content.
- [x] JSWindowActor-based content injection (survives Fission).
- [x] **Zen Boost universal tint.** Every visited http(s) domain
      auto-gets a Zen Boost driving the C++ color-boost layer from
      the matugen accent. Hot-swaps on every wallpaper change.
- [x] **Zen Boost per-site CSS.** Per-site templates are pushed
      into the site's Zen Boost `customCSS` field (AGENT_SHEET),
      eliminating the need for our own actor to handle per-site
      injection.
- [x] **Workspace gradient sync.** Bridge pushes the matugen
      accent into `gZenWorkspaces.getActiveWorkspace().theme.gradientColors`
      and fires `zen-space-gradient-update`. With workspaces
      disabled, falls back to HSL on each universal boost.
- [x] **Rename-to-disable.** Renaming
      `matugen-userstyles-<site>.css` to `.css.disabled` clears
      the per-site CSS from the boost; the domain falls through
      to the universal tint.
- [x] 109 `[class*="prc-..."]` wildcards covering all Primer modules
      used on profile, dashboard, and repo pages.
- [x] File tree, file preview, file nav bar, header bar, action
      buttons, PR/issue list, filter search bar — all themed.
- [x] Smooth 0.35s color transitions on theme switch.
- [x] Transparent user-status button + circle badge.
- [x] Removed Refined GitHub heat-map coloring on `<relative-time>`.

---

## Contributing

**The main objective of this project is for contributors to add
support for new websites and to improve the GitHub coverage.** The
GitHub file is the reference implementation — copy its style, copy
its conventions, and submit a PR.

### Quick start

1. Fork this repo.
2. **Decide which path you need:**
   - **Quick path (universal Zen Boost tint)** — works for every
     site out of the box, no code changes. See the top of
     [docs/ADDING-A-SITE.md](docs/ADDING-A-SITE.md).
   - **Detailed path (per-site CSS template)** — copy the
     GitHub template as a starting point and write per-site
     overrides. See the rest of
     [docs/ADDING-A-SITE.md](docs/ADDING-A-SITE.md).
3. If you take the detailed path, add a `BOOST_SITES` entry in
   `fx-autoconfig/profile/chrome/JS/matugen-bridge.uc.js` (the
   table at the top of the file). This binds the hostname to a
   CSS file.
4. Submit a PR with:
   - the new template file (or just the bridge change if you only
     fixed a regression),
   - the corresponding `BOOST_SITES` entry,
   - a screenshot of before/after,
   - a one-line entry in this README's "Features" section.

### Adding a new site

See [docs/ADDING-A-SITE.md](docs/ADDING-A-SITE.md) for a full
walkthrough. TL;DR:

1. Generate a `.css.template` from the site's design tokens.
2. Use the 8 matugen variables — never hardcode hex.
3. Use `[class*="..."]` wildcards for hashed class names.
4. Aim for "blends to page bg" — not "every box is a card".
5. Test on at least: profile/account, dashboard/home, a content
   page, a search/filter page, and a modal.

### Theming rules & conventions

See [docs/THEMING-RULES.md](docs/THEMING-RULES.md) for the
non-obvious ones. The short version:

- **`bg` = page background, `bg-dark` = card surface, `bg-light` =
  hover/elevated.** Don't use `bg-light` for body backgrounds; it
  reads as a "raised" color.
- **No `border: 1px solid` on action buttons.** Use `bg-dark` with
  no border; the bg difference is the affordance.
- **Accent is for links, focus rings, and the accent border on
  popups.** Don't tint large surfaces with accent — it makes the
  whole page vibrate.
- **Never round the corners.** This is a sharp / flat theme. If
  the site uses `border-radius: 8px`, set it to `0`.
- **Transitions only on `background-color`, `color`,
  `border-color`, `fill`, `stroke`, `box-shadow`.** Don't add
  `transition: all` — it animates `transform` and `opacity` on
  spinners, which is wrong.

### Debugging tips

See [docs/DEBUGGING.md](docs/DEBUGGING.md). The 30-second version:

```sh
# is the bridge alive?
tail -f ~/.config/zen/<profile>/chrome/matugen-bridge.log

# did the CSS file get rendered?
ls -la ~/.config/zen/<profile>/chrome/matugen-userstyles*.css

# is the actor registered? (open about:debugging → This Firefox → tabs)
# search for "MatugenChild" in the process list
```

For inspecting a specific element, use `about:devtools-toolbox` →
Inspector → click the element → look at the `Style` panel for
`background-color: var(--matugen-...)`. If a `!important` rule from
GitHub's Primer is winning, prefix your override with `:where()` to
keep specificity at 0, or use `[class*="..."]` with a longer
suffix.

---

## License

MIT. See the source headers — the original fx-autoconfig runtime
files retain their respective licenses (BSD-2-Clause for utils.*,
MPL-2.0 for the bridge).
