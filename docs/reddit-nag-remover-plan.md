# Reddit Nag Remover — Safari iOS Web Extension (build handoff)

## Goal
A Safari **Web Extension** (not a content blocker) that removes Reddit's
"Get the app to keep using Reddit" blocking sheet and restores scrolling on
mobile Safari. Succeeds where Sink It fails because it runs real JS, releases
the body scroll-lock, and self-heals against rotating IDs / re-injection.

## Repo layout
```
extension/
  manifest.json
  block-reddit-nag.js      <- the content script (the whole product)
  icon-48.png / icon-96.png / icon-128.png
.github/workflows/
  ci.yml                   <- lint + invariant tests + auto-release on green main
  release.yml              <- build unsigned IPA + update AltStore source + GitHub Release
  bump-version.yml         <- manual workflow_dispatch version bump
  security.yml             <- CodeQL on the content script
docs/reddit-nag-remover-plan.md
altstore-source.json       <- SideStore/AltStore source manifest (updated by CI)
icon.png                   <- 512px icon for the AltStore source
test/extension.test.js     <- node:test invariant guards (no deps)
```

## manifest.json (MV3)
A content script scoped by `matches` is all that is needed — no `permissions`
/ `host_permissions` block. The user just toggles the extension on for
reddit.com in Safari settings.

## Build & distribution (CI → SideStore)
This repo does **not** commit an Xcode project. The release workflow generates
one at build time on a macOS runner with Apple's converter, then builds an
**unsigned (ad-hoc) IPA** that SideStore/AltStore re-signs with your Apple ID:

1. `xcrun safari-web-extension-converter extension --ios-only ...` generates the
   container App + Web Extension appex targets.
2. `xcodebuild archive` with `CODE_SIGN_IDENTITY="-" AD_HOC_CODE_SIGNING_ALLOWED=YES`
   builds a fake-signed archive (no Apple Developer team required).
3. The `.app` is wrapped into `Payload/…app` and zipped to `Shreddit.ipa`.
4. `altstore-source.json` is updated with the new version + download URL and a
   GitHub Release is created with the IPA attached.

### Cutting a release
- **Automatic:** merge a Conventional-Commit `feat:` / `fix:` to `main`. `ci.yml`
  computes the next semver, tags it, and calls `release.yml`.
- **Manual:** run the **Bump version and make a release** workflow
  (`workflow_dispatch`) and pick patch/minor/major. Or push a tag: `git tag
  v1.0.0 && git push origin v1.0.0`.

### Installing on iPhone
Add this source URL in SideStore/AltStore:
```
https://raw.githubusercontent.com/ssalonen/shreddit/main/altstore-source.json
```
Install Shreddit, then on device: **Settings ▸ Apps ▸ Safari ▸ Extensions ▸
Shreddit** → enable, set reddit.com to **Allow** (Allow Always avoids the
per-visit prompt). The app re-signs every 7 days on a free Apple ID — SideStore
refreshes it.

### Local build (optional, needs a Mac with Xcode)
```
xcrun safari-web-extension-converter ./extension --ios-only
```
Open the generated `.xcodeproj`, set your signing team, build to an iPhone.

## How it works (and why it survives rotation)
- Matches the nag by **stable structural signatures** — `rpl-bottom-sheet[blocking]`,
  `[id*="app-upsell-blocking"]`, and the `faceplate-loader[name^="AppUpsellBlocking"]`
  loader prefix — not the exact IDs (`...-seo` / `...-direct`), which Reddit rotates.
- **Removes the host element**, which pulls any top-layer `<dialog>` (shown via
  `showModal()`) out cleanly, regardless of open/closed shadow root.
- **Releases the scroll lock**: strips `rpl-scroll-lock` / `scroll-is-blocked`
  from `<html>`/`<body>` (their CSS is `overflow:hidden !important`) and re-asserts
  it via an injected stylesheet in case Reddit re-locks without re-rendering a sheet.
- **Self-heals**: a `MutationObserver` (childList+subtree, plus class/open/style
  attribute changes) re-runs on every relevant change, so the lazy 30-second
  injection and any later re-injection get cleaned up automatically.
- Static CSS handles the cosmetic, non-blocking nags (top-nav "Get app" button,
  small header, app-selector) cheaply.
- **The "fog gradient"** is the bottom sheet's `::part(overlay)`
  (`linear-gradient(... var(--color-ui-modalbackground))`), with the card on
  `::part(panel)` and `z-index:999` on `::part(base)`. Removing the host takes
  the whole shadow tree (fog included); the injected stylesheet also neutralizes
  these parts directly as a race-free backstop. This is exactly what Sink It
  misses: it hides only the `-direct` host, the live sheet is `-seo`, so the
  `-seo` overlay (fog) and its scroll lock both pass straight through.

## Caveats / things to verify on-device
- **Cookie/visit gating.** The blocking sheet is partly cookie- and visit-count
  gated (`blocking_seo_lo_30s` = SEO, logged-out, 30s). Logged-in sessions may not
  see it at all. The removal logic is agnostic to the trigger, so it works either
  way — but if you ever want to confirm a regression, clearing reddit.com cookies
  re-arms the trigger for testing.
- **If a future variant slips through**, capture a fresh HTML snapshot and check
  whether the new host still contains `rpl-bottom-sheet[blocking]` or an
  `app-upsell` substring. If Reddit renames both, add the new signature to
  `NAG_SELECTORS`. This is the one line you'd ever need to touch.
- **Don't over-block the loader.** We remove only `faceplate-loader` whose name
  starts with `AppUpsellBlocking`; other faceplate-loaders drive legitimate content.

## References (current as of mid-2026; expect drift)
- piunikaweb.com/2026/05/04/reddit-blocking-mobile-browser-access/ — running log of
  community uBlock filters incl. the `###app-upsell-blocking-bottom-sheet-{seo,direct}`
  hides and the `body,html:style(overflow:auto !important)` scroll fix; also notes
  the cookie-clear reset and that filters "stop working after a few hours."
- Ars Technica (5 May 2026) / MacRumors (11 May 2026) — context on the rollout as a
  logged-out mobile-web A/B block with no native dismiss.
- Snapshot-confirmed names: `app-upsell-blocking-bottom-sheet-seo`, `...-direct`,
  `rpl-bottom-sheet[blocking][open]`, body `rpl-scroll-lock scroll-is-blocked`
  with CSS `.rpl-scroll-lock{overflow:hidden!important}`, and the fog at
  `#app-upsell-blocking-bottom-sheet-seo::part(overlay){background:linear-gradient(...)}`.
