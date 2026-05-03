# app-standardization — Roadmap

The standardization repo's own roadmap, eating its own dog food (per `standard-features.md` §17.5). Tracks portfolio-wide infrastructure work — shared libraries, cross-app initiatives, and the standard-features doc itself.

## Shipped

| Item | Version | Owner |
|---|---|---|
| `standard-features.md` v3.x — Clerk auth, multi-tenant orgs, affiliate program, partner program design, support, notifications, webhooks, audit log, deletion, Sentry conventions, marketing-site front-door signup, team artifacts | v3.6 | Kal |

## In flight

| Item | Owner | ETA |
|---|---|---|
| Foundry IMS audit + alignment to v3.6 | Kal | next 3-4 sprints (see `app-foundry-ims/docs/audits/foundry-auth-affiliate-audit.md` for the punch list) |

## Planned

| Item | Why | Target |
|---|---|---|
| **`@epic/disposable-emails`** package | Required by §3.4 (signup-time blocking); apps currently have no shared list. Punted at IMS audit phase — `disposable-email-domains` npm package is the interim. | When second app adopts the standard |
| **`@epic/sentry-config`** package | Required by §17 (PII redaction in `beforeSend`, shared init). Apps currently roll their own Sentry init. | When second app adopts the standard |
| **`@epic/email-templates`** package | Required by §9 (shared header / footer / button / layout components for `react-email`). Apps currently inline their own. | When second app adopts the standard |
| **`@epic/cookie-consent`** package | Required by §18.4 (consent banner UI + `hasConsent('functional' \| 'analytics' \| 'marketing')` API). Apps currently target US-only or punt the banner entirely. | When second app adopts the standard, or when EU traffic becomes material for any app |
| **`@epic/eslint-plugin-tenancy`** package | Required by §21.12 (CI rule that flags unscoped `find*`/`update*`/`delete*` queries on org-owned tables). Currently a PR-review checklist item. Custom ESLint rule using TypeScript AST + Prisma schema introspection. | When second app adopts the standard, or when an unscoped-query bug surfaces in any app |
| **Shared library packaging infra** — `Epic-Design-Labs/shared-libraries` monorepo + GitHub Packages publishing | Required by §17.6. Repo doesn't exist yet. All five `@epic/*` packages above will live here. | Block on the first shared-lib actually being built |
| **Throttle billing platform** | Throttle is intended to replace per-app billing. v3.5 §23 currently a stub. Several v3.5 sections (§7 commissions, §8 trials, §11 payment_failed, §22 per-key budgets) are ⏸️ blocked on Throttle landing. | TBD — gated on its own design doc |
| **Partner program implementation** | Designed in v3.5 §7 but no app has built it yet. Schema is reserved (`User.accountType`, `Referral.referralType`); UI + flows pending. First-app implementation will be the reference for the rest. | After Foundry's audit alignment completes |

## Parked

| Item | Reason |
|---|---|
| Cross-app identity (one Clerk user across multiple Epic apps) | Out of scope per v3.5 §1. Deliberate v1 simplification — each app has its own Clerk project. Revisit when the operational pain of separate identities exceeds the design cost of a centralized service (likely never). |
| Mobile / PWA | Out of scope per v3.5 §17.5. App-by-app decision. |
| i18n beyond English | Out of scope per v3.5 §17.5. Per-app when justified. |
