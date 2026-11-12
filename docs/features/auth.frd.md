# FRD: Authentication

**PRD refs:** FR-25, FR-56 to FR-63, §11.1 · **Milestone:** 1 · **See also:** `team.frd.md` (users, roles, invites, access)

## Purpose
Protect the app behind a sign-in. The app holds every talent's earnings, bank details and advances, so the baseline below is required, not optional. Roles, pages, talent scope and invites are defined in the team FRD; this document covers sign-in and sessions.

## Requirements
- Email and password login page for every user. No public sign-up: accounts exist only by seed (the first admin) or admin invite.
- Session cookie (HTTP-only, secure, same-site); logout button.
- **Every route and every server action** checks the session through one shared helper, which also checks the user's role, pages and assigned talents from the database on every request (never cached in the session). Middleware alone is not enough. Unauthenticated users are redirected to `/login`.
- Passwords hashed (argon2 or bcrypt); minimum 12 characters and a common-password check.
- WhatsApp actions are authorised by a verified, allowlisted sender number linked to a user, through the same shared helper, with that user's access (whatsapp FRD); no other channel can reach a money action.
- **Login rate limiting** (per IP and per account) with a generic error and a short lock-out after repeated failures.
- **Session expiry:** idle timeout and an absolute maximum lifetime; expired sessions redirect to login.
- **Immediate revocation:** deactivating a user, or a password change or reset, ends that user's sessions.
- Authenticated responses send `Cache-Control: no-store`.
- **Password reset by email link** for every user (single-use token valid 1 hour; the response never reveals whether the email is registered). An operator script remains as the last resort for the admin.
- **Invites:** single-use hashed tokens valid for 7 days (team FRD).

## Out of scope
Public sign-up, two-factor authentication and single sign-on (open questions in the team FRD), talent or brand logins.

## Acceptance criteria
- Visiting any page logged out redirects to login.
- Calling a server action with no session is rejected, whatever the UI shows.
- Calling a server action as a member without the page or talent is rejected as "not found" or "no access".
- Wrong credentials show a generic error; repeated failures are rate limited.
- After logout, the back button does not reveal protected pages.
- A session past its idle or absolute limit is rejected.
- A deactivated user's open session fails on its next request.
- A password reset link works once, expires after an hour, and ends other sessions on use.
- The operator reset script sets a new admin password and invalidates existing sessions.
