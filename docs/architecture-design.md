# CHEFS Activity Sign-Up — Architectural Design

Date: 2026-09-22
Updated: 2026-09-25 (added §12 Internationalization; two admin levels, §4.5; requirements §14 gaps: no-show, admin cancel, activity cancel/reschedule, email)
Status: Approved
Source requirements: [docs/requirements-en.md](./requirements-en.md) ([中文](./requirement-zh.md))

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
| Languages | English (default), Simplified Chinese |

## 2. Technology Choices

| Layer | Choice | Reason |
| --- | --- | --- |
| Framework | Next.js (App Router) + TypeScript | One repo serves the UI and the REST API. Already assumed by `.gitignore`. |
| Database | Supabase Postgres | Transactions and constraints for the credit ledger and roster. |
| Authentication | Supabase Auth, email + password | Sign-up, email confirmation, and password reset are provided; no auth code to write. |
| Styling | Tailwind CSS, mobile-first | Responsive layouts without a component library. |
| Localization | Typed message dictionaries + built-in `Intl` | Two languages need no library; see §12 Internationalization. |
| Email | Resend HTTP API via `fetch` | Urgent notifications only (§8). No SDK, so no new dependency. |
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
    |-- src/i18n/...                        message dictionaries, t()
    |-- fetch() --> Resend HTTP API         urgent email only (§8)
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
authorize(adminCtx, action, scope): void // throws FORBIDDEN_SCOPE
```

Route handlers do the I/O: open a transaction, load state, call the domain function,
write the result, commit. The rules where a bug costs someone money or their seat are
therefore unit-testable without a database.

### 3.2 Trust Boundary

The browser never talks to Supabase directly. It holds a session JWT and calls the
Next.js API; route handlers verify the JWT, resolve the caller's member record and
admin context (§4.5), and only then act. The service role key exists only in the server
environment. There is exactly one code path that can mutate a roster or a ledger.

## 4. Data Model

Ten tables.

| Table | Key columns |
| --- | --- |
| `alliance` | `id`, `name` |
| `venue` | `id`, `name`, `alliance_id` |
| `member` | `id`, `auth_user_id`, `name`, `email`, `alliance_id` (null = non-alliance), `membership_starts_on`, `membership_ends_on`, `is_system_admin` (default false), `locale` (default `en`) |
| `venue_admin` | `member_id`, `venue_id`, `appointed_by`, `appointed_at`; primary key `(member_id, venue_id)` |
| `activity` | `id`, `venue_id`, `activity_date`, `starts_at`, `ends_at`, `max_primary` (default 6), `max_waitlist`, `non_member_fee` (default 5.00), `registration_opens_at`, `registration_closes_at`, `cancellation_deadline_hours`, `status` |
| `registration` | `id`, `activity_id`, `member_id`, `list`, `sort_at`, `checked_in_at`, `cancelled_at`, `no_show` (default false), `no_show_charged`, `created_at` |
| `credit_ledger` | `id`, `member_id`, `delta`, `reason`, `activity_id`, `actor_id`, `created_at` |
| `venue_fee` | `id`, `registration_id`, `amount`, `status`, `paid_at`, `actor_id` |
| `notification` | `id`, `member_id`, `type`, `payload`, `read_at`, `email_status`, `created_at` |
| `audit_log` | `id`, `actor_id`, `action`, `entity_type`, `entity_id`, `before`, `after`, `venue_id`, `alliance_id`, `created_at` |

Enumerations:

- `activity.status`: `draft` | `open` | `closed` | `settled` | `cancelled`
- `registration.list`: `primary` | `waitlist`
- `venue_fee.status`: `unpaid` | `paid` | `waived`
- `notification.email_status`: null (in-app only) | `pending` | `sent` | `failed`

`activity.status` and registration timing are separate concerns. Registration is open
when `status = 'open'` **and** now falls between `registration_opens_at` and
`registration_closes_at`. An admin "closing registration" early (§12) narrows
`registration_closes_at`; reopening widens it. Setting `status = 'closed'` is the
distinct act of ending the activity so it can be settled, and it cannot be undone except
by an admin reverting it before settlement. `cancelled` means the activity will not run
(§6.5); it is terminal and is never settled.

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

### 4.5 Admin Roles and Scope

Two admin levels (requirements §12, Administrator Roles):

- **System admin**: `member.is_system_admin = true`. Any number.
- **Venue admin**: one `venue_admin` row per venue they manage. A venue can have several
  admins; a member can admin several venues.

Each admin request loads an **admin context**: `{ isSystemAdmin, venueIds, allianceIds }`,
where `allianceIds` are the alliances owning `venueIds`.

| Action | System admin | Venue admin |
| --- | --- | --- |
| Create or edit alliances and venues; appoint or remove admins | ✓ | ✗ |
| Create, edit, cancel, or reschedule activities; rosters; check-in; settlement | ✓ | own venues |
| Mark fees paid or waive them | ✓ | fees from own venues' activities |
| Manual credit adjustment and batch grant | ✓ | members of own `allianceIds` only |
| View the audit log | all | rows whose `venue_id` is in `venueIds` or `alliance_id` is in `allianceIds` |
| View members | all | members of own `allianceIds`, plus anyone registered at own venues |

Enforcement:

1. The route handler builds the admin context. A caller with neither flag nor
   `venue_admin` rows gets `403 NOT_ADMIN`.
2. Mutations call `authorize(ctx, action, { venueId?, allianceId? })` before writing.
   Out of scope → `403 FORBIDDEN_SCOPE`.
3. List endpoints filter their queries by the same context, so out-of-scope rows are never
   returned.
4. Every audit row stores the `venue_id` and/or `alliance_id` it concerns, so the scoped
   audit view is a plain filter.
5. Appointing or removing an admin writes an audit row. Removing or demoting the last
   system admin → `409 LAST_SYSTEM_ADMIN`.

Settlement at a venue still deducts credits from cross-venue participants (§6.3). That is
automatic, not a manual adjustment, so a venue admin may run it.

The first system admin is set by a seed SQL script. There is no bootstrap UI.

## 5. REST API

`/api/admin/*` requires an admin and is filtered by the admin context (§4.5). All other
routes are scoped to the caller.

### 5.1 Member

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/me` | Profile, alliance, membership validity, credit totals, no-show count (§11) |
| PATCH | `/api/me` | Set `locale`; called by the language toggle (§12.1) |
| GET | `/api/activities?from=&to=` | Open activities with seats remaining |
| GET | `/api/activities/:id` | Detail, roster, caller's own status |
| POST | `/api/activities/:id/registrations` | Register |
| DELETE | `/api/registrations/:id` | Cancel |
| GET | `/api/me/registrations` | "My Activities" (§11), including no-show flags |
| GET | `/api/me/fees` | Own fee records and payment status |
| GET | `/api/me/notifications` | In-app feed |
| POST | `/api/notifications/:id/read` | Mark read |

### 5.2 Admin

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/api/admin/activities` | Create with §2 configuration |
| PATCH | `/api/admin/activities/:id` | Edit; close or reopen registration; reschedule; cancel (§6.5) |
| GET | `/api/admin/activities/:id/roster` | Primary list, waiting list, check-in state |
| POST | `/api/admin/activities/:id/check-ins` | Record attendance |
| POST | `/api/admin/activities/:id/settle` | Close and settle (§10) |
| PATCH | `/api/admin/registrations/:id` | Move list, reorder, admin-cancel (ignores the cancellation deadline) |
| POST | `/api/admin/credits/grant` | Batch grant to selected members |
| PATCH | `/api/admin/fees/:id` | Mark paid, or waive |
| GET | `/api/admin/audit-log` | Filterable audit trail |
| GET | `/api/admin/notifications?email_status=failed` | Emails that failed to send, for manual follow-up |
| PUT / DELETE | `/api/admin/venues/:id/admins/:memberId` | Appoint or remove a venue admin (system admin only) |
| PUT / DELETE | `/api/admin/system-admins/:memberId` | Grant or revoke system admin (system admin only) |
| CRUD | `/api/admin/members`, `/venues`, `/alliances` | Reference data; venues and alliances are system admin only |

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
   `409 PAST_CANCELLATION_DEADLINE`. This check applies only to the member's own cancel.
   An admin cancel (`PATCH /api/admin/registrations/:id`) skips it and writes an audit row.
3. Set `cancelled_at` on the registration.
4. If the cancelled row was `primary`, find the earliest `sort_at` active waitlist row
   and set its `list = 'primary'`. Write a notification to the promoted member: "You have
   been automatically promoted from the waiting list to the official participant list for
   this activity." This notification is urgent, so it is also emailed (§8).
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

Every no-show row gets `no_show = true` and `no_show_charged` set to the admin's choice,
whichever way they chose. No-show counts are derived from these columns, never stored.
There is no automatic penalty.
| Waitlist member, never promoted | Nothing (§10) |

1. Set `status = 'settled'`, write one `audit_log` entry for the settlement, commit.

### 6.4 Batch Credit Grant

The admin filters members by venue or alliance, selects all or a subset, and enters a
delta and a reason. One transaction writes N `credit_ledger` rows and one `audit_log`
entry recording the actor, the member count, the delta, and the reason.

A venue admin may select only members of their own alliances (§4.5); any other member ID
in the request → `403 FORBIDDEN_SCOPE`, and nothing is written.

### 6.5 Activity Cancel and Reschedule

Both go through `PATCH /api/admin/activities/:id`, holding the activity lock.

**Cancel** (`status: cancelled`, allowed from `draft`, `open`, or `closed`):

1. Set `cancelled_at` on every active registration. Promote no one.
2. Write no ledger or fee rows.
3. Notify every affected member with type `activity_cancelled` (urgent, emailed).
4. Write one audit row. Later registration or settle attempts → `409 ACTIVITY_CANCELLED`.

**Reschedule** (a change to `activity_date`, `starts_at`, or `ends_at`):

1. Update the times and write an audit row with before and after.
2. Notify every active registrant with type `activity_rescheduled`, carrying old and new
   times in the payload (urgent, emailed).

### 6.6 Appointing Admins

System admins only. `PUT` inserts the `venue_admin` row or sets `is_system_admin`;
`DELETE` removes or clears it. Each writes an audit row with the target member and venue.
Revoking the last system admin is rejected (§4.5).

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

Every event writes a `notification` row, shown in-app: registration confirmed, placed on
the waiting list, promoted from the waiting list, activity cancelled or rescheduled, fee
generated, admin roster change affecting the member.

**Urgent types are also emailed:** `promoted`, `activity_cancelled`,
`activity_rescheduled`. These are the events where a member who doesn't open the app
would show up wrongly or miss a seat.

1. The route writes the `notification` row inside its transaction, with
   `email_status = 'pending'` for urgent types and null otherwise.
2. After commit, the handler renders the email in the member's `member.locale` and sends
   it through the Resend HTTP API with `fetch`. It then sets `email_status` to `sent` or
   `failed`.
3. A failed email never rolls back the roster change. Admins list `failed` rows
   (`GET /api/admin/notifications?email_status=failed`) and follow up by hand.

There is no retry queue. If failures become common, add a Cloudflare Cron Trigger that
resends `pending` and `failed` rows. The API key lives only in the server environment.

Notifications store no prose. The UI renders `type` + `payload` in the viewer's current
language at read time (§12 Internationalization).

## 9. Error Handling

A single `DomainError` carrying a `code`, mapped to HTTP at the route boundary:

| Code | Status |
| --- | --- |
| `ACTIVITY_FULL`, `ALREADY_REGISTERED`, `REGISTRATION_CLOSED`, `PAST_CANCELLATION_DEADLINE`, `ALREADY_SETTLED`, `ACTIVITY_CANCELLED`, `LAST_SYSTEM_ADMIN` | 409 |
| `NOT_ADMIN`, `NOT_OWNER`, `FORBIDDEN_SCOPE` | 403 |
| `NOT_FOUND` | 404 |
| Validation failure | 400 |

Route handlers own the transaction. A domain error rolls it back, so nothing partial
persists. Unexpected errors log with a request id and return an opaque 500; no stack
trace reaches a browser.

The API returns the `code`, not a sentence. The UI maps each code to a translated
message (§12 Internationalization).

## 10. Testing

| Layer | Tool | Covers |
| --- | --- | --- |
| Domain | Vitest, no database | Seat assignment, promotion order, classification, settlement outcomes per participant type, cancellation-deadline math, credit arithmetic, `authorize` over every row of the §4.5 matrix |
| Integration | Vitest + local Supabase (Docker) | Activity row locking under concurrent registration, settle-twice idempotency, audit rows landing with their change, ledger totals, venue admin list filtering, last-system-admin guard, email failure leaving the roster change committed (Resend stubbed) |
| i18n | Vitest | Every error code, ledger reason code, and notification type has a message key, including email subjects and bodies for urgent types |
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
batch credit grant, fees, audit log, failed emails. System admins also get venues,
alliances, and admin appointments. Venue admins see the same screens, filtered to their
scope (§4.5), and without the system-only sections.

## 12. Internationalization

The UI supports English (`en`, default) and Simplified Chinese (`zh-CN`).

### 12.1 Choosing the Language

Each request resolves its locale in this order:

1. The `locale` cookie.
2. The browser's `Accept-Language` header.
3. `en`.

An EN / 中文 toggle in the header sets the cookie, calls `PATCH /api/me` to save
`member.locale`, and refreshes the page. The cookie drives the UI; `member.locale` drives
emails (§8), which are sent when the member is not present. The language is not in the
URL.

### 12.2 Mechanism

- `src/i18n/en.ts` is the source of truth. `src/i18n/zh-CN.ts` is typed `typeof en`, so
  a missing or extra key fails the build.
- `getT(locale)` returns `t(key, vars?)` for Server Components. Client components get
  `t` from a small context that the root layout fills with the active dictionary.
- Dates and times use `Intl.DateTimeFormat`. The venue fee uses `Intl.NumberFormat`,
  in USD in both languages. English plurals use `Intl.PluralRules`.

No i18n library is used, keeping dependencies few (§2.1 Hosting Constraints, rule 3).

### 12.3 Codes, Not Prose

The layering rule (§3.1 The Layering Rule) extends to language: `src/domain/`, the database, and the API
emit **codes**, and only the UI turns them into words.

| Stored or returned as a code | Translated where |
| --- | --- |
| Error codes (§9 Error Handling) | UI, from the API's `code` field |
| Ledger reason codes (§4.1 Credits Are a Ledger) | UI, when showing credit history |
| Enum values (§4 Data Model) | UI labels |
| Notification `type` + `payload` (§8 Notifications) | UI, at read time |

As a result, the API is language-neutral, and the audit log and ledger keep stable
English codes. A notification written while a member used English shows in Chinese if
they switch later.

### 12.4 Not Translated

- **Admin-entered names** (alliances, venues, activities, members) show as typed.
- **Notification emails** are translated, using `member.locale` (§8).
- **Supabase Auth emails** (confirmation, password reset) come from Supabase templates,
  which are single-language. Write them as bilingual text in the Supabase dashboard.
  This is configuration, not code.

## 13. Out of Scope for v1

- Email for non-urgent notification types, push notifications, and automatic email retry (§8)
- Automatic no-show penalties
- Online payment for the $5 venue fee; admins confirm payment manually (§8 of requirements)
- Member self-service profile editing beyond password reset
- Scheduled or automatic settlement
- Multi-organization tenancy
- Languages beyond English and Simplified Chinese; translated admin-entered data
