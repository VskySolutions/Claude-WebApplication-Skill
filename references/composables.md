# Composables, utilities and services that drive the UI

What each shared function does, what it returns, and when a page reaches for it. Source:
`src/composables/**`, `src/utils/*`, `src/services/*`, `src/stores/*`.

## Contents
1. Feedback: `useNotify`, `useConfirm`
2. Persistence: `usePreferences`, `useFilterMemory`, `useDrawerResize`
3. Lists: `useListTable`, `useColumnFilters`, `useAuditColumns`, `useColumnOrder`, `useColumnResize`
4. Dates and formatting: `useDateFormat`, `formatDateOnly`, legacy `useFilters`
5. Fields: `useFieldLabel`, `useCountries`, `useFileDrop`, `useFilePreview`, `useMentions`, `utils/personName`, `utils/richText`
6. Auto-save: `useAutoSave`
7. Access and tenancy: `usePermissions`, `useTenantOptions`, `useTenantScope`, `useRoleOptions`, `usePermissionCategories`, `useDeletedRecords`, `useShowDeleted`
8. Option lists: `useOptionSet`, the option catalogue pattern, label-map composables
9. Session: `useSessionExpiry`, `services/jwt`
10. Stores: `auth`, `tenant`, `branding`
11. Dashboard and universal-feature composables (pointers)

---

## 1. Feedback

**`useNotify()`** → `{ notifyPositive, notifyNegative, notifyWarning, notifyInfo, notifySuccess, notifyError,
success, error, warning, info }`. All create a bottom, multi-line, `html: true` Quasar notification with the
standard icon (`o_check_circle`, `o_error`, `o_warning`, `o_info`). Call with a string or
`{ message, caption, timeout, actions }`; a second `opts` object overrides. Pattern:
`notify.success("Person created.")`, `notify.error(getApiErrorMessage(err))`.

**`useConfirm()`** → `{ confirm }`; `await confirm({ title = "Please confirm", message, confirmLabel = "Confirm",
cancelLabel = "Cancel", type = "primary" | "danger" })` resolves `true`/`false` (dismiss counts as false).
Builds `Dialog.create` with ok `unelevated noCaps` in `primary` or `negative`, cancel `flat noCaps grey-8`,
`persistent: true`.

## 2. Persistence

**`usePreferences(pageKey)`** → `{ all, get(key, fallback), set(key, value), merge(partial), remove(key), clear }`
over one JSON cookie `pref:<pageKey>` (365 days, path `/`, `sameSite: Lax`). Used by the table (columns,
widths, order, page size), `useListTable` (sort), form drawers (`formDraft`), the dashboard (`dateRange`,
`seenWidgets`) and sticky notes (`hideAll`).

**`useFilterMemory(pageKey, bindings, defaults)`** — remembers quick-filter picks in LocalStorage
`filters:<pageKey>`. `bindings` is `{ name: ref }` (use `toRef(filters, "status")` for a column filter);
`defaults` is what a filter starts on for someone who never touched it; a filter the user cleared stays
cleared (stored as null). Restores synchronously during setup so the first load already carries the values.

**`useDrawerResize({ storageKey, getDefault, getMin, getMax })`** → `{ width, startResize, resetWidth }` for a
right-side drawer whose left edge is dragged (`mousedown` → document move/up, `user-select: none` while
dragging, width clamped and saved to LocalStorage). `viewportWidth()` helper for viewport-relative sizes.

## 3. Lists

**`useListTable({ fetcher, defaultPageSize = 20, defaultSortBy = "updatedOnUtc", defaultDescending = true,
pageKey, onError, reloadOnTenantSwitch = true })`** → `{ rows, loading, totalRecords, search, filterOpen, selected,
pagination, load, onRequest }`.
- `fetcher({ page, limit, sortBy, descending })` must resolve `{ data, total }`; the page maps the API
  envelope: `.then((r) => ({ data: r?.data, total: r?.meta?.totalRecords }))`. For load-all endpoints
  return `{ data: r || [], total: (r || []).length }`.
- Restores saved `pageSize`, `sortBy`, `descending` from the page cookie; `onRequest` persists them and
  resets to page 1 on a sort change; loads on mount and on `tenant-switched`.
- Pages add their own debounced reload: `const reload = debounce(() => { pagination.value.page = 1; load(); }, 300); watch([search, filters], reload, { deep: true });`

**`useColumnFilters(columns, rows, { server = true })`** → `{ filters, filterableColumns, filteredRows, filterChips,
removeFilter, clearFilters, rangeBounds }`.
- A column filters as text "contains" unless it has `filterOptions` (select; array when `filterMultiple`) or
  `filterType: "dateRange"` (`{ from, to }` ISO dates). `filterable: false` excludes it; `actions` is always excluded.
- Server mode (default) only holds state; client mode filters `rows` (equality, `filterMatch: "includes"`,
  or case-insensitive contains) and the page binds `:rows="filteredRows"`.
- `filterChips` yields one chip per filter and one per **end** of a date range (`key: "name:from"`), labelled
  `Column: value` with option labels resolved; `removeFilter(key)` handles both shapes.
- `rangeBounds(name, toUtc)` returns `{ from, to }` converted by `useDateFormat().zonedDayBoundaryUtc`.

**`useAuditColumns()`** → `auditColumns({ overrides, only, dateRanges })` returning the four standard columns
`createdBy`, `createdOnUtc` (default hidden), `updatedBy`, `updatedOnUtc` (default visible), formatted with
`formatDateTime`, sorting on the raw instant, `filterable: false` except the two dates as ranges when
`dateRanges: true`. `overrides` remaps field names (`{ createdBy: "createdByName" }`).

**`useColumnOrder({ columns, initialOrder, saveOrderState })`** → `{ order, reorder, move, resetOrder, orderColumns }`
(actions always last, unknown saved names dropped, new columns appended).
**`useColumnResize({ columns, resizeWidths, saveResizableWidthState })`** → `{ startResize, stopResize,
resetColumnsWidth, isResizing }` (default 120px, min 80px).

## 4. Dates and formatting

**`useDateFormat()`** → `{ formatDateTime, formatDate, formatTime, tenantTimeZone, zonedDayBoundaryUtc }`.
Backend values are UTC instants (a missing designator is treated as UTC) rendered in the **active tenant's
time zone** (`activeTenant.timeZoneId`, default `UTC`) with `en-US` parts: `MM/DD/YYYY hh:mm AM`,
`MM/DD/YYYY`, `hh:mm AM`; `—` placeholder. `zonedDayBoundaryUtc("2026-10-27", "start"|"end")` turns a
picked calendar day into the real UTC boundaries for date-range filters (DST-aware).

**`formatDateOnly(value, placeholder = "—")`** — a calendar `DateOnly` (`YYYY-MM-DD`) → `MM/DD/YYYY`,
string-reformatted, never through a `Date` (so it never shifts a day).

**`useFilters()`** (legacy) — `toCurrency`, `toNumeric`, `toPercentage`, `toPhone`, `toDate`, `truncate`…
Kept for old screens; new code formats with `Intl` directly or the helpers above.

## 5. Fields

**`useFieldLabel(label, required)`** → `{ text, isRequired }`: strips a trailing `*` and reports required when
the prop or the star says so. Every App field uses it for the visible label and the `aria-label`.

**`useCountries()`** / exports — `orderedCountries` (US, India, then the rest), `dialCodeOption`,
`countryOption`, `countryNameOption`, `countryNameFromIso`, `isoFromDial`, `dialFromIso`, `DEFAULT_COUNTRY_ISO = "US"`.
Flag emoji are deliberately omitted (they render as letters on Windows).

**`useFileDrop()`** → `{ dragOver, onDragOver, onDragLeave }` plus exports `MAX_UPLOAD_MB`, `ATTACHMENT_ACCEPT`,
`MAX_ATTACHMENT_FILES`, `MAX_ATTACHMENT_TOTAL_MB`, `ATTACHMENT_HINT`, `extOf`, `normalizeExt`,
`isImageExtension`, `isImageFile`, `iconForExtension`, `iconForFile`, `validateFiles(fileList, { accept,
maxSizeMb })` → `{ accepted, error }`, `formatFileSize(bytes)`.

**`useFilePreview`** (module exports) — `mediaIdOf`, `nameOf`, `extOfStored`, `iconForStored`, `isImageStored`,
`describeStored` ("PDF · 1.2 MB"), `fetchStoredBytes`, `openStoredFile(file, fetchBlob?)` (opens a tab
synchronously, then points it at a blob URL; closes it on failure), `downloadStoredFile`.

**`useMentions`** (module exports) — `MENTION_CLASS = "uf-mention"`, `MENTION_ID_ATTR = "data-user-id"`,
`mentionTokenHtml(user)`, `extractMentionIds(html)`, `fetchMentionCandidates(term)` (never throws).

**`utils/personName.js`** — `NAME_MAX_LENGTH = 100`, `NAME_SUFFIXES`, `NAME_SUFFIX_MAX_LENGTH = 16`,
`nameIssue(value, label)` (sentence or ""), `nameIsValid`, `nameRules(label, { required })` → Quasar rules
(letters from any script, single spaces, hyphen/apostrophe/period, no digits, no leading/trailing space).

**`utils/richText.js`** — `escapeHtml`, `sanitizeHtml` (drops script/style/iframe/object/embed/link/meta/
form/input/button/svg/math, `on*` attributes and `javascript:`/`data:` URLs; forces `_blank` + `noopener`
on links), `isHtml`, `renderRichText` (legacy plain text keeps its line breaks), `stripHtml` (for cells and
tooltips), `hasRichTextContent` (an emptied editor still posts `<p>&nbsp;</p>`).

## 6. Auto-save

**`useAutoSave(savers, { delay = 1200, enabled, onSaved })`** → `{ state, message, pending, mark(key), flush,
suspend, resume, reset }`. `savers` is `{ part: async () => string | void }` where a returned string means
"blocked, not an error" (e.g. a required selection still missing). States: `idle | pending | saving | saved |
blocked | error`. One write pass at a time; a part re-dirtied mid-pass is written next; a 429 reschedules
after `Retry-After`; `suspend()/resume()` wrap a reload so seeding forms does not trigger saves. Pair it with
a small header indicator (a 24px round dot: idle outline, busy teal, ok green, warn amber, bad red, with the
message on a tooltip).

The simpler per-card pattern used on detail pages is a local `runAutoSave(target, fn)` that sets
`{ state: "saving" } → "saved"` (fades to idle after 2.5s) or `"error"` + message, shown with `AppAutoSaveState`.

## 7. Access and tenancy

**`usePermissions()`** → `{ has, hasAny }`; `Permissions` is the frozen key catalogue. See `app-shell.md`.

**`useTenantOptions()`** → `{ canChooseTenant (tenants.write), activeTenantId, tenantOptions, loadingTenants,
loadTenants, tenantName }` for forms that may assign a tenant: platform admins get a dropdown, everyone else
is auto-scoped. Loads once, silently empty on failure.

**`useTenantScope()`** → `{ canScopeTenant (SuperAdmin role), selectedTenantId, isScoped, scopedTenantName,
tenantOptions, loadingTenants, loadTenants({ force }), refreshTenants, setScope, clearScope }`; module-level
shared state, LocalStorage `adminTenantOverride`, dispatches `tenant-switched` on change, forgets on
`session-cleared`. Pages that create/rename/archive tenants call `refreshTenants()`.

**`useRoleOptions()`** → `{ roles, roleOptions, loading, isSuperAdmin, loadForTenant(tenantId) }`; `roleOptions`
is a **grouped** list (System / Operational / REMS Seats / Custom) using `header: true` rows, SuperAdmin
excluded for non-super-admins. `roleCategoryChip(name)` → `{ category, color, textColor }` for coloured role chips
(`blue-grey`, `primary`, `deep-purple-6`, `teal`).

**`usePermissionCategories()`** → `{ CATEGORY_ORDER, ELEVATED_KEYS, categoryForKey, humanizeKey, groupKeysByCategory }`;
`humanizeKey("users.reset_password")` → `"Users · Reset Password"`.

**`useDeletedRecords()`** → `{ showDeleted, canManageDeleted (records.adminDelete) }` — the "Show deleted?"
toggle state for a list. **`useShowDeleted(reload)`** → `{ canShowDeleted, showDeleted, toggle, isDeleted(row), isOverdue(row) }`.

## 8. Option lists

**`useEntityTypeOptions()`** → `{ options }` (entity types as `{ label, value }`).
**`useOptionSetOptions(getParams)`** → `{ options, loading, error, reload }`; resolves the active, ordered values
for `{ entityType, key, parentItemId? }` (cascading lists) and keeps the raw item on `raw`.

**The option catalogue pattern** (`modules/rems/useRemsOptionCatalog.js`, generic in shape): one module-level
`reactive` map of named lists resolved once per session (`ensureXxxLoaded()`), re-resolved on
`tenant-switched`, each item normalised to `{ label, value, description, backgroundColor, textColor, icon }`.
A meta composable then exposes `xxxLabel(code)`, `xxxHint(code)` and `xxxOption(code)` (the whole option for
`AppOptionBadge`); an unknown code renders as itself in grey rather than as a blank. **Every word, colour and
icon a status is rendered with comes from the tenant's list, not from a map in the code.**

**Label-map composables** (`useSmtpOptions`): constant `[{ label, value }]` option arrays mirrored from server
enums plus `xxxLabel(value)` helpers with `"—"` fallback, so the form, the table and the dialog agree.

## 9. Session

**`useSessionExpiry()`** — mounted once (headless `session_expiry_warning.vue` in `App.vue`): polls every 30s;
5 minutes before `sessionExpiresAt` shows a persistent dialog "Session expiring … Extend your session?" with
**Extend** (`unelevated primary`) / **Logout** (flat); auto-logs-out at expiry.

**`services/jwt.js`** — `decodeJwtPayload`, `decodeJwtPermissions` (the repeated `permission` claim),
`jwtExpiresAt`, `isJwtExpired(token, skew = 60s)`. Never verifies, only reads.

## 10. Stores

- **`auth`**: `token`, `refreshToken`, `user { userId, email, displayName, tenants }`, `mustChangePassword`,
  `permissions`, `sessionExpiresAt`; getters `isAuthenticated`, `roles`, `hasPermission`, `hasAnyPermission`;
  actions `login`, `beginMicrosoftLogin`, `completeMicrosoftLogin`, `loadProfile`, `refresh` (single-flight),
  `ensureSession`, `logout`, `logoutAll`, `initialize`, `setUserInfo`, `clearSession` (wipes LocalStorage,
  clears the tenant store, dispatches `session-cleared`).
- **`tenant`**: `assignments`, `activeTenantId`, `hasUnsavedForm`, `loading`; getters `activeTenant`,
  `activeRoles`, `hasMultipleTenants`; actions `setAssignments`, `setActiveTenant`, `setUnsavedForm`,
  `switchTenant(tenantId, { confirm })`, `clear`.
- **`branding`**: `theme`, `assets`, `previewing`; getters `resolved`, `appName`, `tagline`, `footerText`,
  `supportEmail`, `supportUrl`, `showNameBesideLogo`, `login`, `logoUrl`, `logoDarkUrl`, `logoMarkUrl`,
  `loginBackgroundUrl`; actions `set`, `hydrate`, `keepCache`, `load`, `loadPublic(tenant)`,
  `loadForInvite(code)`, `preview(theme, assets)`, `endPreview`.

## 11. Dashboard and universal-feature composables

`useDashboardData` / `useDashboardLayout` are described in `dashboard.md`; `composables/uf/*`
(`useEntityMeta`, `useEntityActions`, `usePins`, `useColourCodes`, `useRowPersonalisation`,
`useFieldLogCounts`, `useNotificationMeta`, `useNotificationRoute`, `useShowDeleted`) in
`universal-features.md`.
