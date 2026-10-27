# Detail pages, forms, drawers and dialogs

How a record is shown, edited and saved. Patterns are taken from the user, role, tenant, person,
permission-group, option-set, profile and request pages and their drawers/dialogs.

## Contents
1. Detail page anatomy
2. Card grammar
3. Read-only rendering (the record vocabulary)
4. Saving: explicit save, per-card auto-save, multi-part auto-save
5. Form drawers (create / edit)
6. Dialogs
7. Confirmations and leave guards
8. Validation and error mapping
9. Field layout and specific field patterns
10. Repeatable rows, toggled sections, choice chips
11. Tabs, split panes, sticky action bars
12. Files on forms
13. Public (anonymous) forms

---

## 1. Detail page anatomy

```vue
<q-page padding>
  <app-detail-header :items="[{ label: 'Home', icon: 'o_home', to: '/' }, { label: 'Users', to: { name: 'users' } }, { label: user?.displayName || 'User' }]"
                     :back-to="{ name: 'users' }">
    <template #actions>
      <!-- status badges / option badges (become pills), then buttons -->
    </template>
  </app-detail-header>

  <div v-if="loading" class="row flex-center q-pa-xl"><q-spinner color="primary" size="40px" /></div>

  <div v-else-if="record">
    <!-- 0..n banners: read-only notice, warning, call-to-action -->
    <!-- identity / summary card -->
    <!-- row q-col-gutter-md: col-12 col-md-7 (who/what) + col-12 col-md-5 (relationships, side panels) -->
    <app-record-audit :audit="record.audit" />
  </div>

  <!-- drawers and dialogs owned by the page -->
</q-page>
```

- The last crumb is the record's own name with a fallback (`role?.name || 'Role'`).
- **Every action on the record lives in the header's `actions` slot** (status first, then buttons: filled
  primary for the main move, outlined primary for secondary reads such as Conversation / Email log, round
  outlined icon for Activity, coloured buttons for workflow moves). There is no bottom action bar on detail pages.
- `loading.value = !record.value` on reload so the spinner shows only on the first load.
- A load failure toasts and may `router.replace` back to the list.
- Banners after the header: `bg-blue-grey-1 text-blue-grey-9` + `o_lock` for "read-only for you, and why";
  `bg-orange-1 text-orange-9` + `o_warning` for warnings ("This is the user's only tenant assignment.");
  `bg-teal-1 text-blue-9/text-primary` with an action button for a next step ("Convert to User").
- **Identity card**: `q-card-section class="row items-center q-col-gutter-md"` → `div.col-auto` with a 72px
  `q-avatar` (image or initials, `primary` when active / `grey-5` when inactive, or an icon) → `div.col`
  (`min-width: 0`) with `row items-center q-gutter-sm`: `text-h6 text-weight-bold ellipsis` name + status
  badges; `text-body2 text-grey-7` email/phone lines; a `q-mt-sm` chips row (department chip
  `blue-grey-1`, role chips coloured by category with tooltips; "No roles assigned" caption) → `div.col-12
  col-sm-auto column q-gutter-sm` with the primary actions (Deactivate/Activate filled, Reset password
  outlined, a flat link to a related record). Summary chips with counts ("3 permissions", "2 users",
  "Updated <date>" caption).
- Right column cards hold relationships (tenants & roles list, groups, delegates); left column holds the
  record's own fields.
- Tabs on a detail page: `q-tabs dense no-caps align="left" active-color="primary" indicator-color="primary"
  class="text-grey-8 q-mb-md"` above the content; heavy tabs mount lazily and stay mounted (`v-if` once
  opened + `v-show`) so drafts survive switching.

## 2. Card grammar

```vue
<q-card flat bordered class="user-card q-mb-md">            <!-- .user-card { border-radius: 12px } -->
  <q-card-section class="row items-center q-gutter-sm">
    <q-icon name="o_badge" color="primary" size="sm" />
    <div class="text-subtitle1 text-weight-medium">Basic information</div>
    <app-info-tip text="Saves as you leave each field…" />
    <q-space />
    <app-auto-save-state :state="save.state" :message="save.message" />   <!-- or a dense Add/Manage button -->
  </q-card-section>
  <q-separator />
  <q-card-section class="row q-col-gutter-md"> <!-- fields --> </q-card-section>
  <!-- or: <q-list separator> rows </q-list> for relationship cards; <q-card-actions align="right"> for a Save -->
</q-card>
```

- Simple heading variant: `q-card-section class="text-subtitle1 text-weight-medium"` (theme blue by the
  global rule) followed by `q-separator`.
- Compact heading variant for nested/secondary cards: `q-card-section class="q-py-sm text-subtitle2 text-primary"`
  with an 18px icon.
- Relationship lists: `q-list separator` of `q-item`s: avatar icon `grey-7`, `q-item-label` (bold when it is
  a name) with inline `q-badge`s ("Inactive", "You"), caption with context (`email · also Partner`), and a
  `side` section holding `flat round dense` icon buttons (`primary` manage, `negative` remove) with tooltips.
  Empty list → a `q-item` with `text-grey-6` text. Row highlight for "needs you": `background #fff8e1` + an
  inset left bar (`box-shadow: inset 3px 0 0 #ff8f00`; the reader's own row gets the primary colour, 4px).
- Danger zone: `q-card` with `border-color: var(--q-negative)`, `text-negative` title, explanatory caption
  left, `outline color="negative"` button right; an inline `bg-red-1 text-negative` banner for its error.
- Side-by-side cards of equal height: `full-height` on the card inside equal columns.

## 3. Read-only rendering (the record vocabulary)

Never show a disabled input to display a value. Use:

- **`DetailGrid`** (`rows: [{ label, value, wide?, html?, hideWhenEmpty?, suffix? }]`): a
  `repeat(auto-fit, minmax(220px, 1fr))` grid of label (11px uppercase `--ink-500`) over value (14.5px
  `--ink-900`); `—` for empty; booleans as Yes/No; arrays joined with `, `; `html` rows through
  `renderRichText`; `suffix` rows through `AppNameWithSuffix`; `wide` spans the grid.
- The same vocabulary by hand: `div.rems-label` + `div.rems-value` inside `col-12 col-sm-6` cells of a
  `row q-col-gutter-md`; `div.rems-subhead` over a group; `q-separator class="q-my-md"` between groups
  inside one card; `q-card flat bordered class="rems-inner q-mt-md"` for a conditional sub-block.
- A value with an explanation: `q-icon name="o_info" size="14px" class="rems-value__info"` (`--ink-300`,
  `cursor: help`) after it, with a 300ms-delayed tooltip.
- **Review / submitted-form summaries**: groups with a title row (13px / 600 / uppercase / primary / icon,
  `border-bottom: 1px solid #e0e6ed`), a field grid `minmax(190px, 1fr)` (narrow panes), and "people cards"
  (`minmax(220px, 1fr)` grid of bordered 8px-radius `#fbfcfd` cards: role label, name, caption with email ·
  phone). Empty groups say so in `text-grey-6`; sections that do not apply are omitted entirely.
- Monospace for identifiers (`font-family: ui-monospace, Consolas, monospace`, small, bordered chip).
- Pre-formatted user text keeps its line breaks (`white-space: pre-wrap`).

## 4. Saving

Three models, chosen by how consequential the write is:

1. **Explicit Save** (profile, tenant name, branding, option-set form): `row justify-end q-mb-md` with
   `q-btn unelevated no-caps color="primary" label="Save" :loading="saving"` (disabled while not dirty when
   a dirty check is cheap: `:disable="name === tenant.name"`). Writes that affect other records or other
   tenants stay behind Save on purpose.
2. **Per-card auto-save** (user and role detail): text fields commit on `@blur`, pickers/toggles on
   `@update:model-value`, rich text 1200ms after typing stops; each card shows `AppAutoSaveState` in its
   header and an `AppInfoTip` saying "Saves as you leave each field". The helper:

   ```js
   const runAutoSave = async (target, fn) => {
     target.state = "saving"; target.message = "";
     try { await fn(); target.state = "saved"; setTimeout(() => { if (target.state === "saved") target.state = "idle"; }, 2500); return true; }
     catch (err) { target.state = "error"; target.message = getApiErrorMessage(err); return false; }
   };
   ```
   Only changed fields are sent (`if (value === original) return`); a failed save puts the field back to
   the value in force; a consequential change (changing a sign-in username, taking a headship from someone)
   asks `confirm()` first and reverts on cancel; something that affected another person also toasts
   (`notify.info("X is no longer the department head.")`).
3. **Multi-part auto-save** (`useAutoSave`) for a page made of independently written sections: each section
   calls `mark("section")` on change; the page shows one indicator (a 24px dot in the header with the state
   on its tooltip) and a "blocked" reason when a section cannot be saved yet.

Save feedback: toast `"<Thing> updated."` / `"<Thing> saved."` for explicit saves; no toast for auto-saves
(the indicator is the feedback).

## 5. Form drawers (create / edit)

```vue
<app-form-drawer v-model="formOpen" :title="editing ? 'Edit Tenant' : 'Create Tenant'" :saving="saving"
                 save-label="Save" @submit="submitForm" @cancel="resetForm">
  <q-form ref="formRef" greedy>
    <app-text-field v-model="form.name" label="Name" required class="q-mb-md" :rules="[(v) => !!v || 'Name is required']" />
    <app-text-field v-model="form.identifier" label="Identifier" required :disable="editing" hint="Lowercase letters, numbers and hyphens"
                    :error="!!identifierError" :error-message="identifierError" :rules="editing ? [] : [...]" />
    <app-select v-model="form.timeZoneId" label="Time Zone" required use-input :clearable="false" :options="allZones" class="q-mt-md" />
  </q-form>
</app-form-drawer>
```

```js
const submitForm = async ({ clearDraft } = {}) => {
  identifierError.value = "";
  if (!(await formRef.value?.validate())) return;
  saving.value = true;
  try {
    editing.value ? await api.update(id, payload) : await api.create(payload);
    notify.success(editing.value ? "Tenant updated." : "Tenant created.");
    clearDraft?.(); formOpen.value = false; resetForm(); load();
  } catch (err) {
    if (getApiErrorCode(err) === ApiErrorCodes.DuplicateIdentifier) identifierError.value = "This identifier is already in use.";
    else notify.error(getApiErrorMessage(err));
  } finally { saving.value = false; }
};
```

- Fields stack with `class="q-mb-md"` (or wrap them in `q-form class="column q-gutter-md"`); two short
  fields share a `row q-col-gutter-md` with `col-12 col-sm-6` wrappers.
- Immutable identifiers are `:disable`d (or `readonly`) on edit, with the create-only rules dropped.
- Open resets the form (`watch(open)` or `openCreate()` → `resetForm()` first), then loads lookups
  (`await loadTenants()`), then opens; edit copies the row into the form before opening.
- Standalone drawer components (`UserCreateDrawer`, `SmtpAccountFormDrawer`, `OptionSetFormDrawer`) take
  `v-model`, an id/row prop (null = create), scope props, and emit `saved`/`created`; they own their lookups
  and load them `watch(() => props.modelValue, …, { immediate: true })` when opened.
- Draft persistence (`draft-key` + `:draft`) for long forms; the draft view excludes secrets
  (`const { password, ...rest } = form`).
- Secrets on edit: never pre-filled, hint "Leave blank to keep existing password", only sent when typed.
- After a create that starts a longer setup, route to the new record's page instead of reloading the list.
- Follow-up outcomes after a create are separate toasts: `info` for side effects, `warning` for a partial
  failure ("User created, but the invitation email could not be sent…"), and a one-time secret goes into
  `temp_password_dialog`.

## 6. Dialogs

Use a `q-dialog` for a short, focused decision or a self-contained sub-flow; use a drawer for a form that
edits a record; use a route for a detail.

Standard card (`q-card style="min-width: 420px; max-width: 92vw"`):
- Header: `q-card-section class="text-h6"` ("Add users to this role"), or the drawer-style
  `row items-center bg-primary text-white` with an icon, `text-h6 q-ml-sm` title, `q-space`, white `o_close`.
  Sub-flows name their record in a `text-caption text-grey-7` subtitle ("REMS-000123 — Acme").
- `q-separator`, body `q-card-section` (`column q-gutter-md` for fields), `q-separator`,
  `q-card-actions align="right"` (optionally `bg-grey-1`): Cancel `flat no-caps color="grey-8"` (or
  `v-close-popup`) then the primary `unelevated no-caps color="primary"` with `:loading` and `:disable`
  until valid; destructive primaries are `color="negative"`; workflow moves may use `orange-9`/`amber-8`/`teal-7`.
- `persistent` for dialogs holding typed input; plain for viewers.
- Sizes used: 320–380px (confirmations with a reason box, export options), 420–480px (pick users, manage
  groups, delegate), 560–640px (previews, logs, activity, conversation `height: 76vh`), `70vw` (compose
  email, with `max-width: 70vw` to beat Quasar's 560px cap; 92vw below 1024px), `96vw × 92vh` (full
  editor; `:maximized="$q.screen.lt.md"`). Bespoke dialog cards have `border-radius: 12px`.
- Multi-state dialogs use a `phase` string (`loading → preview → sending → sent | error`) and render one
  section per phase; the success phase is a centred block with a 48px positive icon, a `text-subtitle1`
  headline, `text-grey-7` detail, and a single "Done" action.
- Two-step destructive confirmation: explain the consequence, "Type **DELETE** to confirm" with an
  `autofocus` input, primary `negative` button disabled until the text matches.
- Dialogs owned by a menu item live as **siblings** of the menu (not inside it) so they survive its closing.
- Reset the dialog's state in `watch(open)` when it opens; keep the previous content while it animates shut.
- Scrollable body with fixed head/actions: card `display: flex; flex-direction: column`, body `flex: 1 1 auto;
  overflow-y: auto` (or `max-height: 62vh; overflow: auto` on the body section).

## 7. Confirmations and leave guards

- `const ok = await confirm({ title, message, confirmLabel, cancelLabel, type })`; return early on `false`.
- Confirm: deletes and permanent removals (`type: "danger"`), deactivations, resets, sending something to
  people, locking a list, switching off something tenant-wide, clearing a group of typed rows, discarding a
  dirty dialog ("Discard these corrections?" with "Keep editing"), leaving a dirty editor
  (`onBeforeRouteLeave` → confirm "Leave without saving…?", `type: "danger"`), tenant switch with unsaved form.
- Do not confirm: adding to a list, toggling a preference, saving, removing one of several visible rows
  ("the cards around it are still on screen to make the mistake obvious"), posting a chat message.
- Message style: name the thing in quotes or by name, say the consequence in one sentence, end with
  "Continue?" when the action is not destructive. Button labels are verbs ("Delete", "Archive", "Make head",
  "Turn off", "Send", "Pick up"), never "OK/Yes".

## 8. Validation and error mapping

- `q-form ref="formRef" greedy` wraps the fields; `const ok = await formRef.value?.validate()`; rules are
  arrays of `(v) => condition || "Message"`; shared rule builders: `nameRules(label, { required })`,
  `requiredRule(message)`, a local `httpsRule`, `limitRule`. Email: `/.+@.+\..+/` or `/^[^@\s]+@[^@\s]+\.[^@\s]+$/`.
- Field sets without a `q-form` expose `validate()` (`AppAddressFields`, `AppPhoneInput`) and the host calls
  them at submit; a `q-form` host gets the same coverage from `:rules`.
- Auth forms use vuelidate: `useVuelidate(rules, model, { $lazy: true, $autoDirty: true })`,
  `:error="v$.email.$error" :error-message="v$.email.$errors[0]?.$message" @blur="v$.email.$touch"`,
  `helpers.withMessage(...)`, `await v$.value.$validate()`.
- Server errors: `getApiErrorCode(err)` → a specific field message for duplicates
  (`DUPLICATE_IDENTIFIER` → `identifierError = "This identifier is already in use."`), a specific toast for
  known codes (`PERMISSION_CEILING_EXCEEDED`, `CAPACITY_BELOW_USAGE`), else `notify.error(getApiErrorMessage(err))`.
  `VALIDATION_FAILED` details (`"field: message; field2: message"`) are parsed into a per-field `errors` map
  (bound as `:error="!!errors.x" :error-message="errors.x"`) plus a red summary banner "Please fix the
  following:" with a bullet list, and a toast "Please fix the highlighted fields."; typing clears stale
  server highlights.
- Required marking: `required` prop or a trailing `*` in the label (both render the red star). A whole
  required address block passes `required` down. Field-level `hint` is the helper line under the control;
  `info` is the tooltip beside the label for *why the options are what they are*.
- Placeholders are examples ("https://host/maconomy-api", "e.g. Finance Team"), never labels.
- Lock icons (`o_lock` 18px `grey-6` in `append`) with a tooltip explain a read-only value ("Locked — the
  intake form was sent to this address").
- `maxlength` is set where the column has a length; `type="number"` fields hide the spin buttons globally.

## 9. Field layout and specific field patterns

- Grid: `div.row.q-col-gutter-md` → fields with `col-12 col-sm-6` (pairs), `col-12 col-sm-4` (triplets),
  `col-12 col-sm-8` + `col-12 col-sm-4`; `col-12` for a long single field (email). Sub-blocks start with
  `div.col-12.section-subhead` ("Name", "Demographics", "Contact").
- Name block: First `col-12 col-sm-6`, Middle `col-12 col-sm-6`, Last `col-8 col-sm-4`, Suffix `col-4 col-sm-2`
  (`AppNameSuffixField`), Preferred/Display `col-12 col-sm-6`. Names use `nameRules`.
- Phone: `AppPhoneInput` with `v-model` + `v-model:country`; put it in its own `div.col-12.col-sm-6` when a
  server error line follows it (`div.text-negative.text-caption.q-mt-xs`).
- Money/percent: `app-text-field type="number"` with `<template #prepend><span class="text-grey-7">$</span></template>`
  or `#append` `%`; compact 32px variants inside rate tables.
- Dates: `AppDateField` pairs bounded with `:max-date`/`:min-date` ("each pair has to run forwards");
  `datetime-local` through `app-text-field type="datetime-local"` for reminders.
- Toggles: `q-toggle` with a `div.text-caption.text-grey-7` note under it (`q-ml-lg` when aligned under the
  label); `color="primary"`.
- Checkbox grids: `q-checkbox v-for … class="col-12 col-sm-6 col-md-4"` inside `row q-col-gutter-x-md`; a
  checkbox beside an input sits in `div.col-12.col-sm-6.column.justify-center`.
- Searchable pickers that query a server as you type are built on `q-select` directly with `@filter`,
  `fill-input hide-selected`, 300ms debounce, a two-line `#option` (`q-item-label` + caption), and a
  `#no-option` that says why it is empty ("Type at least 2 characters…", "No customers match.", or the
  error in red). Keep the chosen option in the list.
- Autocomplete over two sources (portal records + external system): a `q-input` with `q-menu fit no-focus
  no-refocus no-parent-event`, keyboard up/down/enter/esc, `@mousedown.prevent` on rows, each row tagged
  with its source (`.cif-source` pill) and type badge, a "Searching…" row, an "unavailable" row that is a
  failure not "no match", and a "More match than are shown — keep typing" row.
- Read-only derived values use `AppReadonlyField` with `hint`/`hintAlert` ("Department Director" resolved
  from the department).
- "Copy from X" buttons (`flat dense no-caps size="sm" color="primary" icon="o_content_copy"`) sit on the
  heading row of the block they fill, right-aligned.
- Email composition: `AppEmailListField` (To) + `AppTextField` (Subject) + `AppRichTextField` (Message) in a
  `column` with `gap: 14px`, a caption explaining what is pre-filled, and a monospace link box with a copy button.

## 10. Repeatable rows, toggled sections, choice chips

- **Repeatable cards**: `q-card flat bordered class="block"` (10px radius, `#fbfcfe`), header
  `row items-center no-wrap q-pb-none` with `text-subtitle2 text-weight-medium col ellipsis` title (name the
  item by its content once typed, by position until then) and a `flat round dense color="negative" icon="o_delete"`
  with tooltip and `aria-label`; body `row q-col-gutter-sm`; under the list an `outline no-caps color="primary"
  icon="o_add" label="Add another …"` button disabled at the cap with a tooltip stating the cap.
- **Yes/No gate** over a section: a `q-btn-toggle` (`no-caps unelevated dense toggle-color="primary"
  color="grey-3" text-color="grey-8"`, options Yes/No) beside a 13px/500 question; "Yes" opens one empty
  row, "No" clears the rows and asks first when any carry data (the **host** owns the confirm dialog;
  the field set emits `confirm-clear(applyFn)`).
- **Rule-locked options** inside a `q-btn-toggle`: disabled rather than removed, with an 11px note under
  the control saying why ("A minor child is billed to the primary client.").
- **Chip radios** for a short classification: `button.rf-chip[role=radio]` — white with `--line` border,
  13px, 8px radius; `--on` fills `--teal-900` with white text (hover `--teal-800`); disabled chips are muted
  `--ink-300` with `cursor: not-allowed` and the reason under the row; optional `o_info` with tooltip inside
  the chip. **Tile choices** (`.sbd-choice`, flex `1 1 200px`, icon + name + role caption) for picking a person.
- Chip multi-select over grouped options (marketing tags): selected items as removable `teal-1` chips in a
  dashed box, a search field, then groups (`.rems-group__title` uppercase primary) of clickable chips
  (`primary` white when selected with `o_check`, `grey-3` otherwise).
- Allocation lists (commission splits): `q-list bordered separator` rows with a truncating name, a 120px
  numeric box with `%` suffix and a remove button; one feedback line for the total coloured by state
  (`ok` green `#e8f5e9/#1b5e20`, `warn` amber, `bad` red) and a bordered count pill `n / 10` that turns
  amber when full.

## 11. Tabs, split panes, sticky action bars

- **Form tabs**: `q-card flat bordered` → `div.rf-tabbar` (flex) with `q-tabs dense align="left" no-caps
  inline-label class="text-grey-7 col"` (icons + labels, `:disable` on tabs not yet reachable) and an end
  group (`flex: 0 0 auto`) holding an `o_info` note icon with tooltip (why tabs are greyed), `Prev` (flat
  dense), `Next` (filled dense `icon-right="o_chevron_right"`, outlined on the finish tab), and the finish
  action (`teal-7`). `q-separator`, then `q-tab-panels keep-alive animated`. On phones the end group wraps
  under the tabs with a top border.
- **Review-only tabs** (approval packet): same tabs over read-only `rems-label/rems-value` grids with
  sub-cards per conditional section.
- **Split pane** (reading pane beside a form): `div.rf-work.rf-work--split` flex row; left pane
  `flex: 0 0 auto` with inline `flex-basis: <pct>%` (default 40, min/max clamped), right pane `flex: 1 1 0`;
  a 12px `role="separator"` gutter with a 2px grip that grows and turns primary on hover/focus; drag with
  `pointerdown` + `body.rf-dragging { cursor: col-resize; user-select: none }`; arrow keys step, Home resets,
  double-click resets; below 1024px the panes stack (reading pane first) and the gutter hides. `min-width: 0`
  on panes so wide content does not defeat the drag.
- **Sticky action bar** (long anonymous form): `position: sticky; bottom: 0`, background `#f4f6fb`, top
  border, flex with Cancel (flat grey-8) left, a save-state line centred (`o_cloud_queue` / spinner /
  `o_cloud_done` positive / `o_cloud_off` negative: "Your progress saves automatically" / "Saving…" /
  "Saved just now" / "Couldn't save — we'll retry as you edit"), the primary (`icon-right="o_arrow_forward"`)
  right with a tooltip when disabled; stacks on phones.
- A "to-do before you continue" card (`#f7f9fc`, `o_checklist` primary, bold line, bullet list) lists what is
  still required; the continue button stays disabled until it empties.

## 12. Files on forms

- Attachments: `AppMultiFileUpload` with the shared `ATTACHMENT_ACCEPT`, `MAX_ATTACHMENT_FILES`,
  `MAX_UPLOAD_MB` and `ATTACHMENT_HINT`; staged files appear as `AppFilePreviewItem` rows; an "Upload n
  files" primary button appears when something is staged (panels) or the files go up with Save (forms).
- Already-saved files render as `AppStoredFileItem` rows under an "Attached" label; `removable` when the
  form is editable, with a confirm and a per-row `disable` while removing.
- Single required document (a signed form): a status banner first (green "on file" / orange "required" /
  teal "attached when you save" / teal "saving replaces…"), then the picker (`accept=".pdf"`, hint "PDF only,
  up to 10 MB"), then the stored row at the bottom, in both states.
- Image avatars: `AppImageUpload` → `@crop` uploads, then `closeCrop()`; `@remove` clears the media id.

## 13. Public (anonymous) forms

- Load → one of four states: unavailable (neutral card `o_link_off` 52px grey, "This form isn't available",
  discloses nothing), cancelled ("Your progress is saved" with `o_bookmark_border`), submitted (thank-you
  card `o_check_circle` 56px positive, personalised headline, "You can safely close this window."), editable.
- Editable: intro (`text-h5 text-weight-bold` + `text-body2 text-grey-7` "Your progress saves automatically"),
  a red summary banner for server validation, the field set in dense cards (`cif-card` 12px radius,
  10px/12px section padding, heads 15px/600 primary with a regular caption), the to-do card, the sticky
  action bar; then a Review step (one card "Review your details" with the grouped read-only summary) with
  "Go Back and Edit" (flat primary `o_arrow_back`) and "Confirm & Submit" (`positive` `o_check`).
- Autosave 1s debounced after seeding completes (`ready` flag), with `scrollTop()` on step changes.
