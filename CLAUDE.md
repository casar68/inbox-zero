# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

`AGENTS.md` (imported above) is the source of truth for build/test/lint commands, code style, and the fullstack workflow. The notes below add the big-picture architecture that spans multiple files. For a deeper directory-level map and the rationale behind the AI assistant design, see `ARCHITECTURE.md`.

## Monorepo Layout

Turborepo + pnpm workspaces. Install packages inside the app that uses them (e.g. `cd apps/web && pnpm add ...`), not at the root.

- `apps/web` — the main Next.js (App Router) app; frontend, API routes, server actions, Prisma schema, and nearly all business logic live here. This is where most work happens.
- `apps/worker` — background worker process for long-running / queued jobs.
- `apps/image-proxy`, `apps/image-proxy-aws` — Cloudflare Worker / AWS image proxies.
- `packages/*` — shared libraries: `api` (public API client), `cli` (self-hosting/deploy), `scheduling`, `tinybird` + `tinybird-ai-analytics` (analytics), `resend`/`loops` (email), `tsconfig`.

## Provider Abstraction (most important cross-cutting pattern)

The product is multi-provider (Gmail + Outlook, and beyond). Almost all integrations go through a factory → interface pattern so feature code stays provider-agnostic:

- Email: `createEmailProvider({ emailAccountId, provider })` in `utils/email/provider.ts` returns an `EmailProvider` (`GmailProvider` | `OutlookProvider`). Feature code calls the `EmailProvider` interface, never a raw Gmail/Outlook client.
- The same pattern exists for `utils/calendar/`, `utils/drive/`, `utils/messaging/` (Slack/Telegram), and `utils/llms/`.

Per `AGENTS.md`: prefer the `EmailProvider` abstraction; only use provider-type checks (`isGoogleProvider`, `isMicrosoftProvider`) at true provider boundary / integration code.

## Multi-Account Routing

A user can connect multiple email accounts. App routes are namespaced per account under `apps/web/app/(app)/[emailAccountId]/...` (assistant, automation, reply-zero, bulk-unsubscribe, briefs, drive, calendars, stats, settings, etc.). Route groups: `(app)` (authenticated product), `(landing)`, `(marketing)`, `(redirects)`. API routes live under `app/api/`.

Middleware tiers (see `AGENTS.md` for usage): `withError` (public), `withAuth` (user), `withEmailAccount` (scoped to one email account).

## AI Engine

All AI logic lives under `apps/web/utils/ai/`, split by feature: `choose-rule` (rule matching), `reply`, `digest`, `meeting-briefs`, `document-filing`, `categorize-sender`, `clean`, `knowledge`, `mcp`, `assistant`. See `.claude/skills/llm/SKILL.md` and `.claude/skills/llm-test/SKILL.md` before changing prompts or LLM behavior.

The AI personal assistant compiles a user's plain-English prompt file into individual **rules stored in the database** (Prisma); the DB rules — not the prompt file — are what reach the LLM, via a two-way sync. `ARCHITECTURE.md` explains the trade-offs and known rough edges of this design.

### Email processing flow

1. Gmail/Outlook webhook fires → provider webhook handler fetches the email.
2. `utils/ai/choose-rule` matches the email against the account's DB rules.
3. Matching rule's actions run through the active `EmailProvider` (archive, label, draft reply, etc.), with AI-generated arguments where needed.
4. Executed rules/actions are persisted (Prisma), enabling per-rule analytics.

Reply tracking is implemented as a special rule type on top of this engine; the cold-email blocker is a separate path (it runs an LLM only for senders the user has never emailed before).
