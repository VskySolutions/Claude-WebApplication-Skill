---
name: vsky-ui-standards
description: The VSky / EMS Portal professional UI standard for Quasar 2 + Vue 3 business applications — design tokens, typography, the App* component library, list/detail/form/dialog recipes, feedback rules, app shell, tenant theming, dashboard widgets, auth screens, the API envelope the UI expects, and code conventions, with a verbatim UI kit to copy from. Use this whenever building or restyling any admin/portal web UI for a VSky project or a new application that should "look like EMS Portal", whenever someone asks for a standard list page, detail page, form drawer, dialog, dashboard, login screen, data table, filters, notifications, or a professional/consistent UI, and whenever reviewing a Vue/Quasar screen for consistency — even if the request only says "make a page for X" or "add a screen".
---

# VSky UI Standards (EMS Portal design system)

This skill captures **how the EMS Portal web application looks and behaves** — not its business model or
workflows — so other applications can be shaped the same way. It is written from the source
(`WEB/src`, Quasar 2 / Vue 3, snapshot 27 Oct 2026). Every rule below exists because the original codebase
learned it the hard way; the references explain the reasoning so you can apply the spirit, not just the letter.

## How to use this skill

1. Read this file. Then open the reference that matches the job (table below) — they are long and specific;
   read the whole file, not just the headings.
2. When starting a new application, copy from `assets/ui-kit/` (verbatim source, see its README) rather than
   re-implementing components. When working inside an existing VSky app, reuse its `App*` components and
   composables; never hand-roll a table, drawer, field or toast.
3. When reviewing a screen, walk the "Non-negotiables" list and the relevant recipe's checklist.

| Task | Read |
|---|---|
| Colours, fonts, spacing, radius, icons, breakpoints, CSS architecture | `references/foundations.md` |
| Layout, header, menu, routing, permission gating, persisted UI state | `references/app-shell.md` |
| What a shared component does and its props/slots | `references/components.md` |
| What a shared composable/helper does | `references/composables.md` |
| Building or reviewing a list / inbox / queue page | `references/list-pages.md` |
| Building a detail page, form drawer, dialog, read-only view, auto-save | `references/detail-and-forms.md` |
| Toasts, confirms, loading, empty states, banners, badges, tooltips, errors | `references/feedback-and-states.md` |
| Conversation, activity, checklists, attachments, tags, pins, colours, reminders, notifications, deleted records | `references/universal-features.md` |
| Per-tenant theming (logos, colours, fonts) and the branding editor | `references/branding-theming.md` |
| Dashboard widgets, KPI tiles, charts, layout customisation | `references/dashboard.md` |
| Login, reset, SSO, change password, session expiry, tenant switch | `references/auth-and-session.md` |
| The response envelope, list params, error codes the UI assumes | `references/api-contract.md` |
| File layout, naming, comments, lint, new-module checklist | `references/code-conventions.md` |
| Copying real source into a new project | `assets/ui-kit/README.md` + `FILELIST.txt` |

## The system in one page

**Stack.** Quasar 2.19 (Vite), Vue 3.5 `<script setup>` JavaScript, Pinia, vue-router (history), axios with
a shared envelope, Material Icons **Outlined** (`o_*`), Poppins 15px base, Quasar `Notify`/`Dialog`/`Cookies`/`LocalStorage`.

**Palette.** Teal primary `#1f6478` (`--q-primary` = `--teal-700`) with a ramp from `--teal-900 #123645` to
`--teal-050 #f3f9fa` (the page background); gold accent/warning `#b9862f`; warm ink greys `#16262c / #3a4d54 /
#647a82 / #a6b8bd`; border `#dde6e8`; positive `#21ba45`, negative `#c10015`, info `#2b7f96`. Quasar colour
props for Quasar things, CSS variables for bespoke surfaces; never a literal brand hex in a component
(tenants can re-theme the whole app at runtime).

**Shape.** White `q-card flat bordered` with 12px radius on a tinted page; nested cards 10px; inputs
outlined, dense, 40px tall, label **above** the control with a red star for required; buttons `no-caps`,
filled `unelevated color="primary"` for the one main action, `outline color="primary"` for secondary,
`flat` for cancel/tertiary, `negative` for destructive; icon buttons `flat round dense` with a tooltip.

**Type.** Titles 17px/600 in theme blue (`text-subtitle1`, `text-h5`); labels/buttons/badges/breadcrumbs 15px;
uppercase 11px/600 tracked sub-headings (`.section-subhead`) split forms into blocks; the read-only
"record vocabulary" (uppercase 11px grey label over a 14.5px value) renders every record view.

**Every screen** sits in `q-page padding` under a carded header: `AppListHeader` (breadcrumbs · search ·
Filters-with-count · Add · Back) for lists and settings, `AppDetailHeader` (breadcrumbs · status pills ·
actions · Back) for records. Every detail page ends with `AppRecordAudit`. Every list is `AppDataTable` +
`useListTable` + `useColumnFilters` with server paging/sorting, persisted column visibility/order/width/page
size, quick-filter bars with counts, a filter drawer, "one icon button per action" rows, a custom empty state
with a CTA, and a deleted-records panel for admins.

**Feedback.** Past-tense success toasts ("Person created."), the server's sentence on failure
(`getApiErrorMessage`), `useConfirm` before anything destructive or consequential (named record +
consequence + verb button), auto-save indicators instead of toasts for auto-saving cards, banners for
decisions and context, badges from option-set data (`AppOptionBadge`), tooltips on every icon button and on
every disabled control's reason.

**Access.** Routes, menu items and buttons are gated by permission keys; hide what can never be done,
disable-with-reason what is blocked by state; read-only is announced once in a banner. Multi-tenant: a
header tenant switcher, a super-admin "View as" scope with a page-wide orange banner, and `tenant-switched`
events that reload every list.

## Non-negotiables (the review checklist)

1. Reuse `App*` components and the shared composables; extract a component when a pattern repeats twice.
2. Labels above fields (`AppFieldLabel`), `aria-label` on the control, required star via `required`/trailing `*`.
3. Dense 40px fields in `row q-col-gutter-md` grids with the standard column widths (pairs `col-12 col-sm-6`,
   name block 6/4/2); `q-gutter-*` only for flex children; CSS `gap` in bespoke rows.
4. Lists: server-side paging and sorting, default sort `updatedOnUtc desc`, audit columns from
   `useAuditColumns`, actions column last and never wrapping, View as a real link, tooltips everywhere,
   filter count badge, custom empty state, 300ms debounced reload to page 1.
5. Details: header actions only (no bottom bar), identity card, 7/5 column grid of cards, record audit at
   the foot, read-only values as record text (never disabled inputs), banners for read-only/warnings.
6. Create/edit in `AppFormDrawer` (resizable, draft-saving, Cancel + Save footer); decisions in `q-dialog`;
   records on routes.
7. Dates: display `MM/DD/YYYY hh:mm AM` in the tenant zone; calendar dates as `YYYY-MM-DD` strings; date
   pairs bound each other; range filters convert to UTC day boundaries.
8. Status/type/decision values come from option sets and render through `AppOptionBadge` (label, colour,
   icon, description tooltip from data).
9. Feedback per `feedback-and-states.md`: toast writes, not reads; confirm destructive and consequential;
   silence for degraded background reads; errors in place where typing happens.
10. Icons `o_*` outlined; 20px row icons; the vocabulary table in `foundations.md` §8.
11. Phone-first details: fields 16px below 600px, headers and bars wrap, split panes stack below 1024px,
    big dialogs maximise below md.
12. Comments explain why in one or two lines; double quotes, semicolons, 2 spaces; no legacy `Zw*`/`zw*` helpers.

## Bootstrapping a new application (summary)

1. Create the Quasar project from the kit's `package.json`/`quasar.config.js`/`eslint.config.js`; copy
   `src/css`, `src/boot`, `src/layouts`, `src/shared`, `src/pages`, `src/router`, `src/components/common`,
   `src/composables`, `src/utils`, `src/stores`, `src/services/{api,branding,jwt}.js` (trim resource groups),
   `src/components/app_menu.vue`; add `config/env.*.cjs`.
2. Set the brand in three places together: `css/tokens.scss`, `css/quasar.variables.scss`,
   `services/branding.js DEFAULT_THEME`; set `applicationName`, stock logo `assets/logo.png`, favicons.
3. Replace the `Permissions` catalogue, `EntityType`, `useEntityMeta`, the menu sections and `LANDING_PATH`.
4. Make the API return the envelope in `api-contract.md`; wire `authApi`, `brandingApi`, `optionSetApi`, `uf*Api`.
5. Build each module with the list and detail recipes; add universal features by registering the entity type.
6. Keep the option-set pattern for every status-like value so tenants can rename/recolour without code.

## Scope notes

- Domain workflows (REMS, approvals, Maconomy sync) are deliberately absent; where a workflow screen
  contributed a reusable UI pattern (split pane, quick filters, badge-as-dropdown, compose dialog,
  repeatable rows, acting-as banner), only the pattern is described.
- Known legacy in the source (old `Zw*` inputs, `dialogs/*`, old global CSS) is flagged, not endorsed.
- Facts not verifiable from the code (e.g. how a screen renders in a browser) are not claimed; everything
  here is read from the source.
