# quickcloset.github.io — rules for this repo

This is the live marketing site (`quickcloset.co.za`) and admin portal (`admin.html`), deployed via GitHub Actions to Pages on every push to `main`. It is a separate repo from the QuickCloset app (`Quickcloset/quickcloset`) — don't confuse the two.

## Light/dark mode — check both, every time

The site supports light and dark rendering via CSS custom properties defined in `index.html`'s `<style>` block: a base `:root { --sand: ...; --ink: ...; ... }`, a `@media (prefers-color-scheme: dark) { :root { ... } }` override, and `:root[data-theme="dark"]` / `:root[data-theme="light"]` overrides for the in-page toggle. Several variables (`--sand`, `--surface`, `--text`, etc.) intentionally flip value between light and dark mode; others (`--sage`, `--ink`) stay dark/muted in both themes on purpose.

**Real incident (2026-08-29):** the "problem" section and the footer both sit on a permanently-dark photo/ink backdrop — same look in both themes, by design. But their text/border colors were set to `var(--sand)`, whose *other* job is the page background — which correctly flips to near-black in dark mode. Result: in dark mode, the text and the backdrop it sat on collapsed to the same near-black color, making headings and footer text invisible. It looked fine in light mode, so it shipped unnoticed until a user's friend opened the site in a browser set to dark mode. Fixed by introducing `--on-dark`, a fixed light value defined once in the base `:root` and never overridden — used only where text/borders sit on a permanently-dark section, instead of reusing a theme-reactive variable for a role it wasn't designed for.

**Rule going forward:** before treating any CSS color change as done —
1. Know which existing variable you're reusing actually does, in both themes. Check its value in the base `:root`, the `@media (prefers-color-scheme: dark)` block, and both `[data-theme]` blocks — don't assume a variable named for its typical role (e.g. "sand" = light cream) holds that role in *every* mode.
2. If a section is deliberately single-treatment (a fixed dark photo backdrop, for example) and must look the same regardless of site theme, its text/border colors need a **fixed, non-flipping variable** (like `--on-dark`), not a theme-reactive one repurposed for a different job.
3. Actually render the change in dark mode before calling it done — toggle the OS/browser dark mode (or Chrome DevTools' Rendering tab → "Emulate CSS media feature prefers-color-scheme") and eyeball contrast on every section touched. A `tsc`/lint pass or "it renders" is not the same as "it's legible in both themes."
4. This applies to `index.html` and `admin.html` both — `admin.html` uses the same variable system.
