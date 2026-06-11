---
Title: User & Global Preferences
Author:
  - Paul Lemesle
Status: draft
---

# User & Global Preferences

## Summary

Introduce two persistent, schema-backed preference surfaces:

- `CoreGlobalPreference` — admin-defined defaults that apply to every user (one singleton node).
- `CoreUserPreference` — per-user overrides (one node per account).

A single backend-computed query returns the **effective** preferences for the calling user (global merged with their personal overrides), so the frontend never has to merge them itself. Standard auto-generated CRUD mutations are used everywhere; no custom write path.

V1 ships only two attributes — `date_format` and `timezone` — to validate the schema, query, mutation, and UI plumbing end to end. Additional preferences (dark mode, etc.) land in follow-up tickets once the foundation is proven.

Three adjacent concerns get their own spec/ticket and are explicitly **not** part of V1:

- Saved views — `dev/specs/2026-04-saved-views.md`
- Saved filters — `dev/specs/2026-04-saved-filters.md`
- "Show extra fields" toggle persistence — `dev/specs/2026-04-show-extra-fields.md`

## Problem Statement

- UI defaults (date format, timezone) live in component state or the browser's locale. They cannot be set centrally by an organization, and a user's choice does not survive a device switch.
- There is no place for an admin to express "use ISO dates organisation-wide" or "default everyone to UTC".
- There is no schema-backed concept of "preferences" that the SDK or other clients can read.

## Solution Overview

Two schema nodes, one read query that fuses them.

| Node | Cardinality | Who writes | Who reads |
|---|---|---|---|
| `CoreGlobalPreference` | Singleton (0..1) | Admin only | Authenticated users (read effective query) |
| `CoreUserPreference` | One per `CoreGenericAccount` | Owning user (and admin) | Owner (and admin) |

Effective resolution per attribute: **user value if set, else global value, else built-in default**. Defaults live in the frontend (so the SDK sees a `null` for "no opinion stored" and can apply its own).

## Success Criteria — V1

- An admin can set `date_format` and `timezone` once for the organisation; an unauthenticated browser-issued read returns those values for any user without a personal override.
- A user can override either attribute on their own preferences page; the override takes effect on next page load and is visible from any device.
- Removing a user override falls back to the global value; removing the global value falls back to the frontend default.
- Standard CRUD mutations work for both nodes (admin tooling, SDK use, Postman).
- A single GraphQL query returns the effective value to render with — no client-side merging.

## V1 Attributes

Identical attribute set on both nodes (so the merge is trivially per-attribute):

| Attribute | Kind | Notes |
|---|---|---|
| `date_format` | Text | date-fns pattern string, chosen in the UI from a curated preset list (e.g. `dd/MM/yyyy`, `yyyy-MM-dd HH:mm`). Literal `relative` is a preset for relative-time rendering. Presets are a UI constraint, not a schema one — backend stores the pattern verbatim, so the SDK can write any pattern. |
| `timezone` | Text | IANA timezone name (`Europe/Paris`, `UTC`). Selected in the UI from `Intl.supportedValuesOf('timeZone')`. Unset = browser-resolved zone. |

All attributes optional on both nodes. Other candidates (dark mode, language, density) are deferred — see "Future preferences" below.

## Backend

### `CoreGlobalPreference`

- Defined in `backend/infrahub/core/schema/definitions/core/preference.py` (new file), registered in `definitions/core/__init__.py`.
- `name="GlobalPreference"`, `namespace="Core"`. Note: `Core` (not `Internal` like `AccountToken`) is deliberate — these nodes are user-facing and SDK-visible.
- `branch=BranchSupportType.AGNOSTIC`.
- Singleton: an empty row is seeded in `first_time_initialization()` (`core/initialization.py`) for new installs, plus a graph migration in `core/migrations/graph/` for existing installs. App code treats it as 0..1 and refuses to create a second row.
- No relationships in V1.

```yaml
- name: GlobalPreference
  namespace: Core
  label: Global Preference
  description: Organisation-wide defaults applied to every user unless overridden.
  branch: agnostic
  include_in_menu: false
  generate_profile: false
  display_label: "Global Preferences"
  icon: mdi:cog
  attributes:
    - name: date_format
      kind: Text
      optional: true
      order_weight: 1000
    - name: timezone
      kind: Text
      optional: true
      order_weight: 1100
```

### `CoreUserPreference`

- Same file as above. Same V1 attribute set as `CoreGlobalPreference`.
- Relationship `account` → `CoreGenericAccount`, cardinality ONE, required, identifier `account__preferences`, `on_delete: cascade`.
- Uniqueness constraint on `account` so an account has at most one preference node.

```yaml
- name: UserPreference
  namespace: Core
  label: User Preference
  description: Per-user overrides of global preferences.
  branch: agnostic
  include_in_menu: false
  generate_profile: false
  display_label: "Preferences of {{ account__name__value }}"
  icon: mdi:account-cog-outline
  uniqueness_constraints:
    - ["account"]
  attributes:
    - name: date_format
      kind: Text
      optional: true
      order_weight: 1000
    - name: timezone
      kind: Text
      optional: true
      order_weight: 1100
  relationships:
    - name: account
      peer: CoreGenericAccount
      identifier: account__preferences
      kind: Parent
      cardinality: one
      optional: false
      on_delete: cascade
      order_weight: 100
```

### GraphQL operations

**Standard auto-generated mutations** are the primary write path:

- `CoreGlobalPreferenceUpsert / Update / Delete` — admin only via permissions.
- `CoreUserPreferenceUpsert / Update / Delete` — node-level permission scoped to owner-or-admin.

No custom write mutation — keeps the surface predictable and aligned with the rest of the schema.

**Standard auto-generated queries** are also exposed (`CoreGlobalPreference`, `CoreUserPreference`) for admin tooling and SDK introspection.

**One custom read query** for the rendering path:

```graphql
query InfrahubEffectivePreferences {
  InfrahubEffectivePreferences {
    date_format   # value or null
    timezone      # value or null
    # scalar fields, not Attribute wrappers — this is a computed view, not a node
  }
}
```

Scalar fields (no `Attribute { value }` wrapper) is the decided shape: this is a computed view, not a node, and the lean shape is what the rendering path wants.

Implementation in `backend/infrahub/graphql/queries/preferences.py` (resolver + Graphene `Field`, exported in `queries/__init__.py`, attached to `InfrahubBaseQuery` in `graphql/schema.py`):

1. Resolve account from JWT via `graphql_context.account_session.account_id` (same pattern as `resolve_account_tokens` in `graphql/queries/account.py`).
2. Read the singleton `CoreGlobalPreference` (cache-friendly, branch-agnostic).
3. Read the caller's `CoreUserPreference` if any.
4. Per attribute: return user value if set, else global value, else `null`.

The frontend interprets `null` as "use built-in default". The SDK can do the same.

### Permissions

| Operation | Allowed for |
|---|---|
| Read `InfrahubEffectivePreferences` | Any authenticated account (returns their own effective view) |
| Read `CoreGlobalPreference` | Any authenticated account |
| Write `CoreGlobalPreference` | Admins (via `ObjectPermission` on `Core` / `GlobalPreference`) |
| Read/write `CoreUserPreference` | Owner; admin can also read/write any user's prefs |

Two distinct mechanisms, matching how the codebase actually works:

- **`CoreGlobalPreference`** — standard node-level `ObjectPermission` model. Admin roles get update/delete on `Core/GlobalPreference`; everyone keeps read.
- **`CoreUserPreference`** — owner-scoping follows the `AccountToken` mechanism, which is *not* the node-level permission system: the query resolver filters on `account__ids = [calling account]` (see `graphql/queries/account.py`) and mutations re-check ownership before writing (see `AccountMixin` in `graphql/mutations/account.py`). Admins bypass the ownership check.

## Frontend

### Data layer

- TanStack Query hook `useEffectivePreferences()` in `frontend/app/src/entities/preferences/` exposes `{ date_format, timezone }` (already merged).
- Hooks for admin paths: `useGlobalPreferences()` / `useUpdateGlobalPreferences()`.
- Hooks for the user override path: `useMyUserPreferences()` / `useUpdateMyUserPreferences()`.
- All write hooks invalidate `useEffectivePreferences()` on success.
- No `localStorage` dual-write.

### Preferences tabs (account settings)

Decided: preferences live as new tabs in the existing account settings page (`/profile`, tabs declared in `entities/user-profile/ui/profile-tabs.tsx` — currently Profile / Tokens / Password):

- **Preferences** tab (`/profile/preferences`) — always visible. Editable form for the user's own `date_format`, `timezone`. Each field shows the inherited global value as its placeholder/hint when the user has no override; a "reset to global" button clears the override. The `CoreUserPreference` row is created lazily on first save (upsert), not at account creation.
- **Organisation defaults** tab (`/profile/organisation-defaults`, naming TBD at implementation) — same fields on `CoreGlobalPreference`. Visible only when the user has update permission on `CoreGlobalPreference`, checked via `useGetObjectPermissions` (the frontend has no super-admin flag; object permissions are the only gating mechanism).
- Form inputs (presets only — no free-text patterns in the UI):
  - `date_format`: select from a curated preset list, including `relative` for relative-time rendering.
  - `timezone`: searchable select over `Intl.supportedValuesOf('timeZone')`.

### Date rendering — DateDisplay as the consolidation vehicle

`DateDisplay` (`frontend/app/src/shared/components/display/date-display.tsx`) is where the preferences land. An internal hook (`useDateFormat`) feeds it:

- Reads `useEffectivePreferences()`.
- Default `date_format` if both global and user are unset: `yyyy-MM-dd HH:mm` (decided).
- Default `timezone` if unset: `Intl.DateTimeFormat().resolvedOptions().timeZone`.
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

Formerly open questions, now decided:

- **Singleton enforcement for `CoreGlobalPreference`** — seeded empty row in `first_time_initialization()` for new installs + graph migration for existing installs; app refuses a second row.
- **Effective query shape** — scalar fields. It is a computed view, not a node; the lean shape wins.
- **Default `date_format` when nothing is stored** — `yyyy-MM-dd HH:mm`, applied in the frontend.
- **Settings page location** — tabs in the existing account settings page (`/profile`), not a top-level route.
- **Format input style** — curated presets only in the UI (incl. `relative`); free-text patterns remain possible via SDK/API since the backend stores verbatim.
- **`CoreUserPreference` creation** — lazy upsert on first save, no row at account creation.

## Migration & Rollout

- Purely additive. New schema nodes, new GraphQL query, new frontend hooks + tabs.
- Existing date-rendering code keeps working until each call site migrates to `DateDisplay`. Migration can ship incrementally per call site.
- One small graph migration seeds the empty `CoreGlobalPreference` row on existing installs.
