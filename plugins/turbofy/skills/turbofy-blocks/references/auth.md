# App authentication

Use the helpers from `@/lib/auth` for app end users. The platform manages the session; block code does not need to read, store, or refresh tokens itself. Console sign-in is separate from app sign-in.

Enable the workspace's App Users (AUTH) feature and the app's `auth.enabled` setting. Configure pages and redirects with `turbofy-apps`.

## Built-in forms

- **Login** provides email/password sign-in. A blank or `#` `forgotPassword.href` opens built-in recovery; a configured link opens a custom recovery page. Login hides while signed in.
- **Signup** provides registration and email confirmation. Enable `auth.allowSignup` when offering registration.
- **Account** shows the signed-in email, sign-out, and password change controls. It hides while signed out, so Account and Login can share a public page.

Reuse an existing system block when it fits. For custom forms or social buttons, use the helpers below. Put labels and error messages in the block's `localizations` and read them from `config.copies`.

## Helpers

| Task | Call |
|---|---|
| Sign in | `login(email, password)` |
| Register | `signup({ email, password, name? })` |
| Confirm email | `confirmSignup(email, code, password?)` |
| Request recovery code | `forgotPassword(email)` |
| Set a new password with a recovery code | `resetPassword(email, code, newPassword)` |
| Change password while signed in | `changePassword(currentPassword, newPassword)` |
| Read the current user once | `currentUser()` → `{ user, error? }` |
| Follow session changes in React | `useCurrentUser()` → `{ user, isLoading, error, refetch }` |
| Sign out | `logout()` |
| Navigate after successful sign-in | `redirectAfterAuth(next)` |

Auth mutations return `{ ok, error?, needsConfirmation?, signedIn?, signInError? }`; relevant fields depend on the operation. `user` is `null` when signed out, otherwise `{ sub, email?, groups? }`.

For signup, check `needsConfirmation`. Confirmation and password reset can succeed without signing in: `ok: true, signedIn: false` means show a separate sign-in step. Pass the available password to `confirmSignup` to allow automatic sign-in. If an older deployment omits `signedIn`, use a separate `login` when appropriate. Do not submit a successfully consumed confirmation or recovery code again.

After successful login, social sign-in, or `signedIn: true`, call `redirectAfterAuth(searchParams.next)` if navigation is wanted. Auth mutations do not navigate by themselves. Use this helper for the app's configured fallback destination rather than implementing a redirect yourself.

`useCurrentUser` follows successful auth-helper session changes. Call `refetch()` when a session changes outside those helpers. Use it instead of duplicating auth state in a shared store. Hiding UI and protecting routes do not replace record permissions.

## Social sign-in

Configure the desired provider in the workspace's authentication settings, including its credentials and allowed callback URLs. Use the configured provider identifier, such as `"Google"`, `"Facebook"`, or `"SignInWithApple"`. Provider credentials belong in workspace secrets, never in block source.

Call `signInWithSocialProvider` directly from a click handler, before unrelated asynchronous work, so the browser can open its popup:

```tsx
import { redirectAfterAuth, signInWithSocialProvider } from "@/lib/auth";

const result = await signInWithSocialProvider("Google", {
  signal: controller.signal,
  redirectUri: config.socialCallbackUrl,
});
if (result.ok) {
  redirectAfterAuth(searchParams.next);
}
```

Here `controller` is an `AbortController` owned by the component; abort it on unmount. Disable repeated clicks while a sign-in is pending. Handle `result.errorCode` with localized feedback: `NOT_CONFIGURED`, `POPUP_BLOCKED`, `CANCELLED`, `TIMED_OUT`, `IN_PROGRESS`, or `SIGN_IN_FAILED`.

- **Preview/editor:** the helper chooses the console callback automatically. An app does not need to be published to test social sign-in. The workspace's provider configuration must allow `https://cloud.turbofy.com/auth/app-callback` (production) or `https://cloud.alpha.turbofy.com/auth/app-callback` (alpha).
- **Published app:** supply `redirectUri` as an exact allowed URL on the app's origin, and provide a public callback page there. Include the locale prefix if the page's actual URL has one. Preview overrides this value automatically.
- **Callback page:** keep it `visibility: "public"`, including after the user signs in. Use `parseSocialCallback` and `relaySocialCallback` from `@/lib/social-callback`. Parse the callback query once, relay it from an effect, return the relay's cleanup function, and render localized status feedback. The relay handles returning to the opener; do not build a custom message protocol.

A callback component can use this pattern with its query props and status setter:

```tsx
import { useEffect, useState } from "react";
import { parseSocialCallback, relaySocialCallback } from "@/lib/social-callback";

const [callback] = useState(() =>
  parseSocialCallback(new URLSearchParams(searchParams).toString()),
);
useEffect(() => relaySocialCallback(callback, setStatus), [callback]);
```

`setStatus` receives `returning`, `returned`, `invalid`, `missingOpener`, or `timedOut`. If a configured provider or callback is rejected, check workspace authentication settings and whether those changes have been published. Do not change the provider's settings merely to work around a block error.

## Custom OAuth flows only

Prefer `signInWithSocialProvider`. The lower-level helpers are available when the app requires a different flow:

- `getSocialAuthorizeUrl({ provider, redirectUri, codeChallenge })` → `{ ok: true, url, state }` or `{ ok: false, error }`.
- `socialSignIn({ code, state, codeVerifier })` → auth result.

Custom flows must retain their pending PKCE verifier and verify callback state before exchanging the code. `refreshAccessToken()` is available for explicit session refresh, but ordinary forms should use the platform-managed session.
