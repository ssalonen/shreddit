# Shreddit — Reddit Nag Remover (Safari iOS Web Extension)

Removes Reddit's "Get the app to keep using Reddit" blocking sheet on mobile
Safari and restores scrolling. The single content script
`extension/block-reddit-nag.js` does all the work. Full background and
references: @docs/reddit-nag-remover-plan.md

## Layout
- `extension/block-reddit-nag.js` — the content script (the whole product). Runs
  at `document_start` on `*://*.reddit.com/*`.
- `extension/manifest.json` — MV3 manifest declaring the content script.
- `extension/icon-{48,96,128}.png` — extension icons.
- `.github/workflows/` — CI/CD (see below).
- `altstore-source.json` — SideStore/AltStore source manifest, updated by CI.
- `test/extension.test.js` — `node:test` invariant guards (no dependencies).
- `docs/reddit-nag-remover-plan.md` — diagnosis, build steps, references.

## Build / release / test
- **CI** (`ci.yml`): on every push/PR, syntax-checks the script and runs
  `node --test`. On a green push to `main`, computes the next semver from
  Conventional Commits and triggers a release.
- **Release** (`release.yml`): on a `v*` tag (or `workflow_call`), a macOS runner
  runs `xcrun safari-web-extension-converter ./extension --ios-only` to generate
  the Xcode project, builds an **unsigned ad-hoc IPA** (no Apple Developer team),
  updates `altstore-source.json`, and publishes a GitHub Release with the IPA.
- **Bump** (`bump-version.yml`): manual `workflow_dispatch` → pick patch/minor/major.
- **Install**: add the source URL
  `https://raw.githubusercontent.com/ssalonen/shreddit/main/altstore-source.json`
  in SideStore/AltStore; it re-signs the IPA with your Apple ID. Then on device:
  Settings ▸ Apps ▸ Safari ▸ Extensions ▸ enable, set reddit.com to Allow Always.
- **No Xcode project is committed** — it is generated at build time. Locally on a
  Mac you can run the converter yourself to debug (see the plan doc).
- **Manual on-device test**: open a subreddit post logged-out in Safari, wait past
  the ~30s timer. PASS = no fog overlay, page scrolls, no "Get the app" sheet. To
  re-arm the trigger between tests, clear reddit.com cookies (it is visit/cookie-gated).
- iOS has no DOM inspector locally; use Safari Web Inspector from a Mac for
  diagnosis. Disable Safari's "Hide distracting items" while testing so you are
  measuring the extension, not the manual rule.

## How the nag works (1-paragraph model)
The nag is `<rpl-bottom-sheet blocking open>` inside
`app-upsell-blocking-bottom-sheet-{seo,direct}`, lazy-loaded via a
`faceplate-loader[name^="AppUpsellBlocking"]`. The "fog gradient" is the sheet's
`::part(overlay)`; the card is `::part(panel)`. The thing that actually freezes
the page is the `.rpl-scroll-lock` / `.scroll-is-blocked` class on `<body>`
(`overflow:hidden !important`). Sink It fails because it hides only the `-direct`
host; the live variant is `-seo`, so its fog and scroll-lock pass through.

## Invariants — do not regress
- IMPORTANT: any fix MUST do BOTH of these, never just one: (1) remove/neutralize
  the blocking sheet + its `::part(overlay)` fog, and (2) release the body scroll
  lock. Hiding the sheet without releasing scroll leaves the page frozen.
- Match by STABLE signatures (`rpl-bottom-sheet[blocking]`,
  `[id*="app-upsell-blocking"]`, loader name prefix `AppUpsellBlocking`), NOT the
  rotating `-seo` / `-direct` suffixes. Adding a new signature to `NAG_SELECTORS`
  is the only line that should ever need maintenance.
- Do NOT broaden the `faceplate-loader` removal beyond the `AppUpsellBlocking`
  name prefix — other faceplate-loaders drive legitimate content.
- Keep the `MutationObserver` reactive design: the sheet is injected late and can
  re-inject; one-shot removal is not enough.
- `test/extension.test.js` enforces these invariants in CI — keep it green.

## Conventions
- Vanilla JS only, no dependencies, no build step for the script itself.
- Make minimal changes; do not refactor the script's structure without reason.
- Use Conventional Commits (`feat:`, `fix:`, `chore:`…) — CI derives the release
  version from them.
- If a new variant slips through, capture a fresh HTML snapshot first, confirm the
  new host's signature, then extend `NAG_SELECTORS` / the injected `::part` rules.
