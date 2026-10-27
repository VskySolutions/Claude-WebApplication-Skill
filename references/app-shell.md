# Application shell: layouts, navigation, header, routing, permissions, session plumbing

How the frame around every page is built. Source: `src/layouts/*`, `src/components/app_menu.vue`,
`src/shared/*`, `src/router/*`, `src/boot/*`, `src/pages/*`.

## Contents
1. The three layouts
2. Header (toolbar) contents
3. Left navigation drawer and AppMenu
4. User menu
5. Notification bell
6. Multi-tenant controls in the shell
7. Footer
8. Error pages (404 / 403)
9. Routing conventions and the global guard
10. Permission gating in the UI
11. App-wide events and persisted UI state
12. Document title and favicon
13. Feature switches

---

## 1. The three layouts

| Layout | Used for | Structure |
|---|---|---|
| `layouts/layout.vue` | Every authenticated page | `q-layout view="lHh Lpr lFf"`: fixed header, left drawer (`show-if-above`), page container, footer. Also mounts the global tenant-scope banner and the (switched-off) sticky-note layer. |
| `layouts/auth_layout.vue` | Login, forgot/reset password, SSO callback | No header/drawer. A centred column (`max-width: 420px`, `min-height: 100vh`, `padding: 24px`, background `#f4f6fb` or the tenant's login colour/image). Logo 44px (a tenant logo keeps its height, auto width, max 240px), optional app name beside it, optional tagline, the routed card, a "Need help?" line (support email / link) and a copyright line with the current year. |
| `layouts/public_layout.vue` | Anonymous forms reached by an emailed link | `q-header` in the primary colour with the dark-background logo (32px) and optional app name; page background `#f4f6fb`; content column `max-width: 960px`, padding `16px 16px 28px` (phone `12px 10px 24px`). |

Every routed page starts with `<q-page padding>` (except the public form, which is a plain `div` inside
the layout's own padded column, and the two error pages, which are `fullscreen`).

## 2. Header (toolbar) contents

`q-header bordered class="header"` is **white** with dark ink (Quasar's default white-on-primary is
overridden). Toolbar height 45px (`.header-top`), horizontal padding 23px. Left to right:

1. Menu toggle `q-btn flat dense round icon="o_menu" class="text-black"` (only when signed in).
   Above the drawer breakpoint it toggles the icon rail; below it, it opens/closes the overlay drawer.
2. Application name as a flat `no-caps` button (`text-weight-bold fs-18 text-primary`) that navigates to `/`.
3. Right cluster (`row q-gutter-sm items-center no-wrap`):
   - Tenant switcher `q-btn-dropdown flat no-caps icon="o_apartment"` (only when the user belongs to more
     than one tenant) listing tenants with their role names as captions and a check icon on the active one.
   - Active-tenant role chips `q-chip dense square color="teal-1" text-color="primary" class="text-capitalize"` (hidden on xs).
   - Super-admin "View as" tenant-scope dropdown (`AppTenantScopeSelect`): `flat no-caps dense icon="o_travel_explore"`,
     label "View as" or the scoped tenant's name, `text-orange-9` while scoped, with "Back to my tenant".
   - Notification bell (`NotificationCentre`).
   - User menu (`user_info.vue`), or a `Login` button (`unelevated color="primary" icon="o_login"`) when signed out.

## 3. Left navigation drawer and AppMenu

- `q-drawer show-if-above bordered class="bg-white" :width="292" :mini-width="60" :breakpoint="1024" :mini="collapsed"`.
- Top: `aside_header.vue`, 57px tall, bottom border, logo (tenant logo or stock, height from branding,
  default 57px with 5px padding), optional wordmark `text-weight-bold fs-20`. Collapsed to the rail the
  wordmark hides (`q-mini-drawer-hide`) and a compact mark shows if the tenant uploaded one (`q-mini-drawer-only`).
- Body: `q-scroll-area class="fit"` wrapping `AppMenu`.
- Open/closed and mini state persist in LocalStorage (`leftDrawerOpen`, `leftDrawerMini`).

`AppMenu` (`components/app_menu.vue`) is data-driven:

```js
{ key, label|null, icon, startCollapsed?, items: [{ label, icon, to|action, permissions|null, exact?, hidden? }] }
```

- A section with `label: null` renders its items flat at the top (Dashboard). Labelled sections are
  `q-expansion-item dense` groups with header class `app-menu__group text-primary text-weight-bold`
  (11px uppercase, .08em tracking, 38px tall); their open/closed state persists per key in
  LocalStorage `appMenuCollapsed`; `startCollapsed: true` for housekeeping groups (Tenant Settings, Account).
- Items: `q-item dense clickable v-ripple` 34px min-height, 20px icon, avatar section 32px wide,
  nested items indented 20px, `active-class="text-primary bg-teal-1 app-menu__active"`.
- Permission filtering: `permissions: null` means every signed-in user; otherwise `hasAnyPermission`.
  Items the user cannot see are dropped and empty sections disappear. `hidden` removes an item behind a
  feature switch.
- `action: "logout" | "logoutAll"` items run handlers instead of navigating.
- Collapsed rail: each group becomes one icon; hovering (or clicking) opens a `q-menu` flyout to the right
  (`anchor="top right" self="top left"`, `no-focus no-refocus no-parent-event`, `jump-right`/`jump-left`
  transitions) with the group name as a header and the items inside; a 150ms close delay lets the pointer
  cross from icon to flyout. Flat items get a right-side `q-tooltip` while collapsed.
- Order principle: overview → set up → configure → operate → personal.

## 4. User menu

`shared/user_info.vue`: a flat `grey-8` button showing a 38px `o_account_circle` icon (orange, legacy),
the display name (`fs-16 text-capitalize`) and a `o_keyboard_arrow_down`. Its `q-menu` is a 250px list:

- Identity item: avatar icon, name (`text-subtitle1 text-weight-medium text-capitalize`), email caption,
  roles joined with ` · ` plus an `o_info` icon whose dark tooltip (`bg-grey-9 text-white`, max 340px)
  lists every permission in two columns, prettified (`users.reset_password` → `Users · Reset password`),
  then the tenant name with an `o_apartment` icon.
- Separator, then links with orange outlined icons: My Notifications, My Mentions, My Pinned,
  Notification Preferences; separator; Profile, Change Password; separator; Logout, Logout all devices.

## 5. Notification bell

`components/universal/NotificationCentre.vue`: `q-btn flat round dense color="grey-8" icon="o_notifications"`
with a floating `negative` badge (`99+` cap). Unread count polls every 30s. Its `q-menu` is a 500px
(max 90vw) card: title row with "Mark all read" (`flat dense no-caps size="sm"`, disabled at zero), a
separated list (max-height 360px, scroll) where unread rows are `bg-teal-1`, each row shows a type icon
in the type's colour, a wrapping bold title, a two-line body caption and the timestamp; footer "View all".
Clicking marks read and routes to the record.

## 6. Multi-tenant controls in the shell

- **Switching tenant** (user belongs to several): confirms first when a form is dirty
  (`Dialog.create` "Unsaved changes… Continue?"), swaps the token, then dispatches
  `window.dispatchEvent(new CustomEvent("tenant-switched"))` so every list reloads.
- **Viewing another tenant** (super admin): one global selection kept in LocalStorage
  (`adminTenantOverride`) and sent as the `X-Site-Id` header. While active, every authenticated page shows
  a `q-banner dense inline-actions class="bg-orange-2 text-orange-10 q-px-md"` under the header:
  "Viewing **Tenant** — changes apply to that tenant." with a "Back to my tenant" action. Pages that are not
  lists listen to `tenant-switched` themselves to reload.

## 7. Footer

`q-footer bordered class="bg-white"` with centred 13px text: the tenant's footer text when set, otherwise
a copyright line with a link to the vendor site.

## 8. Error pages (404 / 403)

`pages/error.vue` and `pages/not_authorized.vue`: `div.fullscreen.flex.flex-center.bg-grey-2.q-pa-md` →
`q-card flat bordered class="text-center q-pa-xl" style="max-width: 460px; border-radius: 16px"` →
80px icon (`o_travel_explore` primary / `o_lock` negative), `text-h2 text-weight-bold` code,
`text-h6 text-grey-9` title, `text-body2 text-grey-7` explanation, `q-btn unelevated color="primary" icon="o_home" label="Go Home" no-caps class="q-mt-lg"`.

## 9. Routing conventions and the global guard

Each feature module owns `modules/<feature>/routes.js`:

```js
export default [{
  path: "/users",
  component: () => import("layouts/layout.vue"),
  children: [
    { path: "", name: "users", component: () => import("modules/user/pages/index.vue"),
      meta: { requiresAuth: true, permissions: ["users.read"], title: "Users" } },
    { path: ":id", name: "user_detail", component: () => import("modules/user/pages/detail.vue"),
      meta: { requiresAuth: true, permissions: ["users.read"], title: "User" } }
  ]
}];
```

- Route names are `snake_case`; list `name` is the plural, detail is `<singular>_detail`; create/edit
  variants are separate paths of the same page (`requests/new`, `requests/edit/:id`, `requests/:id`).
- `router/routes.js` holds `/` → landing redirect, `/not-authorized`, and the catch-all `pages/error.vue`.
  `router/index.js` pushes every module's routes, uses `createWebHistory`, `scrollBehavior` to top.
- Global `beforeEach`:
  1. If any matched record has `meta.requiresAuth`, `authStore.ensureSession()` (refreshes an expired
     token first); not signed in → `{ name: "login", query: { redirect: fullPath } }` (no redirect for the
     default landing).
  2. Signed in and on `/auth/*` → landing.
  3. `mustChangePassword` → forced to `change_password`.
  4. Permission gate: the deepest matched record's `meta.permissions` array; needs **any one** of them,
     otherwise `notifyWarning("You do not have permission to access that page.")` and `next("/")`.
- Redirect routes keep old links alive (`redirect: (to) => …`) when pages are split or renamed.
- Permalink convention `/entity/:type/:id` → `PermalinkResolver` → `useEntityMeta().routeFor()`.
- Post-login destination accepts only internal paths (`startsWith("/") && !startsWith("//")`).

## 10. Permission gating in the UI

- `composables/usePermissions.js` exports a frozen `Permissions` catalogue (string keys like
  `users.write`) mirrored from the API, and `usePermissions()` → `{ has, hasAny }` reading the auth store's
  permissions decoded from the JWT (reactive across tenant switches).
- Gate at three levels: route `meta.permissions`, menu item `permissions`, and in-page `v-if="has(...)"`
  on buttons/cards/columns. Compute them once: `const canWrite = computed(() => has(Permissions.PersonsWrite))`.
- **Hide** what the user can never do; **disable with a tooltip** what is blocked by context ("Only a
  Super Admin can manage this user.", "The active account cannot be deleted."). Read-only is said once in
  a banner at the top of a page rather than inferred from dead fields.
- Role names (`SuperAdmin`, `TenantAdmin`) are checked only for scope decisions (whole-tenant views, who may
  grant SuperAdmin); everything else is permission keys.

## 11. App-wide events and persisted UI state

| Mechanism | Key / event | Purpose |
|---|---|---|
| `window` event | `tenant-switched` | Lists (`useListTable`) and settings pages reload; option catalogues re-resolve; branding reloads. |
| `window` event | `session-cleared` | Module-level caches forget themselves; branding keeps its cache for the next sign-in screen. |
| LocalStorage (Quasar) | `token`, `refreshToken`, `user`, `sessionExpiresAt`, `tenantAssignments`, `activeTenantId`, `adminTenantOverride`, `remsActingForUserId`, `leftDrawerOpen`, `leftDrawerMini`, `appMenuCollapsed`, `appDrawerWidth`, `appFilterDrawerWidth`, `filters:<pageKey>`, `Login` (remember-me email) | Wiped by `clearSession()` (`LocalStorage.clear()`). |
| Browser `localStorage` | `branding`, `last_route` | Survive sign-out on purpose. |
| Cookies (`usePreferences(pageKey)`) | `pref:<pageKey>` JSON, 365 days, `sameSite: Lax` | Page size, sort, visible columns, column order, column widths, form drafts, dashboard date range, seen widgets, sticky-note hideAll. |
| sessionStorage | `microsoftLogin` | Per-tab SSO state. |

## 12. Document title and favicon

`boot/title.js`: `document.title = meta.title ? "<title> - <appName>" : appName`, re-rendered when the
branded app name arrives. Favicons are the `<link rel=icon>` tags in `index.html`; branding swaps their
`href` to the tenant's uploaded icon and restores the stock ones on reset.

## 13. Feature switches

`shared/features.js` exports booleans for built-but-hidden features (`STICKY_NOTES_ENABLED = false`).
Menu items use `hidden: !FLAG`, layouts use `v-if="FLAG"`, and the admin page/route/API stay untouched so
flipping the flag is the whole change.
