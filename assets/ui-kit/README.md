# UI kit: a verbatim snapshot of the EMS Portal web shell and component library

Copied from `THF_EMSPortal-v2/WEB` at commit `71ac45a` (26 Oct 2026) on 27 Oct 2026, keeping the
original `src/...` paths so a file can be dropped into a new Quasar project at the same place. The
feature modules (REMS, tenants, users, roles, persons, SMTP, email templates, option sets, permission
groups, user groups, Maconomy) are **not** included; `FILELIST.txt` lists what is.

The source of truth is the repository, not this copy. For the full working shell including the API,
start from the sibling base repository (`D:\GIT-VSky Solutions\VSkyBaseFramework`, forked 9 Sep 2026 and
older than this snapshot) and refresh its web layer from these files.

## What is here

| Path | Contents |
|---|---|
| `package.json`, `quasar.config.js`, `eslint.config.js`, `jsconfig.json`, `postcss.config.cjs`, `index.html`, `public/web.config` | Project scaffolding (dependency versions, Quasar config, lint rules, IIS rewrite). `config/env.*.cjs` files are deliberately absent; create them with `API_BASE_URL`, `WEB_BASE_URL`, `BUILD_PUBLIC_PATH`, `IGNORE_PUBLIC_FOLDER`, `PUBLISH_FOLDER`. |
| `src/css/*` | Tokens, Quasar variables, typography, globals. `app.scss`/`custom.scss` carry legacy rules; keep `tokens.scss`, `quasar.variables.scss`, `typography.scss`, the standard-typography block at the end of `custom.scss`, and `rems.scss` (rename its classes to `record-*`). |
| `src/boot/*`, `src/App.vue`, `src/router/*`, `src/layouts/*`, `src/shared/*`, `src/pages/*`, `src/components/app_menu.vue` | The shell: axios + interceptors, branding/title boot, router guard, three layouts, header pieces, 404/403, the data-driven menu. `router/index.js` and `app_menu.vue` list THF modules; prune them. |
| `src/components/common/*` | The 33 `App*` building blocks plus `QuickFilterBar`/`QuickFilterGroup`. |
| `src/components/universal/*`, `src/composables/uf/*`, `src/modules/universal/*` | Universal Features UI and pages. |
| `src/components/dashboard/*`, `src/modules/dashboard/*` | Dashboard shell, widget wrapper, charts, registry, widgets. |
| `src/modules/branding/*`, `src/services/branding.js`, `src/stores/branding.js`, `src/services/google-fonts.json` | Tenant theming engine and editor. |
| `src/modules/auth/*`, `src/modules/account/*`, `src/components/account/*` | Sign-in, reset, SSO callback, account/profile/change password. |
| `src/components/person/*`, `src/components/temp_password_dialog.vue` | Examples of a reusable field set, quick-add dialog, one-time-secret dialog. |
| `src/composables/*.js`, `src/composables/dataTable/*`, `src/utils/*`, `src/services/api.js`, `src/services/jwt.js`, `src/stores/*` | Composables, helpers, the API client (THF resource groups included as reference; keep the envelope helpers, `EntityType`, the `uf*Api` groups and trim the rest), stores. |
| `src/modules/rems/useAutoSave.js`, `src/modules/rems/components/DetailGrid.vue`, `src/modules/rems/components/RelatedStatusSelect.vue` | Generic pieces that happen to live under a feature module: multi-part auto-save, the read-only detail grid, the badge-as-dropdown. |

## Coupling points to adjust when reusing

- `services/api.js`: `getApiErrorMessage`, `getApiErrorCode`, `ApiErrorCodes`, `EntityType`, `mediaApi`,
  `brandingApi`, `authApi`, `tenantApi`, `optionSetApi`, `dashboardApi`, `uf*Api` are imported by the shared
  layer. Keep those shapes (see `references/api-contract.md`).
- `stores/tenant.js`: `AppFormDrawer` calls `setUnsavedForm`; `useDateFormat` reads `activeTenant.timeZoneId`.
- `composables/usePermissions.js`: the `Permissions` catalogue is THF's; replace the keys, keep the shape.
- `composables/uf/useEntityMeta.js`, `useNotificationMeta.js`, `useNotificationRoute.js`: entity types,
  notification types and routes are THF's; replace.
- `components/app_menu.vue`, `router/index.js`, `modules/auth/returnPath.js` (`LANDING_PATH`): THF's menu,
  module list and landing page.
- `AppTenantScopeSelect` / `useTenantScope`: only meaningful with a super-admin tenant override header.
- Legacy files intentionally left out: `components/zw_*.vue`, `dialogs/*`, `assets/utils.js`,
  `composables/useFilters.js` (`boot/components.js` still registers the `Zw*` globals; remove it or them).

## npm dependencies the kit needs

`quasar`, `@quasar/extras`, `vue`, `vue-router`, `pinia`, `axios`, `@vuelidate/core`, `@vuelidate/validators`,
`@vueuse/core`, `libphonenumber-js`, `country-state-city`, `validator`, `ckeditor5`, `@ckeditor/ckeditor5-vue`,
`vue-advanced-cropper`, `date-fns`; dev: `@quasar/app-vite`, `eslint`, `@eslint/js`, `eslint-plugin-vue`,
`neostandard`, `cross-env`, `dotenv`, `postcss`, `autoprefixer`, `nps`. Versions are in `package.json`.
