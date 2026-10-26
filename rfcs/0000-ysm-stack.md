# RFC 0000: YSM stack (Yesod + Servant + Miso)

- **Status:** Draft, for discussion (revised 2026-10-26: §6 research pass)
- **Created:** 2026-10-26
- **Scope:** A web application with server-rendered pages, Miso front ends for web
  and Android, and access gated behind OAuth-provided users.

> This RFC records one candidate architecture and the reasoning behind it.
> It is not a decision. It is meant to be challenged, prototyped against, and
> amended or rejected. Claims are tagged:
> **[verified]** = checked against the Yesod source tree,
> **[reported]** = from general ecosystem knowledge and not re-checked here,
> **[unverified]** = an assumption that needs a spike before relying on it.

---

## 1. Summary

Split the system into three roles:

| Role                         | Candidate                            | Responsibility                                                                                       |
|------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------|
| Auth core + HTML site        | **Yesod** (Hamlet templates)         | OAuth login with upstream providers, users table, sessions, server-rendered pages, issuing API tokens |
| Data API (resource server)   | **Servant** (+ `AuthProtect`, jose)   | Typed JSON API, stateless JWT verification                                                           |
| Interactive clients          | **Miso** (web and Android builds)    | Rich UI where needed; separate views for desktop and mobile on a shared core                         |

Persistence uses **Beam** in all server components. i18n is out of scope.

## 2. Context and requirements

1. **R1** Most of the site is server-rendered HTML. It should be maintainable and
   checked at compile time as far as possible.
2. **R2** Almost everything is behind login. Identities come from external
   OAuth/OIDC providers (e.g. Google, GitHub).
3. **R3** Miso adds interactivity to parts of the web site ("islands").
4. **R4** A Miso Android app talks to the same backend, with a UI tuned
   for tasks that happen more often on phones.
5. **R5** Beam is the database layer.
6. **R6** Avoid lock-in to a single toolchain (e.g. requiring Nix) where practical.

## 3. Findings that led here

### 3.1 Template engine: Hamlet over Ede

An earlier variant used Ede templates. Ede takes an untyped JSON-like context,
so with Ede:

- `@{Route}` interpolation is unavailable, and URLs must be rendered to `Text` in
  Haskell and passed in. Links are checked only where the context is built.
- Template variable typos become runtime errors or empty output.
- Yesod widgets (bundled HTML/CSS/JS with deduplicated `addScript`/`addStylesheet`)
  and widget-based `yesod-form` rendering lose most of their value.

`yesod-core` depends on `shakespeare` and `blaze` anyway **[verified]**
(`yesod-core/yesod-core.cabal`), so Ede adds a dependency without removing one.
With Ede, Yesod's advantage over a minimal framework shrinks to routing,
sessions and the auth ecosystem. **Hamlet is assumed from here on.**

### 3.2 What Yesod provides that minimal frameworks (scotty, twain, raw WAI) don't

- Typed routes generated from `parseRoutes`, with compile-time overlap checks
  **[verified]** (`yesod-core/src/Yesod/Routes/Overlap.hs`).
- A central `isAuthorized :: Route site -> Bool -> HandlerFor site AuthResult`
  hook **[verified]** (`yesod-core/src/Yesod/Core/Class/Yesod.hs`). Writing it
  without a wildcard case gives a non-exhaustive-match warning for every new
  route, which suits a site that is locked down by default (R2).
- Encrypted client-side sessions, flash messages, body size limits, logging and
  `yesodMiddleware` hooks **[verified]**.
- CSRF protection via `defaultCsrfMiddleware` plus header-based checks
  (`checkCsrfHeaderNamed`, `setCsrfCookie`) **[verified]**. Note that it is
  **opt-in**: it must be added to `yesodMiddleware`.
- `yesod-auth` with redirect-to-login (`authRoute`), return to the original page
  after login, and `onLogin`/`onLogout` hooks **[verified]**
  (`yesod-auth/Yesod/Auth.hs`).
- Applicative input validation without markup (`runInputPost`, `runInputGet`)
  **[verified]** (`yesod-form/Yesod/Form/Input.hs`).
- `yesod-static` (hashed static routes), `yesod-test` (sessions, CSRF, DOM
  assertions), subsites **[verified as packages in this tree]**.

Costs: Template Haskell, long compile times, a heavier dependency tree, and a
fixed `HandlerFor site` monad.

### 3.3 OAuth support: use `yesod-auth-oauth2`, not the in-tree package

The in-tree `yesod-auth-oauth` package is **OAuth 1.0a**: it depends on
`authenticate-oauth` **[verified]** (`yesod-auth-oauth/yesod-auth-oauth.cabal`).
For OAuth2/OIDC providers the candidate is the separately maintained
`yesod-auth-oauth2` package **[reported]**.

### 3.4 Comparison with Servant-only and IHP

| Concern                        | Yesod + Hamlet                     | Servant only                                       | IHP                                    |
|--------------------------------|------------------------------------|----------------------------------------------------|----------------------------------------|
| Server-rendered site (R1)      | Strong                             | Weak: no layouts, flash messages or form re-display | Strong (HSX, AutoRoute)                |
| Browser OAuth + login redirect (R2) | Strong (yesod-auth + oauth2)  | DIY (`hoauth2`, sessions, 302 vs 401)              | Built in, short provider list **[reported]** |
| Sharing types with Miso (R3, R4) | Weak: route types live in a TH-heavy server package | Strong: API type in a small shared package | Weak: Miso not a first-class path |
| Beam (R5)                      | Fine                               | Fine                                               | Conflicts: IHP's ORM is its core       |
| Toolchain (R6)                 | Cabal or Stack                     | Cabal or Stack                                     | Nix-centric **[reported]**             |

IHP looks strong apart from R5 and R3/R4. If those requirements change, it
should be reconsidered.

The table points to a **hybrid**: Yesod where the requirement is website and
auth, Servant where it is an API shared with Miso clients.

## 4. Proposal

### 4.1 Overall shape

```
                ┌──────────── Yesod (auth core + HTML pages) ────────────┐
 OAuth provider ◄┤ yesod-auth-oauth2 → users table (Beam)                │
                │ issues: session cookie (pages) + API tokens            │
                └──────┬──────────────────────────────┬──────────────────┘
       JWT in HttpOnly │ cookie                       │ JWT bearer + refresh token
                       ▼                              ▼
              Miso web (same origin)          Miso Android app
                       │                              │
                       └──────────► Servant API ◄─────┘
                         checks signature only (public key), no DB lookup per request
```

### 4.2 Package layout

```
shared/     DTOs, Servant API type, AuthUser.
            Depends on servant (core) and miso (built with its `aeson` flag) only.
            NOT servant-auth: it pulls in jose and lens (§6.1).
            Must build with GHC's JS or wasm backend.
app-core/   Miso model, actions, update, typed API client.
web/        Desktop-oriented Miso views + web entry point.
mobile/     Mobile-oriented Miso views + Android entry point.
server/     Yesod (pages, auth, token issuance) + Servant API + Beam.
```

Platform differences live in the views. Model, actions and API calls are shared.

### 4.3 Authentication and tokens

The API type in `shared/` marks protected endpoints with servant core's
`AuthProtect "jwt"` (`Servant.API.Experimental.Auth`) **[verified on Hackage:
servant-0.20.3.0]**, not servant-auth's `Auth '[JWT, Cookie]`, which would drag
`jose` into the client builds (§6.1). The two sides then interpret it:

- **Server:** servant-server's generalized auth (`AuthHandler`) checks the
  `Authorization: Bearer` header first, then falls back to the JWT cookie plus
  XSRF header. Verification uses `jose` directly (or servant-auth-server's helpers),
  and either way handlers get the same `AuthUser`.
- **Miso clients:** `servant-miso-client` has no `AuthProtect` instance, but it
  exports `HasClient (..)` and `Request (..)`, so a short orphan instance
  (modelled on its `Header` instance) can add the header **[verified in source]**.
  The web build can pass the XSRF value there; the Android build passes the bearer token.

**Web client (same origin):**

- On login (`onLogin`), Yesod also sets a short-lived JWT cookie: `HttpOnly`,
  `Secure`, `SameSite=Strict`, `Path=/api`, plus a readable XSRF cookie.
  Miso echoes the XSRF cookie into a header.
- When the API returns 401 (token expired), Miso calls a Yesod endpoint
  (e.g. `/auth/api-cookie`). It checks the Yesod session and reissues the cookie.
  The Yesod session acts as the refresh credential, and the browser never sees a
  token.
- Tokens are deliberately **not** kept in `localStorage`.
- Serving the API under `/api` on the same origin avoids CORS.

**Android client (native-app OAuth, RFC 8252 + PKCE):**

1. The app opens a Custom Tab at `/auth/mobile/start?code_challenge=…`.
2. Yesod stores the challenge and runs the normal provider login.
3. Yesod redirects to a verified Android App Link with a short-lived (~60 s)
   one-time code.
4. The app POSTs code + `code_verifier` to `/auth/mobile/token` and receives a
   short access JWT (5–15 min) and a rotating refresh token.
5. API calls use `Authorization: Bearer …`. The XSRF check is not needed on
   this path.

Embedded-WebView login pages are ruled out. Major providers block them.

**Token policy:**

- Asymmetric signing (EdDSA or ES256). Yesod holds the private key, and the API
  holds only the public key.
- Minimal claims: `sub` (internal `UserId`, not the provider id), `aud`, `iss`,
  `exp`, optionally roles. Ownership checks are done against data in the API.
- Refresh tokens are stored hashed in a Beam table, rotated on every use, and
  the whole family is revoked if an old one is reused. Logging out or banning
  a user deletes the rows.

This makes Yesod a minimal first-party OAuth2 authorization server: a few routes
and two tables (pending PKCE codes, refresh tokens).

**Sketch (Yesod side):**

```haskell
issueApiToken :: UserId -> Handler BL.ByteString
issueApiToken uid = do
  App{appJwtSettings} <- getYesod
  roles <- runBeam (lookupRoles uid)
  exp'  <- addUTCTime 600 <$> liftIO getCurrentTime
  either (const $ permissionDenied "token") pure
    =<< liftIO (makeJWT (AuthUser uid roles) appJwtSettings (Just exp'))
```

### 4.4 Hosting the API

Two options, not mutually exclusive:

- **Same process:** mount the Servant app inside Yesod with
  `WaiSubsiteWithAuth`. It "applies parent site's middleware and isAuthorized"
  **[verified]** (`yesod-core/src/Yesod/Core/Types.hs`). JWT auth makes a
  session bridge unnecessary.
- **Separate deployment:** possible because verification needs only the public
  key.

### 4.5 API evolution

Android installs lag behind deployments. So:

- Version the API from the start (`"v1" :> …`), with additive-only changes within
  a version.
- Expose a minimum-supported-client endpoint so the app can ask users to update.
- Mobile-specific endpoints (coarser, offline-friendly payloads) are acceptable
  within the same API type.

## 5. Alternatives considered

| Alternative | Why not (for now) | What would change the call |
|-------------|-------------------|----------------------------|
| Yesod + Ede | Loses checked templates and widgets; narrows Yesod's advantage (§3.1) | Designers must edit templates without a Haskell toolchain |
| scotty / twain / raw WAI | Sessions, CSRF, auth flows, typed links all DIY | Site shrinks to an API plus a few pages |
| Servant only | Website and browser-auth concerns are DIY (§3.4) | Server-rendered part becomes negligible |
| IHP | Conflicts with Beam; Miso not first-class; Nix-centric | Beam dropped, or Miso limited to small islands |
| External IdP (Keycloak, Zitadel, Ory Hydra) instead of Yesod-issued tokens | More moving parts for a first-party-only setup | Third-party clients, scopes or consent, SSO across products, compliance needs |
| Plain Yesod JSON handlers for Miso (no Servant) | No typed client or links for the Miso side | Few endpoints; acceptable starting point if the Servant layer is deferred |

## 6. Risks and open questions

Researched on 2026-10-26 against Hackage releases and package sources:
miso-1.14.0.0 (2026-09-13), servant-0.20.3.0 and servant-auth-0.4.10.0
(2026-08-21), yesod-auth-oauth2-0.8.0.0 (2026-01-07), hoauth2-2.15.2,
servant-miso-client 0.2.1.0 (git), and yesod-core 1.7.0.1 in this tree.
Nothing below was compiled: no GHC toolchain was available in the
environment where this was checked.

### 6.1 Resolved (or narrowed)

1. **Miso on Android: viable, through Lynx, with small native modules.**
   - Miso supports iOS/Android/HarmonyOS via LynxJS. The former `miso-lynx`
     project "has been upstreamed into miso proper" and is enabled with miso's
     `native` cabal flag, which exposes `Miso.Native.*` **[verified in source]**.
     The entry point is `Miso.Native.native`, not `miso`/`startApp`.
   - Haskell code can call Lynx native modules via `Miso.Native.Module.callNativeModule`
     (since miso 1.13, background thread only) **[verified in source]**. Lynx native
     modules on Android are Kotlin/Java classes extending `LynxModule` with
     `@LynxMethod` methods **[reported: Lynx docs]**.
   - Therefore: **secure refresh-token storage** = a small Kotlin module over Android
     Keystore that we own, and the **PKCE flow** = a Kotlin module that opens a
     Custom Tab plus App Link intent handling in the host Activity. Neither is
     provided out of the box: the Lynx guide's storage example uses plain
     `SharedPreferences` **[reported]**.
   - Lynx's `fetch` is Web-compatible but has no CORS, redirect or keepalive
     handling, and no FormData/Blob **[reported: Lynx docs]**. This matches
     §4.3's choice of bearer tokens over cookies on mobile. Login must
     not rely on fetch following redirects (it runs in the Custom Tab anyway).
   - `Miso.Native` is marked `Stability: experimental` **[verified]**. This is
     now the main maturity risk, rather than an open feasibility question.

2. **Miso routing vs. Servant: decoupled.** miso-1.14 does not depend on servant.
   `Miso.Router` is its own generic `Router` class ("servant-style", not
   servant-based) **[verified]**. Consequences:
   - Servant's role on the client is **the API client only**, via
     `servant-miso-client` (recommended by the miso README).
   - Client-side page routes (Miso `Router`) and server pages (Yesod
     `parseRoutes`) stay separate type definitions. Links between them are not
     type-checked across the boundary. Keep that boundary small: a handful of
     entry URLs.
   - `servant-miso-client` is **not on Hackage** (git only, README suggests
     `tag: master`). Pin a commit. It defines its own `HasClient` and covers
     `Capture`, `QueryParam`, `QueryFlag`, `Header`, `ReqBody`, `Verb`, `NoContentVerb`,
     `Fragment`, `CaptureAll` and `:<|>`, but not `AuthProtect` (see §4.3)
     **[verified in source]**.

3. **`shared/` cross-compilation: `servant-auth` must stay out.**
   - `servant-auth` (the "core" package) depends on **`jose`, `lens`** and `aeson`
     **[verified: servant-auth-0.4.10.0.cabal]**. `jose` pulls in `crypton`. The
     earlier draft's "servant-auth core, no jose" was wrong. §4.2–4.3 now use
     servant core's `AuthProtect` instead.
   - `servant` core is a dependency of `servant-miso-client`, which is maintained
     by miso's author for JS/wasm targets. That is good evidence it cross-compiles
     **[reported, not built]**.
   - **JSON class mismatch:** by default miso uses its own `Miso.JSON` classes, and
     `servant-miso-client` decodes with them. Servant-server needs aeson's.
     miso's `aeson` flag makes `Miso.JSON` re-export aeson's classes
     **[verified: miso.cabal]**. So build miso with `+aeson` everywhere and derive
     DTO instances once with aeson. The remaining check is that aeson builds on the
     chosen JS/wasm backend: that flag exists for that purpose, but it is
     **[unverified]** here.

4. **`yesod-auth-oauth2`: current and very likely compatible.**
   - 0.8.0.0 was released 2026-01-07 and supports GHC 9.12 and hoauth2-2.15
     (latest is 2.15.2) **[verified: Hackage changelog]**.
   - Providers include Google, GitHub, GitLab, AzureAD/AzureADv2, Auth0,
     Bitbucket, Slack, Salesforce and others, plus the generic `authOAuth2`
     for any other OAuth2 provider **[verified in source]**.
   - It bounds `yesod-core >= 1.6.0` with no upper bound, and imports only
     the public `Yesod.Core`/`Yesod.Auth` APIs. Its only TH is `$(logError)`.
     The breaking changes in yesod-core 1.7 are in route TH codegen internals,
     so they should not affect it. In-tree `yesod-auth` 1.6.12.1 already
     "Support[s] `yesod-core` 1.7" **[verified in source; not compiled]**.
   - It protects the upstream flow with a session-stored `state` value
     (`setSessionCSRF`/`verifySessionCSRF`) and doesn't do PKCE upstream. That
     is fine for a confidential server-side client. PKCE in §4.3 is between
     the Android app and *our* Yesod server, which is a separate concern.

5. **SSR/hydration: whole-body only, so islands are client-rendered.**
   - `Miso.Html.Render.toHtml`/`toHtmlWith` render a `View` to a lazy
     `ByteString` on the server, which is easy to embed in Hamlet with
     `preEscapedToMarkup` **[verified]**.
   - But hydration (`miso`, `prerender`) forces `mountPoint = Nothing` and
     "always mounts to `<body>`" **[verified in source]**.
     Hydration therefore suits pages where Miso owns the whole `<body>` (Hamlet
     supplies only `<head>`). It does not suit islands inside a Hamlet layout.
   - Islands use `startApp` with the component's `mountPoint = Just "<element-id>"`
     (resolved via `getElementById`) **[verified]**, rendered on the client into
     a Hamlet-provided `<div id=…>`. Server markup can still go inside the div as
     a loading placeholder, but it won't be hydrated.

### 6.2 Still open

1. **Several islands on one page [unverified]:** whether several top-level
   `startApp` roots can coexist on one page, or whether one root should mount
   child components into multiple places.
2. **Hand-rolled authorization server:** the PKCE and refresh-token code is
   security-sensitive. It needs tests and review, or should be replaced by an
   external IdP (§5). Not resolvable by research.
3. **Compile times:** Yesod TH plus Servant type-level APIs in one `server`
   package may be slow. Partial mitigation: yesod-core 1.7's split route
   compilation (`setFocusOnNestedRoute`) can generate nested route blocks in their
   own modules **[verified: yesod-core ChangeLog]**. Needs measurement.
4. **Toolchain for JS/wasm + Lynx builds:** miso's reference workflows (sampler,
   former miso-lynx counter bundle) lean on Nix **[reported]**, which is in tension
   with R6. Confirm a non-Nix path or accept Nix for client builds only.
5. **aeson on the client backend [unverified]:** see §6.1 item 3.

## 7. Suggested validation spikes

Each spike should be small enough to throw away.

1. **Mobile login:** Lynx host app with two Kotlin modules (Custom Tab +
   App Link callback, Keystore storage), Miso `native` build calling them via
   `callNativeModule`, PKCE against a stub Yesod server.
   *(Addresses §6.1 item 1 and §6.2 item 2.)*
2. **Shared package:** `shared/` with one `AuthProtect` endpoint and aeson DTOs,
   miso built `+aeson`, used by the server and by web + `native` Miso builds,
   with the orphan `HasClient (AuthProtect …)` instance.
   *(Addresses §6.1 items 2–3 and §6.2 items 4–5.)*
3. **Locked-down Yesod:** `yesod-auth-oauth2` 0.8 against yesod-core 1.7 + Beam
   users table + exhaustive `isAuthorized` + `defaultCsrfMiddleware`, with a
   `yesod-test` suite. *(Confirms §6.1 item 4 by actually compiling.)*
4. **Mounted API:** Servant app under `WaiSubsiteWithAuth` with an `AuthHandler`
   accepting bearer or cookie + XSRF. *(Validates §4.3–4.4.)*
5. **Island page:** Hamlet layout with two `startApp` islands by `mountPoint`.
   *(Addresses §6.2 item 1.)*

## 8. What would make us reject this RFC

- Spike 1 shows `Miso.Native` is too immature for a production Android app, or
  the Kotlin bridge for the login flow or secure storage is impractical.
- Spike 2 shows the shared Servant API type or aeson can't build for the Miso
  targets, removing the main reason for the Servant layer (fall back to plain
  Yesod JSON handlers and `Miso.Fetch`).
- Requirements shift toward Postgres-only with IHP's ORM acceptable (re-evaluate
  IHP), or toward third-party API clients (adopt an external IdP).
