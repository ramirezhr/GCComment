# Changelog

All notable changes to GCComment are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [v104] 2026-09-19

Revival work after roughly eight years without maintenance. The script header
still declares `@version 103`; bump it together with `src/version.json` before
publishing, otherwise the update check will not offer these changes.

### Removed

- **Legacy profile page handler.** `gccommentOnProfilePage`, the handler for the
  old `/my/default.aspx` profile, has been deleted (768 lines). It dated from the
  transition period when geocaching.com served both the old profile and the new
  dashboard; only the dashboard remains.
- **Dispatch branch for `/my/`** in `main()`. The chain now goes straight from
  `cdpf.aspx` to `/account/dashboard`.
- **Old/new branch in `DropboxShowAuthLink`**, which built OAuth redirect targets
  pointing at `https://www.geocaching.com/my/default.aspx`.
- **IDResolver, entirely.** The service at `idresolver.azurewebsites.net` mapped a
  permanent GUID to the ID of the most recent Gist export, so that a printed link
  or QR code kept resolving to the current data. Six sites were involved: the
  settings block on the profile page, the QR codes for the permanent link, the
  click handlers for Remove / Login (`/check`) / Create (`/register`), the
  permanent link as the default value of the import field, the resolution of
  GUID-shaped IDs on import, and the `PUT` that pushed the new target ID after
  each Gist export.
- **Gist export and comment sharing.** `gist.uploadNewGist` posted to
  `api.github.com/gists` without an `Authorization` header, i.e. it created
  anonymous gists — something GitHub disabled in 2018, so this path could not
  work regardless. Gone with it: `performFilteredGistExport` and its button,
  `shareComment` and the share icon on the cache detail page (including the nine
  `style.display` lines that toggled it), `gistShare.shareComment`, the
  `gistNotice` banner, and the language keys `gistNotice*`,
  `export_toGistPerformFilteredExport` and `detail_share`.
- **`jquery.qrcode` and `nyroModal`.** Once link creation was gone these had no
  callers left — the QR codes only ever rendered links the script had just
  produced. Two `@require` lines, the `@resource nyroModalCss` plus its
  `appendCSS` call, and the orphaned `linkIcon` constant were removed with them.
  The header is down from five external dependencies to three: the Dropbox SDK,
  jQuery and DataTables.

Gist **import** is deliberately untouched: `gistShare.getComment` reads via an
unauthenticated `GET api.github.com/gists/{id}`, which still works for secret
gists when the ID is known. Links people saved or printed years ago can still be
loaded. `loadFromGist`, the input field, the "Load from link" button and
`gccommentOnSharingPage` all remain.

The shift-click on the settings button that dumps `GistIdLog` was also kept. The
log no longer grows, but for many users it is the only record of the links they
created previously — and those can still be imported.

### Changed

- **`gccommentOnNewProfilePage` renamed to `gccommentOnProfilePage`.** With the
  legacy handler gone, "New" no longer distinguished anything.
- **Update check moved to the dashboard.** `updateCheck()` was gated on
  `/my/default.aspx` and friends, so it silently stopped running once that page
  disappeared — no error, just no update notifications. The gate now matches
  `/account/dashboard`.
- **`waitForElement` moved to top-level scope** (just before `mainCode`). It had
  been declared inside `mainCode` and was therefore invisible to `updateCheck`,
  which is a separate top-level function. The existing call in
  `gccommentOnDetailpage` is unaffected.
- **`updateAvailable` is now `async` and waits for `#gccRoot`.** The element is
  created by `mainCode()`, on the dashboard asynchronously inside a
  `MutationObserver`, while `updateCheck()` runs first — the previous
  `insertBefore` could therefore run against `null`. The notice is now assembled
  in full and inserted once at the end, after `await waitForElement("#gccRoot",
  15000)`. On timeout it logs instead of throwing. DOM order is unchanged.
- **`waitForElement` observes `document.body || document.documentElement`**, so it
  survives being called before the body exists.

### Dropbox

Authentication and the import/export round trip have been tested end to end and
work.

- **SDK updated from 2.5.13 to 10.46.0**, loaded from jsDelivr
  (`dropbox@10.46.0/dist/Dropbox-sdk.min.js`) instead of cdnjs. Three breaking
  changes were adopted:
  - The UMD bundle exports a namespace object, so `new Dropbox({...})` became
    `new Dropbox.Dropbox({...})` / `new Dropbox.DropboxAuth({...})`.
  - Responses are wrapped in `DropboxResponse`: `response.entries` and
    `response.fileBlob` became `response.result.entries` and
    `response.result.fileBlob`.
  - `getAuthenticationUrl` returns a Promise, because the PKCE challenge is
    hashed asynchronously via `crypto.subtle`.
- **Authentication switched from the implicit flow to Authorization Code with
  PKCE.** Dropbox moved to short-lived access tokens in 2021, so reading
  `access_token` out of the URL hash no longer yields anything durable. The
  authorization code now arrives as `?code=` and is exchanged for an access token
  plus a refresh token. The exchange runs through the SDK's
  `getAccessTokenFromCode`; Dropbox's token endpoint returns usable CORS headers
  for this, so no `GM_xmlhttpRequest` workaround is needed.
- **Automatic token refresh.** `doDropboxAction()` calls
  `checkAndRefreshAccessToken()` before every operation; the SDK renews an expired
  access token from the refresh token on its own.
- **New storage keys:** `Db_Refresh_Token`, `Db_Access_Token_Expires` and the
  transient `Db_Code_Verifier`, alongside the existing `Db_Access_Token`.
- **Fixed `DropboxShowAuthLink`.** It assigned the auth links through implicit
  globals (`Db_AuthLinkImport`, `Db_AuthLinkExport`) that were only set inside a
  URL check, so `$(Db_AuthLinkImport).show()` ran against `undefined` elsewhere.
  It now resolves the elements via `getElementById` with null checks, and the URL
  check is gone because the redirect URI is fixed.

The call sites are untouched: `doDropboxAction()` still returns a jQuery Deferred
with `.done()` / `.fail()`.

#### Dropbox App Console prerequisites

The new flow depends on the app being configured accordingly:

1. The redirect URI `https://www.geocaching.com/account/dashboard?AppId=GCComment`
   must be registered verbatim — Dropbox compares it including the query string.
2. The app needs the scopes `account_info.read`, `files.metadata.read`,
   `files.content.read` and `files.content.write`. No `scope` argument is passed,
   so whatever the console grants is what gets requested.
3. The app key `w23bgpsespnddow` dates back to lukeIam. If nobody can still reach
   that console, a new key is needed; the constant is `DROPBOX_APP_KEY`.

An access token stored during the implicit-flow era keeps working until Dropbox
revokes it: with no refresh token and no expiry recorded,
`checkAndRefreshAccessToken()` does nothing and the old token is reused.

### Deferred

- **Google Drive as a second backend.** Feasible, but it carries two obstacles
  Dropbox did not. Google only permits redirect URIs and JavaScript origins on
  domains you own and have verified, so `geocaching.com` cannot be registered — a
  small static auth page on a domain of our own is required, with the script
  matching it via `@include` and lifting the tokens into GM storage. And Google's
  "Web application" client type has no public-client mode: the token exchange
  demands a `client_secret` even with PKCE, which would ship in plain sight inside
  the userscript. The scope to use would be `drive.file`, classified as
  non-sensitive and needing only basic app verification rather than a CASA
  security assessment. Postponed until the cleanup is finished; a suitable domain
  is available.
- If Drive does get built, the four Dropbox functions should not simply be copied.
  A small backend interface with `list()`, `upload()` and `download()`, plus a
  selector in the settings, would keep both providers on one code path.

### Known issues

- **`sendToGPS` is called but never defined.** `main()` dispatches to it for
  `/sendtogps.aspx`, producing a `ReferenceError`. Pre-existing, unrelated to this
  work.
- **`idResolverId` and `idResolverSecret` linger** in users' Tampermonkey storage
  with nothing reading them any more. `doMaintenance()` would be the place for a
  one-off `GM_deleteValue` if that should be tidied up.
- **`checkforupdates` comment is wrong.** The interval is commented as "equals 1
  day" but `14400000` is four hours. `86400000` would be a day.
- **Update URLs point at `Birnbaum2001/GCComment`** while the icon comes from
  `ramirezhr/GCComment`. If version stands are maintained elsewhere now,
  `version.json` has to live wherever `updatechangesurl` points, or the repaired
  update check will come up empty.

### Testing notes

- To trigger the update check, clear `updateDate` in Tampermonkey storage or set
  it to an old value — otherwise the four-hour throttle in `checkforupdates`
  prevents the request from going out at all.
- To re-test Dropbox authentication, clear `Db_Access_Token` and
  `Db_Refresh_Token`, then use the auth link in the import or export section.

## [103] and earlier

See the project history on GitHub.
