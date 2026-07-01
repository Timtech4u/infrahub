---
Title: User & Global Preferences
Author:
  - Paul Lemesle
Status: draft
---

# User & Global Preferences

## Summary

Introduce two persistent preference surfaces, modelled as **`StandardNode` objects** (the internal
object type used by `Branch`/`Root`, `backend/infrahub/core/node/standard.py`) — **not** schema
nodes:

- `GlobalPreference` — admin-defined defaults that apply to every user (one singleton).
- `UserPreference` — per-user overrides (one per account, keyed by `account_id`).

A single backend-computed query returns the **effective** preferences for the calling user (global
merged with their personal overrides), so the frontend never has to merge them itself. There is no
auto-generated schema CRUD; the entire read/write surface is **custom GraphQL** (like `Branch`),
which is exactly what gives us control over visibility.

V1 ships only two fields — `date_format` and `timezone` — to validate the model, query, mutations,
and UI plumbing end to end. Additional preferences (dark mode, etc.) land in follow-up tickets once
the foundation is proven.

Three adjacent concerns get their own spec/ticket and are explicitly **not** part of V1:

- Saved views — `dev/specs/2026-04-saved-views.md`
- Saved filters — `dev/specs/2026-04-saved-filters.md`
- "Show extra fields" toggle persistence — `dev/specs/2026-04-show-extra-fields.md`

## Why `StandardNode` and not a schema node

This is the central design decision, reversing an earlier draft that used `CoreGlobalPreference` /
`CoreUserPreference` schema nodes.

A schema `Node` gets an **auto-generated generic GraphQL query** and is governed by **per-kind**
object permissions. Infrahub permissions cannot restrict reads **per row** — so any account with
read on the kind can query *every* user's preference row. There is no way to hide user B's settings
from user A on a schema node short of suppressing the generic query and bolting a filter onto a
custom resolver, which is fragile.

A `StandardNode` (e.g. `Branch`) has **no schema-registry entry and no auto-generated GraphQL**. The
only read/write paths are the resolvers we write by hand, so visibility is **structural**: there is
no generic query that could leak another user's preferences, and the custom query binds to the
calling account. Secondary benefits: it keeps the product's core schema lean, and `StandardNode`s
are global (not branch-versioned), which matches preferences exactly.

> Profiles were considered for the "defaults + override" merge and **rejected**: they are a
> schema-`Node` feature (irrelevant to `StandardNode`), and a two-field global→user merge is three
> lines in a resolver. No profiles are used.

## Problem Statement

- UI defaults (date format, timezone) live in component state or the browser's locale. They cannot be set centrally by an organization, and a user's choice does not survive a device switch.
- There is no place for an admin to express "use ISO dates organisation-wide" or "default everyone to UTC".
- There is no backend-stored concept of "preferences" that the SDK or other clients can read in a controlled way.

## Solution Overview

Two `StandardNode` objects, one custom read query that fuses them.

| Object | Cardinality | Who writes | Who reads |
|---|---|---|---|
| `GlobalPreference` | Singleton | Holders of `manage_global_preferences` (super admins implicitly) | Any authenticated account (via the effective query) |
| `UserPreference` | One per account (`account_id`) | The owning account only | The owning account only (admins via tooling) |

Effective resolution per field: **user value if set, else global value, else the browser's own value** (the browser-resolved timezone, and the browser locale's date/time formatting). There is no fixed built-in pattern fallback — when neither preference is set the UI renders exactly as the user's browser would by default.
Defaults live in the frontend (so the API returns `null` for "no opinion stored" and the SDK can
apply its own).

## Success Criteria — V1

- An admin (holder of `manage_global_preferences`) can set `date_format` and `timezone` once for the organisation; any authenticated user without a personal override sees those values.
- A user can override either field on their own preferences page; the override takes effect on next page load and is visible from any device.
- Clearing a user override falls back to the global value; clearing the global value falls back to the frontend default.
- **A user cannot read another user's `UserPreference`** — there is no generic query, and the custom query only ever returns the caller's own row. (This is the primary reason for the `StandardNode` model.)
- A single GraphQL query returns the effective value to render with — no client-side merging.

## V1 Fields

Identical field set on both objects (so the merge is trivially per-field). On a `StandardNode` these
are plain pydantic fields, not schema attributes:

| Field | Type | Notes |
|---|---|---|
| `date_format` | `str \| None` | A **semantic format key**, not a rendering pattern — one of `ISO_8601`, `ISO_DATETIME`, `ISO_DATETIME_SECONDS`, `EU_DATETIME`, `US_12H` (see "Date format: semantic keys" below). Each client maps the key to its own renderer (web → date-fns, backend → strftime), so the stored value is not coupled to one frontend library. Typed as the `DateFormat` GraphQL enum on write, so an unknown key is rejected at the API layer — the value is *not* free-form. |
| `timezone` | `str \| None` | IANA timezone name (`Europe/Paris`, `UTC`). Selected in the UI from `Intl.supportedValuesOf('timeZone')`. Unset = browser-resolved zone. |

`UserPreference` additionally carries `account_id: str` (the owning account). Both fields above are
optional/nullable on both objects. Other candidates (dark mode, language, density) are deferred —
see "Future preferences" below.

### Date format: semantic keys

`date_format` stores a **semantic key**, never a library-specific rendering pattern. The value is
decoupled from any one frontend library, so every client maps the key to its own renderer: the web
app to a date-fns pattern, the backend to `strftime` (`render_datetime` in
`core/preferences/formats.py`), a future SDK to whatever it uses. A `DateFormat` GraphQL enum built
from the same canonical key list validates writes for free (unknown keys rejected before any write)
and keeps the enum and the render map from drifting.

| Key | date-fns (web) | strftime (backend) | Example |
|---|---|---|---|
| `ISO_8601` | `yyyy-MM-dd'T'HH:mm:ssXXX` | `%Y-%m-%dT%H:%M:%S%z` | `2026-07-01T14:30:00+02:00` |
| `ISO_DATETIME` *(default)* | `yyyy-MM-dd HH:mm` | `%Y-%m-%d %H:%M` | `2026-07-01 14:30` |
| `ISO_DATETIME_SECONDS` | `yyyy-MM-dd HH:mm:ss` | `%Y-%m-%d %H:%M:%S` | `2026-07-01 14:30:00` |
| `EU_DATETIME` | `dd/MM/yyyy HH:mm` | `%d/%m/%Y %H:%M` | `01/07/2026 14:30` |
| `US_12H` | `MM/dd/yyyy hh:mm a` | `%m/%d/%Y %I:%M %p` | `07/01/2026 02:30 PM` |

Every preset includes date **and** time. The set is deliberately limited to formats that render
identically on every client with no locale library and no ambiguity. Locale-dependent forms
(a localized long/month-name format) and a relative-time mode ("2 days ago") were considered and
**dropped**: the former needs a locale library server-side and renders differently per client
(defeating a portable key), and relative time is a display *mode* — inherently relative to "now" and
ambiguous server-side — not a format, so it doesn't belong in this enum. Both can return later
(relative time as a separate display toggle) without reshaping the stored value.

## Backend

### Models (`StandardNode`)

New `backend/infrahub/core/preferences/models.py` (or similar), following `Branch`
(`core/branch/models.py`) and `Root` (`core/root.py`):

```python
class GlobalPreference(StandardNode):
    date_format: str | None = None
    timezone: str | None = None

class UserPreference(StandardNode):
    account_id: str
    date_format: str | None = None
    timezone: str | None = None
```

- Persisted via the dedicated `StandardNode` Cypher queries (`StandardNodeCreate/Update/GetItem/GetList`, `core/query/standard_node.py`) — `create()` / `update()` / `get_list()` on the base class.
- `StandardNode`s are **global, not branch-scoped** (attached to `Root` via `IS_PART_OF`); no `BranchSupportType` handling needed.
- **No schema definition, no `definitions/core/preference.py`, no entry in `core_models`, no generic CRUD.**
- **No graph migration.** New `StandardNode` types persist on first write. `GlobalPreference` is the one singleton: fetched via a `get_global()` helper that **lazily creates** the empty instance if absent (so existing installs need no migration; new installs may also seed it in `first_time_initialization()` alongside `Root`, decision flagged at implementation — lazy-create is sufficient).
- `UserPreference` lookup is **by `account_id`**. `StandardNode` has no get-by-field query, so add a small custom Cypher query (a `StandardNodeQuery` subclass) to fetch the one row for an account id — do **not** `get_list()`-all-and-filter-in-Python (that would scan every user's row).

### GraphQL surface (custom, `Branch`-style)

Modelled on `graphql/types/branch.py`, `graphql/queries/branch.py`, `graphql/mutations/branch.py`,
wired into the root query/mutation in `graphql/schema.py` (the custom path, **not** the
auto-generated `InfrahubMutation` schema path).

The whole surface is organised around one axis: **scope** (`EFFECTIVE` | `GLOBAL` | `USER`) × keys.
One read query and one write mutation, both scope-parameterized.

**Read** — `InfrahubPreferences(scope: PreferenceScope = EFFECTIVE)`:

```graphql
query InfrahubPreferences($scope: PreferenceScope = EFFECTIVE) {
  InfrahubPreferences(scope: $scope) {
    preferences {                # one entry per preference key
      key                        # "date_format" | "timezone"
      value                      # resolved value, or null when nothing is defined for that scope
      source                     # USER | GLOBAL | DEFAULT — where `value` came from
    }
    can_edit_global_preferences  # boolean — drives the "Organisation defaults" surface (see Permissions)
  }
}
```

- **`EFFECTIVE`** (default) — the caller's resolved view: per key `value` = user value if set, else global, else `null`; `source` = `USER`/`GLOBAL`/`DEFAULT`. Any authenticated account; the global singleton is read *internally* to resolve, but the caller only ever gets their own resolved values. `source: DEFAULT` (value `null`) ⇒ the client applies the **browser** value (the only client-side step, since only the browser knows its locale/zone).
- **`USER`** — the caller's own raw override values (`source: USER`), bound to `account_session.account_id`; never another account.
- **`GLOBAL`** — the org-wide raw values (`source: GLOBAL`); **requires `manage_global_preferences`** (used by the Organisation-defaults editor, which needs the raw global — an admin who also has a personal override would see `source: USER` under `EFFECTIVE`).

The backend **computes the value and attaches an explicit source**, so the frontend never compares
"user vs global." `preferences` is a list (extensible to future keys) that the hooks collapse into a
keyed map (`prefs.date_format.{value,source}`).

**Write** — one mutation, scope-parameterized:

```graphql
mutation { InfrahubSetPreferences(scope: USER, date_format: "…", timezone: "…") { ok date_format timezone } }
```

- **`scope: USER`** — **always the caller's own row** (`account_session.account_id`); no account/target argument exists, so there is no path to write another user's row. Lazy-creates on first write; an explicit `null` for a field resets it (the "Automatic" selection).
- **`scope: GLOBAL`** — gated on `manage_global_preferences`; updates the singleton.
- **`scope: EFFECTIVE`** — rejected: the resolved view is read-only.

No generic `…Upsert/Update/Delete`, no SDK-introspectable kind.

### Permissions

Enforced imperatively at the single read + single write entry points, keyed on `scope`, fail-closed
(unauthenticated/anonymous rejected first):

| Operation | Allowed for | Mechanism |
|---|---|---|
| Read `scope: EFFECTIVE` | Any authenticated account (own resolved view) | Resolver binds to `account_session.account_id`; global read internally, no gate |
| Read `scope: USER` | The owning account only | Bound to `account_session.account_id`; no account argument |
| Read `scope: GLOBAL` | Holders of `manage_global_preferences` (super admins implicitly) | `active_permissions.raise_for_permission(...)` before any raw value is read |
| Write `scope: USER` | The owning account only | Bound to caller's row; no account argument |
| Write `scope: GLOBAL` | Holders of `manage_global_preferences` | `raise_for_permission(...)` before the read-modify-write |
| Write `scope: EFFECTIVE` | — | Rejected (read-only view) |

- **User preferences are private by construction.** No generic query, no account argument on reads/writes at `USER` scope — the resolver only ever touches `account_session.account_id`, so user A cannot read or write user B's preferences.
- **The global scope (read *and* write)** is gated by the new `GlobalPermissions.MANAGE_GLOBAL_PREFERENCES`, checked **imperatively** via `active_permissions.raise_for_permission(...)` (the `Branch`/global-permission idiom, `permissions/manager.py`) — **not** through `get_global_permission_for_kind()` / the object-permission pipeline (that is schema-`Node`-specific and does not apply to a `StandardNode`). Assignable to any role via the existing permissions UI; `super_admin` bypasses. (The global *values* aren't secret — every user's `EFFECTIVE` view already reflects them — so gating the raw `GLOBAL` read is a "only managers touch the global scope directly" principle, not secrecy.)
- **Frontend gating signal.** `StandardNode`s have no object permission, so the frontend can't use `useGetObjectPermissions`. Every read scope returns `can_edit_global_preferences` (from `active_permissions`) which hides/shows the Organisation-defaults surface. The backend is the source of truth regardless — the `GLOBAL`-scope read and write both enforce the permission.

## Frontend

### Data layer

- `useEffectivePreferences()` reads `InfrahubPreferences` (default `EFFECTIVE` scope) and exposes a keyed, already-resolved map: `prefs.date_format` / `prefs.timezone` as `{ value, source }` (source `user`/`global`/`default`) + `canEditGlobalPreferences`. Consumers read `value` + `source` directly — no comparing user-vs-global. A `source: "default"` value is `null`, so the consumer applies the browser value.
- `useGlobalPreferences()` reads `InfrahubPreferences(scope: GLOBAL)` (raw org values) and is used *only* by the Organisation-defaults editor — so it edits the raw global, correct even for an admin who also has a personal override. Gated server-side by `manage_global_preferences`.
- Writes go through `InfrahubSetPreferences(scope, …)`: the user card writes `scope: USER` (Automatic = explicit-null reset), the org card writes `scope: GLOBAL`. Success invalidates the effective query (and the global-scope query for org writes).
- There is **no** generic read of another user's preferences and no generic CRUD mutation; reads go through `InfrahubPreferences(scope)` and writes through the single `InfrahubSetPreferences(scope, …)`.
- All write hooks invalidate `useEffectivePreferences()` on success.
- No `localStorage` dual-write.

### Preferences surfaces (account settings)

User preferences live **inside the Profile tab**, in a "Preferences" card rendered **below the
profile details** (not a separate tab). Organisation/global preferences stay in their own gated tab:

- **Personal preferences** — a "Preferences" card on the Profile tab (`/profile`), below the account details. Card title: "Preferences". Each field is pre-filled from the caller's own override when they have one, otherwise it shows the **"Automatic"** option (see below). The `UserPreference` row is created lazily on first save.
- **Organisation defaults** tab (`/profile/organisation-defaults`) — edits the raw global values on `GlobalPreference` (never the merged values, so an admin who also has a personal override still edits the org default correctly). Visible only when the caller may manage global preferences (see Permissions). Card title: "Global date and time". No "Automatic" option here — the org card sets the defaults themselves.
- **"Automatic" option (= no override / inherit).** Each dropdown has an **Automatic** entry at the top. When the user has no override the field shows "Automatic"; selecting it clears the override (explicit-null write). It replaces a separate "reset to global" button — selecting Automatic *is* the reset. What "Automatic" resolves to is explained by the source indicator, not the option label.
- **Source indicator.** Instead of a sentence under the input, an **(i) info icon to the right** of each field carries a tooltip explaining where the current effective value comes from: **your preference** / **the organisation default** / **your browser** (with the resolved value). Keyboard-accessible, AA contrast.
- **Layout.** Object-details-style rows (a shared `DetailRow`: icon + label / control) with full-bleed separators between the rows and before the action buttons. Both dropdowns use the same shared `ComboboxField` at the **same fixed width**; each date-format option is labelled by its format (the pattern text, or a name where the pattern is unfriendly, e.g. "ISO 8601"), with a live example of the selected format shown beside the (width-capped) input.
- Form inputs (presets only — no free-text patterns in the UI):
  - `date_format`: the five semantic-key presets (see "Date format: semantic keys") plus the Automatic entry. The option's stored value is the key; its label is the human-facing format.
  - `timezone`: a searchable list over `Intl.supportedValuesOf('timeZone')` (with `UTC` ensured present — V8/Chrome omits it) plus the Automatic entry.

### Date rendering — DateDisplay as the consolidation vehicle

`DateDisplay` (`frontend/app/src/shared/components/display/date-display.tsx`) is where the preferences land. An internal hook (`useDateFormat`) feeds it:

- Reads `useEffectivePreferences()`.
- Resolves the effective `date_format` **key** to its date-fns pattern via the frontend key→pattern map (`entities/preferences/domain/date-format-presets.ts`), then formats.
- When both global and user are unset (`source: "default"`), fall back to the **browser**: `date_format` → the browser locale's date/time formatting (e.g. `toLocaleString` / `Intl.DateTimeFormat` locale defaults, not a fixed key); `timezone` → `Intl.DateTimeFormat().resolvedOptions().timeZone`. (Server-side rendering has no browser, so `render_datetime` falls back to the `ISO_DATETIME` key instead — see below.)
- date-fns is v4 — use the first-party `@date-fns/tz` package (not the legacy `date-fns-tz`) for timezone-aware formatting.
- The timezone preference applies to absolute renderings and tooltips (`shared/utils/date.ts`); relative-time text ("2 days ago") is timezone-independent and unchanged.
- Known non-`DateDisplay` display call sites to migrate to `DateDisplay` (preferred) or the hook:
  - `entities/events/ui/global-event.tsx` (two direct `format()` calls)
  - `entities/navigation/ui/time-selector.tsx`
  - `shared/components/display/duration-display.tsx`
  - `entities/navigation/ui/search-anywhere/search-nodes.tsx`
- Form inputs (`datetime.field.tsx`, `date-picker.tsx`) do submission/validation, not display — out of scope.

## Future Preferences (out of V1, listed for context)

These will be added incrementally once the V1 plumbing is proven. Each requires its own design pass:

- **Dark mode / theme** — needs a decision on system-vs-stored preference, transition strategy, and whether the global default is meaningful.
- **Language** — depends on the i18n strategy.
- **Density** (compact / comfortable list rows).
- **Default landing page after login**.

These are listed here as a backlog hint, not committed scope.

## Out of Scope (separate specs/tickets)

- Saved views — `dev/specs/2026-04-saved-views.md`
- Saved filters (formerly `last_used_filters`) — `dev/specs/2026-04-saved-filters.md`
- "Show extra fields" toggle persistence — `dev/specs/2026-04-show-extra-fields.md`
- `branch_delete_mode` — explicitly dropped. Persisting a destructive default is too dangerous; the dialog will keep prompting per action.
- Cross-account preference import/export.
- Schema graph visualisation state (fold/zoom/positions) — stays in `localStorage` for now.

## Resolved Decisions

- **Model** — `StandardNode` objects (`GlobalPreference`, `UserPreference`), not schema nodes. Reads/writes are custom GraphQL only; this is what makes user preferences private per-account (no generic query can leak them).
- **Profiles** — not used (schema-`Node` feature; irrelevant to `StandardNode`; merge is trivial in a resolver).
- **Singleton enforcement for `GlobalPreference`** — single instance, fetched via `get_global()` with lazy create-if-missing; no graph migration required.
- **Effective query shape** — scalar fields, plus a `can_edit_global_preferences` boolean for tab gating.
- **Default when nothing is stored** — on the web, the **browser's own values** (browser locale date/time formatting + browser-resolved timezone). Server-side (no browser) the backend's `render_datetime` falls back to the `ISO_DATETIME` key. The `ISO_DATETIME` format (`yyyy-MM-dd HH:mm`) is otherwise just one selectable preset.
- **Surface location** — user preferences render in a "Preferences" card on the Profile tab, below the account details (not a separate tab); global/organisation preferences stay in their own gated tab.
- **Format input style** — the five semantic-key presets only; the `DateFormat` GraphQL enum validates writes, so unknown keys are rejected even via the SDK/API (the value is no longer free-form). Adding a format is one enum/key-map entry on each renderer, no schema migration.
- **"Automatic" option** — each dropdown offers an Automatic (= inherit / no override) entry; selecting it clears the override, replacing a separate reset button. Shown as the selected option when the user has no override.
- **Source indicator** — an (i) tooltip to the right of each field states whether the effective value comes from the user, the organisation default, or the browser (no sentence under the input).
- **`UserPreference` creation** — lazy create on first save, no row at account creation.
- **Admin gating for `GlobalPreference`** — new `manage_global_preferences` global permission, checked imperatively in the mutation resolver (not via the object-permission kind mapping, which does not apply to a `StandardNode`).

## Migration & Rollout

- Purely additive. New `StandardNode` types, new custom GraphQL query + mutations, new frontend hooks + tabs.
- **No graph migration and no schema change** — `StandardNode`s persist on first write; the `GlobalPreference` singleton is lazily created on first access.
- Existing date-rendering code keeps working until each call site migrates to `DateDisplay`. Migration can ship incrementally per call site.
