# Epic Design Labs — Standard Features (v3.9)

**Canonical reference** for the foundational features every Epic Design Labs app should have. New apps adopt this whole stack so users get a consistent experience — same login, same org model, same affiliate program, same support widget — across the whole portfolio.

> **About this document**
>
> This is Epic's global standardization guide for all product apps (Foundry, Throttle, EvidentUGC, Dispatch Tickets, Rally Attribution, Clarion, and any future apps). The goal is consistency where it reduces friction — not consistency for its own sake.
>
> If your current implementation diverges from these standards, **discuss with leadership before making changes.** Some divergences may be intentional or load-bearing. Others may represent opportunities to align. Major rework should never happen without conversation first.
>
> **Platform direction:** All Epic apps are migrating to a standardized auth and payments stack. **Clerk** handles identity (users and organizations). **Throttle** handles transactions and billing — **live as of v3.7**, taking real payments in Evident since 2026-08-08; §23 is now a verified integration contract rather than a stub. (The v3.6 naming flag still stands: the shipped THROTTLE *sales-channel* platform is a different thing.) Remaining Stackbe apps are in active migration; Foundry completed its Stackbe removal 2026-05-23. This standardization assumes Clerk + Throttle as the baseline architecture going forward.
>
> Visual design is intentionally *not* standardized — each app earns its own look and feel. What we standardize is **feature parity**: every app has the same login, org model, affiliate program, support, notifications, etc.

**Status legend (status reflects reality, not aspiration):**

- ✅ **Implemented** — Shipped and verified in at least one app
- 🚧 **Partial** — Shipped in one app but not universally consistent, or incomplete in the reference implementation
- 🧭 **In design** — Spec exists, implementation pending
- ⏸️ **Blocked** — Blocked on a dependency (e.g., a third-party credential or an unshipped upstream)
- 📋 **Proposed** — New in v3; not yet implemented anywhere

**Reference implementation:** Foundry IMS (api: `Epic-Design-Labs/app-foundry-ims-api`, admin: `app-foundry-ims-admin`, marketing: `astro-foundryims`). Foundry is the most current implementation; if you find a better pattern, propose a standard update rather than diverging silently.

**Document structure (v3.9):** sections are grouped into thematic parts; **§ numbers are stable identifiers and are no longer strictly sequential** (relocated sections keep their numbers so cross-references — including code comments citing them — stay valid).

- **Part I — Foundations:** §1 Overview · §2 Status Table + Foundry audit checklist
- **Part II — Auth & Identity:** §3 Auth & Organizations · §4 Users & Roles · §5 Account Types · §16 Session Permission Re-Validation
- **Part III — Affiliate Program:** §6
- **Part IV — Referrals & Partner Program:** §7
- **Part V — Billing (Throttle):** §8 Trials + subscription lifecycle · §23 Throttle Integration · §24 Plan Entitlements & Feature Gating
- **Part VI — Communication & Support:** §9 Email · §10 Support · §11 Notifications
- **Part VII — Platform Infrastructure:** §12 Outbound Webhooks · §13 API Keys · §14 Activity Log · §14.5 Audit Log · §15 Export & Deletion · §17 Sentry · §17.5 Operational Patterns · §22 Rate Limiting · §25 Tenancy Enforcement · §26 Agent Access (MCP) · §27 Public API Documentation
- **Part VIII — Web Presence & Marketing:** §18 Marketing Site Contract · §19 Domain Conventions
- **Part IX — Adoption & Governance:** §20 New-App Checklist + Required Screens · §21 Principles

### Changes in v3.9

**Theme: the reference implementation reached 100% of the non-billing auth/affiliate/partner baseline (Foundry api v3.66–v3.68, 2026-08-31) — and the standard absorbs what building it settled.** Every v3.6 "needs a decision" flag in those areas is now decided: some by fixing Foundry, some by amending the prescription that shipped experience contradicted. Billing-dependent items (§6 commission accrual, §7 conversion/election mechanics, §8 lifecycle) remain with the Throttle adoption phase.

**Prescriptions amended (reality won):**

- **§3.1/§3.3 — org auto-provisioning is Clerk-native now.** The app-side `user.created` provisioning path + admin first-load fallback are withdrawn: Clerk's "Create first organization automatically" (with naming rules) ships the §3.3 experience with zero code, and the webhook path would *race* Clerk's membership-required signup UI. §3.1's `automatic_organization_creation: false` is inverted to ON. Webhooks now exist to mirror, never to provision. Verified in Foundry production 2026-08-31.
- **§3.6 — "push on login" deleted.** Per-request Clerk-metadata writes are banned outright: the pattern caused Foundry's 5–15s page loads (guard writes serializing on a row lock). Write-on-change is the whole sync story; the client tolerates briefly-stale `publicMetadata` because the API never trusts it.
- **§16 — the `permission_changed` frontend handling is reload-and-explain, not redirect-to-/login.** With Clerk the session is still valid, so a /login redirect bounces straight back in. The intent — no client operating on stale permissions — is met by a hard reload plus a "your permissions changed" notice. Also stated: the change detector may be per-process; each instance failing once is acceptable.
- **§4 — MEMBER gets a real definition.** The old baseline gave MEMBER nothing but settings/support/affiliate reads — unusable in any real app, so every app would have extended it divergently. MEMBER is now defined as **standard operational access**: full day-to-day read/write in the app's domain, zero org administration (no users/API keys/webhooks/apps/settings-write/audit/export). Includes the migration pattern for apps that had MEMBER aliased to ADMIN: promote existing rows, never silently strip.
- **§3.7/§3.9 — guarded role sync.** Clerk's `org:admin`/`org:member` is a lossy 2-value projection of an app's role enum; a membership event may only move a user within the generic ADMIN↔MEMBER pair, and must never overwrite OWNER or an app-specific role. (The unguarded mapping silently reset a WAREHOUSE invitee to MEMBER — invisible in Foundry precisely while MEMBER≡ADMIN.) Plus: the first `org:admin` member of an OWNER-less org becomes OWNER — without this rule, Clerk-organic orgs have no one holding `org.close`.
- **§3.9 — webhook failure handling gets a taxonomy.** "Return 5xx so Clerk retries" taken literally causes retry storms on permanently-unprocessable events; ack-and-log-everything (Foundry's old behavior) hides real failures behind healthy 200s. The rule is now: *unresolvable* events (unknown org, missing rows) log-and-return 200 inside the handler; only *transient* failures (DB down, provider error) propagate as 5xx. Handlers stay idempotent either way.
- **§12 — header prefix ruling: `x-<app>-*`, not `X-Epic-*`.** Customers integrate with the app's brand; the portfolio is an internal fact. `X-Epic-App` survives as an optional disambiguator.
- **§14 — the denormalized actor-email snapshot is blessed.** `userEmail` alongside `userId` keeps history readable after a user leaves; the cost is a §15.2 obligation — the deletion job MUST scrub the snapshots. Principle 11 gains this exception explicitly.
- **§4 guard hazards — "audit for it" becomes "a test fails on it."** The class-level-`@Public()` grep is now a required CI spec (a source-scan test that fails the build), alongside the `@SkipSessionAuth`-style narrow opt-out: a controller may skip the session guards only by a decorator honored by *named* guards, and the same spec enforces that every user of it installs its replacement auth guard. A decayed one-time audit is how the hazard shipped in the first place.

**New sections:**

- **§26 Agent Access (MCP)** — the portfolio pattern for giving AI agents (Claude, ChatGPT, n8n, etc.) access to an app: MCP server topology, per-user org-pinned credentials, first-party OAuth with RFC 7591 dynamic client registration, directory listings, and the operational lessons (token-rotation races, one-session-per-tool-call clients). Foundry is the reference implementation; every app should expect this ask.
- **§27 Public API Documentation** — the audited-docs contract: an explicit module allow-list per public doc, 100% operation summaries and schema-property descriptions, maintenance endpoints excluded, and the toolchain traps that silently produce undocumented or over-documented APIs.

**Also:**

- **§3.4** disposable-email override mechanism now ✅ (Foundry: staff-gated, audited, expiring; UI placement and staff-email attribution accepted as equivalent to the spec's shape).
- **§7** gains assignment-enforcement semantics: the `autoAssignAll` default (ON = every partner-org member may act on every seat, assignment rows advisory; OFF = explicit assignments are the access list) and the rule that enforcement, when wired, happens at org-resolution time. Foundry status updated: profile/payout + team + assignment endpoints ✅ (management surfaces; enforcement deferred to the billing phase).
- **§13** absorbs Foundry's key-architecture extensions as the standard: key *types* (ADMIN vs surface-scoped keys like STOREFRONT with resource binding), per-key scopes that override role when present, and **grantable-scopes** — a user can never mint a key more powerful than their own role.
- **§14.5** gains the platform-level-events convention: global staff actions (e.g. disposable overrides) log to the acting staff member's org with a self-describing `resource`, rather than inventing an org-less audit store.
- **§16** status 🚧→✅ (Foundry ships the 401 + client handling).
- **§18.3** conversion pages: Foundry's `/signup` + `/welcome` are now noindex and sitemap-excluded.
- §2 status table + Foundry audit checklist synced to api v3.68.0 / admin (2026-08-31).

### Changes in v3.8

**Theme: three places where this document asserted something that shipped experience contradicts.** Each is a demotion of an existing claim rather than a new feature area.

- **§24 Plan Entitlements & Feature Gating — new.** The portfolio bills by tier, and until now the standard covered *who may act* (§4) and *what they pay* (§8/§23) but never the layer that turns a subscription state into an actual restriction. It separates **plan entitlement** ("your tier doesn't include this") from **tenant feature toggle** ("you turned this off") — different causes, different UI, different recovery, and a single field for both eventually tells a paying customer their own setting is a billing problem. Carries two 🔴 callouts: a no-op gate is indistinguishable from a working one (Evident's enforced nothing for months, green dashboards throughout), and **activating a gate on an existing customer base is a destructive operation** requiring a full audit of live accounts first — one customer would have been locked out of an account holding 4,700+ of their own records. Also stipulates that a 403 without UI to handle it is a dead end, not a paywall.
- **§25 Tenancy Enforcement — new, and it supersedes PR review as the mechanism.** Principle 12 named manual review as the control for four revisions while the lint rule stayed unbuilt; in that time the same bug recurred **three times in one app**, the last after a dedicated audit had already fixed every Critical finding. Three repeats with review actively looking is evidence about the mechanism, not the reviewers — an absent clause is precisely what review is worst at catching. The standard is now **default-deny at run time**: unscoped queries on org-owned tables fail, cross-org work uses an explicit greppable annotation, and a completed audit is evidence about the past rather than a property the codebase holds. Principle 12 rewritten to match.
- **§17.5 gains a background-worker testing floor.** The testing standards described the API, while the code that revokes access and moves money runs on a timer in a different service. **Any job that can revoke access, delete data, or move money needs a test, and the service running it needs a harness** — Evident's worker has no jest config and no test script, and that is where its trial-expiry cron lives. Tests must cover **who the job skips**, not just who it acts on, and a production dry run is not a substitute.
- Every guard now needs a **denial-asserting test** (§17.5, §4, §24.2) — an allow-only test cannot distinguish a working guard from an unregistered one.
- **Scope note on app marketplaces:** only **Foundry and Throttle** will host third-party app marketplaces. Every other app, Evident included, has internally-built integrations only — so the app-marketplace gap in the v3.6 list is a two-app concern, not a portfolio standard.

### Changes in v3.7

**Theme: billing stops being a design document.** Throttle is live and has been billing real customers since 2026-08-08. Everything in v3.2–v3.6 that treated it as unbuilt is now either verified against production or corrected — and several of the design-phase guesses were wrong in ways that failed *silently*, which is why they get their own callouts rather than a quiet edit. **Reference implementation for Part V is Evident, not Foundry**; Foundry has not integrated billing.

**Commission economics — settled:**

- **§6 affiliate tier is 100% of the referred org's first month, one time, capped at $500 per referral, held 30 days.** This replaces the "10% recurring, no cap" figure carried since v3.1, which no app implemented. The cap is per referral with no per-referrer or per-period ceiling, and exists so one rule can span apps priced from $49/mo to four figures.
- **§7 partner tier is unchanged and now stated precisely:** 10% of every renewal, life of the subscription, **no cap**, and **nothing on the first payment** — that conversion is the partner's own work.
- **§7 gains a three-tier compensation table.** Affiliate, partner-client-paid, partner-white-label. No org is ever on two at once.
- **Reporting the commission basis is required; paying it is not.** Accrual and per-client reporting must ship. Payout rails (method, tax forms, mass pay) legitimately may not exist — Evident has none by choice — but nothing may imply to a partner that a reported commission was paid.

**Pay-for-client and handoff — the mechanics, not just the model:**

- **§7 Path A gains five required mechanics** for partner-pays-for-client checkout, each written after the corresponding production failure: record billing intent without flipping entitlement (an abandoned checkout otherwise marks a client partner-billed forever and silently suppresses the partner's own commission); roll the intent back on failure; route takeovers through change-plan, never create-checkout (otherwise the client gets a *second* recurring subscription); pass the discount flag explicitly; require explicit confirmation of the commission forfeit.
- **§7 Path B gains six required mechanics** for handoff — headlined by 🔴 **the payment method does not move.** The provider's customer keeps the partner's vaulted card, so the next renewal charges the partner or fails, and a browser-side card wallet means the server cannot even detect it. `cardHandoffPendingAt` + a persistent banner + an acknowledge endpoint are now required, along with re-pricing outside the transaction and creating a referral where none exists.
- **§5 gains `AccountEntitlement`** (NONE / AFFILIATE / REFERRAL / AGENCY) — the org-side axis that actually drives pricing and commission suppression. `accountType` answers "is this user a partner"; it never answered "how is this org compensated."

**§23 rewritten from stub to contract:**

- **Verified event vocabulary.** The guessed list (`invoice.paid`, `trial.ending`, `subscription.canceled`) does not exist. Real events are `subscription.activated / renewed / resumed / paused / plan_changed / past_due / payment_failed / cancelled` (two Ls), `payment.captured`, `payment.failed`, `cart.*`. Trial expiry is scheduled by the app, never announced by Throttle.
- **Org resolution is a four-step chain, and getting it wrong returns 200.** Subscription webhooks do not carry `externalCustomerId`; reading it alone dropped every subscription event for weeks behind healthy logs. `externalId` and `externalCustomerId` are different fields.
- **🔴 The one-customer-per-org invariant does not hold under partner billing.** All orgs a partner created share one customer, so `customer.externalId` identifies the *payer*, not the client — and invoice queries scoped by customer leak across a partner's clients (open defect in Evident).
- **Environment and auth facts that are not guessable:** the API key selects the environment (both share a host), nothing reads `*_LIVE_*` var names, auth is `x-api-key` and a Bearer request returns **200 with an empty body**, base path is `/api/v1`.
- **`BillingEvent` idempotency log is required and exists in no app**, Evident included.
- **Known gap: no billing address is collected**, so live authorizations carry no AVS or postal code.

**§8 trials:**

- 🔴 **The trial-expiry sweep must never expire a subscription paid into the future.** The obvious query (`TRIALING AND trialEndsAt < now`) locks out paying customers, because a provider can leave `trialing` on an already-charged subscription. Evident caught this one day before it fired. Skipped rows must be logged loudly, and the cron needs a test — background workers are exactly where harnesses don't exist.
- Extending a trial **delays the first charge**; nulling `trialEndsAt` removes the deadline entirely, which is lockout insurance, not a fix.
- The lifecycle table gains canonical API paths. **Reactivate is commonly missed** — Evident has no dedicated path for it.

**Auth:**

- **§4 gains two guard hazards, both of which fail open and silent:** `@Public()` on a controller *class* short-circuits every other guard on it and makes identity spoofable (route-level only, never class-level); and guard **registration location** decides whether a guard runs at all — root-module guards run before imported-module ones, and a misplaced registration still instantiates while blocking nothing. Every guard needs a test that asserts a **denial**.

**§21 principles:**

- **New principle 16 — silent success is the failure mode to design against.** The expensive bugs here never threw: a plan gate that never ran, an import that reported COMPLETED having imported nobody, a webhook that 200'd every event and recorded none. Prefer a loud skip to a quiet pass.
- **New principle 17 — provider contracts are verified against live traffic, not our own design docs.** §23 carries three worked examples of what a design-phase guess costs.
- Principle 6 rewritten for the new two-tier economics.

**§20:** billing screens are no longer ⏸️; adds the card-handoff banner and the commission statement (showing the uncapped basis where the $500 cap bound).

### Changes in v3.6

Two things at once: a **reality refresh** against the shipped Foundry repos (2026-08, api v3.54.x), and a set of **new stipulations** tightening the auth/affiliate/referral/billing baseline so these common functionalities are consistent and testable across apps.

**New stipulations:**

- **§7 partner economics — the per-client election (headline change):** a partner either keeps paying a client's bill white-label at a **20% discount netted at source** (raised from v3.1's 10%; Throttle invoices at 80% of list), or spins ownership off to the client and earns the **10% recurring commission**. One client, one model, never both. §6/§7/§21 and the §23 reserved integration points updated to match.
- **§6 affiliate-by-default:** every org has an affiliate code and every user in every role can copy a working share link from day one — the program is never opt-in, gated, or applied-for.
- **§4 multi-user orgs:** every org is explicitly multi-user/multi-role; no app may assume one-user-per-org.
- **§7 partner applications:** any agency can apply — two entry points (marketing `/partners/apply` and in-app Settings → Partner Program), one reviewed application; partners refer by physically creating the trial (unchanged, restated).
- **§8 subscription lifecycle table:** Throttle must support start-trial, convert, upgrade, downgrade, cancel, reactivate, and the white-label ⇄ client-paid model switch — each with an API path and a UI surface.
- **§20 required-screens inventory:** the canonical screen list for auth/affiliate/partner/billing, as the QA walk-through target.
- **§18.3 conversion pages are `noindex`.**

**Reality refresh:**

- The §2 status table and Foundry audit checklist are synced to what actually ships; per-section **Foundry status** callouts record where the reference implementation diverges, so divergences are visible decisions instead of silent drift. Where reality contradicts a prescription, the prescription is left intact and flagged — amending it stays a leadership call per the preamble.
- **Document reorganized into Parts I–IX** (see above); §16, §22, and §23 moved into their thematic parts, keeping their numbers.
- **Now ✅ (were 🧭/📋):** partner program §5/§7 (full API surface, partner dashboard, trial creation, seats on both sides, applications with approve/reject, abandoned-trial cleanup cron — commissions still ⏸️ billing), outbound webhooks §12, security audit log §14.5, right to deletion §15.2, and the entire v3.5 marketing-site front door (§3.2/§18: embedded signup, `/welcome` pixels, cross-subdomain cookie, vanity subdomain, link rewriting removed).
- **Statuses corrected:** disposable-email blocking 🧭→🚧 (enforced on invite/org-create/partner paths; warn-only in the Clerk `user.created` webhook), rate limiting 📋→🚧 (custom in-memory limiter — §22 callout), cron conventions 📋→🚧 (26 jobs live; no cross-instance locking).
- **Divergence callouts added** (each needs a decision — amend the standard or file the gap as work): §3.3/§3.9 org auto-provisioning unimplemented on both ends; §3.6 per-request Clerk-metadata push not implemented and contraindicated by a Foundry perf incident; §3.9 webhook handler returns 200 on processing errors rather than 5xx; §4 MEMBER aliased to full ADMIN permissions and VIEWER granted `support.write`; §11 notification schema is org-scoped with no per-user rows, audience, or transactional tier; §12 delivery contract uses `x-foundry-*` headers with no dead-letter state and no dual-signing on customer endpoints; §14 ActivityLog deliberately denormalizes `userEmail` (tension with principle 11); §17 no PII redaction shipped; §17.5 health-check shape differs and DB failure doesn't 503; §18.4 no cookie consent shipped.
- **§23 flag:** "Throttle" now names two things — the still-unbuilt billing platform this doc assumes, and the shipped THROTTLE sales-channel platform Foundry integrates with (Foundry serves catalog; Throttle owns checkout/orders). Needs a naming/priority decision next revision.
- Checklist items completed since v3.3 are checked off with completion notes; items done differently than specified are annotated rather than silently reworded.
- Known doc gaps for a future revision (Foundry shipped these with no standard): MCP server + first-party OAuth, app marketplace *(v3.8 scope note: **Foundry and Throttle only** — every other app has internally-built integrations, so this is not a portfolio standard)*, accounting integrations (QuickBooks live, Xero planned), headless storefront API (storefront-scoped keys + per-channel webhooks), public API docs, n8n / hosted-SFTP integrations.

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

# Part I — Foundations

## 1. Overview

Every Epic Design Labs app provides:

1. **Clerk identity** — magic link + Google + Apple. Each app runs its own Clerk project.
2. **Multi-tenant organizations** — every customer is one Clerk Organization mirrored locally for data isolation.
3. **Throttle for payments and billing** ✅ — live and taking real payments (Evident since 2026-08-08); §23 carries the verified integration contract. Remaining apps adopt it.
4. **Universal affiliate links** — any user with `affiliate.read` can copy their org's `?r=CODE` link and the org earns credit on signups.
5. **Partner program** ✅ (accrual + reporting live; payout rails ⏸️) — designated partners get a dashboard, can create client trials directly, and earn 10% of every renewal via partner seats. White-label (partner pays, 20% discount) and transfer-to-client (handoff, 10% commission) paths both supported, with required mechanics for each in §7.
6. **In-app support** — Dispatch Tickets wired into a Help section, scoped per-org.
7. **Transactional email via Resend** — `react-email` templates compiled at build time (🚧 Foundry still on inline HTML — see §9).
8. **Outbound webhooks** ✅ — push events to customer endpoints with HMAC signing and retries (DLQ + usage metrics 🚧 — see §12 Foundry status).
9. **Notifications, activity log, security audit log, API keys, data export, RBAC** — table-stakes infrastructure shared across apps.
10. **Right-to-deletion compliance** ✅ — GDPR/CCPA-compliant user data deletion via tombstone model.
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
| Clerk auth (magic link + Google + Apple) | ✅ | Foundry fully off Stackbe (2026-05-23); remaining Stackbe apps still migrating. |
| Org create / switch / leave / close | ✅ | All Clerk-native. Owner sole-leave guard enforced. |
| Org switcher UI | ✅ | Foundry's lives in the sidebar (visual freedom); switch / create / leave actions per §3.8. Create-org currently round-trips through sign-out for a token refresh. |
| Affiliate code generation + dashboard | ✅ | One code per org. `Organization.affiliateCode` (8-char), Settings → Affiliate page, stats + signups table. `User.affiliateCode` dropped. |
| Affiliate signup attribution | ✅ | `?r=` cookie → `unsafeMetadata.affiliateCode` → attribution on `organizationMembership.created` webhook (code cleared from Clerk after use); admin post-auth callback kept as fallback. Idempotent; last-touch wins. |
| Marketing-site signup front door (§3.2/§18) | ✅ | `foundryims.com/signup` embedded Clerk + `/welcome` conversion page + cross-subdomain cookie live; link rewriting removed; `accounts.` vanity subdomain CNAME'd. |
| Disposable email blocking at signup | 🚧 | Rejects on invite, org-create, and partner-trial paths; per-domain staff overrides with audit + expiry shipped (§3.4). Still warn-only for organic signups (the `user.created` webhook can't reject a user Clerk already created — blocking organic signups needs a Clerk-side restriction, open question). |
| Partner role / `accountType` | ✅ | `User.accountType` live; toggled on application approval; pushed to Clerk `publicMetadata` on change (not per-request — see §3.6). |
| Partner dashboard + partner seats | ✅ | Referrals/Seats/Team/Settings tabs, "Create trial for client" dialog, profile + payout settings, team seat assignments + `autoAssignAll` (§7), client-side per-seat tier control. Commissions endpoints still ⏸️ billing; assignment *enforcement* deferred with them. |
| Partner-created trials (direct referral) | ✅ | `POST /partner/trials` + `referralType: "direct"` + daily abandoned-trial cleanup cron. |
| Trial period / `trialEndsAt` | ✅ Evident · 🚧 Foundry | Live in Evident with the §8 expiry guard. Foundry reserved the fields 2026-08-31 (api v3.69.0: `Organization.status`/`trialEndsAt`/`throttleCustomerId`, existing orgs backfilled `active`); no lifecycle reads them yet. |
| Trial-expiry sweep safety guard | 🚧 | **Required** (§8): never expire a subscription paid into the future; log skipped rows loudly. Implemented in Evident after it nearly locked out a paying customer. Not present anywhere else; no test harness on the worker that runs it. |
| Conversion detection (`Referral.convertedAt`) | ✅ Evident · ⏸️ Foundry | Written from the subscription-activation webhook. |
| Commission accrual + reporting | ✅ Evident | Affiliate 100%-of-first-month capped $500 (§6); partner 10% of renewals, no cap (§7). Rates in one shared constants module. |
| Commission **payout** rails | ⏸️ | Accrual and reporting exist; nothing moves money. No payout method, tax forms, or mass-pay anywhere. Do not imply to partners that a reported commission has been paid. |
| Pay-for-client (white-label) checkout | ✅ Evident | 20% discount at source. Five required mechanics in §7 Path A — all five were written after a production failure. |
| Handoff (white-label → client-paid) | ✅ Evident | Six required mechanics in §7 Path B. 🔴 The payment method does **not** move; `cardHandoffPendingAt` + banner + acknowledge endpoint are required. |
| Throttle webhook contract | ✅ Evident · ✅ Foundry | Verified event vocabulary, signature + 5-min replay window, four-step org resolution (§23). The v3.6 guessed vocabulary was wrong and failed silently. Foundry receiver live 2026-08-31 (api v3.69.0), `resolve-organization.spec.ts` ported verbatim; `cart.expired` is retired upstream (rejected at endpoint registration). |
| `BillingEvent` idempotency log | ✅ Foundry · 📋 elsewhere | Required by §23; Foundry shipped the first one 2026-08-31 (api v3.69.0, persist-before-process, unique on provider event id with body-hash fallback). Evident still lacks it. Webhook delivery is at-least-once. |
| Billing-address / AVS collection | 🚧 | Checkout sends no billing address, so live authorizations reach the processor with no AVS or postal code — an unexplained decline risk. Blocked on confirming the `collect` field shape with Throttle (§23). |
| Support (Dispatch Tickets) | ✅ | Tickets scoped per-org via tag, third-party API wrapped server-side. |
| Transactional email (Resend) | 🚧 | Foundry uses Resend with inline HTML for PO send/follow-up. v3 standardizes `react-email`. Migration required. |
| Notification center (in-app bell + dropdown) | ✅ | Bell icon (unread dot, no count), dropdown of recent 20. No full-history page yet. |
| Notification preferences (per-method) | 🚧 | Foundry has per-category on/off only, and its `Notification` rows are org-scoped with no `userId` — the §11 baseline needs a schema migration, not an extension. |
| Notification transactional tier | 📋 | `deliveryClass: "user_pref" \| "transactional"`. Not implemented anywhere. |
| Outbound webhooks | ✅ | Shipped in Foundry: `WebhookEndpoint`/`WebhookDelivery`, management API + rotate + replay, every-minute worker, 8-attempt backoff, auto-disable. Contract deltas in §12 Foundry status (headers, no DLQ state, no usage metrics). |
| Activity log | ✅ | Per-org change history with entity-type/id pivot. Actor = `userId`/`userEmail`/`actorType` via AsyncLocalStorage — see §14 Foundry status. |
| Security audit log | ✅ | `AuditLog` model + `/audit` page live in Foundry; `auth.login`/`auth.logout` written from Clerk session webhooks. Remaining baseline events instrumented incrementally. |
| API keys | ✅ | `<prefix>_*` prefixed keys, role- or scope-gated, hashed in DB. Foundry adds `keyType` (ADMIN \| STOREFRONT) + channel binding; `expiresAt` supported by API but not exposed in the create UI. |
| Data export | ✅ | `GET /orgs/me/export` returns a ZIP of CSVs. |
| Right to deletion (GDPR/CCPA) | ✅ | Shipped in Foundry: 30-day grace + 24h immediate path, Clerk-delete-first tombstone, HMAC email audit. Shape differs from spec (state on `User`; no `DataDeletionRequest` model) — see §15.2 Foundry status. |
| Users & roles (RBAC) | ✅ | Invite via Clerk, local role assignment, permission decorators. Foundry's mapping now matches §4 (MEMBER operational re-map shipped 2026-08-31, existing MEMBERs promoted to ADMIN); contract pinned by a denial-asserting spec. |
| Session permission re-validation | ✅ | Foundry: role re-read behind a 30s single-flight cache; role change fails once with the coded 401; admin reloads + explains. (§16, amended handling.) |
| Sentry observability | 🚧 | Foundry has Sentry init + global filter. No `beforeSend`/PII redaction shipped; `@epic/sentry-config` does not exist yet; release var still `SENTRY_RELEASE`. |
| Rate limiting | 🚧 | Foundry runs a custom in-memory fixed-window limiter (per ECS task): 300/min per ADMIN API key + per-endpoint storefront limits, with `X-RateLimit-*`/`Retry-After` headers. No Redis, no per-IP signup limit, no usage endpoints — see §22 Foundry status. |
| Health checks | 🚧 | Foundry has `/health` but not the standardized shape: no `checks` object, DB failure returns 200 "degraded" rather than 503 — see §17.5 Foundry status. |
| Cron conventions | 🚧 | 26 jobs live via `@nestjs/schedule`; no advisory locks / Redis SETNX anywhere — single-task-safe only. See §17.5 Foundry status. |
| Plan entitlement gate | 🚧 | §24. Live and correct in Evident; **the 403 has no UI anywhere** so it currently renders as a generic error. Gate was a silent no-op for months before that. |
| Tenant feature toggles | ✅ Evident | Per-tenant on/off, modelled separately from plan entitlement (§24.1); nav hides disabled groups. |
| Tenancy: default-deny data layer | 🧭 | §25. **Not built anywhere.** Supersedes PR review as the mechanism — that mechanism let the same bug through three times in one app. |
| Background-worker test harness | 📋 | §17.5. Required where a job can revoke access, delete data, or move money. Evident's worker has **no jest config and no test script**; the trial-expiry cron lives there. |
| Custom fields | ✅ | Per-entity custom field defs + values. |
| Agent access (MCP) | ✅ Foundry · 📋 elsewhere | §26: edge MCP server, per-user org-pinned keys, first-party OAuth + RFC 7591 DCR, directory listings. In production since 2026-07. |
| Public API documentation | ✅ Foundry · 📋 elsewhere | §27: allow-listed public docs, 100% summaries + schema descriptions, maintenance endpoints excluded. Audited 2026-08-30. |

### Foundry audit checklist (work to align reference impl with v3)

Statuses synced to the shipped repos 2026-08-31 (api v3.69.0).

- [x] Drop `User.clerkUserId @unique` constraint, add `@@index([clerkUserId])` to support multi-org users *(done — nullable + indexed, `20260503143134_user_tombstone_fields`)*
- [x] Make `User.email`, `User.name`, `User.clerkUserId` nullable for tombstoning per §3.5 *(done, same migration)*
- [x] Migrate per-user `User.affiliateCode` to per-org `Organization.affiliateCode` *(done — `20260503143806_affiliate_per_org`)*
- [x] Verify `Referral.referrerOrgId` references local `Organization.id` *(done — SetNull relation + `affiliateCode` snapshot + `sharedByUserId`)*
- [x] Migrate `ActivityLog.createdBy` from email to userId *(done differently: actor is `userId` + `userEmail` + `actorType` enum stamped via AsyncLocalStorage; pre-existing rows read `UNKNOWN`; `userEmail` deliberately denormalized — see §14 Foundry status for the principle-11 tension)*
- [x] Add `support.read/write`, `affiliate.read`, `apikeys.manage`, `activity.read`, `audit.read`, `export.run`, `notifications.manage`, `webhooks.manage` as named permissions per §4 baseline *(all present in `src/auth/permissions.ts`; `notifications.manage` not yet checked by any admin surface)*
- [x] Add `permission_changed` 401 emission on session role mismatch per §16 *(done — api v3.67.0 + admin reload/toast handling, 2026-08-31)*
- [ ] Add per-method notification preferences (bell/toast/email per category) + `deliveryClass` field per §11 — note this now requires migrating `Notification` to per-user rows first (§11 Foundry status)
- [ ] Migrate transactional emails to `react-email` (current PO send/follow-up are inline HTML strings) per §9
- [ ] Adopt `@epic/sentry-config` with PII redaction per §17 *(library itself not yet created)*
- [ ] Rename `SENTRY_RELEASE` env var to `APP_VERSION` per §17 *(`APP_VERSION` currently exists only as a hand-maintained constant in `health.controller.ts`)*
- [x] Implement smooth-signup auto-create-org flow per §3.3 *(resolved 2026-08-31 the v3.9 way: Clerk's "Create first organization automatically" toggle flipped ON — no app code; the previously-prescribed webhook path is withdrawn)*
- [x] **Move signup form from `app.foundryims.com/signup` to `foundryims.com/signup`** per §3.2 *(done — `@clerk/astro` embedded `<SignUp />` with cookie→`unsafeMetadata` wiring; admin `/signup` retained as fallback)*
- [x] Build `foundryims.com/welcome` thank-you page per §18.3 *(done — GTM/GA4 + Meta + LinkedIn wired, env-gated; TikTok/Reddit not wired; noindex + sitemap exclusion live 2026-08-31)*
- [x] Configure Clerk vanity subdomain `accounts.foundryims.com` per §19 *(done — CNAME live)*
- [x] Set Clerk post-signup redirect to `https://foundryims.com/welcome` *(done via `forceRedirectUrl="/welcome/"` on the embedded component)*
- [x] Update marketing-site cookie capture script to use `Domain=.foundryims.com` per §18.2 *(done)*
- [x] Remove the URL-bridge link rewriting from `astro-foundryims/src/layouts/Layout.astro` *(done)*
- [x] ~~Add `user.created` org auto-provisioning per §3.9~~ *(withdrawn in v3.9 — `user.created` is mirror-only by design; org creation is Clerk-native per §3.3)*
- [ ] Migrate from current billing (whatever is in place) to Throttle per §23 — **Phase 1 shipped 2026-08-31** (api v3.69.0: schema + webhook receiver + `BillingEvent`; `resolve-organization.spec.ts` copied verbatim as instructed). Remaining: checkout/lifecycle, conversion, commissions — blocked on the plan/pricing decision, not on Throttle
- [x] **New (v3.7):** audit for `@Public()` on controller *classes* (§4) *(done 2026-08-31 — eliminated across 10 storefront controllers via `@SkipSessionAuth`, route-level-only enforcement in both guards, and a CI source-scan spec that fails the build on recurrence)*
- [x] **New (v3.7):** add a denial-asserting test for every guard (§4) *(done for the auth-critical set 2026-08-31: ClerkGuard, PermissionGuard, StorefrontAuthGuard, feature-toggle guard, plus the §4 role-mapping contract spec)*
- [x] **New (v3.7):** add the `BillingEvent` idempotency table per §23 *(done 2026-08-31, api v3.69.0 — first app to have one; Evident still lacks it)*
- [ ] **New (v3.8):** implement the plan entitlement gate per §24, with the §24.3 live-account audit run **before** activation and the §24.4 upgrade prompt shipped **first**
- [ ] **New (v3.8):** separate plan entitlement from tenant feature toggles per §24.1 if they currently share a field
- [ ] **New (v3.8):** build the default-deny tenancy layer per §25 *(supersedes the long-open `@epic/eslint-plugin-tenancy` item — the lint rule is now a secondary signal, not the control)*
- [ ] **New (v3.8):** give the worker/scheduler service a test harness and cover every access-revoking or money-moving job, **including its skip cases**, per §17.5
- [ ] Build backup runbook documenting 30-day expiry + deletion-rerun-on-restore per §15.2
- [ ] Set up 7-year audit-log cold-storage infrastructure per §14.5
- [x] Convert Stackbe → fully-on-Clerk *(done 2026-05-23; Stackbe fully removed)*
- [ ] PR-review enforcement of tenancy-scoped queries until `@epic/prisma-tenancy-lint` ships per §21.12 *(ongoing; 2026-07 tenancy audit fixed all Critical findings — Important/Minor and the raw-SQL slice remain)*
- [ ] **Schema audit deliverable:** produce a diff document comparing Foundry's `prisma/schema.prisma` against the baseline schemas in §3.5 (Org/User), §6 (Referral), §7 (PartnerSeat / PartnerSeatAssignment), §11 (Notification with `audience` + `deliveryClass`, NotificationPreference per-method), §12 (Webhook + WebhookDelivery with secret rotation fields), §13 (ApiKey with `expiresAt` + `scopes`), §14 (ActivityLog with `partnerSeatId` + standard `source` values), §14.5 (AuditLog with `partnerSeatId`), §15.2 (DataDeletionRequest, DataDeletionAudit). One doc, one PR, ratified *(the 2026-08-23 drift report is a working input, not the ratified deliverable)*
- [x] Drop `User.affiliateCode` column once per-org migration completes *(done — same migration as the per-org move)*
- [x] **New (v3.6):** reserve the billing schema per §23 — `Organization.trialEndsAt`, `Organization.status`, `Organization.throttleCustomerId`, `BillingEvent` table *(done 2026-08-31, api v3.69.0; existing orgs backfilled `status='active'`, nothing gates on it yet pending §24.3)*
- [ ] **New (v3.6):** decide + implement cookie consent per §18.4 *(nothing shipped; Ahrefs analytics currently loads unconditionally)*
- [ ] **New (v3.6):** verify account deletion scrubs `ActivityLog.userEmail` (and any other denormalized email snapshots) per §14/§15.2

---

# Part II — Auth & Identity

## 3. Auth & Organizations

### 3.1 Clerk dashboard config

Required settings in every Clerk project:

- **Sign-up enabled** — open self-serve signup. (Restrict via Clerk's allowlist if you need invite-only later.)
- **Magic link, Google, Apple** sign-in methods enabled.
- **Organizations enabled.**
- **Membership required** — users must belong to an organization (standard B2B mode).
- **Create first organization automatically: ON** *(v3.9 — inverted from `automatic_organization_creation: false`)* — Clerk creates the first org during sign-up using the naming rules below; the member never sees an org-creation form. Local rows are created by our `organization.created` / `organizationMembership.created` webhook handlers (§3.9), which mirror — they never provision.
- **Default naming rules** — personalize from member name (`user.first_name` → `"Kal's Organization"`), with a generic fallback. Org rename stays available in Settings.
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

**Amended in v3.9 — the mechanism is Clerk-native, not app code.** Clerk's **"Create first organization automatically"** toggle (with the §3.1 naming rules) creates the org *during* sign-up: the member never sees a naming form, and the org exists before any redirect. Our side does exactly two things:

- **Mirror, don't provision.** The `organization.created` and `organizationMembership.created` webhooks (§3.9) create the local `Organization` and `User` rows, run affiliate attribution from `unsafeMetadata.affiliateCode`, and apply the OWNER rule below. `user.created` never creates orgs.
- **First admin becomes OWNER.** Clerk has no OWNER concept — its creator role maps to `org:admin`. When a membership event would create a local user with the generic admin mapping and the org has **no** OWNER, that user is stored as OWNER instead. Without this rule, every Clerk-organic org has nobody holding `org.close`, and the sole-owner-leave guard points at no one.

Why not the previously-prescribed app-side `user.created` provisioning: with "Membership required" ON, Clerk's own signup flow drives org membership synchronously — an async webhook creating a second org *races* it. The platform feature is the entire implementation; the withdrawn admin first-load fallback is unnecessary for the same reason.

> **Foundry status (v3.9): implemented — this section now describes what ships.** Toggle flipped ON in the production instance 2026-08-31; mirror webhooks + first-admin→OWNER live since api v3.66.0. The v3.6 open question ("what creates the Clerk org for an organic signup?") is answered: previously nothing — Clerk's membership-required flow forced a manual org-creation screen; now the auto-create toggle.

### 3.4 Disposable email blocking

Every app blocks signups from known disposable email providers using the **`disposable-email-domains`** npm package (actively maintained, ~3.7k domains). When a blocked domain is detected:

- Reject the signup with a clear error message: **"We don't allow signups with temporary or disposable email addresses. Please use your real email."**
- Log the attempt to Sentry at `info` severity (per §17 — the disposable email block is a signal, not an error).
- **CSR override is per-domain**, not per-email — granting `mailinator.com` access opens it for everyone using that domain. UI lives in an admin-only Settings page (`/admin/disposable-overrides` or equivalent). Each override row carries: `domain`, `addedBy` (User.id), `reason`, `createdAt`, optional `expiresAt`. Override creation/removal writes an `auth.disposable_override` entry to the AuditLog (§14.5). This makes overrides explicit, time-bound by default, and auditable — not informal Slack favors.

> **Foundry status (v3.9): override mechanism shipped** (api v3.68.0): `DisposableEmailOverride` table, staff-only `GET/POST/DELETE /staff/disposable-overrides` (dual-gated: `users.manage` + the `FOUNDRY_STAFF_EMAILS` allowlist), audit events on add/remove, expiry honored at read time. Accepted equivalences: the UI lives as a tab on the staff settings page rather than a dedicated route, and attribution is the staff member's **email** rather than `User.id` (the staff gate is email-based). The blocklist check stays a synchronous in-process set; overrides refresh into it on boot, on a 60s timer, and after each mutation — note it's a plain unref'd interval, not a scheduled job, so it works where crons are disabled (Foundry prod runs `RUN_CRONS=false`).

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

🔴 **No per-request Clerk writes — of any kind** *(v3.9; replaces the withdrawn "push on login" pattern)*. The auth guard must never write to Clerk (or to hot local rows) on the request path. Foundry shipped the milder version of this — a per-request `lastLoginAt` update — and a single page load's ~20 concurrent requests serialized on that one row lock, producing 5–15-second page loads. Write-on-change is the whole sync story; the client tolerates briefly-stale `publicMetadata` because the API never trusts it anyway. If a divergence is noticed during permission re-validation, log at `warn` and reconcile asynchronously (local DB → Clerk), never inline.

### 3.7.1 Guarded role sync from membership events (v3.9)

Clerk's membership role is a **2-value projection** (`org:admin` / `org:member`) of an app's role enum, which may hold a dozen values. Mapping it straight onto `User.role` on `organizationMembership.created`/`.updated` destroys information — and it did: a user invited as WAREHOUSE was silently reset to MEMBER when they accepted (invisible in Foundry exactly as long as MEMBER was aliased to ADMIN; real drift the moment it wasn't). The rule:

- A membership event may move a user **only within the generic pair**: local MEMBER → ADMIN on `org:admin`, local ADMIN → MEMBER on `org:member`.
- **OWNER and every app-specific role are local decisions the event must never touch** (consistent with "local DB is canonical for role").
- **Creation path OWNER rule:** when the event *creates* a local user with the generic admin mapping and the org has no OWNER, store OWNER instead (§3.3).

> **Foundry status (v3.9):** implemented in api v3.66.0 (`guardedRoleSync` in the Clerk webhook service), with the invite-clobber bug it fixes covered by the §4 contract spec.

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
| `user.created` | **Mirror-only** *(v3.9 — provisioning withdrawn; Clerk auto-creates the org, §3.3)*. Run the disposable-email signal (§3.4) and any bookkeeping. Do **not** create orgs here — it races Clerk's own signup flow. Affiliate attribution runs on `organizationMembership.created` (below), where the org exists. |
| `user.deleted` | Tombstone the local `User` row(s) for this `clerkUserId` so they can't be recreated on next login attempt. Tombstone in the §15.2 sense — null PII, set `deletedAt`, keep the row for FK integrity. |
| `organization.deleted` | Cascade-close the local `Organization` row matching `clerkOrgId`. Same flow as `DELETE /orgs/me`. |
| `user.updated` | Propagate email/name changes to local `User` rows (across all orgs the user belongs to). Critical when the user changes their email in their account-tab UI. |
| `session.created` | Drives the `auth.new_device_login` audit event (§14.5) when the IP / user-agent doesn't match the user's recent sessions. |
| `organizationMembership.created` / `.deleted` | Sync local `User` rows with Clerk org membership (lazy-create on `created`, tombstone or remove on `deleted`). Defends against memberships changed via Clerk dashboard. **Role writes follow the §3.7.1 guarded sync** (generic ADMIN↔MEMBER pair only; first-admin→OWNER on create). On `created`, run affiliate attribution from `unsafeMetadata.affiliateCode` and clear the consumed code from Clerk. |

**Endpoint pattern:** `POST /webhooks/clerk` (per-app, public — secured by signature verification, not auth). Handler must be idempotent — Clerk retries on non-2xx, and webhook at-least-once delivery means a user might see the same `user.deleted` event twice.

**Failure handling (v3.9 — a taxonomy, because both extremes shipped and both were wrong):**

- **Unresolvable events** — unknown org, no matching rows, a payload referencing state we never had — are handled *inside* the handler: log-and-return, so the endpoint still 200s. Retrying these can never succeed; 5xx-ing them produces retry storms and a permanently red webhook dashboard.
- **Transient failures** — database down, provider API error — propagate out as **5xx so Clerk retries**. Handlers must be idempotent (at-least-once delivery). Log to Sentry at `error`.
- The banned third option is ack-and-log-everything: swallowing processing errors behind a 200 hides real failures behind healthy logs (§21 principle 16). Foundry ran this way for months.

> **Foundry status (v3.9):** compliant. Svix-verified handler covers 12 event types; `user.created` is mirror-only per the amended table; attribution runs on `organizationMembership.created` and clears the consumed code (now the standard); role writes use the §3.7.1 guarded sync; processing errors follow the v3.9 taxonomy (transient → 5xx as of api v3.67.0; unresolvable events log-and-return inside handlers). Remaining delta: `session.created` writes a plain `auth.login` audit event — no new-device detection yet.

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

**Stipulated (v3.6): every org is multi-user and multi-role.** An organization is never modeled as a single account — it holds any number of users, each with exactly one role from the shared enum below, invitable and removable at any time. Solo users are simply orgs of one. No app may assume one-user-per-org anywhere (queries, billing seats, UI copy).

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
- **MEMBER** — **standard operational access** *(v3.9 — redefined; the old near-empty baseline was unusable, so every app would have extended it divergently)*: the universal baseline (`settings.read`, `support.read`, `support.write`, `affiliate.read`, `activity.read`, `notifications.manage`) **plus full day-to-day read/write across the app's domain permissions** (in Foundry: products, variants, orders, POs, invoices-without-approve, shipments, stock, reports), and **zero org administration** — never `users.manage`, `apikeys.manage`, `webhooks.manage`, `apps.manage`, `org.manage`, `settings.write`, `audit.read`, `export.run`, `affiliate.manage`, or approval-tier permissions.
- **VIEWER** — `settings.read`, `support.read`, `affiliate.read`, `activity.read`, `notifications.manage`, plus read-only domain permissions. Strictly no writes — including `support.write`.

**Migration rule for apps that aliased MEMBER to ADMIN** (Foundry did, "for backward compatibility"): promote every existing MEMBER row to ADMIN in the same change that tightens the mapping. Nobody's access changes — the label becomes truthful — and new "Member" invites get the correctly-scoped role. Never silently strip permissions from live users.

**Pin the mapping with a contract test** that asserts the *denials* (MEMBER lacks `users.manage`, VIEWER lacks every write, unknown permission → false for every role). An allow-only test can't catch a role quietly aliased to a bigger set — which is precisely how MEMBER≡ADMIN survived unnoticed.

> **Note on VIEWER + `affiliate.read`:** Every org has a single affiliate code (§6). VIEWER can see their org's code and commission stats but cannot manage settings. This makes the affiliate program viewable to all org members regardless of role.

> **Foundry status (v3.9): compliant** (api v3.66.0+). MEMBER carries the operational mapping above (existing MEMBERs promoted to ADMIN per the migration rule); VIEWER lost `support.write`; the §4 contract is pinned by `permissions.spec.ts` with denial assertions; class-level `@Public()` is eliminated and CI-enforced (`public-decorator-hygiene.spec.ts`); denial tests exist for ClerkGuard, PermissionGuard, StorefrontAuthGuard, and the feature-toggle guard. The permission guard fails closed (no `@RequirePermission` ⇒ denied unless `@NoPermission()`) — adopt portfolio-wide. Remaining minor delta: `notifications.manage` is granted but not yet checked by an admin surface (§11's schema migration comes first).

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

#### Guard hazards — both fail open and fail silently (v3.7)

Two ways to end up with a guard that returns "allow" forever while looking correctly wired. Neither throws, neither logs, and both have shipped to production.

**1. `@Public()` on a controller *class* disables every other guard on it.**

Guards conventionally start `if (isPublic) return true`. Put `@Public()` on a class and it short-circuits **all** of them — `@RequirePermission()`, `@RequirePartner()`, tenancy — for every route on that controller. Identity then comes from whatever headers the caller sends, which makes it spoofable.

- `@Public()` is **route-level only**. Never class-level. Make the guards enforce it: read the public flag from the route handler *only*, so a class-level `@Public()` doesn't fail open — it fails **closed** (every route 401s), which is discoverable in the first minute.
- A controller with genuinely public routes marks those routes, not the class.
- A controller whose auth is its **own guard stack** (e.g. storefront-key routes) uses a narrow opt-out decorator (`@SkipSessionAuth()`-style) that is honored by *named* guards only — and it MUST ship paired with the replacement auth guard on the same class.
- **(v3.9) "Audit for it" is not the control — a CI spec is.** Ship a source-scan test that fails the build on (a) any class-level `@Public()` and (b) any `@SkipSessionAuth()` controller missing its replacement guard. The one-time grep is how the hazard shipped in the first place; Foundry's `public-decorator-hygiene.spec.ts` is the reference. This is §25's default-deny philosophy applied to guards: wherever this document says "audit for X," prefer "a test fails on X."

**2. Guard registration order and location decide whether a guard runs at all.**

With Nest-style `APP_GUARD` providers, **guards registered in the root module run before guards registered in imported modules.** A guard that must see the result of an earlier one has to be registered where that ordering holds — move its registration to a different module and it still instantiates, still appears in the DI graph, and never blocks anything.

The same applies to guards that must sit alongside a specific peer: a tenancy guard for API keys must be registered next to the JWT guard, or it silently no-ops on every request.

- Registration location is **load-bearing**; comment it at the registration site with *why*.
- Every guard needs at least one test that asserts a request is **denied**. A guard tested only on the allow path is indistinguishable from a guard that does nothing.

> This is the §21 principle 16 failure mode in its purest form: the system reports success while enforcing nothing.

### Invite flow

`POST /users/invite` with `{ email, role }`. Calls `clerk.organizations.createOrganizationInvitation`. Local User row created with `isPending: true` until they accept.

Settings → Users page lists current members + pending invites with role dropdown and remove button.

---

## 5. Account Types

Every Clerk user has `accountType` in `publicMetadata` (mirrored to `User.accountType` locally). Default `client`.

| Type | Who | What they see |
|------|-----|---------------|
| `client` | End users running the app for their business | Standard app UI. Can share their org's affiliate link. |
| `partner` ✅ | Agencies, consultants, resellers | Adds Partner sidebar section: Referrals tab, "Create Trial for Client" button, partner profile, payout settings, partner seat assignments. |

Both types log in identically. Partner is a role, not a separate auth system.

### Org entitlement — the client-side axis (v3.7)

`accountType` answers "is this **user** a partner?" It does not answer "how is this **org** compensated?" — and every commission and pricing decision in §6/§7 turns on the second question. Apps need both axes.

```prisma
enum AccountEntitlement {
  NONE        // organic signup; no referral relationship
  AFFILIATE   // arrived via an affiliate link; §6 first-month bounty applies once
  REFERRAL    // partner-originated, client pays their own way; partner earns 10% of renewals (Path B)
  AGENCY      // partner-billed white-label; invoiced at 80% of list, NO commission accrues (Path A)
}

model Organization {
  entitlement AccountEntitlement @default(NONE)
}
```

Why this has to be an org column rather than derived on the fly:

- **It is what the pricing call reads.** The 20% discount is applied by looking at the org's entitlement, not by joining back through partner seats at invoice time.
- **It is what suppresses commission.** `AGENCY` earning no commission is the "one client, one model" rule (§7) expressed as data. Deriving it from the presence of a partner seat gets this wrong — a client can hold a partner seat *and* pay their own bill, which is exactly Path B.
- **It survives the relationship.** A partner seat can be revoked by either side while the billing arrangement continues; the two must be able to disagree.

**The transitions are the partner election (§7):** `REFERRAL → AGENCY` on partner-checkout **activation** (never at checkout start — Path A rule 1), and `AGENCY → REFERRAL` on handoff. Nothing else may write this column.

---

## 16. Session Permission Re-Validation ✅

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

**Amended in v3.9.** When the API returns a 401 with code `permission_changed`, the client must stop operating on stale permissions — that's the requirement. The old prescription ("redirect to `/login`") is wrong under Clerk: the session is still valid, so /login bounces straight back in. Instead:

- Flag the event (e.g. sessionStorage) and **hard-reload** so every rendered surface re-derives under the new role.
- On boot, surface the flag as a notice: "Your permissions have changed."
- Never silently retry the failed request — it would succeed under the new role while the whole rendered UI still reflects the old one.

**Server-side shape:** track the last role served per `(user, org)`; on change, record the *new* role first, then fail exactly once with the coded 401 — the retry proceeds under the new permissions. A per-process map is acceptable: each instance fails once independently, and detection latency is bounded by the auth-cache TTL.

> **Foundry status (v3.9): implemented** (api v3.67.0 + admin). Role re-read behind the 30-second single-flight cache; `PermissionChangedException` carries the machine-readable code; admin reloads + toasts. Status flips 🚧 → ✅.

---

# Part III — Affiliate Program

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
      100% of the referred org's      10% of every renewal,
      FIRST MONTH, one time,          for the life of the
      capped at $500 per referral,    subscription. Nothing on
      held 30 days.                   the first payment.
                                      Plus: partner seat in client org
```

**The two tiers pay differently, and deliberately so** (v3.7 — see "Commission terms" below). An affiliate makes an introduction and is done: they get a large one-time bounty. A partner carries the client relationship indefinitely: they get a smaller slice of every renewal. **The differentiator is partner status** — partners get the dashboard, the "Create Trial" button, and the partner seat in client orgs. Partner referrals require the partner to physically create the trial (action is proof of attribution); affiliate referrals attribute via shared link.

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

1. **One affiliate code per organization — and every user has one by default.** The affiliate program is not opt-in, gated, or applied-for: every org gets a code (lazy-generated on first view), and every user in every role holds `affiliate.read`, so **every signed-in user of every Epic app can open Settings → Affiliate and copy a working share link from day one.** Multiple users in the same org share one code; commissions accrue to the org, not to individual users. (See trade-offs above.)

2. **Last-touch attribution wins.** If a prospect clicks two different affiliate links in the 30-day cookie window, the most recent code overwrites the earlier one. This is a deliberate trade-off: simpler than first-touch, aligned with industry standard, but unfair to top-of-funnel educators who may lose credit to bottom-of-funnel coupon sites. Paying the affiliate tier a full first month is intended to make even a single successful introduction worth the effort.

3. **Self-referral is rejected by membership match against the referrer's org.** If the signing-up user's email matches **any active member** of the referrer's org, attribution is blocked. With per-org codes (see rule 1), the entire org shares the link — so a teammate signing up via their own org's link IS the org self-referring. Returns `{ attributed: false, reason: "self_referral" }`. (Earlier drafts checked only the referrer's own email — that was incoherent with the per-org primitive; a teammate signup with a different email would have passed and the org would have credited itself.)

4. **Code regeneration: snapshots are immutable.** A user with `affiliate.manage` can regenerate their org's `affiliateCode`. Existing `Referral` rows keep the old code (we snapshot `affiliateCode` on the Referral). The old code stops working for *new* attributions.

### Commission terms (v3.7 — stipulated)

**Affiliate referrals earn 100% of the referred org's first month, one time, capped at $500 per referral, held 30 days from the qualifying payment.**

Every clause is load-bearing:

| Clause | Why |
|---|---|
| **100% of the first month** | A single successful introduction is worth a real amount of money, which is what makes a non-partner bother to share the link at all. A trickle of 10% on a $49 plan never motivated anyone. |
| **One time — first payment only** | Affiliates make an introduction and are done. Ongoing revenue share is the *partner* tier's compensation for carrying an ongoing client relationship (§7). Paying both would pay twice for one act. |
| **Capped at $500 per referral** | Portfolio apps price very differently. On a $49/mo app the cap never binds; on an app with four-figure monthly plans, an uncapped first month is a payout nobody signed off on. The cap is what lets one commission rule cover every app in the portfolio. |
| **Per referral** | The cap applies to each referred org independently. An affiliate who refers ten orgs can earn ten capped commissions; there is no per-referrer or per-period ceiling. Productive affiliates are never penalised for volume. |
| **30-day hold** | The commission is calculated at the qualifying payment but only becomes payable 30 days later, so refunds and chargebacks land first. Partner commissions have no hold — they accrue on renewals, which are already proven payments. |

**Worked examples:**

```
App bills $49/mo   → first month $49    → commission $49    (cap does not bind)
App bills $499/mo  → first month $499   → commission $499   (cap does not bind)
App bills $2,000/mo → first month $2,000 → commission $500  (capped)
```

**Implementation requirements:**

- Rate, cap, and hold live in **one constants module** read by both the code that *creates* commissions (the billing webhook) and the code that *reports* them (the partner/affiliate dashboard). A rate that can drift between what you pay and what you tell people is a support incident waiting to happen. Reference: `apps/api/src/modules/billing/commission.constants.ts` in Evident.
- The cap is applied at commission *creation*, and the uncapped basis is stored alongside it so the dashboard can show "capped from $X" rather than an unexplained number.
- **Discounted subscriptions commission on the amount actually invoiced**, not list price. A white-label agency org invoiced at 80% of list (§7 Path A) earns no affiliate commission at all — see §7's "one client, one model" rule.

> **Partner rate is different and is specified in §7:** 10% of every renewal, for the life of the subscription, **no cap**, nothing on the first payment. The asymmetry is intentional — see §7 "Reward tier and commission timing."

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

> **Foundry status (v3.6):** implemented to spec — per-org code, lazy 8-char generation, snapshot-on-regenerate, any-active-member self-referral check, SetNull/Cascade FK asymmetry, all four endpoints, Settings → Affiliate page with clean vocabulary. One flow delta: attribution fires from the `organizationMembership.created` webhook (not `user.created` — see §3.9), with the admin's sessionStorage + `AffiliateAttribution` post-auth callback as fallback.

---

# Part IV — Referrals & Partner Program

## 7. Partner Program ✅ (accrual + reporting live; payout rails ⏸️)

The partner program is a fundamentally different model from affiliates. **Partners earn commission by *creating* the trial directly.** This eliminates attribution disputes that plague most B2B SaaS partner programs.

### Core principle: action is proof of attribution

If you want partner-tier credit, **you must be the one that physically sets up the trial.** There's a database row showing you created the org. No forms, no claims to validate, no disputes.

A user who promotes the product via affiliate link still gets affiliate credit. Partner status unlocks the partner dashboard, the partner seat (continued access to client orgs), and the trial-creation flow.

**Affiliate and partner referrals are compensated differently** (v3.7): an affiliate takes 100% of the referred org's first month, one time, capped at $500 (§6); a partner takes **10% of every renewal for the life of the subscription, no cap**, and nothing on the first payment — that one is the partner's own conversion. The difference tracks the *kind* of relationship — affiliates are link-sharers making a one-off introduction; partners are integrators carrying an ongoing client relationship. **Partners additionally get an election per client** (v3.6): keep paying the client's bill white-label at a **20% discount netted at source**, or spin ownership off to the client and earn the **10% recurring commission** — see "Two ownership paths" below.

### Becoming a partner

**Stipulated: any agency, consultant, or reseller can apply to become a partner** — partnership is applied-for and reviewed, never invite-only or ad-hoc. Two entry points, same application:

1. User clicks "Become a Partner" on the marketing site (`/partners` → `/partners/apply`) **or applies in-app** (Settings → Partner Program; any signed-in user).
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
- **Partner referral credit is independent of partner seat status.** If Agency A referred Client X but the client later replaces them with Agency B, Agency A *still* receives the referral commission. This is intentional — the original partner did the originating work. The deliberately modest 10% commission rate plus the "action is proof" requirement (you must physically create the trial) makes this self-balancing: bad actors can't easily farm signups because each one requires real client engagement to convert.
- **Partner orgs decide which of their team members access which client seats** via `PartnerSeatAssignment`. The partner org's Partner Dashboard manages this.

**Assignment enforcement semantics (v3.9 — previously unspecified):** the partner org carries an **`autoAssignAll`** default (on `PartnerProfile`, default **true**).

- `autoAssignAll: true` — every partner-org member may act on every client seat; `PartnerSeatAssignment` rows are *advisory* (they record intent and drive the dashboard, nothing blocks).
- `autoAssignAll: false` — the assignment rows ARE the access list; an unassigned member is denied at org-resolution time when acting through the seat.
- Default true preserves behavior for orgs that predate the field; flipping to false is the org's explicit opt-in to enforcement.
- Assignment add/remove is idempotent and audit-logged **on the client org** (`partner_seat.assignment_added` / `.assignment_removed`) — the client's audit trail must show who could reach their account.

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

### Two ownership paths after conversion — the partner's election

When a partner-created trial converts to paid, the **partner elects one of two models per client**. This election is the core partner economic stipulation (v3.6):

- **Path A — white-label:** the partner keeps ownership and pays the client's bill at a **20% discount netted at source**.
- **Path B — spin-off:** the partner transfers ownership to the client, the client pays full list, and the partner earns a **10% recurring commission**.

The election is per-client, and switching from A to B is always available (promotion flow below). The economics are deliberately asymmetric: white-label carries the partner's own billing risk and support burden, so it earns the deeper margin.

#### Path A: White-label (partner stays owner, 20% discount)

The partner pays the subscription fee on the client's behalf as part of an all-inclusive service offering. The partner remains OWNER of the org indefinitely.

- Partner stays as `OWNER` of the client org.
- Partner's payment method is on file in Throttle.
- **Partner receives a 20% white-label discount on the bill** — netted at source, not paid out as commission. Throttle invoices the partner at 80% of list price for white-label-mode subscriptions. *(v3.6: raised from the 10% set in v3.1 to make the white-label margin meaningfully better than the commission path.)*
- No 10% commission accrues on a white-label org — the discount **is** the partner economics for that client. One client, one of the two models, never both.
- The client may not have visibility into the bill (this is a partner choice).

> **Legal status:** Whether this is a "discount" (treated as net revenue) or a "commission" (gross revenue minus a 1099 expense) for accounting and tax purposes is **pending legal review**. The economics match either way; the line items on financial statements and 1099 forms differ. Path A specifics may evolve once finance/legal sign off. Implementations should treat the discount-at-source mechanism as the default direction but expect refinement.

##### Pay-for-client checkout — required mechanics (v3.7)

Putting a partner's card behind a client's subscription is the single most trap-dense flow in this document. Every rule below was written after the corresponding failure was found in production. **Implement all five.**

**1. Record the billing intent; do not flip entitlement until the subscription activates.**

Starting a checkout is not paying. If the client org is marked as partner-billed when the checkout *opens*, an abandoned checkout leaves it marked that way permanently — nobody is actually paying, and because partner-billed orgs accrue no commission (see the one-model rule above), the partner's 10% is silently suppressed on a client they never took over. Write the intent to a nullable pointer on the subscription; let the **activation webhook** promote the entitlement. That is the moment the decision becomes real.

**2. Roll the intent back if the checkout call fails.**

The intent is written before the outbound provider call, so a provider failure strands it. It is not inert: the activation webhook will later read it and promote an org to partner-billed even if the *client* subscribes on their own card. Wrap the provider call and clear the pointer on failure.

**3. Route a takeover through change-plan, never through create-checkout.**

A partner frequently takes over a client who already converted on their own. `createCheckout` on a live subscription issues a **second recurring subscription** — the client is now billed twice, and the two rows fight over the same org. Move the existing subscription in place, and fall back to checkout only when there is genuinely nothing to move.

**4. Pass the discount flag explicitly — do not derive it from entitlement.**

By rule 1, entitlement has *not* flipped yet at the moment the checkout is priced. Code that reads entitlement to decide whether to apply the 20% will price the very first partner invoice at full list, and the org can never reach the discounted state. The pricing call takes the flag as an argument.

**5. Require explicit confirmation of the trade-off.**

Converting a client to partner-billed forfeits the partner's 10% lifetime commission on that client. That is a commercial decision, not a formality. The API must reject the request unless it carries an explicit opt-in flag, and the UI must state the forfeit in words before sending it. Reference wording: *"Take over this client's subscription at the 20% partner rate. This forfeits your 10% lifetime commission on this client."*

> **Reference implementation:** `initiateAgencyCheckout` in `apps/api/src/modules/partner/partner.service.ts` (Evident) implements all five, with the reasoning in comments.

#### Path B: Promote client to owner (spin-off, 10% commission)

The partner promotes the client user to OWNER. Partner becomes a `PartnerSeat` and continues to earn commission.

- Partner navigates to the client org and clicks "Promote Client to Owner."
- Dialog confirms: "This will transfer ownership of `<Client Org>` to `<client@email.com>`. You will retain a partner seat with `<role>` permissions."
- On confirm:
  - Clerk org membership: client user promoted to `org:admin`, partner user demoted.
  - Local: client user `role` set to `OWNER`. Partner user removed from local `User` table for that org (they retain access via the existing `PartnerSeat` row, created at trial creation).
  - `Referral.status` will transition `active` → `converted` when the client's first invoice is paid.
- Partner continues to earn 10% commission on client's bill (paid out as commission since the client now pays the bill, not as discount).

##### Handoff — required mechanics (v3.7)

A handoff out of a **white-label** arrangement is not just an ownership change: it moves the org between two billing models mid-subscription. Six requirements:

**1. Clear the billing intent, not just the entitlement.** Set the org back to client-paid *and* null the partner-billing pointer on the subscription. Leaving the pointer behind means a later re-activation silently promotes the org straight back to partner-billed, undoing the handoff without anyone touching it.

**2. 🔴 The payment method does not move — say so, in the product.** This is the one that surprises people. After handoff, the billing provider's customer record still holds **whatever card the partner vaulted at partner checkout**, so the next renewal charges the *partner* — or fails. Nothing in an ownership transfer moves a payment method, and where the card wallet is a browser-side component (it is, in Throttle) the server cannot even read whether a card is present, let alone re-assign one.

  Required: persist a `cardHandoffPendingAt` timestamp on the subscription at handoff, surface a persistent banner to the new owner until they add their own card, and expose an acknowledge endpoint that clears it. **A handoff that silently leaves the partner's card on file is a billing incident with a delay fuse.**

**3. Re-price outside the database transaction.** Dropping the 20% discount is an outbound provider call. Holding a DB transaction open across it is a long-lived lock for no benefit. Do the ownership transfer transactionally, then re-price **best-effort** — a failed re-price leaves the client on the partner rate, which is visible on the dashboard and correctable with a plan change. A failed *ownership transfer* is not recoverable, so that is the part that must be atomic.

**4. Create the referral if none exists.** An org can reach partner-billed status by paths other than a partner-created trial (manual link, support action, migration). Handing such an org off with no `Referral` row means the 10% has nothing to attach to and the partner earns nothing forever. Create one at handoff, status `converted`.

**5. Push the role change to Clerk metadata, fire-and-forget.** The new owner's client-side `publicMetadata` is stale until something writes it (§3.7). The API re-validates against the local DB regardless, so this is a UI-correctness write, not a security one — never block the handoff on it.

**6. Return which kind of handoff happened.** The UI needs to know whether this was a white-label exit (warn about the card and the rate change at renewal) or an ordinary ownership transfer (no billing consequences at all). Two very different confirmation screens.

> **Reference implementation:** `handoffOwnerToClient` in `apps/api/src/modules/partner/partner.service.ts` (Evident), plus `POST /billing/card-handoff/acknowledge`.

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

The portfolio has exactly **three** compensation shapes. Every referred org is on exactly one of them at any time.

| Tier | Rate | Cap | Timing |
|---|---|---|---|
| **Affiliate** (`referralType: affiliate`) | 100% of the referred org's **first month**, one time | **$500 per referral** | Accrues at the qualifying payment; **payable 30 days later** (refund/chargeback window). §6. |
| **Partner — Path B, client-paid** (`referralType: direct`) | **10% of every renewal**, for the life of the subscription. **Nothing on the first payment.** | **No cap, no sunset** | Accrues on each renewal payment; no hold. Month N events pay in month N+1. |
| **Partner — Path A, white-label** | **20% discount netted at source** — the partner is invoiced at 80% of list | n/a — never paid out | Applied on every partner invoice for as long as the arrangement lasts. |

**No org ever earns on two tiers at once.** A white-label (Path A) org accrues *no* commission — the discount is the partner economics for that client. Electing Path B gives up the discount and starts the 10%; electing Path A gives up the 10% and starts the discount. One client, one model, never both. The election is per client and reversible (§7 "Two ownership paths").

**Why the first payment is excluded from the partner 10%:** the partner set the trial up themselves. The conversion is their own work on their own client, not a renewal they're being retained to protect. Paying commission on it would be paying the partner to close a deal they were already closing.

**Why the partner tier has no cap or sunset, while the affiliate tier has both:** they compensate different things. The affiliate bounty is priced for a single act and capped so one commission rule can span apps with wildly different price points. The partner 10% is priced for an ongoing relationship — a 24/36-month sunset would lower lifetime cost but adds cap tracking, sunset notifications, and partner disputes near expiry, and it weakens exactly the long-term commitment the tier exists to buy. We accept that a partner who originated a client five years ago still earns 10% on that client's bill.

**Commission basis is the amount actually invoiced, not list price.** Discounts, proration, and credits all flow through to the basis. Reporting surfaces must show the basis alongside the commission so a partner can reconcile a number that moved.

> **Reporting vs. paying are separate problems.** A conforming implementation must *calculate and report* the commission basis per client, per period. Actually moving money (payout method, tax forms, 1099s, mass-pay batching) is a further step and may legitimately not exist yet — Evident reports the basis and has no payout desk at all, deliberately. Do not let the absence of a payout rail block shipping the accrual and reporting, and do not imply to partners that a reported commission has been paid.

### API surface

> **Foundry status (v3.9):** the full non-billing surface is live (api v3.67.0+) — everything from the v3.6 list **plus** `GET/PUT /partner/me/profile` (payout settings; payout-email changes audit-logged as `partner.payout_email_changed`), `GET /partner/me/team`, `PUT /partner/me/team/defaults` (`autoAssignAll`), and `POST/DELETE /partner/seats/:id/assignments[/:userId]`, with Team + Settings tabs on the partner dashboard. Payout fields are reporting targets only (payout rails unbuilt portfolio-wide — v3.7 rule). Assignment *enforcement* when `autoAssignAll=false` is not yet wired at access time — deferred to the billing-phase access work; the data model, management API, and audit trail are what ship today. Still ⏸️ billing: commissions endpoints (Evident has them). The marketing site's `/partners/apply` routes into signup + the in-app application (no standalone form).

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

# Part V — Billing (Throttle)

## 8. Trials ✅ (live in Evident; ⏸️ elsewhere pending adoption)

### Public signup ✅ (mechanic exists, no time-bound trial)

Today: anyone hitting `/signup` creates a Clerk user + org instantly. Org has no expiration, no billing state. Effectively a permanent free tier.

### Trial lifecycle

- New orgs get `trialEndsAt: NOW + 14 days` (default; configurable per app).
- A scheduled job flips `status` to `expired` past the deadline — **subject to the guard below, which is not optional.**
- Affiliate `Referral.status` transitions `active` → `expired` when a trial expires without converting.

> #### 🔴 The trial-expiry sweep must never expire a paying customer
>
> The obvious implementation — match `status = TRIALING AND trialEndsAt < now`, flip to `EXPIRED` — **is wrong, and it locks out paying customers.** Throttle can leave `status: trialing` on a subscription it has already charged. Apps mirror provider status verbatim, so a stale provider status becomes a local `TRIALING` row with a past `trialEndsAt` on an account that is paid up. The sweep then expires it, and the plan gate locks the customer out of an account they are paying for. Evident caught this one day before it fired on a live account.
>
> **Required in every implementation:**
>
> 1. **Never expire a subscription paid into the future.** The match must also require `currentPeriodEnd IS NULL OR currentPeriodEnd <= now`. A trial end date alone is not evidence that nothing has been paid.
> 2. **Log the skips loudly.** Query the paid-but-`trialing` rows separately and emit a `warn` naming each org and its paid-through date. Skipping them quietly hides a provider bug behind a healthy-looking log line — see §21 principle 16.
> 3. **Cover it.** A billing cron that can revoke access needs a test, and background workers are exactly where test harnesses tend not to exist. Evident's does not have one; this cron is covered by typecheck and production dry runs only, which is not good enough for the blast radius.
>
> Reference: `apps/worker/src/schedulers/trial-expiry.scheduler.ts` (Evident).

> **Extending a trial delays the first charge.** Trial-remaining calculations read `Subscription.trialEndsAt`, and a future date is handed to the provider at checkout as free days — so "give them another week" also means "don't charge them for another week," on the very checkout you are trying to complete. If a conversion must bill same-day, null the column first (null grants zero days). Nulling is also the only way to *remove* a deadline: a null `trialEndsAt` can never be matched by the expiry sweep, which grants indefinite access until someone converts the account. That is deliberate lockout insurance, not a fix — track anything you park that way.

### Required subscription lifecycle operations (v3.6)

**Stipulated: every app, through Throttle, must support the full subscription lifecycle** — these are the operations the billing integration exists to provide, and every one needs both an API path and a UI surface (§20 screens list):

| Operation | Who initiates | Canonical path | Notes |
|---|---|---|---|
| **Start trial** | Signup (self-serve) or partner | `POST /partner/trials` | Sets `trialEndsAt`; org `status: "trial"`. |
| **Convert trial → paid** | Client (or partner on white-label) adds payment method + picks plan | `POST /billing/checkout` | Fires `Referral` conversion (§7); org `status: "active"`. |
| **Upgrade plan** | OWNER (or white-label partner) | `POST /billing/change-plan` | Prorated; effective immediately. **Never `checkout` on a live subscription** — that issues a second one. |
| **Downgrade plan** | OWNER (or white-label partner) | `POST /billing/change-plan` | Takes effect at next renewal; feature gates adjust then. |
| **Cancel** | OWNER (or white-label partner) | `POST /billing/cancel` | Runs to end of paid period, then org `status: "suspended"`; data retained per §15. |
| **Reactivate** | OWNER | `POST /billing/change-plan` | From suspended back to active without data loss. ⚠️ **Commonly missed** — Evident has no dedicated reactivate path. |
| **Switch billing model** | Partner (white-label ⇄ client-paid) | `POST /partner/trials/:id/agency-checkout` · `POST /partner/trials/:id/handoff-owner` | The §7 election. Both directions have required mechanics — Path A rules 1–5, handoff rules 1–6. |
| **Recover from a dead pointer** | System | `POST /billing/sync` | A stored `externalSubscriptionId` that 404s (sandbox id in a live environment) must clear itself and fall through to a fresh checkout, not 500. |

Payment-failure dunning rides on `billing.payment_failed` (§11, transactional) rather than being a lifecycle state of its own.

### Partner-created trial ✅ (mechanic live; time-bound lifecycle ⏸️)

Same lifecycle, but `Referral.referralType = "direct"`. Partner sees the trial countdown in their Referrals tab. *(Foundry status: `POST /partner/trials` + abandoned-trial cleanup are live; the countdown itself is ⏸️ until `trialEndsAt` exists.)*

### Conversion detection ✅

When Throttle reports a paid subscription:
1. Find the `Referral` row by `referredOrgId`.
2. Set `convertedAt = NOW`, `status = "converted"`.
3. Calculate reward (10% of MRR).
4. Set `rewardStatus = "pending"` until payout runs (next month).

> **The integration contract — environments, auth, event vocabulary, org resolution, idempotency — is §23.** Read it before writing a webhook handler.

---

## 23. Billing & Throttle Integration ✅ (live — reference implementation: Evident)

> **Status as of v3.7: Throttle billing is BUILT AND LIVE.** This section is no longer a stub. Evident has run the production Throttle key since 2026-08-08 on the **Stripe** connector, with real invoices settled, and implements the full lifecycle in §8. The v3.2–v3.6 instruction to "not depend on Throttle for any blocking design decision" is **withdrawn** — but note that the design-phase guesses in those revisions were wrong in specific, expensive ways, corrected below. If you built to the old stub, re-read "Event vocabulary" and "Resolving the org" before shipping.
>
> **Reference implementation is Evident, not Foundry,** for this section only. Foundry has not integrated billing. `apps/api/src/modules/billing/` and `apps/api/src/modules/throttle/`.

> **Naming flag (carried from v3.6, still unresolved):** "Throttle" names two different things — the billing platform this section describes (live, Evident bills on it) and a separate shipped THROTTLE **sales-channel** platform that Foundry integrates with as a channel (Foundry serves catalog; Throttle owns checkout/orders — `ChannelPlatform.THROTTLE`). Nothing in Foundry's channel integration is billing. This still needs a rename or an explicit two-role definition.

### What Throttle is (and isn't)

Throttle is Epic's billing platform — a Stripe-style layer that issues invoices, processes subscriptions, and emits billing events. **It is not a system of record for users or organizations** — Clerk is, and stays so. Throttle's customer records are a downstream subscription view of the same orgs that already exist in Clerk + the local DB.

### Environment and auth — the facts that cost time

Every one of these has produced a wrong conclusion in practice. None are guessable.

| Fact | Detail |
|---|---|
| **The API key selects the environment, not the host** | Sandbox and production share one host. There is no `sandbox.` prefix to check. The *only* way to know which environment you are talking to is which key you sent. |
| **Nothing reads `*_LIVE_*` variable names** | Code reads `THROTTLE_API_KEY` / `THROTTLE_WEBHOOK_SECRET`. `_LIVE_` prefixes are labels for humans pasting values into the deploy dashboard — a var named `THROTTLE_LIVE_API_KEY` is read by nothing. Scripts that source a local env file hit **sandbox** unless handed the live key explicitly. |
| **Auth header is `x-api-key`, not `Authorization: Bearer`** | A Bearer request returns **200 with an empty body** — which reads exactly like a successful query that found nothing. This is the single most misleading failure mode in the API. |
| **Base path is `/api/v1`, not `/v1`** | — |
| **Some write endpoints are `PATCH`, not `PUT`** | And the generated client omits request bodies from certain method signatures, so raw `fetch` is sometimes the only option. Verify against the live endpoint rather than the SDK types. |
| **Cheap environment probe** | `GET /api/v1/subscriptions/<id>` with each candidate key — live rows 404 under a sandbox key. |

**Required env vars:** `THROTTLE_API_KEY`, `THROTTLE_WEBHOOK_SECRET`, `THROTTLE_APPLICATION_ID` (the application id is the same across environments).

### Customer-to-org mapping

The intended invariant is **one Throttle customer per Clerk Organization**, with the local `Organization` row carrying the pointer.

- **Throttle has no concept of users.** Subscription state belongs to the org.
- **Direction of trust:** the local DB is canonical for org existence; Throttle is canonical for subscription state.
- **No customer record exists for users who aren't in any org.**

> #### 🔴 The invariant does not hold under partner billing — design for that
>
> When a partner pays for multiple clients (§7 Path A), **every org that partner created shares ONE Throttle customer** — the partner's. The consequences are not cosmetic:
>
> - **`customer.externalId` can never identify the client org** on a partner-billed subscription. It identifies the *payer*. Any handler that maps customer → org will attribute all of a partner's clients to whichever org it resolves first.
> - **Resolve the client through the checkout session or the subscription**, never through the customer, on any code path that can be partner-billed.
> - **Invoice queries scoped by customer leak across clients.** Listing "this client's invoices" by customer id returns every client that partner pays for. This is a live, unfixed defect in Evident; treat it as a known trap, not a solved problem.
>
> Do not write code that assumes customer↔org is 1:1. It is 1:1 for self-serve orgs and 1:many for partner-billed ones.

### Resolving the org from a webhook — required

> **This is where the design-phase spec was most wrong, and it failed silently.** An early implementation read `externalCustomerId` off the subscription payload. Subscription webhooks **do not carry that field** — so every subscription event was dropped while the endpoint returned `200 OK`. Healthy logs, healthy dashboards, nothing recorded, for weeks.

Two field names look interchangeable and are not:

- **`externalId`** — the id *we* set when creating the customer (our `organizationId`). This is the one you want.
- **`externalCustomerId`** — a per-connection mapping, **null for direct API use**. Not a substitute.

Implement org resolution as an ordered fallback chain, and **log loudly and skip when every step misses** — never return a bare 200 on an unresolvable event:

1. `data.customer.externalId` — the sibling customer object. *(Deliberately not `customer.externalCustomerId`.)*
2. `externalCustomerId` on the subscription itself, if present.
3. The local `Subscription` row, by `externalSubscriptionId`.
4. Resolve `customerId → externalId` via an API call.

Reference: `resolveOrganizationId` in `apps/api/src/modules/billing/throttle-webhook.controller.ts`, with `resolve-organization.spec.ts` as the contract test. **Copy the tests, not just the code** — this is the highest-value test file in the billing integration.

### Event vocabulary (verified against live deliveries)

The v3.6 stub guessed at this and got it wrong. There is **no `invoice.paid`**, **no `invoice.payment_failed`**, and **no `trial.ending` / `trial.expired`** — trial expiry is something the app schedules for itself (§8), not something Throttle announces.

| Event | Maps to |
|---|---|
| `subscription.created` | `TRIALING` |
| `subscription.activated` | `ACTIVE` |
| `subscription.renewed` | `ACTIVE` — **this is the partner-commission trigger** |
| `subscription.resumed` | `ACTIVE` |
| `subscription.updated` | `ACTIVE` |
| `subscription.plan_changed` | `ACTIVE` |
| `subscription.paused` | `PAST_DUE` |
| `subscription.past_due` | `PAST_DUE` |
| `subscription.payment_failed` | `PAST_DUE` |
| `subscription.cancelled` | `CANCELLED` — **spelled with two Ls** |
| `payment.captured` | `ACTIVE` |
| `payment.failed` | `PAST_DUE` |
| `cart.abandoned` | recovery flows; carries `customer.externalId`. **`cart.expired` is retired** — endpoint registration rejects it (returned in `retiredEventsIgnored`, verified 2026-08-31) |

**Signature verification:** `verifyWebhookSignature` from `@usethrottle/webhook-types`, with a **5-minute replay window**. Verify before parsing; never trust the body. ⚠️ That package is **ESM-only** and cannot be `require`d from a CJS runtime (NestJS default) — keep it as a types-only dependency and implement the check locally: HMAC-SHA256 of `` `${t}.${rawBody}` `` against the `v1=` value from `X-Throttle-Signature: t=<unix>,v1=<hex>` (Foundry's `throttle-signature.ts`, with test vectors).

### Idempotency — `BillingEvent` (required)

Webhook delivery is at-least-once. Persist every inbound event keyed on the provider event id **before** processing, and no-op on a duplicate:

```prisma
model BillingEvent {
  id          String   @id @default(uuid())
  externalId  String   @unique          // provider event id — the idempotency key
  type        String
  orgId       String?                   // resolved from payload; null when unresolvable
  payload     Json
  processedAt DateTime?
  receivedAt  DateTime @default(now())

  @@index([orgId, type])
}
```

Beyond replay safety this is the audit trail for "why did this org's status change" — worth having before the first billing dispute rather than after.

> **Evident divergence (open work, not a pattern to copy):** Evident has **no `BillingEvent` table** and **no `Organization.throttleCustomerId`**. It keeps billing state on `Subscription` (`externalSubscriptionId`, `agencyBillingOrgId`, `cardHandoffPendingAt`, `currentPeriodEnd`, `trialEndsAt`) and resolves the org per-event through the chain above. Storing the customer pointer would have made that chain unnecessary. **New apps: add both.**

### Known gap — no AVS on live authorizations

The checkout payload carries email and name only — **no billing address** — and the hosted card embed collects number, expiry and CVV. Every live authorization therefore reaches the processor with **no AVS and no postal code**, which issuers and fraud rules decline at meaningfully higher rates. This has already produced an unexplained decline on a real conversion attempt.

The checkout-session API exposes a `collect` field (`Record<string, any>`; one documented key is `collect: { shippingAddress: true }`). **Confirm the billing-address key with Throttle rather than guessing** — a wrong payload breaks checkout entirely. Until then, treat card declines on live conversions as plausibly environmental, not necessarily a bad card.

### Webhook flow (architectural decision)

Billing events flow: **Throttle → the app's `/webhooks/throttle` handler → the app emits its own standardized event to customer-defined webhooks per §12.**

The app re-emits because customers integrate with the app's domain events, the app can enrich (org context, plan tier names) and filter, and Throttle credentials never reach customer endpoints.

### Still blocked / unbuilt

Not blocked on Throttle existing any more — these are simply unbuilt:

- **Commission payout rails** — accrual and reporting are live (§7); moving money (payout method, tax forms, mass pay) exists nowhere.
- **Buyer-portal add-card** and **cart-abandonment email**.
- **Marketplace billing** for platform app stores, which bill on the platform's rails rather than Throttle's.

---

## 24. Plan Entitlements & Feature Gating 🚧

**Every app in the portfolio bills by tier, so every app needs a gate that turns a subscription state into an actual restriction.** Until v3.8 this document specified who may act (§4 RBAC) and what they pay (§8, §23) but never the layer in between. That layer is where the most dangerous class of bug in this portfolio lives, because both of its failure modes are invisible: a gate that never runs lets everyone through, and a gate that runs too eagerly locks out paying customers.

### 24.1 Two axes that are not the same thing

These get conflated constantly. They have different causes, different UI, and different recovery, and an app that models them as one field will eventually tell a paying customer their own setting is a billing problem.

| | **Plan entitlement** | **Tenant feature toggle** |
|---|---|---|
| Question it answers | "Does their tier include this?" | "Do they want this on?" |
| Set by | Us, via the subscription's plan tier | The customer, in settings |
| Reason it's off | They haven't paid for it | They chose to turn it off |
| Correct UI | Visible but locked, with an upgrade path | Hidden entirely |
| How they fix it | Upgrade | Flip the switch back |
| Server response | `403` with an upgrade code | `404`/empty — the feature isn't part of this tenant's app |

**Rules:**

- Model them as **separate fields**. Never a single `featuresEnabled` blob that both billing and the customer write to.
- **Entitlement is checked first.** A customer cannot toggle on something their tier doesn't include, and turning a feature off must never look like a downgrade.
- **A toggle is not a licence.** Toggling a feature off does not entitle a refund or a tier change, and toggling it back on must not require re-purchase.

### 24.2 The gate

- **One tier→feature map, as data, in shared code** — read by the server guard *and* the client. If the sidebar computes entitlement differently from the API, you ship a nav item that 403s.
- **The gate fails closed.** An unknown tier grants the lowest feature set, never the highest.
- **The gate is a guard, so §4's registration hazards apply in full.** Where guard ordering is significant, the plan gate usually has to run *after* identity resolution and therefore must be registered where that ordering holds — moving its registration to a different module leaves it instantiated, injected, and enforcing nothing.
- **Every gate needs a test that asserts a denial.** See §21 principle 16 and §4.

> 🔴 **A plan gate that is a no-op looks exactly like a plan gate that is working.** Evident's shipped for months while enforcing nothing — every request passed, no error was raised, and the dashboards were green. It was discovered by trying to hit a gated endpoint on a free account by hand, not by any alarm. **Verify a new gate by attempting a gated action on an under-entitled account and confirming the 403.** Until you have seen the denial, you have not shipped a gate.

### 24.3 🔴 Activating a gate on an existing customer base

**Turning a gate on is a destructive operation against live accounts, and it must be treated as one.** The gate is correct code; the customer data is what's wrong. Every long-lived org predates the tiering and carries whatever state it happens to have.

Two specific traps, both of which have produced a near-miss lockout:

1. **An org with no subscription row at all falls back to the lowest tier.** Every account created before billing existed is in this state. Activating the gate silently strips features from customers who have been using them for a year. The fallback tier for a missing row is a load-bearing choice, not a default to pick casually — write it down and justify it.
2. **A stale provider status locks out a paying customer.** Same root cause as the §8 expiry-sweep guard: an account that is paid up can carry a `trialing` or `expired` local status. Gating on status alone acts on data you already know can be wrong.

**Required before merging a change that activates or tightens a gate:**

- **Audit every live org against the new gate** and produce the list of accounts that would lose access. Not a spot check — the full list.
- **Reconcile that list to zero, or make each remaining entry a deliberate, recorded decision.** An account you intend to lock out is fine; an account you didn't know about is an outage.
- **Ship the 403-handling UI first** (§24.4). Activating a gate before anything handles its response converts a paywall into a dead end.

> Evident's plan gate sat merged-but-unactivated for weeks specifically so this audit could happen. One customer would have been locked out of an account holding 4,700+ of their own records. The delay was the correct call.

### 24.4 The 403 must have a destination

A gate that returns `403` and no UI that handles it is a broken product, not a paywall — the customer sees a generic error on a feature they can legitimately buy.

- The gate returns a **distinguishable code** (e.g. `plan_required`) plus the tier that would satisfy it. A bare 403 is indistinguishable from a permissions failure and gets routed to the wrong support queue.
- The client renders an **upgrade prompt naming the required tier**, linking to the plan picker (§20 screen 18).
- Gated nav is **visible but locked**, not hidden. Hiding it means the customer never learns the feature exists, which defeats the point of tiering.

> **Known gap (Evident):** no UI handles the plan-gate 403 today. The gate is live and correct; the customer-facing half of it isn't built.

### 24.5 Downgrade, expiry, and data

- **Gating restricts access. It never deletes data.** A downgraded or expired org keeps everything; it just can't reach some of it. Deletion happens only through §15.
- **Re-entitlement is immediate and complete.** Paying restores access to the untouched data — no re-import, no re-configuration.
- Retention while gated follows §15's rules, not the gate's.

---

# Part VI — Communication & Support

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

> **Foundry status (v3.6):** still fully inline HTML — no `react-email` dependency, no `src/emails/`. The per-org `poEmailTemplate` override shipped, but as `{{variable}}` string substitution over inline HTML. Resend integration, `isConfigured()` degradation, `RESEND_API_KEY`/`RESEND_FROM_ADDRESS` naming, and the signed inbound webhook (PO replies, `RESEND_WEBHOOK_SECRET`) all match the standard.

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

> **Foundry status (v3.6): the largest schema gap in this doc.** Foundry's `Notification` is **org-scoped with no `userId` at all**, uses a boolean `read` (not `readAt`), and has no `audience` or `deliveryClass` — per-user fanout, audience resolution, and the transactional tier are unbuildable on the current shape without a migration. Categories are Foundry domain events (`order_sync`, `inventory`, `import`, `po_status`, `manufacturing`, `system`); none of the standard categories below exist. `NotificationPreference` is a single per-category `enabled` toggle (no bell/toast/email axis). UI: unread red dot (no count), recent-20 dropdown, no `/notifications` history page, no locked categories. Aligning Foundry here is a schema migration project, not an extension — sequence it as: per-user rows first, then per-method prefs, then `deliveryClass`.

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

# Part VII — Platform Infrastructure

## 12. Outbound Webhooks ✅

For pushing events to customer endpoints. Inverse of API keys.

> **Foundry status (v3.6): shipped, with contract deltas to reconcile.** Live: `WebhookEndpoint`/`WebhookDelivery` models (note the model name — not `Webhook`), full management API incl. `rotate-secret` and per-delivery replay, every-minute delivery worker, 8-attempt backoff, auto-disable on repeated failure, Settings → Outbound webhooks UI. Deltas from this section: headers are **`x-foundry-event` / `x-foundry-delivery-id` / `x-foundry-signature`** (`t=<ts>,v1=<hex>`), not `X-Epic-*` — since customers integrate with the app brand, the header prefix should probably become `x-<app>-*` in the standard; there is **no `dead_letter` state** (failed-after-8 is terminal but replayable); **no 3-failure warning notification**; **no usage endpoint / metrics**; **no truncation signaling**; and **secret rotation cuts over instantly** — the 24-hour dual-signing window prescribed below was actually built in Foundry's *other* webhook system (per-channel storefront webhooks carry `webhookSecretPrevious` + rotation timestamp) and should be ported here. The delivery worker's claim step is also a non-atomic read-then-update — racy the moment the API runs more than one instance (see §17.5 cron concurrency).

### Why this matters

Apps like Dispatch and Rally are integration-heavy — customers will want to subscribe to events and react in their own systems. Standardizing this once means reliable, auditable webhook delivery instead of six divergent implementations.

### Standard event prefixes

Events use namespaced types: `<resource>.<action>` — e.g. `ticket.created`, `attribution.matched`, `order.shipped`.

Shared events every app should emit (when applicable):

- `org.created`, `org.updated`
- `user.added`, `user.removed`
- `subscription.changed` — re-emitted from the Throttle webhook per §23, never forwarded raw
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
x-<app>-event: <event_type>              // v3.9 ruling: per-APP prefix (x-foundry-*, x-dispatch-*, …).
x-<app>-delivery-id: <delivery_uuid>     // Customers integrate with the app's brand; the portfolio is an
x-<app>-signature: t=<timestamp>,v1=<hmac_sha256_hex>  // internal fact. The old X-Epic-* spelling is withdrawn.
x-<app>-timestamp: <unix_seconds>
X-Epic-App: <app_name>                   // optional disambiguator for customers integrating with several Epic apps

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
- **Per-key scopes override role** *(v3.9 — no longer reserved; live in Foundry)*: when `scopes` is empty the key derives its permissions from `role` (original behavior, every old key unchanged); when non-empty, the scopes ARE the key's exact permission set and `role` is ignored. Lets an org mint a least-privilege key (products-only, read-only) without inventing a role.
- **Grantable scopes** *(v3.9)*: a user may only grant a new key scopes **their own role holds**, intersected with the assignable set — nobody mints a key more powerful than themselves, and OWNER-only powers (`org.close`, restore-class permissions) are never assignable to any key.
- **Key types** *(v3.9)*: keys carry a `keyType` when an app exposes more than one API surface. The reference case is Foundry's `ADMIN` vs `STOREFRONT`: storefront keys use a distinct prefix (`fims_sf_`), bind to exactly one resource (a channel), authenticate only the public storefront surface, and are **rejected by the admin guard outright** — a surface-scoped key must never pass the richer surface's auth, and vice versa. Each type gets its own creation endpoint so the binding is impossible to omit.

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

> **Foundry status (v3.9):** the email→userId migration happened to a richer shape than specified, and **v3.9 blesses it as the standard**: actor = `userId` + `userEmail` + `actorType` (enum `MANUAL | SYSTEM | API_KEY | WEBHOOK | UNKNOWN`), stamped from ambient AsyncLocalStorage context; pre-migration rows read `UNKNOWN` (deliberately not `SYSTEM`, so old rows make no false claims). **The denormalized `userEmail` snapshot is accepted** — history stays readable after a user leaves — with a hard condition: **the §15.2 deletion job MUST scrub the snapshots** (still an open checklist verification for Foundry). Principle 11 gains this as its one exception: IDs in *log lines*, snapshot-with-scrub in *audit/activity records*. Remaining deltas: no `partnerSeatId` on either log yet, and Foundry's live `source` values (`promote`, `gtin_match`, `csv_import`, `api`, `webhook`) don't come from the closed list above.

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

## 14.5 Security Audit Log ✅

A separate log for **security-sensitive events**, distinct from the business activity log. Compliance teams will query this directly.

> **Foundry status (v3.6):** shipped — `AuditLog` model (deliberately relation-free so rows survive org/user deletion) and the `/audit` admin page gated on `audit.read`. `auth.login` / `auth.logout` write from the Clerk session webhooks. Remaining baseline events below are instrumented incrementally; `partnerSeatId` not yet added. 7-year cold storage still open (checklist).

**Platform-level events (v3.9):** the audit log is org-scoped, but some staff actions are global — a disposable-email override (§3.4) affects every org. Convention: log the event **to the acting staff member's own org**, with a self-describing `resource` (e.g. `disposable_override:<id>`) and the operation in `metadata`. Do not invent a second, org-less audit store for a handful of platform events; do not skip logging them either.

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

### 15.2 User Right to Deletion ✅

A **user** (not org) can request their personal data be deleted, independent of whether the org stays open.

> **Foundry status (v3.6):** fully implemented — request/confirm/cancel/status endpoints under `/users/me`, 30-day grace + 24-hour immediate path, every-30-minutes execution cron, Clerk-delete-first ordering with the local tombstone in a single transaction, and `DataDeletionAudit` with HMAC'd email. Shape delta: there is no `DataDeletionRequest` model — request state lives as columns on `User` (`deletionRequestedAt`, `deletionScheduledFor`, `deletionFailureCount`, …). Functionally equivalent; the schema below should be treated as "either shape" for new apps. Open verification (checklist): confirm the deletion job scrubs denormalized email snapshots, including `ActivityLog.userEmail` (§14).

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

> **Foundry status (v3.6):** `instrument.ts` loaded first ✅, `SentryGlobalFilter` ✅, plus uncaught-exception handlers and a prod heap-usage watchdog (worth standardizing). **Not shipped: any PII redaction** — no `beforeSend` at all, and `@epic/sentry-config` doesn't exist as a package yet. Release var is still `SENTRY_RELEASE`; `APP_VERSION` exists only as a hand-maintained constant in the health controller (drift risk — wire it from the build instead). The redaction gap is the substantive one.

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

## 17.5 Operational Patterns 🚧

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

> **Foundry status (v3.6):** shape differs — `/health` returns `{status, db, timestamp, version, config}` with no `checks` object and no third-party sub-checks, and a DB failure returns **200 with `status: "degraded"` rather than 503**, so the LB-rotation behavior this section calls critical isn't implemented. `version` comes from a hand-maintained constant (§17).

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

- Trial expiry sweep — **with the §8 paid-into-the-future guard**, and a test covering its skip cases (§17.5)
- Webhook retry sweep
- Deletion grace period sweep (§15.2)
- Stale API key reminder (§13)
- Activity log cold-storage migration (§14)
- **Stale notification cleanup** — delete read `Notification` rows older than 90 days; unread rows kept indefinitely (or per app policy). Prevents the table from growing unbounded.
- **Abandoned partner-trial-org cleanup** — delete pre-created partner trial orgs after 14 days if the invited client never signed in (§7).

> **Foundry status (v3.6):** 26 cron jobs live via `@nestjs/schedule` (colocated per feature module rather than a single file — acceptable), including the abandoned-partner-trial cleanup and deletion-grace sweep. **No advisory locks or Redis SETNX anywhere** — five jobs have a per-process `running` boolean, which does nothing across instances, and the webhook delivery worker's claim step is a non-atomic read-then-update. This is safe only while the API runs a single ECS task; the locking requirement above is unmet and becomes urgent the day the service scales out. Also missing from the job list: the stale-API-key reminder (§13).

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

> **Foundry status (v3.6):** exists as `scripts/seed-test-futurama.ts` (Planet Express, Bender component BOMs, Mom's / Omicron Persei / Slurm / DOOP channels). Add an npm script alias so it's discoverable.

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
- **Every guard has a test that asserts a *denial*.** An allow-path-only test cannot distinguish a working guard from one that was never registered (§4, §24.2).

#### 🔴 Background workers and scheduled jobs (v3.8 — stipulated)

**Any job that can revoke access, delete data, or move money requires a test, and the service that runs it requires a test harness.**

This is stated separately because the standard's testing requirements have until now described the API, while the highest-consequence code in these apps runs somewhere else entirely. A scheduler that flips subscription statuses can lock a paying customer out of their account, and it does so on a timer, at night, with no user watching and no request to trace.

The failure mode is structural rather than careless. A worker service starts life as "just a queue consumer," gets no test setup because there is nothing to test yet, and by the time it owns billing enforcement the absence of a harness has stopped being visible — nobody is choosing not to write tests, there is simply nowhere to put them. Evident's worker has **no jest config and no test script**, and that is where its trial-expiry cron lives.

**Minimum bar:**

- The worker service has a test harness, even if it starts with one test.
- Every job matching the criteria above has a test covering **who it acts on and, more importantly, who it must skip**. The skip cases are the ones that cause incidents (§8's paid-into-the-future guard is exactly such a case).
- Jobs are structured so the selection logic is callable independently of the schedule. A job whose query can only be exercised by waiting for a cron cannot be tested.
- **A production dry run is not a substitute for a test.** It proves the job's behaviour against today's data only, and the data is the part that changes.

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

### Throttle webhook endpoint

`POST /webhooks/throttle`, per-app, public, secured by signature verification rather than auth — the same shape as the Clerk webhook (§3.9).

**The real event vocabulary, signature verification, org-resolution chain, and the required `BillingEvent` idempotency table all live in §23.** Earlier revisions carried a guessed event shape here; it was wrong in ways that failed silently, so it has been removed rather than left to be copied.

---

## 22. Rate Limiting 🚧

Every public surface is rate-limited. Usage metrics are exposed so users see consumption before they hit caps.

> **Foundry status (v3.6):** partially built, with a documented substitution — a custom in-memory fixed-window limiter runs per API instance (no Redis; the code marks distributed limiting as future work). Live: 300 req/min per ADMIN API key, per-endpoint storefront limits, `X-RateLimit-*` + `Retry-After` headers, structured 429 bodies. Not built: `@upstash/ratelimit`, per-IP signup limiting (signup is Clerk-hosted), the affiliate `/attribute` and auth per-IP limits, usage endpoints, and 80%-threshold notifications. The per-key default below (10k/hr) also doesn't match Foundry's actual 300/min — reconcile when Throttle tiers land.

### Baseline limits

| Surface | Limit | Notes |
|---|---|---|
| **Per-IP signup attempts** | 10 / hour | Blocks bot signup farms; doesn't friction real users. |
| **Per-API-key request budget** | **Default for free / no-billing apps: 10,000 requests / hour per key.** Plan-tier budgets are set per app off the subscription's plan tier. | Apps without billing keep the default; billed apps override per tier. |
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

## 25. Tenancy Enforcement 🧭 (v3.8 — supersedes "PR review" as the mechanism)

### Why this is now its own section

Principle 12 has said since v3 that every query filters by org and that **PR review is the enforcement mechanism** until a lint rule ships. That position is no longer tenable, and this section exists to record why rather than quietly leaving the principle in place.

- The lint rule (`@epic/eslint-plugin-tenancy`) has been 🧭 *not built* across four revisions.
- In the meantime **the same class of bug has recurred three separate times in a single app** — most recently in Evident's loyalty module, after a dedicated tenancy audit had already run and fixed every Critical finding.

Three repeats in one codebase, with review actively looking for it, is evidence about the mechanism rather than about any individual review. **A control that depends on a human noticing an absent clause does not scale to every query, forever.** The absence is the bug, and absences are exactly what review is worst at catching — there is no wrong line to spot, only a missing one.

This is also the §21 principle 16 shape: an unscoped query does not error. It returns *more* rows, cheerfully, and the app renders them.

### The standard

**Default-deny at the data layer.** A query against an org-owned table that does not carry a tenant scope must **fail**, not succeed broadly. Enforcement moves from review-time to run-time.

Required properties:

1. **Org-owned tables are declared**, once, in one place. Everything else is global by declaration rather than by omission.
2. **A query on a declared table without the tenant key throws** — in development and in production alike. A dev-only check trains people to write unscoped queries and discover it at 3am.
3. **Cross-org operations get an explicit, greppable escape hatch** — a named annotation that says "this is deliberately cross-org," never a silent pass. GDPR sweeps, admin tooling, and "list this person's memberships" are legitimate and rare. `grep` for the annotation should return a short list a person can read in one sitting.
4. **The escape hatch is auditable.** Every use is reviewable on its own merits, which is what review is actually good at — judging a small number of deliberate exceptions rather than policing every query.
5. **Raw SQL is covered or explicitly excluded, in writing.** An ORM-level layer does not see raw queries. If raw SQL is exempt, say so and keep the manual review obligation scoped to that slice — a much smaller surface than "all queries."

**Implementation direction:** a Prisma client extension is the natural home — it sits under every call site, needs no per-query cooperation, and cannot be forgotten by a new file. A lint rule remains useful as a fast local signal but must not be the primary control: it sees static queries only, and the failures so far have not all been static.

**Postgres RLS remains the long-term hardening layer**, below the application entirely. It is the strongest form of this control and the most work; the client extension is the version that can ship this quarter.

### Until it ships

The manual obligation stands, and is now narrower and more specific than "review carefully":

- **Any `find*` / `update*` / `delete*` on an org-owned table with no tenant key in the where-clause is a bug**, not a style note — including `findUnique` by a globally unique id, which is the most common way this slips through. A globally unique id proves the row exists; it does not prove the caller may see it.
- **Permission checks scope by `(clerkUserId, orgId)`, never `clerkUserId` alone** (§3.5). A multi-org user's rows in *other* orgs are returned by the unscoped version, and permissions leak with them.
- **New modules are the highest-risk surface.** Every recurrence so far arrived with a new feature, not with a change to an audited one. A tenancy audit is a snapshot; it does not cover code written after it.
- Treat a completed tenancy audit as **evidence about the past**, never as a property the codebase now holds.

---

## 26. Agent Access (MCP) ✅ Foundry · 📋 elsewhere (v3.9)

Customers increasingly reach apps through AI agents — Claude, ChatGPT, n8n flows, custom assistants — and the integration surface they expect is an **MCP server**. Foundry has run one in production since 2026-07 (listed in Anthropic's directory, the official MCP Registry, Smithery, LobeHub; submitted to OpenAI's app directory) and the pattern below is what survived contact. Every app should assume this ask is coming; new apps should reserve the architecture even if they don't build it at launch.

### Topology

- The MCP server is a **separate edge service** (Foundry: a Cloudflare Worker at `mcp.<rootdomain>`), not routes on the API. It translates MCP tool calls into ordinary authenticated API calls — the API stays the single enforcement point for auth, tenancy, and rate limits.
- Tools are **task-shaped, not endpoint-shaped**: `search_products`, `set_bom`, `bom_can_build` — a curated verb set with real descriptions, not a generated mirror of the REST surface.
- Destructive tools state their blast radius in the description and, where the host supports it, require confirmation.

### Identity: per-user, org-pinned credentials

The server must act as **a specific user in a specific org** — never as an app-wide service account:

- **First-party OAuth** on the app's own domain: the agent platform starts an OAuth flow, the user consents on our page, and the exchange mints a **per-user, org-pinned API key** scoped like any other key (§13). Secretless public clients via PKCE.
- **RFC 7591 dynamic client registration** — agent platforms register their clients programmatically; do not hand-maintain a client list.
- Tenancy comes from the key's org pinning, so a hijacked or confused agent can never reach across orgs. Org switching, where offered, is an explicit tool that re-validates membership server-side.

### Operational lessons (each cost an incident or a debugging day)

- **Token refresh races**: two concurrent tool calls refreshing the same token must coalesce — one wins, the other reuses the result. Rotating refresh tokens without this locks the grant out entirely.
- **Hosted agent platforms may open a NEW session per tool call.** No warm caches, no sticky sessions; every call must be independently cheap and independently authenticated.
- **Legacy grants drain slowly**: when the auth model changes, old connections keep working until each user reconnects. Version the grant, monitor both paths, and never force-revoke without messaging.
- **Monitor like a product surface**: error tracking on the worker, a periodic canary that exercises a real tool call, and usage metrics per tool.
- Directory listings are marketing surfaces with review processes — keep the registry keys/credentials safe (a registry listing is DNS-pinned), and expect reviewer accounts to need a constrained demo org.

---

## 27. Public API Documentation ✅ Foundry · 📋 elsewhere (v3.9)

If customers or partners can hold an API key (§13), the API's public documentation is a product surface with a correctness bar, not a build artifact. The contract, distilled from Foundry's 2026-08 full audit (241 operations, 100% described):

### The docs are generated, the *scope* is declared

- Docs regenerate from code on every deploy — never hand-maintained.
- Each public doc (partner API, storefront API, …) is built from an **explicit module allow-list**, commented at the definition site with "review every controller in a module before adding it." The full internal/admin spec is never exposed in production.
- Maintenance, debug, and staff endpoints are **excluded by annotation** (`@ApiExcludeEndpoint` or equivalent) even inside allow-listed modules. Absence of a docs entry is not security — those routes keep their guards — but advertised internals become support tickets and probe targets.

### Completeness bar (auditable, so audit it)

- **Every operation** has a one-line summary written for an integrator, not a restatement of the method name.
- **Every schema property** has a description. 100%, not "mostly" — the audit query is `properties without description == 0`, which makes the bar mechanically checkable in CI or a periodic sweep.
- Auth requirements, pagination style, and error shapes are documented once, centrally, and linked — not re-explained per endpoint.

### Toolchain traps (each silently produced wrong docs)

- **Doc-comment dialect matters**: the Swagger CLI plugin reads `/** */` only — `///` comments compile fine and never reach the spec. One character, zero docs.
- **A TypeScript `interface` used as a request body is invisible** to the doc generator. Converting it to a class changes runtime validation behavior (whitelist pipes start stripping), so the conversion is a deliberate change with validators added — not a mechanical rename.
- **Dead fields in DTOs ship as documented API.** A field the server ignores is a lie with a schema; delete it from the DTO, don't describe it.

---

# Part VIII — Web Presence & Marketing

## 18. Marketing Site Contract

The marketing site is the **front door** for new signups, not just a brochure. It hosts the actual signup form (via embedded Clerk), the conversion thank-you page where pixels fire, and the affiliate-link cookie capture.

### 18.1 Required pages and routes

Every marketing site must have:

1. **Home + content pages** — the usual marketing surface. Any of these can receive an affiliate `?r=` URL.
2. **`/signup`** — embedded Clerk `<SignUp />` component (per §3.2). Signup happens here, on the marketing domain.
3. **`/welcome`** — thank-you page, conversion pixel host (per §18.3). Users land here after Clerk completes signup.
4. **CTAs throughout the site** that route to `/signup` (NOT to `app.<rootdomain>`).
5. **`/partners`** ✅ — explains the partner program.
6. **`/partners/apply`** ✅ — submits to API. *(Foundry status: page exists but has no form — it routes into `/signup` + the in-app application (§7). Either build the form or amend this line to "routes to the in-app application.")*

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
- **Is `noindex`** (v3.6) — `/welcome` (and `/signup`) should carry a noindex meta and stay out of the sitemap; conversion pages aren't crawl targets.

> **Foundry status (v3.9):** `/welcome` is live with GTM/GA4 + Meta + LinkedIn wired (env-gated) and clears the cookie; TikTok/Reddit not wired (fine — no ads on those channels). **The noindex rule is now met**: both `/signup` and `/welcome` carry `noindex, nofollow` and are excluded from the sitemap (astro-foundryims#61, live 2026-08-31). Remaining gap: GTM loads **only on `/welcome`**, not site-wide as recommended above.

### 18.4 Cookie consent

Every marketing site that targets EU traffic implements a consent banner. **Standard library: `@epic/cookie-consent`** (shared dev dependency, see §1) — handles the banner UI, consent storage, and exposes a `hasConsent('functional' | 'analytics' | 'marketing')` API.

- **Functional cookies** (the affiliate `?r=` cookie, Clerk session cookies): set without explicit opt-in. Disclosed in the banner's "what we use" panel.
- **Analytics + marketing cookies** (GA4, Meta Pixel, LinkedIn, TikTok, Reddit, etc.): require explicit opt-in. Pixels must check `hasConsent('marketing')` before firing the conversion event.

**Legal sign-off on the affiliate cookie's "functional" categorization is still pending.** Until cleared, apps targeting EU traffic should re-categorize the affiliate cookie as "marketing" and require opt-in. Apps not targeting EU traffic may skip the banner entirely; document that decision per app.

Reference: `astro-foundryims/src/layouts/Layout.astro` for the cookie capture script.

> **Foundry status (v3.6): nothing shipped.** No consent banner, `@epic/cookie-consent` doesn't exist as a package, Ahrefs analytics loads unconditionally on every page, and the affiliate cookie is set without a consent gate. The standard requires either a banner or a documented per-app decision to skip — neither exists. Decision needed (now on the audit checklist).

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

# Part IX — Adoption & Governance

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
- [ ] **Throttle `BillingEvent` table** + signature-verified webhook handler per §23 — required, not reserved
- [ ] **Plan entitlement gate** per §24 — one tier→feature map in shared code, fails closed, denial-asserting test, `plan_required` 403 with an upgrade prompt behind it
- [ ] **Tenant feature toggles** modelled separately from plan entitlement per §24.1
- [ ] **Default-deny tenancy layer** per §25 — new apps get this from day one rather than retrofitting it after the third incident
- [ ] **Worker test harness** per §17.5 — set up with the first scheduled job, not after it owns billing
- [ ] **Secret rotation runbook** per §17.6 (annual cadence + compromise-driven)
- [ ] **Integration test suite** for auth + core domain per §17.6 (CI gate)
- [ ] **Shared library access** — `.npmrc` + `GITHUB_PACKAGES_TOKEN` for `@epic/*` packages per §17.6

### Deferred (build when ready, schema reserves now)
- [ ] **Outbound webhook infrastructure** per §12 — `Webhook` + `WebhookDelivery` models in initial schema even if not wired
- [ ] **Partner program** per §7 — port from Foundry (shipped); `accountType` + `Referral.referralType` already in schema
- [ ] **Trial period + Throttle integration** per §8 / §23 — live contract; includes the paid-into-the-future expiry guard and the four-step org resolver

### Required screens — auth / affiliate / partner / billing (v3.7)

The canonical screen inventory for the standardized functionality. Every app ships all of these (billing group ⏸️ until Throttle); "screen" includes dialogs that carry a full flow. This is the bulletproofing checklist for QA: each screen maps to API surfaces defined in its section, and a workflow test should walk each one as a brand-new user.

**Marketing site (§3.2, §18):**
1. `/signup` — embedded Clerk `<SignUp />`, cookie → `unsafeMetadata` wiring, noindex
2. `/welcome` — conversion pixels, "Continue to Dashboard", noindex
3. `/partners` — program explainer
4. `/partners/apply` — application entry (form or route into the in-app application)

**Auth & org (§3, §4, §15):**
5. Admin `/login` (+ `/signup` fallback route)
6. Org switcher — switch / create / leave, active indicator
7. Settings → General — org name/logo, Danger Zone close with typed confirm + final-export offer
8. Settings → Users — members, pending invites, role dropdown, remove
9. Settings → Privacy — export my data · delete my account (30-day + immediate) · deletion status/cancel

**Affiliate (§6):**
10. Settings → Affiliate — link card + copy, stat cards, signups table; visible to every role

**Partner / referral (§5, §7):**
11. Settings → Partner Program — in-app application + application status (any user)
12. Partner dashboard — Referrals tab · Seats tab · "Create trial for client" dialog · Team tab (⏸️) · Commissions tab (⏸️) · Profile/payout tab (⏸️)
13. Promote-client-to-owner / handoff confirm flow (the Path B election) — must state that the client needs their own card and that the partner rate ends at renewal (§7 handoff rule 2)
14. Settings → Partners (client side) — seats granted, permission-tier select, revoke
15. Internal admin — partner application review queue (approve / reject)

**Billing (§8, §23 — live; see §23 for the integration contract):**
16. Settings → Billing — current plan, payment method, invoice history
17. Trial state — countdown banner + upgrade CTA (admin shell)
18. Plan picker — convert / upgrade / downgrade, with proration preview
19. Cancel + reactivate flow — typed confirm, end-of-period notice
20. Partner billing view — white-label client subscriptions invoiced at 80% of list, per-client model election (white-label ⇄ client-paid), with the commission-forfeit confirmation (§7 Path A rule 5)
21. Dunning — payment-failed banner + `billing.payment_failed` transactional notification
22. **Card-handoff banner** — persistent notice to a new owner after a white-label handoff that the partner's card is still on file and the next renewal will charge them or fail; clears via the acknowledge endpoint (§7 handoff rule 2)
23. **Commission statement** — per-client basis and period, showing the uncapped basis where the §6 $500 cap bound. Reporting the basis is required even where no payout rail exists.

**Plan gating (§24):**
24. **Upgrade prompt** — what a `plan_required` 403 renders as: names the required tier, links to the plan picker. **Ship this before activating any gate** (§24.3); a gate without it is a dead end, not a paywall.
25. **Locked feature state** — gated nav and entry points are visible but locked, never hidden. Hiding them means the customer never learns the feature exists.

Supporting infrastructure screens (support §10, notifications §11, API keys §13, activity §14, audit §14.5, webhooks §12) are enumerated in their own sections' "UI surfaces" blocks.

---

## 21. Principles

1. **Standardize the boring, customize the value.** Signup, billing, support, affiliate, RBAC, notifications, API keys, observability — identical across every app. The product itself is where each app earns its keep.

2. **Visual design is not standardized.** Each app earns its own look and feel. We standardize behavior, not appearance. Common UI patterns (drag handles for ordering, header sorting, column controls) are encouraged when applicable but not mandated.

3. **Clerk owns identity. Throttle owns billing.** Within a given app, Clerk is the source of truth for users and orgs. Throttle handles all transactions. Local DB mirrors what's needed for joins. Cross-app identity is not a thing in v1.

4. **Attribution at signup, not retroactively.** Affiliate and partner referral credit is captured when the org is created. No after-the-fact claims.

5. **Action is proof of attribution.** Partner-tier referral credit requires the partner to physically create the trial. No forms, no disputes.

6. **Two referral tiers, priced for two different acts.** An affiliate makes an introduction and is done: **100% of the referred org's first month, one time, capped at $500 per referral**, held 30 days (§6). A partner carries the relationship: **10% of every renewal, for the life of the subscription, no cap**, and nothing on the first payment (§7). Partner status additionally unlocks the dashboard, partner seats, the trial-creation flow, and a per-client election — stay white-label and take a 20% discount at source, or hand ownership to the client and take the 10%. **One client, one model, never both.**

7. **Schema reserves the future.** `convertedAt`, `rewardStatus`, `accountType`, `Webhook`, `AuditLog`, `scopes` exist in the model even before billing/partner/scope code does. New apps copy them so the migration on activation is zero-schema.

8. **External services stay server-side.** Clerk, Dispatch, Resend, Throttle, and webhook secrets never reach the browser.

9. **One-time secrets, hashed at rest.** API keys, webhook secrets, deletion confirm tokens are shown to the user once, hashed in the DB. Never recoverable.

10. **Permissions are checked on every request.** Sessions don't cache role data beyond a short documented TTL (5–30 seconds default per §16). A demotion takes effect within seconds, not at session/token expiry.

11. **Privacy and deletion are first-class.** Right-to-deletion is built in, not bolted on. PII is in nullable fields only. Activity logs reference user IDs, not emails. Audit logs use HMAC for any necessary email hashes. Backups have documented 30-day retention with deletion-rerun-on-restore.

12. **Defense in depth on tenancy — enforced by the data layer, not by review.** Every query filters by `orgId`; every API request is scoped to the caller's active org. **PR review is no longer the stated mechanism** — it was, for four revisions, and the same bug recurred three times in one app anyway. The standard is now **default-deny at run time**: an unscoped query on an org-owned table fails, and genuine cross-org operations use an explicit, greppable annotation. See **§25**. Postgres RLS remains the long-term hardening layer.

13. **PII boundaries are explicit.** No PII in Sentry tags, URL paths, log lines, or aggregate analytics. PII lives in `User`, `Organization`, and explicit snapshot fields only.

14. **Rate-limit by default.** Every public endpoint and per-key access has a rate limit. Default deny, raise as needed. Usage metrics are visible to users so they see limits coming.

15. **Transactional notifications bypass preferences.** Users can't opt out of deletion confirmations, security alerts, or payment failures. Compliance and account safety override convenience.

16. **Silent success is the failure mode to design against.** The expensive bugs in this portfolio have not thrown errors — they reported success while doing nothing. A plan gate that never ran. An import that reported COMPLETED having imported nobody. A webhook that returned 200 on every event and recorded none. A sync endpoint that never checked its caller. Each looked healthy in logs and dashboards for weeks.

    Design against it: **prefer a loud skip to a quiet pass.** When a code path declines to act, say so at `warn` with the identifiers needed to chase it. Every guard gets a test that asserts a **denial**, not just an allow. Any handler that can fail to resolve its subject logs the miss rather than returning 200. If a feature cannot be observed working, assume it is not.

17. **Provider contracts are verified against live traffic, not against our own design docs.** Event names, field names, HTTP verbs, and auth headers described in a design-phase document are guesses until a real delivery confirms them — and a wrong guess here fails silently (§23 has three worked examples, including an auth header that returns `200` with an empty body). Before building on a provider's shape: capture one real payload, assert against it in a test, and record the verified vocabulary in the standard so the next app doesn't re-derive it.

