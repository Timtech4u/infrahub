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
| `date_format` | `str \| None` | date-fns pattern string, chosen in the UI from a curated preset list (e.g. `dd/MM/yyyy`, `yyyy-MM-dd HH:mm`). Literal `relative` is a preset for relative-time rendering. Presets are a UI constraint, not a storage one — the backend stores the string verbatim, so the SDK can write any pattern. |
| `timezone` | `str \| None` | IANA timezone name (`Europe/Paris`, `UTC`). Selected in the UI from `Intl.supportedValuesOf('timeZone')`. Unset = browser-resolved zone. |

`UserPreference` additionally carries `account_id: str` (the owning account). Both fields above are
optional/nullable on both objects. Other candidates (dark mode, language, density) are deferred —
see "Future preferences" below.

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

**Effective read query** — the rendering path:

```graphql
query InfrahubEffectivePreferences {
  InfrahubEffectivePreferences {
    date_format                  # merged — user > global > null (the value to render with)
    timezone                     # merged — user > global > null
    user_date_format             # the caller's OWN override, or null
    user_timezone                # the caller's OWN override, or null
    global_date_format           # the org default, or null
    global_timezone              # the org default, or null
    can_edit_global_preferences  # boolean — drives the "Organisation defaults" tab (see Permissions)
  }
}
```

Scalar fields (no `Attribute { value }` wrapper) — this is a computed view, not a node. The query
returns three views of each field so one round trip serves every surface: the **merged** value for
rendering, the caller's **own** override (so the user form shows what they set vs. inherited, and
"reset to global" maps to clearing it), and the raw **global** value (so the Organisation-defaults
editor edits the actual org default — not the merged value — and the user form shows it as the
inherited hint). Privacy holds: `global_*` is org-wide and fine to expose to any authenticated
account, while `user_*` is the caller's own only (the query is account-bound). Resolver:

1. Resolve the caller via `graphql_context.account_session.account_id`; reject unauthenticated/anonymous sessions (`PermissionDeniedError`).
2. Read the singleton `GlobalPreference` (`get_global()`).
3. Read the caller's `UserPreference` by `account_id` (or none).
4. Per field: emit `user_<field>` (own override), `global_<field>` (org default), and the merged `<field>` = user value if set, else global, else `null`.
5. Compute `can_edit_global_preferences` from `graphql_context.active_permissions` (see Permissions).

**Mutations** (custom, `Branch`-style classes registered in `graphql/schema.py`):

- `InfrahubUserPreferenceUpsert(date_format, timezone)` — **always operates on the caller's own row**, identified by `account_session.account_id`. The mutation never accepts an account/target argument, so there is no path to write another user's row. Lazy-creates the row on first write. A reset/clear sets fields back to `null`.
- `InfrahubGlobalPreferenceUpdate(date_format, timezone)` — gated on `manage_global_preferences` (see Permissions), updates the singleton.

No generic `…Upsert/Update/Delete`, no SDK-introspectable kind.

### Permissions

| Operation | Allowed for | Mechanism |
|---|---|---|
| Read effective preferences | Any authenticated account (own view only) | Resolver binds to `account_session.account_id` |
| Read/write own `UserPreference` | The owning account only | Structural — no generic query; mutation targets the caller's own row only |
| Update `GlobalPreference` | Holders of `manage_global_preferences` (super admins implicitly) | Imperative check in the mutation resolver |

- **User preferences are private by construction.** There is no generic query and the custom query/mutation only ever touch the caller's own row, so user A cannot read or write user B's preferences. No object permissions, no owner-re-check machinery.
- **Global write** is gated by a new `GlobalPermissions.MANAGE_GLOBAL_PREFERENCES` enum value, checked **imperatively** in the `InfrahubGlobalPreferenceUpdate` resolver via `graphql_context.active_permissions.raise_for_permission(...)` (the `Branch`/global-permission idiom — `permissions/manager.py`). It is **not** wired through `get_global_permission_for_kind()` / the object-permission pipeline (that is schema-`Node`-specific and does not apply to a `StandardNode`). Because global permissions are regular `CoreGlobalPermission` nodes, this permission is assignable to any role through the existing permissions UI; `super_admin` bypasses it.
- **Frontend gating signal.** Since there is no object permission on a `StandardNode`, the frontend cannot use `useGetObjectPermissions`. Instead the effective query returns a `can_edit_global_preferences` boolean (computed from `active_permissions`) that drives the "Organisation defaults" tab's visibility. The backend remains the source of truth — `InfrahubGlobalPreferenceUpdate` enforces the permission regardless of the flag.

## Frontend

### Data layer

- A single TanStack Query hook `useEffectivePreferences()` in `frontend/app/src/entities/preferences/` reads the one `InfrahubEffectivePreferences` query and exposes everything every surface needs: the merged `{ date_format, timezone }` (rendering), the caller's own `{ user_date_format, user_timezone }` (user form state + reset detection), the org `{ global_date_format, global_timezone }` (inherited hint + org-defaults editor), and `can_edit_global_preferences` (tab gating). No separate per-row read hooks.
- `useUpdateMyUserPreferences()` → calls `InfrahubUserPreferenceUpsert` (caller's own row; no account argument). "Reset to global" sends explicit `null` for the field(s) (there is no delete mutation).
- `useUpdateGlobalPreferences()` → calls `InfrahubGlobalPreferenceUpdate`.
- There is **no** generic read of another user's preferences and no generic CRUD mutation; reads go through the single effective query and writes through the two custom mutations above.
- All write hooks invalidate `useEffectivePreferences()` on success.
- No `localStorage` dual-write.

### Preferences surfaces (account settings)

User preferences live **inside the Profile tab**, in a "Preferences" card rendered **below the
profile details** (not a separate tab). Organisation/global preferences stay in their own gated tab:

- **Personal preferences** — a "Preferences" card on the Profile tab (`/profile`), below the account details. Editable form pre-filled from `user_*` (the caller's own override); each field shows `global_*` as its placeholder/hint when the user has no override, falling back to the **browser default** (browser-formatted example + resolved timezone) when global is also unset; a "reset to global" button clears the override (explicit-null upsert). The `UserPreference` row is created lazily on first save. Card title: "Preferences".
- **Organisation defaults** tab (`/profile/organisation-defaults`) — edits the raw `global_*` values on `GlobalPreference` (not the merged values, so an admin who also has a personal override still edits the org default correctly). Visible only when `can_edit_global_preferences` (from the effective query) is true. (Not `useGetObjectPermissions` — there is no object permission on a `StandardNode`.) Card title: "Global date and time".
- The card uses object-details-style rows (a shared `DetailRow`: icon + label / control) with full-bleed separators between rows and before the action buttons. Both dropdowns use the same shared `ComboboxField`; date-format options are pattern-only with a live example shown beside the input.
- Form inputs (presets only — no free-text patterns in the UI):
  - `date_format`: select from a curated preset list, including `relative` for relative-time rendering.
  - `timezone`: searchable select over `Intl.supportedValuesOf('timeZone')`.

### Date rendering — DateDisplay as the consolidation vehicle

`DateDisplay` (`frontend/app/src/shared/components/display/date-display.tsx`) is where the preferences land. An internal hook (`useDateFormat`) feeds it:

- Reads `useEffectivePreferences()`.
- When both global and user are unset, fall back to the **browser**: `date_format` → the browser locale's date/time formatting (e.g. `toLocaleString` / `Intl.DateTimeFormat` locale defaults, not a fixed pattern); `timezone` → `Intl.DateTimeFormat().resolvedOptions().timeZone`.
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
- **Default when nothing is stored** — the **browser's own values** (browser locale date/time formatting + browser-resolved timezone), not a fixed pattern. `yyyy-MM-dd HH:mm` remains only as one selectable preset.
- **Surface location** — user preferences render in a "Preferences" card on the Profile tab, below the account details (not a separate tab); global/organisation preferences stay in their own gated tab.
- **Format input style** — curated presets only in the UI (incl. `relative`); free-text patterns remain possible via the SDK/API since the backend stores verbatim.
- **`UserPreference` creation** — lazy create on first save, no row at account creation.
- **Admin gating for `GlobalPreference`** — new `manage_global_preferences` global permission, checked imperatively in the mutation resolver (not via the object-permission kind mapping, which does not apply to a `StandardNode`).

## Migration & Rollout

- Purely additive. New `StandardNode` types, new custom GraphQL query + mutations, new frontend hooks + tabs.
- **No graph migration and no schema change** — `StandardNode`s persist on first write; the `GlobalPreference` singleton is lazily created on first access.
- Existing date-rendering code keeps working until each call site migrates to `DateDisplay`. Migration can ship incrementally per call site.
