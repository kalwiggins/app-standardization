# Epic Design Labs — Standard Features (v3.1)

**Canonical reference** for the foundational features every Epic Design Labs app should have. New apps adopt this whole stack so users get a consistent experience — same login, same org model, same affiliate program, same support widget — across the whole portfolio.

> **About this document**
>
> This is Epic's global standardization guide for all product apps (Foundry, Throttle, EvidentUGC, Dispatch Tickets, Rally Attribution, Clarion, and any future apps). The goal is consistency where it reduces friction — not consistency for its own sake.
>
> If your current implementation diverges from these standards, **discuss with leadership before making changes.** Some divergences may be intentional or load-bearing. Others may represent opportunities to align. Major rework should never happen without conversation first.
>
> **Platform direction:** All Epic apps are migrating to a standardized auth and payments stack. **Clerk** handles identity (users and organizations). **Throttle** handles transactions and billing. Apps currently on Stackbe are in active migration. This standardization assumes Clerk + Throttle as the baseline architecture going forward.
>
> Visual design is intentionally *not* standardized — each app earns its own look and feel. What we standardize is **feature parity**: every app has the same login, org model, affiliate program, support, notifications, etc.

**Status legend (status reflects reality, not aspiration):**

- ✅ **Implemented** — Shipped and verified in at least one app
- 🚧 **Partial** — Shipped in one app but not universally consistent, or incomplete in the reference implementation
- 🧭 **In design** — Spec exists, implementation pending
- ⏸️ **Blocked** — Blocked on a dependency (e.g., Throttle billing doc)
- 📋 **Proposed** — New in v3; not yet implemented anywhere

**Reference implementation:** Foundry IMS (api: `Epic-Design-Labs/app-foundry-ims-api`, admin: `app-foundry-ims-admin`, marketing: `astro-foundryims`). Foundry is the most current implementation; if you find a better pattern, propose a standard update rather than diverging silently.

### Changes in v3.1

Tightening pass on v3 — no architectural reversals, several gaps closed:

- **§3.1 / §3.2** — flipped Clerk's `force_organization_selection` to `false` and clarified the auto-create-on-first-load flow (v3 had these settings contradicting each other).
- **§3.4** — explicit migration note for making `User.email/name/clerkUserId` nullable in apps that started with required columns.
- **§3.6** — pinned the Clerk metadata sync direction (one-way, write-on-change, eventually consistent / advisory).
- **§7** — `PartnerSeat` is now created at trial creation (not just at promotion), so the Partner Dashboard has a single source of truth.
- **§7** — white-label commission reframed as a 10% **discount netted at source** rather than a paid commission, removing the 1099 / tax-form gross-up problem.
- **§12** — webhook secret rotation has a 24-hour dual-signing overlap window so in-flight deliveries don't break.
- **§15.2** — added an immediate-delete option for users invoking right-to-erasure without grace period; backup retention acknowledges compliance-driven extensions beyond 30 days.
- **§17** — explicit "PII never in log lines" rule, hosting-agnostic (CloudWatch, Render, Cloudflare, Vercel, etc.).
- **§17.5** — added Environments (preview / staging / prod), Seed conventions (Futurama theme), Internationalization, Accessibility (WCAG 2.1 AA), and Throttle webhook stub.
- **§22** — picked `@upstash/ratelimit` as the default library; clarified the Sentry rate cap is app-side defense against runaway error loops, separate from Sentry's own quota.
- **§20 checklist** — expanded with the new items.

---

## 1. Overview

Every Epic Design Labs app provides:

1. **Clerk identity** — magic link + Google + Apple. Each app runs its own Clerk project.
2. **Multi-tenant organizations** — every customer is one Clerk Organization mirrored locally for data isolation.
3. **Throttle for payments and billing** — once Throttle integration is finalized, all apps adopt it.
4. **Universal affiliate links** — any user with `affiliate.read` can copy their org's `?r=CODE` link and the org earns credit on signups.
5. **Partner program** 🧭 — designated partners get a dashboard, can create client trials directly, earn 10% recurring via partner seats. White-label (partner stays owner) and transfer-to-client paths both supported.
6. **In-app support** — Dispatch Tickets wired into a Help section, scoped per-org.
7. **Transactional email via Resend** — `react-email` templates compiled at build time.
8. **Outbound webhooks** 🧭 — push events to customer endpoints with HMAC signing, retries, DLQ, usage metrics.
9. **Notifications, activity log, security audit log, API keys, data export, RBAC** — table-stakes infrastructure shared across apps.
10. **Right-to-deletion compliance** 🧭 — GDPR/CCPA-compliant user data deletion via tombstone model.
11. **Sentry observability** with shared logging conventions and PII redaction (`@epic/sentry-config`).
12. **Rate limiting with usage metrics** — every public surface rate-limited; usage visible to users before they hit caps.

UI vocabulary in the affiliate flow never says "referral" — that word is reserved for partner referrals.

### Per-app, not cross-app — and the cost we accept

**Each app has its own Clerk project, its own User/Org/Referral tables, its own affiliate codes, its own Dispatch brand, its own Resend workspace, its own Sentry project, its own root domain.**

A person referring or filing a support ticket in two apps will use two different codes / two different ticket queues. There's no shared identity service in v1; cross-app identity is a much larger design problem and isn't worth solving until the payoff is clear.

This isolation has a real cost — N apps × Clerk plan × Resend workspace × Sentry project × root domain × email warm-up. We accept that cost because cross-app identity at the architecture layer is a much harder problem than it looks, and the per-app model lets each app evolve independently. New apps adopt the same *pattern*, not a shared backend.

The exceptions are **shared libraries** (e.g., `@epic/disposable-emails`, `@epic/sentry-config`, `@epic/email-templates`), which are fine — those are dev dependencies, not runtime services.

---

## 2. Status Table

| Capability | Status | Notes |
|---|---|---|
| Clerk auth (magic link + Google + Apple) | ✅ | Stackbe being phased out portfolio-wide. |
| Org create / switch / leave / close | ✅ | All Clerk-native. Owner sole-leave guard enforced. |
| Org switcher UI | ✅ | Header dropdown with switch / create / leave actions. |
| Affiliate code generation + dashboard | ✅ | One code per org. `Organization.affiliateCode` (8-char), Settings → Affiliate page, stats + signups table. |
| Affiliate signup attribution | ✅ | `/signup?r=CODE` flow + post-auth callback. Idempotent on the API side. Last-touch wins on cookie overwrite. |
| Marketing site → admin signup forwarding | ✅ | `?r=` cookie + link rewriting for `app.<domain>` links. |
| Disposable email blocking at signup | 🧭 | Not yet implemented; standardized on `disposable-email-domains` npm package. |
| Partner role / `accountType` | ✅ schema · 🧭 logic | `User.accountType` column live with default `"client"`. No partner UI yet. |
| Partner dashboard + partner seats | 🧭 | Planned: Referrals tab, "Create Trial" button, partner seat assignment. White-label and transfer-to-client paths. |
| Partner-created trials (direct referral) | 🧭 | `Referral.referralType = "direct"` reserved. |
| Trial period / `trialEndsAt` | ⏸️ | Blocked on Throttle billing doc. |
| Conversion detection (`Referral.convertedAt`) | ⏸️ | Blocked on Throttle billing doc. |
| Reward calculation + payout | ⏸️ | 10% recurring, no cap. Commissions paid month N+1. Blocked on Throttle billing doc. |
| Support (Dispatch Tickets) | ✅ | Tickets scoped per-org via tag, third-party API wrapped server-side. |
| Transactional email (Resend) | 🚧 | Foundry uses Resend with inline HTML for PO send/follow-up. v3 standardizes `react-email`. Migration required. |
| Notification center (in-app bell + dropdown) | ✅ | Bell icon, unread badge, dropdown. |
| Notification preferences (per-method) | 🚧 | Foundry has per-category prefs but not per-method (in-app/email/toast). Schema extension required across all apps. |
| Notification transactional tier | 📋 | New in v3: `deliveryClass: "user_pref" \| "transactional"`. Transactional bypasses preferences (for deletion confirms, security alerts, payment failures). |
| Outbound webhooks | 🧭 | No app has built customer-facing outbound webhooks yet. Standardized below with HMAC, retries, DLQ, usage metrics. |
| Activity log | ✅ | Per-org change history with entity-type/id pivot. |
| Security audit log | 📋 | New in v3: separate `AuditLog` for security-sensitive events (login, role change, key creation, deletion). |
| API keys | ✅ | `<prefix>_*` prefixed keys, role-scoped, hashed in DB. |
| Data export | ✅ | `GET /orgs/me/export` returns a ZIP of CSVs. |
| Right to deletion (GDPR/CCPA) | 🧭 | Tombstone model + 30-day backup retention + HMAC email hash. |
| Users & roles (RBAC) | ✅ | Invite via Clerk, local role assignment, permission decorators. Shared baseline permission set defined below. |
| Session permission re-validation | 🚧 | Foundry reads role fresh on every request, but does not emit `permission_changed` 401 on demotion. |
| Sentry observability | 🚧 | Foundry has Sentry init + global filter. v3 adds shared `@epic/sentry-config` with PII redaction. |
| Rate limiting | 📋 | Per-IP signup, per-API-key, per-org webhooks, error tracking. Usage metrics exposed. |
| Health checks | 🚧 | Foundry has `/health`. Standardized format below. |
| Cron conventions | 📋 | Standardized job naming, error logging to Sentry, timezone handling. |
| Custom fields | ✅ | Per-entity custom field defs + values. |

### Foundry audit checklist (work to align reference impl with v3)

- [ ] Drop `User.clerkUserId @unique` constraint, add `@@index([clerkUserId])` to support multi-org users
- [ ] Verify `Referral.referrerUserId` references local `User.id` (not Clerk ID)
- [ ] Migrate `ActivityLog.createdBy` from email to userId (with backfill for departed users)
- [ ] Add `support.read/write`, `affiliate.read`, `apikeys.manage`, `activity.read`, `export.run`, `notifications.manage` as named permissions per §4 baseline
- [ ] Add `permission_changed` 401 emission on session role mismatch per §16
- [ ] Add per-method notification preferences (bell/toast/email per category) per §11
- [ ] Migrate transactional emails to `react-email` per §9
- [ ] Adopt `@epic/sentry-config` with PII redaction per §17
- [ ] Full schema audit before v3 ratification

---

## 3. Auth & Organizations

### 3.1 Clerk dashboard config

Required settings in every Clerk project:

- **Sign-up enabled** — open self-serve signup. (Restrict via Clerk's allowlist if you need invite-only later.)
- **Magic link, Google, Apple** sign-in methods enabled.
- **Organizations enabled.**
- **`force_organization_selection: false`** — let users land in the app immediately after signup; we auto-create an org in code (see §3.2).
- **`automatic_organization_creation: false`** — Clerk doesn't auto-create either; we control the flow so we can also write the local Org row + send the right magic link.
- **Organization name template** — `"{{user.first_name}}'s Organization"` (fallback `"My Organization"`).
- **2FA / passkeys enabled** at the Clerk level for every app.

### 3.2 Smooth signup path (avoid conversion drag)

Showing prospects a "Create your organization" screen between signup and the dashboard is conversion drag. We avoid it by combining `force_organization_selection: false` (Clerk doesn't gate on org selection) with our own auto-create on first dashboard load.

1. User signs up via Clerk's hosted UI (magic link / Google / Apple).
2. Clerk session is live with zero memberships.
3. On first authenticated load of the admin app, **automatically create an org** named `"<First Name>'s Workspace"` via `clerk.organizations.createOrganization` if the user has zero memberships, then `setActive` it.
4. Land directly on the dashboard.

No extra "create your org" screen unless the user already has multiple orgs and is choosing between them. Org rename is available later in Settings.

### 3.3 Disposable email blocking

Every app blocks signups from known disposable email providers using the **`disposable-email-domains`** npm package (actively maintained, ~3.7k domains). When a blocked domain is detected:

- Reject the signup with a clear error message: **"We don't allow signups with temporary or disposable email addresses. Please use your real email."**
- Log the attempt to Sentry at `info` severity (per §17 — the disposable email block is a signal, not an error).
- Soft-block: a CSR can override for legitimate cases (some businesses route via Mailinator-style services for testing).

### 3.4 Local schema mirror

```prisma
model Organization {
  id              String   @id @default(uuid())
  clerkOrgId      String?  @unique     // canonical link to Clerk
  name            String
  slug            String   @unique
  affiliateCode   String?  @unique     // see §6 — one code per org
  trialEndsAt     DateTime?            // null = paid or no-billing-yet
  status          String   @default("trial")  // "trial" | "active" | "suspended" | "closed"
  // ... app-specific fields

  users     User[]
  referral  Referral?
  partnerSeats PartnerSeat[]      // §7 — orgs this client has added as partners
  // ... other relations
}

model User {
  id             String    @id @default(uuid())
  orgId          String
  clerkUserId    String?                                 // NOT @unique — see note below
  email          String?                                 // nullable for tombstones (§15)
  name           String?                                 // nullable for tombstones
  affiliateCode  String?   @unique                       // legacy — orgs are now the affiliate unit; field reserved for migration
  accountType    String    @default("client")            // "client" | "partner" — see §7
  role           UserRole  @default(MEMBER)
  lastLoginAt    DateTime?
  deletedAt      DateTime?                               // §15 — tombstone marker
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt

  organization   Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)

  @@index([clerkUserId])         // not unique: multi-org users share a clerkUserId
  @@unique([orgId, email])
}

enum UserRole {
  OWNER
  ADMIN
  MEMBER
  VIEWER
  // ... app-specific roles
}
```

**Schema notes:**

- `User.clerkUserId` is **indexed but NOT unique**. A Clerk user that's a member of three orgs has three local `User` rows, all with the same `clerkUserId`. A `@unique` constraint blocks this. (Evident hit this exact bug during their migration.)
- `User.email`, `User.name`, `User.clerkUserId` are **nullable** to support GDPR tombstoning (§15). PII is nulled on deletion; the row itself survives so foreign keys keep working. **Migration note:** apps that started with `email String` (required) — including Foundry — need an audited `ALTER TABLE` to make these columns nullable. The `@@unique([orgId, email])` constraint continues to work; Postgres allows multiple NULLs in unique constraints by default.
- The shift from per-user to per-org affiliate codes is locked in for v3. The `User.affiliateCode` field is reserved for migration only.

### 3.5 ClerkGuard + permission re-validation

Every API request goes through a guard that:

1. Accepts API keys (`<app>_*` prefix) without a Clerk session — see §13.
2. Validates the Clerk JWT.
3. Resolves the active org via `org_id` claim → `X-Clerk-Org-Id` header (fallback) → user's only org.
4. **Re-validates the user's role against the local DB on every request** — see §16. Cache TTL of 5–30s is acceptable for most apps; security-sensitive flows can override to <1s.
5. Lazily creates the local User row on first request if missing.

Reference: `app-foundry-ims-api/src/auth/guards/clerk.guard.ts`.

### 3.6 Source of truth on Clerk-mirrored fields

Some fields exist in both Clerk and the local DB (`accountType`, `role`, org membership). On divergence:

- **Clerk is canonical for auth + org membership.** Local DB is a read cache.
- **Local DB is canonical for `role` and `accountType`.** Clerk metadata is a downstream sync target.

**Sync direction is one-way and write-on-change.** When local `role` or `accountType` changes (admin promotes a user, partner application is approved), we write the new value to Clerk metadata in the same transaction. We do **not** poll or sync from Clerk back to local. Treat Clerk metadata as **eventually consistent / advisory** — the frontend may read it for UI hints (`useUser().publicMetadata`), but the API always re-validates against the local DB. Direct edits to Clerk metadata via the dashboard during incidents will get overwritten on the next local update for that user.

If a divergence is detected during permission re-validation, log to Sentry at `warn` and reconcile by reading the local DB and writing to Clerk.

### 3.7 Org switcher (header dropdown)

Universal pattern: a dropdown in the top-right header showing the user's Clerk memberships, with switch / create / leave actions.

Behavior contract:

- **List orgs** — read from Clerk's `useOrganizationList()` directly. Don't hit the local API for this.
- **Switch** — call Clerk's `setActive({ organization })`, then **immediately** update `localStorage.clerkOrgId` so the next API request includes the new `X-Clerk-Org-Id` header. Then reload (`window.location.href = "/"`) to reset all React Query / server state.
- **Create** — Calls `POST /orgs` (any authenticated user can create their own org).
- **Leave** — any member. Calls `POST /orgs/me/leave` (sole-owner guard enforced server-side).
- **Active indicator** — checkmark on the currently active membership.
- **Cross-tab staleness** — known limitation: a second tab open to the dashboard keeps stale React Query cache after switching in tab 1. The wrong-org-data window is brief; document this in the help center.

The `X-Clerk-Org-Id` header pattern requires every manual `fetch()` site to use a shared `buildAuthHeaders()` helper instead of constructing headers inline. See `app-foundry-ims-admin/src/lib/api/core.ts`.

### 3.8 Org lifecycle endpoints

| Action | Endpoint | Who | Notes |
|---|---|---|---|
| Create org (signup auto-create) | (internal call from admin shell) | Authenticated user | Auto-runs on first load if user has zero memberships. Names org `<First>'s Workspace`. |
| Create additional org | `POST /orgs` | Any authenticated user | User can create more orgs at any time; they become OWNER of the new one. |
| List user's orgs | (Clerk SDK directly — `useOrganizationList`) | Anyone | Don't expose this from your API. |
| Switch org | (Clerk SDK — `setActive({ organization })`) | Anyone | Update `localStorage.clerkOrgId` so the next API call sends the right header. |
| Update settings | `PATCH /orgs/me/settings` | OWNER | Name change propagates to Clerk. |
| Delete org (close) | `DELETE /orgs/me` body `{ confirm: <orgId> }` | OWNER | Deletes Clerk org + local cascade. UI: "Danger Zone" card with type-org-name confirm. **Final data export remains available for 30 days post-close** (see §15.1). |
| Leave org | `POST /orgs/me/leave` | Any member | Removes Clerk membership + local User row. Sole owners blocked → must close instead. |

---

## 4. Users & Roles (RBAC)

Every app needs invite, role assignment, and permission gating.

### Schema

```prisma
enum UserRole {
  OWNER       // full control, can close the org
  ADMIN       // full control short of close
  MEMBER      // standard access
  VIEWER      // read-only
  // ... app-specific roles (PURCHASING, WAREHOUSE, etc. for IMS)
}
```

Role lives on `User.role`. Clerk's own `org:admin` / `org:member` is mapped 1:1 with our `OWNER`/`ADMIN` vs `MEMBER`.

### Shared permission baseline

**Every app MUST implement these permissions.** App-specific permissions extend this list. This baseline ensures roles mean the same thing across the portfolio.

| Permission | Description |
|---|---|
| `settings.read` | View org settings |
| `settings.write` | Modify org settings |
| `users.manage` | Invite, remove, change roles of users |
| `org.manage` | Modify org-wide configuration |
| `org.close` | Delete the org (OWNER only) |
| `support.read` | View support tickets |
| `support.write` | Create/comment on support tickets |
| `affiliate.read` | View own org's affiliate code, stats, and commissions |
| `apikeys.manage` | Create/revoke API keys |
| `activity.read` | View activity log |
| `audit.read` | View security audit log |
| `export.run` | Export org data |
| `notifications.manage` | Configure own notification preferences |
| `webhooks.manage` | Configure outbound webhooks |

Default role-to-permission mapping (apps may extend, must not contract):

- **OWNER** — all permissions including `org.close`
- **ADMIN** — all except `org.close`
- **MEMBER** — `settings.read`, `support.read`, `support.write`, `affiliate.read`, `activity.read`, `notifications.manage`
- **VIEWER** — `settings.read`, `support.read`, `affiliate.read`, `activity.read`, `notifications.manage`

> **Note on VIEWER + `affiliate.read`:** Every org has a single affiliate code (§6). VIEWER can see their org's code and commission stats but cannot manage settings. This makes the affiliate program viewable to all org members regardless of role.

### Permission gating

Decorator-based on the controller side:

```ts
@Get('settings')
@RequirePermission('settings.read')
async getSettings(...) { ... }
```

UI components also gate via `usePermission()`:

```tsx
const { can } = usePermission();
if (!can("org.manage")) return null;
```

### Invite flow

`POST /users/invite` with `{ email, role }`. Calls `clerk.organizations.createOrganizationInvitation`. Local User row created with `isPending: true` until they accept.

Settings → Users page lists current members + pending invites with role dropdown and remove button.

---

## 5. Account Types

Every Clerk user has `accountType` in `publicMetadata` (mirrored to `User.accountType` locally). Default `client`.

| Type | Who | What they see |
|------|-----|---------------|
| `client` | End users running the app for their business | Standard app UI. Can share their org's affiliate link. |
| `partner` 🧭 | Agencies, consultants, resellers | Adds Partner sidebar section: Referrals tab, "Create Trial for Client" button, partner profile, payout settings, partner seat assignments. |

Both types log in identically. Partner is a role, not a separate auth system.

---

## 6. Affiliate Program ✅

### Two referral paths (one model)

```
                ┌──────────────────────────┐
                │  New Clerk org created   │
                └────────────┬─────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
      Affiliate referral             Direct referral (Partner)
      (anyone shares ?r=)            (partner created the trial)
              │                             │
              ▼                             ▼
      10% recurring commission        10% recurring commission
      (no cap, paid month N+1)        (no cap, paid month N+1)
                                      Plus: partner seat in client org
```

Both paths share the same commission rate. **The differentiator is partner status** — partners get the dashboard, the "Create Trial" button, and the partner seat in client orgs. Partner referrals require the partner to physically create the trial (action is proof of attribution); affiliate referrals attribute via shared link.

### Standardized affiliate rules

These rules apply to every app:

1. **One affiliate code per organization.** Any user with `affiliate.read` (which all roles have) can see their org's code, share it, and view commission stats. Multiple users in the same org share one code; commissions accrue to the org, not to individual users.

2. **Last-touch attribution wins.** If a prospect clicks two different affiliate links in the 30-day cookie window, the most recent code overwrites the earlier one. This is a deliberate trade-off: simpler than first-touch, aligned with industry standard, but unfair to top-of-funnel educators who may lose credit to bottom-of-funnel coupon sites. Pricing the affiliate tier at 10% with no cap is intended to keep both kinds of partners engaged.

3. **Self-referral is rejected** by email match against the *referrer's own email*. If the signing-up user's email matches the referrer's, attribution is blocked. (Earlier drafts said "any user in the referrer's org" — this was too broad and would block legitimate co-worker referrals.) Returns `{ attributed: false, reason: "self_referral" }`.

4. **Code regeneration: snapshots are immutable.** A user with `affiliate.read` + `org.manage` can regenerate their org's `affiliateCode`. Existing `Referral` rows keep the old code (we snapshot `affiliateCode` on the Referral). The old code stops working for *new* attributions.

### Schema

```prisma
model Referral {
  id              String    @id @default(uuid())
  referrerOrgId   String                              // local Organization id (NOT Clerk id)
  referredOrgId   String    @unique                   // local Org id; one credit per signup
  referralType    String    @default("affiliate")     // "affiliate" | "direct"
  affiliateCode   String                              // snapshot at attribution time

  status          String    @default("pending")       // pending | active | converted | expired
  rewardStatus    String    @default("pending")       // pending | paid | ineligible

  createdAt       DateTime  @default(now())
  convertedAt     DateTime?

  referrerOrg     Organization @relation("ReferrerOrg", fields: [referrerOrgId], references: [id])
  referredOrg     Organization @relation("ReferredOrg", fields: [referredOrgId], references: [id], onDelete: Cascade)

  @@index([referrerOrgId])
  @@index([affiliateCode])
}
```

**Schema fix from v2:** `Referral.referrerOrgId` is a local `Organization.id`, not a Clerk ID. (v2 had `referrerUserId` referencing a Clerk user ID, which is unstable across instance recreation and breaks GDPR deletion.)

When constructing a Referral row from a user action (e.g., affiliate attribution or partner trial creation), resolve the referring user's local `User.orgId` and use that — not `partner.organizationId` shorthand, which doesn't exist as a literal field on the user.

### Status state machine

```
pending  → active     (org enters trial / starts using product)
active   → converted  (org converts to paid subscription) [Throttle webhook]
active   → expired    (trial elapses without conversion)  [scheduled job]
```

Code regeneration does NOT cause expiry — old refs stay valid.

### Affiliate code generation

8-char alphanumeric, lazy-generated on first `GET /affiliates/me/code`:

```ts
import { randomBytes } from 'crypto';

function makeCode(): string {
  return randomBytes(16).toString('base64url').replace(/[-_]/g, '').slice(0, 8);
}

private async generateUniqueCode(): Promise<string> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const code = makeCode();
    const taken = await this.prisma.organization.findUnique({ where: { affiliateCode: code } });
    if (!taken) return code;
  }
  throw new BadRequestException('Could not generate a unique affiliate code; try again.');
}
```

### URL format

`https://<marketing-domain>/?r=ABC12345`

Cookie: `affiliateR` (30-day TTL, samesite=lax). The `?r=` is short by design.

### Flow

1. Existing org member visits Settings → Affiliate, clicks Copy.
2. They share the URL.
3. Prospect lands on the marketing site → script reads `?r=`, validates against `/^[A-Za-z0-9]{8}$/` (tightened from earlier `{4,16}` to match generated code length), stores cookie (overwriting any previous), rewrites every `<a href*="app.<domain>">` link to point at `/signup?r=CODE`.
4. Prospect clicks any "Sign in" / "Get started" CTA → lands at `app.<domain>/signup?r=CODE`.
5. Admin's `/signup` page captures `r` into `sessionStorage.affiliateR`, renders Clerk's `<SignUp />` (with disposable email check).
6. Prospect completes signup → Clerk org created.
7. On first dashboard load, the `AffiliateAttribution` component fires `POST /affiliates/attribute` with `{ orgId, code }`.
8. API creates the `Referral` row, idempotent on duplicate org. Self-referral check runs against the referrer's own email.
9. Referrer sees the new sign-up under Settings → Affiliate.

### API surface

```
GET  /affiliates/me/code              → { code: "k7q3xZ9p" }
POST /affiliates/me/code/regenerate   → { code: "newCode1" }
GET  /affiliates/me/signups           → { stats: {...}, signups: [...] }
POST /affiliates/attribute            → { attributed, reason?, referralId? }
```

`/attribute` is auth'd and idempotent. Anti-fraud checks: self-referral rejected; duplicate attribution returns `already_attributed` no-op; disposable email already blocked at signup.

### UI surfaces

- **Settings → Affiliate** subpage (visible to all roles via `affiliate.read`). Three sections: link card with Copy button, three stat cards (total / active / converted), table of attributed orgs.
- **`/signup?r=CODE` route** — public, captures ref into sessionStorage.
- **`AffiliateAttribution` component** mounted in dashboard layout — fires once per session.

UI vocabulary: "your link", "sign-ups via your link", "Affiliate" page name. **Never** "referral" — reserved for §7.

---

## 7. Partner Program 🧭

The partner program is a fundamentally different model from affiliates. **Partners earn commission by *creating* the trial directly.** This eliminates attribution disputes that plague most B2B SaaS partner programs.

### Core principle: action is proof of attribution

If you want partner-tier credit, **you must be the one that physically sets up the trial.** There's a database row showing you created the org. No forms, no claims to validate, no disputes.

A user who promotes the product via affiliate link still gets affiliate credit. Partner status unlocks the partner dashboard, the partner seat (continued access to client orgs), and the trial-creation flow.

**Both affiliate and partner referrals earn 10% recurring commission.** The differentiator is the *kind* of relationship — affiliates are link-sharers; partners are integrators with ongoing client relationships.

### Becoming a partner

1. User clicks "Become a Partner" or visits `/partners/apply` (marketing site).
2. Submits application: company name, website, expected volume, etc.
3. Admin reviews via internal tool, approves or rejects.
4. On approval: Clerk metadata + local User row updated to `accountType: "partner"`. Partner UI surfaces in admin sidebar.

### Partner organizations

Partners have their own Clerk org, just like clients. Within their partner org:

- They can have multiple users (their team — agency staff, consultants, etc.)
- They have access to a Partner Dashboard showing all client orgs they've referred or been added to
- They can configure team-wide defaults (e.g., "all my team members auto-get access to all my partner accounts")

### Partner seats in client orgs

The partner-to-client relationship is modeled as a **partner seat** at the client org level, separate from regular user invites.

```prisma
model PartnerSeat {
  id              String    @id @default(uuid())
  clientOrgId     String                              // the client's org
  partnerOrgId    String                              // the agency/consultant's org
  permissions     Json                                // client-configurable permission set
  addedAt         DateTime  @default(now())
  addedByUserId   String                              // who in the client org added them (local User.id)
  removedAt       DateTime?

  clientOrg       Organization @relation("ClientSeats", fields: [clientOrgId], references: [id], onDelete: Cascade)
  partnerOrg      Organization @relation("PartnerSeats", fields: [partnerOrgId], references: [id], onDelete: Cascade)
  assignments     PartnerSeatAssignment[]            // which partner team members can use this seat

  @@unique([clientOrgId, partnerOrgId])
}

model PartnerSeatAssignment {
  id              String    @id @default(uuid())
  partnerSeatId   String
  partnerUserId   String                              // user from the partner org (local User.id)
  assignedAt      DateTime  @default(now())

  partnerSeat     PartnerSeat @relation(fields: [partnerSeatId], references: [id], onDelete: Cascade)

  @@unique([partnerSeatId, partnerUserId])
}
```

**Key properties:**

- **One client can have multiple partner seats.** A client might have an agency, a consultant, and a reseller all simultaneously.
- **Partners can be removed by either side.** The client can revoke the seat. The partner can step away.
- **Partner referral credit is independent of partner seat status.** If Agency A referred Client X but the client later replaces them with Agency B, Agency A *still* receives the referral commission. This is intentional — the original partner did the originating work. The 10% rate (rather than 20%) plus the "action is proof" requirement (you must physically create the trial) makes this self-balancing: bad actors can't easily farm signups because each one requires real client engagement to convert.
- **Partner orgs decide which of their team members access which client seats** via `PartnerSeatAssignment`. The partner org's Partner Dashboard manages this.

### Permissions on partner seats

When a client adds a partner seat, the **client decides** what permissions that seat grants. Standardized options:

- `view_only` — read-only access to the app
- `operator` — can perform day-to-day operations (most common)
- `admin` — full administrative access (short of org closure or billing)
- `custom` — granular permission selection

The client can change a partner seat's permission level at any time without removing the seat.

### Trial creation by partner (the referral act)

```
Partner Dashboard → "Create Trial for Client" button → dialog:
  - Client email
  - Client first/last name
  - Client company name
  - Optional: trial duration override

On submit:
  1. clerkService.createOrganization({ name: <company> })
  2. clerkService.findOrCreateUserByEmail(<client_email>)
  3. clerkService.createInvitation(clerkOrgId, client_email)
  4. localUser = create User for partner with role: OWNER in the new client org
  5. Partner is the OWNER (Clerk org:admin) of the trial org
  6. prisma.partnerSeat.create({
       clientOrgId: localOrg.id,
       partnerOrgId: partnerUser.orgId,                  // partner's own org
       permissions: { tier: "admin" },                    // co-exists with their OWNER role
       addedByUserId: partnerUser.id,
     })
  7. prisma.partnerSeatAssignment.create({               // creator gets the seat assigned
       partnerSeatId: seat.id,
       partnerUserId: partnerUser.id,
     })
  8. prisma.referral.create({
       referrerOrgId: partnerUser.orgId,
       referredOrgId: localOrg.id,
       referralType: "direct",
       affiliateCode: partnerOrg.affiliateCode,
       status: "pending"
     })

Client → clicks invite link → Clerk signup → joins the pre-created org as a member
```

**Why create the PartnerSeat at trial creation, not just at promotion:** Without it, the dashboard's "list this partner's client orgs" query has to UNION (`Users where role=OWNER and accountType=partner`) with `PartnerSeats` — two sources of truth for the same conceptual relationship. Creating the seat upfront makes `PartnerSeat` the canonical answer; OWNER is just an additional role the partner happens to hold during a white-label arrangement.

**Audit trail during partner-controlled trial:** All `ActivityLog` entries during this period must be tagged `source: "partner_seat"` to distinguish partner-attributed actions from client-attributed actions. This creates a clear audit trail for any disputes about what was done before the client took ownership.

### Two ownership paths after conversion

When a partner-created trial converts to paid, there are **two valid ownership models**:

#### Path A: White-label (partner stays owner)

The partner pays the subscription fee on the client's behalf as part of an all-inclusive service offering. The partner remains OWNER of the org indefinitely.

- Partner stays as `OWNER` of the client org.
- Partner's payment method is on file in Throttle.
- **Partner receives a 10% white-label discount on the bill** — netted at source, not paid out as commission. (Earlier drafts described this as a self-paid commission; that route creates a 1099 / tax-form gross-up problem where the partner reports income they're really just netting against an expense. Discount-at-source delivers identical economics with cleaner accounting.) Throttle invoices the partner at 90% of list price for white-label-mode subscriptions.
- The client may not have visibility into the bill (this is a partner choice).

#### Path B: Promote client to owner

The partner promotes the client user to OWNER. Partner becomes a `PartnerSeat` and continues to earn commission.

- Partner navigates to the client org and clicks "Promote Client to Owner."
- Dialog confirms: "This will transfer ownership of `<Client Org>` to `<client@email.com>`. You will retain a partner seat with `<role>` permissions."
- On confirm:
  - Clerk org membership: client user promoted to `org:admin`, partner user demoted.
  - Local: client user `role` set to `OWNER`. Partner user removed from local `User` table for that org (they retain access via the existing `PartnerSeat` row, created at trial creation).
  - Throttle billing setup flow surfaces to the client.
  - `Referral.status` will transition `active` → `converted` when the client's first invoice is paid.
- Partner continues to earn 10% commission on client's bill (paid out as commission since the client now pays the bill, not as discount).

### Client-initiated ownership claim (escape hatch via support)

If a partner refuses to promote a client who wants ownership, the client has an escape hatch:

1. Client opens a support ticket: "I want ownership of my account transferred from `<Partner>`."
2. Support contacts the partner with a **7-day grace period** to cooperate.
3. If the partner doesn't promote within the grace period, support manually promotes the client to OWNER and demotes the partner to `PartnerSeat` with default `operator` permissions (client can revoke entirely if desired).
4. The partner retains their referral credit (the 10% commission keeps flowing) — the dispute is about ownership, not attribution.

This is intentionally a **manual support process** rather than self-service: the human review prevents abuse in either direction (clients claiming ownership of accounts where the partner is genuinely paying, or partners holding accounts hostage).

### Race-to-create

Multiple partners can create trials for the same prospect email — each gets their own `Referral` row in `pending` status. Whichever partner converts the prospect to paid wins the credit. If both somehow convert (unlikely edge case where a prospect actively tries both partners), both get credit; the work was done in both cases.

This is intentional: a cooldown would constrain partners' selling processes for marginal benefit. The "action is proof" + "convert to paid" combination is self-balancing.

### Partner Dashboard

- Gated by `useUser()`'s `publicMetadata.accountType === "partner"`.
- **Referrals tab** — table of orgs they've directly created or been credited for. Each row: client name, status (trial/active/converted/expired), commission earned to date, commission expected next month.
- **Partner Seats tab** — all client orgs where this partner has a seat. Filter by "originated by me" vs "added by client." Shows seat permissions and team assignments.
- **"Create Trial for Client" button** — top-right primary action.
- **Team tab** — manage which of the partner org's users get assigned to which client seats. Default-assignment toggle ("auto-assign all team members to new partner seats").
- **Profile tab** — company info, payout method (PayPal email), tax form on file.
- **Commissions tab** — monthly statement with line-item attribution showing which clients generated which earnings.

### Visibility rules

- **Partners can see commission attribution per client.** Critical — partners deserve to know exactly which clients are generating their earnings. (Existing partner programs that hide this leave partners frustrated and distrustful.)
- **Partners cannot see client billing details** (specific subscription tier, payment method, invoice details) — *unless* the partner is the OWNER paying the bill (white-label path). In that case the partner sees the bill because it's theirs.
- **Clients can see all partner seats they've granted** at Settings → Partners.

### Reward tier and commission timing

- **10% recurring commission, no cap** for both `affiliate` and `direct` referrals.
- Commissions for billing events in month N are paid in month N+1. Example: a client pays their April invoice on April 15. The partner's $X commission accrues to the May commission statement, paid early May.
- This gives clean monthly reconciliation, time for refunds/chargebacks to settle, and predictable payout timing.

### API surface (planned)

```
GET    /partner/me/profile                  → partner profile + payout settings
PUT    /partner/me/profile
GET    /partner/me/referrals                → all referrals (direct + affiliate this partner earned)
GET    /partner/me/seats                    → all client orgs where this partner has a seat
GET    /partner/me/commissions              → monthly statements
GET    /partner/me/team                     → partner org team + seat assignments
PUT    /partner/me/team/defaults            → auto-assign-all toggle
POST   /partner/trials                      → create org + invite + Referral + partner seat
POST   /partner/applications                → user applies for partner status (any user)

POST   /orgs/me/partner-seats               → client adds a partner (client side)
DELETE /orgs/me/partner-seats/:id           → client removes a partner
PATCH  /orgs/me/partner-seats/:id           → client changes permissions
POST   /partner/seats/:id/leave             → partner removes themselves
PATCH  /partner/seats/:id/assignments       → partner re-assigns team members

POST   /support/ownership-claim             → client requests ownership transfer (creates support ticket)
```

---

## 8. Trials ⏸️

### Public signup ✅ (mechanic exists, no time-bound trial)

Today: anyone hitting `/signup` creates a Clerk user + org instantly. Org has no expiration, no billing state. Effectively a permanent free tier.

### Once Throttle billing ships:

- New orgs get `trialEndsAt: NOW + 14 days` (default; configurable per app).
- A scheduled job flips `status` to `expired` past the deadline.
- Affiliate `Referral.status` transitions `active` → `expired` when trial expires without converting.

### Partner-created trial 🧭

Same lifecycle, but `Referral.referralType = "direct"`. Partner sees the trial countdown in their Referrals tab.

### Conversion detection ⏸️

When Throttle reports a paid subscription:
1. Find the `Referral` row by `referredOrgId`.
2. Set `convertedAt = NOW`, `status = "converted"`.
3. Calculate reward (10% of MRR).
4. Set `rewardStatus = "pending"` until payout runs (next month).

> **Detailed billing standardization is a separate document — see §23 stub.** This section reserves the integration points.

---

## 9. Email Infrastructure (Resend + react-email)

Every app uses **Resend** for transactional email delivery and **`react-email`** for templates. Each app has its own Resend workspace.

### Standardized conventions

**From-address format:** `<App Name> <function@<subdomain>.<rootdomain>>`

Examples:
- Foundry IMS: `Foundry IMS <orders@ims.foundryims.com>`
- Dispatch Tickets: `Dispatch <support@tickets.dispatch.com>`
- Rally Attribution: `Rally <reports@rally.attribution.com>`

The `function@` prefix indicates email type (`orders`, `support`, `noreply`, `notifications`, `billing`). The subdomain isolates the app's email reputation from the marketing/admin domains.

### Templates: react-email, not inline HTML

**All transactional emails use [`react-email`](https://react.email).** Templates are JSX components in `src/emails/` (or equivalent), version-controlled, compiled to HTML at build time. This gives apps:

- Syntax highlighting and type checking
- Component reuse (header, footer, button)
- Local preview during development
- Dark-mode handling without string concatenation
- Easy testing

Inline HTML strings are reserved for trivial one-off cases only (e.g., a single utility email that takes no variables). For Foundry, this means migrating PO send/follow-up templates to `react-email`.

Variable substitution happens via JSX props at render time — not via Resend's hosted template variables. This keeps templates portable and prevents Resend lock-in.

**Per-org template overrides** (e.g. Foundry's `Organization.poEmailTemplate` for customizable PO emails) are stored as serialized JSX-compatible content or a structured override schema in the DB and rendered into the standard template at send time.

### API key management

API keys are managed per-app and per-environment. Each app decides ENV files, secret managers, etc. — but the API key is named consistently: `RESEND_API_KEY` (and `RESEND_FROM_ADDRESS` for the default sender).

### Required transactional email types

Every app MUST send these via Resend:

- **Auth-related** — sent by Clerk (magic links, invitations, password resets)
- **Notifications** — when a user has email-delivery enabled for a notification category, OR when the notification is `transactional` class (see §11)
- **Support replies** — handled by Dispatch when an agent replies to a ticket
- **Affiliate signup notifications** — "Your link was used to sign up `<org>`"
- **Right-to-deletion confirmations** — see §15 (always transactional class)
- **Security alerts** — login from new device, role change, key created (always transactional class)

App-specific transactional emails extend this list.

### Service skeleton

```ts
@Injectable()
export class EmailService {
  private readonly apiKey: string;
  private readonly fromAddress: string;

  constructor(private config: ConfigService) {
    this.apiKey = config.get('RESEND_API_KEY')!;
    this.fromAddress = config.get('RESEND_FROM_ADDRESS')!;
  }

  isConfigured(): boolean {
    return !!this.apiKey;
  }

  async send(to: string, subject: string, reactComponent: ReactElement, opts?: SendOpts) {
    if (!this.isConfigured()) {
      this.logger.warn('Email skipped: RESEND_API_KEY not configured');
      return null;
    }
    const html = await render(reactComponent);
    // ... call Resend API
  }
}
```

`isConfigured()` enables graceful degradation — apps still run in dev/staging without Resend setup; emails just don't send (and log a warning).

### Inbound email (optional, app-specific)

Some apps need to receive replies (Foundry receives PO replies). Resend posts these to a webhook endpoint at `/email/inbound-webhook` (or app-specific name). The webhook handler MUST verify the Resend signature before processing.

---

## 10. Support (Dispatch Tickets) ✅

In-app support is powered by **Dispatch Tickets** at `https://dispatch-tickets-api.onrender.com/v1`. Each app is a Dispatch "brand" with its own ticket queue.

### How it works

```
[Admin: Support page]
   │
   │  Authenticated request
   ▼
[App API: /support/tickets]
   │
   │  Server-side proxy with API key + per-org tag filter
   ▼
[Dispatch API: /brands/<brandId>/tickets]
   │
   │  Returns tickets with org tag/metadata
   ▼
[Filtered to caller's org, returned to admin]
```

The app API never exposes Dispatch credentials to the browser. All Dispatch calls happen server-side. Tickets are scoped to the caller's org via a tag named `org:<orgId>` and `metadata.orgId`.

### Required env vars

```
DISPATCH_API_URL=https://dispatch-tickets-api.onrender.com/v1
DISPATCH_API_KEY=<per-app secret>
DISPATCH_BRAND_ID=<per-app brand identifier>
```

### API surface

```
GET    /support/tickets               → list this org's tickets (status, search, cursor params)
GET    /support/tickets/:id           → single ticket (404 if not in caller's org)
POST   /support/tickets               → create ticket (auto-tags org:<orgId>)
PATCH  /support/tickets/:id           → update title / status
POST   /support/tickets/:id/comments  → add a comment
POST   /support/tickets/:id/attachments → presigned upload URL
```

### UI surfaces

- **Help menu in header** (question-mark icon, top-right). "Get support" → routes to `/support`.
- **`/support` page** — list of caller's tickets with status filter + search.
- **`/support/new`** — create form (title, body, attachments, priority).
- **`/support/[id]`** — single ticket with comment thread + attachments.

Reference: `app-foundry-ims-api/src/support/support.service.ts`.

---

## 11. Notifications & Notification Center

Every app needs a **notification center** for in-app delivery, with **per-method preferences** users can configure. New in v3: a **transactional tier** that bypasses preferences for compliance-critical messages.

### Required delivery methods

Every app supports three delivery methods, each toggleable per notification category per user:

1. **Notification bell** (in-app, persistent) — badge in header, dropdown shows recent items, click marks read
2. **In-app toast** (in-app, ephemeral) — slides in from corner, auto-dismisses
3. **Email** (via Resend / `react-email`, see §9)

Users mix and match per category. Example: "Email me for new tickets, but only show toast for team mentions."

### Transactional vs. user-pref delivery class

```prisma
model Notification {
  id              String   @id @default(uuid())
  orgId           String
  userId          String?  // null = whole-org notification
  category        String   // e.g. "support.ticket_replied", "affiliate.signup"
  deliveryClass   String   @default("user_pref")  // "user_pref" | "transactional"
  title           String
  body            String?
  link            String?  // deep link path
  readAt          DateTime?
  createdAt       DateTime @default(now())
}

model NotificationPreference {
  id        String  @id @default(uuid())
  orgId     String
  userId    String
  category  String
  bell      Boolean @default(true)
  toast     Boolean @default(true)
  email     Boolean @default(false)  // default off to prevent inbox spam

  @@unique([userId, category])
}
```

**`deliveryClass: "transactional"`** bypasses user preferences and sends via all three methods (bell, toast, email) regardless of opt-outs. Reserved for:

- Right-to-deletion confirmations (§15)
- Security alerts (login from new device, role change, key creation, password change)
- Payment failures / billing issues
- Org closure warnings

**`deliveryClass: "user_pref"`** (default) respects the user's `NotificationPreference` settings.

### Standard notification categories

Every app fires these:

- `org.member_invited` (user_pref)
- `org.member_joined` (user_pref)
- `org.member_removed` (transactional — security)
- `org.role_changed` (transactional — security)
- `support.ticket_replied` (user_pref)
- `affiliate.signup` (user_pref)
- `partner.client_created` 🧭 (user_pref — sent to partner)
- `partner.commission_earned` 🧭 (user_pref)
- `auth.login_new_device` (transactional — security)
- `auth.password_changed` (transactional — security)
- `apikey.created` (transactional — security)
- `billing.payment_failed` (transactional)
- `privacy.deletion_confirmed` (transactional)

App-specific categories extend this list.

### Notification center UI

- **Bell icon in header** with unread badge — dropdown shows recent (most recent 10–20), click any item navigates to its `link` and marks read.
- **"Mark all read" link** at the top of the dropdown.
- **"View all notifications" link** at the bottom → routes to `/notifications` for full history.
- **Settings → Notifications** subpage — per-category checkboxes for bell / toast / email. **Transactional categories show a lock icon and cannot be disabled.**
- **Toast component** mounted in the app shell.

### Sender behavior

When an event fires `notificationsService.create({ category, deliveryClass, ... })`:

1. Create the `Notification` row.
2. If `deliveryClass === "transactional"`: send via all three methods, ignore preferences.
3. If `deliveryClass === "user_pref"`:
   - Look up `NotificationPreference` for that category.
   - Bell, toast, email each fire if enabled.
4. If no preference row exists, fall back to defaults defined in code.

---

## 12. Outbound Webhooks 🧭

For pushing events to customer endpoints. Inverse of API keys.

### Why this matters

Apps like Dispatch and Rally are integration-heavy — customers will want to subscribe to events and react in their own systems. Standardizing this once means reliable, auditable webhook delivery instead of six divergent implementations.

### Standard event prefixes

Events use namespaced types: `<resource>.<action>` — e.g. `ticket.created`, `attribution.matched`, `order.shipped`.

Shared events every app should emit (when applicable):

- `org.created`, `org.updated`
- `user.added`, `user.removed`
- `subscription.changed` ⏸️ (when Throttle ships)
- `partner_seat.added` 🧭, `partner_seat.removed` 🧭

App-specific events extend this list.

### Schema

```prisma
model Webhook {
  id            String   @id @default(uuid())
  orgId         String
  url           String
  secret        String   // for HMAC signing; shown to user once
  events        String[] // subscribed event types; "*" for all
  isActive      Boolean  @default(true)
  failureCount  Int      @default(0)  // consecutive failures
  createdAt     DateTime @default(now())
  lastSuccessAt DateTime?
  lastFailureAt DateTime?

  deliveries    WebhookDelivery[]
}

model WebhookDelivery {
  id              String    @id @default(uuid())
  webhookId       String
  eventType       String
  payload         Json
  status          String    // "pending" | "succeeded" | "failed" | "dead_letter"
  attempt         Int       @default(0)
  lastAttemptAt   DateTime?
  nextAttemptAt   DateTime?
  responseStatus  Int?
  responseBody    String?   // truncated to first 1KB
  createdAt       DateTime  @default(now())

  webhook         Webhook   @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([status, nextAttemptAt])
}
```

### Delivery contract

**Headers on every webhook delivery:**

```
POST <customer_url>
Content-Type: application/json
X-Epic-App: <app_name>           // foundry-ims, dispatch, rally, etc. — disambiguates if customer integrates with multiple Epic apps
X-Epic-Event: <event_type>
X-Epic-Delivery-Id: <delivery_uuid>
X-Epic-Signature: t=<timestamp>,v1=<hmac_sha256_hex>
X-Epic-Timestamp: <unix_seconds>

{ "id": "...", "type": "...", "data": {...} }
```

**Signature:** `HMAC-SHA256(secret, "{timestamp}.{body}")`. Customers verify within a 5-minute window to prevent replay.

**Payload size cap: 1 MB.** Larger events are truncated with a flag indicating truncation; customers can fetch the full resource via API.

**Request timeout: 10 seconds.** Non-2xx response = failure.

### Secret rotation

When a webhook owner calls `POST /webhooks/:id/rotate-secret`, the new secret is returned once and stored. To avoid breaking in-flight deliveries (worker has the old secret in memory while customer rotates):

- **Both old and new secrets sign for a 24-hour overlap window.** The header carries multiple `v1` segments:
  ```
  X-Epic-Signature: t=<ts>,v1=<hex_with_new>,v1=<hex_with_old>
  ```
- Customers verify against either value. Once they confirm new-secret-only deliveries are working, no migration on their end is required.
- After 24 hours, the old secret is dropped from signing. Subsequent deliveries carry only `v1=<hex_with_new>`.
- Schema: store `secret` (current) and `previousSecret` + `previousSecretValidUntil` on the `Webhook` row.

### Retry policy and warnings

- **Up to 8 attempts** with exponential backoff: 1m, 5m, 15m, 1h, 6h, 12h, 24h, 24h
- After 8 attempts, delivery moves to `dead_letter` status (kept for 30 days, replayable)
- **Warning at 3 consecutive failures** — email to webhook owner via §11 transactional notification
- **Email + auto-disable at 5 consecutive failures** — `isActive: false`, transactional notification sent

### Delivery worker pattern

Standardized across apps:

1. Trigger event publishes to a queue (in-memory, BullMQ, or app-appropriate)
2. Worker picks up, finds active webhooks subscribed to the event type
3. For each: create a `WebhookDelivery` row, attempt POST
4. On success: mark `succeeded`, increment `lastSuccessAt`
5. On failure: increment attempt, schedule next retry, increment `failureCount`
6. Cron sweeps for `pending` deliveries past `nextAttemptAt`

### Ordering guarantee

**Webhook ordering is best-effort.** Retries with exponential backoff mean events can arrive out of order. **Treat the resource's own state as authoritative** — webhooks are notifications, not source of truth. Document this clearly to customers.

### API surface

```
GET    /webhooks                              → list this org's webhooks
POST   /webhooks                              → create (returns secret once)
PATCH  /webhooks/:id                          → update url / events / isActive
DELETE /webhooks/:id
POST   /webhooks/:id/rotate-secret            → returns new secret once
GET    /webhooks/:id/deliveries               → recent deliveries (paginated)
POST   /webhooks/:id/deliveries/:dId/replay   → re-attempt a failed delivery
GET    /webhooks/:id/usage                    → current rate usage (see §22)
```

### UI surfaces

- **Settings → Webhooks** page — list of webhooks with active/disabled toggle, event subscriptions, last delivery status
- **Per-webhook detail page** — recent delivery log with status, request/response, replay button on failures
- **One-time secret display** on creation
- **Usage meter** showing current minute's deliveries vs. cap (per §22)

---

## 13. API Keys ✅

For programmatic / CI access to the app's API.

### Pattern

- Prefix every key with `<app>_`: `fims_<random>`, `disp_<random>`, etc.
- Store the **hash** (sha256) in the DB, not the plaintext. Show the plaintext to the user **once** on creation.
- Each key has: `name`, `role` (same enum as `User.role`), `lastUsedAt`, `revokedAt`, `expiresAt` (optional).
- ClerkGuard short-circuits when token starts with `<prefix>_` — validates against `ApiKey` table instead of Clerk JWT.

### Schema

```prisma
model ApiKey {
  id          String    @id @default(uuid())
  orgId       String
  name        String
  prefix      String    // first 8 chars for display
  hash        String    @unique  // sha256(plaintext)
  role        UserRole
  scopes      String[]  // reserved for future granular scopes (read-only keys, etc.)
  lastUsedAt  DateTime?
  expiresAt   DateTime?  // optional expiration
  revokedAt   DateTime?
  createdAt   DateTime  @default(now())
}
```

### Lifecycle

- **Optional expiration** — keys can have `expiresAt` set on creation. Past-expiry keys reject auth.
- **"Older than 90 days" view** — Settings page surfaces stale keys with a rotation prompt.
- **Rotation reminders** — automated notification at 90-day mark for keys without expiration.
- **`scopes` field reserved** — full granular scope implementation is a future feature; field is in schema now to avoid migration later.

### Generation

```ts
import { randomBytes, createHash } from 'crypto';

const raw = randomBytes(32).toString('base64url');
const plaintext = `fims_${raw}`;
const hash = createHash('sha256').update(plaintext).digest('hex');
```

### UI surfaces

- **Settings → API Keys** subpage — table of keys (name, prefix, role, last used, expiration), Create / Revoke actions.
- One-time display of plaintext on creation, with a Copy button.
- **`apikey.created` notification** fires (transactional class).

---

## 14. Activity Log ✅

Per-org change history for **business events**, queryable by entity. (Security events go to §14.5 Audit Log.)

### Schema

```prisma
model ActivityLog {
  id          String   @id @default(uuid())
  orgId       String
  entityType  String
  entityId    String
  action      String
  summary     String
  source      String   // "manual" | "webhook" | "import" | "cron" | "partner_seat" | ...
  sourceId    String?
  metadata    Json?
  createdBy   String?  // local User.id (NOT email — see §15)
  createdAt   DateTime @default(now())

  @@index([orgId, entityType, entityId])
  @@index([orgId, createdAt])
}
```

> **Migration note:** Foundry currently stores `createdBy` as email. Migration required to backfill from email → User.id, with edge cases for emails of users who've left the org.

### Source field and partner attribution

The `source` field includes `"partner_seat"` for actions taken during partner-controlled trials, ensuring a clear audit trail before client ownership transfer.

### Retention

- **Hot retention:** queryable in app for 90 days
- **Cold retention:** archived to S3 (or equivalent), downloadable on request, 7 years
- Apps with low activity volume can keep all in DB with documented size budget

### How to use

```ts
await this.activityLog.log(orgId, {
  entityType: 'Organization',
  entityId: orgId,
  action: 'deleted',
  summary: 'Organization deleted',
  source: 'manual',
  createdBy: userId,
});
```

### UI surfaces

- **`/activity` page** — filterable timeline of org-wide events.
- **Per-entity pages** — embedded "Recent activity" section filtered by `entityType` + `entityId`.

---

## 14.5 Security Audit Log 📋

A separate log for **security-sensitive events**, distinct from the business activity log. Compliance teams will query this directly.

### Why separate?

- Compliance audits filter for security events; mixing with business events makes that harder
- Security audit logs typically have longer retention (7+ years for SOC 2 / regulatory requirements)
- Different access controls — audit log is read-restricted to OWNER + ADMIN with `audit.read`

### Schema

```prisma
model AuditLog {
  id          String   @id @default(uuid())
  orgId       String
  userId      String?  // local User.id — actor of the action
  action      String   // see baseline list below
  resource    String?  // affected entity (e.g., "user:abc-123", "apikey:xyz")
  ipAddress   String?
  userAgent   String?
  metadata    Json?
  createdAt   DateTime @default(now())

  @@index([orgId, createdAt])
  @@index([userId, createdAt])
  @@index([action, createdAt])
}
```

### Baseline events every app logs

- `auth.login` — successful authentication
- `auth.logout` — session terminated
- `auth.login_failed` — failed authentication attempt
- `auth.password_changed`
- `auth.new_device_login`
- `user.role_changed` — includes from/to roles in metadata
- `user.removed` — user kicked from org
- `user.deleted` — GDPR deletion completed
- `org.closed` — org permanently deleted
- `apikey.created`
- `apikey.revoked`
- `webhook.created`
- `webhook.disabled` — manual or auto-disable
- `permission.changed` — when a role's permission set is modified
- `partner_seat.added`
- `partner_seat.removed`

### App-specific extensions

Each app evaluates additional security events relevant to its domain. Examples:

- Dispatch: `ticket.reassigned_to_partner`, `ticket.priority_escalated_by_admin`
- Rally: `attribution_model.changed`, `tracking_pixel.regenerated`
- Foundry: `inventory.bulk_import`, `bigcommerce_credentials.rotated`

### Retention

**7 years.** Audit log retention is a compliance requirement (SOC 2, GDPR, regional data laws). Cold-storage past 1 year is acceptable.

### UI surfaces

- **`/audit` page** — filterable by action, user, date range. Gated by `audit.read`.
- **Settings → Security Activity** card — links to full audit log.

---

## 15. Data Export & Right to Deletion

Two related but distinct flows. Both required for GDPR/CCPA compliance.

### 15.1 Data Export ✅

Owner-level export of all org data as a ZIP of CSVs.

```
GET /orgs/me/export    → application/zip stream
```

Returns one CSV per top-level model. Stream-encoded so it doesn't buffer in memory.

UI: **Settings → Data Export** card with "Export All Data" button. Auth'd to OWNER.

**Post-close availability:** When an org is closed (§3.8), the final export remains available for **30 days** before final purge. Closure dialog warns the OWNER and offers immediate export before deletion.

### 15.2 User Right to Deletion 🧭

A **user** (not org) can request their personal data be deleted, independent of whether the org stays open.

#### Endpoint

```
DELETE /users/me/request-deletion
  body: { confirm: <user's_email>, reason?: string }
  → { confirmationEmailSent: true, deletionScheduledFor: "<ISO_date>" }
```

#### Flow

1. User initiates deletion request from Settings → Privacy.
2. They confirm by typing their email address.
3. App sends a confirmation email (transactional notification class) with a final confirm link.
4. User clicks the link — deletion is scheduled for **30 days out** (grace period for accidental requests).
5. During the grace period, the user can cancel via Settings → Privacy.
6. After 30 days, a scheduled job runs the deletion procedure.

#### Immediate deletion (no grace period)

GDPR Article 17 requires deletion "without undue delay." The 30-day grace exists for the user's protection — but the user can opt out of it. The Privacy page includes a **"Delete immediately"** option behind a stronger confirmation gate (re-type email + checkbox: "I understand this is irreversible and there is no recovery period"). When chosen, the deletion job runs within 24 hours instead of 30 days. Use this for users who explicitly invoke their right to immediate erasure.

#### Tombstone deletion model

We use the **tombstone model**: the `User` row survives but PII is nulled. This keeps foreign keys intact (avoiding orphaned `Referral`, `ActivityLog`, `AuditLog` rows) while removing the personal data.

**What gets nulled / deleted:**

- `User.email` → `null`
- `User.name` → `null`
- `User.clerkUserId` → `null`
- `User.deletedAt` → set to current timestamp (marks as tombstone)
- **Clerk user** — deleted via `clerk.users.deleteUser()`
- `ActivityLog.createdBy` rows referencing this user — preserved (the `createdBy` is a `User.id`, which still exists; the user's PII is just gone)
- `AuditLog.userId` rows — same; the local ID survives, the PII does not
- `Notification` rows for this user — deleted
- Email/name in any snapshot fields (Order recipient names, ticket attribution) — replaced with `"[deleted user]"`

**What survives:**

- `User.id` (tombstone — used for FK integrity only)
- `Referral` rows referencing the org (the org keeps its referral attribution; the original user's identity is gone)
- Aggregated analytics (no PII)
- Financial records (invoices, payouts) — required by tax law
- Audit trail of the deletion itself

#### Backups

GDPR requires backups be re-scrubbed or excluded from restoration in a documented way.

**Standard policy:**
- **Backups expire after 30 days by default.** No general-purpose backup older than 30 days exists in any system.
- **On any restoration from backup, the deletion job is re-run** before the restored database goes live. Documented runbook required.
- This is the most-failed audit point in real GDPR enforcement; it MUST be in the runbook.

**Compliance-driven extensions:** Apps processing financial, medical, or other regulated data may be required to retain backups longer (often 7 years for financial / SOX, varies for healthcare). When that's the case:
- Document the retention requirement and its legal basis in the app's compliance runbook.
- The deletion-on-restore obligation still applies — every restore from a long-retained backup re-runs the deletion job for any users whose deletion completed before the backup snapshot.
- Long-retained backups should live in encrypted, access-restricted cold storage with audit logging on access.

#### Confirmation audit

Use **HMAC** instead of plain SHA256 to prevent rainbow-table reversal:

```prisma
model DataDeletionAudit {
  id           String   @id @default(uuid())
  // userId omitted - the original userId is itself PII (links to a real person via Clerk history)
  emailHmac    String   // HMAC-SHA256(server_secret, email_at_deletion_time)
  requestedAt  DateTime
  completedAt  DateTime
}
```

The HMAC requires a server-side secret, making it irreversible without that secret. Plain `sha256(email)` is trivially reversible with breach lists.

#### Edge case: sole org owner

If the deleting user is the sole owner of an org, the deletion request first prompts: "You're the sole owner of `<Org>`. You must either transfer ownership or close the org before deleting your account."

#### Schema

```prisma
model DataDeletionRequest {
  id              String    @id @default(uuid())
  userId          String
  emailAtRequest  String    // captured at request time; cleared on completion
  reason          String?
  requestedAt     DateTime  @default(now())
  confirmedAt     DateTime?
  scheduledFor    DateTime?
  cancelledAt     DateTime?
  completedAt     DateTime?

  @@index([userId])
}
```

### UI surfaces

- **Settings → Privacy** page with two cards:
  - **Export My Data** — user-scoped export
  - **Delete My Account** — initiates deletion flow

---

## 16. Session Permission Re-Validation 🚧

**Every authenticated request re-validates the user's role and org membership against the local DB.** Sessions do NOT cache permissions until token expiry.

### Why

If an Admin is demoted to Viewer, the change must take effect quickly — not after their session expires. Silent stale permissions are a security and UX problem.

### Implementation

In `ClerkGuard` (or equivalent), after validating the Clerk JWT:

1. Look up the local `User` row by `clerkUserId` + active `orgId`.
2. Verify the user is still a member of the active org.
3. Check `User.role` for the current value (don't trust JWT claims for role).
4. If the user's role has changed or they've been removed: invalidate the session (return 401 with code `permission_changed`) and the frontend forces re-auth.

### Cache TTL

A 5–30 second cache TTL is acceptable for most apps and reduces DB load. Security-sensitive flows (admin actions, billing changes, partner seat permission changes) should bypass cache for sub-second invalidation. Document the TTL choice per app — 30s of stale Admin-vs-Viewer permissions is acceptable as a default; some apps may want tighter.

### Frontend handling

When the API returns a 401 with code `permission_changed`:
- Clear local auth state
- Redirect to `/login` with a flash message: "Your permissions have changed. Please sign in again."

> **Foundry status:** Reads role fresh on every request ✅, but does not currently emit `permission_changed` 401. Audit item.

---

## 17. Error Tracking & Observability (Sentry)

Every app uses **Sentry** for error tracking and performance monitoring via the shared **`@epic/sentry-config`** library.

### Required setup

```
@sentry/nestjs (API) or @sentry/react (admin) or @sentry/astro (marketing)
@sentry/profiling-node (API only)
@epic/sentry-config (shared config + PII redaction)
```

Initialization MUST happen in `instrument.ts` loaded **before any other imports**:

```ts
// src/instrument.ts (loaded first in main.ts)
import * as Sentry from '@sentry/nestjs';
import { nodeProfilingIntegration } from '@sentry/profiling-node';
import { createBeforeSend } from '@epic/sentry-config';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  integrations: [nodeProfilingIntegration()],
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  beforeSend: createBeforeSend({ appName: 'foundry-ims' }),
});
```

`SentryGlobalFilter` catches all exceptions on the NestJS app.

### `@epic/sentry-config` PII redaction

The shared `beforeSend` filter automatically redacts:

- **Headers:** `Authorization`, `X-API-Key`, `Cookie`, any header with `auth` or `secret` in the name
- **Body fields:** any field named `password`, `secret`, `token`, `apiKey`, `apikey`, `creditCard`
- **URL params:** email addresses in path or query string replaced with `[email]`
- **Custom redactors per-app** can extend the base set

This is one place to maintain redaction rules — all apps stay in sync.

### Required env vars

```
SENTRY_DSN=<per-app DSN>
APP_VERSION=<git sha or release tag>
```

**Canonical env var: `APP_VERSION`** for the release identifier across every app. (Foundry currently uses `SENTRY_RELEASE` — audit item to rename.)

### Logging conventions

| Level | When to use |
|---|---|
| `error` | Unexpected exceptions, failed external calls that should have succeeded, data integrity issues |
| `warn` | Recoverable failures, degraded behavior, configuration gaps, retried operations |
| `info` | Significant business events (org created, partner trial started), abuse signals (disposable email blocked) |
| `debug` | Development-only diagnostics |

### PII never goes into log lines

Application logs land in whatever sink the host provides — CloudWatch (AWS / ECS / Lambda), Render's log viewer, Cloudflare Logs / Tail Workers, Vercel's runtime logs, etc. Different sinks, same rule:

**Never log PII.** Pass IDs and resolve in tools.

```ts
// ❌ Bad — email lands in the log sink
logger.log(`User ${user.email} requested deletion`);

// ✅ Good — only IDs hit the sink
logger.log(`Deletion requested by user`, { userId: user.id, orgId: user.orgId });
```

This rule is hosting-agnostic: it applies the same way whether logs flow into AWS CloudWatch, Render, Cloudflare, Vercel, or wherever the next app lands. Sentry redaction (above) is a safety net for when logs do reach Sentry; the primary defense is not logging PII in the first place.

### Required Sentry tags

Every captured event includes:

- `org_id` — the active org
- `user_id` — local User.id (NOT email — PII)
- `route` — request path
- `app` — app name (foundry-ims, dispatch, rally, etc.)
- `app_version` — release identifier

For manual `Sentry.captureException`, use `withScope`:

```ts
Sentry.withScope((scope) => {
  scope.setTag('org_id', orgId);
  scope.setTag('action', 'webhook_delivery');
  scope.setExtra('event_type', event.type);
  Sentry.captureException(error);
});
```

### Graceful degradation patterns

- **Fail-soft for non-critical paths.** EmailService.isConfigured() returns false → log a warning, return null, don't throw.
- **Fail-loud for data integrity.** Database write fails → throw, let Sentry capture, return 500.
- **Always include context.** When catching an exception to convert to a user-friendly response, capture the original to Sentry first.
- **Idempotent retries.** External integrations should retry with exponential backoff — but only for idempotent operations.

---

## 17.5 Operational Patterns 📋

### Health checks

Every app exposes `GET /health`:

```json
{
  "status": "ok",
  "timestamp": "2026-05-02T02:19:43Z",
  "version": "1.2.3",
  "checks": {
    "database": "ok",
    "clerk": "ok",
    "resend": "ok",
    "dispatch": "ok"
  }
}
```

Returns 200 if all critical checks pass, 503 if any critical check fails. Load balancers ping this for liveness. Sub-checks help debugging without manual intervention.

### Cron conventions

Apps using a cron library (node-cron, BullMQ, etc.) follow this pattern:

- Cron jobs defined in a single file (e.g., `src/cron/index.ts`)
- Each job has: `name`, `schedule` (cron expression), `handler`, optional `timezone`
- All jobs log to Sentry with tags: `cronJob: "<name>"`, `started_at`, `duration_ms`, outcome
- Failures captured as exceptions; recovered failures logged at `warn`
- Standard timezone handling: jobs default to UTC unless explicitly overridden

Standard scheduled jobs every app may need:

- Trial expiry sweep (when Throttle ships)
- Webhook retry sweep
- Deletion grace period sweep (§15.2)
- Stale API key reminder (§13)
- Activity log cold-storage migration (§14)

### Database migrations

Standardized on **Prisma migrations** across the portfolio:

- Migrations live in `prisma/migrations/`
- Naming: `<timestamp>_<description>` auto-generated by Prisma
- Apply via `prisma migrate deploy` in CI/CD
- Never edit a committed migration; always create a new one
- Include rollback notes in PR description for non-trivial migrations

### CI / auto-versioning

- Semantic versioning from git tags
- PR-title-driven semver (e.g., `feat:`, `fix:`, `breaking:`) auto-bumps version
- Version surfaced via `/health.version` and `APP_VERSION` env var
- Changelogs auto-generated from PR titles

### API versioning

- Standardized on URL prefix: `/api/v1/`
- Breaking changes go to `/api/v2/`
- Version sunset announced via headers (`Sunset:`, `Deprecation:`) at least 6 months before removal

### Environments (preview / staging / production)

Every app runs at least three environments:

| Environment | Clerk | Sentry | Database | Hosting |
|---|---|---|---|---|
| **Production** | Per-app prod Clerk project | Per-app Sentry project (env tag `production`) | Production primary | Per-app choice |
| **Staging / preview** | **Per-app staging Clerk project** (separate from prod) | Same Sentry project, `environment: "staging"` tag | Per-environment DB (no prod data) | Per-app choice |
| **Local dev** | Clerk dev instance | Sentry disabled or `environment: "development"` | Local DB | Local |

**Hosting is a per-app choice.** Render, Vercel, Cloudflare Workers, AWS, Neon for Postgres — apps pick what fits their workload. The standard locks down the *identity* boundaries (separate Clerk project per environment, Sentry env tags, no prod data in lower envs), not the runtime.

Critical rules:
- **Preview / staging never points at production Clerk.** Avoids accidentally provisioning real users into Clerk dev with prod credentials, and keeps trial signups from polluting prod analytics.
- **Preview / staging never reads production data.** Use anonymized staging snapshots or seeded fixtures (see Seed Conventions below).
- **Same `app_version` and Sentry release across environments**, distinguished by the `environment` tag — so a release rolls through dev → staging → prod with traceable telemetry.

### Seed conventions (Futurama theme)

Every app ships a seed script that populates a `test` instance with realistic-but-obviously-fake data. **Standard theme: Futurama.** Fry is an OWNER, Leela is an ADMIN, Bender is a MEMBER, the Professor is a VIEWER, Planet Express is the org name, etc. Each app maps Futurama into its own domain:

- Foundry IMS: Planet Express has products (slurm, popplers), suppliers (Mom's Friendly Robot Co.), warehouses, orders.
- Dispatch: Planet Express has tickets ("Where's my package, the package is in space"), the Professor as a partner seat.
- Rally: Planet Express tracks attribution for their delivery ad spend.

Why this matters:
- **Same canonical fixture set across apps** makes onboarding new devs faster — they recognize the cast immediately.
- **Obvious that it's fake.** No risk of mistaking seed records for real customers.
- **Nice to demo with.** Screenshots and Loom videos with Futurama fixtures look intentional, not amateur.

Convention: `scripts/seed-test.ts` (or app-equivalent) creates a complete `test` org wired with the Futurama cast, runnable locally and in CI. Each role has at least one Futurama character, every entity type has at least one example.

### Internationalization (i18n)

**v1 of every app is English-only.** No i18n framework, no locale switching, no per-region copy. This is a deliberate scope decision — adding i18n later is mechanical (wrap user-facing strings in a translation function); adding it preemptively introduces complexity without payoff.

When an app has a justifiable i18n need (specific market, regulatory copy translation, customer demand), the standard adoption is:
- **Library:** `next-intl` (Next.js apps) or `astro-i18n` (Astro marketing sites).
- **Locale data lives next to source** — `src/locales/en.json`, `src/locales/es.json`. Not a separate repo.
- **English is canonical.** Other locales are translations of the English source — never the reverse.
- **Date/number formatting** uses `Intl.*` APIs, not custom logic.

Until that need lands, English-only stays the standard. Don't preemptively wrap strings.

### Accessibility (WCAG 2.1 AA)

Every app commits to **WCAG 2.1 Level AA conformance** as the minimum. This is the bar most enterprise procurement teams + government buyers expect; falling below blocks deals.

Practical baseline:
- **Keyboard navigation** through every interactive surface — no mouse-only widgets.
- **Visible focus indicators** on every focusable element (Tailwind's `focus-visible:` is fine).
- **Sufficient contrast ratios** — 4.5:1 for body text, 3:1 for large text. Tailwind's default palette mostly clears this; verify when picking a brand color.
- **`<label>` associations** on every form input.
- **Alt text** on every meaningful image; empty `alt=""` on decorative ones.
- **ARIA only when native HTML can't express the semantic.** Don't add ARIA-soup to elements that are already semantic.

Tooling: `axe-core` integrated via `@axe-core/playwright` in E2E tests, or as a manual audit step before major releases. CI gate is optional but encouraged.

This is a *commitment*, not a one-time checklist — every new component or page is built to meet AA. If a feature can't, we don't ship it; we redesign.

### Throttle webhook stub (placeholder until billing doc lands)

While Throttle integration is ⏸️ blocked, every app should reserve the webhook endpoint shape so adoption is a code change, not a schema migration:

```ts
// POST /webhooks/throttle (per-app endpoint, signed by Throttle)
// Standard Throttle event shape (subject to confirmation in the billing doc):
{
  id: "evt_...",
  type: "subscription.created" | "subscription.updated" | "subscription.canceled"
      | "invoice.paid" | "invoice.payment_failed" | "trial.ending" | "trial.expired",
  data: { /* Throttle resource snapshot */ },
  occurredAt: "<ISO 8601>",
}
```

Reserve a `BillingEvent` table to log inbound Throttle events for auditability before processing:

```prisma
model BillingEvent {
  id          String   @id @default(uuid())
  externalId  String   @unique          // Throttle event id (idempotency key)
  type        String                    // Throttle event type
  orgId       String?                   // resolved from event payload
  payload     Json
  processedAt DateTime?
  receivedAt  DateTime @default(now())

  @@index([orgId, type])
}
```

This lets apps stub the webhook handler now (verify signature, persist event, no-op processing) and wire actual handlers when the billing doc arrives.

---

## 18. Marketing Site Contract

Every marketing site must:

1. **Capture `?r=` from any landing URL** into a 30-day `affiliateR` cookie (samesite=lax). Validate against tightened regex `^[A-Za-z0-9]{8}$` (matches the actual generated code length, prevents arbitrary code stuffing).
2. **Last-touch wins on overwrite.** New `?r=` overwrites previous cookie value.
3. **Rewrite all `<a href*="app.">` links** when the cookie is set:
   - Reroute root `/` and `/login` paths to `/signup`.
   - Append `?r=<cookie>` to every admin link (preserve any existing `?r=`).
4. **Have a "Sign Up Free" or "Get Started" CTA** that points at `app.<domain>/signup` directly (not `/login`).
5. **`/partners` landing page** 🧭 — explains the partner program.
6. **`/partners/apply` form** 🧭 — submits to API.
7. **Cookie consent banner** — no analytics cookies set without explicit consent. The affiliate cookie is functional and can argue exemption, but document the stance.

Reference: `astro-foundryims/src/layouts/Layout.astro`. Cookie capture and link rewriting:

```js
(function () {
  var COOKIE = 'affiliateR';
  var MAX_AGE = 30 * 24 * 60 * 60;
  var VALID = /^[A-Za-z0-9]{8}$/;  // tightened to exact code length

  // 1. Capture ?r= into cookie (last-touch overwrites)
  var ref = new URLSearchParams(location.search).get('r');
  if (ref && VALID.test(ref)) {
    document.cookie = COOKIE + '=' + encodeURIComponent(ref) +
      ';max-age=' + MAX_AGE + ';path=/;samesite=lax';
  }

  // 2. Rewrite admin links to /signup?r=<code>
  function getCookie(n) {
    var m = ('; ' + document.cookie).split('; ' + n + '=')[1];
    return m ? decodeURIComponent(m.split(';')[0]) : null;
  }
  function rewrite() {
    var c = getCookie(COOKIE);
    if (!c || !VALID.test(c)) return;
    document.querySelectorAll('a[href*="app."]').forEach(function (a) {
      try {
        var u = new URL(a.href);
        if (u.searchParams.has('r')) return;
        if (u.pathname === '/' || u.pathname === '/login') u.pathname = '/signup';
        u.searchParams.set('r', c);
        a.href = u.toString();
      } catch (_) {}
    });
  }
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', rewrite);
  } else { rewrite(); }
})();
```

---

## 19. Domain Conventions

Every app uses its own root domain. The structure within that domain is consistent:

| Subdomain | Purpose | Example (Foundry IMS) |
|---|---|---|
| `<rootdomain>` | Marketing site | `foundryims.com` |
| `app.<rootdomain>` | Admin / product app | `app.foundryims.com` |
| `api.<rootdomain>` | API endpoint | `api.foundryims.com` |
| `<app>.<rootdomain>` | Email sender subdomain | `ims.foundryims.com` |

**Each app gets its own root domain.** Not subdomains under `epic.dev`.

---

## 20. New-App Standardization Checklist

When bootstrapping the next app, replicate in this order:

### Foundation
- [ ] **Root domain registered** + DNS configured per §19
- [ ] **Clerk projects: prod + staging** with the §3.1 dashboard config (`force_organization_selection: false`)
- [ ] **Sentry project** + `instrument.ts` + `@epic/sentry-config` per §17 — `APP_VERSION` env var
- [ ] **Resend workspace** + sender domain per §9
- [ ] **`Organization` + `User` models** per §3.4 (include `accountType`, `affiliateCode` on Organization, `deletedAt` on User, NO `@unique` on `clerkUserId`, nullable `email` / `name` / `clerkUserId` for tombstones)
- [ ] **`ClerkGuard`** with permission re-validation per §3.5 and §16
- [ ] **Disposable email blocking** at signup per §3.3
- [ ] **Smooth signup path** — auto-create org on first load per §3.2
- [ ] **Per-environment isolation** — staging Clerk + Sentry env tag + per-env DB per §17.5

### Core features
- [ ] **Org create / switch / leave / close flows** per §3.8
- [ ] **Org switcher dropdown** in the header per §3.7
- [ ] **Users & roles** per §4 (UserRole enum, shared baseline permissions, `RequirePermission` decorator, Settings → Users)
- [ ] **`Referral` model + `Organization.affiliateCode`** per §6 — note: per-org, not per-user
- [ ] **Affiliate endpoints** per §6
- [ ] **`/signup?r=` route + `AffiliateAttribution` + Settings → Affiliate** per §6
- [ ] **Marketing site script** per §18

### Communication
- [ ] **`EmailService` wrapper** with Resend + `react-email` per §9
- [ ] **Notification model + bell + Settings → Notifications** with per-method preferences AND transactional class per §11
- [ ] **Dispatch brand** provisioned + `SupportService` wrapper + Help menu + 3 support pages per §10

### Infrastructure
- [ ] **API keys (`<prefix>_*`) + Settings → API Keys** per §13
- [ ] **`ActivityLogService` + `/activity` page** per §14 (use `User.id` for `createdBy`, NOT email)
- [ ] **`AuditLogService` + `/audit` page** per §14.5 — all baseline events instrumented
- [ ] **Data export endpoint + Settings card** per §15.1
- [ ] **User deletion request flow + Settings → Privacy** per §15.2 (tombstone model, HMAC audit, immediate-delete option)
- [ ] **Backup runbook** documenting 30-day expiry (or longer if compliance-driven) + deletion-rerun-on-restore
- [ ] **Rate limiting** per §22 with usage metrics (`@upstash/ratelimit` default)
- [ ] **Health check** endpoint per §17.5
- [ ] **Cron jobs** named/logged per §17.5
- [ ] **`/api/v1/` prefix** per §17.5
- [ ] **PII never in log lines** — pass IDs, resolve in tools per §17
- [ ] **Seed script** with Futurama theme — `scripts/seed-test.ts` per §17.5
- [ ] **WCAG 2.1 AA conformance** baseline per §17.5
- [ ] **English-only** for v1 — no preemptive i18n wrapping per §17.5
- [ ] **Throttle `BillingEvent` table reserved** + signed webhook stub per §17.5

### Deferred (build when ready, schema reserves now)
- [ ] **Outbound webhook infrastructure** per §12 — `Webhook` + `WebhookDelivery` models in initial schema even if not wired
- [ ] **Partner program** per §7 — port wholesale once first app ships it; `accountType` + `Referral.referralType` already in schema
- [ ] **Trial period + Throttle integration** per §8 / §23 — separate billing standardization doc, integration points reserved

---

## 21. Principles

1. **Standardize the boring, customize the value.** Signup, billing, support, affiliate, RBAC, notifications, API keys, observability — identical across every app. The product itself is where each app earns its keep.

2. **Visual design is not standardized.** Each app earns its own look and feel. We standardize behavior, not appearance. Common UI patterns (drag handles for ordering, header sorting, column controls) are encouraged when applicable but not mandated.

3. **Clerk owns identity. Throttle owns billing.** Within a given app, Clerk is the source of truth for users and orgs. Throttle handles all transactions. Local DB mirrors what's needed for joins. Cross-app identity is not a thing in v1.

4. **Attribution at signup, not retroactively.** Affiliate and partner referral credit is captured when the org is created. No after-the-fact claims.

5. **Action is proof of attribution.** Partner-tier referral credit requires the partner to physically create the trial. No forms, no disputes.

6. **Two referral tiers, one model.** Both `affiliate` and `direct` referrals earn 10% recurring, no cap. Partner status unlocks the partner dashboard, partner seats, and the trial-creation flow — but the commission rate is the same.

7. **Schema reserves the future.** `convertedAt`, `rewardStatus`, `accountType`, `Webhook`, `AuditLog`, `scopes` exist in the model even before billing/partner/scope code does. New apps copy them so the migration on activation is zero-schema.

8. **External services stay server-side.** Clerk, Dispatch, Resend, Throttle, and webhook secrets never reach the browser.

9. **One-time secrets, hashed at rest.** API keys, webhook secrets, deletion confirm tokens are shown to the user once, hashed in the DB. Never recoverable.

10. **Permissions are checked on every request.** Sessions don't cache role data beyond a short documented TTL. A demotion takes effect immediately, not at token expiry.

11. **Privacy and deletion are first-class.** Right-to-deletion is built in, not bolted on. PII is in nullable fields only. Activity logs reference user IDs, not emails. Audit logs use HMAC for any necessary email hashes. Backups have documented 30-day retention with deletion-rerun-on-restore.

12. **Defense in depth on tenancy.** Every query filters by orgId. Every API request is scoped to the caller's active org. CI lint flags unscoped database queries. Postgres RLS as a future hardening layer.

13. **PII boundaries are explicit.** No PII in Sentry tags, URL paths, log lines, or aggregate analytics. PII lives in `User`, `Organization`, and explicit snapshot fields only.

14. **Rate-limit by default.** Every public endpoint and per-key access has a rate limit. Default deny, raise as needed. Usage metrics are visible to users so they see limits coming.

15. **Transactional notifications bypass preferences.** Users can't opt out of deletion confirmations, security alerts, or payment failures. Compliance and account safety override convenience.

---

## 22. Rate Limiting 📋

Every public surface is rate-limited. Usage metrics are exposed so users see consumption before they hit caps.

### Baseline limits

| Surface | Limit | Notes |
|---|---|---|
| **Per-IP signup attempts** | 10 / hour | Blocks bot signup farms; doesn't friction real users. |
| **Per-API-key request budget** | Standardize concept; numbers in Throttle billing tier doc | Tier-based; lower tiers get lower budgets. |
| **Per-org webhook deliveries (outbound)** | 1000 / minute baseline | Configurable higher for ecommerce-heavy customers. |
| **Per-org Sentry error submission** | 1000 errors / minute | App-side cap on what we submit to Sentry — defends Sentry budget against runaway error loops in our own code. (Sentry has its own quota separately; this is upstream of that.) |
| **Per-IP affiliate `/attribute`** | 30 / hour | Anti-fraud; blocks attribution farming. |
| **Per-IP password reset / magic link** | 10 / hour | Anti-brute-force on auth flows. |

### Usage metrics — visibility before failure

**Every rate-limited resource must expose usage metrics** so users see how close they are to limits. Implementation patterns:

- **API responses** include `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers
- **Settings pages** show real-time consumption gauges (e.g., "850 / 1000 webhooks this minute")
- **`GET /webhooks/:id/usage`**, `GET /apikeys/:id/usage` etc. surface usage programmatically
- **Notifications fire at 80% threshold** (transactional class) so users have time to react

This is a deliberate operational stance: **rate-limit transparently, not silently.** Hitting a hard limit without visibility is a worse experience than seeing the limit approach and either upgrading tier or reducing load.

### Implementation

- **Default library: `@upstash/ratelimit`** with the Upstash Redis backend (or self-hosted Redis if the app already runs one). Edge-friendly, distributed, simple sliding-window primitive. Apps free to substitute (`express-rate-limit + rate-limit-redis`, fastify-rate-limit, etc.) if they have a load-bearing reason — but document the divergence.
- 429 responses include `Retry-After` headers and structured error bodies (`{ error: "rate_limited", retryAfter: <seconds>, limit, remaining: 0 }`).

---

## 23. Billing & Throttle Integration ⏸️ (stub)

Epic apps use **Throttle** for payments and billing. Standardized integration patterns will be documented in a separate billing standardization doc.

This v3 doc reserves integration points for:

- Trial creation, expiration, and reminder emails (§8)
- Trial-to-paid conversion detection and `Referral.convertedAt` updates (§7, §8)
- Partner commission calculation (10% recurring, monthly accrual, paid month N+1) (§7)
- Payout processing (PayPal Mass Pay direction; details in billing doc) (§7)
- Subscription lifecycle webhook events (`subscription.changed`) (§12)
- Dunning and payment failure handling (§11 — `billing.payment_failed` is transactional notification class)
- Per-API-key request budget tier definitions (§22)

**Sections currently blocked on the billing doc:**

- §7 Partner Program — commission calculation and payout flows
- §8 Trials — conversion detection and expiration enforcement
- §11 Notifications — `billing.payment_failed` semantics
- §22 Rate Limiting — per-tier API key budgets

Until the billing doc lands, these features remain ⏸️ blocked. Schema reserves the relevant fields so activation is a code change, not a migration.
