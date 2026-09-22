# CHEFS Activity Sign-Up — Architectural Design

Date: 2026-09-22
Status: Approved
Source requirements: [docs/requirements.md](./requirements.md)

## 1. Scope and Targets

Build the CHEFS Activity Sign-Up web application: a responsive app where members
register for daily activities at school alliance venues, and administrators manage
rosters, credits, and fees.

| Target | Value |
| --- | --- |
| Members, year one | 100–500 |
| Venues | A few |
| Client/server protocol | REST over HTTPS |
| Screens | Responsive, desktop and mobile |

## 2. Technology Choices

| Layer | Choice | Reason |
| --- | --- | --- |
| Framework | Next.js (App Router) + TypeScript | One repo serves the UI and the REST API. Already assumed by `.gitignore`. |
| Database | Supabase Postgres | Transactions and constraints for the credit ledger and roster. |
| Authentication | Supabase Auth, email + password | Sign-up, email confirmation, and password reset are provided; no auth code to write. |
| Styling | Tailwind CSS, mobile-first | Responsive layouts without a component library. |
| Hosting | Cloudflare Workers via `@opennextjs/cloudflare`, GitHub-connected | Free tier with no non-commercial restriction, so development and production share one environment. |
| Domain tests | Vitest | Pure functions, no database needed. |
| Integration tests | Vitest + local Supabase (Docker) | Covers transactions and locking. |

### 2.1 Hosting Constraints

1. **No Vercel-specific APIs.** Do not use `@vercel/blob`, Vercel KV, or Vercel Cron.
   The app must stay deployable to any Next.js host.
2. **No `next/image` optimization.** Cloudflare has no Vercel-style image optimizer.
   Set `images.unoptimized: true`. This app shows lists and forms, not photo galleries.
3. **Keep dependencies few.** The Workers free tier caps the server bundle at 3 MB
   gzipped. An app this size fits comfortably, but heavy dependencies erode the margin.

Scaffold with `npm create cloudflare@latest -- --framework=next`, which generates
`wrangler.jsonc`, `open-next.config.ts`, and the `preview` / `deploy` scripts.

## 3. System Architecture

```
Browser (React, responsive)
    |  fetch() with Supabase session JWT
    v
Next.js on Cloudflare Workers
    |-- app/(member)/...  app/(admin)/...   pages, RSC-rendered
    |-- app/api/...                         the REST API
    |-- src/domain/...                      pure business logic, no I/O
    |-- src/db/...                          Supabase queries and transactions
    |  supabase-js, service role key, server-side only
    v
Supabase: Postgres + Auth
```

### 3.1 The Layering Rule

**`src/domain/` performs no I/O.** Its functions take plain data and return plain data:

```ts
assignList(activity, currentRoster): "primary" | "waitlist" | Full
promote(roster): { promotedRegistrationId } | null
classify(member, activity): "home_venue" | "cross_venue" | "non_alliance"
settle(activity, registrations, noShowDecisions): SettlementResult
canCancel(registration, activity, now): true | PastDeadline
```

Route handlers do the I/O: open a transaction, load state, call the domain function,
write the result, commit. The rules where a bug costs someone money or their seat are
therefore unit-testable without a database.

### 3.2 Trust Boundary

The browser never talks to Supabase directly. It holds a session JWT and calls the
Next.js API; route handlers verify the JWT, resolve the caller's member record and
role, and only then act. The service role key exists only in the server environment.
There is exactly one code path that can mutate a roster or a ledger.

## 4. Data Model

Nine tables.

| Table | Key columns |
| --- | --- |
| `alliance` | `id`, `name` |
| `venue` | `id`, `name`, `alliance_id` |
| `member` | `id`, `auth_user_id`, `name`, `email`, `alliance_id` (null = non-alliance), `membership_starts_on`, `membership_ends_on`, `role` |
| `activity` | `id`, `venue_id`, `activity_date`, `starts_at`, `ends_at`, `max_primary` (default 6), `max_waitlist`, `non_member_fee` (default 5.00), `registration_opens_at`, `registration_closes_at`, `cancellation_deadline_hours`, `status` |
| `registration` | `id`, `activity_id`, `member_id`, `list`, `sort_at`, `checked_in_at`, `cancelled_at`, `created_at` |
| `credit_ledger` | `id`, `member_id`, `delta`, `reason`, `activity_id`, `actor_id`, `created_at` |
| `venue_fee` | `id`, `registration_id`, `amount`, `status`, `paid_at`, `actor_id` |
| `notification` | `id`, `member_id`, `type`, `payload`, `read_at`, `created_at` |
| `audit_log` | `id`, `actor_id`, `action`, `entity_type`, `entity_id`, `before`, `after`, `created_at` |

Enumerations:

- `member.role`: `member` | `admin`
- `activity.status`: `draft` | `open` | `closed` | `settled`
- `registration.list`: `primary` | `waitlist`
- `venue_fee.status`: `unpaid` | `paid` | `waived`

`activity.status` and registration timing are separate concerns. Registration is open
when `status = 'open'` **and** now falls between `registration_opens_at` and
`registration_closes_at`. An admin "closing registration" early (§12) narrows
`registration_closes_at`; reopening widens it. Setting `status = 'closed'` is the
distinct act of ending the activity so it can be settled, and it cannot be undone except
by an admin reverting it before settlement.

Constraints:

- `registration` has a unique index on `(activity_id, member_id)` where `cancelled_at IS NULL`.
- `venue_fee` has a unique index on `registration_id`.
- `credit_ledger.delta` is a non-zero integer.

### 4.1 Credits Are a Ledger

`credit_ledger` is append-only. The §9 figures are derived, never stored:

- Total = sum of positive deltas
- Used = absolute sum of negative deltas
- Remaining = sum of all deltas

Consequences:

1. Restoring a credit (§12) is a `+1` row with a reason, not an edit. Every dispute has
   an intact history.
2. A batch grant writes N rows in one transaction plus one `audit_log` entry describing
   the batch (actor, member count, delta, reason).
3. A member's balance is never overwritten by a concurrent write, because nothing is
   overwritten.

Reason codes: `Grant`, `Home Venue Participation`, `Cross-Venue Participation`,
`No-Show`, `Admin Adjustment`.

### 4.2 Waiting-List Position Is Derived

A registration carries `list` and `sort_at` (defaulting to `created_at`). Position is
the rank by `sort_at` among that activity's non-cancelled `waitlist` rows.

Therefore §5's promotion is: flip the top waitlist row to `primary`, and touch nothing
else. Everyone below moves up because their rank changed. There is no renumbering loop
and no way for two members to hold the same position.

Admin reordering (§12) edits `sort_at`, which is the only operation that needs its own
audit entry.

### 4.3 Participant Type Is Computed

Comparing `member.alliance_id` to the activity's `venue.alliance_id` yields the type:

| Condition | Type |
| --- | --- |
| `member.alliance_id` equals venue's alliance | Home-venue member |
| `member.alliance_id` set but different | Cross-venue member |
| `member.alliance_id` is null | Non-alliance member |

The resolved type is written onto the settlement's ledger or fee row, where it records
what was true on the day. It is not denormalized onto `registration`.

### 4.4 Concurrency

Every write that changes seat allocation — register, cancel, admin roster edit — first
takes `SELECT ... FOR UPDATE` on the `activity` row. Seat assignment is serialized per
activity, so two members cannot both claim the last primary seat.

## 5. REST API

`/api/admin/*` requires `role = admin`. All other routes are scoped to the caller.

### 5.1 Member

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/me` | Profile, alliance, membership validity, credit totals (§11) |
| GET | `/api/activities?from=&to=` | Open activities with seats remaining |
| GET | `/api/activities/:id` | Detail, roster, caller's own status |
| POST | `/api/activities/:id/registrations` | Register |
| DELETE | `/api/registrations/:id` | Cancel |
| GET | `/api/me/registrations` | "My Activities" (§11) |
| GET | `/api/me/fees` | Own fee records and payment status |
| GET | `/api/me/notifications` | In-app feed |
| POST | `/api/notifications/:id/read` | Mark read |

### 5.2 Admin

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/api/admin/activities` | Create with §2 configuration |
| PATCH | `/api/admin/activities/:id` | Edit; close or reopen registration |
| GET | `/api/admin/activities/:id/roster` | Primary list, waiting list, check-in state |
| POST | `/api/admin/activities/:id/check-ins` | Record attendance |
| POST | `/api/admin/activities/:id/settle` | Close and settle (§10) |
| PATCH | `/api/admin/registrations/:id` | Move list, reorder, admin-cancel |
| POST | `/api/admin/credits/grant` | Batch grant to selected members |
| PATCH | `/api/admin/fees/:id` | Mark paid, or waive |
| GET | `/api/admin/audit-log` | Filterable audit trail |
| CRUD | `/api/admin/members`, `/venues`, `/alliances` | Reference data |

### 5.3 API Rules

1. **The client never chooses its list.** `POST /registrations` takes an empty body. The
   server holds the activity lock, counts seats, and assigns `primary` or `waitlist`. A
   client that could request a primary seat could jump the queue.
2. **Settlement is idempotent.** `POST /settle` proceeds only when `status = 'closed'`,
   and sets `status = 'settled'` in the same transaction that writes the ledger rows and
   fee records. A double click cannot double-deduct.
3. **Refusals are 409 with a code.** Full activity, closed registration, past the
   cancellation deadline, already registered — these are legitimate states, not malformed
   requests. Each returns a machine-readable `code` so the UI can explain why.
4. **Overrides audit atomically.** Every admin override writes its `audit_log` row inside
   the transaction that makes the change. An override cannot succeed while its audit
   entry fails.

## 6. Core Flows

### 6.1 Registration

1. Verify the JWT; resolve the caller's member record.
2. Lock the `activity` row.
3. Reject unless `status = 'open'` and now is between `registration_opens_at` and
   `registration_closes_at` → `409 REGISTRATION_CLOSED`.
4. Reject a duplicate active registration → `409 ALREADY_REGISTERED`.
5. Classify the member (§4.3). A non-alliance member is waitlist-only (§8); if primary
   seats remain, they still go to the waiting list.
6. Count primary seats. Under `max_primary` → `primary`. Otherwise count waitlist rows;
   under `max_waitlist` → `waitlist`. Neither → `409 ACTIVITY_FULL`.
7. Insert the registration, write a notification, commit.

Nothing is charged or deducted at this step, for any member type.

### 6.2 Cancellation and Automatic Promotion

1. Lock the `activity` row.
2. Reject if now is within `cancellation_deadline_hours` of `starts_at` (§14.2) →
   `409 PAST_CANCELLATION_DEADLINE`.
3. Set `cancelled_at` on the registration.
4. If the cancelled row was `primary`, find the earliest `sort_at` active waitlist row
   and set its `list = 'primary'`. Write a notification to the promoted member: "You have
   been automatically promoted from the waiting list to the official participant list for
   this activity."
5. Commit.

Remaining waitlist members move up implicitly (§4.2). No credit is deducted or restored,
because nothing was ever deducted.

### 6.3 Check-In and Settlement

Settlement is triggered by the administrator, not a scheduler.

1. The admin records attendance on the roster, setting `checked_in_at`.
2. The admin sets `status = 'closed'`.
3. The roster presents each **no-show** — registered as `primary`, never cancelled, never
   checked in — with a Charge / Don't Charge choice.
4. The admin submits `POST /settle`. In one transaction, for every `primary` registration:

| Case | Action |
| --- | --- |
| Home-venue member, checked in | `credit_ledger` −1, reason `Home Venue Participation` |
| Cross-venue member, checked in | `credit_ledger` −1, reason `Cross-Venue Participation`, recording their own alliance, the venue attended, and the activity date |
| Non-alliance member, checked in | `venue_fee` row for `non_member_fee`, status `unpaid` |
| No-show, admin chose Charge | Same settlement as if they had attended (ledger −1 for an alliance member, fee row for a non-alliance member), but with reason `No-Show` |
| No-show, admin chose Don't Charge | No ledger or fee row; the no-show is still recorded |
| Waitlist member, never promoted | Nothing (§10) |

1. Set `status = 'settled'`, write one `audit_log` entry for the settlement, commit.

### 6.4 Batch Credit Grant

The admin filters members by venue or alliance, selects all or a subset, and enters a
delta and a reason. One transaction writes N `credit_ledger` rows and one `audit_log`
entry recording the actor, the member count, the delta, and the reason.

## 7. Resolved Requirement Conflicts

**§6 versus §14.1.** §6 assumes credits are deducted at registration, so a cancellation
raises the question of restoring one. §14.1 and the project invariants say settlement
keys off check-in. This design follows §14.1: nothing is deducted until settlement, so a
member who cancels was never charged and has nothing to restore. §6's "Restore Activity
Credit / Do Not Restore" decision moves to the no-show case at settlement (§6.3, step 3),
which is where the member actually consumed a seat that someone else could have used.

**§7 versus §10.** §7 says a cross-venue member's credit is deducted "once the member is
successfully promoted from the waiting list," while §10 says deduction happens after the
activity, based on participation. This design follows §10: promotion alone deducts
nothing. A member promoted and then cancelling within the deadline, or an activity that
never runs, must not cost a credit. Deduction requires promotion **and** participation,
settled together in §6.3.

**§6's restore action.** Folded into the generic admin credit adjustment (§12), which
already writes an audited ledger row and covers every restore case, including correcting
a bad settlement. No dedicated restore flow is built.

## 8. Notifications

In-app only for v1. Events write a `notification` row: registration confirmed, placed on
the waiting list, promoted from the waiting list, activity cancelled or rescheduled, fee
generated, admin roster change affecting the member.

**Known limitation:** a member promoted the night before an activity will not learn of it
unless they open the app. The `notification` table carries `type` and a JSON `payload`,
so adding an email channel later is a new delivery worker reading existing rows — no
schema change.

## 9. Error Handling

A single `DomainError` carrying a `code`, mapped to HTTP at the route boundary:

| Code | Status |
| --- | --- |
| `ACTIVITY_FULL`, `ALREADY_REGISTERED`, `REGISTRATION_CLOSED`, `PAST_CANCELLATION_DEADLINE`, `ALREADY_SETTLED` | 409 |
| `NOT_ADMIN`, `NOT_OWNER` | 403 |
| `NOT_FOUND` | 404 |
| Validation failure | 400 |

Route handlers own the transaction. A domain error rolls it back, so nothing partial
persists. Unexpected errors log with a request id and return an opaque 500; no stack
trace reaches a browser.

## 10. Testing

| Layer | Tool | Covers |
| --- | --- | --- |
| Domain | Vitest, no database | Seat assignment, promotion order, classification, settlement outcomes per participant type, cancellation-deadline math, credit arithmetic |
| Integration | Vitest + local Supabase (Docker) | Activity row locking under concurrent registration, settle-twice idempotency, audit rows landing with their change, ledger totals |
| End-to-end | Deferred | Playwright once the UI stabilizes |

Settlement is table-driven: participant type × attendance × no-show decision, with the
expected ledger and fee rows asserted for each combination.

## 11. Responsive UI

Mobile-first Tailwind. Member screens are lists and forms that stack naturally. The one
layout needing deliberate work is the **admin roster**: a wide table on desktop, stacked
cards on mobile, with check-in toggles reachable by thumb.

Member routes: activity list, activity detail and registration, my activities, my
credits, my fees, notifications.
Admin routes: activity list and editor, roster and check-in, settlement, members and
batch credit grant, fees, audit log.

## 12. Out of Scope for v1

- Email and push notifications (§8 records the upgrade path)
- Online payment for the $5 venue fee; admins confirm payment manually (§8 of requirements)
- Member self-service profile editing beyond password reset
- Scheduled or automatic settlement
- Multi-organization tenancy
