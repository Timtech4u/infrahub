# Frontend Error Management & Authentication — Improvements Backlog

> Captured 2026-05-29 after reviewing `pog-infp-468-us1-graphql-error-formatter`.
> Each item is independent — pick whichever fits the next iteration.

## Context

The `INFP-468` US1 branch wired the backend GraphQL error catalogue into the
frontend `errorLink` and introduced a typed `FetchError` for REST. While
reviewing it, several rough edges in the surrounding error / auth code surfaced
that were either pre-existing or deliberately scoped out. This document is the
parking lot.

Touched files for reference:

- `frontend/app/src/shared/api/graphql/graphqlClientApollo.tsx`
- `frontend/app/src/shared/api/graphql/errors.ts`
- `frontend/app/src/shared/api/rest/fetch.ts`
- `frontend/app/src/entities/authentication/domain/refresh-access-token.ts`
- `frontend/app/src/entities/authentication/ui/useAuth.tsx`
- `frontend/app/src/pages/auth-callback.tsx`
- `frontend/app/src/pages/login.tsx`

---

## Error management

### E1 — Wrap retried request in `Bearer ` prefix (dead code, not a bug today)

**Where:** `graphqlClientApollo.tsx` → `retryWithRefreshedToken`.

The retried request sets the header as:

```ts
authorization: newToken.access_token,
```

while `authLink` uses `` `Bearer ${accessToken}` ``. The backend
(`backend/infrahub/api/dependencies.py:43-51,95-113`) uses FastAPI's
`HTTPBearer` scheme, which requires the `Bearer ` prefix and silently drops
a bare-token header. **However**, both `/auth/login` and `/auth/refresh`
set `access_token` as an httpOnly cookie
(`backend/infrahub/api/auth.py:40,75`), and `get_access_token` /
`get_current_user` fall back to that cookie when the JWT header is
absent/invalid. So the retried request *does* authenticate — via the
cookie, not the header. The header is misleading dead code.

Fix anyway, for two reasons:

1. Anyone reading the retry block assumes the header is what authenticates
   the replay. It isn't, and the silent fallback hides the bug.
2. If the cookie path is ever disabled (cross-origin deploy, stricter
   `SameSite`, non-browser caller hitting the same code), the retry breaks
   with no observable signal until production.

```ts
authorization: `Bearer ${newToken.access_token}`,
```

Add an E2E test that exercises the TOKEN_EXPIRED retry path **with cookies
disabled** so the header is the only thing authenticating the replay.

### E2 — Cap the silent-refresh retry

**Where:** `graphqlClientApollo.tsx` → `errorLink` switch arm for `TOKEN_EXPIRED`.

If the refreshed token is itself rejected as `TOKEN_EXPIRED` (clock skew, race
with a server-side revoke, malformed token), the errorLink loops indefinitely:
the retried operation triggers another `TOKEN_EXPIRED`, which calls
`retryWithRefreshedToken` again, and so on.

Minimum fix: stash a counter on `operation.getContext()` and bail to
`redirectToLogin()` after one retry.

```ts
const attempts = (operation.getContext().authRetryCount ?? 0) + 1;
if (attempts > 1) return redirectToLogin();
operation.setContext({ authRetryCount: attempts, ... });
```

### E3 — Normalise the 2xx-with-errors branch in `auth-callback.tsx`

**Where:** `pages/auth-callback.tsx:33`.

```ts
if (result.errors) {
  throw result;
}
```

Now that `fetchUrl` throws a real `FetchError` carrying `errors`, the catch
handler sees two shapes:

- non-2xx → `FetchError` (with `.errors?: RestErrorItem[]`)
- 2xx with `errors` → raw response body object

`setErrors(error.errors)` happens to work for both, but the type contract is
fuzzy. Normalise to:

```ts
if (result.errors) {
  throw new FetchError(200, result.errors);
}
```

…and type the `errors` state as `RestErrorItem[] | null` instead of `any`.

### E4 — Tighten the `RestErrorItem.extensions.code` type

**Where:** `shared/api/rest/fetch.ts`.

Today: `extensions: { code: number }`. The REST envelope still surfaces an
integer HTTP status, but the GraphQL envelope (catalogue) uses a string
`code` plus an integer `http_status`. If the REST endpoints ever migrate to
the catalogue we'll need a discriminated union. Worth a comment at the type
declaration noting which envelope this matches and pointing at the catalogue
mirror in `errors.ts`.

### E5 — Delete the hand-written catalogue mirror once US2 lands

**Where:** `shared/api/graphql/errors.ts`.

The file header is explicit: it must be removed once `INFP-468 US2` (T027–T030)
generates the bindings. Add a tracking item in US2's tasks.md and a CI check
that fails if both `errors.ts` and the generated module coexist after that
point.

### E6 — Add a backend pytest that snapshots the catalogue codes + field names

**Where:** `backend/tests/unit/errors/test_catalogue_snapshot.py` (new).

Until US2 lands, the only protection against frontend/backend drift is human
diligence. A snapshot test that asserts the JSON-serialised catalogue (codes
and payload field names) matches a checked-in file gives us a noisy failure
on the backend when a payload model gains/loses a field. The frontend can
read the same snapshot in `errors.test.ts` to assert parity locally.

### E7 — Surface a typed alert message for `UNDEFINED_ERROR`

**Where:** `graphqlClientApollo.tsx` → `notifyUser`.

`UNDEFINED_ERROR` codes are catalogue gaps — they should be visible to
engineers in dev builds (banner + console.warn referencing the original
message) rather than silently toasted as a generic error. Gate it on
`import.meta.env.DEV` so prod stays unchanged.

### E8 — Telemetry for unmatched error codes

**Where:** `graphqlClientApollo.tsx` → `errorLink`.

Forward `UNDEFINED_ERROR` occurrences to whatever telemetry sink the app uses
(or stash them in localStorage during dev). This is what gives us the
"catalogue gap" signal the spec promises.

---

## Authentication

### A1 — `refreshAccessToken` does a hard reload on failure

**Where:** `entities/authentication/domain/refresh-access-token.ts:17,27`.

On both the "no refresh token" and "refresh failed" branches the function does
`window.location.reload()`, which drops in-flight React Query state and
swallows the `from` route. The errorLink already has `redirectToLogin()` that
preserves `pathname !== "/login"`. Make `refreshAccessToken` *throw* and let
the caller (`retryWithRefreshedToken` and `useAuth`) decide whether to redirect.

Side-benefit: it stops the double-redirect that currently happens when both
errorLink AND `refreshAccessToken` think it's their job to navigate.

### A2 — `redirectToLogin()` loses the `from` location

**Where:** `graphqlClientApollo.tsx:147` (`window.location.assign("/login")`).

Hard navigation is necessary (errorLink runs outside React Router), but we can
still encode the original path:

```ts
const from = window.location.pathname + window.location.search;
window.location.assign(`/login?from=${encodeURIComponent(from)}`);
```

…and have `LoginPage` consume `searchParams.get("from")` as a fallback when
`location.state?.from` is absent (which it is after a hard nav).

### A3 — Token persistence boundary

**Where:** `entities/authentication/utils.ts` + `useAuth.tsx`.

`removeTokensInLocalStorage` is called from four places (the errorLink, the
refresh flow, the logout flow, the auth hook). Each call site has a slightly
different idea of what "log the user out" means (clear tokens, clear React
Query cache, redirect, reload). Consolidate behind a single
`logoutLocally({ redirect?: boolean, reason?: "expired" | "manual" | "refresh-failed" })`
helper and replace the four ad-hoc call sites. Same payoff as A1: one place
that decides what happens.

### A4 — `authLink` reads `localStorage` synchronously on every request

**Where:** `graphqlClientApollo.tsx:48`.

Each operation does `localStorage.getItem(ACCESS_TOKEN_KEY)`. localStorage
access is synchronous and main-thread, and on top of that the value is also
held in `useAuth`'s React state. Read it from a module-scoped variable that is
kept in sync via the `storage` event + `useAuth.setToken`. Micro-optimisation,
but worth doing if/when we add Apollo subscriptions or batch requests.

### A5 — SSO callback rerenders on every config tick

**Where:** `pages/auth-callback.tsx:22`.

The `useEffect` depends on `[config, protocol, provider]`. `config` is an
object; whether the provider returns a stable reference depends on
`config-provider`. Worth verifying — if it isn't stable, every config refetch
re-runs the SSO exchange. Quick fix: pick out `config.sso.enabled` and a
provider-id string as the actual deps.

### A6 — `login.tsx` renders an `any`-typed error array

**Where:** `pages/login.tsx:26`.

```ts
location?.state?.errors?.map(
  (error: { extensions: { code: number }; message: string }, index: number) => (...)
)
```

Now that `RestErrorItem` is exported from `shared/api/rest/fetch.ts`, type the
inline annotation as `RestErrorItem` and drop the duplicate shape declaration.

### A7 — `useAuth` doesn't reconcile with cross-tab logout

**Where:** `entities/authentication/ui/useAuth.tsx`.

Logging out in one tab leaves other tabs authenticated until the next
TOKEN_EXPIRED or page reload. A `storage` event listener that re-reads
`ACCESS_TOKEN_KEY` and updates the hook's state would close this gap. Also
covers the case where a developer manually clears storage from devtools.

### A8 — Login error rendering is bare

**Where:** `pages/login.tsx`.

Errors render as plain red `<p>` tags. With the new `RestErrorItem` shape the
page could show the catalogue `code` when present (post-US2 backend) so SSO
failures are diagnosable from the UI without devtools. Low priority but
trivial once A6 is in.

---

## Suggested ordering

1. **E2** (retry cap) — guards against an infinite TOKEN_EXPIRED loop. Highest
   real risk in current code.
2. **E1** (Bearer prefix) — single-line cleanup; not a correctness bug today
   thanks to the cookie fallback, but the dead header invites future
   regressions.
3. **A1 + A3** (consolidate auth side-effects) — clears the conceptual mess
   before US2 layers more on top.
4. **A2** (preserve `from`) — UX win that becomes user-visible the moment a
   token expires mid-session.
5. **E6** (catalogue snapshot test) — prevents drift before US2 ships.
6. Everything else — opportunistic.
