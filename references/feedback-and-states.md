# Feedback and states: notifications, confirmations, loading, empty, errors, badges, tooltips

The rules for how the app talks back to the user. Consistency here is most of what makes the product
feel professional.

## Contents
1. Toast notifications
2. Confirmation dialogs
3. Loading states
4. Empty states
5. Banners
6. Badges, chips and pills
7. Tooltips and info tips
8. Save-state indicators
9. Disabled, read-only, hidden
10. Error handling end to end
11. Progress and counts
12. Keyboard behaviour

---

## 1. Toast notifications

`useNotify()` → bottom, multi-line, `html: true`, standard icons. Types and when:

| Call | Use for | Wording |
|---|---|---|
| `notify.success` | A write the user asked for has landed | Past tense, the thing then the verb: "Person created.", "Status updated.", "Template reset to default.", "Link copied." Add the one fact worth knowing: "X pinned to the top of this page, for you only." |
| `notify.error` | A write or an essential read failed | `getApiErrorMessage(err)` (the server's sentence), or a fallback: `getApiErrorMessage(err, "Could not change password. Please try again.")`; local refusals: "Select a person.", "Please fix the highlighted fields." |
| `notify.warning` | Partial success or a soft block | "User created, but the invitation email could not be sent…", "You can pin at most 5 records…", "Copy failed — please select and copy manually.", 429: "Too many requests. Please wait N seconds and try again." (one per 5s burst) |
| `notify.info` | A side effect on somebody/something else, or an undo | "X is no longer the department head.", "Removed from pinned.", "Opening print dialog. Choose "Save as PDF" to export." |

Do **not** toast: a chat message posting (the message appearing is the confirmation), auto-saves (the
indicator is the feedback), failed background reads that cost the reader nothing (row colours, filter
option lists, quick counts, branding, delegations), the forgot-password response (always the same banner).
Prefer the server's own `message` when it carries a count ("3 users granted the role").

## 2. Confirmation dialogs

`useConfirm().confirm({ title, message, confirmLabel, cancelLabel, type })` — `type: "danger"` colours the
confirm button `negative`. Titles are short noun phrases ("Delete role", "Reset password", "Change
username", "Turn off email notifications"); messages name the record and the consequence; the confirm label
is the verb. See `detail-and-forms.md` §7 for when to confirm.

Leaving a dirty editor: `onBeforeRouteLeave(() => dirty ? confirm({ title: "Unsaved changes", message: "Leave without saving …?", confirmLabel: "Leave", type: "danger" }) : true)`.
The forced password-change screen blocks leaving with `notifyWarning("Please set a new password before continuing.")`.

## 3. Loading states

| Context | Pattern |
|---|---|
| Detail page first load | `div.row.flex-center.q-pa-xl` → `q-spinner color="primary" size="40px"`; reloads keep the page (`loading = !record`) |
| Panel / card body | `q-inner-loading :showing="loading && !items.length"` |
| Dialog body | `q-card-section class="row flex-center q-pa-xl"` → spinner 32–36px; smaller panels 24–28px |
| Dashboard widget | `q-skeleton type="text"`, `type="rect" height="80px"`, `type="text" width="60%"` |
| Table | `:loading` on `AppDataTable` (Quasar progress bar; the refresh icon spins) |
| Button | `:loading="saving"` on the primary; per-row `:loading="busyId === row.id"` so exactly one spins |
| Select | `loading` prop → spinner in `append` |
| Inline lookup | `q-spinner size="18px"` in the input's `append` |
| Full-page redirect / SSO | `q-spinner-dots color="primary" size="40px"` with a line of text |
| Pre-bundle | the CSS `.loader` ring in `index.html` |

Never block the whole screen with the `Loading` plugin for ordinary fetches.

## 4. Empty states

- List: `div.full-width.column.flex-center.q-pa-xl.text-grey-6` → `q-icon size="40px" class="q-mb-sm"`
  (a topic icon: `o_mail`, `o_workspaces`, `o_assignment`, `o_task_alt`) → `div.text-subtitle1.q-mb-xs`
  ("No email accounts configured") → one line of explanation (`q-mb-md` when a button follows) → the
  primary CTA (same label/icon as the header's Add). Filter-aware copy: "No matching notifications" / "Try
  clearing the filters or the search."
- Default table empty: `o_inbox` 32px + "No Data Available" (`q-pa-lg`).
- Panel: `div.text-grey-6.q-pa-md.text-center` "No attachments yet." / `q-item` with `text-grey-6` section
  "Nobody in this tenant holds this role."; a hint on how to fix it where useful ("No tags. Create some under
  Tenant Settings → Tag Management.").
- Positive empty (alerts, onboarding): `column flex-center q-pa-lg text-positive` with `o_check_circle` 32px
  and `text-subtitle2` "No Active Alerts" / "All onboarded ✓".
- Pickers: `#no-option` row in `text-grey-6` with the reason; an empty options list gets a `hint` naming the
  fix ("Nobody holds the "CSE" role — assign it on a user's page in Administration → Users.").
- A master-detail with nothing selected: a card with `o_groups` 36px and "Select a group to see its members."

## 5. Banners

`q-banner dense rounded` (or `class="rounded-borders"`) with `#avatar` icon in the text colour and an
optional `#action` button (`flat no-caps` in the same hue). Placement: directly under the header card
(page-level), inside a card above its fields (card-level), inside a dialog body (dialog-level), under a
control it explains (control-level, e.g. "This client has no email address on file, so nothing can be sent.").
Colour semantics are in `foundations.md` §4. Lists in banners use `ul.q-my-xs.q-pl-md`. A banner that
states a *decision* (read-only, closed, you approved) replaces the explanation that would otherwise be
inferred from dead controls — say it once at the top.

Inline notes (not banners): `div.text-caption.text-grey-7` under a toggle or picker; `.rf-hint` 12px
`--ink-500` under a chip row (`--error` variant in `#c10015`); small flex note boxes with an icon and a
button for a found-record suggestion (`.cif-onfile` red tint, `.cif-maconomy` teal tint, `--warn` amber).

## 6. Badges, chips and pills

- `q-badge` for a **state**: coloured by meaning (`foundations.md` §4), optional leading icon 12–13px with
  `q-mr-xs`; inside a detail header it is reshaped into a 24px pill automatically.
- `q-chip dense` for **metadata and selections**: `teal-1`/primary (brand tint), `blue-grey-1`/`blue-grey-8`
  (neutral meta), category colours for roles (`roleCategoryChip`), `square` for compact role chips,
  `removable` for selections, `clickable` for pick-lists, `icon` when the category needs it. Chips may carry
  a `q-tooltip` with their meaning.
- `AppOptionBadge` whenever the value comes from an option set (status, type, decision, email event, role):
  the tenant's label, colours, icon and description tooltip.
- Count badges: `q-badge color="primary" class="q-ml-sm"` after a card title; `q-badge floating color="primary"`
  on the Filters button; `q-badge floating color="negative"` on the bell; "n selected" `q-chip dense color="primary" text-color="white"`.
- Tally badges on quick-filter buttons (`rounded`, translucent black on white buttons, white on the chosen one).
- Tags as `q-chip dense removable text-color="white"` with the tag's own background colour.
- A badge that is also a control (`RelatedStatusSelect`): the badge inside a padding-less flat `q-btn` with a
  trailing `o_expand_more`, opening a `q-menu` whose items are the same badges with a check on the current.

## 7. Tooltips and info tips

- Every icon-only `q-btn` carries a `q-tooltip` with its verb; conditional tooltips explain a disabled state.
- `AppInfoTip` beside section titles (16px grey-6, `cursor: help`) and in list headers (18px primary) for
  "what this is for"; `info` prop on fields for "why these options".
- Informational tooltips on values use `:delay="300"` (badges 300, labels 200, timeline rows 400) and
  `max-width` 260–360px; anchor `top middle / bottom middle` by default, `bottom right / top right` for icons
  at the right edge, `anchor="center right" self="center left"` for rail items.
- A tooltip never carries the only copy of something essential; it explains or adds.

## 8. Save-state indicators

- `AppAutoSaveState` (spinner "Saving…" / green "Saved" / red API message) in a card header.
- Header dot (`.rf-save`, 24px round): idle outline `--line`/`--ink-500`, busy `--teal-050`/`--teal-900`,
  ok `#e8f5ec`/`#1b5e34`, warn `#fff4e5`/`#8a5300`, bad `#fdecea`/`#8a1c12`; state text on the tooltip.
- Sticky bar line (`.pef-save`): icon + sentence, `positive`/`negative` colour by state.
- Field-level "Saved just now" is never shown per field; one indicator per card or per page.

## 9. Disabled, read-only, hidden

- **Hidden** (`v-if`): actions the user may never take (permission), menu items, columns (Tenant on
  non-platform lists), whole sections that do not apply to the record type.
- **Disabled** (`:disable`) with a tooltip or a note: blocked by the current state ("The active account
  cannot be deleted.", "Complete the required fields to continue.", "Max 5" beside a pin), by another
  person's claim, or by a cap. Disabled options in a select carry the reason in their label
  (`"Jane Doe — approves as CSE"`).
- **Read-only** (`readonly`): values the viewer may see but not change, drawn with a lock icon + tooltip
  or as `AppReadonlyField`/record text; a page that is read-only for the viewer says so once in a banner.
- A locked toggle in a `q-btn-toggle` stays visible and disabled with a note underneath.

## 10. Error handling end to end

1. `getApiErrorMessage(err, fallback)` reads `error.details` → `message` → string body → `err.message` →
   fallback; `getApiErrorCode(err)` reads the stable code.
2. Page load failure: toast, keep the previous content, or `router.replace` to the list; a widget shows a
   red banner with **Retry**; a dialog preview failure shows the error banner inside the dialog; a lookup
   failure shows the message in the dropdown's empty row ("a toast per keystroke is a wall of toasts").
3. Action failure: toast; keep the dialog/drawer open with the typed data; revert optimistic UI
   (`assignedGroups.value = previous`); reload when the server is the authority on what survived.
4. Validation failure (`VALIDATION_FAILED`): map to fields + summary banner + "Please fix the highlighted fields."
5. 401: silent single-flight refresh and retry; a dead refresh token clears the session and routes to login
   with `?redirect=`. 403 on login: "Your account is disabled. Please contact your administrator."
6. 429: the request fails like any other and one warning toast per burst states the wait.
7. Rate-limited auto-save reschedules itself after `Retry-After`.
8. Degraded-but-working states are said in place, not toasted: "The tax-form list could not be loaded for
   your account.", "Searching Maconomy…", "Only clients already in the portal are listed."

## 11. Progress and counts

- `q-linear-progress :value color="positive" track-color="grey-3" size="6px" rounded` for checklist completion,
  with `completed/total` beside the title.
- Round progress: `q-badge` approved count (grey until > 0, then positive) + `negative` rejected count +
  "of N" caption; tooltip spells out the breakdown.
- Caps as pills: `n / 10` bordered pill, amber when full.
- Load more: `row justify-center q-mt-sm` → `q-btn flat no-caps color="primary" label="Load more" :loading`
  (pages of 20–50); shown while `items.length < total`.

## 12. Keyboard behaviour

- Enter submits `q-form`s (`@submit.prevent.stop`); Enter in an inline "create" box creates (`@keyup.enter`).
- Chat composer: Enter sends, Shift+Enter breaks a line; `@` opens mention autocomplete with ↑/↓, Enter/Tab,
  Esc.
- Email list field: Enter, `,` or `;` commits an address.
- Autocomplete inputs: ↑/↓ move, Enter picks, Esc closes.
- Pane splitter: ←/→ step, Home resets; dialogs close on Esc unless `persistent`; fullscreen widgets close on Esc.
