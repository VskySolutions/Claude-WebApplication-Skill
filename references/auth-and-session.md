# Authentication screens and session behaviour

Source: `modules/auth/*`, `modules/account/*`, `components/account/ChangePasswordForm.vue`,
`stores/auth.js`, `stores/tenant.js`, `boot/axios.js`, `boot/interceptors.js`, `composables/useSessionExpiry.js`.

## Contents
1. Sign-in card
2. Forgot / reset password
3. SSO callback
4. Change password (and the forced change)
5. Account pages
6. Session rules the UI relies on
7. Request headers

---

## 1. Sign-in card

Rendered inside `auth_layout.vue` (centred 420px column). `q-card flat bordered class="auth-card q-pa-lg"`
(16px radius, full width of the column):

1. Branded copy: `text-h5 text-weight-bold` headline (default "Welcome back 👋") and `text-body2 text-grey-7`
   subtext, both from the branding store.
2. An error banner (`bg-red-1 text-negative`, `o_error`) shown only when there is an error; it is the one
   place sign-in failures are reported (no field hints: "Invalid email or password.", "Your account is
   disabled. Please contact your administrator.", "Unable to sign in right now. Please try again."); SSO
   return errors map `?ssoError=` codes to sentences and the query is removed so a refresh does not repeat it.
3. `q-form greedy @submit.prevent.stop`: `AppTextField type="email" label="Email" maxlength="128" autofocus`
   with an `o_mail` prepend; `AppPasswordField label="Password" maxlength="28" autocomplete="off"` with an
   `o_lock` prepend; vuelidate errors on blur.
4. A row: `q-checkbox dense label="Remember me" color="primary"` (stores the email only, never the password)
   and a flat dense "Forgot password?" link button.
5. Full-width `q-btn label="Login" type="submit" color="primary" unelevated no-caps size="md" :loading`.
6. An "or" divider (two `q-separator`s with `text-caption text-grey-6` between).
7. Full-width outlined `grey-8` "Login with Microsoft" with an inline 18px four-square SVG; it navigates
   away (spinner stays until the page is gone).

After login: `mustChangePassword` → change-password page; otherwise the `redirect` query (internal paths
only) or the landing path.

## 2. Forgot / reset password

- **Forgot**: same card; `text-h5` "Forgot your password?", explanation, an `AppTextField type="email"
  required autocomplete="on"`, full-width "Send reset link"; on submit the form is replaced by a
  `bg-green-1 text-green-9` banner (`o_mark_email_read`) "If that email address has an account, a reset link
  is on its way. The link expires in 60 minutes." — shown for **any** address (never confirm which accounts
  exist). Footer: flat "Back to sign in" with `o_arrow_back`.
- **Reset**: without a token, a red banner and a "Request a new link" button; with one, "Choose a new
  password", two password fields with the strength rules as `:rules`, the requirements banner (`bg-grey-2
  text-grey-8`, `o_info`, bullet list: 8+ chars, lowercase, uppercase, number and special), full-width "Set
  new password"; success toasts and routes to login.

Password rules used everywhere: ≥ 8 characters, a lowercase, an uppercase, a digit, a special from `#?!@$%^&*-`.

## 3. SSO callback

A card with `q-spinner-dots color="primary" size="40px"` and "Signing you in with Microsoft…", or (on
failure) a title, a red banner with the message and a full-width "Back to sign in". The state echoed back
must match the one this tab left with (sessionStorage), otherwise the trip is refused.

## 4. Change password

`ChangePasswordForm` (props `cancelTo`, `submitLabel = "Set New Password"`, `autofocus`): Current / New /
Confirm `AppPasswordField`s with `o_lock` / `o_lock_reset` prepends, the requirements banner, a footer with
optional Cancel (flat grey-8, `:to`) and the primary submit; vuelidate with "New password and confirmation
do not match" on the confirm field. On success: toast "Your password has been changed. Please sign in
again.", clear the session (the server bumped the token version) and route to login.

The page `change_password.vue` centres a 560px column (`text-h5 text-weight-bold` title, a card with
`o_lock` + "Update your password" head) and, when the change is **forced**, blocks navigating away with
`onBeforeRouteLeave` + `notifyWarning("Please set a new password before continuing.")`; the form's
`changed` event clears the guard before the redirect.

## 5. Account pages

- **My Account** (`/account`): `AppDetailHeader`; a summary card (92px avatar with image or initials in
  `primary`, `text-h6 text-weight-bold` name + optional "Verified" badge, email, distinct role chips in
  `primary`), side column with "Edit Profile" (filled) and "Change Password" (outlined); then two half-width
  cards: Contact (`q-list` of icon · caption label · value rows) and Tenants & roles (count badge, list with
  teal-1 avatars and role badges).
- **My Profile**: profile picture card (`AppImageUpload`), personal details (name block), address
  (`AppAddressFields`), a right-aligned "Save profile", then read-only cards: tenant assignments, effective
  permissions grouped Role → Direct / Via group (teal-1 badges with the raw key on tooltip), an embedded
  change-password form, and the record audit.

## 6. Session rules the UI relies on

- Tokens live in Quasar LocalStorage (`token`, `refreshToken`, `sessionExpiresAt`); permissions are decoded
  from the JWT on every token change; another tab's newer tokens are adopted (`_hydrateFromStorage`).
- `ensureSession()` runs before every protected route: an expired token is refreshed first; a dead refresh
  token clears the session; an unreachable API keeps the tokens and lets the page report.
- The authenticated axios instance retries a 401 once after a **single-flight** refresh; a 401/403 from the
  refresh itself clears the session and routes to login with `?redirect=`.
- Session expiry dialog 5 minutes before the refresh window closes ("Extend" / "Logout"), checked every 30s;
  auto-logout at zero.
- Logout clears LocalStorage entirely, the tenant store, and dispatches `session-cleared`; "Logout all
  devices" revokes every session server-side first.
- Switching tenant swaps the access token only (the refresh token and its clock are kept), reloads the
  profile, and fires `tenant-switched`; a dirty form prompts "Switching tenant will discard your unsaved
  changes. Continue?".

## 7. Request headers

Set by the authenticated instance on every call: `Authorization: Bearer <token>`, `X-Correlation-Id` (a
fresh UUID per request), `X-Site-Id` (the super-admin override or the user's tenant), `X-Site-Name`,
`X-Site-LandingPage`, `X-Site-Timezone` when known, and feature-specific acting-as headers. The anonymous
instance (`http2`) sends none and is used for login, refresh, forgot/reset, public branding and public forms.
