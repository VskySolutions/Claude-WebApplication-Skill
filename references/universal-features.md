# Universal Features: the collaboration and personalisation layer

A set of components that attach to **any** record through an `(entityType, entityId)` pair, giving every
module the same conversation, activity trail, checklists, attachments, tags, pins, colours, reminders,
permalinks, PDF export, deleted-records management, field history and notifications. This file describes
their UI; the API groups are in `api-contract.md`.

## Contents
1. Wiring a record type into the layer
2. `EntityUniversalPanel` (tabs)
3. Conversation panel
4. Activity timeline
5. Checklists
6. Attachments
7. Tags
8. Per-record actions: header bar and row menu
9. Row personalisation: pins and colours
10. Field change history
11. Deleted records management
12. Notifications: bell, page, preferences, mentions, pinned
13. Sticky notes
14. Admin pages

---

## 1. Wiring a record type into the layer

- `EntityType` (frozen numeric enum in `services/api.js`) names every record type; retired numbers are
  never reused.
- `useEntityMeta()` maps a type to `{ label, icon, route(id) }` with a fallback `{ "Record", "o_description", dashboard }`;
  `typeSlug(type)` feeds the `/entity/:type/:id` permalink.
- `useNotificationMeta()` maps notification types to `{ label, icon, color }`; `useNotificationRoute()`
  decides where a notification opens (falls back to the record).
- A detail page drops in `<entity-universal-panel :entity-type :entity-id />` and/or
  `<entity-header-actions :entity-type :entity-id label="user" />`; a list adds `EntityRowMarks` (or
  `EntityRowActionsMenu`) to its actions column and `DeletedRecordsPanel` under the table.

## 2. `EntityUniversalPanel`

`q-card flat bordered` with an optional `title` (`text-subtitle1`), an inline **Tags** row (`showTags`),
a separator, `q-tabs dense align="left" active-color="primary" indicator-color="primary" class="text-grey-7"`
with icon+label tabs from a registry (`conversation o_chat`, `activity o_history`, `checklists o_checklist`,
`attachments o_attach_file`; `tabs` prop picks and orders them, `initialTab` chooses the first), and
`q-tab-panels keep-alive animated` whose panels mount lazily on first open.

A feature page may instead open the same panels in dialogs from its header (an **Activity** dialog
640px with tags on top, a refresh icon and the timeline in a `max-height: 62vh` body; a **Conversation**
dialog 640px × 76vh with an avatar-icon header and the chat panel at `height: 100%`).

## 3. Conversation panel

`EntityConversationPanel` (`height` prop, default 440px) is a chat, not a comment list:
- Thread (`#f5f7f9`, 1px `#e2e7ee`, 10px radius) scrolls on its own; oldest first; grouped under centred
  day pills ("Today", "Yesterday", or the date — 11px uppercase grey on `#e7ebf0`); consecutive messages by
  one author within 5 minutes form a run (avatar and name only on the first, 2px gaps after).
- Bubbles: others left with a 30px initials avatar coloured by hashing the author id (10-colour palette),
  white bubble `12px 12px 12px 3px`; mine right, `#e3f1fd` on `#cfe4f7`, `12px 12px 3px 12px`; body 13.5px;
  time 10.5px right-aligned inside the bubble with an italic "edited" mark; a hover-only `o_more_vert`
  (`size="xs"`, top-right) opening Edit/Delete (own messages; admins may delete any).
- Mentions render as `.uf-mention` (blue text on `#e3f2fd`, 4px radius; white on my bubble).
- Search box (`dense outlined clearable debounce="300"`, max 280px) with a match count, shown once there is
  more than one message; "Jump to newest" round primary button when scrolled away from the bottom.
- Composer (always present, white, 12px radius, primary border + soft glow when focused): a borderless
  `q-editor` (`min-height 2.25rem`, `max-height 9rem`, placeholder "Write a message…  @ to mention someone"),
  a formatting toggle (`o_text_format`) that reveals the toolbar (bold/italic/underline/strike, lists, link,
  removeFormat), and a round primary **Send** (`o_send`, tooltip "Send · Enter", disabled while empty).
- `@` autocomplete: a `q-menu no-focus no-parent-event` above the composer with avatar + name + email rows,
  keyboard navigation, and token insertion done by hand on the DOM (QEditor's `runCmd` restores a stale caret).
- Empty: `o_forum` 34px + "No messages yet." + caption "Start the conversation below — type @ to bring somebody in."
- No toast on send; edits keep the scroll position; deletes confirm.

## 4. Activity timeline

`EntityActivityTimeline`: one line per event in a 4-column grid (icon dot · what · who · when) over a
hairline rail; 12.5px text; hover tint `#f6f9fc`; the change (`old → new`) in a quieter grey after the label
with a tooltip repeating both; `o_settings` before "System" actors; timestamps in tabular figures; 50 per
page with "Load more"; empty "No activity yet." Event types map to icons and **human past-tense labels**
("Status changed", "Tag applied", "Request raised", "Fully approved"); an unmapped type prints its raw
name so it is still visible. On phones the who/when columns fold under the event.

## 5. Checklists

`EntityChecklistsPanel`: "Add checklist" (dense filled primary) top-right; each checklist is a bordered card
with a bold title (struck through grey when complete), `done/total`, a delete button, a 6px positive
`q-linear-progress`, a dense list of checkbox items (struck when done, `o_close` to remove), and an inline
"Add item" input + flat Add button (Enter adds). Create uses an `AppFormDrawer` with Title and a textarea
"Items (one per line)".

## 6. Attachments

`EntityAttachmentsPanel`: `AppMultiFileUpload` with the shared limits and hint, an "Upload n files" button
when something is staged (per-file failures toast but the loop continues), then `AppStoredFileItem` rows
(`removable`, custom `fetchBlob`) each followed by a caption `uploadedBy · date`; delete confirms with
`type: "danger"`.

## 7. Tags

`EntityTagsPanel`: `o_label` + "Tags" title with a small round `o_add` whose menu lists unapplied tags
(coloured `o_label` icon, name, category caption; or a hint to create tags under Tenant Settings); applied
tags as removable chips in the tag's colour with white text; empty text "No tags added yet. Use the + button
to tag this record."

## 8. Per-record actions: header bar and row menu

`useEntityActions(entityType, entityId, label, initialPinned)` powers both:
- **`EntityHeaderActions`** (detail header): a row of `flat round dense` icons — Pin (`o_push_pin`, primary
  when pinned), Colour (`o_palette` in the current colour, menu of 28px swatches with a 2px blue outline on
  the current and a "None" button), Reminder (`o_alarm`, `orange-8` when set; opens an `AppFormDrawer` with
  a `datetime-local` "Remind me at" and a Note, plus a "Cancel reminder" footer action), Copy link
  (`o_link`, copies the `/entity/:type/:id` permalink, toasts "Link copied."), Export PDF (`o_print`, a small
  dialog with an "Include conversation" toggle and a Download button); a `negative` "Reminder overdue" badge.
- **`EntityRowActionsMenu`** (list row `⋮`): the host's own actions in the default slot, a separator, a
  "Universal actions" header, then Pin (disabled with "Max 5" when the cap is reached), Colour (nested
  swatch menu), Reminder, Copy link, Export PDF; state loads lazily the first time the menu opens.

## 9. Row personalisation: pins and colours

`useRowPersonalisation(entityType)` + `useColourCodes` + `usePins`:
- A **pin** floats the row to the top of the current page (`pinnedRowKeys` on the table; rows tinted
  `#fbfaf5`), is private to the viewer, capped at 5 per type with the cap explained in the button's tooltip
  and a warning toast; the pinned row also shows `EntityPinnedMark` (a 14px primary pin rotated −35°) before
  its identifier. Toasts: "X pinned to the top of this page, for you only." / "X unpinned."
- A **colour** paints a 4px stripe on the row's first cell (`rowColours`), from the 8-swatch palette;
  the palette menu is titled "Colour this row", has a Clear button and the note "Private to you — nobody
  else sees it." Toasts: "X coloured, for you only." / "Colour cleared from this row."
- `EntityRowMarks` renders the two controls (`type="a"` buttons, the pin leaning when on, the coloured icon
  overlaid on the grey one) and emits `toggle-pin` / `set-colour`; the page calls `syncMarks(ids)` after
  every load (one batch read of colours, one read of the user's pins).

## 10. Field change history

`FieldLogIcon` (shown beside a field when its count > 0: small `o_history` button with a floating count
badge over 5) opens `FieldModifiedLogDrawer`: an `AppViewDrawer` (560px, "Change History — <field>") with a
search box, a `q-markup-table flat dense bordered` of Changed On / Changed By (`o_settings` for System) /
From / To, and "Load more". Counts come from `useFieldLogCounts(entityType, entityId).getCount(field)`.

## 11. Deleted records management

`DeletedRecordsPanel` (`entityType`, `show`): appears under a list when the admin's "Show deleted?" toggle
is on. It is an `AppDataTable` titled "Deleted <Plural>" with a red left border and red title, columns
Record (struck-through identity + grey "Deleted" badge + orange "Overdue" chip), Deleted By, Deleted On,
Actions (Restore `o_restore` primary, Permanently Delete `o_delete_forever` negative), an "n overdue"
badge in `#actions`, bulk Restore / Permanently Delete, and a two-step hard-delete dialog ("Type DELETE to
confirm"). Restore confirms; the host reloads on `@restored`.

## 12. Notifications: bell, page, preferences, mentions, pinned

- **Bell** (`NotificationCentre`): see `app-shell.md` §5.
- **Notifications page**: a standard list (`AppListHeader` with "Mark all read" filled and "Preferences"
  outlined in `#actions`), columns Type (icon in the type colour + label, filterable by type), Notification
  (bold-when-unread title over a two-line grey body, click opens), Received, Status (Unread primary / Read
  grey-5), Actions (Open record `o_open_in_new` when linked, Mark read `o_mark_email_read` when unread);
  extra Received From/To date filters; sorted by `createdOnUtc`; filter-aware empty state
  (`o_notifications_none`).
- **Preferences page**: `AppDetailHeader` with a Save action and a `q-markup-table flat` matrix of
  notification type (icon + label) × an "In-app" `q-toggle`.
- **Mention inbox**: list with Entity (icon + label) and Status selects in the header's `#actions`,
  columns Record, Mention (two-line preview, "by author"), Date, Status; click opens the record.
- **Pinned records page**: a `q-card flat bordered` list of `q-item`s (teal-1 avatar with the entity icon,
  type label, "Pinned <date>" caption, an unpin button), "Load more", empty `o_push_pin` "You haven't pinned
  any records yet."

## 13. Sticky notes

`StickyNoteLayer` (off by feature switch): a fixed full-screen layer with two round FABs bottom-right
(`o_sticky_note_2` primary "New sticky note", `o_visibility`/`o_visibility_off` grey hide-all); desktop notes
are draggable, resizable (bottom-right grip), minimisable, z-ordered cards in pastel colours with a
"Team" badge for tenant-wide notes; mobile shows them as stacked banners at the bottom. Create dialog:
title, note (required), colour swatches, scope select ("Everywhere" / "This page only"), optional team toggle
with expiry. Per-user layout state persists through the API (debounced 800ms).

## 14. Admin pages

- **Tag Management**: table (Name, Colour as a badge in the colour, Category, Usage, audit, actions) with an
  `AppFormDrawer` (Name, Category, 9-swatch colour row).
- **Sticky Notes admin**: list of team notes with a "New team note" dialog.
- **Deleted Records Retention**: a 520px card with a numeric "Retention (days)" + Save, and an "Overdue
  records" card listing per-type counts as orange badges.
- **Modified Log configuration**: a `q-markup-table` of Entity / Field / Type (indigo "System" or grey
  "Optional" badge) / Enabled toggle (disabled for system-tracked fields), saving on toggle with a
  "Updated." toast.
