# RFC 0000: YSM stack (Yesod + Servant + Miso)

- **Status:** Draft, for discussion
- **Created:** 2026-10-26
- **Scope:** A web application with server-rendered pages, Miso front ends for web
  and Android, and access gated behind OAuth-provided users.

> This RFC records one candidate architecture and the reasoning behind it.
> It is not a decision. It is meant to be challenged, prototyped against, and
> amended or rejected. Claims are tagged:
> **[verified]** = checked in package source or metadata (see Appendix A),
> **[reported]** = from third-party documentation, not re-checked in code,
> **[unverified]** = an assumption that needs a spike before relying on it.
> Nothing in this RFC has been compiled yet.

---

## 1. Summary

Split the system into three roles:

| Role                       | Candidate                                    | Responsibility                                                                                        |
|----------------------------|----------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Auth core + HTML site      | **Yesod** + Hamlet + `yesod-auth-oauth2`     | OAuth login with upstream providers, users table, sessions, server-rendered pages, issuing API tokens |
| Data API (resource server) | **Servant** (`AuthProtect` + `jose`)         | Typed JSON API, stateless JWT verification                                                            |
| Interactive clients        | **Miso**: web islands + Android via LynxJS   | Rich UI where needed; separate views for desktop and mobile on a shared core                          |

Persistence uses **Beam** in all server components. i18n is out of scope.

## 2. Context and requirements

1. **R1** Most of the site is server-rendered HTML. It should be maintainable and
   checked at compile time as far as possible.
2. **R2** Almost everything is behind login. Identities come from external
   OAuth/OIDC providers (e.g. Google, GitHub).
3. **R3** Miso adds interactivity to parts of the web site.
4. **R4** A Miso Android app talks to the same backend, with a UI tuned
   for tasks that happen more often on phones.
5. **R5** Beam is the database layer.
6. **R6** Avoid lock-in to a single toolchain (e.g. requiring Nix) where practical.

## 3. Why these frameworks

### 3.1 Hamlet over Ede

An earlier variant used Ede templates. Ede takes an untyped JSON-like context,
so with Ede:

- `@{Route}` interpolation is unavailable, and URLs must be rendered to `Text`
  in Haskell and passed in.
- Template variable typos become runtime errors or empty output.
- Yesod widgets and widget-based `yesod-form` rendering lose most of their value.

`yesod-core` depends on `shakespeare` and `blaze` anyway **[verified]**, so Ede
adds a dependency without removing one, and narrows Yesod's advantage over a
minimal framework to routing, sessions and auth.

### 3.2 What Yesod provides that scotty, twain or raw WAI don't

- Typed routes from `parseRoutes`, with compile-time overlap checks **[verified]**.
- A central `isAuthorized :: Route site -> Bool -> HandlerFor site AuthResult`
  **[verified]**. Writing it without a wildcard case gives a non-exhaustive-match
  warning for every new route, which suits a site locked down by default (R2).
- Encrypted client-side sessions, flash messages, body size limits, logging and
  `yesodMiddleware` hooks **[verified]**.
- CSRF protection via `defaultCsrfMiddleware` and header-based checks
  (`checkCsrfHeaderNamed`, `setCsrfCookie`) **[verified]**. It is **opt-in**:
  it must be added to `yesodMiddleware`.
- `yesod-auth`: redirect-to-login (`authRoute`), return to the original page after
  login, `onLogin`/`onLogout` hooks **[verified]**.
- Markup-free applicative input validation (`runInputPost`) **[verified]**.
- `yesod-static`, `yesod-test`, subsites.

Costs: Template Haskell, long compile times, a heavier dependency tree, and a fixed
`HandlerFor site` monad. yesod-core 1.7's split route compilation
(`setFocusOnNestedRoute`) can move nested route blocks into their own modules,
which may reduce rebuild times **[verified: ChangeLog]**.

### 3.3 Comparison with Servant-only and IHP

| Concern                             | Yesod + Hamlet                         | Servant only                                         | IHP                                          |
|-------------------------------------|----------------------------------------|------------------------------------------------------|----------------------------------------------|
| Server-rendered site (R1)           | Strong                                 | Weak: no layouts, flash messages or form re-display  | Strong (HSX, AutoRoute)                      |
| Browser OAuth + login redirect (R2) | Strong (`yesod-auth-oauth2`)           | DIY (`hoauth2`, sessions, 302 vs 401)                | Built in, short provider list **[reported]** |
| Sharing an API type with Miso (R3–4) | Weak: route types live in a TH-heavy server package | Strong: API type in a small shared package | Weak: Miso not a first-class path |
| Beam (R5)                           | Fine                                   | Fine                                                 | Conflicts: IHP's ORM is its core             |
| Toolchain (R6)                      | Cabal or Stack                         | Cabal or Stack                                       | Nix-centric **[reported]**                   |

The table points to a **hybrid**: Yesod for the website and auth, and a Servant
API type shared with the Miso clients.

## 4. Proposal

### 4.1 Overall shape

```
                ┌──────────── Yesod (auth core + HTML pages) ────────────┐
 OAuth provider ◄┤ yesod-auth-oauth2 → users table (Beam)                │
                │ issues: session cookie (pages) + API tokens            │
                └──────┬──────────────────────────────┬──────────────────┘
       JWT in HttpOnly │ cookie + XSRF header         │ JWT bearer + refresh token
                       ▼                              ▼
         Miso web islands (same origin)     Miso Android app (LynxJS)
                       │                              │
                       └──────────► Servant API ◄─────┘
                         checks signature only (public key), no DB lookup per request
```

### 4.2 Package layout

```
shared/     DTOs (aeson), Servant API type, AuthUser.
            Depends on servant (core) and miso built with `+aeson`.
            Must build for native GHC and the chosen JS/wasm backend.
app-core/   Miso model, actions, update; API client via servant-miso-client.
web/        Desktop-oriented Miso views; entry point `startApp`.
mobile/     Mobile-oriented Miso views; miso `+native`, entry point `Miso.Native.native`.
android/    Lynx host app (Kotlin): native modules for login and secure storage.
server/     Yesod (pages, auth, token issuance) + Servant server + Beam.
```

Constraints that shape this layout:

- **No `servant-auth` in `shared/`.** Even the "core" `servant-auth` package
  depends on `jose` and `lens` (and `jose` pulls in `crypton`) **[verified]**.
  These shouldn't end up in client builds. Protected endpoints use servant core's
  `AuthProtect` instead (§4.4.3).
- **One set of JSON classes.** By default miso uses its own `Miso.JSON` classes,
  and `servant-miso-client` decodes with them **[verified]**. Building miso with
  its `aeson` flag makes `Miso.JSON` aeson's classes **[verified]**, so DTOs derive
  aeson instances once for both server and clients. Whether aeson builds on the
  chosen JS/wasm backend is **[unverified]** (§6).
- **`servant-miso-client` is git-only** (not on Hackage). Pin a commit in
  `cabal.project`. It depends on `servant` core and is maintained alongside
  miso, which is good evidence that `servant` cross-compiles **[reported]**.
- Platform differences live in the views. Model, actions and API calls are shared.

### 4.3 Three route spaces

miso 1.14 does not depend on servant. `Miso.Router` is its own generic `Router`
class **[verified]**. That leaves three separate typed route definitions:

| Routes          | Defined in               | Typed links within | Used by                  |
|-----------------|--------------------------|--------------------|--------------------------|
| HTML pages      | Yesod `parseRoutes`      | Hamlet `@{…}`      | Browser                  |
| JSON API        | Servant type in `shared/` | servant-miso-client | Web and Android clients |
| Client screens  | Miso `Router`            | `toURI`/`route`    | Miso apps                |

Links *across* these boundaries are not type-checked. Keep each crossing point
small and explicit (e.g. a few page URLs that host islands, and the app's deep-link
paths).

### 4.4 Authentication and tokens

#### 4.4.1 Upstream login: `yesod-auth-oauth2`

- The in-tree `yesod-auth-oauth` is OAuth **1.0a** (`authenticate-oauth`)
  **[verified]**. Not a candidate.
- `yesod-auth-oauth2` 0.8.0.0 (2026-01-07) supports GHC 9.12 and hoauth2 2.15
  (latest 2.15.2) **[verified]**. Providers include Google, GitHub, GitLab,
  AzureAD v2, Auth0, Bitbucket, Slack and Salesforce, plus the generic
  `authOAuth2` for anything else **[verified]**.
- It imports only the public `Yesod.Core`/`Yesod.Auth` APIs and bounds
  `yesod-core >= 1.6.0`. yesod-core 1.7's breaking changes are in route TH codegen
  internals, and in-tree `yesod-auth` 1.6.12.1 already supports 1.7. So
  compatibility is expected but **not yet compiled**.
- The upstream flow is protected by a session-stored `state` value, not PKCE.
  That is appropriate for a confidential server-side client. The PKCE in
  §4.4.5 is between the Android app and our server, which is a separate concern.
- In `authenticate`, map the credentials to a Beam user row and return
  `Authenticated uid`. `sub` in issued tokens is this internal `UserId`, never
  the provider's id.

#### 4.4.2 Token policy

- Asymmetric signing (EdDSA or ES256). Yesod holds the private key, and the API
  holds only the public key.
- Minimal claims: `sub`, `aud`, `iss`, `exp`, optionally roles. Ownership checks
  are done against data in the API.
- Access tokens last 5–15 minutes. Refresh tokens (mobile only) are stored hashed
  in Beam, rotated on every use, and the whole family is revoked if an old one is
  reused. Logging out or banning a user deletes the rows.

```haskell
issueApiToken :: UserId -> Handler BL.ByteString
issueApiToken uid = do
  App{appJwtSettings} <- getYesod
  roles <- runBeam (lookupRoles uid)
  exp'  <- addUTCTime 600 <$> liftIO getCurrentTime
  either (const $ permissionDenied "token") pure
    =<< liftIO (makeJWT (AuthUser uid roles) appJwtSettings (Just exp'))
```

(`makeJWT` is from `servant-auth-server`. This is fine on the server; only the
shared and client packages must avoid `jose`.)

#### 4.4.3 Protecting the API

- `shared/` marks protected endpoints with `AuthProtect "jwt"` from
  `Servant.API.Experimental.Auth` (servant core) **[verified]**.
- **Server:** a servant-server `AuthHandler` reads the `Authorization: Bearer`
  header first, then falls back to the JWT cookie plus XSRF header, verifies
  the token, and yields `AuthUser`.
- **Clients:** `servant-miso-client` has no `AuthProtect` instance, but it exports
  `HasClient (..)` and `Request (..)` **[verified]**. A short orphan instance,
  modelled on its `Header` instance, adds the header:

```haskell
instance HasClient api => HasClient (AuthProtect tag :> api) where
  type ClientType (AuthProtect tag :> api) = (MisoString, MisoString) -> ClientType api
  toClientInternal Proxy req (name, value) =
    toClientInternal (Proxy @api) req { _headers = M.insert name value (_headers req) }
```

Web passes `("X-XSRF-TOKEN", xsrf)`; Android passes `("Authorization", "Bearer " <> jwt)`.

#### 4.4.4 Web client

- On login (`onLogin`), Yesod sets a short-lived JWT cookie: `HttpOnly`, `Secure`,
  `SameSite=Strict`, `Path=/api`. It also sets a readable XSRF cookie, which Miso
  echoes into a header.
- When the API returns 401 (token expired), Miso calls a Yesod endpoint
  (e.g. `/auth/api-cookie`). It checks the Yesod session and reissues the cookie.
  The Yesod session acts as the refresh credential, and the browser never sees a
  token.
- No tokens in `localStorage`. The API is served under `/api` on the same origin,
  so there is no CORS.

#### 4.4.5 Android client (RFC 8252 + PKCE)

1. A Kotlin module opens a Custom Tab at `/auth/mobile/start?code_challenge=…`.
2. Yesod stores the challenge and runs the normal upstream login (§4.4.1).
3. Yesod redirects to a verified Android App Link with a ~60 s one-time code.
   The host Activity receives the intent and passes the code to Miso.
4. The app POSTs code + `code_verifier` to `/auth/mobile/token` and receives an
   access JWT and a refresh token.
5. The refresh token is stored by a Kotlin module backed by Android Keystore.
6. API calls use `Authorization: Bearer …`. No XSRF check is needed on this path.

Embedded-WebView login pages are ruled out. Major providers block them.

Yesod thereby becomes a minimal first-party OAuth2 authorization server:
a few routes and two tables (pending PKCE codes, refresh tokens).

### 4.5 Web: islands, not hydration

- Server rendering of Miso views is possible: `Miso.Html.Render.toHtml` and
  `toHtmlWith` produce a lazy `ByteString` **[verified]**, which can be embedded
  in Hamlet with `preEscapedToMarkup`.
- But hydration (`miso`, `prerender`) forces the mount point to `<body>`
  **[verified]**. It cannot hydrate an island inside a Hamlet layout.
- So **islands are client-rendered**: Hamlet emits `<div id="…">`, and the island
  starts with `startApp` with `mountPoint = Just "<element-id>"` (resolved via
  `getElementById`) **[verified]**. Server markup can sit inside the div as a
  placeholder; it is replaced, not hydrated.
- Server rendering plus hydration remains an option for pages where Miso owns the
  whole `<body>` and Hamlet supplies only `<head>`.

### 4.6 Android: Miso on LynxJS

- miso supports iOS, Android and HarmonyOS through LynxJS. The former
  `miso-lynx` project is now part of miso, enabled by the `native` cabal flag
  (`Miso.Native.*`) **[verified]**. The entry point is `Miso.Native.native`,
  not `miso`/`startApp`.
- Platform features beyond rendering come from Lynx native modules. They are
  Kotlin/Java classes extending `LynxModule` with `@LynxMethod` methods
  **[reported: Lynx docs]**. They are called from Haskell with
  `Miso.Native.Module.callNativeModule`, on the background thread
  (miso ≥ 1.13) **[verified]**.
- This RFC needs two such modules, written in-house:
  - **login**: Custom Tab launch and App Link callback;
  - **secure storage**: Keystore-backed storage for the refresh token.

  Lynx's own storage example uses plain `SharedPreferences` **[reported]**, so
  it isn't suitable as-is.
- Lynx `fetch` is Web-compatible but has no CORS, redirect or keepalive handling,
  and no FormData/Blob **[reported: Lynx docs]**. That is consistent with
  bearer-only API calls and with doing login in the Custom Tab rather than via fetch.
- `Miso.Native` is marked `Stability: experimental` **[verified]**. This is the
  main maturity risk in the proposal.

### 4.7 Hosting the API

- **Same process:** mount the Servant app inside Yesod with
  `WaiSubsiteWithAuth`, which "applies parent site's middleware and
  isAuthorized" **[verified]**. JWT auth makes a session bridge unnecessary.
- **Separate deployment:** possible because verification needs only the public key.

### 4.8 API evolution

Android installs lag behind deployments. So:

- Version the API from the start (`"v1" :> …`), with additive-only changes within
  a version.
- Expose a minimum-supported-client endpoint so the app can ask users to update.
- Mobile-specific endpoints (coarser, offline-friendly payloads) are acceptable
  within the same API type.

## 5. Alternatives considered

| Alternative | Why not (for now) | What would change the call |
|-------------|-------------------|----------------------------|
| Yesod + Ede | Loses checked templates and widgets (§3.1) | Designers must edit templates without a Haskell toolchain |
| scotty / twain / raw WAI | Sessions, CSRF, auth flows, typed links all DIY | Site shrinks to an API plus a few pages |
| Servant only | Website and browser-auth concerns are DIY (§3.3) | Server-rendered part becomes negligible |
| IHP | Conflicts with Beam; Miso not first-class; Nix-centric | Beam dropped, or Miso limited to small islands |
| External IdP (Keycloak, Zitadel, Ory Hydra) | More moving parts for a first-party-only setup | Third-party clients, scopes or consent, SSO across products, compliance |
| Plain Yesod JSON handlers + `Miso.Fetch` (no Servant) | No typed API client for Miso | Few endpoints, or the shared package fails to cross-compile |
| `servant-auth`'s `Auth '[JWT, Cookie]` in the API type | Pulls `jose`/`lens` into client builds (§4.2) | `jose` shown to build cleanly for the client targets |
| Hybrid Android shell (WebView) instead of Lynx | Weaker native UX; cookie and WebView quirks | `Miso.Native` proves too immature |

## 6. Risks and open questions

1. **`Miso.Native` maturity:** experimental, and with a small user base. Its
   Kotlin bridge is our responsibility (§4.6).
2. **Client-target builds [unverified]:** that `servant`, aeson and miso `+aeson`
   build together for the chosen JS/wasm backend and for Lynx.
3. **Toolchain:** miso's reference build workflows lean on Nix **[reported]**,
   which is in tension with R6. Confirm a non-Nix path, or accept Nix for client
   builds only.
4. **Several islands per page [unverified]:** whether several top-level `startApp`
   roots can coexist, or whether one root should mount child components into
   several elements.
5. **Hand-rolled authorization server:** the PKCE and refresh-token code is
   security-sensitive. It needs tests and review, or should be replaced by an
   external IdP (§5).
6. **Compile times:** Yesod TH plus Servant type-level APIs may be slow even with
   split route compilation. Needs measurement.
7. **`yesod-auth-oauth2` on yesod-core 1.7:** expected to work (§4.4.1) but not
   yet compiled.

## 7. Suggested validation spikes

Each spike should be small enough to throw away.

1. **Mobile login:** Lynx host app with the two Kotlin modules, a miso `+native`
   build calling them, and PKCE against a stub Yesod server. *(Risks 1, 5.)*
2. **Shared package:** `shared/` with one `AuthProtect` endpoint and aeson DTOs,
   miso `+aeson`, consumed by the server, a web build and a `+native` build,
   including the orphan `HasClient` instance. *(Risks 2, 3.)*
3. **Locked-down Yesod:** `yesod-auth-oauth2` 0.8 on yesod-core 1.7 + Beam users
   table + exhaustive `isAuthorized` + `defaultCsrfMiddleware`, with a
   `yesod-test` suite. *(Risks 6, 7.)*
4. **Mounted API:** Servant app under `WaiSubsiteWithAuth` with an `AuthHandler`
   accepting bearer or cookie + XSRF. *(§4.4.3, §4.7.)*
5. **Island page:** Hamlet layout with two islands mounted by element id.
   *(Risk 4.)*

## 8. What would make us reject this RFC

- Spike 1 shows `Miso.Native` is too immature for a production Android app, or the
  Kotlin bridge is impractical. Then fall back to a WebView shell, or to a
  different mobile stack against the same API.
- Spike 2 shows the shared API type can't build for the client targets. Then
  drop the Servant layer for plain Yesod JSON handlers and `Miso.Fetch`.
- Requirements shift toward Postgres-only with IHP's ORM acceptable (re-evaluate
  IHP), or toward third-party API clients (adopt an external IdP).

---

## Appendix A. Evidence base

Checked on 2026-10-26 by reading package metadata and source. Nothing was
compiled: no GHC toolchain was available in the environment used.

| Package | Version (upload date) | What was checked |
|---------|-----------------------|------------------|
| yesod-core | 1.7.0.1 (this tree) | `isAuthorized`, CSRF helpers, `WaiSubsiteWithAuth`, ChangeLog |
| yesod-auth | 1.6.12.1 (this tree) | `onLogin`, `authRoute`; ChangeLog "Support yesod-core 1.7" |
| yesod-auth-oauth | this tree | depends on `authenticate-oauth` (OAuth 1.0a) |
| yesod-form | this tree | `runInputPost` |
| yesod-auth-oauth2 | 0.8.0.0 (2026-01-07) | changelog, provider modules, imports, dependency bounds |
| hoauth2 | 2.15.2 (2026-08-10) | latest version |
| miso | 1.14.0.0 (2026-09-13) | cabal flags (`native`, `aeson`, `ssr`), `Miso.Router`, `Miso.Html.Render`, hydration mount point, `Miso.Native.Module` |
| servant | 0.20.3.0 (2026-08-21) | exposes `Servant.API.Experimental.Auth` |
| servant-auth | 0.4.10.0 (2026-08-21) | depends on `jose`, `lens`, `aeson` |
| servant-miso-client | 0.2.1.0 (git master) | not on Hackage; `HasClient` instances; exported `Request (..)` |

Third-party documentation:

- [miso-lynx repository](https://github.com/haskell-miso/miso-lynx): upstreamed into miso
- [servant-miso-client](https://github.com/haskell-miso/servant-miso-client)
- [Lynx native modules](https://lynxjs.org/guide/use-native-modules)
- [Lynx networking](https://lynxjs.org/guide/interaction/networking) and
  [fetch API](https://lynxjs.org/api/lynx-api/global/fetch)
