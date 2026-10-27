# The API contract the UI is built against

The shared components assume these shapes. A backend that follows them plugs into the list, form and
error patterns without changes; one that does not needs adapters in `services/api.js`.

## Contents
1. Response envelopes
2. Error codes
3. List endpoints
4. Writes
5. Files and media
6. Option sets
7. Universal-feature endpoints (shape)
8. Client conventions in `services/api.js`
9. Headers and rate limiting

---

## 1. Response envelopes

Success:
```json
{ "success": true, "message": "Users retrieved.", "data": <T>, "meta": { "page": 1, "limit": 20, "totalRecords": 137 } }
```
`meta` is present on paginated lists only. Error:
```json
{ "success": false, "message": "One or more validation errors occurred.", "error": { "code": "VALIDATION_FAILED", "details": "Name: Name is required; Email: Enter a valid email" } }
```
`details` is caller-safe prose; validation details join `field: message` pairs with `; `. The client reads
`error.details` first, then `message`.

## 2. Error codes

`ApiErrorCodes` (mirrored in `services/api.js`): `VALIDATION_FAILED`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`,
`DUPLICATE_IDENTIFIER`, `DUPLICATE_GROUP_NAME`, `PERMISSION_CEILING_EXCEEDED`, `CAPACITY_BELOW_USAGE`,
`CAPACITY_LIMIT_REACHED`, `TENANT_INACTIVE`, `TENANT_NOT_FOUND`, `TENANT_ARCHIVED`, integration-specific
codes, `INTERNAL_ERROR`. The UI switches on codes for field-level messages (duplicates → the field) and on
HTTP status for auth (401 refresh, 403 disabled, 409 conflicts shown as the server's sentence, 429 wait).

## 3. List endpoints

Query parameters: `page` (1-based), `limit`, `sortBy` (**the column `name` as the UI defines it**; the
server maps it), `descending` (bool), `search`, plus named filters (`isActive`, `status`, `type`, ids).
Multi-value filters are repeated keys (`clientPersonIds=a&clientPersonIds=b`; axios
`paramsSerializer: { indexes: null }`). Date-range filters are UTC instants (`sentFrom`/`sentTo`,
`createdFrom`/`createdTo`, `updatedFrom`/`updatedTo`) computed from the picked day in the tenant zone.
Booleans arrive as `true`/`false` strings from filter selects and are converted by the page.

Rows carry the audit quartet `createdBy`, `createdOnUtc`, `updatedBy`, `updatedOnUtc` (actor **names**,
sortable server-side), an `isActive`/`status`, and soft-delete markers (`deleted`, `deletedOnUtc`) where
applicable. A few endpoints return everything (`data` is a bare array, no `meta`) and still accept
`sortBy`/`descending`; the UI treats them as load-all lists.

Quick-filter counts: a sibling `…/quick-counts` endpoint taking the same filters and returning
`{ <filterName>: { <value>: count } }`.

Every timestamp is an ISO 8601 UTC instant; calendar dates (`DateOnly`) are `YYYY-MM-DD` strings and are
never shifted.

## 4. Writes

- `POST` create returns the created record in `data` (lists then route to it or reload); `PUT` full/partial
  updates return the updated record; `DELETE` returns the envelope (`.then(envelope)`), sometimes with a
  `message` worth toasting.
- Partial update semantics: omitted/`null` fields are unchanged on most endpoints, so the UI sends `""`
  explicitly to clear a text field where that matters.
- Status toggles are their own endpoints (`PUT …/{id}/status { isActive }`), archive/activate likewise.
- Reconcile-style endpoints (`PUT …/{id}/groups { groupIds }`, `POST …/tenant-assignments { tenantId, roleIds }`)
  replace a whole set; the UI edits the set in a dialog and submits once.
- Secrets are write-only: never returned, omitted on update to keep the stored value.
- One-time secrets (`temporaryPassword`) come back once in the create/reset response and are shown in a dialog.

## 5. Files and media

- `POST /api/media` multipart (`file`, `mediaCategory`, optional `entityType`/`entityId`) → `{ id, publicUrl, … }`;
  `GET /api/media/{id}/content` (authenticated, `responseType: "blob"`) for bytes; public URLs are prefixed
  with the API base URL.
- Feature attachments: `POST /api/uf/attachments` multipart, `GET …/{id}/download` blob, `DELETE …/{id}`.
- Record-linked documents are attached by media id after upload (`POST …/{id}/files { mediaIds }`), removed by
  link id; removals are idempotent.
- PDF export: `POST /api/uf/pdf-export { entityType, entityId, includeConversation }` → blob.
- Server limits mirrored in the UI: 10 MB per file, 15 files / 100 MB per set, extension allow-list.

## 6. Option sets

Tenant-editable value lists keyed by `(entityType, key)`. `GET /api/option-sets/resolve?entityType=&key=&parentItemId=`
returns the active, ordered items:
`{ id, value, label, description, backgroundColor, textColor, icon, isDefault, isActive, isSystem, parentItemId, metadataJson }`.
Some lists are referenced by **code** (`value`), others by **item id**; `metadataJson` carries grouping
(`{ "group": "Geography" }`). Standard lists are seeded per tenant and editable; closed lists lock the codes
the application branches on (`isSystem` items keep their `value`, cannot be hidden on a closed list).
Admin CRUD: `/api/option-sets`, items under `…/{setId}/items`, `…/items/reorder { itemIds }`.

## 7. Universal-feature endpoints (shape)

All keyed by `entityType` (number) + `entityId` (GUID string):
conversation messages (list paginated with `search`, create `{ body, mentionedUserIds }`, update, delete,
`mention-candidates?search=`), entity tags (`/api/uf/entity-tags`, picker `/api/uf/tags`, admin `/api/admin/tags`),
attachments, activity (paginated events `{ id, eventType, oldValue, newValue, actorId, actorName, occurredOnUtc }`),
reminders (`{ entityType, entityId, dueAtUtc, note, isOverdue, isDispatched }`), notifications
(`/api/notifications` list with `isRead`, `type`, `search`, `createdFrom/To`; `unread-count`; `{id}/read`;
`read-all`; `preferences`), mentions (`/api/uf/mentions`), pins (`/api/uf/pins`, max 50 per user, 5 per type),
colour codes (`GET /api/uf/colour-codes?entityType&entityIds[]` → `{ id: "#hex" }`, `PUT` upsert),
checklists (`/api/uf/checklists`, items add/toggle/edit/reorder/remove), sticky notes (+ per-user `sticky-note-states`),
deleted records (`/api/uf/deleted` paginated, `restore`, `restore/bulk`, `hard-delete` with
`confirmationToken` = the record's identity, `hard-delete/bulk` with `confirmationCount`, retention config,
`retention-overdue`), modified log (`/api/uf/modified-log` history, `icon-counts`, admin config toggles),
dashboard (`/api/dashboard/users`, `/platform` with `X-Dashboard-Force-Refresh: 1`, `/layout` get/put).

## 8. Client conventions in `services/api.js`

- One exported object per resource (`userApi`, `tenantApi`, `ufPinApi`, …) with short verb methods:
  `list(params)`, `get(id)`, `create(payload)`, `update(id, payload)`, `remove(id)`, `setStatus(id, isActive)`.
- `.then(unwrap)` returns `data`; `.then(envelope)` returns the whole envelope (lists needing `meta`, deletes,
  calls whose `message` is shown).
- A comment above each method documents the params/payload shape and any non-obvious rule.
- `getApiErrorMessage(err, fallback)`, `getApiErrorCode(err)`, `webUrl(path)` (prefixes `WEB_BASE_URL`),
  `mediaApi.absoluteUrl(publicUrl)`, `assetUrl(path)`.
- Enums the UI switches on are mirrored as frozen objects (`EntityType`, `OptionItemSortMode`, `NotificationType`).
- Environment: `process.env.API_BASE_URL`, `WEB_BASE_URL`, `BUILD_PUBLIC_PATH`, `PUBLISH_FOLDER` from
  `config/env.<QENV>.cjs` (never committed secrets).

## 9. Headers and rate limiting

See `auth-and-session.md` §7 for the headers. 429 responses carry `Retry-After` seconds; the interceptor
rejects the request, sets `error.retryAfterSeconds`, and shows one warning toast per 5-second burst;
auto-save honours the wait.
