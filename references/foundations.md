# Foundations: stack, tokens, type, shape, spacing, icons, responsive rules

The visual system of the EMS Portal web app (Quasar 2 / Vue 3), reduced to the facts a new
application needs to look and behave the same. Everything here is taken from the source
(`WEB/src/css/*`, `quasar.config.js`, the shared components) as of 27 Oct 2026.

## Contents
1. Stack and versions
2. Quasar configuration that shapes the UI
3. Design tokens (colour)
4. How colour is used (semantics)
5. Typography
6. Shape, elevation, surfaces
7. Spacing and grid
8. Icons
9. Responsive rules
10. Accessibility rules baked into the components
11. CSS architecture (load order, globals vs scoped, naming)
12. Legacy to leave behind

---

## 1. Stack and versions

| Concern | Choice | Notes |
|---|---|---|
| Framework | Vue 3.5, `<script setup>` only, JavaScript (no TypeScript) | Options API is not used anywhere. |
| UI kit | Quasar 2.19 via `@quasar/app-vite` 2.6, Vite 8 | SPA, history mode, no SSR/PWA. |
| State | Pinia 3 (`stores/auth`, `stores/tenant`, `stores/branding`) | Small stores; most page state is local `ref`/`reactive`. |
| Routing | vue-router 5, one `routes.js` per feature module | Route `meta` carries `requiresAuth`, `permissions`, `title`. |
| HTTP | axios 1.14, two instances (`http` authenticated, `http2` anonymous) | Interceptors: bearer token, correlation id, tenant headers, 401 refresh, 429 notice. |
| Validation | Quasar `:rules` on fields + `q-form greedy`; `@vuelidate/core` on auth forms | Server validation errors are mapped back onto fields. |
| Dates | `Intl.DateTimeFormat` in the tenant time zone (`useDateFormat`); `date-fns` available | Display `MM/DD/YYYY hh:mm AM`. |
| Phone | `libphonenumber-js` (as-you-type, E.164 storage) | `AppPhoneInput`. |
| Geography | `country-state-city` (country → state → city cascade), `validator` (postal codes) | `AppAddressFields`, `useCountries`. |
| Rich text | CKEditor 5 (v48) with a deliberately small plugin set; Quasar `q-editor` for the chat composer | `AppRichTextField`, `EntityConversationPanel`. |
| Images | `vue-advanced-cropper` | `AppImageUpload` crops to PNG. |
| Numbers | `vue-currency-input` (legacy `ZwCurrency`/`ZwNumeric`), `big.js` | New code uses `app-text-field type="number"` with `$`/`%` affixes. |
| Charts | Dependency-free inline SVG components; `d3`, `d3-org-chart`, `@fullcalendar/*`, `powerbi-client` are installed but unused by the current screens | Prefer the SVG components for simple KPIs. |
| Misc | `@vueuse/core` (`useDebounceFn`), `lodash`, `qs`, `vuedraggable` (installed), `vue-i18n` 11 (installed, en-US only, strings are hard-coded English) | |
| Lint | ESLint 9 flat config, `neostandard` with `semi: true`, double quotes, 2-space indent, `eslint-plugin-vue` flat/recommended | `no-unused-vars` is an error; `vue/no-undef-properties` is an error. |

Browser targets: `es2019, edge88, firefox78, chrome87, safari13.1`. Node 16/18 for the toolchain.

## 2. Quasar configuration that shapes the UI

- `framework.iconSet: "material-icons-outlined"` and every icon name in the code uses the `o_` prefix
  (`o_edit`, `o_visibility`). Extras loaded: `material-icons-outlined`, `mdi-v5`, `fontawesome-v6`, `roboto-font`.
- Plugins: `Notify`, `Dialog`, `Loading`, `LocalStorage`, `Cookies`, `Meta`, `BottomSheet`.
- CSS files load in this order and the order matters (later wins at equal specificity):
  `typography.scss` → `app.scss` (imports `tokens.scss`) → `page.scss` → `custom.scss` → `rems.scss`.
  `custom.scss` ends with the "Standard application typography" block that deliberately overrides the
  older size rules above it.
- Boot files in order: `error`, `axios`, `interceptors`, `branding`, `title`, `components`, `i18n`.
- Path aliases: `components/`, `composables/`, `services/`, `stores/`, `modules/`, `dialogs/`, `utils/`,
  `shared/`, `assets/`, `layouts/`, `pages/`, `boot/`, `src/`.
- Manual vendor chunks: `vendor-vue`, `vendor-quasar`, `vendor-fullcalendar`, `vendor-d3`, `vendor`.
- `devServer.open: true`; `distDir` is `../publish/spa/<folder>` per environment (`config/env.<env>.cjs`,
  selected with `QENV`; scripts via `nps`: `dev`, `build_test`, `build_prod`, `lint`).
- `index.html` preconnects to Google Fonts, sets a plain CSS spinner (`.loader`) inside `#q-app` before the
  bundle paints, and has no `maximum-scale` in the viewport meta (pinch-zoom is allowed on purpose).
- IIS hosting needs `public/web.config` with the history-mode rewrite to `index.html`.

## 3. Design tokens (colour)

Defined once in `src/css/tokens.scss` on `:root`, and mirrored into Quasar's brand variables in
`quasar.variables.scss`. The two **must stay in step** because `color="primary"` and `var(--teal-700)`
are used side by side.

| Token | Hex | Role |
|---|---|---|
| `--teal-900` | `#123645` | Dark chrome, the "selected/on" state of custom chips and tiles, `$dark-page` |
| `--teal-800` | `#184a5c` | Hover of an "on" teal-900 element |
| `--teal-700` | `#1f6478` | **Primary / action colour** (`$primary`, `--q-primary`) |
| `--teal-600` | `#2b7f96` | `$info` |
| `--teal-500` | `#3d97ad` | Focus rings (`outline: 2px solid var(--teal-500)`) |
| `--teal-300` | `#8fc7d6` | Hover borders |
| `--teal-100` | `#e7f3f6` | Tints |
| `--teal-050` | `#f3f9fa` | **Page background** (`body`), locked/info banners, card heads |
| `--gold-600` | `#b9862f` | `$accent` and `$warning` (warning is gold, not stock amber) |
| `--gold-100` | `#f7ecd8` | Gold tint |
| `--cream` | `#f4f2e6` | Rarely used warm surface |
| `--ink-900` | `#16262c` | Body text (`$dark`, `--q-dark`) |
| `--ink-700` | `#3a4d54` | Secondary text, quick-filter labels |
| `--ink-500` | `#647a82` | Muted text, field labels in record views (`$secondary`) |
| `--ink-300` | `#a6b8bd` | Disabled text, quiet info icons |
| `--line` | `#dde6e8` | Borders and dividers |
| `--white` | `#ffffff` | |
| `--radius` | `10px` | Default bespoke radius |
| `--shadow` | `0 2px 10px rgba(18,54,69,.07)` | Soft shadow |
| `--shadow-lg` | `0 12px 32px rgba(18,54,69,.16)` | Raised shadow |

Quasar semantic colours: `$positive #21ba45`, `$negative #c10015`, `$info #2b7f96`, `$warning #b9862f`.

Rule of thumb from the source: use Quasar colour props (`color="primary"`, `bg-teal-1`, `text-grey-7`)
for anything Quasar styles (buttons, chips, badges, text utilities), and the CSS variables for bespoke
surfaces, borders and shadows. Never hard-code the brand hex in a component; branding (see
`branding-theming.md`) rewrites both the `--q-*` and the token variables at runtime.

## 4. How colour is used (semantics)

**Brand tint chips.** `q-chip dense color="teal-1" text-color="primary"` is *the* neutral brand chip
(selected values, role badges in the header, group memberships, permission keys, filter chips,
placeholder tokens). Quasar's `teal-1` (`#e0f2f1`) is also the menu's active background
(`bg-teal-1`) and the unread row tint. Branding re-tints `.bg-teal-1` from the primary colour.

**Meta chips.** `color="blue-grey-1" text-color="blue-grey-8"` for informational metadata that is not a
status (a record code, a department, "3 permissions", a person type).

**Status badges (`q-badge`).**

| Meaning | Colour |
|---|---|
| Active / success / present | `positive` |
| Inactive / default / off | `grey`, `grey-5`, `grey-6` |
| Archived / System / platform-owned | `blue-grey` (or `indigo-5` for "Platform" scope) |
| Custom / ours / a count | `primary` |
| Error / full / rejected | `negative` |
| Waiting / overdue / attention | `orange-8` or `amber-8` |
| Warning (palette gold) | `warning` |

**Banners (`q-banner dense rounded`, avatar icon in the matching colour).**

| Intent | Classes |
|---|---|
| Information / context | `bg-teal-1 text-primary` (icon `o_info`), or `bg-blue-grey-1 text-blue-grey-9` for a read-only notice (icon `o_lock`) |
| Warning / needs action | `bg-orange-1 text-orange-9` (icon `o_warning`), page-wide scope banner `bg-orange-2 text-orange-10` |
| Error | `bg-red-1 text-negative` or `text-red-9` (icon `o_error`) |
| Success | `bg-green-1 text-green-9` (icon `o_check_circle` / `o_verified`) |
| Neutral / closed | `bg-grey-2 text-grey-8` |
| Attention that is a fact, not a problem (corrected data, pending round) | amber `#fff8e1` / `#6d4c00` or `#8a5a00` |

Custom alert blocks follow the same hues as hex: warn `#fff4e5/#8a5300`, reject `#fdecea/#8a1c12`,
lock `var(--teal-050)/var(--teal-900)`, ok `#e8f5ec/#1b5e34`.

**Personal marks.** Row pin tint `#fbfaf5`; row colour stripe 4px on the first cell from the
8-swatch palette `#ef5350 #ec407a #ab47bc #5c6bc0 #42a5f5 #26a69a #9ccc65 #ffa726`.

**Data-driven colours.** Option-set values carry their own `backgroundColor`/`textColor`/`icon`;
`AppOptionBadge` renders them and falls back to `#9e9e9e` on white. Tags carry a `colour` (default
`#607d8b`). Avatars for people are coloured by hashing the user id into a 10-colour palette.

**Legacy orange.** The header hover colour `#fc9c04` and the orange `o_account_circle` avatar icon in
the header are legacy accents; do not spread them.

## 5. Typography

- Body font **Poppins** (Google Fonts, every weight), set in `quasar.variables.scss`
  (`$typography-font-family`) and on `body`. Merriweather is also loaded (legacy description styling).
  Roboto is loaded for numbers: `.text-number`, `.text-currency`, `.text-date`, `.text-roboto`.
- Base size **15px** on `body` (set in `custom.scss`, the last stylesheet); fields are `12px` by legacy
  global (`.q-field { font-size: 12px }`) but every App field is `dense` and reads at the browser size;
  below 600px fields become 16px (iOS zoom rule).
- Headings have no margins (`h1..h6 { margin: 0 }`).

| Class | Size / weight | Colour |
|---|---|---|
| `h1`, `.text-h1`, `h2`, `.text-h2` | 18px / 600 | inherit (dialog titles use `text-h2` in legacy dialogs) |
| `h3`, `.text-h3` | 16px / 600 | inherit |
| `.text-h4`, `.text-h5`, `.text-h6`, `.text-subtitle1` | **17px / 600**, line-height 1.3 | `.text-h5` and `.text-subtitle1` are **theme blue** (`var(--q-primary)`); `h6`/`h4` keep the inherited colour so titles on primary backgrounds stay white |
| `.text-subtitle2` | 15px / 600 | inherit (often `text-primary` for small card heads) |
| `.text-body1/.text-body2`, `.text-caption`, `.text-overline` | Quasar defaults, line-height 1.3 | captions are usually `text-grey-7` |
| `.q-btn`, `.q-breadcrumbs`, `.q-badge`, `.q-chip`, `.q-table thead th` | **15px** | |
| `.fs-8` … `.fs-30` | utility font sizes in px | used sparingly (`fs-12` captions, `fs-16`/`fs-18`/`fs-20` wordmarks) |

**Section sub-heading** (one definition, `custom.scss`):
`.section-subhead { font-size: 11px; font-weight: 600; letter-spacing: .04em; text-transform: uppercase; color: var(--q-primary); margin: 16px 0 8px; }`
Used to split a form into labelled blocks ("Name", "Contact", "Billing"). Several components still carry
a scoped copy with `margin: 4px 0 8px`; a new app should rely on the global one.

**Record vocabulary** (`rems.scss`, global on purpose; rename to `.record-*` in a new app):
- `.rems-label` 11px / 600 / .04em / uppercase / `var(--ink-500)` / margin-bottom 3px, the name of a field in a read-only view.
- `.rems-value` 14.5px / `var(--ink-900)` / `word-break: break-word` / `white-space: pre-wrap`, the answer.
- `.rems-subhead` same metrics as the label but in `var(--q-primary)` with `margin: 4px 0 8px`, a heading over a group of fields.
- `.rems-inner { border-radius: 10px }` a card nested inside a card.

External **field labels** (`AppFieldLabel`): 12px / 500 / `#423939` / line-height 1.2 / margin-bottom 4px;
the required star is 18px red `#e53935` with `line-height: 0` so it does not grow the line.

Other recurring text roles: `.LastUpdatedBy` 11px italic grey; list group headers in the menu 11px uppercase
.08em bold primary; quick-filter group labels 12px / 700 / uppercase / `--ink-700`; tooltip body `text-body2`.

## 6. Shape, elevation, surfaces

- Page surface: `body { background: var(--teal-050) }` so white cards read as raised. Auth and public
  pages use `#f4f6fb`.
- **Cards in pages are `q-card flat bordered`** with a scoped `border-radius: 12px` (every page defines a
  `*-card { border-radius: 12px }` class; the shared headers, table and audit card do the same). The
  global `.q-card` has `border-radius: 5px` and a soft shadow for anything not flat.
- Nested card inside a card: 10px radius (`.rems-inner`, `.ai-block`, `.cif-entity`, role blocks).
- Dialogs: 12px radius on bespoke dialog cards; auth/error cards 16px; account cards 16px.
- Chips as status pills in a detail header: `border-radius: 999px`, 24px tall, 11.5px / 600.
- Inputs: Quasar outlined, dense, `hide-bottom-space`, control background `#f9f9f9`, min-height 40px
  (`.app-select :deep(.q-field__control) { min-height: 40px !important }`). Read-only pseudo-field
  (`AppReadonlyField`): `#f7f9fc` background, `#e0e6ed` border, 8px radius, 40px tall.
- Dropzones: `1.5px dashed #c2cad6`, 10px radius, `#fafbfc` background; hover/over turns the border primary
  and the background `#f3f8fc` / `#e9f3fb`.
- Buttons: Quasar default 3px radius; `.btnRounded { 6px }` exists. Global `.q-btn { padding: 3px 10px }`.
- Header: white, `border-bottom: 1px solid rgba(202,202,202,.171)`, shadow `0 2px 4px #0000001a, 0 6px 20px #0000001a`.
  Drawer: white, bordered. Footer: white, bordered.
- Table: `.app-data-table { border-radius: 12px }`, sticky white header cells, body cells 28px tall / 12px
  text (legacy global), header 38px / 15px / weight 500.
- Shadows for floating things: sticky notes `0 4px 14px rgba(0,0,0,.2)`, jump button `0 2px 8px rgba(16,24,40,.2)`,
  KPI card hover `0 2px 8px rgba(0,0,0,.12)`.

## 7. Spacing and grid

- Quasar spacing scale: `xs 4px`, `sm 8px`, `md 16px`, `lg 24px`, `xl 48px`. Pages are `q-page padding`.
  Cards stack with `q-mb-md`. Card sections keep Quasar's 16px padding except dense forms (public intake
  form: 10px 12px) and compact card heads (`q-py-sm`).
- **Field grids** are `div.row.q-col-gutter-md` with App fields as direct children carrying `col-*` classes.
  Public/dense forms use `q-col-gutter-sm`.
- **Flex children** (avatars, chips, buttons) use `q-gutter-sm`/`q-gutter-xs`, never `q-col-gutter-*`
  (the grid gutter pads the child and breaks avatar centring).
- Bespoke rows prefer CSS `gap` over `q-gutter-*`: the utility's negative margins fight card padding once a
  row wraps, and a nested gutter never cancels (documented in `AppColumnFilters` and `AppListHeader`).
- Standard column patterns:
  - pairs `col-12 col-sm-6`; triplets `col-12 col-sm-4`; `col-12 col-sm-8` + `col-12 col-sm-4` for a long field beside a short one.
  - name block: First `col-12 col-sm-6`, Last `col-8 col-sm-4`, Suffix `col-4 col-sm-2` (suffix shares the line with the surname at every width).
  - detail pages: left `col-12 col-md-7`, right `col-12 col-md-5`; master-detail lists `col-md-5` / `col-md-7`.
  - dashboard widgets `col-12 col-md-6 col-lg-4`; KPI tiles `col-6 col-sm-4` or `col-4`; auto-fit CSS grids for record views (`minmax(220px,1fr)`, `minmax(190px,1fr)` for narrow panes).
- Avatar in a gutter row goes in its own `col-auto` so the gutter padding does not land inside the circle.
- Drawers: form drawer default 50% of viewport (min 30%, max 95%), filter drawer 360px (min 300, max 60% or 92% on phones), view drawer 420px (560 for history), customise panel 360px. Main nav drawer 292px / 60px mini.

## 8. Icons

Material Icons **Outlined** everywhere, named with the `o_` prefix. Sizes: 20px row-action icons
(enforced by `.app-data-table tbody .q-btn .q-icon { font-size: 20px !important }`), 20px menu icons,
18px icons in list items and small card heads, `size="sm"` beside card titles, 14px info icon in a field
label, 16px section info tip, 38px header avatar, 40px empty-state icon, 80px error-page icon.

Vocabulary used consistently:

| Action / meaning | Icon |
|---|---|
| View / open | `o_visibility` |
| Edit | `o_edit`; inline note edit `o_edit_note`, document edit `o_edit_document` |
| Delete / remove | `o_delete`; permanent `o_delete_forever`; take off a list `o_remove_circle_outline`; member removal `o_person_remove`; close/remove chip `o_close` |
| Add / create | `o_add`; add person `o_person_add` / `o_group_add` |
| Back | `o_arrow_back`; forward `o_arrow_forward` / `o_arrow_right`; step `o_chevron_left` / `o_chevron_right` |
| Save | `o_save`; send `o_send` |
| Search | `o_search`; filters `o_filter_list`; clear filters `o_filter_alt_off`; columns `o_view_column`; refresh `o_refresh`; drag `o_drag_indicator` |
| Status | success `o_check_circle` / `o_verified` / `o_task_alt`; error `o_error` / `o_error_outline`; warning `o_warning`; info `o_info`; locked `o_lock`; closed `o_cancel`; waiting `o_hourglass_top` / `o_pending` / `o_schedule` |
| Activate / deactivate | `o_check_circle` / `o_block`; toggles `o_toggle_on` / `o_toggle_off`; archive `o_archive`; restore `o_restore` / `o_restore_from_trash`; reset `o_restart_alt` |
| Files | upload `o_upload` / `o_cloud_upload` / `o_upload_file`; download `o_download`; attachment `o_attach_file`; file types `o_image`, `o_picture_as_pdf`, `o_table_chart`, `o_description`, `o_slideshow`, `o_folder_zip`, `o_insert_drive_file`; open in new tab `o_open_in_new`; copy `o_content_copy`; link `o_link` / `o_link_off`; print/PDF `o_print` |
| Collaboration | chat `o_forum` / `o_chat`; history `o_history`; tags `o_label` / `o_label_off`; pin `o_push_pin`; colour `o_palette`; reminder `o_alarm`; mention `o_alternate_email`; checklist `o_checklist`; notifications `o_notifications` / `o_notifications_active` / `o_notifications_off` / `o_mark_email_read` / `o_done_all` |
| Navigation / shell | menu `o_menu`; home `o_home`; dashboard `o_dashboard`; account `o_account_circle`; profile `o_person`; tenant `o_apartment` / `o_business` / `o_corporate_fare`; users `o_group` / `o_groups`; roles `o_admin_panel_settings`; permissions `o_workspaces` / `o_vpn_key` / `o_key`; settings `o_settings` / `o_tune`; mail `o_mail` / `o_drafts`; branding `o_palette`; integrations `o_hub`; option lists `o_list_alt`; more `o_more_vert`; expand `o_expand_more` / `o_expand_less`; fullscreen `o_open_in_full` / `o_close_fullscreen`; logout `o_logout`; devices `o_devices`; view-as `o_travel_explore`; calendar `o_calendar_month` / `o_calendar_today` / `o_event`; export `o_table_view`; password `o_lock_reset`; show/hide `o_visibility` / `o_visibility_off`; empty inbox `o_inbox`; 404 `o_travel_explore` |

Icons sit left of their label (`icon=`) except "continue" buttons which use `icon-right`.

## 9. Responsive rules

Quasar breakpoints: `xs < 600`, `sm 600–1023`, `md 1024–1439`, `lg 1440–1919`. Rules the code applies:

- Drawer `breakpoint 1024`: above it the nav is always mounted (and can collapse to a 60px icon rail with
  hover flyouts); below it the menu button opens/closes an overlay.
- `gt-xs` hides the role chips and the tenant-scope select on phones.
- Below 600px: form fields become 16px (stops iOS zoom); list-header tools take a full line and the search
  box flexes; sticky action bars stack (save state on top, two buttons split the row); card section
  padding drops to `12px 10px`; tab-strip actions wrap under the tabs; quick filter bars wrap.
- Below 1024px: split panes stack (reading pane first), the drag gutter hides; large editing dialogs
  `maximized`; compose dialog goes to 92vw.
- Grids use `repeat(auto-fit, minmax(…))` instead of fixed column counts wherever a record is laid out.
- Mobile-only alternatives: sticky notes render as stacked `q-banner`s at the bottom on `$q.platform.is.mobile`.

## 10. Accessibility rules baked into the components

- Every field has a visible label **above** the control (`AppFieldLabel`) and the control carries an
  `aria-label` derived from the same text (`useFieldLabel` strips a trailing `*`).
- Icon-only buttons always carry a `q-tooltip`; destructive icon buttons also carry `aria-label` when the
  tooltip is conditional.
- Custom choice controls are real `<button type="button" role="radio" aria-checked>` groups inside
  `role="radiogroup"` with `aria-labelledby`; the pane splitter is `role="separator"` with `aria-valuenow`,
  keyboard arrows, Home, and double-click reset; dropzones are `role="button" tabindex="0"` with Enter/Space.
- Focus: `:focus-visible { outline: 2px solid var(--teal-500); outline-offset: 2px }` on custom controls.
- Disabled controls that need explaining get their reason as a sibling note (a disabled button shows no
  tooltip) or the tooltip is placed on the wrapper.
- Pinch zoom is allowed (WCAG 1.4.4); fields are 16px on phones so iOS does not zoom.
- Rich text is sanitised before `v-html` (`utils/richText.sanitizeHtml`), external links get
  `target=_blank rel=noopener noreferrer`.
- Branding preview warns when the primary button's contrast ratio falls under 4.5.

## 11. CSS architecture

- Globals live only in `src/css/*.scss`. Components use `<style scoped>` with BEM-like names
  (`.block__element--modifier`, e.g. `.app-list-header__tools`, `.uf-chat__bubble--mine`) and `:deep()`
  to reach Quasar internals. Dialog/menu content that Quasar portals to `body` cannot be reached from a
  scoped parent, so those blocks restate their own spacing (`.app-menu__flyout`).
- `!important` appears only to beat legacy globals (`.q-field__control { min-height: auto !important }`
  in `app.scss`) or Quasar's own `!important`s; new globals should not need it.
- Comments explain **why** a rule exists, in one or two lines, often naming the bug it fixed. Keep that
  habit; it is how the codebase stays maintainable.
- Height alignment is engineered, not eyeballed: action rows set `--dh-action-height: 36px` and
  `--dh-status-height: 24px` and size children from them; card heads that may or may not carry a button
  get `min-height: 52px` so siblings line up.
- `white-space: nowrap` plus `width: 1%` keeps an actions column on one line; everything placed in an
  actions cell must be inline-level (no `div`, no `q-separator`).
- Print: a `@media print` block hides `.no-print`, drawers, banners, breadcrumbs and buttons and widens widgets.

## 12. Legacy to leave behind

`app.scss`/`custom.scss` still carry rules from older products (`.PersonMain`, `.project6`, `.WhiteTable`,
`.GreyTable`, `.statusTable`, `.Hot-Prospect`, `.swimlane`, `.task-box`, `.PlannerDialog`, `.bg-*-4`
dropdown tints, `.Custom-Table`, `.table-sm`). The `Zw*` global components (`zw_date`, `zw_currency_input`,
`zw_numeric_input`), `dialogs/*.vue` with `assets/utils.js` (`zwConfirm`, `notifySuccess`…) and
`composables/useFilters.js` are superseded by the `App*` components, `useNotify`, `useConfirm` and
`useDateFormat`. Do not carry any of these into a new application; copy only the token file, the
typography block, the standard-typography section of `custom.scss`, the record vocabulary, and the
`App*` layer (see `assets/ui-kit/README.md`).
