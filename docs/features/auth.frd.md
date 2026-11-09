# FRD: Authentication

**PRD refs:** FR-25, §11.1 · **Milestone:** 1

## Purpose
Protect the app behind a single admin login. The app holds every talent's earnings, bank details and advances, so the baseline below is required, not optional.

## Requirements
- Email and password login page.
- One admin account, created by seed script (no sign-up page).
- Session cookie (HTTP-only, secure, same-site); logout button.
- **Every route and every server action** checks the session through one shared helper. Middleware alone is not enough. Unauthenticated users are redirected to `/login`.
- Passwords hashed (argon2 or bcrypt).
- WhatsApp actions are authorised by a verified, allowlisted sender number through the same shared helper (whatsapp FRD); no other channel can reach a money action.
- **Login rate limiting** (per IP and per account) with a generic error and a short lock-out after repeated failures.
- **Session expiry:** idle timeout and an absolute maximum lifetime; expired sessions redirect to login.
- Authenticated responses send `Cache-Control: no-store`.
- **Admin password reset** by a documented script run by the operator (there is no email reset).

## Out of scope
Multiple users, roles, password reset by email, talent or brand logins.

## Acceptance criteria
- Visiting any page logged out redirects to login.
- Calling a server action with no session is rejected, whatever the UI shows.
- Wrong credentials show a generic error; repeated failures are rate limited.
- After logout, the back button does not reveal protected pages.
- A session past its idle or absolute limit is rejected.
- Running the reset script sets a new password and invalidates existing sessions.
