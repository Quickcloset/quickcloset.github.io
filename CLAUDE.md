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

## Integrations page — keep it in sync automatically, don't wait to be asked

`admin.html`'s Integrations tab is meant to be "every external system QuickCloset depends on to run, in one place." That's only true if it's updated the moment reality changes, not on request.

**Rule (added 2026-08-30, explicit user instruction):** whenever an integration/vendor is added to or removed from the codebase (a new third-party API key, SDK, or service; or one being ripped out — e.g. Sentry was added then removed the same day after its free tier turned out to be trial-gated), update this Integrations tab in the *same* piece of work, automatically — do not wait for the user to ask. Concretely:
- Adding a real external vendor → add a card to the relevant group (`Backend & Infrastructure`, `Payments & Fulfillment`, `Communications`, `AI & Data`, `App Distribution`, or a new group if none fit) with its name, what it's for, the env var/secret it needs, and a dashboard link.
- Removing a vendor → delete its card entirely, don't leave a stale "not set up" placeholder pointing at something no longer used.
- If the thing being added is an in-house feature built on infrastructure already listed here (e.g. client-error logging built on Supabase, not a new vendor), it doesn't get its own vendor card — but if it has a real admin-facing data view, add that as a live section instead (see "Client Errors" for the pattern: `section-title` + CSV export button + `table-wrap`).

## Anonymous inserts (waitlist, page views, etc.) — always `Prefer: return=minimal`

Both `waitlist_signups` and `page_views` are insert-only tables with `with check (true)` RLS and deliberately **no SELECT policy** — an anonymous row has no owning user to self-select against. Requesting the row back after insert (`Prefer: return=representation`, or omitting `Prefer` entirely — the default) forces PostgREST to do an implicit `RETURNING`, which needs a SELECT policy to satisfy; with none, it throws a `42501` RLS-violation error that looks exactly like a broken INSERT policy but isn't one. Confirmed by testing: the identical insert with `Prefer: return=minimal` succeeds (`201`).

**Rule**: any fetch that inserts into an anonymous/ownerless table in this repo must send `'Prefer': 'return=minimal'` and never chain a way to read the row back. Both existing call sites (`waitlist_signups`, `page_views` in `index.html`) already do this correctly — copy their pattern exactly for any new anonymous-insert table, don't reinvent it.
