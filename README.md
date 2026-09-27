# Roam — Vercel + Supabase edition

This separate deployment replaces Sites identity and D1 with Supabase Auth and PostgreSQL. It preserves rider, driver and operator portals, vehicle photos (including bike and auto), shared dispatch, pickup OTP, real map tiles, address search and road routes. Existing Sites users and trip data are not migrated. People create new accounts.

## 1. Create database tables

In your existing Supabase project, open **SQL Editor → New query**. Paste the full contents of `supabase/schema.sql` and run it once. It creates a private `roam` schema. Do not add `roam` to the Data API exposed schemas or grant browser users access. All database access goes through server routes with verified Supabase identity and role checks. RLS is enabled with no browser policies.

## 2. Upload this project to GitHub

Create a private repository named `roam`. Extract the ZIP and upload the contents of the `roam-vercel` folder, so `package.json` is at the repository root. Include `pnpm-lock.yaml` and `supabase/schema.sql`. Do not upload `.env.local`, database passwords, node_modules or build folders. This project does not include any secret keys.

## 3. Import into Vercel

Vercel → Add New → Project → import `roam`. Framework: **Next.js**. Root Directory: repository root. Keep the normal `pnpm build` build command; this edition no longer uses Vinext or Wrangler. Use Node.js 22 or newer supported by Vercel.

Set these Environment Variables before deploying:

| Name | Value |
| --- | --- |
| NEXT_PUBLIC_SUPABASE_URL | https://wecbgfytvhkmhyecouix.supabase.co |
| NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY | Your existing `sb_publishable_...` key from Supabase API Keys |
| DATABASE_URL | Supabase Connect → Transaction pooler → PostgreSQL URI, with the database password substituted and URL-encoded |
| OPERATOR_USER_ID | Leave unset initially; set to the owner's Supabase Auth UUID in step 5 |

`DATABASE_URL` is a **server-only secret**, never prefix it with NEXT_PUBLIC. The database password is the password from project creation, NOT Gmail's app password. Enter it directly in Vercel, never in chat or GitHub. Keep SSL certificate verification enabled (`sslmode=verify-full`). If your connection needs Supabase's CA certificate, configure the trusted CA rather than disabling verification. Use the exact pooler hostname Supabase displays; do not guess it. Transaction pooling is suitable for this serverless app; queries are unnamed and use short transactions.

Click Deploy. This archive has not been deployed to your Vercel account; a successful local build does not verify your remote credentials.

## 4. Configure email links

After Vercel gives you the live URL, in Supabase Authentication → URL Configuration:

- **Site URL**: your actual Vercel URL (not `roam.com` unless you own and connect it).
- **Redirect URLs**: add `https://YOUR-VERCEL-HOST/auth/callback` and `https://YOUR-VERCEL-HOST/auth/callback?next=**` (or the single scoped pattern `https://YOUR-VERCEL-HOST/auth/callback**`). Restrict it to your own exact hostname. For localhost testing only, add the corresponding `http://localhost:3000/auth/callback**`.
- Keep email confirmation enabled, signups enabled, and your saved SMTP setup.

Signup uses Supabase's default confirmation template and PKCE callback. Open confirmation/recovery links in the same browser used to request them. If an email confirmation opens another browser, confirm the email then return to the original browser's login page. Request a new recovery link if needed. SMTP settings being saved is not proof of delivery; test inbox and spam folders.

## 5. Assign the operator

Open `https://YOUR-VERCEL-HOST/login?next=/operator`. Create your own email account and confirm it. In Supabase Authentication → Users, copy that user's **UID**. Set Vercel `OPERATOR_USER_ID` to that UID and redeploy. The operator does not register as a rider/driver; keep this account separate. There is no first-user admin rule and no public role escalation. Without this environment variable, nobody has operator access.

## 6. Test all three portals

Use different email addresses and separate browser profiles/devices:

1. `/driver`: sign up, confirm email, register name/plate/vehicle.
2. `/operator`: approve the driver.
3. Driver: go online and allow GPS; keep the page open.
4. `/rider`: sign up with another email, register as a rider, choose pickup near driver and matching vehicle class, book.
5. Driver: accept, navigate to pickup, mark arrived, enter the rider's four-digit pickup OTP, then complete.
6. Rider: rate the completed trip. Operator: verify status events and estimated fare.
7. Test sign out, wrong password, password recovery, and verify a rider cannot open operator data.

All three portals use one Vercel deployment and one shared database. Status polling is every seven seconds. Driver heartbeat expires after 90 seconds. Request expiry is ten minutes. Unique PostgreSQL indexes and conditional updates prevent duplicate acceptance/active rides. The pickup OTP is separate from email verification and is consumed when a ride starts.

## Local development

Copy `.env.example` to `.env.local`, enter values privately, then run `pnpm install`, `pnpm dev`. Run `pnpm test` for PostgreSQL behavior tests and `pnpm build` for the production build.

## Limits

This is a testing pilot, not a finished commercial transport platform. There is no online payment processing, payout, background mobile GPS, native Android/iOS app, emergency service, document verification or fraud detection engine. Phone login is not enabled: it requires a configured SMS provider and a separate integration. Email SMTP via personal Gmail is for initial low-volume testing; use a transactional email provider for a public rollout.

Maps use OpenStreetMap tiles, Photon and OSRM public endpoints. Search/routing are cached and globally rate limited. Bike/auto fare routes use car-road geometry and may include restricted roads; use appropriate licensed vehicle routing before real operations. No live traffic is provided. Optional OSRM_BASE_URL and PHOTON_BASE_URL can target managed/self-hosted endpoints.

## Architecture and security

- Next.js App Router on Vercel Node runtime; Supabase SSR cookie sessions.
- Server identity uses `auth.getUser()`, never browser-provided user IDs or OpenAI forwarded headers.
- Proxy refreshes cookies. POST mutations check Origin. Auth redirects only accept an allowlist of local portal paths.
- PostgreSQL is accessible only from server code using DATABASE_URL; private `roam` schema, RLS and revoked anonymous/authenticated grants prevent browser Data API reads.
- The database adapter binds all values and executes batches atomically; an event is written only when its conditional state update succeeded.
- OPERATOR_USER_ID is the exact Supabase UUID, not a user-editable field.
- GET logout shows a confirmation form; actual logout is POST.

Official references: https://supabase.com/docs/guides/auth/server-side/creating-a-client and https://supabase.com/docs/guides/database/connecting-to-postgres
