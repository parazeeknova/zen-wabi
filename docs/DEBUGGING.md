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
| `Loaded userstyles[github]: N bytes`        | Per-site userstyles loaded.         |
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
