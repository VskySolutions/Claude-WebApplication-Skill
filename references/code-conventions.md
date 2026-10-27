# Code conventions for the web layer

How the Vue code is organised and written, so a new app reads like this one. Derived from the source and
`eslint.config.js`.

## Contents
1. Folder layout
2. Naming
3. Component authoring
4. Composables and modules
5. Styling inside components
6. Comments
7. Data access and defensive defaults
8. Lint and formatting
9. Checklist for a new feature module

---

## 1. Folder layout

```
src/
├── boot/            # one concern each: axios, interceptors, branding, title, components (legacy globals), i18n, error
├── css/             # tokens.scss, quasar.variables.scss, typography.scss, app.scss, page.scss, custom.scss, <record-vocab>.scss
├── components/
│   ├── common/      # App* building blocks (fields, table, drawers, headers, badges), QuickFilter*
│   ├── universal/   # Entity* collaboration components, NotificationCentre, DeletedRecordsPanel, StickyNoteLayer
│   ├── dashboard/   # DashboardWidgetWrapper, charts/
│   └── <entity>/    # small reusable field sets / dialogs for one entity (person/, user/, account/)
├── composables/     # use*.js (+ dataTable/, uf/)
├── dialogs/         # legacy
├── layouts/         # layout.vue, auth_layout.vue, public_layout.vue
├── modules/<feature>/{routes.js, pages/, components/, use*.js, *.js}
├── pages/           # error.vue, not_authorized.vue
├── router/          # index.js (guard), routes.js (root routes)
├── services/        # api.js, branding.js, jwt.js, google-fonts.json
├── shared/          # aside_header.vue, user_info.vue, session_expiry_warning.vue, features.js
├── stores/          # auth.js, tenant.js, branding.js
├── utils/           # personName.js, richText.js
└── App.vue
```

A feature module owns its routes, pages (`index.vue` list, `detail.vue`, or named pages), components, and
module-level composables/constants (`useXxxMeta.js`, `xxxStatus.js`). Anything used by two modules moves
to `components/common` or `composables`.

## 2. Naming

- Components: `PascalCase.vue` for everything new (`AppDataTable.vue`, `EntityTagsPanel.vue`,
  `UserCreateDrawer.vue`); older shell files are `snake_case.vue` (`app_menu.vue`, `user_info.vue`).
  Shared building blocks are prefixed `App`; universal-feature pieces `Entity`; drawers end in `Drawer`,
  dialogs in `Dialog`, panels in `Panel`, field sets in `Fields`.
- Templates use kebab-case tags (`<app-text-field>`, `<entity-row-marks>`).
- Composables: `useThing.js` exporting `useThing()`; pure helpers exported alongside (`formatDateOnly`,
  `validateFiles`). Constants are `UPPER_SNAKE` and frozen (`Object.freeze`).
- Routes: names `snake_case` (`user_detail`, `uf_notifications`, `rems_request_edit`); paths kebab-case
  (`/permission-groups`, `/settings/branding`); universal-feature names prefixed `uf_`.
- Page keys (table/preference cookies): kebab or snake, unique per screen (`users`, `tenant-users`,
  `rems-approvals-tenant`, `uf_notifications`).
- Permission keys: `resource.action` (`users.write`, `optionSets.manage`, `records.adminDelete`).
- CSS: BEM-like `.block__element--modifier` with a short block prefix per component (`.app-`, `.uf-`, `.rf-`,
  `.cif-`, `.qf-`); page card classes `.<page>-card`.
- API fields: camelCase; UTC instants end in `Utc` (`createdOnUtc`); booleans start with `is`/`can`/`has`.

## 3. Component authoring

```vue
<template> … </template>

<script setup>
// One or two lines saying what this component IS and the decision behind it.
import { ref, computed, watch } from "vue";
import { someApi, getApiErrorMessage } from "services/api";
import { useNotify } from "composables/useNotify";
import AppTextField from "components/common/AppTextField.vue";

const props = defineProps({
  // Why this prop exists / what shape it takes, when not obvious.
  modelValue: { type: String, default: "" },
  readonly: { type: Boolean, default: false }
});
const emit = defineEmits(["update:modelValue", "saved"]);
// or: const model = defineModel({ type: Object, required: true });

// ---- Section ----
…
defineExpose({ validate });
</script>

<style scoped>
.block { … }
</style>
```

- `<script setup>` only; props declared with types and defaults (so they do not fall onto the wrapper
  `div`); `defineModel` for field sets that write through to the parent's object; `defineExpose` for
  `validate()`, `load()`, `reload()`.
- Import every component explicitly (only the legacy `Zw*` globals are registered).
- A `computed` get/set wrapper (`open`) over `modelValue` for dialogs and drawers; reset state in a
  `watch(open, …)` on opening.
- Sections separated by `// ---- Name ----` comment rules inside long scripts.
- Lookups load on open/mount with silent failure when optional; data loads with `loading` flags and
  `getApiErrorMessage` toasts when essential.
- Use `debounce` from `quasar` for reloads, `useDebounceFn` from `@vueuse/core` for persistence.
- `storeToRefs` when destructuring Pinia state; stores expose getters for derived values.

## 4. Composables and modules

- Composables return plain objects of refs/computed/functions; module-level state (`ref` outside the
  function) is used for app-wide singletons (tenant scope, acting-as, option catalogue) and must reset on
  `session-cleared` and re-resolve on `tenant-switched`.
- Server enums and value lists are mirrored once (`useSmtpOptions`, `Permissions`, `EntityType`) and
  imported, never retyped in templates.
- Feature "meta" composables expose `xxxLabel`, `xxxHint`, `xxxOption`, `xxxOptions` for a value family so
  every screen renders it identically.

## 5. Styling inside components

- `<style scoped>`; reach Quasar internals with `:deep(.q-field__control)`; avoid `!important` except to
  beat the known legacy globals; prefer `gap` to margin utilities in bespoke flex rows; `min-width: 0` on
  flex children that must truncate; `ellipsis` class or explicit overflow rules on names.
- Use the tokens (`var(--line)`, `var(--ink-500)`, `var(--q-primary)`, `var(--teal-050)`) and Quasar colour
  utility classes; hard-coded hex only for one-off tints documented with a reason.
- Put a shared vocabulary that several screens render in a global stylesheet rather than copying it per
  component (the record label/value classes exist because three copies drifted).
- Unscoped `<style>` blocks are reserved for `body` state classes and print rules.

## 6. Comments

The codebase's comments are short, in plain English, and explain **why** — the user need, the bug that
prompted the rule, or the alternative that was rejected — not what the code does. Keep them to one or two
lines above the code they concern. Examples of the voice:

```js
// A new sort is a new list, not a new page of the old one: the row that was 40th may now be 1st.
// Said rather than left to be inferred from a missing button.
// Silent on purpose: a list that cannot read its colours is still a list.
```

Avoid restating the code, long essays, or commented-out code left in place (the few that exist are flagged
with the reason they are kept).

## 7. Data access and defensive defaults

- Read optional server fields with `?.` and fall back to `"—"` for display, `[]` for lists, `null` for ids.
- Convert at the boundary: `dateOfBirth: form.dateOfBirth || null`, `Number(form.port)`,
  `form.capacityLimit === "" ? null : Number(...)`, trim strings before sending.
- Never mutate a prop; copy rows into local form state on edit (`Object.assign(form, blankForm(), {...})`).
- Replace reactive maps rather than mutating in place when a computed depends on them.
- Guard double submits with a `saving` flag and `:loading`/`:disable` on the button.
- Keep a `blankForm()` factory per form and reuse it for reset.

## 8. Lint and formatting

ESLint 9 flat config: `@eslint/js` recommended, `eslint-plugin-vue` flat/recommended, `neostandard({ semi: true })`,
`@stylistic/indent: 2`, `@stylistic/quotes: double`, `@stylistic/semi: always`, `no-unused-vars: error`,
`vue/no-undef-properties: error`, `vue/multi-word-component-names: off`, `vue/max-attributes-per-line: off`,
`no-debugger` error in production. Run `npm run lint` (`nps lint`). Template attribute order follows the
Vue plugin's recommended order (directives, props, events). Line width in practice ≈ 110 characters.

## 9. Checklist for a new feature module

1. `modules/<feature>/routes.js` with layout wrapper, `meta.requiresAuth`, `meta.permissions`, `meta.title`;
   register it in `router/index.js`.
2. Menu entry in `app_menu.vue` under the right group with the same permission.
3. `services/api.js` resource group with documented params.
4. Permission keys added to the `Permissions` catalogue (and `usePermissionCategories` if grouped).
5. List page per `list-pages.md`; detail page per `detail-and-forms.md`; drawers/dialogs as separate
   components once they exceed a screenful.
6. `EntityType` + `useEntityMeta` entry when the record should carry universal features; drop in
   `EntityUniversalPanel` / `EntityHeaderActions` / `DeletedRecordsPanel`.
7. Option-set-driven values rendered through `AppOptionBadge` with a meta composable.
8. Lint clean; comments explain why.
