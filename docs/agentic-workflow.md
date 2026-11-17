# Agentic development workflow

Klastay does not have an engineering team. The output of one came from running several AI coding agents at the same time, with me acting as tech lead: writing the specs, setting the rules, reviewing, integrating and releasing.

## Tools

- **Claude Code** for most of the work: separate terminal sessions per feature, and sub-agents that Claude Code runs in their own isolated git worktrees.
- **OpenAI Codex** for some isolated tasks on the mobile app (branches prefixed `codex/`).

## 1. Rules first: instruction files in every repository

Each repository has a `CLAUDE.md` (and, for the web, an `AGENTS.md`) that every agent reads before touching code. They are short and concrete, and they encode the things an agent must never get wrong:

- **Invariants.** The booking and negotiation state machine (10 statuses, 24-hour negotiation window) is not to be refactored; prices, availability and the 10 % commission are computed only on the server; MAD is the canonical currency and other currencies are display-only.
- **Data changes.** Any database change is a new, sequentially numbered migration — never an edit to an old one, never a change made from the app.
- **Privacy.** Data stays in EU infrastructure; other users' email, phone and bank details are never read directly — only through dedicated RPCs.
- **Product rules.** Users can browse without an account and login is asked only at high-intent actions; a user can switch between traveller and host mode without logging in again.
- **Working rules.** No mock data, no secrets in code, run analysis, tests and build before asking for review, and never push to the main branch.

The mobile `CLAUDE.md` also maps the backend for the agent: which RPCs exist, which API routes need which auth, which external services are server-only. This removed a whole class of errors where an agent "re-implemented" server logic in the client.

## 2. Splitting the work

A feature is cut into tasks that touch different areas, so that agents do not edit the same files. Some of the feature streams that had their own worktree in the web repository:

| Worktree / branch (real names) | Scope |
|---|---|
| `feature/checkout-bearer` | bearer-token auth on checkout routes for the mobile app |
| `feature/android-fcm` | native push sending (FCM HTTP v1) |
| `feature/block-user` | two-way user blocking (database + web chat) |
| `feature/crm-call-leads` | CRM: host lead pipeline |

Each stream got:

1. a short written spec — what to build, which files are in scope, what "done" means and which tests must pass;
2. its own branch, created from the current integration branch;
3. its own working directory: a **git worktree** (e.g. `klastay-web-fcm`, `klastay-web-block`), so agents could build and run tests at the same time without overwriting each other.

I usually ran **up to four** streams at once. More than that and the review queue, not the agents, became the bottleneck.

## 3. Branches and promotion

```text
feature/*  ──PR──▶  demo  ──(staging: separate database + deployment)──▶  QA
                      │
                      └──PR "demo → production"──▶  production  (klastay.com)
```

- `demo` is the integration branch and is deployed to the **staging** environment, which has its own Supabase project and seed data.
- `production` is deployed to klastay.com. Changes reach it through pull requests: 53 merged so far, 37 of them `demo → production` promotions and 16 targeted production fixes.
- Schema migrations are applied to staging first, checked, and only then to production — with a backup of the affected data before any production write.
- The mobile app follows the same idea with integration branches and two flavours: `development` (staging backend) and `production`.

## 4. Reviewing agent output

I review agent work like a pull request from a fast but new team member:

- **Read the diff.** Check scope (did it touch files outside the task?), the invariants above, and anything that changes auth, RLS, money or personal data.
- **Ask for evidence, not claims.** "Tests pass" needs to come with the actual command output; "it works" needs a screenshot or a run against staging.
- **Run it myself.** `flutter analyze` / `flutter test`, the web test suites, a build, and the relevant Playwright script against staging.
- **Look for the usual agent mistakes**, which I saw often enough to check every time:
  - reporting a fix as done when only the symptom moved;
  - adding client-side logic that duplicates (and drifts from) a server RPC;
  - editing an existing migration instead of adding a new one;
  - mock or placeholder data left in UI code;
  - over-broad changes ("while I was here…") that make review harder.
- **Re-integrate often.** Long-lived agent branches conflict; I merged small and rebuilt the next task on top of the current `demo`.

## 5. Keeping unvalidated changes out of production

- Nothing is pushed to `production` directly; changes arrive as pull requests, and promotions from `demo` are made after QA on staging.
- Production database writes are never done by an agent on its own: I run them, after a backup.
- Payment code has a dedicated safety test (production refuses to charge without real gateway keys).
- Store builds are produced by CI (Codemagic) from integrated branches, then installed and checked on a device or emulator before release.

## 6. Session continuity

Long tasks outlive a single agent context. Each project keeps a progress file that the agent updates at every milestone — goal, what is done (with commits and file paths), what is in progress, open questions, useful commands. A new session starts by reading it, so work continues without re-exploring the repository.

## Illustrative example

*The following is a simplified, illustrative example of a task spec — not a copy of a real one.*

```markdown
Task: host can delete a listing from the mobile editor (parity with web)
Branch: feature/host-delete-listing (worktree: klastay-app-delete-listing)
Scope: lib/host/listing_editor/**, packages/host_repository/**
Server: use the existing delete RPC; do not add client-side checks that duplicate it
UI: destructive action behind a confirmation sheet; disabled if the listing has upcoming bookings (server answer)
Done when:
  - unit tests for the repository call and the cubit states
  - widget test for the confirmation sheet
  - flutter analyze: no new issues; flutter test: all green
  - verified on the Android emulator against staging (screenshot)
```
