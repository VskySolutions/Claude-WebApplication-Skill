# Component catalogue (`src/components/common`, `src/components/universal` entry points)

Every shared component, with its contract and the behaviour that makes it "standard". Props are listed
with defaults; anything not listed falls through to nothing (these components do **not** inherit attrs
onto the inner Quasar control: their root is a wrapper `div.app-field`, so an undeclared prop such as
`:rules` on a component that does not declare it silently does nothing).

Import components explicitly in every `<script setup>`; only `ZwDate`, `ZwCurrency`, `ZwNumeric` (legacy)
are registered globally.

## Contents
1. Field wrapper and labels
2. Text, password, select
3. Dates
4. Phone, email list, address
5. Rich text, read-only, names
6. Files and images
7. Badges, tips, audit, breadcrumbs
8. Page furniture: list header, detail header
9. Data table and filters
10. Drawers
11. Quick filters
12. Smaller shared pieces
13. Legacy components (do not reuse)

---

## 1. Field wrapper and labels

Every input is wrapped in `<div class="app-field">` with the label **outside and above** the control.

**`AppFieldLabel`** — `label`, `required` (bool), `info` (string). Renders nothing when `label` is empty.
A trailing `*` in the label text also marks it required (`useFieldLabel` strips it from the visible text
and the aria-label). `info` adds a 14px `o_info` icon with a tooltip (max 280px) that explains *why the
field offers what it offers* (scoping, source of values); it never changes the field's height.

## 2. Text, password, select

**`AppTextField`** (`q-input outlined dense hide-bottom-space`):
`modelValue` (String|Number), `label`, `required`, `type="text"` (also `email`, `number`, `textarea`,
`datetime-local`), `placeholder`, `rules: []`, `error`, `errorMessage`, `disable`, `readonly`, `hint`,
`autogrow`, `mask`, `maxlength` (declared on purpose so it reaches the input), `autofocus`, `clearable=false`,
`dense=true`, `autocomplete="off"` (browser autofill is off app-wide; pass `"on"` to opt in). Slots
`prepend`, `append`. Emits `update:modelValue`, `blur`. Use `v-model.number` for numeric fields.

**`AppPasswordField`** — wraps `AppTextField` with a show/hide eye toggle in `append` (`o_visibility` /
`o_visibility_off`, tooltip "Show password"/"Hide password"). `autocomplete="new-password"` by default.
Same props as the text field plus `maxlength`, `autofocus`; forwards `prepend`.

**`AppSelect`** (`q-select outlined dense hide-bottom-space`):
`modelValue`, `options: [{ label, value, disable?, header? }]` (bare strings also work), `label`, `required`,
`info`, `ariaLabel` (for a select without a visible label), `loading`, `clearable=true`, `multiple`,
`optionValue="value"`, `optionLabel="label"`, `emitValue=true`, `mapOptions=true`, `error`,
`errorMessage`, `rules`, `hint`, `dense=true`, `useChips` (defaults to `multiple`), `readonly`, `disable`,
`autocomplete="off"`, `useInput` (type-to-search; **switches on automatically above 10 options**),
`inputDebounce=0`. Slot `after` (an inline button beside the control, e.g. a `+` quick-add).
Behaviours worth knowing:
- Filtering is done inside the component over the options it already holds (not via QSelect's `@filter`),
  so hand it the full list. The selected option always stays visible while searching.
- A searchable single select behaves as an autocomplete (`fill-input hide-selected`); multi keeps chips.
- Grouped lists: options carrying `header: true` render as uppercase primary category labels; a heading
  only stays while something under it matches. Build them as `{ label, value: "__cat_x", disable: true, header: true }`.
- Multi-select chips: `q-chip dense color="teal-1" text-color="primary"` removable unless readonly.
- Loading shows a spinner in `append`; "No options" is the empty message.
- Control min-height 40px matches the text fields.

## 3. Dates

**`AppDateField`** (the standard): `modelValue` is an ISO calendar date `"YYYY-MM-DD"` (never a timestamp,
never time-zone shifted) while the box reads/accepts `MM/DD/YYYY` (mask `##/##/####`). Props: `label`,
`required`, `rules` (run against the ISO value), `error`, `errorMessage`, `disable`, `readonly`, `hint`,
`dense=true`, `clearable=true`, `autocomplete="off"`, `minDate`, `maxDate` (ISO; bound each other for a
Start/End pair and the calendar greys the rest out). The calendar is a `q-popup-proxy` on the 21px primary
`o_calendar_month` icon in `append` (grey and inert when readonly/disabled), `q-date minimal today-btn`
with `Clear` and `Done`; picking a day closes it. Leaving the box with a non-date restores the stored
value; an empty field opens the calendar on the month of its bound.

**`AppDatePicker`** — older read-only-input variant (icon in `prepend`, built-in "Start must be on or
before end" error). Prefer `AppDateField`.

For a date **range filter** use two `AppDateField`s with `:dense="false"`, cross-bounded, plus a
negative caption when `from > to` (see `AppColumnFilters`).

## 4. Phone, email list, address

**`AppPhoneInput`** — one label, two controls flush together: a dial-code `AppSelect` (42% / max 210px,
countries with US then India pinned, labels like "United States (+1)") and a `q-input` number box with
as-you-type formatting and the country's example number as placeholder. Props: `modelValue` (stored
value, E.164 when valid), `country` (dial code string, `v-model:country`), `label="Phone Number"`,
`required`, `countryLabel`, `dense=true`, `disable`, `readonly`, `defaultCountry="US"`. Emits
`update:modelValue`, `update:country`, `blur`, `update:valid`; exposes `validate()`. Validation is
"possible number for the country" (shape and length), with the message naming the example format.
Corner radii are squared where the two halves meet; the focused half paints its border on top.

**`AppEmailListField`** — a free-typed list of addresses (a To line): `q-select multiple use-chips
hide-dropdown-icon` with no options; Enter, comma, semicolon or blur commits an entry; pasted lists are
split; duplicates are ignored case-insensitively; an entry that does not look like an address is refused
with `"x" is not an email address.` Props: `modelValue: []`, `label`, `required`, `info`, `hint`,
`readonly`, `disable`.

**`AppAddressFields`** — THE address block, `v-model` an address object
`{ countryCode, countryName, stateCode, stateName, cityName, addressLine1, addressLine2, postalCode, [landmark, buildingName, floorNumber, unitNumber], [firstName, lastName, suffix, email, phone] }`.
Order: Country → State/Province → City → Address Line 1 → Address Line 2 → Zip Code. State and City are
selects fed by `country-state-city` and degrade to free text when the dataset has none; a stored state
name without a code is recovered. Postal codes are validated on blur per country (`validator.isPostalCode`).
Props: `disable`, `readonly`, `dense=true`, `required` (country, state, city, line 1, zip; line 2 never),
`extended` (landmark/building/floor/unit under an "Additional details" subhead), `contact`
(addressee: first name, last name, suffix, email), `contactLabel="Addressed to"`, `contactFirst`
(addressee before the place via flex `order`), `contactRequired`, `contactPhone`, `contactPhoneRequired`,
`cols` (override grid classes per field key), `gutter="md"`, `errors` (server messages keyed by field).
Exposes `validate()` for hosts that are not inside a `q-form`. Default widths: country/state/city
`col-12 col-sm-4`, line 1 `col-12 col-sm-8`, line 2 and zip `col-12 col-sm-4`, first name/email `col-12 col-sm-6`,
last name `col-8 col-sm-4`, suffix `col-4 col-sm-2`.

## 5. Rich text, read-only, names

**`AppRichTextField`** — CKEditor 5 classic with plugins Essentials, Paragraph, Bold, Italic, Underline,
Strikethrough, Link, List, BlockQuote, RemoveFormat, Undo; toolbar `undo redo | bold italic underline
strikethrough | bulletedList numberedList blockQuote | link removeFormat`, `shouldNotGroupWhenFull`,
links get `target=_blank` and default `https://`, `licenseKey: "GPL"`. Props: `modelValue` (HTML),
`label`, `required`, `placeholder`, `hint`, `readonly`, `disable`, `editorConfig` (merged). Read-only
renders the sanitised markup (`renderRichText`) instead of a disabled editor; empty shows `—`. Editable
area 8rem–22rem tall with a primary focus border; the editor font inherits the page font (`html:root
{ --ck-font-face: inherit; --ck-content-font-family: inherit }`). For table cells use `stripHtml`.

**`AppReadonlyField`** — a value nobody can type into, drawn at exactly an input's height (40px, `#f7f9fc`,
8px radius): `modelValue`, `label`, `placeholder="—"`, `hint` (shown on an `o_info` icon **inside** the
control, never as a caption line), `hintAlert` (switches the icon to `o_error_outline` in `warning`). Default
slot for marked-up values.

**`AppNameSuffixField`** — the generational suffix box (Jr., Sr., II, III, IV) as an `AppTextField` with a
dropdown button offering suggestions plus "No suffix"; typed values allowed; max 16 characters with its
own message. Props: `label="Suffix"`, `placeholder="Jr."`, `readonly`, `disable`, `error`, `errorMessage`.
Emits `blur` on a picked suggestion too (auto-save hosts rely on it).

**`AppNameWithSuffix`** — how a name with a suffix is drawn anywhere (tables, headings, dialogs): the
name then the particle in **bold**; strips a particle the caller already joined; `empty="—"` when there
is no name. Props: `name`, `suffix`, `empty`.

## 6. Files and images

Shared limits live in `composables/useFileDrop.js`: `MAX_UPLOAD_MB = 10`, `MAX_ATTACHMENT_FILES = 15`,
`MAX_ATTACHMENT_TOTAL_MB = 100`, `ATTACHMENT_ACCEPT` (pdf, images, Office, text/csv/rtf/md/json/xml,
msg/eml, zip) and the sentence `ATTACHMENT_HINT`. Every picker states its limit on the control.

**`AppSingleFileUpload`** / **`AppMultiFileUpload`** — a click-or-drop zone (`role="button"`, Enter/Space,
`o_cloud_upload` 30px, "**Click to upload** or drag & drop", hint line, spinner while `loading`), with
validation of extension and size (`validateFiles`) and an error line in red under the zone. Props:
`modelValue` (File | File[]), `label`, `required`, `accept` (comma-separated extensions), `maxSizeMb`,
`maxFiles` (multi), `hint`, `loading`, `disable`, `errorMessage`. Emit `update:modelValue`, `rejected`.
Picked files render as `AppFilePreviewItem` rows.

**`AppFilePreviewItem`** — a staged `File`: thumbnail (object URL) for images or a type icon, name,
`EXT · size`, an `o_close` remove button; click opens it in a new tab.

**`AppStoredFileItem`** — a file already saved on the server (any shape: `mediaId|id`,
`fileName|originalFileName|name`, `fileExtension`, `mimeType`, `fileSize|size`): lazy thumbnail via
`IntersectionObserver` for images, type icon otherwise, name, `PDF · 1.2 MB`, an `o_open_in_new` hint icon,
optional `removable` ✕ (negative). Click fetches the bytes through the authenticated client and opens a new
tab (`openStoredFile`; falls back to download); `fetchBlob` prop overrides the source. Errors toast.

**`AppImageUpload`** — avatar-style picker with crop dialog (`vue-advanced-cropper`, `aspectRatio=1`,
`size="96px"`, `accept="image/*"`, `maxSizeMb=10`, `buttonLabel="Upload"`, `placeholderIcon="o_person"`,
`hint` default "Any image, up to 10 MB", `loading`, `disable`, `fileName="image.png"`). Emits
`update:modelValue` (preview URL), `crop` (a PNG `File` — the host uploads it and calls `closeCrop()`),
`remove`. Refuses non-images and cropped output over the limit with a specific message.

## 7. Badges, tips, audit, breadcrumbs

**`AppOptionBadge`** — THE badge for an option-set value: `option: { label, description?, backgroundColor?,
textColor?, icon? }` → `q-badge` in the data's colours (grey `#9e9e9e` fallback), icon 13px before the
label, the description as a 300ms-delayed tooltip, default slot for a trailing control (chevron when the
badge is also a menu trigger).

**`AppInfoTip`** — an `o_info` icon (16px, `grey-6`, `cursor: help`) carrying a `text-body2` tooltip
(max 320px); default slot for marked-up notes. Props `text`, `icon`, `size`, `color`, `maxWidth`, `anchor`,
`self`. Used beside section titles and in `AppListHeader`'s `note` slot (18px primary, below-left).

**`AppAutoSaveState`** — the auto-save indicator: `state` `idle | saving | saved | error`, `message`.
Spinner + "Saving…", green `o_check_circle` + "Saved", red `o_error_outline` + the API message.

**`AppRecordAudit`** — the provenance card at the foot of every detail page: `audit: { createdBy,
createdOnUtc, updatedBy, updatedOnUtc, deleted?, deletedBy?, deletedOnUtc? }`, `title="Record information"`.
Auto-fit grid (`minmax(200px,1fr)`), uppercase 11px labels, 14.5px values, `Unknown` for a missing actor
(the server sends "System" for platform writes).

**`AppBreadcrumbs`** — `items: [{ label, icon?, to? }]`, `noMargin`; `text-grey-7`, active colour primary.
First crumb is always `{ label: "Home", icon: "o_home", to: "/" }`.

## 8. Page furniture: list header, detail header

**`AppListHeader`** — the carded header of every list/settings page (12px radius, `q-mb-md`): crumbs
left, tools right. Props: `breadcrumbs`, `search` + `update:search` (debounce 300, `o_search` prepend,
placeholder via `searchPlaceholder`), `showSearch`, `showFilters` (+ `filterCount` → floating primary
badge on the outlined "Filters" button), `showAdd` + `addLabel="Add"` + `addIcon="o_add"` + `addDisable`
(filled primary button), `showBack` (flat "Back" with `o_arrow_back`). Slots: `actions` (extra controls
placed before Add: a secondary outlined button, a scope select, a date-range dropdown), `note` (what the
page is for, shown as an `AppInfoTip` beside the crumbs). Emits `filters`, `add`, `back`. Wraps on phones
(tools take their own line, search flexes).

**`AppDetailHeader`** — the carded header of every detail page: crumbs left, `actions` slot + an outlined
primary "Back" (`o_arrow_back`) right. Props: `items` (crumbs), `backTo` (fallback destination; Back
goes to the previous history entry when there is one). Inside the actions row every `q-btn` is normalised
to 36px and every `q-chip`/non-floating `q-badge` becomes a 24px pill (999px radius, 11.5px/600) so
"things to press" and "things to read" are visibly different kinds. The row wraps as a group.

## 9. Data table and filters

**`AppDataTable`** — `q-table flat bordered` with the platform's behaviours. Props: `rows`, `columns`,
`rowKey="id"`, `loading`, `title`, `pageKey` (enables cookie persistence), `totalRecords`, `selectable`,
`pagination` (object the page owns), `defaultSortBy="updatedOnUtc"`, `defaultDescending=true`,
`clientSort` (sort in memory with a type-aware comparator: booleans, numbers, ISO instants, natural text),
`pinnedRowKeys` (float to the top of the page), `rowColours` (`{ rowKey: "#hex" }` → 4px left stripe).
Emits `request` (page/size/sort changed → the server), `refresh`, `update:pagination`, `update:selected`.
Slots: `actions` (buttons in the top bar beside Columns/Refresh), `quick-filters` (a row under the bar),
`bulk-actions` (with `selected`, shown with an "n selected" chip), `no-data`, and every QTable body slot
(`body-cell-<name>`) forwarded.
Built in:
- Top bar: title (`text-subtitle1 text-weight-medium`), spacer, actions, **Columns** menu (`o_view_column`:
  checkbox per data column, drag to reorder, "Reset columns"), **Refresh** (`o_refresh`, spins with `loading`).
- Column visibility: only columns flagged `default: true` show initially (all, if none flagged); the
  `actions` column is always shown and always last.
- Persisted per `pageKey`: `visibleColumns`, `columnOrder`, `columnWidths`, `pageSize` (plus `sortBy`,
  `descending` via `useListTable`).
- Resizable headers (drag handle at the right edge of each `q-th`, min 80px), sticky header row.
- Rows per page `10, 20, 50, 100`; a new sort goes back to page 1.
- Empty state: `o_inbox` 32px + "No Data Available", overridable.
- Actions column never wraps (`white-space: nowrap; width: 1%`); children must be inline-level.
- Selection column left-aligned and 1% wide.

Column object contract (see `list-pages.md` for the full conventions):
`{ name, label, field: string | (row) => value, align: "left", sortable, default, filterable, filterOptions, filterMultiple, filterType: "dateRange", filterMatch: "includes", sort: (row) => key }`.

**`AppColumnFilters`** — `v-model` the reactive `filters` object from `useColumnFilters`, `columns` the
filterable list. Renders per column: a select (`filterOptions`, `multiple` when `filterMultiple`), a From/To
`AppDateField` pair (`filterType: "dateRange"`, with an invalid-range caption), or a clearable text
"contains" box (`:dense="false"`). Spaced with `gap: 8px`.

## 10. Drawers

All right-side drawers are `q-drawer side="right" overlay bordered class="column no-wrap"`, header
`row items-center q-pa-md bg-primary text-white` with `text-h6` title and a white round `o_close`,
`q-separator`, a `q-scroll-area class="col"` body, and (for forms) a `q-separator` + footer
`row justify-end q-gutter-sm q-pa-md bg-grey-1`.

**`AppFormDrawer`** — `modelValue`, `title`, `saveLabel="Save"`, `saving` (sets `:loading`/`:disable` on
Save and blocks double submit), `hideSave` (read-only viewers), `draftKey` + `draft` (auto-persists the
draft to the page cookie 2s after each change, flags the tenant store's unsaved-form guard, emits
`restore-draft` with the saved draft when reopened, clears on cancel/submit). Footer: Cancel (`flat
no-caps color="grey-8"`), slot `footer-actions`, Save (`unelevated no-caps color="primary"`). Emits
`submit({ clearDraft })`, `cancel` (on any close: X, backdrop, Esc, Cancel), `update:modelValue`. Width
is viewport-relative (50% default, 30–95%), drag the left edge (6px handle, double-click resets),
persisted in LocalStorage until logout. Exposes `clearDraft()`.

**`AppFilterDrawer`** — `modelValue`, `chips: [{ key, label }]`, `showChips=true`. Renders the active-filter
chip row above the table (`teal-1`/primary removable chips + "Clear all") unless the page has a quick-filter
bar, and the drawer titled "Filters" with a `q-pa-md column q-gutter-sm` body (default slot) and a footer
with "Clear all" (flat grey-8) and "Done" (primary). Emits `remove(key)`, `clear`. Width 360px default,
drag-resizable (300 → 60% of the viewport, 92% on phones).

**`AppViewDrawer`** — a read-only side panel: `title="Details"`, `width=420`, `fields: [{ label, value }]`
rendered as 5/7 label-value rows (`—` for empty), plus a default slot.

## 11. Quick filters

**`QuickFilterBar`** — the strip under the table's top bar (`#quick-filters` slot): groups left, "n filters
applied" + "Clear all" (`o_filter_alt_off`, disabled at zero) pinned right. `activeCount` must equal the
drawer's chip count so the two agree. Groups are separated by a 2px rule. Background `#f7f9fb`, 1px `--line`.

**`QuickFilterGroup`** — one labelled `q-btn-toggle` (`no-caps unelevated dense`, `toggle-color="primary"
color="white" text-color="grey-9"`, white bordered buttons, 8px radius) whose `modelValue` **is** the
drawer filter's value, so a click here and a pick in the drawer are the same filter. Props: `label`
(uppercase, with the red star when `clearable=false` because one side is always on), `options:
[{ value, label }]`, `counts: { [value]: n }` (a rounded tally badge on each button, inverted on the
chosen one), `clearable=true` (clicking the active button again clears; the chosen button also shows a
small ✕). Counts come from a `quick-counts` endpoint called alongside every list load with the same filters.

## 12. Smaller shared pieces

- **`AppTenantScopeSelect`** — the super-admin "View as" dropdown described in `app-shell.md`.
- **`temp_password_dialog.vue`** — a persistent dialog showing a one-time secret: title with `o_key`,
  "This password will not be shown again…", a read-only input with an `o_content_copy` button (toast on
  copy, warning when the clipboard is blocked), "Done".
- **`PersonFormFields` / `PersonFormDialog`** — the pattern "a field set defined once": a `defineModel`
  component laying out the fields in sub-headed blocks, reused by a drawer and by a quick-add dialog
  (`+` beside a select's `after` slot, 640px dialog with primary header).
- **`ChangePasswordForm`** — three `AppPasswordField`s with `prepend` icons, a `bg-grey-2` requirements
  banner (`o_info`, bulleted rules), footer with optional Cancel and the primary submit.
- Universal-feature components (`EntityUniversalPanel`, `EntityHeaderActions`, `EntityRowActionsMenu`,
  `EntityRowMarks`, `EntityPinnedMark`, `FieldLogIcon`, `DeletedRecordsPanel`, `NotificationCentre`,
  `StickyNoteLayer`…) are documented in `universal-features.md`.
- Dashboard pieces (`DashboardWidgetWrapper`, `BarChart`, `DonutChart`, `LineChart`) in `dashboard.md`.
- `DetailGrid` and `RelatedStatusSelect` (badge-as-dropdown) live under a feature module but are generic;
  see `detail-and-forms.md`.

## 13. Legacy components (do not reuse)

`components/zw_date.vue`, `zw_currency_input.vue`, `zw_numeric_input.vue` (globals `ZwDate`, `ZwCurrency`,
`ZwNumeric`, label inside the box, `stack-label`), `dialogs/alert.vue`, `confirmation.vue`,
`delete_confirmation.vue`, `module_confirmation.vue`, `update_confirmation.vue` and the `zw*` helpers in
`assets/utils.js`. New code uses `App*` fields, `useNotify`, `useConfirm` and bespoke `q-dialog`s.
