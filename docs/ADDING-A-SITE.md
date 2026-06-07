# Adding a new site

There are **two ways** to add theming for a new website. Most users
only need the first one.

## Quick path: Zen Boost universal tint (0 minutes)

The bridge auto-creates a **Zen Boost** for every http(s) domain you
visit. The boost enables Zen's built-in C++ color-boost layer and
points it at your matugen accent. The result: every website gets a
hue-tinted version of itself that swaps within ~3 seconds of a
wallpaper change.

**You don't have to do anything.** Just visit a site. On the next
poll cycle, `~/.config/zen/<profile>/zen-boosts.json` will gain an
entry for the domain, and the page will be tinted.

If you want to remove the tint for a specific domain, open
`about:config` → search `zen.boosts.` → delete the entry, or use
Zen's built-in Boosts UI (hamburger menu → Boosts → toggle off).

> The universal tint is a quick, "good enough for everything" layer.
> It does **not** restyle buttons, surfaces, borders, or text
> contrast. It only changes the hue of the existing colors. For
> sites where you want full theme control (dark mode enforcement,
> custom hover states, hidden chrome elements), use the detailed
> path below.

## Detailed path: per-site CSS template (hours)

For sites where the universal tint isn't enough — dark backgrounds,
re-themed buttons, hidden UI noise, custom scrollbars — write a
`matugen-userstyles-<site>.css` template. The GitHub file
(`userContent.github.template`) is the reference implementation —
copy its style, copy its conventions.

**Status of the GitHub template: functional but a work in progress.**
Top bar, sidebar, file tree, file preview, repo header, action
buttons, PR/issue list, Copilot chat, search suggestions, and the
profile-side vcard are all themed and look correct. Specific
properties still leak through on dashboard "Pinned" cards, certain
dropdowns, and niche admin pages. We're actively finetuning.
Pull requests welcome — see the conventions in
[THEMING-RULES.md](THEMING-RULES.md) before submitting.

The expected outcome: a `userContent.<site>.template` file that,
when rendered by `theme_switcher`, produces
`matugen-userstyles-<site>.css` in your Zen chrome dir. The bridge
detects it, pushes the contents into a Zen Boost's `customCSS`
field (registered as `AGENT_SHEET`), and the page re-themes itself.

---

## 1. Pick a site and a hostname suffix

The bridge matches by **longest suffix** on the document hostname. If
you ship a file for `github.com`, it will match `github.com`,
`gist.github.com`, `pages.github.com`, and `*.github.com`.

| File matches                | What gets themed                  |
| --------------------------- | --------------------------------- |
| `matugen-userstyles-youtube.css` | `youtube.com`, `youtu.be`    |
| `matugen-userstyles-google.css`  | `google.com`, `mail.google.com` |
| `matugen-userstyles-reddit.css`  | `reddit.com`, `old.reddit.com` |

**Decide the suffix carefully.** You almost always want the registrable
domain, not a subdomain. The bridge has a per-site config table in
`BOOST_SITES` (in `matugen-bridge.uc.js`):

```js
const BOOST_SITES = {
  "github.com": {
    cssFile: "matugen-userstyles-github.css",
    options: { /* dot picker knobs, see file */ },
  },
  // add your site here
  "youtube.com": {
    cssFile: "matugen-userstyles-youtube.css",
    options: { ... },
  },
};
```

Domains not in `BOOST_SITES` get the universal Zen Boost tint and no
explicit CSS. The bridge falls through to the universal path
automatically if a `BOOST_SITES` entry's file is missing (the
log line `Per-site CSS for <domain> missing, falling back to
universal tint` confirms this).

> Edge case: some sites use different subdomains with completely
> different designs (e.g. `docs.google.com` vs `mail.google.com`).
> For those, drop a `docs.userContent.google.template` and a
> `mail.userContent.google.template` separately. Add a matching
> `BOOST_SITES` entry for each subdomain.

## 2. Pull the site's design tokens

Open the site, open DevTools → Elements → `<html>`, and read the CSS
variables it sets. Modern sites have them on `:root`:

```js
// in the DevTools console
getComputedStyle(document.documentElement)
  .getPropertyValue('--color-canvas-default');
```

Most sites will give you 5–50 variables. Group them by what they map
to in the matugen palette:

| Site token family       | matugen variable      | Notes                       |
| ----------------------- | --------------------- | --------------------------- |
| `--color-canvas-*`      | `--matugen-bg-dark`   | surface / card backgrounds  |
| `--color-bg-*`          | `--matugen-bg-dark`   | alt naming                  |
| `--color-fg-*`          | `--matugen-fg`        | text                        |
| `--color-text-*`        | `--matugen-fg`        | alt naming                  |
| `--color-accent-*`      | `--matugen-accent`    | links, primary actions      |
| `--color-brand-*`       | `--matugen-accent`    | alt naming                  |
| `--color-border-*`      | `--matugen-bg-light`  | subtle dividers             |
| danger / red            | rose `#bf616a`        | **do not theme**            |

A starter template:

```css
:root,
html {
  /* Surfaces */
  --color-canvas-default: var(--matugen-bg-dark) !important;
  --color-canvas-subtle:  var(--matugen-bg) !important;
  --color-canvas-inset:   var(--matugen-bg-dark) !important;

  /* Text */
  --color-fg-default: var(--matugen-fg) !important;
  --color-fg-muted:   var(--matugen-fg-light) !important;
  --color-fg-subtle:  var(--matugen-fg-light) !important;

  /* Accent */
  --color-accent-fg:       var(--matugen-accent) !important;
  --color-accent-emphasis: var(--matugen-accent) !important;
  --color-accent-muted:    color-mix(in srgb, var(--matugen-accent) 35%, transparent) !important;
  --color-accent-subtle:   color-mix(in srgb, var(--matugen-accent) 12%, transparent) !important;
}
```

> Use `color-mix(in srgb, var(--matugen-accent) X%, transparent)` to
> derive hover / subtle states. Don't hardcode hex.

## 3. Override the structural rules

The token overrides get you ~60% of the way. The rest is per-element
overrides for things the site styles directly (not via tokens).

```css
body {
  background-color: var(--matugen-bg) !important;
  color: var(--matugen-fg) !important;
}

a {
  color: var(--matugen-accent) !important;
}

input, textarea, select {
  background-color: var(--matugen-bg-dark) !important;
  color: var(--matugen-fg) !important;
  border-color: var(--matugen-bg-light) !important;
}

/* buttons */
button, [role="button"] {
  background-color: var(--matugen-bg-dark) !important;
  color: var(--matugen-fg) !important;
  border: none !important;
}
button:hover, [role="button"]:hover {
  background-color: var(--matugen-bg-light) !important;
}
```

## 4. Hunt for hashed class names

Many sites use CSS-in-JS that produces class names like
`sc-1a2b3c-0` or `emotion-1xyz`. These are **stable per build** but
change between deploys. Use the `[class*="prefix"]` wildcard:

```css
/* instead of */
.sc-1a2b3c-0 { ... }

/* write */
[class*="sc-1a2b3c"] { ... }
```

The wildcard still requires the *prefix* to be stable. If the prefix
itself rotates, you're out of luck — but in practice most CSS-in-JS
libraries keep the prefix stable and only change the hash.

For GitHub's `prc-ModuleName-HASH`, we use `[class*="prc-ModuleName"]`
(the module name is the stable part, the hash rotates daily).

## 5. Layout & philosophy

- **Surfaces blend to the page bg.** Don't make every box a card.
  Only true content cards (the ones that are clearly raised — pinned
  items, file rows, code blocks) get a `bg-dark` tint.
- **Action buttons = `bg-dark` bg, no border.** The bg difference is
  the affordance. Adding a border on top doubles the noise.
- **Hover = `bg-light`.** Don't use accent on hover; accent on
  hover reads as "this is the active tab" and confuses the user.
- **No rounded corners.** This is a flat / sharp theme. Set
  `border-radius: 0` on every styled element.
- **Don't tint surfaces with accent.** A 12% accent is fine for
  selected states; 30% accent on a card looks like a button.

## 6. Test on real pages

Don't just theme the homepage. Test on:

- [ ] the homepage / feed
- [ ] an account / profile page
- [ ] a content page (article, repo, video, post)
- [ ] a search / filter results page
- [ ] a settings page
- [ ] a modal / dialog
- [ ] the dark mode / light mode toggle behavior

For each: open DevTools → Elements → confirm `var(--matugen-...)` is
winning the cascade. If a `!important` from the site is winning, you
have two options:

1. Increase specificity by chaining selectors: `body button.foo { ... }`
2. Use `:where()` to keep your override at specificity 0 and re-add
   it later with `[class*="..."]`.

## 7. Submit a PR

1. Add `sites/<sitename>.userstyles.template` to the repo.
2. If you need a new hostname suffix that doesn't exist yet, edit the
   bridge's hostname table (see the `// HOSTNAME_TABLE` comment in
   `fx-autoconfig/profile/chrome/JS/matugen-bridge.uc.js`).
3. Add a one-line entry to the README's "Features" section.
4. Add a "Done" entry to the roadmap.
5. Attach before/after screenshots in the PR description.

---

## Reference: the matugen palette

The 8 variables every site file should reference:

| Variable           | Role                  | Typical value (gruvbox_chan) |
| ------------------ | --------------------- | ---------------------------- |
| `--matugen-accent` | Links, focus, primary | `#fcb974`                    |
| `--matugen-bg`     | Page background       | `#19120c` (darkest)          |
| `--matugen-bg-dark`| Card / surface        | `#261e18`                    |
| `--matugen-bg-light` | Hover / elevated     | `#50453a`                    |
| `--matugen-fg`     | Primary text          | `#eee0d5`                    |
| `--matugen-fg-light` | Secondary / muted    | `#cdb89e`                    |
| `--matugen-secondary` | Alt accent / dim    | `#3a3027`                    |
| `--matugen-tertiary`  | Borders / dividers  | `#5a4c40`                    |

The relationship is: `bg` (page) ⊂ `bg-dark` (card) ⊂ `bg-light` (raised)
in the lightness ordering **for dark palettes**. Light palettes
invert this — the bridge will normalize them in a future version, but
for now, design for the dark-palette case and verify on light.
