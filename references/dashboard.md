# Dashboard: widgets, layout persistence, KPI tiles, charts

Source: `modules/dashboard/*`, `components/dashboard/*`, `composables/useDashboardData.js`,
`composables/useDashboardLayout.js`.

## Contents
1. Page anatomy
2. Widget registry and role resolution
3. `DashboardWidgetWrapper`
4. Layout persistence and the Customise panel
5. KPI tiles
6. Charts
7. Export and print
8. Data composables

---

## 1. Page anatomy

```vue
<q-page padding>
  <app-list-header :breadcrumbs="[{ label: 'Dashboard', icon: 'o_home' }]">
    <template #actions>
      <q-btn-dropdown outline no-caps color="primary" :label="dateRangeLabel" icon="o_calendar_today">…Today / Last 7 / 30 / 90 days (check on current)…</q-btn-dropdown>
      <q-btn outline no-caps color="primary" icon="o_refresh" label="Refresh" :loading="anyLoading" @click="refreshAll" />
      <q-btn-dropdown outline no-caps color="primary" icon="o_download" label="Export">Export CSV (o_table_view) · Print / PDF (o_print)</q-btn-dropdown>
      <q-btn outline no-caps :color="customiseOpen ? 'secondary' : 'primary'" icon="o_tune" label="Customise" @click="customiseOpen = !customiseOpen" />
    </template>
  </app-list-header>

  <q-banner v-if="customiseOpen" dense rounded class="bg-teal-1 text-primary q-mb-md no-print">Drag widgets by the handle to reorder them…</q-banner>

  <div class="row q-col-gutter-md">
    <div v-for="widget in visibleWidgets" class="col-12 col-md-6 col-lg-4 dashboard-grid-item" :draggable="customiseOpen" …>
      <div v-if="customiseOpen" class="dashboard-grid-item__handle"><q-icon name="o_drag_indicator" /></div>
      <component :is="resolveComponent(widget)" v-bind="dataFor(widget)" :collapsed="…" @update:collapsed @retry />
    </div>
  </div>

  <dashboard-customise-panel v-model="customiseOpen" :role :layout />
  <div v-if="!visibleWidgets.length" class="column flex-center q-pa-xl text-grey-6"><q-icon name="o_dashboard" size="40px" />No widgets to display.</div>
</q-page>
```

All header actions are outlined primary buttons; the active Customise button switches to `secondary`.
The date range is remembered in the page cookie (`dateRange`, default `7d`).

## 2. Widget registry and role resolution

`modules/dashboard/widgets/registry.js` is the single source of truth:
`{ key, title, description, role: "common" | "tenantAdmin" | "superAdmin", category, component: () => import(...) }`.
Helpers: `defaultLayoutForRole(role)`, `widgetsForRole(role)`, `defaultHiddenForRole()` (none), `WIDGETS_BY_KEY`.
The page resolves the role from permissions (`tenants.write` → superAdmin; `users.read` + `tenants.read` →
tenantAdmin; else common), mirroring the server. Components load lazily via `defineAsyncComponent`, cached per key.

## 3. `DashboardWidgetWrapper`

Every widget renders inside it (`q-card flat bordered`, 12px radius, full height, column flex):
- Header (`q-py-sm`): `text-subtitle1 text-weight-medium text-primary ellipsis` title; an "Attention" warning
  chip when collapsed and `alert`; spacer; an optional navigation action (`flat dense no-caps size="sm"
  color="primary" icon="o_open_in_new"` + label, shown only with `actionLabel` + `actionRoute` and, when
  `actionPermission` is set, the permission); collapse (`o_expand_more`/`o_expand_less`) and fullscreen
  (`o_open_in_full`/`o_close_fullscreen`) round buttons with tooltips.
- Body: skeleton while `loading`; `bg-red-1 text-negative` banner with **Retry** on `error`; else the slot.
- Fullscreen: `position: fixed; inset: 0; z-index: 6000`, Esc closes, a fixed round primary `o_close` top-right.
- Props: `widgetKey`, `title`, `loading`, `error`, `collapsed`, `alert`, `actionLabel`, `actionRoute`,
  `actionPermission`. Emits `retry`, `update:collapsed`.

A widget component therefore has the uniform props `widgetKey, title, loading, error, collapsed` + its data
props, emits `retry` and `update:collapsed`, and shows `text-grey-6 q-pa-md text-center` "No data" when its
data is null.

## 4. Layout persistence and the Customise panel

`useDashboardLayout(role)` keeps `widgetOrder`, `hiddenWidgets`, `collapsedWidgets`; loads them from the API
(falling back to the role default), saves them debounced (500ms) with an error toast; `setOrder`,
`toggleHidden`, `setCollapsed`, `resetToDefault` (confirm "Reset dashboard … Your customisations will be
lost." then toast "Dashboard layout reset.").

Drag-to-reorder uses native HTML5 drag events only while customising (`dragIndex`/`dragOverIndex`,
`opacity .5` on the dragged item, a dashed primary outline on the target, a `cursor: grab` handle at the top
right); the new visible order is merged back into the full order so hidden widgets keep their places.

`DashboardCustomisePanel`: a right `q-drawer overlay bordered :width="360"` titled "Customise Dashboard"
(`o_tune`), a caption, a scroll list of widgets (title + a positive "New" badge for widgets not yet seen,
description caption, a primary `q-toggle` for visibility) and a full-width outlined "Reset to Default".

## 5. KPI tiles

Inside a widget: `row q-col-gutter-sm` of `col-6 col-sm-4` (or `col-4`) tiles, each a `q-card flat bordered`
with 8px radius and `q-pa-sm`:
- Icon variant: 22px coloured icon left, `text-caption text-grey-7 ellipsis` label over a
  `text-h6 text-weight-bold` value (value class carries the semantic colour, e.g. `text-positive` at ≥ 90%).
- Centred variant: bold `text-h6` value (primary, or `text-warning` when it needs attention with a small
  `o_warning` beside the caption) over the caption.
- Clickable tiles (`cursor-pointer`, hover shadow) navigate to the relevant list with a query
  (`/users?inactive=30`). A `--warn`/`--highlight` modifier colours the border `var(--q-warning)` (and
  `#fff8e1` background for highlights).
- The widget raises `alert` when any tile warns, so the collapsed header shows "Attention".

List-style widgets use `q-list separator` items with icons coloured by severity (`negative` critical/error/high,
`warning` medium, `info` otherwise), a `q-badge` for the severity, and clickable rows that navigate.
Table-style widgets embed `AppDataTable` with `client-sort` and a `page-key`.

## 6. Charts

Dependency-free SVG components with `role="img"` and `<title>` hover tooltips, plus an HTML legend:
- `DonutChart` (`segments: [{ label, value, color }]`, `centerLabel`): 42×42 viewBox ring, "No Data" when the
  total is 0, legend with swatch · label · bold value.
- `BarChart` (`categories`, `series: [{ name, color, values }]`, `stacked`): grouped or stacked bars, 200px tall.
- `LineChart` (`labels`, `series`): multi-polyline, 200px tall.
- Palette used by widgets: `#1976d2 #26a69a #7e57c2 #ef6c00 #c2185b #00897b #5c6bc0 #43a047`; "Unassigned"
  is `#90a4ae` and italic in legends; legend rows are clickable to drill into the list.

## 7. Export and print

- CSV: built client-side from the visible tabular widgets (sections separated by a blank line, values
  escaped), downloaded as `dashboard-YYYY-MM-DD.csv`; warns when nothing exportable is visible.
- Print/PDF: `window.print()` after an info toast; an unscoped `@media print` block hides `.no-print`,
  drawers, banners, breadcrumbs and buttons and makes each widget full width with `page-break-inside: avoid`.

## 8. Data composables

`useUserDashboard(dateRange, tenantId)` and `usePlatformDashboard(dateRange)` each expose `loading`, `error`
(via `getApiErrorMessage`), `refresh()` and the data slices; they refresh on mount and whenever the date
range changes. The platform one supports `refresh(true)` to bypass the server cache (a request header).
The page maps each widget key to its slice in `dataFor(widget)` and routes `retry` to the right `refresh`.
