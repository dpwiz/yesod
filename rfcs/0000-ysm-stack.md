# RFC 0000: YSM stack (Yesod + Servant + Miso)

- **Status:** Draft, for discussion
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
| Data API (resource server)   | **Servant** (+ servant-auth)         | Typed JSON API, stateless JWT verification                                                           |
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
shared/     DTOs (aeson), Servant API type, AuthUser.
            Depends only on servant / servant-auth core (no jose).
            Must build with GHC's JS or wasm backend.
app-core/   Miso model, actions, update, typed API client.
web/        Desktop-oriented Miso views + web entry point.
mobile/     Mobile-oriented Miso views + Android entry point.
server/     Yesod (pages, auth, token issuance) + Servant API + Beam.
```

Platform differences live in the views. Model, actions and API calls are shared.

### 4.3 Authentication and tokens

The API is declared once with `Auth '[JWT, Cookie] AuthUser` (servant-auth),
so handlers get the same `AuthUser` whether the token came in a header or a cookie
**[reported]**.

**Web client (same origin):**

- On login (`onLogin`), Yesod also sets a short-lived JWT cookie: `HttpOnly`,
  `Secure`, `SameSite=Strict`, `Path=/api`, plus servant-auth's XSRF cookie.
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

1. **Miso on Android [unverified]:** which delivery route (WebView-style shell vs.
   Miso's native-rendering efforts such as miso-lynx) is viable, and does it give
   access to Android Keystore-backed storage for the refresh token? This is
   the least certain part of the plan and could reshape §4.3.
2. **Miso routing vs. Servant [unverified]:** older Miso releases built client
   routing on Servant API types; newer ones have their own router. Confirm
   against the version chosen.
3. **`shared/` cross-compilation [unverified]:** confirm `servant`,
   `servant-auth` (core) and the client library build on the chosen GHC JS or
   wasm backend without pulling in `jose`.
4. **`yesod-auth-oauth2` [reported]:** confirm the needed providers, current
   maintenance, and compatibility with the Yesod version in use.
5. **Hand-rolled authorization server:** the PKCE and refresh-token code is
   security-sensitive. It needs tests and review, or should be replaced by an
   external IdP (§5).
6. **Compile times:** Yesod TH plus Servant type-level APIs in one `server`
   package may be slow; splitting packages may be needed.
7. **SSR/hydration for Miso islands:** whether the server should render Miso
   `View`s to HTML for hydration, and how that fits Hamlet layouts.

## 7. Suggested validation spikes

Each spike should be small enough to throw away.

1. **Mobile login:** Miso Android build that runs the PKCE flow against a stub
   Yesod server and stores the refresh token securely. *(Addresses risk 1, 5.)*
2. **Shared package:** `shared/` with one Servant endpoint and DTOs, used by both
   the server and a Miso web build. *(Addresses risks 2, 3.)*
3. **Locked-down Yesod:** `yesod-auth-oauth2` + Beam users table + exhaustive
   `isAuthorized` + `defaultCsrfMiddleware`, with a `yesod-test` suite.
   *(Addresses risk 4.)*
4. **Mounted API:** Servant app under `WaiSubsiteWithAuth`, accepting JWT via
   cookie and bearer. *(Validates §4.3–4.4.)*

## 8. What would make us reject this RFC

- Spike 1 shows no viable secure token storage or login flow for Miso on Android.
- Spike 2 shows Servant types can't be shared with the Miso builds, removing the
  main reason for the Servant layer (fall back to plain Yesod JSON handlers).
- Requirements shift toward Postgres-only with IHP's ORM acceptable (re-evaluate
  IHP), or toward third-party API clients (adopt an external IdP).
