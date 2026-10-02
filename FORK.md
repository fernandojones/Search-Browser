# Search-Browser — what this fork adds

This is a fork of [Search](https://github.com/driceroland/Search) by Office Commun (MIT).
Everything here is upstream's work except the changes below, which are all in
**Settings › Passwords › Bring things over**, for Chromium browsers (Chrome,
Dia, Arc, Brave, Edge, Vivaldi, …).

"Search" and its icon belong to Office Commun. This fork publishes source only;
if you ship builds of it, rename the app first, as upstream's README asks.

## 1. Cookies and sign-ins import

Bring a Chromium profile's cookies into Search so you stay signed in to your
sites, alongside the passwords, bookmarks and history Search already imports.

- Reads the profile's `Cookies` database (`Network/Cookies` on newer
  Chromium), from a copy, never the live file.
- Decrypts `v10` values with the browser's own key from the macOS keychain. The
  key is the one passwords already use, so macOS asks only once.
- Handles cookie schema version 24 and later, where every value starts with
  the SHA-256 of its domain. That hash is checked and then removed, so a value
  that fails it is skipped instead of being imported corrupted.
- Keeps the Secure, HttpOnly, SameSite and expiry attributes. Session cookies
  stay session cookies.
- Skips partitioned cookies (CHIPS, which have a non-empty `top_frame_site_key`)
  because `HTTPCookie` cannot represent them, and skips expired cookies.
- Never overwrites a cookie Search already has. If a site's sign-in exists in
  both browsers, Search's own wins. The result line reports cookies imported,
  cookies already here, cookies skipped and any WebKit refused.
- Goes into the Space on screen. It needs one profile chosen rather than
  "All profiles", so accounts from different profiles don't mix.

**Chromium detail worth knowing:** Chromium writes `has_cross_site_ancestor = 1`
on *every unpartitioned* cookie as a placeholder. Only `top_frame_site_key`
says whether a cookie is partitioned. Filtering on `has_cross_site_ancestor`
drops almost every real cookie, which is the "0 cookies imported, all expired
or unsupported" symptom. The tests use rows shaped the way Chromium really
writes them, so they catch this.

Code: `Sources/Search/ImportCookies.swift`, with small hooks in
`Import.swift` (cookie counts in the preview, a shared keychain key) and
`ImportPanel.swift`.

## 2. Each profile as a Space

One switch turns every profile of a multi-profile browser into its own
**Space**, named after the profile, with its own cookies and sign-ins:

- Each profile gets a Space with the name the browser gives it in
  `Local State` (for example "Work" or "Personal"), with its own WebKit
  website-data store and its own icon.
- The profile used most recently goes into the first Space, where its
  sign-ins already are, so it isn't duplicated.
- A Space that already has the profile's name (in any letter case) is reused,
  so running the import again adds to the existing Spaces instead of creating
  copies.
- Spaces are turned on if they were off. The keychain is asked once for every
  profile together.
- Passwords, bookmarks, history and extensions stay shared across Spaces, as
  Search always shares them. They still follow the Profile picker.

Code: `Browser.spaces(forProfiles:usual:)` in `Sources/Search/Spaces.swift`,
and the "Each profile as a Space" option in `ImportPanel.swift`.

## Tests

- `Tests/SearchTests/ImportCookiesTests.swift` covers the domain-hash check and
  its removal (including empty values), expired and partitioned rows with
  realistic `has_cross_site_ancestor` values, cookie attributes, how a cookie
  is identified as "the same" one, and installing into WebKit without
  replacing an existing sign-in.
- `testChromiumProfilesBecomeNamedSpacesWithSeparateSignIns` in
  `ImportFileTests.swift` covers profile names becoming Spaces, the most
  recent profile mapping to the first Space, reuse on a second run, and a
  cookie set in one profile's Space not being visible in another Space or in
  the first one.

```sh
swift test --filter "ImportCookiesTests|ImportFileTests"
```

## Other

- `build.sh`: `SEARCH_BUILD_DISABLE_SANDBOX=1` passes `--disable-sandbox` to
  SwiftPM, for building inside an environment that is already sandboxed.
