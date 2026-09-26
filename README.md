# ibrahimbnc.github.io

GitHub Pages **user site** for the `IBRAHIMBNC` account, served at the repo
root: `https://ibrahimbnc.github.io/`. Its only job today is hosting the
Mathless app's invite links and the Android App Links verification file. See
`splitly/docs/invite-links-audit/PLAN.md` (Phase 4) in the app repo for the
full context this was built from.

## What's here

- `index.html` — a one-line Mathless placeholder for the bare root.
- `invite/index.html` — served directly for the bare `/invite` or `/invite/`
  path (no token). Always renders the "broken" message, since there's no
  token to parse there.
- `404.html` — the actual invite page. See "The 404 trick" below.
- `.well-known/assetlinks.json` — Android Digital Asset Links statement for
  `com.mathless.app`, so Android's App Links verifier trusts this site to
  open in the app instead of a browser.
- `.nojekyll` — GitHub Pages builds with Jekyll by default, which ignores
  dot-folders. Without this file, `.well-known/` is never served.

## The 404 trick

The app shares links shaped `https://ibrahimbnc.github.io/invite/<token>`
(`AppLinks.inviteUrl` in `lib/core/constants/app_links.dart`). GitHub Pages
is static hosting with no server-side rewrites or routing: there is no real
file at `/invite/<token>` for any given token, so every one of those URLs
404s.

What makes it work anyway: GitHub Pages serves this repo's `404.html` in
place of any path that doesn't resolve to a real file, and the browser
still renders whatever that file contains — the address bar keeps showing
the original `/invite/<token>` URL. `404.html`'s inline script reads
`location.pathname`, pulls out the token, and renders the actual invite UI
(or a "this invite link looks broken" message if the path or token doesn't
look right). This is the standard way to fake client-side routing on GitHub
Pages when the alternative (a real static file per token) isn't possible.

Two things worth being explicit about, since the page still returns HTTP
404:

- **Android App Links verification is unaffected.** Verification only ever
  fetches `/.well-known/assetlinks.json` directly, which is a normal file
  served with a normal 200 — it never touches `/invite/<token>` or
  `404.html` at all.
- **Link previews can suffer.** Some chat apps (WhatsApp, Instagram) look at
  the HTTP status and/or fetch differently for link-preview generation, and
  a 404 status can make them show a generic preview or none at all instead
  of pulling `<title>`/OG tags from the page. The link still opens and
  renders correctly when tapped; only the preview card is affected. This is
  an accepted limitation of the GitHub Pages 404 trick, not a bug to chase
  — a real preview would need actual server-side routing (a proper 200 per
  token), which is out of scope for a static Pages site.

The token itself is validated against the app's own format (a base64url
string — see `DeepLinkService.extractInviteToken` in the app repo) before
it is ever used: it's rejected outright if it doesn't match, and even when
valid it's only ever inserted into the DOM via `textContent`/`setAttribute`,
never through `innerHTML` or string-built HTML, so a malicious path segment
can't turn into a script.

## Android App Links

`.well-known/assetlinks.json` currently lists one SHA-256 fingerprint:

```
38:08:1E:FB:1E:98:7D:85:59:E5:DB:89:3A:C2:A9:1C:3D:9A:62:04:0B:80:2B:FA:BB:53:E4:96:E9:D6:88:5F
```

That's the debug keystore's fingerprint (`~/.android/debug.keystore` on the
dev machine); release builds sign with the same key today, which is why one
entry currently covers both. **Once the app ships on Google Play, add Play
App Signing's own SHA-256 fingerprint to the `sha256_cert_fingerprints`
array** (Play re-signs the APK it distributes with its own key, which is
different from whatever key was used to upload it) — don't replace the
existing entry, just add to the array, since builds signed locally still
need to verify too.

This file is what backs the `<intent-filter android:autoVerify="true">`
block in `android/app/src/main/AndroidManifest.xml` for
`host="ibrahimbnc.github.io"` `pathPrefix="/invite"`. A build whose signing
key isn't listed here simply doesn't verify as the default handler for
these links — the https page still works as a fallback (that's the point of
`404.html`), it just opens in a browser first instead of the app.

## iOS

iOS Universal Links (`.well-known/apple-app-site-association` in this same
repo) are deferred until Apple Developer Program enrollment — see
`docs/apple-native-signin-plan.md` in the app repo. Until then, iOS goes
through this same https page and then the custom scheme
(`com.mathless.app://invite/<token>`), same as Android when it doesn't
verify.
