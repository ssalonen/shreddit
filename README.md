# Shreddit — Reddit Nag Remover

A Safari **Web Extension** for iOS/macOS that removes Reddit's *"Get the app to
keep using Reddit"* blocking sheet on mobile Safari and restores scrolling.

Unlike content-blocker rules (e.g. Sink It), it runs real JavaScript at
`document_start`, **releases the body scroll-lock**, neutralizes the blocking
bottom-sheet *and* its fog overlay, and **self-heals** against re-injection and
Reddit's rotating element IDs.

## Install on iPhone (SideStore / AltStore)

Add this source in SideStore/AltStore:

```
https://raw.githubusercontent.com/ssalonen/shreddit/main/altstore-source.json
```

Install **Shreddit** (SideStore signs the unsigned IPA with your Apple ID), then:

> **Settings ▸ Apps ▸ Safari ▸ Extensions ▸ Shreddit** → enable, and set
> reddit.com to **Allow** (Allow Always avoids the per-visit prompt).

You can also grab the IPA from the [Releases](../../releases) page directly.

## How it works

The nag is `<rpl-bottom-sheet blocking open>` inside
`app-upsell-blocking-bottom-sheet-{seo,direct}`, lazy-loaded via a
`faceplate-loader[name^="AppUpsellBlocking"]`. The thing that actually freezes
the page is the `.rpl-scroll-lock` / `.scroll-is-blocked` class on `<body>`.
The single content script `extension/block-reddit-nag.js`:

- matches the nag by **stable structural signatures**, not the rotating
  `-seo` / `-direct` IDs;
- **removes the host** (pulling any top-layer `<dialog>` out cleanly) and
  neutralizes its `::part(overlay)` fog;
- **releases the scroll-lock** classes and re-asserts `overflow:auto` via CSS;
- **self-heals** with a `MutationObserver` for the lazy ~30s injection and any
  later re-injection.

See [`docs/reddit-nag-remover-plan.md`](docs/reddit-nag-remover-plan.md) for the
full diagnosis and references.

## Development & releases

- `extension/` — the MV3 web extension (the whole product).
- `test/extension.test.js` — `node:test` invariant guards; run with `node --test`.
- No Xcode project is committed; CI generates one with
  `xcrun safari-web-extension-converter` and builds an unsigned IPA.

Releases are automated:

- merge a Conventional-Commit `feat:`/`fix:` to `main` → CI tags + builds, **or**
- run the **Bump version and make a release** workflow and pick patch/minor/major, **or**
- push a tag: `git tag v1.0.0 && git push origin v1.0.0`.

*Unsigned builds — SideStore/AltStore signs them with your Apple ID on install.*
