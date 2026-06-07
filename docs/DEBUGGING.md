# Debugging tips

The 30-second version, plus the deeper debugging for when the
30-second version isn't enough.

---

## The 30-second check

```sh
# 1. Is the bridge alive?
tail -f ~/.config/zen/<profile>/chrome/matugen-bridge.log

# 2. Did the CSS file get rendered?
ls -la ~/.config/zen/<profile>/chrome/matugen-userstyles*.css

# 3. Did the vars get applied to chrome?
grep "matugen-accent" ~/.config/zen/<profile>/chrome/userChrome.css

# 4. Is the actor registered?
#    Open about:debugging → This Firefox → search "MatugenChild"
```

If all four look right, the issue is in the per-site template, not
the infrastructure.

---

## Reading the bridge log

The bridge logs `[matugen-bridge]` followed by `[LEVEL]` and a
message. The interesting events:

| Log line                                     | What it means                       |
| -------------------------------------------- | ----------------------------------- |
| `SCRIPT TOP version X.Y`                     | Bridge loaded.                      |
| `chrome dir: ...`                            | Which profile it picked.            |
| `Watching: .../matugen-userstyles*.css`      | File watcher armed.                 |
| `Registered Matugen JSWindowActor`           | Actor is in.                        |
| `Observers registered for 8 prefs`           | Pref observer armed.                |
| `Wrote 8 prefs from matugen-vars.json`       | Prefs applied.                      |
| `Applied 8 vars to chrome :root`             | Chrome CSS vars applied.            |
| `Broadcast vars to N/M tab actors`           | Content CSS vars applied.           |
| `Loaded userstyles[global]: N bytes`         | Global userstyles loaded.           |
| `Loaded userstyles[<site>]: N bytes`         | Per-site userstyles loaded.         |
| `Scanned userstyles dir: N per-site file(s)` | Bridge parsed `matugen-userstyles-<site>.css` files. |
| `Synced boost[<domain>]: id=X customCSS=YB`  | Per-site CSS pushed into a Zen Boost. |
| `Userstyles[<suffix>] removed, clearing boost[<domain>].customCSS` | File renamed to `.disabled` — the boost is now empty. |
| `Per-site CSS for <domain> missing, falling back to universal tint` | The site has a `BOOST_SITES` entry but the file is gone. |
| `Universal sync: Nw/Mt (http=..., noHost=..., perSite=..., registered=..., already=...) created=N` | Periodic poll that auto-creates Zen Boosts for every visited http(s) tab. |
| `Created new active boost[<domain>]: id=X`   | First-time universal tint applied. |
| `syncWorkspaceTheme: called with accent=...` | A JSON change arrived.              |
| `syncWorkspaceTheme: accent=... → hsl(...)`  | RGB→HSL conversion succeeded.      |
| `Synced workspace gradient: N color(s) from accent ...` | Workspace gradient updated and `zen-space-gradient-update` event fired. |
| `syncWorkspaceTheme: no active workspace, falling back to direct HSL on boosts` | Zen Workspaces is disabled — HSL is applied to each universal boost instead. |
| `syncWorkspaceTheme: gZenWorkspaces not available, falling back to direct HSL on boosts` | Bridge ran before any browser window was open. |
| `Updated N boost(s) with HSL from accent ...` | HSL fallback path succeeded for N domains. |
| `Userstyles changed, broadcasting`           | Theme change detected.              |
| `Broadcast userstyles to N/M tab actors`     | Content styles re-injected.         |
| `Initial apply on startup`                   | First run after Zen start.          |

If the log is **empty** after a restart, the bridge didn't load. Check:

- `userChromeJS.experimental.enabled = true` in `user.js`
- the bridge file is at
  `~/.config/zen/<profile>/chrome/JS/matugen-bridge.uc.js`
- fx-autoconfig is installed (see the main README)

If the log says `Applied 8 vars` but the page is still unthemed, the
content actor isn't running. Check:

- the actor files exist at
  `~/.config/zen/<profile>/chrome/JS/Matugen/*.sys.mjs`
- `chrome.manifest` includes the actor registration line:
  `{ "actor": "Matugen", "jsName": "MatugenChild" }` or similar
- the actor registration code in `MatugenParent.sys.mjs` matches
  the manifest's actor name

## Forcing a reload

```sh
# Kill Zen and restart
pkill -9 zen-bin
/opt/zen-browser-bin/zen-bin &

# Touch the CSS file to force a re-render
touch ~/.config/zen/<profile>/chrome/matugen-userstyles-github.css
```

If the bridge is watching the file, `touch` is enough to trigger a
re-inject within 1 second (the polling interval). You don't need to
restart Zen for a CSS-only change.

For a per-site template change:

```sh
# Re-render from template
theme_switcher wallpaper /path/to/your/wallpaper.jpg
```

## Inspecting a specific element

Open DevTools on the affected page, pick the element, look at the
`Styles` panel. If you see:

```
background-color: var(--matugen-bg-dark) !important;
```

with a `*` selector or a wildcard, your rule is winning. If you see:

```
background-color: #0d1117 !important;  /* from Primer */
```

your rule is losing. Three options:

1. **Reorder your rule to be later in the file.** `!important`
   cascades by source order, not specificity. Move your override
   below the offending Primer rule.
2. **Increase specificity.** Use a longer selector chain:
   `body .Box .Box-body .markdown-body { ... }`.
3. **Add `!important` to your override.** Only do this as a last
   resort — it makes the cascade impossible to debug from the
   site's own DevTools.

For a quick override test in DevTools, right-click the rule, "Add
rule", and write your override directly. If it works in the
console but not in the template, you have a specificity or
`!important` issue.

## The site's dark mode

GitHub (and many other modern sites) have a built-in dark mode. The
template overrides its CSS variables, but the toggle still works.
If the user has GitHub set to "Light" mode, the site re-applies its
light-mode CSS variables on every navigation, *overriding our
template's !important rules* with its own `!important` rules.

**The fix:** the user has to set GitHub to "Dark" or "System" mode.
We can't force the site to ignore its own theme preference — the
toggle is on a different storage key than the CSS, and forcing it
would require a content-script hook (which we don't have).

If you're seeing the theme work on the homepage but not on, say, the
settings page, the user probably toggled to Light at some point. Tell
them to flip it back.

## Why is the file so big?

`userContent.github.template` is 3700+ lines. This is **normal** for
a site that uses CSS-in-JS. Each `prc-ModuleName-HASH` is one
component, and GitHub has hundreds of them.

The file is not the bottleneck. The browser parses it once and caches
the AST. The 130KB compiled CSS is parsed in <5ms on modern hardware.

If you want to reduce the size, profile first: in DevTools →
Performance → "Reload" → look at "Parse CSS" time. If it's <50ms,
the file is fine.

## Why is the theme slow to apply?

Three possibilities:

1. **First paint after navigation.** The content actor registers an
   event listener for `document-element-inserted` and injects on
   first document. There's a ~50ms delay between "page starts
   rendering" and "userstyles applied". On slow CPUs or with a
   large userstyles file, this can flash the unthemed page briefly.
2. **Hot-reload of the CSS file** triggers a `MutationObserver` that
   re-injects. This is intentionally debounced 200ms to avoid
   re-injecting on every DOM change.
3. **Smooth transitions are too long.** 0.35s is the current value.
   If you want snappier, change `--matugen-transition` in the
   template to `0.2s` or `0.15s`.

## Useful Firefox prefs for debugging

```js
// In about:config
browser.tabs.remote.autostart = true;     // Fission is on
browser.tabs.remote.desktopbehavior = 0;  // default
devtools.remote.usb.enabled = false;      // not needed
```

To see the actor's own log lines in the bridge log, ensure
`MatugenChild.sys.mjs` calls `sendSyncMessage("Matugen:log", ...)`
for the events you care about. The default already does this for
"page matched" and "re-injected".

## Asking for help

If you've checked all of the above and the theme is still broken,
open an issue with:

1. The exact URL that's broken.
2. The full bridge log from a fresh Zen start.
3. A screenshot of the broken element with DevTools → Elements
   panel open, showing the `Styles` tab for the broken element.
4. Your palette JSON (`~/.cache/matugen/colors.json` or whatever
   your wallpaper switcher writes).

Without these, we can guess — but guessing takes longer than fixing.

---

## Zen Boost flow (universal + per-site)

The bridge uses **two complementary delivery channels** for theming
web content:

1. **JSWindowActor** — pushes `--matugen-*` variables to every
   content document's `:root` and re-injects the global
   `userContent.css` rules. This is the chrome-side path.
2. **Zen Boosts** — pushes per-site CSS into Zen's own
   `boost.customCSS` field (registered as `AGENT_SHEET`) and creates
   a "universal tint" Zen Boost for every http(s) domain you visit
   (drives the C++ color-boost layer).

### How universal tints work

The bridge polls every 3 seconds. For every open http(s) tab whose
domain is **not** in `BOOST_SITES`, it auto-creates a Zen Boost with
`enableColorBoost: true, autoTheme: true, changeWasMade: true` and
adds the domain to the in-memory `universalBoostedDomains` set.

When you switch wallpaper, the bridge:
1. Tries to push the accent into your active Zen workspace's
   `theme.gradientColors` (primary) + fires
   `zen-space-gradient-update` — Zen's parent actor re-broadcasts to
   every child, the C++ layer re-tints open pages.
2. If Zen Workspaces is disabled, falls back to converting the
   accent RGB → HSL and writing `dotAngleDeg`/`saturation`/
   `brightness` directly on every known universal boost. The C++
   layer reads the same HSL fields, so the tint still updates.

The result: a hue swap on every open site within ~3 seconds of the
JSON file changing.

### How per-site overrides work

Domains in `BOOST_SITES` get a Zen Boost with
`enableColorBoost: false` (we don't want the C++ tint fighting our
explicit CSS) and `customCSS` populated from
`matugen-userstyles-<site>.css`. The boost's `AGENT_SHEET` is what
injects the rules into the content document.

To disable a per-site theme without removing the file:

```sh
mv ~/.config/zen/<profile>/chrome/matugen-userstyles-github.css \
   ~/.config/zen/<profile>/chrome/matugen-userstyles-github.css.disabled
```

On the next poll the bridge logs
`Userstyles[github] removed, clearing boost[github.com].customCSS`
and the domain falls through to the universal tint. Re-enable by
renaming back.

### Verifying the boost is registered

In `about:config`, watch the
`zen.boosts.<profile>` JSON store under
`~/.config/zen/<profile>/zen-boosts.json`. You should see one entry
per visited domain. Each entry has `changeWasMade: true` (without
this flag, the parent actor refuses to return a stylesheet —
silently).

### "no active workspace" in the log

If you see this on the first JSON change after Zen start, that's
normal — `gZenWorkspaces` initializes lazily and the first call
races it. The next poll (≤3s later) succeeds. If you see it on
**every** JSON change, Zen Workspaces is disabled in your profile —
the HSL fallback path runs instead and the log line
`Updated N boost(s) with HSL from accent ...` confirms it.

### GitHub theming is a work in progress

The `userContent.github.template` (3700+ lines) works — top bar,
sidebar, file tree, file preview, repo header, action buttons,
PR/issue list, Copilot chat, search suggestions, and the
profile-side vcard are all themed.

**It is not perfect.** Specific properties still leak through on
some pages: the dashboard "Pinned" cards, certain dropdown menus,
the branch selector's popover, and a few niche admin pages. The
universal Zen Boost tint handles these cases gracefully (it tints
*every* site, not just GitHub), but the explicit per-site CSS is
under active finetuning.

If you find a specific page element that's unthemed, open an issue
with the URL + a screenshot of DevTools → Elements → Styles for the
element. The wildcard `[class*="prc-..."]` pattern usually just
needs a new prefix entry, but on rare occasions a `!important`
Primer rule is winning the cascade and we need to re-order.

---

## Workspace feature

Zen's C++ color-boost layer reads
`workspace.theme.gradientColors[primary].c` (or, if
`autoTheme: false`, the explicit dot-picker knobs on the boost).
The bridge handles both:

- **With workspaces:** bridge writes the matugen accent into
  `gradientColors[0]` and fires `zen-space-gradient-update`. The
  parent actor re-broadcasts to every child. The C++ layer
  re-computes the tint.
- **Without workspaces:** bridge converts accent → HSL and writes
  `dotAngleDeg`/`saturation`/`brightness` directly on each
  universal boost. The C++ layer reads the same fields.

If you want to verify which path ran, look for `Synced workspace
gradient` vs `Updated N boost(s) with HSL` in the log. The
HSL-fallback path requires the universal poll to have populated
`universalBoostedDomains` — that takes one poll cycle (≤3s) on a
fresh start.
