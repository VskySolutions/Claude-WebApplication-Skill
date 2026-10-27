# List pages: the recipe every CRUD / queue / inbox screen follows

Every list in the app has the same skeleton, the same persistence, the same row-action grammar and the
same empty state. Reproduce it exactly; the shared components already encode most of it.

## Contents
1. Page skeleton (template)
2. Script skeleton
3. Column definitions
4. Search, filters, quick filters
5. Row actions
6. Cell renderers
7. Selection and bulk actions
8. Empty, loading and error states
9. Variants: master-detail, embedded table, settings page, load-all list
10. Checklist for a new list page

---

## 1. Page skeleton (template)

```vue
<q-page padding>
  <app-list-header
    :breadcrumbs="[{ label: 'Home', icon: 'o_home', to: '/' }, { label: 'Users' }]"
    :search="search" show-search search-placeholder="Search name or email"
    show-filters :filter-count="filterChips.length"
    :show-add="canCreate" add-label="Create User"
    show-back
    @update:search="search = $event" @filters="filterOpen = true" @add="openCreate" @back="$router.back()"
  >
    <template #note>What this page is for, one or two sentences, on a tooltip.</template>
    <template #actions><!-- secondary outlined buttons / scope selects --></template>
  </app-list-header>

  <!-- optional: context cards and banners (a tenant-wide switch, "no active account" warning) -->

  <app-filter-drawer v-model="filterOpen" :chips="filterChips" @remove="removeFilter" @clear="clearFilters">
    <app-column-filters v-model="filters" :columns="filterableColumns" />
    <!-- extra server filters with no column: app-text-field / app-select / app-date-field pairs with :dense="false" -->
    <q-toggle v-if="canManageDeleted" v-model="showDeleted" label="Show deleted?" dense class="q-mt-md" />
  </app-filter-drawer>

  <app-data-table
    page-key="users" row-key="userId" title="All users"
    :rows="rows" :columns="columns" :loading="loading" :total-records="totalRecords" :pagination="pagination"
    selectable
    @request="onRequest" @refresh="load" @update:selected="selected = $event"
  >
    <template #quick-filters><!-- QuickFilterBar when the list has counted shortcuts --></template>
    <template #bulk-actions="{ selected: sel }"><!-- flat dense no-caps buttons --></template>
    <template #body-cell-isActive="cell">
      <q-td :props="cell"><q-badge :color="cell.value ? 'positive' : 'grey'">{{ cell.value ? "Active" : "Inactive" }}</q-badge></q-td>
    </template>
    <template #body-cell-actions="cell">
      <q-td :props="cell"><!-- see §5 --></q-td>
    </template>
    <template #no-data><!-- see §8 --></template>
  </app-data-table>

  <deleted-records-panel v-if="canManageDeleted" :entity-type="EntityType.User" :show="showDeleted" @restored="load" />

  <!-- create / edit drawer(s) and dialogs -->
</q-page>
```

Breadcrumb trail: `Home` → (group such as `Tenant Settings`/`Configuration`, no link) → page. The list
header card has no separate title row; the table's `title` ("All users", "Email templates") names the grid.

## 2. Script skeleton

```js
const notify = useNotify();
const { confirm } = useConfirm();
const { has } = usePermissions();
const auditColumns = useAuditColumns();
const { showDeleted, canManageDeleted } = useDeletedRecords();
const canCreate = computed(() => has(Permissions.UsersWrite));

const columns = computed(() => [ /* §3 */ ]);

const { rows, loading, totalRecords, selected, search, filterOpen, pagination, load, onRequest } = useListTable({
  pageKey: "users",
  fetcher: ({ page, limit, sortBy, descending }) =>
    userApi.list({ page, limit, sortBy, descending, search: search.value || undefined,
      isActive: typeof filters.isActive === "boolean" ? filters.isActive : undefined, ... })
      .then((r) => ({ data: r?.data, total: r?.meta?.totalRecords })),
  onError: (err) => notify.error(getApiErrorMessage(err))
});

const { filters, filterableColumns, filterChips, removeFilter, clearFilters, rangeBounds } =
  useColumnFilters(columns, rows, { server: true });
const reload = debounce(() => { pagination.value.page = 1; load(); }, 300);
watch([search, filters], reload, { deep: true });
```

Rules embedded here: filters and search always restart at page 1; the debounce is 300ms; option lists
for filters (people, clients) are loaded once on mount and failures stay silent ("a filter nobody can
populate simply stays empty"); the fetcher sends `undefined` for empty filters so nothing is sent.

## 3. Column definitions

```js
{ name: "fullName", label: "Name", field: "fullName", align: "left", sortable: true, default: true }
{ name: "roles", label: "Role", field: (r) => (r.roles || []).join(", "), align: "left", sortable: false, default: true }
{ name: "isActive", label: "Status", field: "isActive", align: "left", sortable: true, default: true,
  filterOptions: [{ label: "Active", value: true }, { label: "Inactive", value: false }] }
{ name: "sentOnUtc", label: "Sent", field: "sentOnUtc", align: "left", sortable: true, default: true, filterType: "dateRange" }
{ name: "client", label: "Client", field: "clientName", ..., filterOptions: clientOptions.value, filterMultiple: true }
...auditColumns({ dateRanges: true }),
{ name: "actions", label: "Actions", field: "actions", align: "left" }
```

Conventions:
- `align: "left"` on every column (numbers included); the Actions column is named `actions`, labelled
  "Actions", left-aligned and always last.
- `default: true` on the columns that matter on first open; the rest are reachable through the Columns
  menu ("every field the row carries is offered rather than being unreachable"). Audit: Updated By/On
  visible, Created By/On hidden.
- A column covered by the quick search is `filterable: false` (no duplicate filter box). Computed/count
  columns are `filterable: false`. Enum/boolean columns get `filterOptions` (values are strings or
  booleans; the fetcher converts: `filters.status === "true"`). People/client columns get options loaded
  from the API, `filterMultiple: true` for any-of.
- Date columns: `field` is the raw value, the cell renders `formatDateTime`; `filterType: "dateRange"`
  when the server can bound them.
- `sortable: false` when the server cannot order by it (multi-valued cells, counts computed after paging).
- `field` as a function when the cell text differs from the raw value (labels resolved from option sets,
  joined arrays) so sorting/export read the words; the cell slot then draws the badge.
- Tenant column only on platform-admin lists; everything else is scoped to the active tenant via the header.
- Columns are a `computed` when they depend on loaded option lists or permissions.

Default sort is `updatedOnUtc` descending (the platform convention: "every list opens on what was touched
most recently"); event lists (notifications, mentions, deleted records) sort by their own timestamp
(`createdOnUtc`, `deletedOnUtc`). Pass `default-sort-by` to the table and `defaultSortBy` to the composable.

## 4. Search, filters, quick filters

- **Search box** in the header: one `search` string sent to the server (`search`, or a named param such as
  `clientName`). Placeholder names what it matches: "Search name, email or code".
- **Filter drawer**: `AppColumnFilters` first, then extra controls for server filters that have no column
  (a contact box matching email or mobile, a nested status, a "Used by roles only" toggle, a date pair),
  then the admin-only "Show deleted?" toggle. Extra filters are kept in a `reactive extras` object, merged
  into `allChips`, and cleared by `onClearFilters` alongside `clearFilters()`.
- **Chips**: the drawer renders active-filter chips above the table (`teal-1`/primary, removable, "Clear
  all"). When the list has a **quick-filter bar**, pass `:show-chips="false"`; the bar already states the
  count and clears.
- **Quick filters** (`QuickFilterBar` + `QuickFilterGroup` in `#quick-filters`): counted shortcuts over the
  2–3 filters a reader toggles most. A group's `v-model` **is** the drawer filter (`filters.status`), so
  both stay in sync. A "View" group (Mine / All, `:clearable="false"`) switches the list's reading. Counts
  come from a `*/quick-counts` endpoint called with the same filter params every time the list loads.
  Remember the picks with `useFilterMemory(pageKey, { status: toRef(filters, "status"), view }, { status: "draft" })`.
- Date-range filters convert to UTC day boundaries: `const r = rangeBounds("sentOnUtc", fmt.zonedDayBoundaryUtc); sentFrom: r.from, sentTo: r.to`.
- Lists reload on `tenant-switched` automatically; the tenant scope is never a per-list dropdown.

## 5. Row actions

"One button per action, all of them on the row", in this grammar:

```vue
<q-td :props="cell">
  <!-- View / open first, as a LINK (real <a href>: middle-click and open-in-new-tab work) -->
  <q-btn flat round dense color="primary" icon="o_visibility" :to="{ name: 'user_detail', params: { id: cell.row.userId } }">
    <q-tooltip>View / Manage</q-tooltip>
  </q-btn>
  <!-- state changes: type="a" so the column renders one kind of tag; colour says direction -->
  <q-btn v-if="canWrite" type="a" flat round dense :color="cell.row.isActive ? 'grey-8' : 'positive'"
         :icon="cell.row.isActive ? 'o_block' : 'o_check_circle'" @click="setStatus(cell.row, !cell.row.isActive)">
    <q-tooltip>{{ cell.row.isActive ? "Deactivate" : "Activate" }}</q-tooltip>
  </q-btn>
  <!-- destructive last-but-one, negative; disabled with the reason in the tooltip -->
  <q-btn v-if="canDelete" type="a" flat round dense color="negative" icon="o_delete" :disable="cell.row.isUser" @click="remove(cell.row)">
    <q-tooltip>{{ cell.row.isUser ? "A person who is a user cannot be deleted" : "Delete" }}</q-tooltip>
  </q-btn>
  <!-- personal marks (pin / colour) last -->
  <entity-row-marks v-if="canMarkRows" ... />
</q-td>
```

- Order: primary action (pick up / claim) → View → Edit → secondary reads (email log, conversation) →
  destructive → personal marks. A claim action that is the first thing to do leads the row.
- Icon buttons are `flat round dense`, 20px icons, `color="primary"` for neutral actions, `negative` for
  destructive, `positive`/`grey-8` for activate/deactivate, `amber-8` for "pick up", a per-row `:loading`
  when an inline action runs.
- Every button has a tooltip; a disabled button's tooltip states why.
- Never put a `div`, `q-separator` or any block element in an actions cell.
- The whole row may also be clickable (`@row-click`) for inboxes, but the link button stays.
- A `⋮` menu (`EntityRowActionsMenu`) is the alternative when the universal actions (pin, colour, reminder,
  copy link, PDF) are offered per row; page actions go in its default slot above a separator.

Confirmation wording for row actions: `Deactivate <name>?`, `Delete <name>? This cannot be undone.`,
`Generate a new temporary password for <name>? Their current sessions will end.` — name the record, state
the consequence, `type: "danger"` for anything that removes or disables.

## 6. Cell renderers

| Need | Pattern |
|---|---|
| Boolean status | `q-badge :color="v ? 'positive' : 'grey'"` with words ("Active"/"Inactive", "User"/"Not a user") |
| Option-set value (status, type, decision) | `<app-option-badge :option="statusOption(row)" />` (label, colour, icon and tooltip from the data) |
| Person's name | `<app-name-with-suffix :name :suffix />` |
| Two-line cell | primary line + `div.fs-12.text-grey-6` / `text-caption text-grey-7` (code under a name, email under a client); long bodies `ellipsis-2-lines` |
| Link cell | `router-link class="text-primary text-weight-medium"` or `q-btn flat dense no-caps color="primary" :label :to`; a pinned mark before it |
| Dates | `fmt.formatDateTime(v)` / `formatDate`; `—` when empty |
| Empty value | `—` (`text-grey-6` when inline) — "a field nobody filled in is still information" |
| Counts | `q-badge color="teal-1" text-color="primary"`; `n / cap` with a `negative` "Full" badge |
| Progress tally | `q-badge` grey until > 0 then positive, plus "of N" caption, tooltip with the breakdown |
| Icon + label | `q-icon :name color="primary" class="q-mr-xs"` then text (entity type with icon) |
| Attention marker | small `q-icon` beside the value with a tooltip (`o_workspace_premium` for a department head) |
| Chips row | `row items-center q-gutter-xs` of `q-chip dense square size="sm"` with per-item tooltips |
| Nested table in a cell | a bordered panel component (`RelatedClientsCell`: tagged parent row with a primary top rule, child rows with fixed-width tags, monospace reference chips, a badge-dropdown per row) |
| Info tooltip on a cell | `q-icon name="o_info" size="14px" color="grey-6" class="q-ml-xs cursor-pointer"` + `q-tooltip` ("Created by …") |
| Strike-through deleted | `span.text-strike.text-grey-7` + `q-badge color="grey-6" label="Deleted"` |

Rich-text columns show `stripHtml(value)`.

## 7. Selection and bulk actions

`selectable` adds a left checkbox column; the table shows a `primary` chip "n selected" and the
`#bulk-actions="{ selected }"` slot: `q-btn flat dense no-caps` buttons (`positive` Activate, `negative`
Deactivate/Delete, `primary` Restore). Bulk handlers confirm with the count ("Delete 3 person(s)?"), skip
rows that cannot be processed and say so ("(1 linked to a user will be skipped.)"), run `Promise.all`, clear
`selected`, reload.

## 8. Empty, loading and error states

- Loading: `:loading="loading"` on the table (progress bar + spinning refresh). Lists never show a
  page-level spinner.
- Empty: the default is `o_inbox` + "No Data Available". Lists that have a creation flow override
  `#no-data` with `div.full-width.column.flex-center.q-pa-xl.text-grey-6` → 40px icon → `text-subtitle1`
  title ("No email accounts configured") → one explanatory line → a primary CTA (same label as the header's
  Add). Wording adapts to the active view ("You have not picked anything up" vs "Nothing to review yet") and
  to active filters ("No matching notifications" / "Try clearing the filters or the search.").
- Errors: `onError` toasts `getApiErrorMessage(err)`; the table stays with its last rows.
- Secondary data failures (filter option lists, quick counts, row colours) are silent.

## 9. Variants

- **Master-detail** (user groups): `row q-col-gutter-md` with `col-12 col-md-5` list and `col-12 col-md-7`
  detail; the detail is a second `AppDataTable` without a `page-key`, `client-sort`, `:pagination="{ rowsPerPage: 10 }"`,
  an `#actions` slot button ("Add members"), and an empty card ("Select a group to see its members.") when nothing is selected.
- **Table inside a detail page** (users of a tenant): `AppDataTable` with its own `page-key`, a search box
  in `#actions` (`q-input dense outlined debounce="300"` with `o_search` prepend, `min-width: 220px`) and an
  Add button; `useListTable({ reloadOnTenantSwitch: false })`.
- **Settings page** (not a list): `AppListHeader` with crumbs, `show-back` and a `#note`; then
  `q-card flat bordered` blocks with `q-card-section q-py-sm text-subtitle2 text-primary` heads
  (`q-icon size="18px" class="q-mr-xs"` + title), bodies, and `q-card-actions class="q-gutter-sm bg-grey-1"`
  footers (destructive flat negative left, `q-space`, outlined secondary + filled primary right). Settings
  pages listen to `tenant-switched` themselves.
- **Load-all lists** (roles, templates, tags): fetcher returns `{ data: r || [], total: (r || []).length }`,
  `useColumnFilters(columns, rows, { server: false })`, bind `:rows="filteredRows" :total-records="filteredRows.length"`;
  ordering is still the server's (`sortBy`/`descending` are sent).
- **Simple admin lists** (tags, sticky notes): a plain `row items-center q-mb-md` with `text-h6` title and
  an Add button is acceptable when there is no search or filter, but prefer `AppListHeader`.
- **Entity filter in the header** (option sets, mentions): an `AppSelect` in `#actions` with
  `style="min-width: 200px"` and `@update:model-value="load"`.

## 10. Checklist for a new list page

1. Route with `meta.permissions`, menu item with the same permission, breadcrumb trail.
2. `AppListHeader` with search placeholder naming the searchable fields, Filters with count, Add gated on
   the write permission, Back.
3. Columns: `default` flags, audit columns, `filterable`/`filterOptions`/`filterType` per column, actions last.
4. `useListTable` + `useColumnFilters` + debounced reload; `pageKey` unique across the app.
5. Row actions in the standard grammar with tooltips; confirmations for state changes.
6. Custom `#no-data` with a CTA when the user can create.
7. `DeletedRecordsPanel` + "Show deleted?" toggle when the entity is soft-deleted.
8. Create/edit via `AppFormDrawer`; detail via a route, never a modal.
9. Toast past-tense success ("Role created."), reload, and route to the new record when creation is the start
   of setting it up.
