# Inbox Zero Architecture

This document gives the big-picture architecture of the repository. For build/test/lint commands and code conventions, see `AGENTS.md`. For a Claude-Code-focused orientation, see `CLAUDE.md`.

Inbox Zero is a Turborepo + pnpm monorepo: a main Next.js web app (`apps/web`) holding nearly all business logic, a background job worker, image proxies, and shared packages.

```txt
├── apps/
│ ├── web/             // Main Next.js (App Router) app — frontend, API routes, server actions, Prisma, AI logic
│ ├── worker/          // BullMQ worker that drains Redis queues and forwards jobs to the web app
│ ├── image-proxy/     // Cloudflare Worker image proxy
│ └── image-proxy-aws/ // AWS image proxy
├── packages/          // Reusable libraries and configurations
│ ├── api/             // CLI for the external/public API
│ ├── cli/             // Self-hosting / setup CLI
│ ├── image-proxy/     // Shared image-proxy utilities
│ ├── loops/           // Loops marketing-email integration
│ ├── resend/          // Resend transactional-email integration
│ ├── scheduling/      // Shared scheduling helpers
│ ├── tinybird/        // Tinybird real-time analytics
│ ├── tinybird-ai-analytics/ // Tinybird AI-usage analytics
│ └── tsconfig/        // Shared TypeScript configs
├── charts/            // Helm chart for Kubernetes deployments
├── docker/            // Dockerfiles and Compose for local/prod deployment
├── docs/              // Public documentation site
├── qa/                // Browser QA flow definitions
└── ...
```

## 1. `apps/web` — Main Web Application

- **Framework:** Next.js (App Router).
- **Auth:** Better Auth (`utils/auth.ts`), with support for email accounts across multiple providers and a mobile/Expo client.
- **Purpose:** All user-facing UI plus the backend — API routes, server actions, the AI engine, and provider integrations.

Key directories:

- `app/` — App Router. Route groups: `(app)` (authenticated product), `(landing)`, `(marketing)`, `(redirects)`. API routes under `app/api/`.
- `components/` — reusable React components shared across pages (shadcn/ui + Radix + Tailwind).
- `providers/` — React context providers (auth/email-account context, SWR, chat, compose modal, PostHog, etc.).
- `store/` — **client-side** Jotai atoms and UI-level work queues (see §4).
- `utils/` — the bulk of backend logic and shared helpers, including the AI engine and the provider abstractions (see §3, §5).
- `prisma/` — PostgreSQL schema and migrations.

### Multi-account routing

A user can connect several email accounts. Authenticated product routes are namespaced per account under `app/(app)/[emailAccountId]/...` (assistant, automation, reply-zero, bulk-unsubscribe, cold-email-blocker, briefs, drive, calendars, smart-categories, stats, settings, integrations, …).

API middleware tiers: `withError` (public, no auth), `withAuth` (user-level), `withEmailAccount` (scoped to a single email account). Mutations use server actions (`next-safe-action`); GET API routes are for data fetching, consumed via SWR on the client.

## 2. `apps/worker` — Background Jobs

A lightweight BullMQ worker (`apps/worker/src/runtime.mjs`) connected to Redis. It listens on queues — `automation-jobs`, `digest-item-summarize`, `email-summary-all`, `email-digest-all` — and for each job forwards an authenticated HTTP request (internal API key) to the corresponding `apps/web/app/api/...` route. This keeps heavy/long-running work (rule automation, digests, summaries) off the request path while letting the actual logic live in the web app. Self-hosted deployments without the worker can drive the same endpoints via cron.

> Note: this server-side queue system (BullMQ + Redis) is distinct from the client-side Jotai queues in `apps/web/store` (§4), which only sequence UI actions in the browser.

## 3. Provider Abstraction (core cross-cutting pattern)

The product is multi-provider (Gmail + Outlook, and more). Integrations go through a factory → interface pattern so feature code stays provider-agnostic:

- **Email:** `createEmailProvider({ emailAccountId, provider })` in `utils/email/provider.ts` returns an `EmailProvider` (`GmailProvider` | `OutlookProvider`). Feature code targets the `EmailProvider` interface, never a raw Gmail/Microsoft Graph client.
- The same pattern is used for `utils/calendar/`, `utils/drive/` (Google Drive / OneDrive attachment filing), `utils/messaging/providers/` (Slack, Telegram), and `utils/llms/` (model providers).

Rule of thumb (also in `AGENTS.md`): prefer the `EmailProvider` abstraction; only use provider-type checks (`isGoogleProvider`, `isMicrosoftProvider`) at true provider boundary / integration code.

## 4. `apps/web/store` — Client State & UI Queues

Client-side state management with Jotai. Includes UI work queues that sequence browser-initiated batch actions (e.g. `archive-queue`, `archive-sender-queue`, `mark-read-sender-queue`, `ai-queue`, `ai-categorize-sender-queue`, `sender-queue`). These run in the browser and call server actions / API routes; they are not the background-job system (§2).

## 5. `apps/web/utils/ai` — AI Engine

All AI logic lives here, split by feature: `choose-rule` (rule matching), `reply`, `digest`, `meeting-briefs`, `document-filing`, `categorize-sender`, `clean`, `knowledge`, `mcp`, `assistant`, `group`, `report`, `snippets`, `calendar`, `automation-jobs`. LLM provider wiring lives in `utils/llms/`. See `.claude/skills/llm/SKILL.md` and `.claude/skills/llm-test/SKILL.md` before changing prompts or LLM behavior, and back prompt changes with evals.

## 6. `apps/web/prisma` — Database Layer

PostgreSQL via Prisma. `schema.prisma` defines the schema; `migrations/` holds migration history. (Do not use dynamic Prisma transactions — see `AGENTS.md`.)

## API Endpoints

Under `apps/web/app/api/`. Notable groups:

- `/api/ai/*` — AI features (categorization, summaries, autocomplete, models).
- `/api/auth/*` — Better Auth; `/api/mobile-auth/*`, `/api/sso/*` for mobile and SSO.
- `/api/google/*`, `/api/outlook/*` — provider API proxies and OAuth.
- `/api/watch/*`, `/api/email/*`, `/api/email-stream/*` — webhook/watch registration and inbound email handling.
- `/api/automation-jobs/*`, `/api/scheduled-actions/*`, `/api/cron/*`, `/api/follow-up-reminders/*` — background-job and scheduled-work endpoints (driven by the worker or cron).
- `/api/digest-preview/*`, `/api/meeting-briefs/*`, `/api/knowledge/*`, `/api/clean/*` — feature endpoints.
- `/api/slack/*`, `/api/telegram/*`, `/api/teams/*` — chat/messaging integrations.
- `/api/stripe/*`, `/api/lemon-squeezy/*`, `/api/apple/*` — payment/subscription webhooks (Stripe, Lemon Squeezy, Apple IAP).
- `/api/resend/*` — transactional/summary emails.
- `/api/chat/*`, `/api/chats/*`, `/api/mcp/*` — assistant chat and MCP.
- `/api/user/*`, `/api/organizations/*`, `/api/admin/*`, `/api/health` — user/org/admin/health.
- `/api/v1/*` — versioned public API for external integrations.

## Key Data Flows

1. **Email processing & AI automation:**
   - A Gmail/Outlook webhook fires; the provider webhook handler fetches the email via the active `EmailProvider`.
   - `utils/ai/choose-rule` matches the email against the account's database rules.
   - The matching rule's actions run through the `EmailProvider` (archive, label, draft reply, file attachment, etc.), with AI-generated arguments where needed.
   - Executed rules/actions are persisted (Prisma), enabling per-rule analytics. Heavy steps are queued to the worker (§2).

2. **Bulk Unsubscriber:** the UI lists newsletters/senders (sourced from Tinybird analytics); the user selects targets; unsubscribe/archive run via server actions + provider integrations; status is persisted.

3. **Email Analytics:** Tinybird data sources/pipes collect activity; `app/(app)/[emailAccountId]/stats` reads from the Tinybird API and renders charts.

4. **Digests & Meeting Briefs:** scheduled jobs (worker queues / cron) summarize email into digests and assemble pre-meeting briefs from email + calendar context.

## Environment Variables

Configuration is env-driven (add new vars to `.env.example`, `env.ts`, and `turbo.json`; prefix client vars with `NEXT_PUBLIC_`). Covers: LLM/API keys (OpenAI, Anthropic, Google AI, Bedrock, Groq, Ollama), OAuth credentials (Google, Microsoft), Postgres + Redis URLs, Pub/Sub topic and verification token, payment providers (Stripe, Lemon Squeezy, Apple IAP), analytics/logging (Tinybird, PostHog, Axiom, Sentry), email (Resend, Loops), feature flags, admin emails, and internal webhook/API keys.

## Feature Design Notes

### AI Personal Assistant

The user sets a prompt file which gets converted to individual rules in our database. What is ultimately passed to the LLM is the database rules, not the prompt file. We maintain a two-way sync between the DB rules and the prompt file. This is messy; a one-way data flow from the prompt file might be cleaner.

Benefits of database rules:

- In most cases the AI only decides whether conditions match.
- Each rule is a distinct entry, so we can track how often each is called — impossible with a fully prompt-based approach.
- Actions are static (unless using templates), so the user can precisely define behavior without LLM interference.

The current structure reflects how the product evolved; a from-scratch design would likely avoid the two-way sync. One downside: information in the prompt file that isn't a rule (e.g. global style guidelines at the top) doesn't naturally reach the LLM — the `about` section on Settings exists for this but is separate.

### Reply Tracking

Built on top of the AI personal assistant via a special rule type. Keeping it as a rule type (rather than a standalone feature like the cold-email blocker) is slightly messy but integrates with the existing assistant and everything built around it. Consequence: each user has their own reply-tracking prompt, which makes global prompt updates harder than for the cold-email blocker.

### Cold Email Blocker

Monitors incoming email; if the sender has never received email from the user, it runs the message through an LLM to decide whether it's a cold email. This feature is independent of the AI personal assistant.
