# Auth and Proxy Reference

## End-User Authentication

Forte authenticates end-users within your projects. Three methods are **live**:

- **Google OAuth** — users sign in with Google; contact method is verified by default.
- **OTP login** — passwordless 6-digit code over email or SMS.
- **Password login** — optional password alongside OTP/OAuth, used with any verified contact method.

All three are part of the **client-side API** (`forte.users.*`) — call them from the frontend, where
Forte sets the `Forte-User-Session-Token` cookie on success. **Never** call them from code that holds
`FORTE_API_TOKEN` (see `api-surfaces.md`). OTP and password login are enabled per project in settings
(`emailLoginEnabled` / `phoneLoginEnabled` for OTP; a Password-login toggle with strength rules).

On top of any first factor, a project can require **[multi-factor authentication](#multi-factor-authentication)** —
an optional second factor configured per project.

### Google OAuth

**Setup checklist:**
1. Create a Google OAuth Client ID at [console.cloud.google.com](https://console.cloud.google.com) for your domain(s).
2. Link it to your Forte project: `forte projects update <projectId> --google-oauth-client-id <clientId>`
3. Render the standard Google Sign-In button in your frontend.
4. On callback, call the Forte SDK to complete login (see below).

Canonical doc: [forteplatforms.com/docs/users/authentication](https://forteplatforms.com/docs/users/authentication)

### Google OAuth Login — SDK Calls

**TypeScript (browser)**
```typescript
import { ForteClient } from '@forteplatforms/sdk';

// No API token for browser-side user-login calls
const forte = new ForteClient();

const { userObject, sessionToken } = await forte.users.googleAuthLoginCallback({
  projectId: 'your-project-id',
  gCsrfToken: gCsrfToken,         // from Google Sign-In response
  credential: credential,          // from Google Sign-In response
  recaptchaToken: recaptchaToken,  // optional — see reCAPTCHA section below
});
// sessionToken.sessionToken — store this or use the cookie Forte sets automatically
```

**Python**
```python
from forte_sdk import ForteClient

forte = ForteClient()
response = forte.users.google_auth_login_callback(
    project_id="your-project-id",
    g_csrf_token=g_csrf_token,
    credential=credential,
)
session_token = response.session_token.session_token
```

### OTP Login (email or SMS)

Passwordless 6-digit code. Same API for both channels — the request's `email` or `phoneNumber`
field selects which (exactly one required, else `NO_CONTACT_METHOD_PROVIDED`). Codes: 10-minute
expiry, single-use, 5-attempt limit.

```typescript
const forte = new ForteClient();

// 1. Request a code
const { pendingLoginId, expirationTime } = await forte.users.createOtpLogin({
  projectId,
  createOtpLoginRequest: { email: "alice@example.com" /* or phoneNumber: "+15551234567" */ },
});

// 2. Verify the code the user entered
const result = await forte.users.completeOtpLogin({
  projectId,
  pendingLoginId,
  completeOtpLoginRequest: { code: "123456" },
});
// result.sessionToken.sessionToken — also set as the session cookie

// 3. (optional) Resend — throttled to once / 60s, does NOT extend the 10-min window
await forte.users.resendLoginOtp({ projectId, pendingLoginId });
```

Notes:
- If no user with that verified contact exists, `createOtpLogin` returns the **same** response shape
  but sends nothing (prevents account enumeration).
- Wrong/expired/used code → `400 INVALID_VERIFICATION_CODE`; after 5 failures the pending login locks.
- Resend within 60s → `429 VERIFICATION_CODE_RATE_LIMITED`.
- **SMS compliance:** your UI must show "By continuing, you agree to receive a one-time passcode via
  SMS" before sending an SMS OTP.
- **Sandbox fixed OTPs:** in sandbox projects, a contact method can carry a fixed test code (see
  "Fixed Test OTP Codes (sandbox)" below). Login OTPs for that contact then use the fixed code and
  nothing is delivered — the flow above is otherwise identical.

### Password Login

Optional. Users set a password at sign-up (or later) and sign in with any **verified** contact value
plus the password. OTP stays available as a recovery path.

```typescript
// Sign up with a password
await forte.users.register({ projectId, email: "alice@example.com", password: "..." });

// Sign in — contactValue is auto-detected as email or phone; must be verified
const result = await forte.users.passwordLogin({
  projectId,
  passwordLoginRequest: { contactValue: "alice@example.com", password: "..." },
});

// Self-service change (revokes the user's other sessions; currentPassword optional for a signed-in user, validated if provided)
await forte.users.changePassword({ projectId, changePasswordRequest: { currentPassword, newPassword } });

// Reset: request always returns 204 (anti-enumeration). Two project modes:
//   "send new password" → Forte sets & sends a generated password
//   "send reset link"   → link carries ?pwdResetToken=<token> (30-min expiry)
await forte.users.requestPasswordReset({ projectId, requestPasswordResetRequest: { contactValue } });
await forte.users.completePasswordReset({ projectId, completePasswordResetRequest: { token, newPassword } });
```

Key error codes: `PASSWORD_LOGIN_NOT_ENABLED`, `PASSWORD_TOO_WEAK`, `INVALID_CREDENTIALS` (wrong
password / no user / unverified contact — same error for all three), `INVALID_RESET_TOKEN`,
`THROTTLED`. Strength floor is 6 chars (ceiling 128); project rules apply on top.

Canonical docs: [authentication](https://forteplatforms.com/docs/users/authentication) ·
[passwords](https://forteplatforms.com/docs/users/passwords)

---

## Multi-Factor Authentication

MFA adds a **second factor** on top of any first factor (Google / OTP / password), configured **per project**
(`mfaConfig`: an enforcement mode `DISABLED` / `OPTIONAL` / `REQUIRED`, plus toggles for TOTP, email OTP, SMS OTP,
and WebAuthn). Projects with MFA off behave exactly as before.

- Every first-factor login response (`googleAuthLoginCallback`, `registerUser`, `completeOtpLogin`, `passwordLogin`,
  password reset) now carries an optional **`mfaStatus`**: `SATISFIED` (full session), `CHALLENGE_REQUIRED`, or
  `ENROLLMENT_REQUIRED`. The latter two return a **short-lived pending token** that only works on the MFA endpoints +
  logout and is rejected everywhere else (including the deployed app) with `401 MFA_REQUIRED`. It is not renewable.
  A pending response carries the top-level `userId` but **no `userObject`** — the full user object arrives from
  `verifyMfa` once the second factor is satisfied.
- Branch on `mfaStatus`. For email/SMS OTP and WebAuthn, call `forte.users.sendMfaChallenge({ type })` then
  `forte.users.verifyMfa({ type, code | webAuthnAssertion })`; for authenticator-app (TOTP) and backup codes call
  `verifyMfa` directly. On success Forte swaps the pending token for a full session. Browser clients ride the
  `Forte-User-Session-Token` cookie automatically; non-cookie clients (mobile/BFF) read the pending token from the
  login response (`login.sessionToken.sessionToken`) and pass it as the `authorization` argument on each MFA call
  (`sendMfaChallenge`, `verifyMfa`, enrollment), the same way `renewSessionToken`/`logout` take it.
- **Factors**: authenticator app (TOTP), passkeys/security keys (WebAuthn — Forte is the relying party; a credential is
  bound to the exact host it was registered on, so it only works on that domain), email/SMS OTP (ride the user's
  verified contact), and one-time **backup codes** (`generateBackupCodes`, shown once).
- **Enroll**: `createMfaMethod({ type })` → `activateMfaMethod` (TOTP code or WebAuthn attestation). Manage with
  `listMfaMethods` / `renameMfaMethod` / `deleteMfaMethod`. Admin reset (`adminResetUserMfa`) is **sandbox-only**.
- **Testing in sandbox**: a contact method with a fixed test OTP (see "Fixed Test OTP Codes (sandbox)" below) uses
  that code for email/SMS MFA challenges too — `sendMfaChallenge` delivers nothing and `verifyMfa` accepts the
  fixed code.

Canonical doc: [forteplatforms.com/docs/users/mfa](https://forteplatforms.com/docs/users/mfa)

## Sessions

Session tokens are opaque (cannot be decoded) and valid for **365 days** by default. A project with MFA enabled may
instead return a **pending** session token from a first-factor login (`mfaStatus` ≠ `SATISFIED`): short-lived (~10 min),
usable only on the MFA endpoints + logout, rejected elsewhere with `401 MFA_REQUIRED`, and not renewable. See
[Multi-Factor Authentication](#multi-factor-authentication).

**Passing the token**: Forte automatically sets a `Forte-User-Session-Token` cookie. You can also pass it as a header: `Authorization: Bearer <sessionToken>`.

**Renew**:
```typescript
await forte.users.renewSessionToken({ projectId, authorization, renewalDurationSeconds });
```

**Logout**:
```typescript
await forte.users.logout({ projectId, authorization });
```

Canonical doc: [forteplatforms.com/docs/users/sessions](https://forteplatforms.com/docs/users/sessions)

---

## Same-Site Session Cookies on Services — the `/_forte` Path

When your **service's** browser frontend calls the user/session APIs (`forte.users.*` — login, OTP,
password, session renewal, logout) directly from the browser, the `Forte-User-Session-Token` cookie
must be set and sent **first-party** on your service's own domain. The cookie is `SameSite=Lax`, and
the SDK's default base URL points at Forte's API on a different domain — so a cookie set there would
never ride along on your frontend's `fetch`/`XHR` calls.

Every Forte **service** reserves the path prefix `/_forte` on its own domain. Requests to:

```
/_forte/api/v1/<projectId>/users/...
```

are routed through Forte's user/session API instead of your application code — and because they hit
your service's *own* origin, the session cookie stays first-party (same-site).

Point the browser SDK at this path so it issues same-site requests:

```typescript
import { ForteClient } from '@forteplatforms/sdk';

// In the browser, on your service's own domain. baseUrl '/_forte' makes the SDK call
// /_forte/api/v1/<projectId>/users/... on the current origin, so the Forte-User-Session-Token
// cookie is set and sent first-party (same-site).
const forte = new ForteClient({ baseUrl: '/_forte' });

await forte.users.completeOtpLogin({ projectId, pendingLoginId, completeOtpLoginRequest: { code } });
```

Notes:
- `/_forte` exposes the `forte.users.*` (user/session) surface **only** — the browser only ever holds
  a `Forte-User-Session-Token`, so nothing else is reachable through it.
- **Services only.** Websites are served directly and never pass through the Forte gateway, so
  `/_forte` is not available on a website domain — see the SSR note in `setup-walkthrough.md`.
- The server-side API (`forte.projects.*`, authenticated with `FORTE_API_TOKEN`) does **not** use
  `/_forte`; it calls Forte directly from your backend.

---

## Reading the Authenticated User in Your Service

Every authenticated request that arrives at your Forte service includes an `X-Forte-User-Id` header. Use it to identify the user:

**TypeScript (Express / any Node.js)**
```typescript
import { ForteClient } from '@forteplatforms/sdk';

const forte = new ForteClient(); // reads FORTE_API_TOKEN automatically

app.get('/api/me', async (req, res) => {
  const userId = req.headers['x-forte-user-id'] as string;
  const user = await forte.projects.getProjectUser({
    projectId: process.env.FORTE_PROJECT_ID!,
    userId,
  });
  res.json(user);
});
```

`FORTE_API_TOKEN`, `FORTE_PROJECT_ID`, and `FORTE_SERVICE_ID` are injected automatically — do not set them manually.

---

## Auth Exclusions

Routes that should skip user auth (health checks, webhooks, public APIs). Uses Ant-style patterns:

```bash
forte services update <projectId> <serviceId> \
  --auth-exclude /health \
  --auth-exclude '/api/webhooks/**' \
  --auth-exclude '/public/*'
```

Pattern rules: `?` matches one char, `*` matches within a path segment, `**` matches across segments.

---

## Local Development with forte proxy

**Applies to services only.** Websites are public-by-default front-ends served on a CDN — they don't go through Forte authentication, so `forte proxy` doesn't apply. To preview a website locally, just run its native dev command (`npm run dev`, etc.) and point your browser at the local server directly.

`forte proxy` runs a local reverse proxy that authenticates requests before forwarding them to your local dev server. Real Forte sessions and cookies work end-to-end against local code.

```bash
# Terminal 1 — your app (example: Next.js / Express)
npm run dev                    # starts on localhost:3000

# Terminal 2 — forte proxy
forte proxy \
  --project-id <id> \
  --service-id <id> \
  -p 3000                      # your local dev server's port
                               # proxy listens on localhost:8080 by default
```

`-p/--port` is just **where your local dev server is running** so the proxy knows where to forward — it has nothing to do with how Forte health-checks the deployed service (that port is detected from your code). If you omit `-p`, the proxy forwards to the service's detected port and only falls back to `3000` if that's unknown.

Point your frontend at `http://localhost:8080`. The proxy also exposes your LAN IP so a phone on the same Wi-Fi can test mobile flows.

**What happens on each request:**
1. Request arrives at `localhost:8080`
2. Proxy sends path, method, and headers to Forte for authorization
3. Authorized → request forwarded to `localhost:3000` with `X-Forte-User-Id` (and any other auth headers) added
4. Not authorized → 401 or 403 returned directly (your app never sees the request)

The proxy terminal shows live request logs: timestamp, ✓/✗ auth result, method, path, status.

Canonical guide: [forteplatforms.com/docs/guides/testing-locally](https://forteplatforms.com/docs/guides/testing-locally)

---

## reCAPTCHA (optional bot protection)

1. Enable reCAPTCHA v3 in the Forte console under Project Settings and enter your secret key.
2. Add `<script src="https://www.google.com/recaptcha/api.js?render=SITE_KEY"></script>` to your frontend.
3. Call `grecaptcha.execute(siteKey, { action: 'login' })` before the login SDK call.
4. Pass the result as `recaptchaToken` in `googleAuthLoginCallback`.
5. Failed validation returns error code `RECAPTCHA_VALIDATION_FAILED` (HTTP 400).

---

## Contact Method Verification

After a user adds an email or phone, Forte requires verification before that contact method can be used for OTP or password login. Google OAuth contact methods are verified by default. See [forteplatforms.com/docs/users/contact-methods](https://forteplatforms.com/docs/users/contact-methods).

Verification codes: 6 digits, 10-minute expiry, 60-second resend cooldown.

### Fixed Test OTP Codes (sandbox)

Sandbox projects can pin a **fixed 6-digit code** on a contact method so automated tests complete OTP flows deterministically. While set, **every** OTP for that contact method — contact verification, OTP login, and email/SMS MFA challenges — uses the fixed code, and **nothing is delivered**. Expiry, resend cooldowns, and attempt limits behave exactly as in production, and the test must still trigger the normal send/resend call to arm the code.

- **Eligibility (server-enforced):** US phone numbers with a `555` exchange (`+1XXX555XXXX`), or emails at reserved test domains (`example.com`/`.net`/`.org`, or any `.test`/`.example`/`.invalid` domain). Anything else → `400 CONTACT_METHOD_NOT_ELIGIBLE_FOR_FIXED_CODE`; non-sandbox projects → `400 SANDBOX_MODE_REQUIRED`.
- **Assign / remove (server-side API or Console user page → contact method menu):** `PATCH /{projectId}/users/{userId}/contact-methods/{contactMethodId}` with `{ "fixedVerificationCode": "123456" }` or `{ "removeFixedVerificationCode": true }` (admin `adminOverrideUserContactMethod`, `FORTE_API_TOKEN` auth).
- **Read back:** the code is visible as `fixedVerificationCode` on `ContactMethod` responses. Assignments are audited (`CONTACT_METHOD_FIXED_OTP_SET` / `CONTACT_METHOD_FIXED_OTP_REMOVED` action-log entries).
- Verifying the contact does **not** clear the fixed code — it keeps working for subsequent login and MFA flows.

Canonical doc: [forteplatforms.com/docs/users/contact-methods](https://forteplatforms.com/docs/users/contact-methods)

### Unverified Contact Method Reservations and Reclaim

When a user adds an email or phone but never enters the verification code, the identifier is **reserved** to that user within the project. Other users attempting to register — or to add the same email/phone to an existing account — get `USER_ALREADY_EXISTS` (HTTP 409) while the reservation is active.

The reservation is **time-bound**, not permanent. Once the 10-minute verification code window has elapsed without the user completing verification or requesting a new code, the unverified contact method is **stale** and a different user can claim it.

**What happens on claim:**

1. The stale entry is removed from the original user.
2. If that was the original user's only contact method, the original user is removed from the project entirely (they had no verified way to authenticate). Any session token issued to that user during their incomplete registration is invalidated.
3. The new caller's request proceeds as if the identifier had been free. A fresh verification code is sent to the new caller.
4. An audit log entry is recorded against the original user noting that the contact method was reclaimed.
5. When the new caller eventually verifies the reclaimed identifier, every other outstanding session for the new caller's account is also invalidated. This protects the account against any stale sessions left over from prior unverified registration attempts on the same identifier.

**Reclaim never applies to verified contact methods** — those permanently own the identifier within the project.

`USER_ALREADY_EXISTS` (HTTP 409) is still returned in two cases the skill should be ready to explain:

- The other user owns the identifier as a **verified** contact method (resolution: the original owner must remove it themselves).
- The other user is **mid-verification** — they added the identifier within the last 10 minutes and the code has not yet expired (resolution: wait for the window to elapse, then retry).

Canonical doc: [forteplatforms.com/docs/users/contact-methods](https://forteplatforms.com/docs/users/contact-methods)
