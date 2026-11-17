# Quality assurance and releases

## Gates before a change can be merged

| Gate | Web | Mobile |
|---|---|---|
| Static analysis | TypeScript compiler, ESLint | `flutter analyze` with `very_good_analysis`, `bloc_lint` |
| Unit / integration tests | Node test runner suites (see below) | 1,000+ unit, widget and bloc tests (`bloc_test`, `mocktail`) |
| Build | `next build` | Android and iOS builds (Codemagic) |
| CI | — | GitHub Actions: semantic PR title, tests with an 80 % coverage gate per package, license check |
| Manual / E2E | Playwright scripts against staging | Android emulator and TestFlight builds against staging |

Agents are required to report the real output of these commands; a change is not reviewed on a claim like "all tests pass".

## Web test suites

Grouped by domain and runnable separately, some against the staging database:

- cancellation engine (pure rules + database integration)
- transactional emails (rendering per language + sending pipeline)
- push notifications (FCM message building and signing)
- WhatsApp phone verification (codes, gate, database)
- host onboarding (database)
- support inbox and attachment policy
- CRM internal operations (signatures, refund execution)
- Stay Together coin sharing

## Playwright checks on staging

Scripts that drive a real browser (Microsoft Edge) against staging, used after each integration and before promotions. Examples:

- authentication: sign-up, email confirmation link, password reset
- WhatsApp OTP phone verification end to end
- live offers board (Live Search requests and host offers)
- booking then cancellation, checkout flow, currency display
- chat with translation and email notification
- host onboarding end to end
- CRM: login guard and session revocation
- authorisation: IDOR probes on API routes and horizontal-access checks between two accounts
- full-site crawl that inventories every page per role (public, traveller, host) and logs broken numbers and errors

## Environments

- **Staging** has its own database with seed data and test accounts, its own web deployment and its own CRM. All migrations, features and fixes land there first.
- **Production** is promoted from staging through a pull request. Production database writes are made by me, after a backup of the affected rows.
- Payments run in a simulator outside production; in production the server refuses to charge unless live gateway keys are configured, and a test enforces it.

## Mobile releases

- Two flavours, `development` (staging backend) and `production`, with configuration injected at build time — no keys in the source.
- **Codemagic** workflows build and sign: iOS → TestFlight (91 builds uploaded across the dev and production apps), Android → AAB + APK for Play internal testing and production.
- Store work handled end to end: App Store Connect and Play Console app setup, store listings in several languages, screenshots, privacy / data-safety declarations, content rating, review notes and replies to review feedback.
- Release-specific fixes found by testing on devices: Android back-button behaviour, notification icon and channels, keyboard covering actions, secure-storage loss after backup restore (disabled `allowBackup`), platform parity between iOS and Android.

## Incidents and debugging

Problems are reproduced before they are fixed, and the fix is verified in the same environment where the problem was seen. Examples: a missing build-time Maps key that left the map empty only in CI-built Android binaries; transactional email reported as "not delivered" that turned out to be filtered into another mailbox by a server rule; a sign-in issue on the CRM traced, through added server logs, to users signing into the staging instance instead of production.
