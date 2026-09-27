# Validation status

- Next.js production build: passed, including TypeScript and route generation.
- Embedded PostgreSQL integration test: passed, using real application route handlers and SQL with mocked identity and routing boundaries.
- Verified signup role immutability, operator denial, driver approval, class matching, ride privacy, acceptance conflict, OTP wrong-attempt lockout, OTP consumption, completion/rating, event counts, auto fare, decline idempotence, expiry, transaction rollback and database browser-access restrictions.
- Redirect allowlist tests: passed.

Not yet verified: real Supabase signup/confirmation/recovery delivery, a remote Supabase database connection, Vercel deployment, browser/device GPS, or live map provider availability. These require deployment configuration and the live tests in README.md.

This is a separate deployment package. The original Sites project has not been changed. No existing users or trips are migrated.
