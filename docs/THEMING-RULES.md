# Theming rules & conventions

The non-obvious rules we follow in this repo. Read this before
submitting a PR — most review feedback is one of these.

---

## 1. The matugen palette is the only source of color

**Never hardcode a hex value in a `.template` file.** The whole point
is that one wallpaper change should re-tint every site. A hardcoded
`#fff` is a bug.

```css
/* BAD */
.box { background: #261e18; }
.box { background: #2a1f17; }  /* "I tweaked it" */

/* GOOD */
.box { background: var(--matugen-bg-dark); }
```

The only allowed hardcoded colors:

- `transparent` — for layered surfaces.
- `currentColor` — for SVG fills.
- The Catppuccin rose `#bf616a` for danger — explicitly kept outside
  the matugen palette because every palette has a different idea of
  what "danger" means.

## 2. Use `color-mix()` for derived shades, not hand-picked hex

Hover states, subtle accents, tinted backgrounds — all should be
derived from the matugen variables:

```css
/* BAD */
.hover { background: #50453a; }
.subtle-accent { background: rgba(252, 185, 116, 0.12); }

/* GOOD */
.hover { background: var(--matugen-bg-light); }
.subtle-accent {
  background: color-mix(in srgb, var(--matugen-accent) 12%, transparent);
}
```

This way, a warm-palette accent tints the hover to a warm 12% —
visually consistent. A cool-palette accent tints to a cool 12%.

## 3. The 3-tier surface model

| Tier | Variable          | Use for                                  |
| ---- | ----------------- | ---------------------------------------- |
| 0    | `--matugen-bg`    | Page background (the dark abyss)         |
| 1    | `--matugen-bg-dark` | Card surfaces, raised content, panels |
| 2    | `--matugen-bg-light` | Hover, focus, "more elevated" state   |

Things that should **not** be on tier 1:

- The page body
- Feed / list containers
- Sidebar columns
- Layout wrappers (`.Layout`, `.application-main`, etc.)

Things that **should** be on tier 1:

- A card showing a pinned repo
- A file tree row
- A code viewer
- A modal/dialog
- A dropdown menu
- A search input

Things that should be on tier 2:

- Hovered rows
- Focused inputs
- Active nav items
- Tooltip backgrounds
- Selected checkboxes

**Tier 0 → tier 1 is the page→card distinction.** Use it sparingly.
**Tier 1 → tier 2 is the resting→active distinction.** Use it freely.

## 4. Action buttons get `bg-dark` and no border

The biggest visual mistake in most "dark themes" is the bordered
button:

```
┌─[ Fork ]─┐    <- hideous double-bordered button
```

We don't do that. Action buttons get:

- `background-color: var(--matugen-bg-dark)`
- `border: none`
- `color: var(--matugen-fg)`
- On hover: `background-color: var(--matugen-bg-light)`
- Primary actions (the "Sign in" button, the "Submit" button): accent
  bg, `bg-dark` text, no border.

This applies to: `<button>`, `[role="button"]`, `.btn`, `.Button`,
and any Primer-`prc-Button-*` element.

## 5. Accent is for accents, not surfaces

The accent color is for:

- Links (`<a>`, `.link`)
- Focus rings (`outline`)
- Selected checkboxes / radio buttons (the fill, not the bg)
- Tab indicators (the underline)
- Mention / highlight (e.g. `@user` in a comment)
- The "kbd" key indicator in a search box

The accent color is **not** for:

- Button backgrounds (use `bg-dark`)
- Hover states (use `bg-light`)
- Card surfaces (use `bg-dark`)
- The page background (use `bg`)
- Body text (use `fg`)

If you find yourself setting `background: var(--matugen-accent)` on
a non-primary button, you're probably doing it wrong.

## 6. No rounded corners

```css
* {
  border-radius: 0 !important;
}
```

This is a flat / sharp theme. If the site defaults to `border-radius:
8px` on its cards, override to `0`. We do not ship "pill" buttons or
"rounded" cards.

The exception is small visual elements that are *supposed* to be
round: avatars, status dots, the user-status circle badge, circular
icons. Those get a `border-radius: 50%` and only on the specific
selector, not via `*`.

## 7. Smooth transitions, but not on `all`

The repo's `userContent.github.template` defines a global transition
on `*` for `background-color`, `color`, `border-color`, `fill`,
`stroke`, and `box-shadow`. Do not extend this to other properties.

```css
/* BAD — would animate transform on spinners, opacity on dropdowns */
* { transition: all 0.35s ease; }

/* BAD — would animate width on flex items, margin on layout */
* { transition: background-color 0.35s ease, width 0.35s ease, margin 0.35s ease; }

/* GOOD — match the existing list */
* { transition: background-color 0.35s ease, color 0.35s ease, border-color 0.35s ease; }
```

## 8. Hashed class names → wildcards

When the site uses CSS-in-JS and emits class names like
`prc-Button-ButtonBase-HASH` or `sc-1a2b3c-0`, the **module/prefix
part is stable**, the hash is not. Use a wildcard:

```css
/* BAD — breaks on next deploy */
.prc-Button-ButtonBase-x7y2k { ... }

/* BAD — matches nothing if the hash has a different shape */
.prc-Button-ButtonBase { ... }  /* wait, this actually works */

/* GOOD */
[class*="prc-Button-ButtonBase"] { ... }
```

The explicit `[class*="..."]` form is preferred over the bare class
match because the latter also matches unrelated elements that happen
to have the same module name in a different context (e.g. an icon
component named `prc-Button-ButtonBase`). The wildcard selector is
unambiguous about what it's targeting.

## 9. Specificity: prefer `:where()` for the base overrides

If your override might conflict with a site rule, use `:where()` to
keep specificity at 0:

```css
:where(.some-class) { ... }
```

Then add a wildcard suffix for higher specificity when needed:

```css
[class*="prc-Button"]:where(.default) { ... }
```

The combined selector has specificity `(0, 2, 0)`, which is enough to
beat most site's `!important`-less rules.

If the site uses `!important` (some Primer rules do), you need
`!important` too. There's no way around it — `!important` cascades by
source order, so put your override **last** in the file.

## 10. The danger color

The matugen palette doesn't have a red. Sites that need a danger
color (closed PRs, error toasts, destructive buttons) get a fixed
rose:

```css
--color-danger-fg:     #bf616a !important;
--color-danger-emphasis: #a14750 !important;
--color-danger-subtle: color-mix(in srgb, #bf616a 15%, transparent) !important;
```

This is a deliberate choice. The Catppuccin red, the Gruvbox red, the
Nord red, and the Matugen red (which varies by wallpaper) all disagree
on what "danger" should look like. Picking a neutral rose that reads
as "warning" on any palette is the lowest-friction default.

If you want to override this per-palette, add a `danger` key to the
matugen JSON and bind it to a `--matugen-danger` variable. (Planned.)

## 11. Don't theme what you don't need to

A common mistake is to override every single element on the page
"just in case". This:

- makes the file huge (the GitHub file is 3700+ lines because of
  this; it shouldn't be),
- causes cascade conflicts with the site's own responsive rules,
- means more rules to update when the site changes.

A good rule: override only elements that are visible by default on
the page you're testing. If an element is `display: none` until you
hover something, you can skip it — GitHub's own styles will apply
when it becomes visible, and if the colors are wrong *then*, add the
rule.

The GitHub file has a section header comment for each major area it
covers (profile, dashboard, repo page, etc.) — read the comments to
see what's been considered.

## 12. Use the section dividers

When adding to a per-site template, add a section comment header:

```css
/* ============================================================
 * YOUTUBE — player page
 * ============================================================ */
```

This makes diffs readable. When you remove a section, also remove its
header. When you split a section (e.g. the "file tree" became "file
tree + file preview + file nav bar"), update the header to list the
sub-areas.

The repo's GitHub file uses the convention:

```
* AREA NAME — short description
```

at the start of each major block. Keep them in order: top-of-page
elements first, sidebar elements next, content elements last, modals
last of all.
