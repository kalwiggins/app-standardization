# Epic Design Labs — Standard Features (v3.6)

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

### Changes in v3.6

Codifies the team-artifact + dev-workflow conventions that were already practiced informally:

- **§17.5 Team artifacts** — added a new subsection naming the four artifacts every repo carries: `README.md`, `CHANGELOG.md`, `docs/roadmap.md`, `CLAUDE.md`. Locked in `roadmap.md`'s four-section format (Shipped / In flight / Planned / Parked) with example. Roadmap is product-language, not engineering-task-language.
- **§17.5 Pull request workflow** — explicit: every change goes through a PR, no direct pushes to `main`. Branch naming (`<author>/<short-description>`), PR title prefixes drive semver (already in CI section), PR body structure (Summary / Changelog / Test plan), squash-merge default, merge gate, branch protection at the GitHub level.
- **§20 checklist** — added branch protection setup and the four team artifacts to the new-app bootstrap.

Version numbering itself was already covered (§17.5 CI section: PR-title-driven semver, `APP_VERSION` env var, `/health.version`, auto-changelog from PR titles).

### Changes in v3.5

Architectural shift: **the marketing site is now the front door for signups, not a forwarder.** Previously signup happened on `app.<rootdomain>/signup`; now it happens on `<rootdomain>/signup` with embedded Clerk. The conversion event fires on a `<rootdomain>/welcome` thank-you page where ad-platform pixels can attribute conversions cleanly. The affiliate cookie is scoped cross-subdomain so it follows the visitor across marketing root, landing-page subdomains, the Clerk vanity subdomain, and the admin.

- **§3.2 marketing-site signup** — new section. Embedded Clerk `<SignUp />` on the marketing domain via `@clerk/astro` (or framework equivalent). Marketing site needs `CLERK_PUBLISHABLE_KEY` (public; never the secret). Clerk's `afterSignUpUrl` set to `<rootdomain>/welcome`.
- **§3.3 smooth org creation** — new section. Org auto-provisioned via `user.created` webhook (primary path); admin first-load auto-create kept as fallback for delayed/failed webhook delivery.
- **§3.9 Clerk webhooks** — added `user.created` to required handlers. Reads `unsafeMetadata.affiliateCode`, creates the org + Referral with attribution.
- **§6 affiliate flow rewritten** — cookie scoped to `.<rootdomain>` (cross-subdomain), `?r=` URL param is the entry point but no longer needs to be propagated through link rewriting. Signup happens on the marketing domain. Welcome page fires conversion pixels + CTA to admin.
- **§18 marketing site contract** — substantial rewrite. Marketing site now hosts `/signup` (embedded Clerk) and `/welcome` (conversion pixel host) in addition to its previous responsibilities. Link-rewriting script removed (no longer needed with cross-subdomain cookies). Cookie capture script updated to use `Domain=.<rootdomain>`.
- **§18.3 conversion tracking** — new section. Standard pixel placement on `/welcome` (Google Ads, Meta, LinkedIn, TikTok, Reddit), GTM-recommended implementation, consent-gating per §18.4.
- **§19 domain conventions** — added `accounts.<rootdomain>` (Clerk vanity subdomain) and the `<purpose>.<rootdomain>` pattern for marketing landing pages / microsites.
- **§20 checklist** — expanded with marketing-site signup, vanity subdomain, conversion-pixel placement, `afterSignUpUrl` configuration.

Re-numbering: §3 subsections shifted (`3.3` → `3.4`, etc.) to make room for the new `3.2` marketing-signup and `3.3` smooth-org-creation. The Clerk webhook section is now `3.9`; org lifecycle endpoints `3.10`. Cross-references updated throughout.

### Changes in v3.4

Refinement pass on v3.3 — corrects positions where v3.3 over-committed on behalf of leadership, simplifies the partner trial creation flow, and flags a scaling concern.

- **§7 commission cap language softened.** The v3.3 paragraph defending no-cap commissions ended with a specific commitment ("the lever to pull is the percentage 10% → 8% for new partnerships, not capping existing ones") that wasn't a leadership decision — that's a future design call. Now reads: "If commission economics ever need adjusting, the standard will be revisited."
- **§7 race-to-create reframed as the design, not a problem.** v3.3 added a `409 pending_trial_exists` conflict warning, a `force: true` override, and a consolidated invitation email. Per leadership: multiple partners creating trials for the same prospect is **intentional and supported** — the conversion-to-paid step is the adjudicator. Real-world cases exist where two partners legitimately work on two distinct instances for the same prospect (e.g., separate brands under one customer). The standard now keeps the rate limit (anti-fraud against scraped lists) and abandoned-org cleanup (data hygiene), but drops the conflict warning, the force flag, and the consolidated email. Each invitation stands on its own.
- **§7 dual-status during white-label clarified.** v3.3 added `PartnerSeat` at trial creation (single source of truth for "which clients does this partner have"). v3.4 makes the resulting redundancy explicit: during white-label, the partner is both `User` (role OWNER) and a `PartnerSeat` row in the same client org. Both serve different queries; both stay if the partner remains owner indefinitely; on promotion to client, the User row drops and PartnerSeat stays. Clean state transition.
- **§3.6 push-to-Clerk rate-limit flag.** v3.3 added opportunistic write of `role` + `accountType` to Clerk metadata on every `ClerkGuard` request when they don't match. The only-on-mismatch pattern keeps steady-state writes near zero, but post-deploy bursts and bulk role changes could hit Clerk's admin-API rate limits. Added explicit guidance: confirm limits against expected traffic, add a per-user 60s mismatch-write cache, log Clerk write failures at `warn` and continue serving (local DB stays canonical).

### Changes in v3.3

Incorporates Evident team's review of v3.2. Five blockers fixed (real internal inconsistencies any implementer would trip over), eight policy-resolution items closed, plus polish.

**Blockers fixed:**

- **§6 self-referral check** — was checking only the referrer's own email; with per-org codes that's incoherent (any teammate signing up with a different email would have passed). Now blocks if the new signup's email matches **any active member** of the referrer's org.
- **§3.8 Clerk webhooks** — entirely new section. Standardizes consumption of `user.deleted`, `organization.deleted`, `user.updated`, `session.created`, `organizationMembership.created/.deleted`. Signature verification via svix + `CLERK_WEBHOOK_SECRET`.
- **§15.2 deletion ordering** — explicit Clerk-delete-first → tombstone-second order, with retry/dead-letter on Clerk failure to avoid the half-deleted-then-un-tombstoned race.
- **§6 Referral FK asymmetry** — `referrerOrg` now `SetNull` (preserves history when referrer org closes), `referredOrg` stays `Cascade`. Closing the referrer org doesn't delete the row, just orphans it.
- **§11 whole-org notification semantics** — new `audience` field (`user | owners | admins | all_members`) with per-category default audience table. `userId: null + audience: 'admins'` means "fan out to all org admins."

**Policy items resolved:**

- **§3.4 `User.affiliateCode` long-term** — column dropped via follow-up migration after Foundry per-org cutover; new apps don't add it. Added to §2 audit checklist.
- **§4 `affiliate.manage` permission** — replaces the ad-hoc `affiliate.read + org.manage` combination from v3.2 for code regeneration.
- **§3.6 push to Clerk on login** — opportunistic write of `role` + `accountType` to Clerk metadata on every authenticated request when local DB doesn't match. Keeps client-side `useUser().publicMetadata` fresh after signups and recent role changes.
- **§14.5 + §14 partner-seat actor** — new `partnerSeatId` field on AuditLog and ActivityLog; actor resolution is `userId` OR `partnerSeatId`. Added `partner.payout_email_changed` to baseline audit events.
- **§12 receiver idempotency** — explicit customer-facing guidance: dedupe by `X-Epic-Delivery-Id`, at-least-once delivery, exactly-once is the customer's responsibility.
- **§17.5 health check criticality** — only `database` is critical (drives 503 + LB rotation); other sub-checks (Clerk, Resend, Dispatch) are informational. Prevents minor third-party hiccups from cascading into total outage.
- **§18.7 cookie consent** — picked a stance: standard library `@epic/cookie-consent`, affiliate cookie categorized as functional (Recital 30), fall back to "marketing" categorization if legal hasn't cleared the functional designation.
- **§21.12 lint tool name** — `@epic/eslint-plugin-tenancy` (TypeScript AST + Prisma schema introspection). Still 🧭 not built; manual review until then.

**New §17.6 sections:**

- Standard env var name table (prevents future divergence).
- Secret rotation policy (per-secret cadence; compromise-driven always wins).
- Uptime monitoring stance (`Better Uptime` default, alert thresholds).
- Email warmup window (2-week ramp before steady-state volume).
- Testing standards (auth + core domain integration tests required; no global Clerk mocking).
- Shared library packaging (`Epic-Design-Labs/shared-libraries` monorepo published to GitHub Packages).
- API versioning scope (customer-facing API gets `/v1/`; admin BFF routes are unversioned).
- Mobile / PWA explicitly out of scope.

**Tiny:**

- §14 `source` standardized as a closed value list (not free-text).
- §17.5 cron list adds stale-notification cleanup + abandoned-partner-trial-org cleanup.
- §22 free-tier API budget default (10k req/hr per key) for apps without billing.
- §2 schema audit deliverable specified (named diff document, ratified before v3.3 audit phase).

### Changes in v3.2

Incorporates feedback from the Dispatch team's review of v3. Most changes are clarifications or new subsections; a few concerns get explicit "this is the trade-off we're accepting" treatment because the underlying intent is confirmed:

- **§23 Throttle clarity** — explicit phase / ETA / "do not depend on Throttle for blocking design decisions yet" callout. Customer-to-org mapping locked in: one Throttle customer per org, Org→Throttle one-way provisioning, two sources of truth (Clerk identity, Throttle billing) instead of three.
- **§6 per-org affiliate trade-offs** — explicit table of trade-offs and mitigations: optional `Referral.sharedByUserId` snapshot for internal "shared by Sarah" attribution, cascade-delete on referrer org closure with warning in close dialog, multi-user sharing intentionally out of scope at MVP.
- **§7 race-to-create prospect-side guards** — partner trial creation rate-limited per §22, conflict warning (`409 pending_trial_exists`) when a pending Referral already exists for the prospect's email, abandoned-org cleanup after 14 days of no client login, consolidated invitation email when multiple partners force-create.
- **§7 white-label legal flag** — discount-vs-commission accounting treatment is pending legal review; specifics may evolve.
- **§3.4 query/security guidance** — common queries with non-unique `clerkUserId`, including the lint principle that `find*` on `User` without `orgId` is a tenancy bug.
- **§14.5 audit-log + tombstone legal note** — pseudonymous userId retention basis (Art. 6.1.c / 6.1.f), region-specific deletion may demand stricter handling.
- **§17.5 cron concurrency** — Postgres advisory lock or Redis SETNX requirement, idempotency on top of locking.
- **§9 / §1 email-templates clarification** — `@epic/email-templates` = shared components (header / footer / button / layout); per-app `src/emails/` = full templates that compose them.
- **§21.12 lint tooling realism** — `@epic/prisma-tenancy-lint` flagged as 🧭 not built; PR review is the enforcement mechanism today.
- **§2 Foundry audit checklist additions** — per-user → per-org affiliate migration, smooth-signup migration, Throttle migration, backup runbook, 7-year audit cold-storage, `SENTRY_RELEASE` → `APP_VERSION` rename.
- **§12 webhook truncation flag** — `X-Epic-Truncated: true` header (not body) so customers can branch before parsing JSON.
- **§14.5 routing rule** — events go to ActivityLog OR AuditLog, not both.
- **§18 cookie consent** — affiliate cookie pending legal sign-off; treat as consent-required for EU traffic until that lands.
- **§3.3 disposable email override** — per-domain mechanism, AuditLog entry, optional expiry. Specifies what was previously hand-wave.
- **§17.5 API path prefix** — clarified that all endpoint examples elsewhere in the doc are implicitly under `/api/v1/`.
- **§7 commission no-cap intentional** — explicit treatment of the trade-off and why we're accepting it (partner's long-term commitment to client success aligns better than capped models).
- **§21.10** — reconciled with §16: "demotion takes effect within seconds, not at session expiry" (was "immediately," which contradicted the documented 5-30s cache TTL).

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

The exceptions are **shared libraries** (e.g., `@epic/disposable-emails`, `@epic/sentry-config`, `@epic/email-templates`), which are fine — those are dev dependencies, not runtime services. (`@epic/email-templates` is the shared component layer — header / footer / button / layout. Full templates live per-app in `src/emails/` and compose those components — see §9.)

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
- [ ] Make `User.email`, `User.name`, `User.clerkUserId` nullable for tombstoning per §3.5
- [ ] Migrate per-user `User.affiliateCode` to per-org `Organization.affiliateCode` (schema + data migration + customer comms if old codes are in circulation)
- [ ] Verify `Referral.referrerOrgId` references local `Organization.id` (not Clerk ID, not user-keyed)
- [ ] Migrate `ActivityLog.createdBy` from email to userId (with backfill for departed users — orphan emails map to `null` or a `"deleted_user"` literal)
- [ ] Add `support.read/write`, `affiliate.read`, `apikeys.manage`, `activity.read`, `audit.read`, `export.run`, `notifications.manage`, `webhooks.manage` as named permissions per §4 baseline
- [ ] Add `permission_changed` 401 emission on session role mismatch per §16
- [ ] Add per-method notification preferences (bell/toast/email per category) + `deliveryClass` field per §11
- [ ] Migrate transactional emails to `react-email` (current PO send/follow-up are inline HTML strings) per §9
- [ ] Adopt `@epic/sentry-config` with PII redaction per §17
- [ ] Rename `SENTRY_RELEASE` env var to `APP_VERSION` per §17
- [ ] Implement smooth-signup auto-create-org flow per §3.3 (currently relies on Clerk's hosted org-selection screen)
- [ ] **Move signup form from `app.foundryims.com/signup` to `foundryims.com/signup`** per §3.2 — embed Clerk `<SignUp />` on the Astro marketing site via `@clerk/astro`
- [ ] Build `foundryims.com/welcome` thank-you page with Google Ads + Meta + LinkedIn conversion pixels per §18.3
- [ ] Configure Clerk vanity subdomain `accounts.foundryims.com` per §19
- [ ] Set Clerk `afterSignUpUrl` to `https://foundryims.com/welcome`
- [ ] Update marketing-site cookie capture script to use `Domain=.foundryims.com` per §18.2 (currently host-only)
- [ ] Remove the URL-bridge link rewriting from `astro-foundryims/src/layouts/Layout.astro` (no longer needed with cross-subdomain cookies)
- [ ] Add `user.created` webhook handler to API per §3.9 (org auto-provision + affiliate attribution) — replaces the post-auth `AffiliateAttribution` callback as the primary attribution path
- [ ] Migrate from current billing (whatever is in place) to Throttle once Throttle ships per §23
- [ ] Build backup runbook documenting 30-day expiry + deletion-rerun-on-restore per §15.2
- [ ] Set up 7-year audit-log cold-storage infrastructure per §14.5
- [ ] Convert Stackbe → fully-on-Clerk for any features still routing through Stackbe (feature requests was the last remaining one as of v3.1)
- [ ] PR-review enforcement of tenancy-scoped queries until `@epic/prisma-tenancy-lint` ships per §21.12
- [ ] **Schema audit deliverable:** produce a diff document comparing Foundry's `prisma/schema.prisma` against the v3.5 baseline schemas in §3.5 (Org/User), §6 (Referral with `referrerOrgId` SetNull + `sharedByUserId`), §7 (PartnerSeat / PartnerSeatAssignment), §11 (Notification with `audience` + `deliveryClass`, NotificationPreference per-method), §12 (Webhook + WebhookDelivery with secret rotation fields), §13 (ApiKey with `expiresAt` + `scopes`), §14 (ActivityLog with `partnerSeatId` + standard `source` values), §14.5 (AuditLog with `partnerSeatId`), §15.2 (DataDeletionRequest, DataDeletionAudit). One doc, one PR, ratified before v3.3 enters audit phase
- [ ] Drop `User.affiliateCode` column once per-org migration completes (separate migration after the data move)

---

## 3. Auth & Organizations

### 3.1 Clerk dashboard config

Required settings in every Clerk project:

- **Sign-up enabled** — open self-serve signup. (Restrict via Clerk's allowlist if you need invite-only later.)
- **Magic link, Google, Apple** sign-in methods enabled.
- **Organizations enabled.**
- **`force_organization_selection: false`** — let users land in the app immediately after signup; we auto-create an org in code (see §3.3).
- **`automatic_organization_creation: false`** — Clerk doesn't auto-create either; we control the flow so we can also write the local Org row + send the right magic link.
- **Organization name template** — `"{{user.first_name}}'s Organization"` (fallback `"My Organization"`).
- **2FA / passkeys enabled** at the Clerk level for every app.

### 3.2 Signup happens on the marketing site, not the admin

The signup form lives on the **marketing site** (e.g., `<rootdomain>/signup`), not the admin app. Reasons:

- **Conversion-pixel attribution.** Google Ads / Meta / LinkedIn / TikTok pixels live on the marketing domain. Conversion events (sign-up complete) need to fire on the same domain that loaded the pixels — putting signup on the admin domain breaks attribution for paid acquisition.
- **Front-door framing.** The marketing site is the prospect-facing surface. Signing up should happen there, not require a hop to a different domain (`app.<rootdomain>`) before form submission.
- **Affiliate cookie continuity.** With the affiliate cookie scoped to `.<rootdomain>` (see §6), the cookie is available to the signup form on `<rootdomain>/signup` directly.

The technical pattern is **embedded Clerk** on the marketing site:

- Marketing site uses `@clerk/astro` (or framework equivalent) and renders `<SignUp />` inline at `<rootdomain>/signup`.
- The marketing site needs `CLERK_PUBLISHABLE_KEY` (public; never the secret key — that stays server-side on the API).
- After successful signup, Clerk redirects the user to `<rootdomain>/welcome` (the conversion thank-you page) instead of straight to the dashboard. Configure this via Clerk's `afterSignUpUrl`.

### 3.3 Smooth org creation (no "create your org" screen)

After Clerk creates the user, an org needs to exist for them to use the app. We avoid showing a "Create your organization" screen by combining `force_organization_selection: false` with two automated paths:

- **Primary: `user.created` webhook.** When Clerk creates the user, our API receives the `user.created` webhook (see §3.9), reads any `unsafeMetadata.affiliateCode` set during signup, auto-creates an org named `"<First Name>'s Workspace"`, and writes the `Referral` row if an affiliate code was present. Org exists before the user lands anywhere.
- **Fallback: admin first-load auto-create.** If the webhook is delayed or fails (Clerk webhook delivery is at-least-once, not strictly real-time), the admin's first-load logic checks for zero memberships and auto-creates the org as a backup. Idempotent — once the webhook lands, this path becomes a no-op.

Either way, by the time the user clicks "Continue to dashboard" from the welcome page, the org exists. No extra screen, no waiting state.

Org rename is available later in Settings.

### 3.4 Disposable email blocking

Every app blocks signups from known disposable email providers using the **`disposable-email-domains`** npm package (actively maintained, ~3.7k domains). When a blocked domain is detected:

- Reject the signup with a clear error message: **"We don't allow signups with temporary or disposable email addresses. Please use your real email."**
- Log the attempt to Sentry at `info` severity (per §17 — the disposable email block is a signal, not an error).
- **CSR override is per-domain**, not per-email — granting `mailinator.com` access opens it for everyone using that domain. UI lives in an admin-only Settings page (`/admin/disposable-overrides` or equivalent). Each override row carries: `domain`, `addedBy` (User.id), `reason`, `createdAt`, optional `expiresAt`. Override creation/removal writes an `auth.disposable_override` entry to the AuditLog (§14.5). This makes overrides explicit, time-bound by default, and auditable — not informal Slack favors.

### 3.5 Local schema mirror

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
  // affiliateCode — REMOVED in v3.3. Orgs are the affiliate unit (see §6). Foundry-only legacy column to be dropped once migration completes; new apps don't add it.
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
- **Query implications** of non-unique `clerkUserId`:
  - `findUnique({ where: { clerkUserId } })` no longer works — use `findFirst({ where: { clerkUserId, orgId } })` for the active-org row, or `findMany({ where: { clerkUserId } })` for cross-org operations.
  - **Permission checks must scope by `(clerkUserId, orgId)`** — not `clerkUserId` alone. A query that filters only by `clerkUserId` will return rows from other orgs and leak permissions across tenants.
  - **Cross-org operations** (GDPR sweep, "log this user out everywhere", "list this person's memberships") use `findMany({ where: { clerkUserId } })` and iterate.
  - **Lint principle (per §21.12):** any `find*` on `User` that doesn't include `orgId` in the where-clause is treated as a tenancy bug unless explicitly annotated as a cross-org operation.
- `User.email`, `User.name`, `User.clerkUserId` are **nullable** to support GDPR tombstoning (§15). PII is nulled on deletion; the row itself survives so foreign keys keep working. **Migration note:** apps that started with `email String` (required) — including Foundry — need an audited `ALTER TABLE` to make these columns nullable. The `@@unique([orgId, email])` constraint continues to work; Postgres allows multiple NULLs in unique constraints by default.
- The shift from per-user to per-org affiliate codes is locked in for v3. The `User.affiliateCode` field is reserved for migration only — present in the schema only so apps mid-migration can read old values during cutover. **After Foundry's per-user → per-org affiliate migration completes, the `User.affiliateCode` column gets dropped via a follow-up migration** (added to the §2 audit checklist as a separate item from the migration itself, so it doesn't get forgotten). New apps that bootstrap from this standard skip this column entirely — `affiliateCode` lives on `Organization`, never on `User`.

### 3.6 ClerkGuard + permission re-validation

Every API request goes through a guard that:

1. Accepts API keys (`<app>_*` prefix) without a Clerk session — see §13.
2. Validates the Clerk JWT.
3. Resolves the active org via `org_id` claim → `X-Clerk-Org-Id` header (fallback) → user's only org.
4. **Re-validates the user's role against the local DB on every request** — see §16. Cache TTL of 5–30s is acceptable for most apps; security-sensitive flows can override to <1s.
5. Lazily creates the local User row on first request if missing.

Reference: `app-foundry-ims-api/src/auth/guards/clerk.guard.ts`.

### 3.7 Source of truth on Clerk-mirrored fields

Some fields exist in both Clerk and the local DB (`accountType`, `role`, org membership). On divergence:

- **Clerk is canonical for auth + org membership.** Local DB is a read cache.
- **Local DB is canonical for `role` and `accountType`.** Clerk metadata is a downstream sync target.

**Sync direction is one-way and write-on-change.** When local `role` or `accountType` changes (admin promotes a user, partner application is approved), we write the new value to Clerk metadata in the same transaction. We do **not** poll or sync from Clerk back to local. Treat Clerk metadata as **eventually consistent / advisory** — the frontend may read it for UI hints (`useUser().publicMetadata`), but the API always re-validates against the local DB. Direct edits to Clerk metadata via the dashboard during incidents will get overwritten on the next local update for that user.

**Plus: push on login.** On every successful authenticated request that hits `ClerkGuard`, after the local DB lookup, opportunistically write the current `role` + `accountType` to Clerk metadata if they don't match. This keeps client-side `useUser().publicMetadata` reads accurate. Without it, a fresh signup or a recently-promoted user would have stale Clerk metadata on the client until the next local-DB-triggered write — and the client SDK would render UI off the wrong values. The write is fire-and-forget (no blocking), and only fires on mismatch (no extra write traffic in the steady state — most requests skip it).

> **Rate-limit caution:** the only-on-mismatch pattern keeps steady-state write traffic to Clerk near zero, but the burst pattern after a deploy that touches role logic, after a bulk role change, or during a re-auth storm could hit Clerk's per-instance write quotas. Each app should:
> - Confirm Clerk's current admin-API rate limits against the app's expected post-login traffic shape (Clerk publishes these per plan).
> - Add a per-`(clerkUserId, key)` short-TTL cache (e.g., 60s) on the mismatch-write path so a flurry of requests from a single client doesn't fan out to repeated Clerk writes within the same window.
> - Log Clerk write failures at `warn` and continue serving the request — the local DB stays canonical, and the next mismatch-check will retry.

If a divergence is detected during permission re-validation, log to Sentry at `warn` and reconcile by reading the local DB and writing to Clerk.

### 3.8 Org switcher (header dropdown)

Universal pattern: a dropdown in the top-right header showing the user's Clerk memberships, with switch / create / leave actions.

Behavior contract:

- **List orgs** — read from Clerk's `useOrganizationList()` directly. Don't hit the local API for this.
- **Switch** — call Clerk's `setActive({ organization })`, then **immediately** update `localStorage.clerkOrgId` so the next API request includes the new `X-Clerk-Org-Id` header. Then reload (`window.location.href = "/"`) to reset all React Query / server state.
- **Create** — Calls `POST /orgs` (any authenticated user can create their own org).
- **Leave** — any member. Calls `POST /orgs/me/leave` (sole-owner guard enforced server-side).
- **Active indicator** — checkmark on the currently active membership.
- **Cross-tab staleness** — known limitation: a second tab open to the dashboard keeps stale React Query cache after switching in tab 1. The wrong-org-data window is brief; document this in the help center.

The `X-Clerk-Org-Id` header pattern requires every manual `fetch()` site to use a shared `buildAuthHeaders()` helper instead of constructing headers inline. See `app-foundry-ims-admin/src/lib/api/core.ts`.

### 3.9 Clerk webhook handling

Apps consume **Clerk webhooks** to keep local state aligned when Clerk-side changes happen (admin deletes a user from the dashboard, a user updates their email, etc.). Without this, local DB drifts from Clerk and bugs surface as "ghost users" or "stale email."

**Required env var:** `CLERK_WEBHOOK_SECRET` (Clerk-issued, per-app, per-environment).

**Signature verification:** every Clerk webhook is signed via [svix](https://docs.svix.com/receiving/verifying-payloads/how). Use the `svix` npm package to verify before processing — never trust the body without verification.

```ts
import { Webhook } from 'svix';

const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);
const evt = wh.verify(rawBody, headers) as ClerkWebhookEvent;  // throws on invalid signature
```

**Required event handlers** every app implements:

| Event | What to do |
|---|---|
| `user.created` | **Auto-provision the user's first org.** Read `unsafeMetadata.affiliateCode` if present (set by the marketing-site signup form per §6), create an `Organization` named `"<First Name>'s Workspace"`, create the local `User` row with `role: OWNER`, add Clerk org membership, and write a `Referral` row if an affiliate code was passed. This is the primary path for org provisioning; the admin first-load fallback (§3.3) handles delayed/failed webhook delivery. |
| `user.deleted` | Tombstone the local `User` row(s) for this `clerkUserId` so they can't be recreated on next login attempt. Tombstone in the §15.2 sense — null PII, set `deletedAt`, keep the row for FK integrity. |
| `organization.deleted` | Cascade-close the local `Organization` row matching `clerkOrgId`. Same flow as `DELETE /orgs/me`. |
| `user.updated` | Propagate email/name changes to local `User` rows (across all orgs the user belongs to). Critical when the user changes their email in their account-tab UI. |
| `session.created` | Drives the `auth.new_device_login` audit event (§14.5) when the IP / user-agent doesn't match the user's recent sessions. |
| `organizationMembership.created` / `.deleted` | Sync local `User` rows with Clerk org membership (lazy-create on `created`, tombstone or remove on `deleted`). Defends against memberships changed via Clerk dashboard. |

**Endpoint pattern:** `POST /webhooks/clerk` (per-app, public — secured by signature verification, not auth). Handler must be idempotent — Clerk retries on non-2xx, and webhook at-least-once delivery means a user might see the same `user.deleted` event twice.

**Failure handling:** if the webhook can't process (database down, Clerk API rate limit during cleanup), return 5xx so Clerk retries. Log to Sentry at `error`.

### 3.10 Org lifecycle endpoints

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
| `affiliate.manage` | Regenerate the org's affiliate code |
| `apikeys.manage` | Create/revoke API keys |
| `activity.read` | View activity log |
| `audit.read` | View security audit log |
| `export.run` | Export org data |
| `notifications.manage` | Configure own notification preferences |
| `webhooks.manage` | Configure outbound webhooks |

Default role-to-permission mapping (apps may extend, must not contract):

- **OWNER** — all permissions including `org.close` and `affiliate.manage`
- **ADMIN** — all except `org.close` (includes `affiliate.manage`)
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

### Trade-offs of per-org codes (and how to mitigate)

The shift from per-user to per-org affiliate codes is locked in. It's the right call for B2B context but creates real trade-offs worth naming explicitly:

| Concern | Resolution |
|---|---|
| **Sarah at Acme tweets her org's link → credit goes to Acme, not Sarah personally** | True. Per-org is the right primitive for B2B (commissions show up on Acme's invoice, not as 1099 income to Sarah). For solo founders / individual users this is a non-issue (they *are* their org). For multi-user orgs, internal kickback is Acme's policy to set, not ours to enforce. |
| **Multi-user "who shared the link?" attribution** | Optional snapshot field: `Referral.sharedByUserId String?` records which user's session captured the `?r=` click. UI can say "shared by Sarah" while commission still accrues to Acme. Anonymized to `null` on user deletion (PII tombstoning). Apps may surface this in their Settings → Affiliate UI; mandatory if the app's customers ask for internal attribution. |
| **Partner-as-individual is awkward** | Solo partners are already an org of one — they get the dashboard and commissions on the org. The schema doesn't change; their "personal" referrals and "partner" referrals both write to the same `Organization.affiliateCode`. |
| **Abandoned / closed referrer org** | `Referral.referrerOrgId` is a foreign key with `onDelete: Cascade`. When a referrer org closes, their open Referral rows cascade-delete and commissions stop. This is intentional — paying out to a closed org is operationally messy (no payout method, no contact). Closing an org with active referral commissions surfaces a warning in the close dialog: "You have $X in open referral commissions; closing forfeits future payouts on N referred orgs." |
| **5 people in an org share one link, only 1 actually distributes it** | Out of scope for v1. The org is the unit of credit; the org decides internally. If apps need granular attribution they can opt into the `sharedByUserId` snapshot above. |

### Standardized affiliate rules

These rules apply to every app:

1. **One affiliate code per organization.** Any user with `affiliate.read` (which all roles have) can see their org's code, share it, and view commission stats. Multiple users in the same org share one code; commissions accrue to the org, not to individual users. (See trade-offs above.)

2. **Last-touch attribution wins.** If a prospect clicks two different affiliate links in the 30-day cookie window, the most recent code overwrites the earlier one. This is a deliberate trade-off: simpler than first-touch, aligned with industry standard, but unfair to top-of-funnel educators who may lose credit to bottom-of-funnel coupon sites. Pricing the affiliate tier at 10% with no cap is intended to keep both kinds of partners engaged.

3. **Self-referral is rejected by membership match against the referrer's org.** If the signing-up user's email matches **any active member** of the referrer's org, attribution is blocked. With per-org codes (see rule 1), the entire org shares the link — so a teammate signing up via their own org's link IS the org self-referring. Returns `{ attributed: false, reason: "self_referral" }`. (Earlier drafts checked only the referrer's own email — that was incoherent with the per-org primitive; a teammate signup with a different email would have passed and the org would have credited itself.)

4. **Code regeneration: snapshots are immutable.** A user with `affiliate.manage` can regenerate their org's `affiliateCode`. Existing `Referral` rows keep the old code (we snapshot `affiliateCode` on the Referral). The old code stops working for *new* attributions.

### Schema

```prisma
model Referral {
  id              String    @id @default(uuid())
  referrerOrgId   String?                             // nullable — set NULL when referrer org closes (preserves referral history for the referred org)
  referredOrgId   String    @unique                   // local Org id; one credit per signup
  referralType    String    @default("affiliate")     // "affiliate" | "direct"
  affiliateCode   String                              // snapshot at attribution time
  sharedByUserId  String?                             // optional: which user shared the link (for internal attribution UI). Anonymized to null on user delete.

  status          String    @default("pending")       // pending | active | converted | expired
  rewardStatus    String    @default("pending")       // pending | paid | ineligible

  createdAt       DateTime  @default(now())
  convertedAt     DateTime?

  referrerOrg     Organization? @relation("ReferrerOrg", fields: [referrerOrgId], references: [id], onDelete: SetNull)
  referredOrg     Organization  @relation("ReferredOrg", fields: [referredOrgId], references: [id], onDelete: Cascade)

  @@index([referrerOrgId])
  @@index([affiliateCode])
}
```

**Asymmetric FK behavior is intentional:**

- **`referrerOrg` uses `onDelete: SetNull`** — when a referrer org closes, the row survives with `referrerOrgId: null`. Preserves the referred org's history (we still know it was originally an affiliate signup) and keeps commission accounting trails intact for any commissions that already accrued. The status is set to `expired` by the close handler so no new commissions accrue. The `affiliateCode` snapshot field still tells you who originated the relationship even though the org is gone.
- **`referredOrg` uses `onDelete: Cascade`** — when the referred org closes, delete the Referral row. The org is gone; there's nothing left to track and no future events to attribute.

The §3.10 close-org dialog still surfaces the open-commission warning ("You have $X in open referral commissions; closing forfeits future payouts on N referred orgs") — that's a UX nicety, not a database constraint. The schema permits the close; the dialog informs the user before they confirm.

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

`https://<rootdomain>/?r=ABC12345`

Cookie: `affiliateR`, **scoped to the entire root domain** (`Domain=.<rootdomain>;path=/;samesite=lax`, 30-day TTL). Cross-subdomain so the cookie follows the prospect across:
- `<rootdomain>` (marketing site main pages)
- `landing.<rootdomain>`, `pricing.<rootdomain>`, etc. (marketing landing pages and microsites)
- `accounts.<rootdomain>` (Clerk vanity subdomain, where Clerk hosts auth flows)
- `app.<rootdomain>` (admin)
- `api.<rootdomain>` (server-side cookie reads if needed)

The `?r=` URL param is the entry point that sets the cookie; once set, the cookie carries the value for 30 days.

### Flow

1. Existing org member visits Settings → Affiliate, clicks Copy.
2. They share the URL `https://<rootdomain>/?r=ABC12345`.
3. Prospect lands on **any** marketing-site page (root, landing page subdomain, pricing, etc.) → script reads `?r=`, validates against `/^[A-Za-z0-9]{8}$/`, stores cookie scoped to `.<rootdomain>` (overwriting any previous).
4. Prospect browses, eventually clicks a **"Sign Up Free"** / **"Get Started"** CTA → routes to `<rootdomain>/signup` (on the marketing domain — not the admin).
5. Marketing-site signup page renders Clerk's `<SignUp />` component (embedded via `@clerk/astro` or framework equivalent). On form submit, the page reads the `affiliateR` cookie and passes it to Clerk as `unsafeMetadata.affiliateCode`. Clerk performs the disposable email check.
6. Clerk creates the user → fires `user.created` webhook → API handler reads `unsafeMetadata.affiliateCode`, auto-creates an org, creates a `Referral` row attributing the new org to the referrer, with self-referral check (per rule 3 above).
7. Clerk redirects the user to **`<rootdomain>/welcome`** — the conversion thank-you page on the marketing site (configured via Clerk's `afterSignUpUrl`).
8. Welcome page fires conversion pixels (Google Ads, Meta Pixel, LinkedIn, etc. — see §18.X), shows a brief celebration, optionally clears the affiliate cookie, and presents a **"Continue to Dashboard"** CTA → `app.<rootdomain>`.
9. User clicks Continue → lands in admin (Clerk session is already live across subdomains) → org already exists from step 6, immediate dashboard access. (If the webhook in step 6 was delayed, the admin's first-load fallback per §3.3 creates the org instead — admin still loads cleanly.)
10. Referrer sees the new sign-up under Settings → Affiliate.

The admin's `/signup` route still exists as a **fallback entry point** for visitors who somehow arrive there directly (deep-linked, return visitor, etc.) — but it's no longer the front door.

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

**Dual-status during white-label — intentional redundancy:** During a Path A (white-label) trial and indefinitely afterward if the partner stays as owner, the partner has **two** representations in the client org simultaneously:

- A row in the client org's `User` table with `role: OWNER` (so they can act as owner — invite team members, change settings, etc.).
- A row in `PartnerSeat` linking the client org to the partner's own org (so the partner dashboard can list this client and the audit log can attribute partner-seat actions).

This is mildly redundant but solves more problems than it creates:

- The partner dashboard query is one straight `SELECT FROM PartnerSeat WHERE partnerOrgId = ?` regardless of arrangement (white-label vs. promoted).
- Permission checks always use the client-org `User` row (the OWNER role grants normal owner permissions).
- Activity / audit logs can choose which actor to attribute based on context (`createdBy: User.id` for logged-in admin actions; `partnerSeatId` for partner-seat-attributed actions).
- On promotion to client (Path B), the partner's `User` row is deleted; the `PartnerSeat` row stays. Client becomes OWNER, partner retains seat. Single, clean state transition.

If the partner stays as owner indefinitely (white-label long-term), the redundancy is permanent. That's fine — both rows serve different purposes and queries are unambiguous.

**Audit trail during partner-controlled trial:** All `ActivityLog` entries during this period must be tagged `source: "partner_seat"` to distinguish partner-attributed actions from client-attributed actions. This creates a clear audit trail for any disputes about what was done before the client took ownership.

### Two ownership paths after conversion

When a partner-created trial converts to paid, there are **two valid ownership models**:

#### Path A: White-label (partner stays owner)

The partner pays the subscription fee on the client's behalf as part of an all-inclusive service offering. The partner remains OWNER of the org indefinitely.

- Partner stays as `OWNER` of the client org.
- Partner's payment method is on file in Throttle.
- **Partner receives a 10% white-label discount on the bill** — netted at source, not paid out as commission. Throttle invoices the partner at 90% of list price for white-label-mode subscriptions.
- The client may not have visibility into the bill (this is a partner choice).

> **Legal status:** Whether this is a "discount" (treated as net revenue) or a "commission" (gross revenue minus a 1099 expense) for accounting and tax purposes is **pending legal review**. The economics match either way; the line items on financial statements and 1099 forms differ. Path A specifics may evolve once Throttle integration ships and finance/legal sign off. Implementations should treat the discount-at-source mechanism as the default direction but expect refinement.

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

### Race-to-create is the design, not a problem to solve

Multiple partners creating trials for the same prospect email is **intentional and supported**. Each partner gets their own `Referral` row in `pending` status against their own pre-created org. Whichever partner converts the prospect to paid wins the credit on that org. If a real-world prospect chooses to bring on two partners with two valid instances (e.g., a Foundry org for IMS work + a separate Foundry org for a different brand they own), both partners legitimately get credit on the orgs they each set up.

The conversion-to-paid step is the ultimate adjudicator — credit follows the org that the prospect actually chose to convert. We don't need a coordination layer to prevent partners from creating trials in parallel; the market sorts it out at conversion.

What we still standardize:

1. **Per-partner rate limit on trial creation** — `10 / hour` per partner-org by default (per §22). Higher tiers configurable. Anti-fraud against scraped email lists; not a coordination mechanism between partners. A bad actor creating thousands of trials gets stopped here.

2. **Abandoned-org cleanup.** If an invited user never signs in after 14 days, the partner-trial org auto-deletes. Releases the email-as-org-member slot for any future invitation, removes orphaned data. Cron sweeps for `Organization.createdAt > 14 days ago AND only one User row (the partner) AND no client login`.

3. **Each invitation email stands on its own.** Prospect receiving multiple invitations from multiple partners is acceptable — it accurately reflects what's happening (multiple partners are trying to set them up). The prospect picks the one they want by clicking through; the others get cleaned up after 14 days via the abandoned-org sweep.

We deliberately do **not** add: pending-trial conflict warnings, consolidated invitation emails, or hard rejects. Those would constrain partners' selling processes without serving a real need.

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

- **10% recurring commission, no cap, no time limit** for both `affiliate` and `direct` referrals.
- Commissions for billing events in month N are paid in month N+1. Example: a client pays their April invoice on April 15. The partner's $X commission accrues to the May commission statement, paid early May.
- This gives clean monthly reconciliation, time for refunds/chargebacks to settle, and predictable payout timing.

> **On the absence of a cap or sunset:** This is a deliberate choice. A 24/36-month cap would lower lifetime commission cost but adds complexity (cap tracking, sunset notifications, partner disputes near expiry) and weakens the partner's long-term commitment to the client relationship. We accept that a partner who originated a client 5 years ago still earns 10% on that client's bill — this aligns the partner's incentive with the client's long-term success. If commission economics ever need adjusting, the standard will be revisited.

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

**All transactional emails use [`react-email`](https://react.email).** Templates are JSX components in `src/emails/` per-app, version-controlled, compiled to HTML at build time. Templates **compose shared components** from `@epic/email-templates` (header, footer, button, signature, layout primitives) — those guarantee visual consistency across apps without locking apps into identical templates. App-specific content (subject lines, body copy, variable substitution) stays in the app's `src/emails/` directory.

This gives apps:

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
  userId          String?  // null = whole-org fanout, see audience rules below
  audience        String   @default("user")  // "user" | "owners" | "admins" | "all_members"
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

### Audience resolution

`userId: null` means the notification fans out to multiple recipients. The `audience` field controls who:

| Audience value | Recipients |
|---|---|
| `user` (default; requires non-null `userId`) | Just the named user |
| `owners` | All `OWNER` users in the org |
| `admins` | All `OWNER` + `ADMIN` users in the org |
| `all_members` | Everyone in the org |

**Per-category target convention** — every standard category maps to a default audience; apps may override on a per-event basis if needed:

| Category | Default audience |
|---|---|
| `org.member_invited` | `admins` |
| `org.member_joined` | `admins` |
| `org.member_removed` | `owners` (security-sensitive) |
| `org.role_changed` | `user` (the person whose role changed) + `owners` |
| `support.ticket_replied` | `user` (the user who filed the ticket) |
| `affiliate.signup` | `all_members` (all org members can see) |
| `partner.client_created` | `admins` of the partner org |
| `partner.commission_earned` | `admins` of the partner org |
| `auth.login_new_device` | `user` |
| `auth.password_changed` | `user` |
| `apikey.created` | `owners` (security-sensitive) |
| `billing.payment_failed` | `owners` (transactional, can't be opted out) |
| `privacy.deletion_confirmed` | `user` |

When `audience` ≠ `user`, the sender writes one `Notification` row per intended recipient (so the bell badge is per-user, mark-read is per-user, etc.). The `audience` field is preserved on the row for analytics/audit but the fanout happens at create time, not at read time.

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

**Payload size cap: 1 MB.** Larger events are truncated; customers can fetch the full resource via API. Truncation is signaled in the **header** (not the body) so customers can branch before parsing JSON:

```
X-Epic-Truncated: true
X-Epic-Resource-Url: https://api.<app>/api/v1/<resource>/<id>
```

The header is the canonical signal. The body remains valid JSON of the truncated event; downstream parsers don't need to handle a non-standard payload shape.

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

### Ordering guarantee + receiver idempotency

**Webhook ordering is best-effort.** Retries with exponential backoff mean events can arrive out of order. **Treat the resource's own state as authoritative** — webhooks are notifications, not source of truth.

**At-least-once delivery, not exactly-once.** Customers MUST dedupe by `X-Epic-Delivery-Id` to handle retries safely. The customer-facing developer docs include this guidance verbatim:

> Receivers should dedupe by `X-Epic-Delivery-Id` to handle retries safely. We guarantee at-least-once delivery; treating each delivery as idempotent on your side is your responsibility.

Document this clearly in every customer-facing webhook integration page.

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
  id            String   @id @default(uuid())
  orgId         String
  entityType    String
  entityId      String
  action        String
  summary       String
  source        String   // see standard values below
  sourceId      String?
  metadata      Json?
  createdBy     String?  // local User.id (NOT email — see §15) — null when partnerSeatId is set
  partnerSeatId String?  // set when actor is a partner acting via a partner seat (§7); see §14.5
  createdAt     DateTime @default(now())

  @@index([orgId, entityType, entityId])
  @@index([orgId, createdAt])
  @@index([partnerSeatId, createdAt])
}
```

**Standard `source` values** (apps may extend; document any additions in app-level changelog):

- `manual` — user-driven action via the admin UI
- `api` — action via API key
- `webhook` — inbound webhook from an external service (BC, ShipStation, Throttle, etc.)
- `import` — CSV/sheet import job
- `cron` — scheduled job
- `partner_seat` — action originated from a `PartnerSeat` actor (paired with non-null `partnerSeatId`)
- `migration` — data migration / bulk fix script

Treat `source` as a closed enum for queryability; if an app needs a new source, add it to the standard list, not as a one-off string.

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

**Routing rule:** an event goes to **either** ActivityLog (§14) **or** AuditLog (§14.5), not both. The baseline events listed below (`auth.login`, `user.role_changed`, `apikey.created`, etc.) write only to AuditLog. Business events (`product.created`, `order.shipped`) write only to ActivityLog. If a category genuinely needs to surface in both UIs, the recommendation is to query both at read time, not double-write.

### Schema

```prisma
model AuditLog {
  id            String   @id @default(uuid())
  orgId         String
  userId        String?  // local User.id — actor of the action (null when acting via partnerSeatId)
  partnerSeatId String?  // set when the actor is a partner acting via a partner seat (§7)
  action        String   // see baseline list below
  resource      String?  // affected entity (e.g., "user:abc-123", "apikey:xyz")
  ipAddress     String?
  userAgent     String?
  metadata      Json?
  createdAt     DateTime @default(now())

  @@index([orgId, createdAt])
  @@index([userId, createdAt])
  @@index([partnerSeatId, createdAt])
  @@index([action, createdAt])
}
```

**Actor resolution:** exactly one of `userId` or `partnerSeatId` is set. When `partnerSeatId` is non-null, the audit UI displays "via [Partner Org Name] partner seat" — the action originated from a partner acting on a client org via a `PartnerSeat`, and there's no local `User` row for the partner in the client org. Same pattern applies to `ActivityLog.source = "partner_seat"` + a `partnerSeatId String?` field there. (`ActivityLog.source` is otherwise a string; valid values are listed in §14.)

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
- `partner.payout_email_changed` — security event (PayPal email controls real money flow)
- `auth.disposable_override_added` — CSR allowed a disposable email domain (§3.4)
- `auth.disposable_override_removed`

### App-specific extensions

Each app evaluates additional security events relevant to its domain. Examples:

- Dispatch: `ticket.reassigned_to_partner`, `ticket.priority_escalated_by_admin`
- Rally: `attribution_model.changed`, `tracking_pixel.regenerated`
- Foundry: `inventory.bulk_import`, `bigcommerce_credentials.rotated`

### Retention

**7 years.** Audit log retention is a compliance requirement (SOC 2, GDPR, regional data laws). Cold-storage past 1 year is acceptable.

> **Legal note on tombstoned userIds in audit retention:** After a user invokes right-to-deletion (§15.2), `AuditLog.userId` rows referencing that user are preserved — the user row is tombstoned (PII nulled) but `User.id` survives so audit trails keep referring to "user `abc-123` did X." This is **pseudonymous data** under GDPR — defensible under the "compliance with legal obligations" and "legitimate interests" lawful bases (Art. 6.1.c / 6.1.f) for security-records retention. **Region-specific deletion requests may demand stricter handling** (full anonymization, or a maximum retention shorter than 7 years for non-financial events). Apps with EU/UK/CA customers should run this past legal before relying on the 7-year default. Document the basis in the app's compliance runbook.

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

**Post-close availability:** When an org is closed (§3.10), the final export remains available for **30 days** before final purge. Closure dialog warns the OWNER and offers immediate export before deletion.

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

**Order of operations is critical** to avoid a race where the local row is tombstoned but the Clerk delete fails — leaving the user able to log in via Clerk and silently un-tombstone themselves on next request:

1. **Clerk delete first** — call `clerk.users.deleteUser(clerkUserId)` and **wait for confirmation**. On failure (network, rate limit, Clerk API down), the deletion job exits without touching local state and re-queues with backoff. **Tombstoning never starts until Clerk confirms the delete.**
2. **Local tombstone second** — only after Clerk confirms, perform the local mutations below in a single transaction.
3. **Failed Clerk deletes** — after 8 retry attempts (1m, 5m, 15m, 1h, 6h, 12h, 24h, 24h), the deletion request moves to a dead-letter queue. The user stays alive in Clerk; an alert fires to ops; manual intervention required. Better to fail loud than half-delete.

**Local mutations (only after Clerk confirms):**

- `User.email` → `null`
- `User.name` → `null`
- `User.clerkUserId` → `null`
- `User.deletedAt` → set to current timestamp (marks as tombstone)
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

**Critical vs informational sub-checks:**

- **`database` is the only critical check.** If the DB is unreachable, the app cannot serve requests — return 503 so the load balancer takes the instance out of rotation.
- **All other sub-checks are informational** (`clerk`, `resend`, `dispatch`, `throttle`, etc.). The app continues serving traffic even if Resend is down — login still works, dashboards still load, only the email-send path degrades. The status string for these can be `ok | degraded | down`, but `503` is reserved for database failure.

This prevents minor third-party hiccups from cascading into total app outage via overzealous LB routing.

### Cron conventions

Apps using a cron library (node-cron, BullMQ, etc.) follow this pattern:

- Cron jobs defined in a single file (e.g., `src/cron/index.ts`)
- Each job has: `name`, `schedule` (cron expression), `handler`, optional `timezone`
- All jobs log to Sentry with tags: `cronJob: "<name>"`, `started_at`, `duration_ms`, outcome
- Failures captured as exceptions; recovered failures logged at `warn`
- Standard timezone handling: jobs default to UTC unless explicitly overridden

**Concurrency / re-entrancy: every job must be safe to run twice simultaneously.** Two common ways the same job fires twice:
- A deploy fires the same scheduled job during cutover (old + new instance both run).
- Two API replicas both have the cron registered.

Standard mitigation — every cron handler acquires a single-instance lock before mutating data:

```ts
// Postgres advisory lock approach (no extra infra):
const lockKey = hashStringToInt(jobName);   // stable hash
await prisma.$executeRaw`SELECT pg_try_advisory_lock(${lockKey})`;
const got = (result[0] as any).pg_try_advisory_lock;
if (!got) {
  logger.info(`[cron:${jobName}] another instance holds the lock; skipping`);
  return;
}
try { await handler(); } finally {
  await prisma.$executeRaw`SELECT pg_advisory_unlock(${lockKey})`;
}
```

Or Redis `SETNX` with a TTL longer than the job's expected runtime. Either works — pick the one that fits the app's existing infra.

**Idempotency on top of locking:** even with a lock, every handler should be safe against partial failures. Mutation patterns: upsert instead of insert when possible; mark rows as "processed" with a timestamp so re-runs skip them; transaction-wrap multi-step writes.

Standard scheduled jobs every app may need:

- Trial expiry sweep (when Throttle ships)
- Webhook retry sweep
- Deletion grace period sweep (§15.2)
- Stale API key reminder (§13)
- Activity log cold-storage migration (§14)
- **Stale notification cleanup** — delete read `Notification` rows older than 90 days; unread rows kept indefinitely (or per app policy). Prevents the table from growing unbounded.
- **Abandoned partner-trial-org cleanup** — delete pre-created partner trial orgs after 14 days if the invited client never signed in (§7).

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

### Team artifacts (every app, every repo)

Every app's repo carries the same small set of living docs at `docs/`:

| Artifact | Location | Purpose | Cadence |
|---|---|---|---|
| **`README.md`** | repo root | One-page orientation: what this is, how to run it, where the docs are | Updated on first impression breaking changes |
| **`CHANGELOG.md`** | repo root | User-facing changelog. **Auto-generated** from PR titles + the optional `## Changelog` section in PR bodies (per the CI section above) | On every merge to main |
| **`docs/roadmap.md`** | per repo | Living team roadmap — what's shipped, in flight, planned, parked. Customer-shareable on request | Updated as PRs merge and as priorities shift |
| **`CLAUDE.md`** | repo root | Project memory for Claude Code agents (and a useful onboarding aid for humans). Architecture overview, conventions, deployment quirks | When non-obvious behavior or convention is added |

**`docs/roadmap.md` format:**

A four-section table — **Shipped** / **In flight** / **Planned** / **Parked** — with these columns: feature, status, version (if shipped), owner, target date (if planned). The roadmap stays product-language (what the customer experiences), not engineering-task-language (what the dev team does). Engineering breakdown lives in PRs, issues, or planning docs, not here.

```markdown
# Foundry IMS — Roadmap

## Shipped
| Feature | Version | Owner |
|---|---|---|
| Affiliate links + dashboard | v1.148.0 | Kal |
| Clerk auth migration | v1.122.0 | Kal |

## In flight
| Feature | Owner | ETA |
|---|---|---|
| Marketing-site signup flow (v3.5 alignment) | Kal | next sprint |

## Planned
| Feature | Owner | Target |
|---|---|---|
| Partner program | Kal | Q3 |
| Throttle billing integration | TBD | gated on Throttle availability |

## Parked
| Feature | Reason |
|---|---|
| Mobile app | Out of scope; revisit when web demand saturates |
```

### Pull request workflow

**Every change to every repo goes through a PR.** No direct pushes to `main`. This is the enforcement layer that makes auto-versioning, auto-changelog, and the audit-via-PR-review tenancy lint principle (§21.12) work.

Conventions every app follows:

- **Branch naming**: `<author>/<short-description>` — e.g., `kal/affiliate-api`, `joud/fix-order-sync`. Lowercase, hyphens, no slashes after the author segment.
- **PR title drives semver**: prefix with `feat:` (minor bump), `fix:` (patch), `breaking:` (major). No prefix → patch by default. (Per the CI section above.)
- **PR body structure**:
  - `## Summary` — 1–3 bullets explaining what and why
  - `## Changelog` — user-facing one-liners that flow into `CHANGELOG.md` (see the CI section above)
  - `## Test plan` — checklist of what to verify after deploy
- **Squash-merge by default** so `main` history reads as one-commit-per-PR. Linear, bisect-friendly, matches the auto-version-per-PR cadence.
- **Merge gate**: CI green + at least one approval. Solo authors on small docs/typo PRs may self-merge with judgment, but anything touching code or schema needs a second set of eyes.
- **Never skip hooks** (`--no-verify`) or bypass signing without explicit reason in the PR body. If a hook fails, fix the underlying cause.

**Direct pushes to `main` are blocked at the GitHub branch-protection level** in every Epic-Design-Labs repo. Setting up branch protection is part of the new-app bootstrap (§20).

### API versioning

- Standardized on URL prefix: `/api/v1/`. **All endpoint paths shown elsewhere in this doc** (`/affiliates/me/code`, `/users/invite`, `/orgs/me`, etc.) are implicitly under this prefix — the prefix is omitted in examples for readability.
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

### Standard env var names

One canonical name across every app — prevents future divergence:

| Env var | Purpose |
|---|---|
| `APP_VERSION` | Release identifier; surfaced via `/health.version` and Sentry `release` |
| `NODE_ENV` | `development` / `staging` / `production` |
| `DATABASE_URL` | Postgres connection |
| `CLERK_PUBLISHABLE_KEY` | Clerk public key |
| `CLERK_SECRET_KEY` | Clerk admin SDK key |
| `CLERK_WEBHOOK_SECRET` | Clerk webhook signature verification (§3.9) |
| `SENTRY_DSN` | Per-app Sentry project DSN |
| `RESEND_API_KEY` | Resend transactional email |
| `RESEND_FROM_ADDRESS` | Default sender, e.g. `Foundry IMS <orders@ims.foundryims.com>` |
| `DISPATCH_API_URL` | Dispatch Tickets endpoint (§10) |
| `DISPATCH_API_KEY` | Per-app Dispatch secret |
| `DISPATCH_BRAND_ID` | Per-app Dispatch brand identifier |
| `THROTTLE_API_URL` ⏸️ | Throttle endpoint (per §23) |
| `THROTTLE_API_KEY` ⏸️ | Throttle admin key |
| `THROTTLE_WEBHOOK_SECRET` ⏸️ | Throttle webhook signature verification |
| `UPSTASH_REDIS_URL` | Rate-limit + lock backing store (§22) |
| `UPSTASH_REDIS_TOKEN` | |
| `INTERNAL_API_KEY` | Cross-service secret for internal-only endpoints |
| `<APP>_API_KEY_PREFIX` | The key prefix this app uses (e.g., `fims_` for Foundry) |

Apps may add their own (third-party integrations, app-specific feature flags). Names of standard things must match this table.

### Secret rotation policy

Every secret has a documented rotation cadence. Default policies:

| Secret | Rotation cadence | Notes |
|---|---|---|
| **API keys** (`<app>_*`) | User-managed; UI surfaces stale-key reminder at 90 days (§13). Mandatory rotation on suspected compromise. | Owners / admins via Settings → API Keys |
| **Webhook secrets** | User-managed via `POST /webhooks/:id/rotate-secret` (§12) with 24h dual-sign window | Mandatory rotation on suspected compromise |
| **`CLERK_SECRET_KEY`** | Annually + on suspected compromise. Requires coordinated env update across all environments running the app | Recovery: Clerk dashboard → Rotate key |
| **`CLERK_WEBHOOK_SECRET`** | Annually + on suspected compromise. Coordinate with restart of webhook handler instances | |
| **`RESEND_API_KEY`** | Annually + on suspected compromise | |
| **`DISPATCH_API_KEY`** | Annually + on suspected compromise | |
| **`THROTTLE_API_KEY`** ⏸️ | Annually + on suspected compromise | |
| **`INTERNAL_API_KEY`** | Annually + on every team-member offboarding with knowledge of the secret | |

**Compromise-driven rotation always wins.** If a secret is suspected compromised, rotate immediately regardless of cadence. Document the rotation in the §14.5 audit log and notify affected stakeholders.

### Uptime monitoring

Sentry catches errors but doesn't tell you "is the API reachable at all." Every production app subscribes to an uptime monitoring service that pings `/health` from outside the VPC.

**Standard service:** [Better Uptime](https://betteruptime.com) (per-app monitor, alerts to Slack / email / SMS). Apps may substitute Pingdom, Cloudflare Health Checks, or AWS Synthetic Canaries with documented justification.

Alert thresholds:
- **Down for 60s** — page on-call.
- **Status page incident auto-created** — public-facing status page reflects outage to customers in real time.

### Email warmup

When a new app sets up its Resend sender domain, **the marketing team needs a 2-week warmup ramp** before high-volume sends. Without it, ESPs (Gmail, Outlook) flag the domain as suspicious and bounce-rate spikes.

Plan a warmup window in the new-app launch timeline:
- Week 1: ≤100 emails/day, monotonically increasing
- Week 2: ramp to expected steady-state volume
- Configure SPF, DKIM, DMARC at DNS level **before** the first send. Resend's onboarding has a checklist; follow it.

### Testing standards

Every app has, at minimum:

- **Integration tests for auth + core domain.** Auth tests cover signup, login, org switching, leave/close. Core domain tests cover the app's primary user-facing flows (Foundry: import + product creation; Dispatch: ticket lifecycle; Rally: attribution match).
- **No coverage target imposed by the standard.** Apps set their own per-domain coverage based on risk. The standard's only requirement is that the auth + RBAC paths are covered; everything else is per-app judgment.
- **CI runs the integration suite on every PR.** Failing tests block merge.
- **No global mocking of Clerk in tests.** Use Clerk's test JWTs or a per-test mock at the boundary; mocking the entire `ClerkGuard` defeats the purpose.

### Shared library packaging

Shared dev dependencies (`@epic/disposable-emails`, `@epic/sentry-config`, `@epic/email-templates`, `@epic/cookie-consent`, future `@epic/eslint-plugin-tenancy`) are published to a **private GitHub Packages registry** scoped to the `Epic-Design-Labs` org.

Apps consume them by:

1. Adding the registry to `.npmrc`:
   ```
   @epic:registry=https://npm.pkg.github.com
   //npm.pkg.github.com/:_authToken=${GITHUB_PACKAGES_TOKEN}
   ```
2. `npm install @epic/<package>` like any other dep.
3. Setting `GITHUB_PACKAGES_TOKEN` in CI (PAT with `read:packages` scope).

Versioning: shared libraries use semver; breaking changes go to a new major version with a 1-month deprecation window for the old major.

Source-of-truth repo: `Epic-Design-Labs/shared-libraries` (a single monorepo, Yarn workspaces). Each library is its own publishable package within.

### API versioning scope

`/api/v1/` (§17.5) refers to the **customer-facing public API** — endpoints third parties integrate with using API keys.

**Internal admin BFF routes** (`app.<domain>/api/*` — Next.js API routes, server actions, admin-only endpoints) are NOT versioned. They co-deploy with the admin frontend, schema changes are coordinated within the same PR, and no third party consumes them. Adding `/v1/` to admin BFF paths is overhead with no benefit.

Customer-facing API routes ALWAYS go through `api.<domain>/api/v1/`. Admin-only BFF routes stay at `app.<domain>/api/...` without a version segment.

### Mobile / PWA

Out of scope for v3 standardization. App-by-app decision, no shared pattern, no mandatory commitment.

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

The marketing site is the **front door** for new signups, not just a brochure. It hosts the actual signup form (via embedded Clerk), the conversion thank-you page where pixels fire, and the affiliate-link cookie capture.

### 18.1 Required pages and routes

Every marketing site must have:

1. **Home + content pages** — the usual marketing surface. Any of these can receive an affiliate `?r=` URL.
2. **`/signup`** — embedded Clerk `<SignUp />` component (per §3.2). Signup happens here, on the marketing domain.
3. **`/welcome`** — thank-you page, conversion pixel host (per §18.3). Users land here after Clerk completes signup.
4. **CTAs throughout the site** that route to `/signup` (NOT to `app.<rootdomain>`).
5. **`/partners`** 🧭 — explains the partner program.
6. **`/partners/apply`** 🧭 — submits to API.

### 18.2 Affiliate cookie capture (cross-subdomain)

Every page on the marketing site captures `?r=` into a cookie scoped to the entire root domain. Two changes from earlier versions of this doc:

- **Cookie domain is `.<rootdomain>`**, not the host. This makes the cookie available on `<rootdomain>`, every subdomain (`landing.<rootdomain>`, `accounts.<rootdomain>`, `app.<rootdomain>`, `api.<rootdomain>`, etc.).
- **Link rewriting is no longer needed.** Because the cookie is cross-subdomain, the admin can read it directly when the user eventually lands there. No URL-param bridging required. (The `?r=` URL param is still the *capture* mechanism on first landing, but it doesn't need to be propagated through links.)

```js
(function () {
  var COOKIE = 'affiliateR';
  var MAX_AGE = 30 * 24 * 60 * 60;
  var VALID = /^[A-Za-z0-9]{8}$/;  // exact generated code length
  var ROOT = location.hostname.replace(/^[^.]+\./, '');  // strip leftmost subdomain — adjust per site if hosted at apex

  var ref = new URLSearchParams(location.search).get('r');
  if (ref && VALID.test(ref)) {
    // Domain=.<rootdomain> makes the cookie cross-subdomain
    document.cookie = COOKIE + '=' + encodeURIComponent(ref) +
      ';domain=.' + ROOT +
      ';max-age=' + MAX_AGE +
      ';path=/;samesite=lax';
  }
})();
```

The signup page reads this cookie when the user submits the Clerk `<SignUp />` form and passes the value as `unsafeMetadata.affiliateCode`. The webhook handler (§3.9 `user.created`) reads it and writes the `Referral` row.

### 18.3 Conversion tracking on the welcome page

The marketing site is responsible for firing **conversion pixels** when a signup completes. Pixels live on `<rootdomain>` (NOT on the admin domain — pixel attribution is host-bound; pixels loaded on the marketing site can't see events on the admin).

`/welcome` is the conversion page. It loads the standard set of pixels and fires the conversion event on page-mount. Standard pixels every app loads (subject to consent — §18.4):

| Pixel | Standard event | Purpose |
|---|---|---|
| Google Ads / GA4 | `sign_up` | Conversion attribution for Google Ads campaigns + GA4 funnel tracking |
| Meta Pixel | `CompleteRegistration` | Facebook / Instagram ads |
| LinkedIn Insight Tag | `Lead` (or custom conversion) | LinkedIn ads |
| TikTok Pixel | `CompleteRegistration` | TikTok ads |
| Reddit Pixel | `SignUp` | Reddit ads |

Apps may skip pixels they don't need; the standard is "if you run paid ads on platform X, fire the conversion event on the welcome page." Recommended implementation: load pixels via Google Tag Manager (GTM) on every marketing-site page; fire the conversion event via `dataLayer.push({ event: 'sign_up', ... })` from `/welcome` only.

The welcome page also:
- Optionally clears the `affiliateR` cookie (it's been used for attribution; no further purpose).
- Shows a brief celebration / orientation (e.g., "Your workspace is ready").
- Presents a **"Continue to Dashboard"** primary CTA → `app.<rootdomain>`. Clerk session is already live (cross-subdomain), so the user lands directly in the dashboard.

### 18.4 Cookie consent

Every marketing site that targets EU traffic implements a consent banner. **Standard library: `@epic/cookie-consent`** (shared dev dependency, see §1) — handles the banner UI, consent storage, and exposes a `hasConsent('functional' | 'analytics' | 'marketing')` API.

- **Functional cookies** (the affiliate `?r=` cookie, Clerk session cookies): set without explicit opt-in. Disclosed in the banner's "what we use" panel.
- **Analytics + marketing cookies** (GA4, Meta Pixel, LinkedIn, TikTok, Reddit, etc.): require explicit opt-in. Pixels must check `hasConsent('marketing')` before firing the conversion event.

**Legal sign-off on the affiliate cookie's "functional" categorization is still pending.** Until cleared, apps targeting EU traffic should re-categorize the affiliate cookie as "marketing" and require opt-in. Apps not targeting EU traffic may skip the banner entirely; document that decision per app.

Reference: `astro-foundryims/src/layouts/Layout.astro` for the cookie capture script.

---

## 19. Domain Conventions

Every app uses its own root domain. The structure within that domain is consistent:

| Subdomain | Purpose | Example (Foundry IMS) |
|---|---|---|
| `<rootdomain>` | Marketing site (front door — hosts signup + welcome per §18) | `foundryims.com` |
| `app.<rootdomain>` | Admin / product app | `app.foundryims.com` |
| `api.<rootdomain>` | API endpoint | `api.foundryims.com` |
| `accounts.<rootdomain>` | Clerk vanity subdomain (CNAME to Clerk for auth UI elements like email-verification links) — required by Clerk for some flows even when signup is embedded on the marketing site | `accounts.foundryims.com` |
| `<purpose>.<rootdomain>` | Optional marketing landing pages / microsites (paid-acquisition, product-launch sites, etc.). Receives the cross-subdomain `affiliateR` cookie automatically. | `launch.foundryims.com`, `pricing.foundryims.com` |
| `<app>.<rootdomain>` | Email sender subdomain | `ims.foundryims.com` |

**Each app gets its own root domain.** Not subdomains under `epic.dev`.

**Cross-subdomain cookie scope.** The `affiliateR` cookie (§6, §18.2) is set with `Domain=.<rootdomain>` so it follows the prospect across `<rootdomain>`, any landing-page subdomain, `accounts.<rootdomain>` (Clerk), `app.<rootdomain>` (admin), and is readable server-side from `api.<rootdomain>` requests. Clerk session cookies follow the same pattern (Clerk handles this when you configure the vanity subdomain).

---

## 20. New-App Standardization Checklist

When bootstrapping the next app, replicate in this order:

### Foundation
- [ ] **Root domain registered** + DNS configured per §19
- [ ] **Clerk projects: prod + staging** with the §3.1 dashboard config (`force_organization_selection: false`)
- [ ] **Clerk vanity subdomain** (`accounts.<rootdomain>`) configured per §19
- [ ] **Marketing site has Clerk publishable key** (NOT the secret key) for embedded `<SignUp />` per §3.2
- [ ] **Marketing-site `/signup` page** with embedded Clerk `<SignUp />` + cookie-to-`unsafeMetadata` wiring per §3.2 / §6
- [ ] **Marketing-site `/welcome` page** with conversion pixels + "Continue to Dashboard" CTA per §18.3
- [ ] **Clerk `afterSignUpUrl`** set to `<rootdomain>/welcome`
- [ ] **Clerk webhooks** wired with `CLERK_WEBHOOK_SECRET` per §3.9 (`user.created` for org auto-provision, plus `user.deleted`, `organization.deleted`, `user.updated`, `session.created`, membership events)
- [ ] **Sentry project** + `instrument.ts` + `@epic/sentry-config` per §17 — `APP_VERSION` env var
- [ ] **Resend workspace** + sender domain + 2-week email warmup window planned per §9 / §17.6
- [ ] **Better Uptime monitor** (or equivalent) for `/health` per §17.6
- [ ] **`Organization` + `User` models** per §3.5 (include `accountType`, `affiliateCode` on Organization, `deletedAt` on User, NO `@unique` on `clerkUserId`, nullable `email` / `name` / `clerkUserId` for tombstones)
- [ ] **`ClerkGuard`** with permission re-validation + opportunistic Clerk metadata sync per §3.5 / §3.6 / §16
- [ ] **Disposable email blocking** at signup per §3.3
- [ ] **Smooth signup path** — auto-create org on first load per §3.2
- [ ] **Per-environment isolation** — staging Clerk + Sentry env tag + per-env DB per §17.5
- [ ] **Standard env var names** match the §17.6 table
- [ ] **Branch protection on `main`** — direct pushes blocked, PR + CI green required, per §17.5 PR workflow
- [ ] **Team artifacts in place** — `README.md`, `CHANGELOG.md`, `docs/roadmap.md`, `CLAUDE.md` per §17.5

### Core features
- [ ] **Org create / switch / leave / close flows** per §3.8
- [ ] **Org switcher dropdown** in the header per §3.7
- [ ] **Users & roles** per §4 (UserRole enum, shared baseline permissions, `RequirePermission` decorator, Settings → Users)
- [ ] **`Referral` model + `Organization.affiliateCode`** per §6 — note: per-org, not per-user
- [ ] **Affiliate endpoints** per §6
- [ ] **Settings → Affiliate** subpage with copy-link UI per §6
- [ ] **Cross-subdomain affiliate cookie** (`Domain=.<rootdomain>`) script on marketing site per §18.2
- [ ] **Conversion pixels on `/welcome`** (Google Ads, Meta, LinkedIn, etc. as applicable) per §18.3
- [ ] **Admin `/signup` route remains** as a fallback entry point but is not the primary CTA target

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
- [ ] **Secret rotation runbook** per §17.6 (annual cadence + compromise-driven)
- [ ] **Integration test suite** for auth + core domain per §17.6 (CI gate)
- [ ] **Shared library access** — `.npmrc` + `GITHUB_PACKAGES_TOKEN` for `@epic/*` packages per §17.6

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

10. **Permissions are checked on every request.** Sessions don't cache role data beyond a short documented TTL (5–30 seconds default per §16). A demotion takes effect within seconds, not at session/token expiry.

11. **Privacy and deletion are first-class.** Right-to-deletion is built in, not bolted on. PII is in nullable fields only. Activity logs reference user IDs, not emails. Audit logs use HMAC for any necessary email hashes. Backups have documented 30-day retention with deletion-rerun-on-restore.

12. **Defense in depth on tenancy.** Every query filters by `orgId`. Every API request is scoped to the caller's active org. **PR review checklist** treats unscoped `find*`/`update*`/`delete*` on org-owned tables as a bug. The future tool is **`@epic/eslint-plugin-tenancy`** (custom ESLint rule using TypeScript AST + Prisma schema introspection to flag unscoped queries on org-owned tables) — 🧭 not built yet. Until it ships, manual review is the enforcement mechanism. Postgres RLS as a future hardening layer.

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
| **Per-API-key request budget** | **Default for free / no-billing apps: 10,000 requests / hour per key.** Tier-based numbers come from the Throttle billing doc when it ships. | Apps without billing yet have a usable default; Throttle-driven tiers override when ready. |
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

> **Status as of v3.2:** Throttle is Epic's intended billing platform — **not yet built or live**. Phase: **design**. ETA: **TBD, gated on its own design doc**. Until §23 is upgraded out of stub status, **do not depend on Throttle for any blocking design decision in any app**. Sections marked ⏸️ in §2 are blocked on this doc landing.

### What Throttle is (and isn't)

Throttle will be Epic's billing platform — a Stripe-style layer that issues invoices, processes subscriptions, and emits billing events. **It is not a system of record for users or organizations** — Clerk is, and stays so. Throttle's customer records are a downstream subscription view of the same orgs that already exist in Clerk + the local DB.

### Customer-to-org mapping (canonical)

To avoid the three-source-of-truth problem (Clerk users, Throttle customers, local DB), the mapping is locked in upfront:

- **One Throttle customer per Clerk Organization.** The local `Organization` row carries a `throttleCustomerId String? @unique` field reserving the link.
- **Throttle has no concept of users.** Subscription state belongs to the org. Individual users don't have separate billing profiles. (If multi-user billing visibility is needed, that's a Throttle Dashboard role concern, not a Clerk-level identity concern.)
- **Direction of trust:** the local DB is canonical for org existence; Throttle is canonical for subscription state. When an org is created, we provision a Throttle customer in the same transaction. When an org is closed, we cancel the Throttle customer.
- **No customer record exists for users who aren't in any org.** Rules out a "personal billing profile separate from work account" model. Aligns with B2B SaaS norms.

This locks in **two** sources of truth (Clerk identity + Throttle billing) instead of three, with one-way provisioning from Org → Throttle.

### Reserved schema

Add to `Organization` ahead of Throttle integration so activation is a code change, not a migration:

```prisma
model Organization {
  // ... existing fields
  throttleCustomerId  String?  @unique  // populated when Throttle integration ships
}

model BillingEvent {
  // see §17.5 Throttle webhook stub — already reserved
}
```

### Reserved integration points

This doc reserves integration points for:

- Trial creation, expiration, and reminder emails (§8)
- Trial-to-paid conversion detection and `Referral.convertedAt` updates (§7, §8)
- Partner commission calculation (10% recurring, monthly accrual, paid month N+1) (§7)
- Payout processing (PayPal Mass Pay direction; details in billing doc) (§7)
- Subscription lifecycle webhook events (`subscription.changed`) (§12)
- Dunning and payment failure handling (§11 — `billing.payment_failed` is transactional notification class)
- Per-API-key request budget tier definitions (§22)

### Webhook flow (architectural decision)

When Throttle ships, billing events flow as: **Throttle → app's `/webhooks/throttle` handler → app emits its own standardized event to customer-defined webhooks per §12.**

The app re-emits because:
- Customers integrate with the app's domain events (e.g., `subscription.changed` with the org's data shape), not Throttle's internal event vocabulary.
- The app can enrich (org context, plan tier names) and filter (only events the customer's webhook subscribed to).
- Throttle credentials never reach customer endpoints.

### Sections currently blocked on the billing doc

- §7 Partner Program — commission calculation and payout flows
- §8 Trials — conversion detection and expiration enforcement
- §11 Notifications — `billing.payment_failed` semantics
- §22 Rate Limiting — per-tier API key budgets

Until §23 is upgraded out of stub status, these features remain ⏸️ blocked. Schema reserves the relevant fields so activation is a code change, not a migration.
