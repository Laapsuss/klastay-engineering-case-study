# Technical challenges

Real problems from building Klastay, written at a level of detail that does not expose private code or security specifics.

## 1. A mobile client on a cookie-authenticated API

**Problem.** The web API was built for the browser: almost every route read the Supabase session from a cookie. A native app has no cookie, so checkout, identity verification, translation, calendar import and other routes returned 401 to the app.

**Approach.** Prefer direct RPC calls with the user's JWT wherever row-level security already protects the data; for the routes that really need the server, accept an `Authorization: Bearer` token as an alternative to the cookie, resolved to the same user and the same permissions. Each route was migrated in its own small PR (checkout, verification, calendar, import, support attachments), so the web behaviour could be verified unchanged after each one.

## 2. Hosts stuck in a loop after publishing a listing

**Problem.** Testers on Android reported that after adding a listing, the app asked them again to "publish your first listing" and still showed "become a host".

**Root cause.** The database deliberately keeps a listing as a hidden draft until the host has verified their identity and accepted the host charter. The app had neither step, so the listing was saved but never published, and the "has a listing" flag was only computed for users who were already hosts.

**Fix.** A single server-side access check (`what does this user still need to become a host?`) drives one onboarding card: listing → identity (KYC in a browser tab that returns to the app) → charter → publish. Publishing activates the draft and the existing database trigger promotes the user to host. Verified end to end on the emulator with a fresh account on staging; the database rules were not touched.

## 3. Native push notifications

**Problem.** Notifications reached users through Web Push only; nothing arrived on a closed mobile app.

**Approach.** A device-token table per user and platform, registration from the app on login, and server-side sending through FCM HTTP v1 for Android (APNs for iOS). FCM's OAuth token is obtained by signing a service-account JWT (RS256) with Node's built-in crypto — no extra SDK in the server bundle. The same notification event fans out to Web Push, FCM and APNs. Covered by unit tests on message building and signing.

## 4. Payments that cannot misfire

**Problem.** During development the checkout ran against a sandbox and a simulator. The risk was a production deployment that silently kept simulating or used test keys.

**Approach.** In production the server refuses to start a payment unless live gateway keys are configured — no simulator, no sandbox, no test secrets — and a dedicated test asserts this. Prices are always quoted server-side; the client never sends an amount the server trusts.

## 5. Privacy built into the data model

- Other users' email, phone and bank details are not selectable through RLS. Contact details are released only for a paid booking, through a dedicated RPC.
- Listing maps show an approximate zone (radius) — exact address and access codes appear only in a confirmed booking.
- Travellers are never asked for a phone number; hosts verify theirs through WhatsApp.
- Data stays in EU infrastructure; data-export and retention jobs exist for GDPR / Moroccan law 09-08 requests.

## 6. App Store requirements for user-generated content

**Problem.** Apple requires apps with user-generated content to let users report content, block abusive users and delete their account in-app.

**Approach.** Reporting a listing, blocking a host or a guest (enforced in both directions in the database, so a blocked user can neither send nor receive messages), visible account deletion, and a public account-deletion page for the stores. Implemented across the database (new migrations), the web chat and the mobile app, tested on staging, then applied to production.

## 7. Making CI builds behave like local builds

Examples of issues that only appeared in CI-built binaries: a Maps API key that existed locally but not in the CI environment group (empty map in store builds), signing and flavour configuration for two iOS apps (development and production bundle IDs), and production builds needing their own OAuth and Maps credentials tied to the Play app-signing certificate. Each was found by installing the CI artifact on a device or emulator, not by trusting a green build.
