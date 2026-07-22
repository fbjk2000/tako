# TAKO — AI-Native CRM

<p align="center">
  <img src="frontend/public/logo-horizontal.svg" alt="TAKO" height="50" />
</p>

<p align="center">
  <strong>The CRM that runs your marketing and sales. Built for European teams that want results, not complexity.</strong>
</p>

<p align="center">
  <a href="https://tako.software">tako.software</a> •
  <a href="#features">Features</a> •
  <a href="#security">Security</a> •
  <a href="#operations">Operations</a> •
  <a href="#pricing">Pricing</a> •
  <a href="#tech-stack">Tech Stack</a> •
  <a href="#distribution">Distribution</a> •
  <a href="#getting-started">Getting Started</a> •
  <a href="#environment-variables">Environment Variables</a> •
  <a href="#api-reference">API Reference</a> •
  <a href="#external-api-v1">External API (v1)</a> •
  <a href="#integrations">Integrations</a> •
  <a href="#recent-updates-aprjun-2026">Recent Updates</a> •
  <a href="#pending--roadmap">Roadmap</a>
</p>

---

## Features

### Core CRM
- **Leads** — Import via CSV, manual creation, AI enrichment & scoring, bulk operations, column visibility, business card capture
- **Contacts** — Convert from qualified leads, rich profiles (budget, timeline, decision maker, pain points)
- **Deals** — Kanban pipeline with drag-and-drop + list view, entity linking (Lead/Contact/Company), lost deals excluded from pipeline
- **Tasks** — Tasks v2 ships across four phases (Phase 1 → 3, May 2026):
  - **Pulse view** (Phase 1) — outcome-first cards bucketed into Suggested by TAKO · Overdue · Today · This Week, ranked by deal urgency, contact decay and external signals (not by raw due date). Multi-entity link chips (deal · company · contact · campaign · project) on every card; click a chip to jump to that entity. Inbound query-string filters from elsewhere in the app pre-scope the view (`?deal=…`, `?contact=…`, etc.). Click any task body to open the **Context Lens** right dock; click the pencil to edit.
  - **Sentinel suggestions** (Phase 2) — daily 08:30 + Mon 06:00 catch-up cron walks the last 7 days of inbound email + activity, classifies threads via Haiku, extracts up to 3 grounded suggestions per positive thread via Sonnet, dedups against pending + last-7-days dismissed via source-hash, caps at 5 pending suggestions per user. Each carries a "why now" reasoning line. Strip across the top of /tasks; one-click Accept (promotes to a real task) / Dismiss (kept for dedup window).
  - **Multi-collaborators + sub-nav** (Phase 2.1) — tasks have `assigned_to` (owner) plus `collaborators[]` (multi-person). Reassignment + collaborator-add fires in-app + email notifications via the cascade helper. Default Pulse scope is "involved" (owner OR collaborator) — closes the pre-2.1 hole where every user could see every task. Contextual left **TaskNav** with FOCUS (Today / Overdue / This week / Snoozed) · SURFACES (By deal / contact / campaign / project / Delegated) · SMART LISTS (4 outcome-driven lists) · BY OWNER (admin-only). Counts via `GET /api/tasks/smart-list-counts`. **NextSteps** AI-suggested actions appear in the Lens dock (Draft email · Add task · Block calendar) following the existing draft-email pattern.
  - **Constellation + Calendar + ⌘K + Janitor** (Phase 3, closing chapter) — three new tabs on /tasks (Pulse | Constellation | Calendar) driven by `?view=`. **Constellation** is a hand-rolled radial SVG: entities arranged on a circle, tasks orbit their primary entity, halos tinted by urgency_signal (amber win-window / coral stalled / teal quiet), curved Bézier arms ("octopus arms"), cross-entity dashed teal links, pulsing ring on overdue task nodes. Cached 1h server-side. **Calendar auto-scheduler** — `POST /api/tasks/schedule` reads busy intervals from Google Calendar cache + scheduled_calls + time-pinned tasks, greedy-fits the day's involved-scope tasks (largest first; deep-work prefers morning, admin prefers afternoon). **Global ⌘K palette** — mounts once at app root, fires from any signed-in route (modifier-gated so plain "k" in inputs doesn't trigger), 300ms debounce → `POST /api/tasks/parse` → live preview with resolved entity chips → Enter creates the task. TopBar pill restyled to "⌘ capture" + clickable. **Janitor Sentinel slice** — sister to the email slice; finds deals with `last_touched_at < now-14d` not in closed-stages, generates one grounded suggestion per stalled deal (cap 3/user/day, source-hash anchored on `(deal_id, stage, ISO-week)` so a still-stalled deal next Monday produces a fresh suggestion). Same Mon-Fri 08:30 + Mon 06:00 cron. Manual trigger via `run-janitor-once.yml` workflow.
  - **Today's Focus card (Phase B)** — daily TaskRanker agent at Mon-Fri 06:55 Europe/London (5 minutes ahead of the campaign-runner so the focus card is fresh before any new step-2/3/4 tasks land). Per-user `priority_mode`: **Revenue** (deals near close, warm reply windows, high-value campaigns) · **Cost** (vendor renewals, expense audits, recurring-spend reviews) · **Balanced** (60/40 weighted blend). Three-mode pill on the focus card switches mode + instantly re-ranks (no 60s wait for the next cron). Top 3 still-open tasks render at the top of `/tasks`, with a one-line italic AI rationale per row (e.g. "overdue + warm reply window"). Rolling: when you complete a task, the next ranked one auto-promotes from the cached `ranked[]`. Manual **Refresh** button re-ranks on demand (rate-limited to 1/min/user). **Reserved**: greyed-out "Include private life" toggle wired schema-side (`users.include_private_chores` + `task.source` field) for the future Private Chores project.
  - The old kanban + list task views, subtasks/checklists, and comments were retired with the Pulse rewrite — they re-land in a future phase if usage signals call for them.
- **Projects** — Full workspace page per project (`/projects/:id`): click-to-edit name/description with autosave + "Saved" flash, drag-and-drop task board (To Do / In Progress / Done) with per-column quick-add + a list view, inline team management (add/remove members, synced to the project chat channel), target date, linked deal, progress tracking, auto-created chat channels
- **Companies** — Target company management with industry, size, and contact tracking. **AI-native typeahead** suggests existing org records + Claude lookups when adding a new company; **AI Enrich** fills missing fields (industry, size, location, founded year, HQ, competitors, key products) on existing records. Single-record and bulk-up-to-20 enrich.
- **Custom Fields** — Every org can extend the core schema with their own taxonomy via Settings → Custom fields (org admins; gated to `owner` / `admin` / `super_admin` / `deputy_admin`). Super-admins additionally see the same panel at Admin → Custom Fields. Text and dropdown field types, per `(org, entity_type)` namespace, applies to contacts / leads / companies / deals. Values live in a `custom_fields` dict on each entity row; values stay on rows when a definition is deleted (recreating the same key restores them). Filterable on every list page (Any / value / (Empty) for dropdowns, substring contains for text; multiple filters AND together).
- **Calendar** — Month/Week views with scheduled calls, task due dates, deal closes, custom events with entity linking, Google Calendar sync. **Per-event reminder lead-time** (None · 5 / 10 / 15 / 30 min · 1 hr · 1 day) — 60s cron fires an in-app `appointment_due` notification at the configured offset with a two-tone WebAudio chime + sonner toast (chime mutable per-user under Settings → Profile). Covers `calendar_events` + `scheduled_calls` + booking-confirm rows. **Recurring events** — Daily / Weekly / Monthly / Yearly + interval + end condition (Never / After N / On date) on the Create + Edit dialogs; instances expand at read time, edits to any occurrence update the series anchor. Clicking a task pill on the calendar deep-links into the full task edit dialog. Tasks/deals with date-only due_dates now render correctly in the all-day strip instead of piling up at the 07:00 row.
- **Campaigns** — Multi-channel (Email via Resend + Kit.com, Facebook, Instagram, LinkedIn), rich email editor (subject + Markdown body + live HTML preview + AI draft), AI-powered drafting (purpose + tone), channel picker on create, social campaigns linked to Listeners. **Per-recipient delivery tracking** with status pills (delivered, bounced, recipient_unknown, inbox_full, deferred, failed, replied, opted_out, pending) driven by Resend webhooks; aggregate counts denormalised on the campaign for cheap list rendering. **Send-batch semantics**: a "Send" only mails recipients whose row status is `pending` — already-delivered/bounced/replied rows are skipped, so adding new people to a sent campaign and pressing Send again only mails the new ones (no re-spam path). **Pending vs Deferred are separate aggregates**: the UI exposes a "Retry N deferred" button (orange band) distinct from "Send to N pending" so operator intent stays explicit. **Provider-safe pacing** — `RESEND_SEND_DELAY_S` (default 250ms ≈ 4 sends/sec) keeps batches under Resend's request-per-second limit. **Add to campaign from Lead detail** — single-lead button on the Lead detail dialog plus the existing bulk-list flow. **Closed-loop replies** — inbound replies are received via Resend Inbound, matched back to the recipient row OR to the underlying lead (via from-address) for transactional sends that bypassed campaigns, and surfaced inline with the full body (fetched from Resend's `/emails/receiving/{id}` API; HTML-only replies render sanitized via DOMPurify when no plain-text part is present). **Self-sourced loop guard** drops inbounds from our own sender domain (bounces, auto-replies, forwarded copies) so they don't overwrite `last_reply_from`. **BCC outbound capture** logs operator replies sent from their own mail client when they BCC `tako@tako.software`, with sender identification via user email or `email_aliases`. **AI suggestions on replies** classify intent (interested / objection / not_now / out_of_office / unsubscribe / question / other), draft a contextual response, and propose follow-up tasks; one-click **Send reply** (with `In-Reply-To` threading) and **Create task** (auto-linked to the lead/deal). **Soft delete + restore** — deleted campaigns + per-recipient history move to /trash; restoring brings the entire send history back intact. **Filter recipients by status** in the detail view. Sent campaigns can be renamed but content stays frozen post-send so history isn't rewritten.
- **Per-entity History** — Every Lead / Contact / Deal / Company detail dialog has a unified activity timeline aggregating emails sent (with full body + delivery status + provider error), tasks, deals, conversion events, chat activity, and creation/update anchors. Each row drills down to the granular trace; email rows link directly to the relevant Campaign detail.
- **Listeners** — AI social listening agents: keyword monitoring, Claude-powered hit classification (buying signal / complaint / question / mention / noise), confidence scoring, sentiment, suggested replies, auto task creation, digest reports, Meta Graph poller, Chrome extension device-code pairing
- **Files** — File upload with AI summarisation (PDF, DOCX, text, images), linked to any entity, one-click task creation from AI suggestions

### AI Features (Claude)
- **Lead Scoring** — AI assigns a 1–100 quality score based on profile completeness and signals
- **Lead Enrichment (web-grounded)** — Researches the lead's company via Anthropic web search and fills company info, tech stack, interests, recommended sales approach — with citations, per-field confidence, and Verified/Unverified marking. Falls back (clearly marked unverified) when web search is unavailable
- **Research Autopilot** — ICP profiles (Settings → Research) drive a weekly web-research discovery agent that stages deduped prospect suggestions for one-click conversion to leads; on-demand company/lead/deal dossiers (news, key people, talking points — cited); meeting-prep briefs auto-compiled before calendar meetings with CRM-matched attendees
- **Company Suggest + Enrich** — Typeahead while creating a company merges hits from your own data (existing companies + names referenced on leads/contacts) with Claude-powered "what is this company" lookups. Operator-confirmation required (AI suggestions tagged "Verify"). Existing company records get a one-click **AI Enrich** that fills empty fields without overwriting operator-set ones; bulk variant up to 20 at a time.
- **Email Drafting** — Personalised sales emails with tone/purpose selection, embedded in the campaign editor with live HTML preview
- **Reply Triage** — Inbound replies are auto-classified by Claude (intent + summary), get a drafted contextual response, and propose 0–3 follow-up tasks. Visible inline in the campaign detail expand row with one-click Send / Create Task.
- **Inbound Email Autopilot** — Every fresh inbound email is auto-attributed (sender matched to contacts/leads, contact links created), intent-classified (interested / question / meeting request / objection / support / invoice / unsubscribe / newsletter / spam), and — when a reply is warranted — arrives with an AI-drafted response in the sender's language built from thread + CRM context (booking link included for meeting requests). Concrete follow-ups land as pending task suggestions. Runs as a budgeted queue worker so AI never slows mail sync; org toggle under Settings → Email compliance.
- **AI Agents (agent core)** — In-platform Claude agents with real tool use (search CRM, read records, pipeline summary, the asker's tasks, file task suggestions — writes are always human-approved suggestions). Mention **@tako** in Team Chat for grounded answers; **Berny**, the chief-of-staff agent, runs on the same framework. Personas/models/toggles live per-org in Settings → AI Agents; every run is traced (tool calls, tokens, cost) and spend is feature-attributed against the daily cap.
- **Call Intelligence** — Voicemail and call transcripts are summarised (summary + sentiment) onto the call log, with follow-up suggestions filed for approval.
- **Lead Summary** — Comprehensive AI profile analysis
- **Smart Search** — Natural language search across all CRM data
- **Call Analysis** — AI feedback on recorded calls (score, strengths, improvements, next steps)
- **File Analysis** — Auto-summary and follow-up task suggestions on uploaded documents
- **Hit Classification** — Claude classifies social Listener hits in real time: category, confidence, sentiment, suggested reply
- **Digest Reports** — AI-generated daily/weekly summaries of Listener activity with recommended actions
- **Included AI** — Licensed TAKO instances ship with unlimited access to Claude via the platform key. Bring-your-own Anthropic key is also supported per organization (Settings → Integrations).

### Automation / Campaign Agent
- **Daily campaign-runner** — Mon-Fri 07:00 Europe/London cron via APScheduler iterates active campaigns, decides which leads are due for a step-2/3/4 follow-up, drafts each message in the operator's voice via Claude (`tako_ai_text`), and creates a ready-to-send Task with the draft body in the description. The operator opens TAKO, sees the day's drafts, copy-pastes to LinkedIn — done in 30 minutes.
- **Sequence config** — Per-campaign `sequence_config_key` (e.g. `outsourcing_beachhead_v1`) defines step count, delays (e.g. +0/+4/+9/+14 days), prompt templates, and voice rules. Adding a new campaign type = add a SequenceConfig + reference it from the Campaign doc. No new code per campaign.
- **Daily per-step cap** — `agent_daily_step_limit` (default 50) caps tasks-created-per-step-per-day. Leads are pre-sorted by `ai_score` desc (with a contactable + recency tiebreak) so the cap surfaces the highest-quality leads first; the rest roll forward to tomorrow. Prevents day-1 blast scenarios where 250+ tasks land in a single morning.
- **Triage tools** — One-shot admin actions in Settings → Organization → Campaign Agent:
  - **Bootstrap campaign** — create / update a campaign doc with `sequence_config_key` + `tag_filter` + daily cap.
  - **Run daily agent now** — manual trigger with a 90-second poll loop and inline result stats.
  - **Triage agent tasks** — keep top N by `ai_score`, soft-delete the rest (recoverable from /trash). Has a Preview mode.
  - **Cleanup link-less tasks** — soft-delete agent todos whose description has no http(s) URL (typically drafts for leads with no `linkedin_url`).
- **Idempotency** — Re-running the cron on the same day creates no duplicate tasks (Step N regex match on titles). Re-pressing "Send daily agent now" is safe.
- **Audit trail** — Every run writes a `db.agent_runs` row (`kind: campaign_runner | triage | triage_no_link`, stats, run_at, started_by_user_id) so operators can see what the agent did over time.
- **Out of scope (deferred)** — `reply_triager` (overlaps with the existing inbound email triage; will revisit when the LinkedIn channel ships via Unipile so the new path can be LinkedIn-only) and `prospect_discoverer` (source classes are documented stubs).
- **Source scaffold + architecture docs** — Reference scaffold (architecture write-up, integration guide, agent + scheduled-task Python files) lives at [`docs/Automation/`](docs/Automation/). Files promoted from there into `backend/agents/` and `backend/scheduled/` are the live ones; the scaffold copies stay as "what was originally proposed" reference for the deferred pieces above.

### Communication
- **Inbox (TAKO Mail Phase 1)** — Native email integration via IMAP / SMTP. Three-pane view at `/inbox`: account switcher · chronological list · selected message. **Setup wizard** at `/settings/email/add` with provider-specific guides for Gmail · Outlook / Microsoft 365 · iCloud · Custom (Fastmail / Migadu / your own server). **Edit account settings** via pencil icon on each row at `/settings/email` (also reachable as a dedicated **Email** tab on `/settings`): change display name, IMAP/SMTP host + port + SSL/TLS, signature HTML, or rotate the password. Server re-runs the IMAP + SMTP probe whenever connection-shaping fields change so a bad save can't break the poller; password rotation re-encrypts under the same `credentials_id` so the FK on account + email_links rows survives. **App-password flow**, no OAuth in Phase 1 (Phase 3 lands `Sign in with Google` + Microsoft Graph push). **Personal accounts + admin-only shared mailboxes** with the assignment / "Create task from email" surface for distributing inbound. **AI summary** on every inbound message (1-2 sentences via Haiku, in the recipient's TAKO language; cap-block returns empty rather than crashing the cycle). **Manual "Log to CRM"** action — explicit user choice (strategy memo decision: no auto-attach), multi-select picker over contacts / deals / companies. **Compose** with tiptap rich-text (bold / italic / link / bullet / numbered), variable-template-aware (`{{first_name}}` / `{{company}}` etc. — same vocabulary the campaign sender uses), localStorage draft persistence keyed by composeId so accidentally closing doesn't lose work. **Sent-folder sync** via IMAP APPEND so messages composed in TAKO appear in your real Sent folder threaded correctly. **Strategy memo decisions locked**: metadata + AI summary stored, NEVER full bodies (fetched on demand from your IMAP server when opened); no relays (your SMTP, your deliverability); no tracking pixels, ever. **30s polling** on `/inbox` while on the page; **60s active-cadence / 5min sleeper** APScheduler poll cycle in the background. **Server-side delete reconcile** — UID-compare every cycle so deletes in Apple Mail / Gmail web propagate to TAKO; **Refresh button triggers an on-demand poll** (not just a re-paint) so the gap closes within seconds rather than 60s. **Bell pings** on inbound from a known CRM contact / lead (newsletters / unknown senders silent — no bell-spam). **Backfill skips AI summary + bell** so cold-start doesn't burn the AI cap or dump 200 notifications on day-one connection. **Hover-row trash + detail-header Delete** with optional "also remove from real mailbox" checkbox (IMAP `STORE \Deleted` + EXPUNGE). **Compose pre-fills To** from the linked contact (preferred) → linked lead → primary contact at the company, with full overwrite. Master credential key (`TAKO_EMAIL_CRED_KEY`, libsodium secretbox) is non-negotiable: backend refuses to start without it. Production setup walkthrough at [`docs/email/PRODUCTION_SETUP.md`](docs/email/PRODUCTION_SETUP.md).
- **Call Scheduling** — Calendar-based scheduling with configurable reminders
- **Telephony (live)** — Inbound calls to the org number: configurable greeting (EN/DE TTS) → forward to a team phone (recording notice announced before connecting when call recording is enabled — two-party consent) → voicemail fallback. Missed calls and voicemails create high-priority callback tasks matched to the caller's lead/contact with the recording attached; transcripts flow through an AI analyzer into call summaries + suggested follow-ups. All Twilio webhooks signature-validated. Configure under Settings → Telephony; outbound click-to-call from lead records.
- **Google Calendar** — OAuth integration, two-way sync, events displayed in calendar view
- **Team Chat** — Real-time messaging with five channel kinds:
  - **DMs** — strict 1:1, only the two participants can read (admins blind in the UI; audit log still records when `CHAT_AUDIT_ENABLED`)
  - **Private groups** — invitation-only, members + org admins; up to 10 members
  - **Public groups** — anyone in org can read; only members + moderators write
  - **Context channels** — auto-created per Lead / Deal / Task / Company / Project
  - **General** — org-wide read+write
  Promote a DM into a named private group when the conversation needs more people. Moderators (subset of members; org admins implicit) can invite, expel, and appoint other moderators.
- **Chat unread tracking** — per-channel, per-user `last_read_at` drives a small red badge next to every channel name in the sidebar (capped at 9+, name turns bold) and lights up the topbar bell. The bell dropdown gets a single pinned "N new chat messages" row that links into Team Chat — no per-message notification spam. Opening a channel marks it read; sitting in it keeps it read while new messages stream in.
- **Chat Audit** — Every channel/message event recorded in `chat_audit` (sent, edited, deleted, member added/removed, moderator added/removed, channel created/promoted/dm-opened). Forensic-grade super-admin viewer at `/admin/chat/audit` includes DM content for investigations. Disable per-region with `CHAT_AUDIT_ENABLED=false`.
- **Chat Archive** — Admins can archive channels, collapsible sidebar sections
- **Capture** — Live camera (getUserMedia) or file upload for business card scanning and lead creation

### Social Listening (Listeners)
- **Listener** — Per-campaign agent monitoring Facebook groups/pages for keyword matches
- **Sources** — Discover, approve, and manage group/page sources; `discover_groups` agent skill files tasks for human review
- **Hits** — Ingested posts/comments classified by Claude with confidence, sentiment, matched keywords, suggested reply
- **Reports** — AI digest with top hits, trends, and recommended actions (`generate_report` skill)
- **Pairing** — Chrome extension device-code pairing (token only issued to UI, never to extension)
- **Poller** — APScheduler jobs: `listener_poll_meta_pages` (per cadence), `listener_generate_digest` (daily/weekly), `listener_rescore_hits` (hourly)
- **Webhooks** — Generic receiver at `/api/webhooks/{provider}/{org_id}` for Meta and Chrome extension payloads

### Platform
- **Organizations** — Roles: member, admin, owner, deputy_admin, super_admin, support. Self-hosted single-tenant or multi-tenant deployment
- **Per-org Integrations** — Each org manages their own API keys (Resend, Kit, Twilio, Google, Anthropic, Meta)
- **Customizable Stages** — Admins can add/remove/rename deal stages and task steps per org. **Drag-to-reorder** with `@hello-pangea/dnd` — the saved `order` field flows through to the Deals + Tasks kanban so column order matches Settings exactly. Custom stages get a neutral colour fallback on the kanban; default stages keep their tuned palette + win-probability mapping.
- **Trash & Recoverability** — Soft-delete is the default for leads, contacts, deals, tasks, companies, projects, campaigns, and calendar events. Deleting an entity sets `deleted_at` + `deleted_by` and writes an `audit_events` row (entity_type / entity_id / actor / timestamp). Live list queries filter `deleted_at: null` so trashed records vanish from operator views; sidebar "Trash" page lists what's recoverable. Member sees their own deletes; admin sees the org-wide trash. One-click restore brings the entity back exactly as it was. Cascade-deleted children (e.g. campaign_recipients on lead delete) are not auto-recreated on restore — relinking is the operator's responsibility.
- **Team Invitations** — Invite via link, email, or CSV import — unlimited users on every licence
- **Partner Programme** — Referral partners earn €500 per sale. Agency partners earn an additional €750 onboarding commission per customer. Two-tier, no MLM chains
- **Self-serve Demo** — Two-stage flow:
  1. **`/demo` (public marketing page)** — intent-capture landing for cold traffic. Primary CTA books a 15-minute walkthrough with the operator via the native booking page; secondary "notify me when the sandbox is live" email-capture posts to `POST /api/newsletter` with `source: 'demo_page'`. Cold prospects always land somewhere useful instead of a generic signup form.
  2. **Trial org spin-up (post-signup)** — after standard signup + email verification, `SetupOrgPage` offers "Try TAKO free for 14 days" which calls `POST /api/demo/create` and seeds a fresh org with sample leads, deals, tasks, and calendar events. On expiry the tenant soft-locks: reads stay open, writes are blocked middleware-wide until the user upgrades. Super admins can extend, expire, or purge demos from the admin panel.
- **Distribution System** — `scripts/build-distribution.sh` produces a self-contained customer tarball (source + Docker + customer `.env.example` + install guide + backup script). Platform-only code — UNYT/crypto, self-serve demo, partner payouts, super-admin UI, distribution tooling — is stripped via sentinel-wrapped regions (`DEMO_BEGIN`/`PLATFORM_BEGIN` markers) so customers never see it.
- **Release Pipeline** — Tagging `vX.Y.Z` on `main` runs `.github/workflows/release.yml`, which builds the distribution tarball, generates SHA-256 checksums, creates a GitHub Release, and posts a notification to `/api/admin/releases/notify` so the changelog and in-app update checker update automatically.
- **In-app Update Checker** — Customer instances call `/api/system/update-check` once per day against the platform; if a newer version is tagged, a banner appears in the admin UI linking to the changelog and download page. Results are cached per-org for 24h to stay polite.
- **Cron Health Watchdog** — APScheduler event listeners stamp every job's last run into `db.job_runs_latest` (status / timestamp / truncated traceback / consecutive-error streak). A 5-minute `cron_health_watchdog` job checks two signals — `next_run_time` >10 min stale (scheduler wedged) or `consecutive_errors >= 3` (job failing every cycle) — and bell-pings every super-admin with a `job_health_alert` notification. 1-hour dedup per `job_id`. `GET /api/admin/jobs/health` returns per-job status for ad-hoc inspection.
- **GDPR Compliance** — Self-service data export (`/api/gdpr/export-my-data`), account deletion with grace period (`/api/gdpr/request-deletion` / `/api/gdpr/cancel-deletion`), and a public DPA endpoint at `/api/legal/dpa`.
- **UNYT Token Payments** — Pay with UNYT on Arbitrum via MetaMask or via [UNYT.shop](https://unyt.shop)
- **License & Billing** — One-time purchase or installment plans via Stripe. UNYT token payments via MetaMask or UNYT.shop. Optional annual maintenance renewal
- **PWA** — Installable on iOS, Android, and desktop
- **i18n** — English and German language support with toggle
- **API Keys and Webhooks** — Programmatic access for n8n, Notion, Zapier, and custom integrations
- **Reporting Engine** — User performance, pipeline forecasts, activity logs, CSV export
- **Onboarding & Support** — In-app onboarding checklist (localStorage-persisted), training modules, FAQ, in-app support ticket form, legal docs

---

## Security

TAKO ships with a defense-in-depth posture suitable for production use by European teams handling customer data. Key controls:

- **Email verification** — New accounts must verify via a tokenized link before they can log in; unverified accounts are blocked at login.
- **Dual-token JWT** — Access tokens expire in 15 minutes (`JWT_ACCESS_EXPIRY_MINUTES`), refresh tokens last 7 days (`JWT_REFRESH_EXPIRY_DAYS`). Refresh is rotated on use; the legacy single-token flow (`JWT_EXPIRY_HOURS`) is still honoured for in-flight sessions during upgrade.
- **Login rate limiting** — Password and 2FA endpoints throttle per-IP to blunt credential stuffing.
- **Password reset** — Tokens are single-use with a 1-hour TTL; reset emails never leak whether an account exists.
- **Stripe webhook hardening** — `/api/webhook/stripe` verifies the `Stripe-Signature` header against `STRIPE_WEBHOOK_SECRET` and de-duplicates events by `event.id` so retries are safe.
- **Invoice XSS hardening** — HTML invoices escape every user-controlled field before rendering; no template injection surface.
- **Session termination on org deletion** — When an organization is deleted, all associated sessions are revoked and users are force-logged-out on next request.
- **VAT-aware invoicing** — `COMPANY_VAT_NUMBER` is emitted on every invoice; missing VAT keeps the obvious placeholder `GB000000000` so no invoice silently ships without one.
- **Tenant isolation** — All CRM queries scope to the caller's `organization_id`; cross-tenant access is not possible via the REST or v1 external API.

See [`docs/VPS-HARDENING.md`](docs/VPS-HARDENING.md) for the host-level companion guide (unprivileged deploy user, SSH key-only auth, UFW, fail2ban, automatic security updates, MongoDB bind-address).

---

## Operations

- **Health check** — `GET /api/health` returns a JSON status payload covering MongoDB reachability, email sender readiness, Sentry init, and **Stripe + Resend webhook configuration**. Returns **200** when MongoDB is reachable, **503** when it is not — use it as the target for an external uptime monitor. Sample response: `{"status":"ok","mongo":"connected","email":"configured","sentry":"disabled","stripe_webhooks":"not_configured","resend_webhooks":"configured"}`.
- **Error monitoring** — Set `SENTRY_DSN` (backend `.env`) and `REACT_APP_SENTRY_DSN` (frontend build env) to enable Sentry. Both are optional soft dependencies — the backend and frontend start without error if the SDK isn't installed or the DSN isn't set. `ENVIRONMENT` / `REACT_APP_ENVIRONMENT` tag events so staging and prod don't pollute each other.
- **Backups** — `scripts/backup-mongo.sh` dumps MongoDB, tars the output, retains 7 days locally, **and uploads offsite** when configured. Two offsite paths:
  - **S3-compatible** (AWS, MinIO, Wasabi, Backblaze B2, Cloudflare R2): set `BACKUP_S3_BUCKET`, `BACKUP_S3_PREFIX` (optional), `BACKUP_S3_ENDPOINT` (optional, for non-AWS), `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`. Requires the `aws` CLI on the host.
  - **rclone** (Dropbox, Google Drive, etc.): set `BACKUP_RCLONE_REMOTE=dropbox:tako-backups` (or any configured rclone remote). Requires `rclone` on the host.
  Both are optional and no-op safe when unset — the local backup runs unchanged. Offsite failure does NOT fail the script (local copy is still good); the WARN line is what cron's `MAILTO` catches.
- **Restart helper** — `scripts/restart-backend.sh` recreates the backend container with the right compose-file chain (auto-detects host nginx and adds the `docker-compose.host-nginx.yml` overlay so port 8001 stays published). Use this — not `docker compose restart` — when picking up new env values; compose loads `env_file` at container creation, not on restart.
- **Backend auto-deploy** — `.github/workflows/deploy-backend.yml` deploys to the production VPS on every push to `main` that touches `backend/**`, `docker-compose*.yml`, `Caddyfile`, `scripts/deploy-production.sh`, or the workflow itself. Requires `DEPLOY_SSH_HOST`, `DEPLOY_SSH_USER`, `DEPLOY_SSH_PORT`, `DEPLOY_SSH_KEY`, `DEPLOY_SSH_KNOWN_HOSTS` in repo Secrets. Smoke-tests `/api/health` after the deploy script runs.
- **Resend webhook ingestion** — `POST /api/webhooks/resend` receives delivery events (sent, delivered, delivery_delayed, bounced, complained, opened, clicked, failed, suppressed). Svix-style HMAC-SHA256 signature verified against `RESEND_WEBHOOK_SECRET`; idempotent on `svix-id` via `processed_resend_events` (14-day TTL). Refuses unsigned events. Inbound replies receiver at `POST /api/webhooks/resend/inbound` (requires Resend Inbound DNS + dashboard config). **System-mail / DMARC filter** — inbound mail from auto-mailer addresses (`mailer-daemon`, `postmaster`, `bounce(s)`, `noreply` / `no-reply`, `dmarcreport`, `dmarc-feedback`, `auto-reply`, `autoresponder`, `abuse`, sub-addressed variants) and mail flagged by RFC 3834 / 2076 headers (`Auto-Submitted: auto-replied|auto-generated`, `Precedence: bulk|list|junk`) is silently swallowed — logged + recorded as `inbound.system-mail` in `processed_resend_events`, but no inbound forward, no bell, no "could not match a campaign or lead" alert. DMARC aggregate reports (Microsoft, Google, Yahoo) are the canonical case.
- **One-click ops workflows** — Trigger from the Actions tab (or `gh workflow run …`):
  - **`set-env.yml`** — surgical `.env` upsert + backend recreate; one key per call. Awk-based, won't touch unrelated keys.
  - **`recreate-backend.yml`** — restart the backend container with the right compose-file chain (host-nginx aware).
  - **`setup-stripe-products.yml`** — idempotent Stripe product + price setup; reads existing products by `metadata.tako_sku` so re-running is safe. Probes all six Stripe envs (`STRIPE_API_KEY`, `STRIPE_WEBHOOK_SECRET`, four `STRIPE_PRICE_*`) and reports presence without leaking values.
  - **`migrate-contacts-companies.yml`** — runs `scripts/migrate_contacts_to_company_join.py` (Phase 3 backfill of the M:N join collection). Default = dry-run; tick `apply` to write.
  - **`triage-tasks-once.yml`** — admin tools the Settings UI exposes (link-less agent-task cleanup + keep-top-N-by-ai-score). Idempotent.
  - **`verify-campaign-runner.yml`** — APScheduler health probe; reads `db.apscheduler_jobs` for next-run-time + last 5 `db.agent_runs` rows. Optional `fire_now=true` to trigger `run_daily_campaigns()` once.
  - **`smoke-probe.yml`** — generic ops probe: paste a Python snippet, runs inside the prod backend container with `from server import db` in scope. Used for one-off "did the row land?" checks.
  - **`rotate-stripe-key.yml`**, **`set-vps-secret.yml`**, **`env-cleanup-malformed.yml`** — operational hygiene utilities.
- **Support tickets** — Users submit issues from any page via `POST /api/support/ticket`; tickets land in the super-admin panel with user, org, and environment metadata auto-attached. Public (unauthenticated) contact enquiries go through `POST /api/support/contact`.
- **Host hardening** — [`docs/VPS-HARDENING.md`](docs/VPS-HARDENING.md) walks through IONOS/VPS first-boot steps: unprivileged deploy user, SSH key-only auth, UFW, fail2ban, unattended-upgrades, MongoDB bound to localhost.
- **Campaign operating artifacts** — Per-campaign durable knowledge (reusable copy, source-list notes, batch logs, lessons learned) lives at [`docs/campaigns/`](docs/campaigns/). One folder per campaign (e.g. `campaign-1-aios-investor-dd/`). Stored close to TAKO execution truth so deliverability fixes, suppression rules, and angle iterations stay close to the code that runs them.

---

## Pricing

TAKO is a self-hosted CRM. Purchase once, deploy on your own infrastructure, own your data forever.

| Option | Price | Notes |
|--------|-------|-------|
| **One-time** | €5,000 | Single payment, perpetual licence |
| **12-month installment** | €500 / month × 12 | €6,000 total, perpetual licence after final payment |
| **24-month installment** | €300 / month × 24 | €7,200 total, perpetual licence after final payment |
| **UNYT Token** | Pay in UNYT on Arbitrum | Perpetual licence, any plan (see [UNYT.shop](https://unyt.shop)) |

**All licences include:**
- Unlimited users
- All CRM features (Leads, Contacts, Deals, Tasks, Projects, Campaigns, Listeners, Files, Calendar, Chat, Calls)
- Unlimited AI via Claude (platform key included)
- All integrations (Google, Resend, Kit, Twilio, Meta, Stripe)
- API access + webhooks
- First year of updates and maintenance

**Maintenance renewal** — €999 per year (optional). Renewing keeps you on the latest version with priority support. If you skip renewal your instance keeps running — you simply stop receiving updates.

**Partner Programme** — Agencies and consultants earn €500 per customer sale, plus €750 per agency onboarding. Public marketing landing at `/partners` (founder-led EN/DE, six sections incl. live active-partner count + FAQ); authenticated dashboard at `/partners/dashboard` for active partners (referral link, sales, balance, agency-upgrade form). New agency applications email `florian@fintery.com` (override via `PARTNER_ADMIN_NOTIFY_EMAIL`) so the operator can triage from the inbox without opening the admin UI.

### Stripe billing setup (one-time, per instance)

The TAKO backend has the four SKUs hard-coded as `tako_selfhost_once` / `tako_selfhost_12mo` / `tako_selfhost_24mo` / `tako_maintenance_yearly` and looks up the matching Stripe `price_…` ids via env vars. To wire a fresh instance:

1. **Set `STRIPE_API_KEY`** on the VPS — Stripe → Developers → API keys → Restricted (or Secret). Use the `set-env.yml` workflow:
   ```
   gh workflow run set-env.yml -f key=STRIPE_API_KEY -f value=sk_live_…
   ```
2. **Create the four products + prices** — `gh workflow run setup-stripe-products.yml -f apply=true`. Idempotent (looks up existing products by `metadata.tako_sku`); re-running won't duplicate. Captures `STRIPE_PRICE_ONETIME` / `_12MO` / `_24MO` / `_MAINTENANCE` and feeds each one through `set-env.yml` automatically. Restarts the backend.
3. **Register the webhook** — Stripe → Developers → Webhooks → Add endpoint:
   - Endpoint URL: `https://<your-domain>/api/webhook/stripe`
   - Events: `checkout.session.completed`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`
   - Copy the **signing secret** (`whsec_…`) and set:
     ```
     gh workflow run set-env.yml -f key=STRIPE_WEBHOOK_SECRET -f value=whsec_…
     ```
4. **Verify**: `curl -s https://<your-domain>/api/health | jq '.stripe_webhooks'` should flip from `not_configured` → `configured`. The Pricing page (`/pricing`) → "Buy now" buttons hit `POST /api/subscriptions/checkout` and redirect to a real Stripe Checkout.

The four `STRIPE_PRICE_*` ids are also surfaced in `backend/.env.example` — if you prefer manual product creation in Stripe's dashboard, just paste the ids there and skip step 2.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Tailwind CSS, Shadcn/UI, @hello-pangea/dnd, ethers.js |
| Backend | Python 3.11, FastAPI, Motor (async MongoDB) |
| Database | MongoDB |
| Auth | Native Google OAuth 2.0 → dual-token JWT (15-min access / 7-day refresh; legacy 7-day single-token still honoured) |
| AI | Anthropic Claude (`claude-sonnet-4-6`) via `anthropic` Python SDK |
| Email | Resend (primary, configurable `SENDER_EMAIL` — founder-led `florian@tako.software` is the production default; `REPLY_TO_EMAIL` routes replies to a shared mailbox), Kit.com (optional subscriber lists). Per-message pacing via `RESEND_SEND_DELAY_S` (default 250ms ≈ 4 sends/sec) keeps batches under Resend's request-per-second cap. |
| Payments | Stripe + UNYT Token (Arbitrum) |
| Calling | Twilio Voice API (inbound forward/voicemail/recording + outbound click-to-call) |
| File parsing | pypdf (PDF, 30 pages), python-docx (DOCX) |
| Job scheduler | APScheduler 3.x with MongoDB jobstore (demo expiry, listener polling, digests, rescoring, **daily campaign-runner**) |
| Error monitoring | Sentry (backend + frontend, both soft dependencies) |
| i18n | Custom `useT` hook with JSON locale files (English + German) |
| Deployment | Docker Compose (mongo + backend + frontend + Caddy), Caddy reverse proxy with automatic Let's Encrypt SSL, optional `docker-compose.host-nginx.yml` override when host nginx is already on :80/:443. Backend `VERSION` file stamped on every deploy via `git describe --tags --always --dirty` so the Settings → Updates panel shows the live version (e.g. `v2026.04.26-47-g67f44f6`). |
| Release | GitHub Actions (`.github/workflows/release.yml`) + `scripts/build-distribution.sh` |

---

## Distribution

TAKO is distributed to customers as a stripped source tarball — they run the same code we do, minus the platform-only plumbing.

### Build script

`scripts/build-distribution.sh [version] [--client <name>]` produces `dist/tako-crm-<version>[-<client>].tar.gz` plus an accompanying `.sha256`, `VERSION`, and `dist/manifest.json`. What it ships:

- Full backend and frontend source (minus stripped regions) + a pre-built frontend bundle
- `docker-compose.production.yml` (Caddy TLS + Mongo + backend + frontend), dev `docker-compose.yml`, `Caddyfile.example`
- Customer-facing env templates for backend (incl. Resend webhook secrets; no platform keys, no Stripe price IDs) and frontend
- `LICENSE.md` stamped with the client name on `--client` builds
- `scripts/backup-mongo.sh` + data migrations; install guide as the package `README.md`
- `docs/VPS-HARDENING.md`, `docs/BCC-OUTBOUND-CAPTURE.md` (whitelist — everything else in `docs/` stays internal)

What it strips:

- **Platform-only pages** (landing, pricing, demo, partners, download, changelog, subscription-success) — overwritten with redirect stubs so routes survive without the content
- **UNYT / crypto payment routes and residue** — customers see Stripe only
- **Self-serve demo system**, founder-email super-admin checks, internal docs/CI/tests/configs

Inline platform-only code is stripped via sentinel markers `DEMO_BEGIN`/`DEMO_END` and `PLATFORM_BEGIN`/`PLATFORM_END` (matched as `(?P<kind>DEMO|PLATFORM)_BEGIN` … `(?P=kind)_END`) — the pairs must stay matched. A verification gate (brand-residue grep + required/forbidden file checks) fails the build rather than emit a dirty package. New platform-only pages must be added to `PLATFORM_STUB_PAGES` in the script.

Per-client frontend variants: put whole-file overrides under `clients/<name>/overlay/**` (see `clients/README.md`) and build with `--client <name>`. Publishing to the client downloads page (`tako.software/downloads`): `scripts/publish-downloads.sh`. Process docs: `docs/handover/RELEASE-PROCESS.md`.

### Release workflow

1. Bump the version in `backend/server.py` (`TAKO_VERSION`) and commit.
2. `git tag vX.Y.Z && git push --tags`.
3. `.github/workflows/release.yml` runs: builds the tarball, computes SHA-256, creates the GitHub Release with both artifacts attached, and POSTs to `/api/admin/releases/notify` (super-admin key) so `/api/releases` and `/changelog` pick it up immediately.
4. Customer instances hit `/api/system/update-check` on their next daily tick, see the new version, and surface a banner linking to `/download` with a fresh signed token from `POST /api/license/download`.

---

## Getting Started

### Prerequisites
- Python 3.11+, Node.js 20+, MongoDB, Docker (recommended)

### Docker (recommended)

```bash
git clone https://github.com/fbjk2000/tako-core.git
cd tako-core
cp backend/.env.example backend/.env   # fill in your values
docker compose up -d
```

App available at `http://localhost:3000` (frontend) and `http://localhost:8001` (backend API).

### Manual

```bash
# Backend
cd backend
pip install -r requirements.txt
cp .env.example .env
uvicorn server:app --host 0.0.0.0 --port 8001 --reload

# Frontend
cd frontend
npm install --legacy-peer-deps
REACT_APP_BACKEND_URL=http://localhost:8001 npm start
```

---

## Environment Variables

**Backend** (`/backend/.env`) — see [`backend/.env.example`](backend/.env.example) for the full template.

**Required** (TAKO will not start without these):

```env
# Database
MONGO_URL=mongodb://localhost:27017
DB_NAME=tako_production

# Auth — dual-token JWT
JWT_SECRET=your_long_random_secret
JWT_ALGORITHM=HS256
JWT_ACCESS_EXPIRY_MINUTES=15
JWT_REFRESH_EXPIRY_DAYS=7
JWT_EXPIRY_HOURS=168          # legacy single-token flow; still honoured

# URLs
FRONTEND_URL=https://yourdomain.com
PUBLIC_URL=https://yourdomain.com

# Google OAuth (login)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret

# AI (platform key — all licensed orgs use this automatically)
ANTHROPIC_API_KEY=sk-ant-...

# TAKO Mail — credential encryption master key (32 random bytes, base64).
# Backend refuses to start without this. Generate with:
#   python -c 'import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())'
# Full setup walkthrough: docs/email/PRODUCTION_SETUP.md
TAKO_EMAIL_CRED_KEY=

# Invoicing
COMPANY_VAT_NUMBER=GB123456789
```

**Optional** — set only the integrations you actually use:

```env
# Email (transactional + campaigns)
RESEND_API_KEY=re_...
SENDER_EMAIL=noreply@yourdomain.com
KIT_API_KEY=
KIT_API_SECRET=

# Calling
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_FROM=+44...

# Social Listening (Facebook / Instagram)
META_APP_ID=your_meta_app_id
META_APP_SECRET=your_meta_app_secret

# Payments (only needed if you resell TAKO or accept Stripe payments)
STRIPE_API_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ONETIME=price_...
STRIPE_PRICE_12MO=price_...
STRIPE_PRICE_24MO=price_...
STRIPE_PRICE_MAINTENANCE=price_...

# Error monitoring (soft dependency — safe to leave blank)
SENTRY_DSN=
ENVIRONMENT=production

# Distribution output dir (platform-only — stripped from customer builds)
DISTRIBUTION_DIR=./dist
```

**Frontend** (`/frontend/.env`):

```env
REACT_APP_BACKEND_URL=https://tako.software
REACT_APP_SENTRY_DSN=               # optional
REACT_APP_ENVIRONMENT=production    # optional
```

---

## Google OAuth Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → APIs & Services → Credentials
2. Create an OAuth 2.0 Client ID (Web application)
3. Add Authorised Redirect URIs:
   - `https://yourdomain.com/api/auth/google/login/callback` (login)
   - `https://yourdomain.com/api/calendar/google/callback` (calendar sync)
4. Set `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` in your backend `.env`

---

## AI Setup

TAKO uses **Anthropic Claude** for all AI features.

Set `ANTHROPIC_API_KEY` in your `.env` file. All AI features are included in your TAKO licence — no token limits, no trial periods. Every licensed organization uses the platform key automatically.

Admins can optionally switch an organization to its own Anthropic key in **Settings → Integrations → AI / LLM**.

---

## Social Listeners Setup

Listeners require a Meta app with **Page Public Content Access** permission (requires Meta app review — allow several weeks).

1. Create a Meta app at [developers.facebook.com](https://developers.facebook.com)
2. Add `META_APP_ID` and `META_APP_SECRET` to your backend `.env`
3. Run the Meta OAuth flow from **Settings → Integrations → Meta** to connect an account
4. Create a campaign with channel type `facebook`, then create a Listener on that campaign
5. *(Optional)* Pair the Chrome extension via **Listeners → Pair Extension** for passive browser-based ingestion

> **Note**: The Chrome extension is a separate repo (`tako-chrome-extension`). Device-code pairing is fully wired on the backend.

---

## Database Migrations

After deploying, run any pending migration scripts:

```bash
# Backfill channel_type on existing campaigns
python scripts/migrate_campaigns_add_channel.py
```

---

## API Reference

> **Base URL**: `https://yourdomain.com/api`

All endpoints require: `Authorization: Bearer <jwt_token>` unless otherwise noted.

### Automation & Agents (Automation-Max, July 2026)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/dashboard/my-day` | Personal morning payload: focus tasks, emails awaiting reply, today's meetings, my pipeline, badges |
| GET | `/api/dashboard/command` | Admin-only management aggregates: pipeline by owner, 7d activity, agent spend, team load, 14d series |
| GET/PUT | `/api/agents/configs[/{agent_key}]` | Admin: list/edit per-org agent configs (enabled, persona, model). Blank persona resets to default |
| GET/PUT | `/api/settings/telephony` | Admin: forward number, recording (consent-announced), voicemail, language, greeting |
| PATCH | `/api/email/{email_id}/autopilot/dismiss` | Hide an autopilot reply draft (classification chips stay) |
| GET | `/api/email/inbox?needs_reply=1` | Inbox filtered to autopilot-flagged inbound awaiting a reply |
| POST | `/api/webhooks/twilio/…` | Twilio voice webhooks (inbound, dial-action, call-status, recording-status, transcription, record-done). Unauthenticated but X-Twilio-Signature-validated |
| GET/POST/PUT/DELETE | `/api/research/icp-profiles[/{icp_id}]` | ICP profiles (member-readable; writes admin-only; max 10/org) |
| GET | `/api/research/prospects?status=` | Discovery-staged prospect suggestions (default pending) |
| POST | `/api/research/prospects/{id}/accept` | Convert prospect → lead (source `research_discovery`); 409 if already actioned |
| POST | `/api/research/prospects/{id}/dismiss` | Dismiss a prospect (never re-suggested) |
| POST | `/api/research/dossier/{entity_type}/{entity_id}` | Queue a dossier compile (company/lead/deal; dedups to in-flight one) |
| GET | `/api/research/dossiers/{entity_type}/{entity_id}` | Dossier history for a record, newest first |
| GET | `/api/approvals[?kind=&count_only=1]` | Unified pending-approvals feed: task suggestions (mine), autopilot drafts (inbox-visible), prospects (org). Read-only; actions stay on the per-type endpoints |
| GET/POST/DELETE | `/api/webhooks[/{id}]` | Outbound webhooks. Create returns the signing `secret` ONCE; deliveries signed `X-Tako-Signature: sha256=<HMAC-SHA256 of body>`; https-only, private/loopback hosts rejected; retried once; last 50 deliveries logged per hook |
| POST | `/api/webhooks/{id}/test` | Fire a signed `webhook.test` event at one hook, result inline |

### Leads

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/leads` | List leads. Params: `status`, `source`, `tags` (csv, ANY-match), `q` (search across name/email/company/title), `assigned_to`, `has_email`, `campaign_id`, `not_in_campaign_id`, `limit`, `offset`. Returns `X-Total-Count` header |
| GET | `/api/export/{entity}` | CSV export for `leads` / `contacts` / `companies` / `deals` / `tasks` — same filter params as the list endpoints, streams all rows, UTF-8 BOM |
| GET | `/api/duplicates/{entity}` | Duplicate groups (email + normalized-name) for `leads` / `contacts` / `companies` |
| POST | `/api/duplicates/{entity}/merge` | Merge `duplicate_ids[]` into `survivor_id` — fills empty fields, unions tags, relinks references, losers go to Trash |
| POST | `/api/leads` | Create lead |
| GET | `/api/leads/{lead_id}` | Get single lead |
| PUT | `/api/leads/{lead_id}` | Update lead |
| DELETE | `/api/leads/{lead_id}` | Delete lead (cascades into `campaign_recipients`) |
| GET | `/api/leads/tags` | All distinct tags currently on this org's leads |
| POST | `/api/leads/import-csv` | Import from CSV |
| POST | `/api/leads/{lead_id}/score` | AI score |
| POST | `/api/leads/{lead_id}/enrich` | AI enrich |
| POST | `/api/leads/{lead_id}/convert-to-contact` | Convert to contact |

### Contacts / Deals / Tasks

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/contacts` | List. Params: `source`, `industry`, `decision_maker`, `tags` (csv), `q`, `has_email`, `campaign_id`, `not_in_campaign_id` |
| POST | `/api/contacts` | Create |
| GET/PUT/DELETE | `/api/contacts/{id}` | Read / update / delete (delete cascades into `campaign_recipients`) |
| GET | `/api/contacts/tags` | All distinct tags |
| POST | `/api/contacts/import-csv` | Import from CSV |
| GET/POST | `/api/deals` | List / create |
| PUT | `/api/deals/{id}/stage` | Move stage |
| GET | `/api/deals/tags` | All distinct tags |
| GET/POST | `/api/tasks` | List (`status`, `assigned_to`, `project_id`) / create |
| POST | `/api/tasks/{id}/comments` | Add comment |

### Companies

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/companies` | List companies for the org |
| POST | `/api/companies` | Create |
| GET/PUT/DELETE | `/api/companies/{id}` | Read / update / delete |
| GET | `/api/companies/suggest` | Typeahead from own data (companies + lead/contact `company` distinct values). Param: `q` (≥ 2 chars) |
| GET | `/api/companies/suggest/ai` | LLM-powered company lookup. Param: `q`. Returns up to 3 candidates with name, domain, industry, location, description. Always tag as "Verify" in the UI |
| GET | `/api/companies/{id}/contacts` | List contacts linked to the company. Phase 3-aware: queries the `contact_companies` join collection AND falls back to legacy `contact.company_id`, unions + dedupes. Each row gets `link_role` + `link_is_primary` decoration when the link came via the join |
| GET | `/api/companies/{id}/leads` | List leads linked to the company via `lead.company_id` FK. Sorted `ai_score` desc so highest-priority leads bubble first. Symmetric to `/contacts` for the lead side of the graph |
| POST/PUT/DELETE | `/api/contacts/{id}/companies` | M:N link management. POST adds a `contact_companies` join row (with `role`, `is_primary`); PUT updates the role / primary flag; DELETE soft-removes. Setting `is_primary: true` mirrors the company name back into the contact's legacy `company_id` field for back-compat |
| POST | `/api/ai/enrich-company/{id}` | One-click enrich on existing record. Fills only empty fields; full enrichment payload (founded_year, HQ, competitors, products, tags) stored under `enrichment` |
| POST | `/api/bulk/enrich` | Body `{entity_type: "company", entity_ids: [...]}`. Cap 20 per call |

**Contact-Company Graph (CRM Phases 2 + 3 + 4)** — Leads, Contacts, and Companies form a graph instead of a 1:1 chain:

- **Typed multi-value channels (Phase 2)** — `Lead.emails` / `phones`, `Contact.emails` / `phones`, `Company.emails` / `phones` are arrays of `{value, label, primary}` (e.g. `[{value: "jane@acme.com", label: "work", primary: true}, {value: "jane@gmail.com", label: "personal"}]`). Exactly one entry per array can be `primary`; the primary is mirrored back to the legacy `email` / `phone` scalar fields so existing code that reads them keeps working. The frontend `<MultiValueInput>` component drives the edit UI.
- **M:N Contact ↔ Company (Phase 3)** — `contact_companies` join collection (`link_id`, `contact_id`, `company_id`, `role`, `is_primary`, `deleted_at`). One contact can be linked to many companies; one company can have many contacts. Lead conversion auto-creates a join row + a company doc if the lead's free-text company string doesn't match an existing record. Migration script: `scripts/migrate_contacts_to_company_join.py` (idempotent dry-run by default; trigger via `gh workflow run migrate-contacts-companies.yml -f apply=true`).
- **Search-graph traversal (Phase 4)** — `POST /api/ai/smart-search` walks the graph from each direct hit. Match a company → its contacts (via join) and leads (via FK) appear in `connected.contacts` / `connected.leads`. Match a contact → its companies appear in `connected.companies`. Match a lead → its company appears too. Each connected row has a `_via: {type, id, label}` field so the UI renders a "via Acme Co" badge. Soft-deleted entities filtered everywhere.

### Campaigns

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/campaigns` | List campaigns |
| POST | `/api/campaigns` | Create draft campaign |
| PUT | `/api/campaigns/{id}` | Edit (sent campaigns: rename only — content/subject/channel frozen) |
| DELETE | `/api/campaigns/{id}` | Delete (cascades into `campaign_recipients`) |
| POST | `/api/campaigns/{id}/send` | Fire send pipeline (Kit → Resend with per-recipient tracking) |
| GET | `/api/campaigns/{id}/recipients` | Per-recipient delivery status + aggregate totals (delivered, bounced, failed, replied, opted_out, **pending** = unsent only, **deferred** = retryable) |
| DELETE | `/api/campaigns/{id}/recipients/{recipient_id}` | Remove a single recipient (also yanks the email from the campaign's raw `recipients` array) |
| POST | `/api/campaigns/{id}/retry-deferred` | Reset `deferred` recipients to `pending` and re-fire the send pipeline with rate-limit pacing. Delivered/bounced/replied untouched (two independent guards). Returns `{deferred_found, retried, send_summary}` |
| POST | `/api/bulk/add-to-campaign` | Body `{campaign_id, entity_type: "lead"\|"contact", entity_ids: [...]}`. Returns `{added, skipped_already_on_campaign, skipped_no_email, total_recipients}` |

### Chat (1:1 DMs, private/public groups, moderators)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/chat/channels` | Visible channels (DMs filtered to participants; private to members + admins; public/general to all org) |
| POST | `/api/chat/channels` | Create. Body accepts `kind: "private" \| "public"`, `members`, `moderators` (≤ 10) |
| GET | `/api/chat/channels/{id}/messages` | Visibility-checked. 403 for non-participants of DMs |
| POST | `/api/chat/channels/{id}/messages` | Visibility + write check |
| POST | `/api/chat/dm/{other_user_id}` | Open or get the deterministic 1:1 DM (idempotent) |
| POST | `/api/chat/channels/{id}/promote` | Promote a DM to a named private group (≤ 10 members; one-way) |
| POST | `/api/chat/channels/{id}/members` | Add member (mods + admins only) |
| DELETE | `/api/chat/channels/{id}/members/{user_id\|"me"}` | Expel (mod) / self-leave (anyone) |
| POST | `/api/chat/channels/{id}/moderators` | Appoint moderator (any moderator can; admins implicit) |
| DELETE | `/api/chat/channels/{id}/moderators/{user_id}` | Demote |
| GET | `/api/admin/chat/audit` | Forensic-grade audit log query (super-admin only). Filters: `channel_id`, `actor_user_id`, `event_type`, `since`, `limit`. Includes DM message content for investigations |

### Entity History

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/entity/{entity_type}/{entity_id}/history` | Unified activity timeline. `entity_type ∈ {lead, contact, deal, company}`. Aggregates: created/updated, email events (with full body + delivery status from `campaign_recipients`), `inbound_email` events (transactional replies via `inbound_messages`, lead-fallback matched), `outbound_email` events (operator BCC capture), tasks, linked deals, conversion events (lead→contact), contextual chat summary. Per-event drill-down fields included |
| PUT | `/api/auth/me/email-aliases` | Self-service: register additional sender addresses on the current user. Used by the BCC outbound matcher when the operator's send address differs from their TAKO login |

### Webhooks

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/webhook/stripe` | Stripe webhook receiver. Signature verified against `STRIPE_WEBHOOK_SECRET`; idempotent on `event.id` |
| POST | `/api/webhooks/resend` | Resend delivery events (sent, delivered, delivery_delayed, bounced, complained, opened, clicked, failed, suppressed). Svix HMAC-SHA256 verified against `RESEND_WEBHOOK_SECRET`; idempotent on `svix-id` |
| POST | `/api/webhooks/resend/inbound` | Resend inbound emails. Matches in this order: (1) `In-Reply-To` → `campaign_recipients.resend_email_id`, (2) `References` → same, (3) from-email → `campaign_recipients.email`, (4) **lead-fallback**: from-email → `leads.email` (most-recent across all orgs), (5) **BCC outbound**: from = a user (or alias) AND a TAKO address in BCC → log as outbound on the lead matched by To-address. Body fetched separately from Resend's `/emails/receiving/{id}` and persisted on the row. Same signature scheme as outbound. Requires Resend Inbound DNS + dashboard config |

### Listeners

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/listeners` | Create listener |
| GET | `/api/listeners` | List for org |
| PATCH | `/api/listeners/{id}` | Update / pause / resume |
| DELETE | `/api/listeners/{id}` | Delete |
| GET/POST | `/api/listeners/{id}/sources` | List / add sources |
| PATCH | `/api/listeners/{id}/sources/{src_id}` | Approve / reject source |
| GET | `/api/listeners/{id}/hits` | List hits |
| POST | `/api/listeners/{id}/hits/{hit_id}/create-task` | Create task from hit |
| GET | `/api/listeners/{id}/reports` | List reports |
| POST | `/api/listeners/{id}/reports/generate-now` | Generate report now |
| POST | `/api/listeners/{id}/discover` | Trigger group discovery |

### Files

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/files/upload` | Upload. Params: `linked_type`, `linked_id`, `description` |
| GET | `/api/files` | List. Params: `linked_type`, `linked_id` |
| GET | `/api/files/{id}/download` | Download |
| DELETE | `/api/files/{id}` | Delete |
| POST | `/api/files/{id}/create-tasks` | Create tasks from AI suggestions |

### Settings

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET/PUT | `/api/settings/integrations` | Org integration keys (admin/owner) |
| GET | `/api/settings/ai-status` | AI availability for current user |
| GET/PUT | `/api/settings/stages` | Custom deal/task stages (with `order` field for drag-to-reorder) |

### Trash & Restore

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/trash` | List soft-deleted entities for the current org. Member sees their own deletes; admin sees the org-wide trash. Returns `{items, scope}` |
| POST | `/api/trash/restore/{entity_type}/{entity_id}` | Restore a soft-deleted lead / contact / deal / task / company / project / campaign / calendar_event. Member can only restore items they themselves deleted; admin can restore anything in their org's trash |

### Campaign Agent (admin)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/admin/agents/bootstrap-campaign` | Create / update a Campaign doc with agent fields (`sequence_config_key`, `tag_filter`, `owner_user_id`, `agent_daily_step_limit`). Idempotent on `campaign_id` |
| POST | `/api/admin/agents/run-daily` | Manual trigger for the daily campaign-runner. Runs as a background task; returns `{agent_run_id, started_at}` immediately |
| GET | `/api/admin/agents/last-run` | Most recent run record from `db.agent_runs` for the current org. UI polls this after triggering `run-daily` |
| POST | `/api/admin/agents/triage-tasks` | One-shot cleanup: keep top N agent-created `todo` tasks by linked-lead `ai_score` desc; soft-delete the rest. Body `{keep_top_n, dry_run, campaign_id?}` |
| POST | `/api/admin/agents/triage-no-link` | Soft-delete agent-created todos whose description has no http(s) URL. Body `{agent_only, dry_run}` |

### GDPR

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/gdpr/export-my-data` | Generate full data export for the calling user (JSON archive) |
| POST | `/api/gdpr/request-deletion` | Schedule account + associated data deletion (grace period) |
| POST | `/api/gdpr/cancel-deletion` | Cancel a pending deletion during the grace window |
| GET | `/api/legal/dpa` | Public Data Processing Agreement (no auth) |

### Support

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/support/contact` | Public contact form (no auth) — routes to super admin inbox |
| POST | `/api/support/ticket` | Authenticated in-app support ticket with user/org/env auto-attached |

### License & Updates

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/license/renew-maintenance` | Start Stripe checkout for the €999 annual maintenance SKU |
| POST | `/api/license/download` | Issue a short-lived signed token for downloading the current release |
| GET | `/api/license/download/{token}` | Redeem the signed token; streams the tarball from `DISTRIBUTION_DIR` |
| GET | `/api/version/latest` | Latest tagged version (public, no auth) |
| GET | `/api/version/check` | Per-caller version comparison (`current` vs `latest`, `update_available` flag) |
| GET | `/api/releases` | Last 20 releases, newest first (public — powers `/changelog`) |
| POST | `/api/admin/releases/notify` | Platform-only. Called by the release workflow after a tag is published |
| GET | `/api/system/update-check` | Customer instance → platform daily check; 24h per-org cache |

### Public surfaces (no auth)

These endpoints power the marketing site and are public — auth is the absence of any auth header. All have rate-limiting / signature-verification where appropriate.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/newsletter` | Newsletter signup. Body: `{email, source}`. Idempotent on email (duplicates return 200 without re-sending the welcome). Used by the footer form, the `/demo` page's "notify me when sandbox is live" capture, and any future opt-in surface |
| POST | `/api/partners/agency-application` | Founder Tier / Agency Partner application from the marketing `/partners` page. Body: `{full_name, work_email, company, website, linkedin, three_clients_text, founder_tier_intent}`. Inserts into `partner_applications`, fires the admin notification cascade (in-product → WhatsApp → email-to-every-admin) and sends the applicant an auto-response. Returns `{ok, application_id}` |
| POST | `/api/admin/partner-applications/{id}/approve` | Super-admin. Creates the partner (terms snapshot, founder slot, referral code wired into attribution), flips the application to `approved`, sends the welcome email (opt-out). Body `{notes?, booking_url?, as_standard?, send_email?}`; 409 on already-approved / slot-full (use `as_standard`) |
| POST | `/api/admin/partner-applications/{id}/reject` | Super-admin. Flips to `rejected`; optional generic decline email (default off; internal reason never leaves reviewer_notes) |
| GET/PUT | `/api/admin/partners[/{partner_id}]` | Super-admin. List (+ `founder_slots {total, used, remaining}`, per-partner referred-licence counts, 90-day condition) / pipeline update `{stage, notes}` (approved → certified → agreement_signed → active, paused suspends attribution) |
| POST | `/api/admin/partners/{partner_id}/onboardings` | Super-admin. Record a delivered onboarding; returns `condition_met_90d` for the founder free-licence condition |
| GET | `/api/partners/founder-slots` | Public. `{total, remaining}` founder slots — powers the live count on `/partners` |
| GET | `/api/booking/{user_id}/info` | Public booking host info (name, avatar, welcome message) plus a `meeting_types` array of active types (`type_id`, `label`, `description`, `duration_minutes`). Powers the `/book/:userId` route's header and type picker |
| GET | `/api/booking/{user_id}/available` | Available slots for a given date. Requires `type_id` query param (duration derives from the meeting type). May return `at_capacity: true` when that type's daily cap is reached |
| POST | `/api/booking/{user_id}/book` | Public booking submission. Body requires `type_id`; accepts an `Idempotency-Key` request header so retrying clients can't double-book. Returns 400 `unknown_or_inactive_meeting_type` or 409 `day_at_capacity`. Sets the row to `pending_confirmation` with a single-use token; deferred side-effects fire on confirm |
| POST | `/api/bookings/confirm` | Confirms a `pending_confirmation` booking via the token from the email. Creates the calendar event + fires the host notification. Returns 410 on expired token, 404 on invalid, 409 on already-cancelled |

### Admin / refund

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/admin/refund/{org_id}` | Issue a 30-day money-back refund (super-admin only). One-time licence → refund payment_intent in full. Instalment licence → cancel subscription + refund every paid invoice on it. Maintenance renewals excluded per Legal page. Pass `{"force": true}` in body to override the 30-day window for goodwill exceptions (audit-logged). Idempotent — returns 409 if the org is already refunded |

### Demo (platform-only)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/demo/create` | Spin up a seeded 14-day demo org. Triggered from `SetupOrgPage`'s "Try TAKO free for 14 days" button after standard signup + email verification — NOT directly from the public `/demo` route, which is the intent-capture marketing page |
| GET | `/api/admin/demos` | Super admin — list all active / expired demo orgs |
| POST | `/api/admin/demos/{org_id}/extend` | Extend a demo by N days |
| POST | `/api/admin/demos/{org_id}/expire` | Force-expire a demo (soft lock activates immediately) |
| DELETE | `/api/admin/demos/{org_id}` | Purge a demo org and all associated data |

### Health

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | 200 when MongoDB is reachable, 503 otherwise. JSON body: Mongo, email, Sentry, Stripe status. |

---

## External API (v1)

**Base URL**: `https://yourdomain.com/api/v1`  
**Auth**: `Authorization: Bearer tako_<key>` **or** `X-API-Key: tako_<key>` (generate in Settings → API & Webhooks)

> Two-tier auth model: human-facing CRM at `/api/...` uses JWT/session; programmatic integrations (n8n, Notion, AI agents) live at `/api/v1/...` with `tako_...` keys. Pasting a `tako_...` key against `/api/leads` will fail with 401 — that's by design. Use the `/v1/...` paths below.

### Leads / Contacts / Companies / Tasks / Projects

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/leads` | List. Params: `limit`, `status` |
| POST | `/api/v1/leads` | Create lead. Fires `lead.created` webhook |
| PUT | `/api/v1/leads/{id}` | Partial update — whitelisted fields only (name, email, phone, company, job_title, linkedin_url, status, source, ai_score, notes, tags, assigned_to, etc.) |
| GET | `/api/v1/contacts` | List |
| POST | `/api/v1/contacts` | Create contact. Fires `contact.created` |
| PUT | `/api/v1/contacts/{id}` | Partial update |
| GET | `/api/v1/companies` | List |
| GET | `/api/v1/deals` | List. Param: `stage` |
| GET | `/api/v1/tasks` | List. Param: `status` |
| POST | `/api/v1/tasks` | Create. Fires `task.created` |
| PATCH | `/api/v1/tasks/{id}` | Partial update with activity log |
| GET/POST | `/api/v1/projects` | List / create |

### Campaigns (full programmatic loop — Berny et al.)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/campaigns` | List campaigns |
| POST | `/api/v1/campaigns` | Create draft. Fires `campaign.created` |
| PUT | `/api/v1/campaigns/{id}` | Edit (sent: rename only) |
| DELETE | `/api/v1/campaigns/{id}` | Delete + cascade. Fires `campaign.deleted` |
| POST | `/api/v1/campaigns/{id}/send` | Fire send pipeline. Fires `campaign.sent` |
| POST | `/api/v1/campaigns/{id}/retry-deferred` | Reset `deferred` recipients to `pending` and re-fire the send pipeline (with rate-limit pacing). Delivered/bounced/replied stay untouched. Returns `{deferred_found, retried, send_summary}` |
| GET | `/api/v1/campaigns/{id}/recipients` | Per-recipient delivery state + aggregate totals (incl. `pending_count` = unsent only, `deferred_count` = retryable) |
| POST | `/api/v1/bulk/add-to-campaign` | Body `{campaign_id, entity_type: "lead"\|"contact", entity_ids: [...]}`. Fires `campaign.recipients_added` with granular counts |

### Files (programmatic upload — Berny et al.)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/files/upload` | Multipart upload. Form fields: `file` (required), `linked_type` (task / project / company / lead / deal / campaign), `linked_id`, `description`, `uploaded_by_name` (defaults to API key owner). Reuses the same `db.files` shape + `UPLOAD_DIR` as in-app uploads, so files appear in the existing Files page |
| GET | `/api/v1/files` | List files for the org. Params: `linked_type`, `linked_id`, `limit` (1..500) |

### Chat / Capture / Notion / Docs

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET/POST | `/api/v1/chat/channels` | List / create |
| GET | `/api/v1/chat/channels/{id}` | Get channel by ID or slug |
| GET | `/api/v1/chat/messages/{channel_id}` | Poll messages (`since`, `before`, `limit`) |
| POST | `/api/v1/chat/messages` | Post message. Body supports `sender_id` for multi-agent attribution (must be a user in your org; cross-org sender_id is rejected with 400). `sender_name` is the display label |
| POST | `/api/v1/capture` | Business card capture (multipart upload) |
| POST | `/api/v1/notion/sync` | Sync entity to Notion |
| GET | `/api/v1/docs` | API self-documentation |

```http
# Example: full campaign loop with one tako_... key
POST /api/v1/leads                                    # ingest
POST /api/v1/contacts
POST /api/v1/campaigns                                # create draft
POST /api/v1/bulk/add-to-campaign                     # attach
POST /api/v1/campaigns/{id}/send                      # fire
GET  /api/v1/campaigns/{id}/recipients                # poll delivery
```

```http
# Multi-agent attribution example
POST /api/v1/chat/messages
Authorization: Bearer tako_...

{
  "channel_id": "general",
  "content": "Booked a demo with @Jane",
  "sender_id": "user_0002f4c76a58",
  "sender_name": "Maestro (PM)"
}
```

---

## Integrations

| Service | Purpose | Configure |
|---------|---------|-----------|
| **Google OAuth** | Login + Calendar sync | Google Cloud Console → Credentials |
| **Anthropic Claude** | All AI features | Platform key in `.env` or per-org in Settings → Integrations |
| **Meta (Facebook)** | Social Listeners | Meta Developer Portal → `META_APP_ID` + `META_APP_SECRET` in `.env` |
| **Resend** | Transactional email + campaigns | Settings → Integrations |
| **Twilio** | Outbound/inbound calling | Settings → Integrations |
| **Stripe** | Licence purchase + installment billing | Admin panel (optional — only needed if you resell TAKO) |
| **Kit.com** | Email marketing automation | Settings → Integrations (optional) |
| **Google Calendar** | Two-way calendar sync | Settings → Integrations → Connect |
| **Sentry** | Backend + frontend error monitoring (soft dep) | `SENTRY_DSN` / `REACT_APP_SENTRY_DSN` |

---

## Deployment

TAKO runs as three Docker containers behind nginx with SSL:

```
nginx (SSL termination)
  ├── tako-frontend  (React app, port 3000)
  ├── tako-backend   (FastAPI, port 8001)
  └── tako-mongo     (MongoDB, port 27017 internal only)
```

```bash
git clone https://github.com/fbjk2000/tako-core.git /opt/tako
cd /opt/tako
cp backend/.env.example backend/.env
docker compose up -d
```

> **Important**: `docker compose restart` only bounces the container without recompiling. After a code change always run:
> ```bash
> docker compose build <service> && docker compose up -d <service>
> ```

nginx config:
```nginx
server {
    listen 443 ssl;
    server_name tako.software;
    location /api/ { proxy_pass http://127.0.0.1:8001/api/; }
    location / { proxy_pass http://127.0.0.1:3000/; }
}
```

For host hardening, backups, health checks, and error monitoring details, see [Security](#security) and [Operations](#operations) above. For VPS-level steps (deploy user, SSH, UFW, fail2ban, MongoDB bind), see [`docs/VPS-HARDENING.md`](docs/VPS-HARDENING.md).

---

## Recent Updates (Apr–Jun 2026)

### Late July 2026: Org-configurable qualification rubric + structured lead scoring

Organizations can now supply a free-text LLM **qualification rubric** (a scoring framework) that TAKO applies to lead generation and lead scoring, capturing a generic structured verdict on the record. The rubric is opaque config, so a team's own framework (blockers, weighted clusters, tiers, pitch angles) drives the AI without any code change.

- **Rubric config (singleton per org).** New `qualification_rubrics` collection with admin-gated CRUD at `GET/PUT/DELETE /api/research/rubric` (name, `rubric_text`, `enabled`, `apply_to` ∈ {discovery, leads}). Unique per-org index.
- **Generic structured output.** A fixed `qualification` contract (`tier`, `verdict`, `score`/`max_score`, `subscores`, `blockers`, `highlights`, `cautions`, `recommended_angle`, `confidence`, `evidence`) that any framework maps onto, appended last to the system prompt so it can never break JSON parsing.
- **Lead generation (discovery).** When a rubric targets discovery, each newly discovered prospect arrives pre-scored and tiered; the block is stored on the suggestion and carried onto the CRM lead on accept. The existing 0-100 fit score is untouched.
- **Lead scoring (on demand).** `POST /api/research/leads/{id}/qualify` (and the company variant) runs a web-search-grounded scoring pass and writes a `qualification` subdoc. Ungrounded, capped, or unparseable results write nothing (no fabricated scores). All calls reuse the per-org researcher model and the daily AI spend cap.

### Late July 2026: Inbound email subjects decoded (Approve-and-send fix)

Subjects fetched over IMAP were stored exactly as they arrived on the wire: RFC 2047 encoded words undecoded and folded header lines still carrying their CR/LF. Long or non-ASCII subjects showed up as `=?UTF-8?Q?...?=` gibberish in the Inbox and Approvals, and clicking "Approve & send" on such a draft failed with "Header values may not contain linefeed or carriage return characters" because the fold's linefeed rode along into the outbound reply subject.

- **Parse-time decode.** `_parse_message_headers` now runs subjects through `decode_mime_header` (RFC 2047 decode plus whitespace unfold), so new rows store clean single-line text everywhere it surfaces (Inbox, Approvals, AI draft prompts, team-chat notifications).
- **Old rows still send.** The Approve-and-send path decodes the stored subject before building the `Re:` reply, so drafts created before this fix go out with a human-readable subject and no crash.
- **Last line of defense.** `build_message` flattens any residual CR/LF in a subject instead of letting `EmailMessage` reject the header.

### Mid July 2026 — Security hardening sweep (audit quick-wins)

Findings from a multi-agent security audit, each verified end-to-end before shipping:

- **Stored-XSS in the inbox closed (critical).** Inbound email HTML in the reading pane is now sanitised with DOMPurify before render (matching every other HTML view), so a crafted message can no longer run script in the app origin and steal a session. The autopilot draft is sanitised too.
- **Credentials out of URLs.** Password-reset (token + new password), forgot-password (email), and the Google OAuth callback (access + refresh tokens) no longer travel in query strings where proxy/APM logs and `Referer` capture them — reset/forgot move to JSON bodies, OAuth tokens move to the URL fragment.
- **Mass-assignment blocked.** `PUT /leads|/contacts|/tasks` now strip immutable/ownership keys (`organization_id`, `created_by`, id fields …) before the Mongo `$set`, so a tenant can't re-home a record into another org or forge provenance.
- **Chat tenant isolation.** Message delete and reaction toggle are scoped to the caller's `organization_id` and gated on channel membership/moderator rights (DM privacy preserved) — no cross-tenant delete or existence-oracle.
- **User-enumeration removed.** Forgot-password always returns a generic 200; login runs a constant-time dummy bcrypt for unknown accounts.
- **Invite fan-out capped.** The invite email/CSV endpoints reject > 100 recipients and bound the CSV upload size, so the sending domain can't be turned into a spam relay; inviter/org names are HTML-escaped into the email.
- **Kit.com endpoints authenticated.** All Kit account/forms/tags/subscribers/broadcasts management routes now require a logged-in user (writes require admin); only the public lead-magnet subscribe flow stays open.
- **Edge hardening (Caddy).** Request bodies are capped at 25 MB and `X-Forwarded-For` is overwritten with the real peer so the per-IP login throttle can't be spoofed; the app derives the client IP from the trusted hop.
- **Multipart DoS closed at the framework layer (CVE-2024-47874).** FastAPI (0.110.1 → 0.115.14) and Starlette (0.37.2 → 0.41.3) were upgraded together so Starlette's default 1 MB `max_part_size` bounds the memory a malicious `multipart/form-data` request can consume. The cap applies only to non-file form fields; file uploads (CSV/xlsx import, CV/document uploads, outreach attachments) stream to disk unchanged. Full backend suite green at parity (1247 tests); complements the 25 MB edge cap above.

### Mid July 2026 — Send attachments with candidate outreach + real HR document uploads

Sending a contract to a candidate is now a first-class flow:

- **Attachments on Message candidates**: the hiring-board compose sheet takes up to 5 files (15MB total) — attach the contract, hit Send, and every recipient gets it. Files ride the campaign engine's per-recipient path (attachments force the tracked lane, never the Kit broadcast), and a send that can't load its attachment **aborts instead of going out without it**.
- **Employee Documents upload fixed**: the Upload button on the employee record's Documents tab was invisible (a phantom Tailwind class rendered it white-on-white) — now a proper button, upload verified end-to-end. A new grep-proof test fails the suite on any future `*-tako-*` colour class that isn't in the palette.
- **HR → Documents uploads real files**: the org-wide Documents page previously asked you to paste a "storage key/URL" by hand; its dialog now has an actual file picker (title auto-fills from the file name) and rows gained the missing Download action (legacy URL-based rows still open).

### Mid July 2026 — Stage-driven candidate messaging: the pipeline answers applicants for you

Every candidate stage can now carry an **auto-message template** (HR → Templates → candidate stages → Auto-message): when a candidate enters the stage — Kanban drag, auto-sourcing, hire or disposition — the stage's templated email is queued for them automatically.

- **Auto-acknowledge applications**: put a template with your booking link on the entry stage and every new applicant gets "thanks — book a call" within a minute of arriving, with zero clicks.
- **Delay = safety window**: a template can wait N minutes before sending; moving the candidate out of the stage within the window **cancels the send** (a mis-drag onto Rejected never fires the decline). Sent once per candidate per stage, ever.
- Under the hood: one campaign per job+stage (`hr_outreach_stage`, HR-fenced, pinned to per-recipient tracking), `hold_until` outbox semantics on recipient rows, and the hr_auto_source worker delivering due rows each minute — which also makes any previously failed board-outreach send self-heal.

### Mid July 2026 — Message candidates from the hiring board

The pipeline is now the outreach surface — no more hopping to Leads and rebuilding your selection from memory:

- **Select on the Kanban**: checkboxes on candidate cards (a column-header checkbox selects the whole stage) → a bulk bar appears → **Message**. The compose sheet shows exactly who will receive it (no-email applicants are flagged and skipped), supports merge tokens ({{first_name}}, {{name}}, {{job_title}} = the job's title) and one-click insertion of your booking link, and remembers your reply-to.
- **Outreach state on every card**: Sent / Replied / Bounced / Queued chips, fed by the campaign engine's per-recipient tracking. Replies (existing inbound webhook) flip the chip and are readable in the candidate drawer's timeline.
- Under the hood it's the existing campaign machinery — each send creates/reuses a "Hiring · {job}" campaign with candidate-keyed recipient rows (new `hr_candidate` entity type), forced onto the per-recipient tracked path (never the Kit broadcast). Applicants are exempt from the double-opt-in gate like contacts (they applied). Also fixed: campaign follow-up replies now honour the campaign's sender/reply-to identity, and merge fields render for candidates.

### Mid July 2026 — Hiring funnel polish: full-column drag-and-drop, clickable links on the record, ops scoring

- **Kanban drag-and-drop fixed**: the drop zone in each column only covered the top ~200px (the flex row stretched columns to equal height but the droppable list didn't stretch with them) — you had to drop a candidate at the very top of the next column. The list now fills the whole column, so a card can be dropped anywhere in it.
- **Links on the candidate record are clickable**: URLs in the recruiter notes and in the imported application text render as safe external links (plain-text parsing — no HTML, no non-http schemes).
- **Ops scoring**: `scripts/score_job_candidates.py` + a "Score job candidates" workflow — the batch twin of the in-app "Score all" button for after bulk sourcing runs, reusing the app's own AI plumbing (caps, spend logging, ai_call_log) inside the backend container.

### Mid July 2026 — The candidate card becomes an action hub; meeting requests join Approvals

Opening an applicant's card now lets you do everything from within (no hunting through other pages):

- **Send email** — the outreach composer (merge tokens, reply-to, booking link) opens right from the drawer for that one candidate; sends are tracked per-recipient and land on the record + timeline.
- **New task** — the real task dialog opens *in* the drawer, pre-linked to the applicant's lead (visible hint), so the task shows up in Tasks/Pulse and on the candidate timeline.
- **Log call** — direction + note, straight onto the activity timeline (`call_logged` audit events).
- **Move to Contacted on send** — messaging entry-stage candidates offers "Move {n} to 'Contacted' after send" (org-config aware: first → second active stage, stage-move audit rows, stage auto-message templates fire like a manual drag).
- **Meeting requests in Approvals** — pending booking approvals (guests who picked a slot via your booking link) now appear on `/approvals` alongside task suggestions, drafts and prospects, with inline Approve meeting / Dismiss (same `/bookings` endpoints).

### Mid July 2026 — Multi-word search fix + the candidate record grows up

Searching **"karen williams"** used to return nothing — the whole query string was matched against each field separately, and a name stored split across first/last name could never match. Now every search box tokenises the query: each word must match at least one field (`search_core`, shared by the TopBar quick-search, the Leads/Contacts list filters and their backend `q` params). Quick-search also covers **HR candidates** now (HR-entitled workspaces + HR-capable roles only) and deep-links straight into the job funnel with the drawer open.

The ATS candidate drawer became a real record:

- **Application / Notes block** — landing-page applicants' answers live on the CRM lead they arrived as; the drawer now shows the linked lead's notes in full (plus a **View lead** link), instead of hiding them. `GET /hr/candidates/{id}` embeds `source_lead`; a one-shot, idempotent backfill (`scripts/backfill_candidate_lead_links.py` / `backfill-candidate-lead-links.yml`) links pre-existing candidates to their lead by email.
- **Edit in place** — name, email, phone, notes, tags and stage are editable from the drawer (same whitelist API as the Kanban drag).
- **Activity timeline** — candidate created, stage changes, edits, notes, scorecards, CV updates, plus tasks and emails involving the applicant, in one chronological feed (the same `audit_events` trail leads and deals use — appended at write time, with a note composer in the drawer).
- **Scoring without a CV** — "Score" on a web-form applicant now scores against their application answers (the linked lead's notes) instead of erroring; the 400 only remains when neither a CV nor notes exist.

### Mid July 2026 — Leads flow into the hiring funnel: quick actions + tag-based auto-sourcing

Getting an applicant lead into the ATS used to mean opening the lead editor and hunting for the funnel button. Now:

- **Row + bulk quick actions** (HR-enabled workspaces): every lead row's ⋮ menu has **Add to hiring funnel**, and selecting leads shows the same action in the bulk bar — one job pick (preselected when only one job is open), one confirm, done. Bulk fan-outs report added / already-in-a-funnel / failed tallies.
- **Tag-based auto-sourcing**: a job (create dialog or the new Automation card on the job page) can declare **auto-source tags** — e.g. `job-application`. From the moment the tags are set, every new lead arriving with one of them is added to that job's funnel automatically by a 60s worker (`hr_auto_source`). Historical leads are never swept up (enablement timestamp is the floor), a lead lands in at most one funnel org-wide, and a re-submitted application (same email on the same job) is skipped instead of duplicating the Kanban card.
- Sourcing a lead that's already a candidate on another job now returns a clean **409 with the other job's name** (was a latent 500 — the idempotency check was per-job while the unique backstop is org-wide).

### Mid July 2026 — Multi-pipelines with qualification criteria: deals that advance themselves (with your approval)

Deals can now live in **several named pipelines** (Client, Partner, …), each with its own stages — and every stage can carry **qualification criteria** the platform works on for you:

- **Pipelines as real config** (Settings → Pipelines): stages with drag-reorder, closing stages carry an **outcome** (won/lost — a custom "Closed Won" stamps `won_date` exactly like the built-in), per-stage probability, and per-stage **entry tasks** that are auto-created for the deal owner the moment a deal enters the stage. Orgs that never touch this keep working unchanged on a built-in default derived from their existing deal stages — zero migration.
- **Qualification criteria per stage**, three kinds: **field checks** (deterministic, free — "company size ≥ 50", "email present", dotted paths into enrichment/custom fields), **AI judgments** ("has a budget signal been confirmed?" — answered by grounded, citation-backed AI, cost-capped), and **manual checks** your team ticks off on the deal.
- **The engine does the legwork**: a 5-minute worker re-evaluates changed deals, runs AI judgments only when they're the last gap, and — when an unmet check is data-shaped — **auto-runs Research Autopilot enrichment on the linked company/lead before ever asking a human** (budgeted, deduped, cooldown-stamped).
- **Ready deals queue for one-click approval**: when every entry criterion of the next stage is met, a **ready-to-advance suggestion** appears in Approvals (with the full criteria checklist, evidence and citations) and the 17:30 digest. **Accept performs the move** — outcome stamping, entry tasks, audit trail, `deal.stage_changed` webhook (now carrying `pipeline_id`). Readiness regressing retires the suggestion automatically.
- **On the board**: pipeline tabs, readiness chips on every card ("2/3 → Proposal"), and a Qualification panel in the deal dialog (re-check / AI check / manual check-off).
- **Agents joined in**: Berny/@tako gained a `propose_stage_move` tool — CONSULT-tier as always, it files a suggestion for a human to approve, never moves the deal itself.
- Stage moves are now **validated against the pipeline's stage set** (free-string stages were how boards and reports drifted apart), and the whole feature shipped through a 5-dimension adversarial review (14 confirmed findings, 13 fixed pre-merge). Design doc: `docs/operations/2026-07-09-multi-pipelines-qualification.md`.
### Mid July 2026 — Sales-rep readiness: Approvals that flow, honest Private life, in-app guidance, Linear + Nuclino

One package to make a new sales rep productive on day one:

- **Approvals grew from a list into a review flow.** Every queue item now expands to its **full story** (`GET /api/approvals/{kind}/{id}`): for an email draft that's the complete original message, the earlier thread, and the whole AI reply; for prospects the full why-fit plus the actual source links; for task suggestions the description and related lead/deal. Email drafts finally have a real **Approve & send** (`POST /api/email/{id}/autopilot/send`) — one click sends the reply through the normal mail path instead of bouncing you to the Inbox. And **"Review one by one"** walks the queue in a dialog with progress, full context, skip, and A/D/S keyboard shortcuts — act on an item and the next loads automatically.
- **"Private life" on Today's Focus is no longer a locked mystery.** Tasks can be created as **Private** (owner-only — hidden from every teammate, admin queries included, across lists/pulse/single-task reads), and the Focus toggle now actually works: it writes `user.include_private_chores` and decides whether your private tasks join the AI ranking.
- **Lead vs Contact is explained where the confusion happens**: persistent **"?" explainers** on the Leads/Contacts/Deals/Approvals page titles, a convert dialog that says what converting actually does, and a note that the "Lead" deal-stage ≠ the Leads list.
- **Interactive help, zero new dependencies**: first-visit coach-mark tours on Leads/Contacts/Deals/Tasks/Inbox/Approvals (spotlight + bilingual copy, skips missing anchors, Esc closes, seen-state in localStorage) plus a floating **?** button on every page (relaunch tour / Help centre).
- **Linear + Nuclino connectors** (Settings → Integrations, org-scoped keys): *Send to Linear* turns a task or deal into a Linear issue with a back-link on the record (`external_refs`); *Push to Nuclino* files a research dossier / meeting prep into the team wiki with sources, idempotently (re-push updates the same item).
- **Mail robustness fixes found while testing the send path**: outbound messages are now serialized with CRLF + quoted-printable (bytes handed to `smtplib` bypass its EOL normalization — strict MTAs rejected long single-line HTML bodies with "500 Line too long"), and SMTP hosts without AUTH no longer 500 the send endpoint.
- **A sales-rep training manual** with screenshots and worked examples lives at `docs/training/sales-rep-manual.md`.

### Early July 2026 — Sovereign frontend serving: VPS-primary with runner-side prerender

tako.software is now served entirely from our own VPS instead of reverse-proxying to Netlify. A 2026-07-04 routing incident (Netlify moved to AWS-eu endpoints the IONOS VPS couldn't reach) took the site dark despite both ends being healthy — so the Netlify hop is gone for good. The frontend deploy workflow now builds **and SEO-prerenders** (`/` + `/partners`, headless Chrome with content assertions) in the GitHub Actions runner and ships the finished artifact to the VPS, where the frontend container serves it via a bind mount (host nginx serves `/partners` from the same artifact directly, since `serve -s` can't express that pretty URL) — no docker build on the production box anymore, and crawlers get the full prerendered HTML from our own edge. Netlify (app.tako.software) remains as deploy previews and a hot spare. Details: `docs/operations/2026-07-04-vps-primary-frontend-runner-prerender.md`.

### Early July 2026 — Founding-partner approve flow

"Approve" on a Founder-Tier application is now a real commercial act, not a triage label. Approving (Admin → Partners) creates the partner with a **snapshot of the published terms** (€500/licence + €750/onboarding paid within 14 days, 20% of maintenance renewals, free licence conditional on 3 onboardings within 90 days of signing), reserves one of the **10 founder slots**, mints a referral code that plugs straight into the existing attribution + commission ledger (their referred sales auto-credit from day one), and sends a personal welcome email with the next step — booking the 30-minute discovery call. The panel gains approved/rejected states, a courteous opt-in decline email, and a **Partners pipeline** (approved → certified → agreement signed → active, with onboarding tracking against the 90-day condition and referred-licence counts). If a founding partner later creates a TAKO account, self-registration links their existing record instead of minting a duplicate code. The public `/partners` page now shows the **live** founder-slot count.

### Early July 2026 — German call transcription (Automation-Max 4b)

Calls and voicemails on German-language orgs now get real transcripts. Twilio's built-in transcription is English-only, so a new **Voice Intelligence pipeline** takes over for German: completed recordings queue for a `de-DE` transcription service, a 60s worker creates and polls the transcript, and the speaker-labelled text lands on the call log — where the existing **AI call analyzer** picks it up and now answers **in the call's language** (German call → German summary, sentiment and follow-up suggestions). Cost-guarded (recordings under 3s skipped, transcribed seconds logged), resilient (retry/timeout bounds, feature stays dark until the one-click provisioning workflow creates the VI service). English calls keep the existing path unchanged.

### Early July 2026 — Approvals, daily digest, Maestro, real outbound webhooks (Automation-Max 3b)

Everything TAKO's agents stage for a human now lives on one **Approvals page** (`/approvals`, System nav with a live count badge + a My Day tile): pending task suggestions, autopilot reply drafts (open straight in the Inbox composer — sending stays a human act), and research prospects — accept/dismiss inline, scoped exactly to what you're allowed to see. A weekday **17:30 digest** closes the loop on "silence means progress": each user gets a bell notification + a TAKO chat DM with what the agents did for them today (drafts prepared, suggestions filed, calls analyzed, tasks completed) and their top pending approvals; owners/admins get an org digest (AI spend by feature, agent runs, suggestion accept/dismiss, autopilot volume, research output, erroring jobs). Nothing to say → no digest. **Maestro** joins as a second chat agent on the shared framework (coordination/triage chief-of-operations — routes asks, tracks deadlines/blockers, labels AUTO/CONSULT/REQUIRE, files suggestions only), toggleable per org in Settings → AI Agents. And **outbound webhooks became a real product feature**: they now fire from the actual product flows (lead/contact/deal/task created, deal stage changes, inbound email linked, call completed, candidate hired — previously only the /v1 API emitted anything), signed with a per-hook secret (`X-Tako-Signature`, HMAC-SHA256, shown once at create), SSRF-hardened, retried once, and logged (last 50 deliveries per hook) — managed from a rebuilt Settings → Webhooks card with a one-click signed test event. EN/DE parity throughout.

### Early July 2026 — Research Autopilot: enrichment you can trust, prospects that find you

AI enrichment is now **web-grounded with citations** — the old enrich literally asked Claude to "generate plausible" firmographics; it now researches the company/person via Anthropic web search, stores **sources, per-field confidence and a retrieved-at stamp**, fills only empty fields as before, and the UI shows a **Verified/Unverified badge**, a collapsible Sources list and an explicit "Unverified: …" line (all four enrich surfaces: leads, companies, contacts, capture). Define **ICP profiles** in the new Settings → Research card (verticals, sizes, geos, signals) and a **weekly discovery agent** (Mon 06:45) researches net-new matching companies — deduped against your whole CRM and everything it ever suggested — and stages them as **Prospects on the Leads page**: Accept converts to a lead carrying the rationale + sources, Dismiss means never again. Every lead/company/deal gets a **"Compile dossier"** button (Overview / news / key people / talking points, citation-marked, queue-processed), and a **meeting-prep agent** scans the next 24h of scheduled calls, Google Calendar events and confirmed bookings, auto-compiles a brief for matched CRM entities and files a "Review prep brief" suggestion. New `researcher` agent in Settings → AI Agents (per-org toggle/model/persona); all research calls run web-search-capped, cost-logged (searches billed into `ai_call_log`) and under the daily AI cap. EN/DE parity throughout.

### Early July 2026 — Telephony: the phone number goes live

Inbound calls to the TAKO number now work like a team phone system. Callers hear an org-configurable greeting (English or German), get **forwarded to your phone** — with a **recording notice announced before connecting** when call recording is enabled (two-party consent, §201 StGB) — and land in **voicemail** when nobody answers. Every missed call and voicemail creates a **high-priority callback task** automatically, matched to the caller's lead/contact and carrying the recording link. Voicemail transcripts (English via Twilio's built-in transcription; German via Voice Intelligence since 4b) flow through a new AI analyzer that writes a call **summary + sentiment** onto the call log and files concrete follow-ups as pending task suggestions. All Twilio webhooks are **signature-validated** now (the legacy earnRM-era handlers — unvalidated, wrong brand greeting, no tasks — are gone). New **Settings → Telephony** card: forward number, recording, voicemail, language, greeting. A one-click GitHub workflow points the Twilio number at the new webhooks. EN/DE parity throughout.

### Early July 2026 — Agent Core: @tako in Team Chat, Berny reborn, personas in Settings

TAKO's AI agents now run on one shared framework with real **tool use**: they search the CRM, read records, summarise the pipeline, list your open tasks, and file follow-up **task suggestions** (always pending your approval — agents never change data directly and cannot send anything). Mention **@tako** in any Team Chat channel and the assistant answers with grounded CRM facts in your language. **Berny** — the chief-of-staff agent — moved off his hardcoded prompt onto the same framework: his persona now lives in the database, he gained the full toolbelt, and his channel behaviour is unchanged. Every agent run writes a full trace (tool calls, tokens, cost, duration) to `agent_traces`, and agent spend is attributed per feature in the Command dashboard. New **Settings → AI Agents** card (admins): switch each agent on or off and edit its persona — changes apply within a minute, no deploy. EN/DE parity throughout.

### Early July 2026 — Dashboards v2: My Day for every seat, Command for management

The dashboard now works in two tiers. **My Day** (everyone) puts the morning at a glance: inbound emails the autopilot flagged as *awaiting your reply* (with a "draft ready" spark, deep-linking straight into the message), today's confirmed bookings and scheduled calls, your open pipeline by stage, and overdue/AI-suggestion badges — alongside the existing AI-ranked focus tasks. **Command** (admins/owners only, loaded lazily so the charting library never ships to member seats) answers the management questions without asking anyone: a 14-day leads-created vs deals-won momentum chart, pipeline value stacked by owner, a 7-day per-member activity table (emails sent · tasks done · messages), an AI-agent digest (runs + spend per feature, suggestion accept/dismiss counts), and open-task load per member. First real use of charts in the product. Backed by two new aggregate endpoints (`/api/dashboard/my-day`, `/api/dashboard/command`) in a dedicated backend module. EN/DE parity throughout.

### Early July 2026 — Inbound Email Autopilot: every email arrives pre-worked

The inbox now works your mail before you open it. Every fresh inbound email is **auto-attributed** (sender matched against contacts — creating the CRM link automatically — and leads, shown as a badge), **intent-classified** (interested / question / meeting request / objection / support / invoice / unsubscribe / newsletter / spam), and, when a reply is genuinely expected, arrives with an **AI-drafted response** in the sender's language — built from the thread history, the sender's CRM context, and (for meeting requests) your booking link. Concrete follow-ups land as pending suggestions in the Tasks Pulse "Suggested by TAKO" strip. The inbox shows intent chips, an urgency dot, a matched-entity badge, a *Needs reply* filter, and a draft panel with **Edit & send** (opens the composer pre-seeded — sending always stays a human decision) or Dismiss. Runs as a dedicated queue worker (30s cadence, budgeted) so AI work never slows mailbox sync; full bodies are fetched transiently and never stored (the mail privacy memo holds); org-level toggle under Settings → Email compliance (default on, governed by the AI daily spend cap). EN/DE parity throughout.

### Early July 2026 — Hardening: governed agent AI spend, CRM indexes, startup resilience

Groundwork for the automation program (and a team of users). **Every background agent** (Berny responder, campaign draft engine, task ranker, listener skills) now calls the AI through the logged + daily-cap-enforced path with per-feature cost attribution — previously they used an unlogged, uncapped helper. **CRM core got its first index pass** (org-scoped compounds on leads/contacts/companies/deals email/status/stage/created_at, chat history, notifications), mirroring what HR/mail/listeners already did. A latent production bug was found and fixed in the process: the HR module's partial-unique indexes used a `$ne` filter that real MongoDB rejects (mongomock accepts it, so 690+ tests never noticed) — the failing index creation had silently aborted the whole startup hook since mid-June, leaving HR uniqueness unenforced and listener rescheduling skipped after restarts. Index creation is now per-module fault-isolated, a static guard test bans unsupported partial-filter operators, and a PR-triggered CI workflow runs the backend suite + locale parity before review instead of at deploy time. Post-login sessions also now hydrate from `/auth/me`, fixing entitlement-gated UI (like the HR nav group) vanishing until a page reload after a fresh sign-in.


### Early July 2026 — Tags v2: one tag experience across the whole CRM

Tags now behave the same everywhere. **Deals** switched from the legacy type-and-add tag box to the shared chip-based picker (with autocomplete against the org's existing deal tags) in both the create *and* edit dialogs — and tags on existing deals are finally editable at all (they were create-only). The **Leads and Contacts tag filter is multi-select**: pick several tags (any-of matching), the trigger shows a count, and the active status/source/tag filters round-trip through the URL so a filtered view can be bookmarked or shared. **Bulk edit** gains tag apply modes — *Add to existing* (default), *Remove these*, or *Replace all* — so retagging fifty records no longer wipes what was already there; replacing with an emptied list explicitly clears all tags. The picker itself got a deep polish pass: Escape closes only the suggestion menu (not the whole edit dialog), full screen-reader combobox wiring, pasting a comma-separated list adds every tag, arrow keys select chips for keyboard removal, and the menu flips above the field near the bottom of the screen. EN/DE parity throughout.

### Late June 2026 — Tag picker for Leads & Contacts

Tagging is no longer a raw comma-separated text box. Lead and Contact **edit *and* create** forms now use a proper chip-based tag picker (`TagInput`): existing org tags surface in a dropdown as you focus/type so you can pick from what already exists, a "Create …" row adds a brand-new tag inline, chips remove with a click (or Backspace), and duplicates are rejected case-insensitively. New tags created on save immediately repopulate the picker and the tag filter. The Contacts tag **filter** was also upgraded from a plain dropdown to the same searchable popover the Leads page already had. EN/DE parity throughout.

### Mid June 2026 — HR module goes live: hiring funnel, editable org chart, documents & scorecards

A wave of capability on the HR/HRIS product line (gated behind the `hr_license_status` SKU + an `hr_manager`/admin role, so no change for non-HR orgs). **ATS hiring funnel** — Jobs with a candidate Kanban across org-configurable stages, AI CV-vs-JD match scoring, source-from-lead/contact, hire→employee (with auto-onboarding), and disposition→tagged CRM lead. **HR navigation & record UX** — the sidebar **HR** item is now an expandable group (People · Hiring · Org chart · Templates · Documents); the People-list name-truncation bug is fixed (a shared grid template assumed a checkbox column the HR rows don't have); a new `evaluating` employee status sits before onboarding; and each employee record is tabbed with a **Communication** activity timeline (link-through to the linked CRM contact, where emails/calls live) and a per-person **Documents** tab (upload/list/download/delete, wiring the opaque `file_ref` to the global file store). **Process tools** — an editable, boxed **org chart** (set a person's manager from their record; reporting lines are cycle-protected); template steps that can **require a document** (completing that onboarding task prompts an upload straight into Documents, threaded additively through the shared `insert_task_doc`); and manual **interview scorecards** (1–5 rating + recommendation + notes, averaged) on each candidate alongside the AI CV score. EN/DE parity throughout; shipped over PRs #99/#105/#106/#107/#108.

### Mid June 2026 — Soft-deleted orgs leave the admin list + a flaky-build fix

Two follow-ups to the admin multi-select work. **(1) Soft-deleted orgs no longer linger.** `DELETE /admin/organizations` is a soft-delete (sets `deleted_at`, keeps the row for audit/restore), but the admin read paths returned every row — so a deleted org stayed in the table and re-deleting it 400'd, making bulk delete look broken. `GET /admin/organizations` and the org `count_documents` in `/admin/stats` + `/admin/reports/overview` now filter `{"deleted_at": None}` (a frontend guard in `AdminPage.jsx` also drops soft-deleted rows, so the list is correct even before the backend ships). **(2) Netlify builds were intermittently failing** with "exit code 2" — root cause was the SEO prerender postbuild (`frontend/scripts/prerender.js`): `page.goto(..., {waitUntil: 'networkidle2'})` could hang until the 60s timeout because a client-rendered SPA never reliably goes network-idle on a constrained CI builder, and the script fails the whole deploy by design. It now navigates with `domcontentloaded` and relies on the existing rendered-DOM assertion (`waitForFunction` on `#root` text), adds a bounded best-effort network-idle settle (can't hang the build), and **retries the whole render up to 3× with a fresh browser** — still throwing if every attempt fails, so a genuine empty-shell regression still fails loudly. Verified with repeated cold builds.

### Mid June 2026 — Admin lists get multi-select + reachable row actions

The Super-Admin **Users** and **Organizations** tables (`/admin`) were hard to operate: the right-hand **Actions** column scrolled off the edge of the wide table, so Edit / Reset-PW / Delete were difficult to reach, and there was no way to act on more than one row at a time. Both lists now have a leading **checkbox column** with a tri-state "select all" header, and a **bulk action bar** that appears on selection — **Delete selected** for both, plus a **Set role** dropdown for Users — so destructive controls are surfaced up-front instead of behind a horizontal scroll. The Actions column is now **pinned to the right** (sticky) so per-row controls stay visible regardless of table width, and the row buttons were tightened (Reset-PW is now an icon). Bulk delete fans out over the existing per-row endpoints (`Promise.allSettled`, single summary toast), so there is no backend change; the protected owner account (`florian@unyted.world`) stays unselectable for deletion. EN/DE strings added.

### Mid June 2026 — People picker gets its styles

The shared `UserPicker` (the people picker behind the project Add-member popover, the task owner/assignee fields, and the collaborator chips) had always emitted `tako-user-picker-*` / `tako-assignee-*` / `tako-collab-*` classnames that were never given any CSS — so every instance rendered as raw HTML, with avatar initials fused into names and the selected check mark stranded below the row. `App.css` now styles the whole family on-palette: a warm paper popover surface, teal-tinted circular avatars, name + email stacked in a truncating meta column, a teal check and tint on selected rows, and proper pill treatment for the assignee chip (including the dashed "unassigned" state) and collaborator chips. Pure CSS against tokens already on `main`.

### Mid June 2026 — Buy without an account + Pricing v2

Checkout no longer requires a TAKO account: **guest checkout** creates the Stripe session directly (email, billing and VAT ID collected by Stripe), fulfilment is **email-keyed** — a `licenses` record plus a signed 72-hour download link in the welcome email, a public `/download-link` page for fresh links, and automatic license attachment if the buyer ever creates an account with that email. Terms acceptance is now enforced at checkout (recorded consent + Stripe consent collection): instalment plans are contractually **deferred payment of the full price** — cancelling the payment plan now suspends the license (previously it stayed active) — and the 30-day money-back guarantee requires a signed **deletion certification**, submitted via the new `/refund` page and processed through an admin refund queue (Stripe refund + license revocation + referral commission claw-back). `/pricing` was rebuilt on the dark-cinematic design system with full EN/DE parity, and the Terms gained instalment/guarantee/UNYT-finality clauses (UK-law drafted, pending legal review).

### Mid June 2026 — Landing v3: the Dark Cinematic merge

The marketing landing at `/` was rebuilt on the "Dark Cinematic" design handoff (near-black blue, Fraunces display serif with teal italic accents, ambient aurora canvas, magnetic buttons, 3D-tilt cards, compliance marquee, a self-playing product-demo frame in the hero) while keeping every content block of the previous page: the four-pillar suite deep-dive with competitor comparison tables, the interactive CRM-tax calculator (tool checkboxes, five currencies, 5-year totals, email-me CTA — now with spring-animated numbers and a "you keep €X" receipt), Sentinel/Janitor/Scout, the why-own manifesto, the world map, and the full EN/DE toggle (now also syncing `<html lang>`). Code-wise the 2,638-line `LandingPage.jsx` became a thin composition over `frontend/src/pages/landing/` (fx primitives, i18n module with a parity audit script, 12 section components, one `.tako-lp3`-scoped stylesheet). All previous prerender SEO assertions still pass, plus four new ones. Every animation respects `prefers-reduced-motion`; offscreen animation loops pause via IntersectionObserver. The frontend bundle was then **code-split** (marketing pages eager, CRM pages lazy — `frontend/src/utils/lazyRetry.js` guards stale-chunk-after-deploy): the main bundle dropped 687 → 304 KB gzip, so landing visitors no longer download the app and app users no longer re-download marketing changes. Invariants in `docs/operations/2026-06-12-frontend-code-split.md`.

### Mid June 2026 — Chat becomes tasks + the Lens links itself

**Chat → Tasks (Phase D, project channels first).** A daily agent (weekdays 07:20 London, plus an on-demand scan button on every project workspace) reads each active project's chat since its watermark and turns clear commitments ("Kate, can you book the demo for Monday?") into pending task suggestions — same accept/dismiss strip as the Sentinel email suggestions, with `source="chat"` provenance, the source quote as the why-now line, assignee resolution against the project team, and hash-based dedupe so re-scans never duplicate. Accepted suggestions land on the project board; unaccepted ones are excluded from boards, health scores and digests. **Lens link suggestions.** When the Context Lens opens an unlinked task, it now offers one-click "Link to:" chips fuzzy-matched from the task text (deterministic, no AI cost) — the last unlinked-task gap after the task-dialog suggestions.

### Mid June 2026 — One design language, everywhere + a luxury marketing pass

A nine-agent audit swept every app surface against the TAKO design tokens (201 deviations found): cool-gray slate utilities replaced with the warm ink/graphite/stone/paper palette across ~25 pages, AA-compliant muted text everywhere real text was rendered in decorative smoke, font literals collapsed onto the token trio (Fraunces/Inter/JetBrains Mono), off-palette blues/cyans retired to indigo/teal semantics, destructive reds unified on coral, and the ⌘K palette CSS made global (it was completely unstyled outside /tasks). The marketing site got a luxury typographic pass — larger, lighter display headlines, a drawn starting-line gesture, grain, an enormous 蛸 watermark behind the ownership manifesto, indigo CTAs with a single coral conversion moment.

### Mid June 2026 — Projects revamped into a real PM surface

The project detail dialog is gone; every project now opens a full **workspace page** at `/projects/:id` (deep-linkable, back-button friendly — same pattern as the lead/contact/company/deal record pages). Following the conventions of Linear/Asana-class tools: **click-to-edit name and description with autosave** and a visible "Saved ✓" flash (the old dialog had no editing and no save feedback), **inline team management** (add/remove members — previously the team was rendered read-only with no way to change it; the roster now also syncs to the project's chat channel), a **drag-and-drop task board** (To Do / In Progress / Done with per-column quick-add) plus a grouped list view, **target date** on projects, and a task detail dialog that **stays open on save** with assignee + due date now editable. Backend: `PUT /projects/{id}` is whitelisted to client-editable fields (was: blind `$set` of any payload), members are validated against the org, and `project_core.py` carries the unit-tested update sanitizer.

**Round 3 — Projects get an AI layer (2026-06-12).** **AI task breakdown**: ✨ Generate tasks on the project workspace reads name/description/linked deal/team/existing tasks and proposes 4-8 concrete, sequenced tasks (priority, effort, due-in-days) — untick what you don't want, one click adds the rest. **Project health**: every project now carries a deterministic health score (overdue/stale/unassigned tasks, target-date pressure) shown as On track / At risk / Off track on the grid and workspace — no AI credits needed, it always works. **Chat digests**: a button posts a "what moved / what's stuck / what's next" digest into the project's chat channel as TAKO, and a Monday-morning cron does it automatically for every active project with recent activity. **Link suggestions**: the task dialog fuzzy-matches what you type against projects/deals/companies/contacts/campaigns and offers one-click link chips (deterministic, free). The ⌘K capture parser now resolves **project** references too. All new write paths went through an adversarial multi-agent review — org-scoping on AI prompt context, member-gating on digest posting, soft-deleted tasks excluded everywhere, BSON-Date/ISO-string due-date handling, and word-bounded fuzzy matching all hardened before ship.

**Round 2 — Tasks ↔ Projects unified.** Project tasks were already scoped into the task-management Pulse view (`/tasks?project=<id>`, "By project" surface, project chips), and the Tasks dialog could already link tasks to projects/deals/contacts/companies/campaigns — but the two surfaces wrote different headline fields (`title` vs `outcome`). The API now keeps the pair coherent at every write site (`tasks_v2.sync_title_outcome_*`, unit-tested): a project-created task carries a real `outcome` for the Pulse cards and AI rankers, and a one-sided rename mirrors across when the pair was in sync. Navigation closes the loop: project chips on task cards land on the project workspace page, the workspace's Tasks header gains **View in Tasks** (project-scoped Pulse), and the task dialog gains **Open in Tasks** (full Context Lens with AI next steps).

### Mid June 2026 — Self-host distribution refit + client downloads page

- **Distribution builder brought current** — `scripts/build-distribution.sh` had drifted since April: internal `docs/` and `.github/` were shipping to customers, the route-strip regexes had rotted, and the prerender postbuild broke the stage build. Platform pages are now replaced with redirect **stubs** (no route surgery), internal paths are excluded with a small docs whitelist, and a **hard verification gate** (brand-residue grep, required/forbidden files) refuses to emit a dirty package. Packages now include `Caddyfile.example`, a per-client-stamped `LICENSE.md`, and corrected env templates (incl. both Resend webhook secrets).
- **Per-client builds** — `--client <name>` applies `clients/<name>/overlay/**` over the stage before the frontend build, so client-specific frontend variants compile into the shipped bundle. Per-client handover records live in `clients/<name>/CLIENT.md`; process docs in `docs/handover/`.
- **Client downloads page** — `tako.software/downloads` → password-protected page carrying the latest package + sha256 + install guide, published via `scripts/publish-downloads.sh` (full-replace deploy: the page always serves exactly the latest release).

### Mid June 2026 — Booking settings made wipe-proof

A failed settings load + Save could silently **reset the host's entire booking configuration** to defaults (the PUT was a full-model replace, so an empty form wrote slug=None, default hours, default meeting types — this bit Florian after a deploy restart). `PUT /booking/settings` is now a **partial merge** (`exclude_unset`): only fields the client actually sends are written, an empty body is an explicit no-op, and `booking_slug: null` still clears the slug when sent deliberately. The settings dialog additionally refuses to save a never-loaded form, shows a Saving… state, and no longer has silent no-feedback paths (expired session now toasts instead of doing nothing).

### Mid June 2026 — Checkout fixes: VAT at checkout, installment plans unblocked

- **Pay Once charged €6,000 instead of €5,000** — the checkout endpoint added a flat 20% "VAT" on top of the advertised net price for every buyer (wrong for 19%-VAT Germany, and contradicting the page's "VAT calculated at checkout" promise). Sessions now carry the **net amount** with `tax_behavior=exclusive` + **Stripe automatic tax**, so checkout shows €5,000 + the buyer-country VAT. Falls back to a plain session (with a loud log) if Stripe Tax isn't enabled on the account. Invoices now split VAT from the **actual charged total** persisted off the Stripe session/webhook.
- **12/24-month plans dead** — they require recurring Stripe Price IDs that were never configured in prod (`STRIPE_PRICE_12MO/24MO`), so checkout 500'd. Missing IDs are now **auto-provisioned** via the Stripe API (Product + recurring monthly Price, cached in platform_settings). Env-configured IDs still take precedence.
- Checkout requests are bounded at 25s client-side and the Stripe line item now carries the real plan name instead of "TAKO Payment".

### Mid June 2026 — Booking approval queue + mobile nav fix

- **Booking approval queue** — guest-verified booking requests now wait for the host instead of auto-confirming. New status `pending_host_approval` (after the existing guest email-confirm step); the Bookings page shows a **"Requests awaiting your confirmation"** queue — opening an item lets the host **confirm with a meeting link + note to the guest** (both emailed, link stored on the booking and included with the .ics receipt) or **reject with a reason** (polite decline email + re-book link; the slot is released). Calendar event / lead / reminder side-effects now fire on host approval. Per-host toggle `require_approval` in Booking Settings (default ON); guards: host-or-org-admin only, no double-approve, rejected/expired slots rebookable. Endpoints: `POST /api/bookings/{id}/approve`, `POST /api/bookings/{id}/reject`.
- **Mobile landing nav** — Sign in / Get TAKO / language toggle were invisible on phones: the actions cluster rendered *behind* the open hamburger menu (two fixed panels at the same offset, z-index 198 vs 199). The actions now live inside the menu on mobile.

### Mid June 2026 — CI gates, template library, double opt-in (audit follow-ups closed)

- **CI gates before deploys** — the backend deploy now requires the full unit suite (Python 3.11) to pass; the frontend deploy requires the locale audit. Prod previously received pushes with zero automated checks.
- **diag-email-poller fixed** — the inline mongosh scripts made the workflow YAML unparseable, registering a failed run on every push; logic moved to `scripts/diag-email-poller.sh`, the workflow is a thin SSH runner again.
- **Email template library** — save campaign subject+body as an org-shared named template (`GET/POST/PUT/DELETE /api/email-templates`, cap 50); "Start from template" picker + "Save as template" in the campaign create dialog. Merge tags render at send time as usual.
- **Double opt-in (UWG §7)** — org toggle in Settings → Email (default off). "Request consent" on leads (single + bulk) sends a confirmation email with an HMAC-signed link; `GET /api/public/confirm-consent` stamps `consent_status=confirmed` + timestamp as proof. With the toggle on, campaign sends **skip unconfirmed leads** (rows stay `pending`, count reported as `skipped_no_consent`); contacts are not gated. Consent badges on lead dialogs + record pages.
- **Laptop-width layout fix** — PageHeader wraps instead of crushing the title; row hover actions float as an overlay instead of reserving ~250px per row (was breaking Leads/Contacts/Companies at ~1100px windows).

### Mid June 2026 — UI/UX plan Waves A–D: global search, record pages, bulk edit, saved views, funnel

The approved follow-up to the audit pass (`docs/operations/2026-06-11-ui-ux-enhancement-plan.md`), shipped in one run:

- **Global record search in the TopBar** (`⌘/`) — one persistent field finds leads, contacts, companies and deals as you type via the new `GET /api/quick-search` (regex, org-scoped, no AI). Results deep-link into the new record pages. ⌘K stays task-capture.
- **Full record pages** — `/leads/:id`, `/contacts/:id`, `/companies/:id`, `/deals/:id`: shareable URLs with browser back/forward, details + custom fields on the left, the unified history timeline as a first-class column on the right. List dialogs remain the quick-peek; each gains a "Full page" button. New `GET /api/deals/{id}` (single-deal read existed only via the list before).
- **Bulk edit** — "Edit fields" in the bulk bar on Leads/Contacts/Companies drives the existing `/bulk/update` (now field-whitelisted server-side): set status/source/tags/decision-maker/industry across N selected records; untouched fields keep their values.
- **Dashboards v2 — My Day + Command** — Every seat gets **My Day**: emails awaiting your reply (with autopilot draft badges, deep-linking into the inbox), today's bookings + calls, your open pipeline, overdue/AI-suggestion badges. Admins/owners additionally get **Command**: a 14-day leads-vs-won momentum chart, pipeline value by owner, 7-day per-member activity, an AI-agent digest (runs + spend per feature), and team load (`GET /api/dashboard/my-day`, `GET /api/dashboard/command`).
- **Dashboard that earns the homepage** — Today's Focus top-3 (same card as /tasks), a **weekly pulse strip** (deal stage moves from the new audit trail, won/lost counts + value, replies awaiting an answer via `GET /api/dashboard/weekly-pulse`) and a **conversion funnel** card (leads → converted → deals → won, with per-source breakdown via `GET /api/dashboard/funnel`).
- **Saved views** — per-user named filter combinations as a chips row on Leads/Contacts (`GET/POST/DELETE /api/views`); snapshot the current filters, re-apply with one click.
- **Server-side sort** — `sort`/`order` params (whitelisted per entity) on the core list endpoints + a sort dropdown in the filter bars (newest/oldest/name/recently updated).
- **Mobile** — entity pages below 640px: full-width search, 50/50 filter chips, wrapping rows, sticky bulk bar.
- **Polish** — org-level **trash retention setting** (Settings → Data, drives the daily purge job), task priority spelled out next to the colour dot, tasks filter-banner i18n, +84 locale keys (en/de parity maintained at 2,545 keys).
- **Deploy hardening** — pip network timeout/retries raised in the backend Dockerfile after two consecutive PyPI read-timeouts took the prod build (and briefly the backend) down on 2026-06-11.

### Mid June 2026 — Data ownership pass: export, import integrity, duplicates, compliance (product audit P0–P2)

A full product audit (2026-06-11, `docs/operations/2026-06-11-tako-product-audit.md`) found the brand promise — *owned* data — wasn't backed by the product: there was no export anywhere, imports silently corrupted semicolon CSVs, core lists silently truncated at 1,000 rows, and the only global search died without an AI key. One overnight pass closed all P0–P2 findings:

- **CSV export everywhere** — `GET /api/export/{leads|contacts|companies|deals|tasks}` (org-scoped, honours active list filters, streams with no row cap, UTF-8 BOM so German Excel opens umlauts cleanly) + Export CSV buttons on every entity page and a new **Settings → Data** tab with per-entity exports and the GDPR Art. 20 **full-workspace JSON export** (existed since the GDPR work, now finally has a UI).
- **Import integrity** — all three import-csv endpoints parse via the new `backend/data_io.py`: utf-8-sig everywhere, **delimiter sniffing** (semicolon German-Excel files used to import as one blank record while reporting success), empty/invalid rows skipped with per-row reasons, duplicate emails/names reported instead of silently re-inserted. Import dialogs show the full report + a downloadable CSV template.
- **Pagination** — `/leads /contacts /companies /deals /tasks` accept `limit`/`offset` and return `X-Total-Count`; list pages show "Showing X of Y · Load more". `/v1` list endpoints get `offset` + `total` so integrations can page a full dataset (previously hard-capped per call).
- **Duplicate search + merge** — `GET /api/duplicates/{entity}` groups by email / normalized name; `POST /api/duplicates/{entity}/merge` fills the survivor's empty fields, unions tags, **relinks references** (deals, tasks + link arrays, campaign_recipients, inbound_messages, contact_company_links, company FKs) and soft-deletes the losers into Trash. "Duplicates" button on Leads/Contacts/Companies opens the finder dialog (pick survivor → merge, or delete singles). Add-Lead warns live on duplicate emails.
- **Smart Search works without AI** — the AI intent/summary calls are individually guarded; keyword + graph search now run on a plain regex fallback when no Anthropic key is configured (was: "0 results" for data that exists).
- **Deal outcomes + audit trail** — `won_date`/`lost_date`/`lost_reason` stamped on stage change with a **lost-reason prompt** in the UI; field-level audit events (stage, value, probability, close date, owner) recorded and rendered in the entity history timeline.
- **Email compliance** — RFC 8058 `List-Unsubscribe` + one-click POST endpoint (HMAC-signed), visible unsubscribe link, and an org-level **imprint footer** (Settings → Email) appended to every campaign send.
- **Trash auto-purge** — daily job hard-deletes soft-deleted rows past `organizations.trash_retention_days` (default 90; 0 disables) — closes the GDPR storage-limitation gap.
- **UX/i18n batch** — error cards with Retry instead of infinite skeletons, unsaved-changes guards on edit dialogs, bulk-delete confirmation dialogs, onboarding checklist derives state from real data, +185 locale keys (full en/de parity, `frontend/scripts/check-locales.js` guards it in CI), login-page copy fixes, undated demo-page promise.
- **Dev env** — `UPLOAD_DIR` env-configurable (backend runs outside Docker again), `DISABLE_VISUAL_EDITS=1` escape hatch for `craco start` on newer Node, backend requires Python 3.11 (pins don't resolve on 3.13+).

### Mid June 2026 — LLM/AI-search visibility for the marketing site

tako.software was invisible to AI search: the raw HTML response was an empty CRA shell ("You need to enable JavaScript…"), so GPTBot, ClaudeBot, PerplexityBot and Bingbot (which powers ChatGPT search) indexed nothing.

- **Build-time prerender** — `frontend/scripts/prerender.js` runs as `postbuild`: headless Chrome renders `/` and `/partners` against the fresh build and writes the full HTML into `build/` (`index.html` + `partners.html`), with hard content assertions (headline, four pillars, agents, pricing, compliance claims) so a blank render fails the deploy instead of silently shipping an empty shell. React boots on top of the static DOM unchanged. The alpine Docker image sets `SKIP_PRERENDER=1` (no headless Chrome on musl) — Netlify is the public serving path and prerenders on every deploy.
- **Owned Business Suite head metadata** — new title/description, canonical, OG/Twitter cards, and JSON-LD (`SoftwareApplication` + `Organization`) in `public/index.html`; `/partners` keeps its own title/meta/canonical in its prerendered file. New `og-cover.png` (1200×630) replaces the stale "pays you back / start free" social banner.
- **robots.txt + sitemap.xml** — real static files in `frontend/public/`, served ahead of the SPA fallback (previously both returned the app shell).

### Early June 2026 — custom fields, appointment reminders, operator UX polish

The "make the partner ecosystem actually possible" pass — every org can now extend the CRM schema themselves without touching the core. Plus a chunk of UX polish that closed gaps Florian flagged from daily use: appointment reminders that actually fire, edit dialogs that close on save, calendar pills that go where you expect.

- **Cross-tab account-switch session sync** — Reported symptom: another account's bell notifications appearing inside an already-open tab. Root cause was not a tenant-isolation breach — the notification API is correctly org-scoped — but the single-origin auth model: the access token lives under one global `localStorage` key shared by every tab, so signing into a second account overwrites it underneath an open tab, which then keeps rendering the first account while its API calls (notably the 30s bell poll) use the new token. `AuthProvider` now listens for the `storage` event and reloads the tab when the token's account identity (`user_id`, decoded from the JWT) changes, re-syncing the tab to the active session. Keyed on `user_id` rather than the raw token so routine silent-refresh rotation doesn't reload sibling tabs of the same account.
- **CSV import ReferenceError fix** — The Companies "Import CSV" handler spread an undefined `headers` variable, throwing `ReferenceError: headers is not defined` the moment a file was chosen, so the upload never fired. Now reads the auth header from `getAx()` like every other request on the page.
- **Custom Fields v1 + v2** — Partner ask (2026-05-15): *"give the admin dashboard an Add Custom Field button so users can extend records themselves; the master system stays clean."* Shipped both phases in one week:
  - **v1 (contacts, #73)** — New `custom_field_definitions` collection (per-org). Admin → Custom Fields tab with full CRUD; admins create text or dropdown fields keyed by a snake_case slug, unique per `(org, entity_type)`. Read open to org members (the contact dialog needs the schema); write admin-gated. Renames allowed; key / field_type / entity_type immutable after creation (mutating those would silently orphan existing values). Deleting a definition does NOT scrub contact data — recreating the same key restores the historical values, matches the partner's "reversibility" promise. `ContactCreate` gets `extra="allow"` so the `custom_fields` dict flows through unchanged.
  - **v2 (leads / companies / deals + list filtering, #74)** — `CUSTOM_FIELD_ENTITY_TYPES` extended to all four core CRM entities; `LeadCreate` / `CompanyCreate` / `DealCreate` get the same `extra="allow"`. New shared `CustomFieldsEditor` + `CustomFieldsDisplay` components used in every entity's detail/edit dialog. New `CustomFieldsFilterBar` (collapsible row above each list table) with one control per definition: dropdown → `<select>` with values + Any + (Empty); text → substring contains. AND semantics across active filters. On DealsPage the filter applies before the kanban groups by stage so totals stay correct. Filter state is client-side per page; backend filtering ships when a partner hits a perf wall.
  - **Settings tab placement fix (#76)** — Florian flagged: as 4Rooks org admin he hit Access Denied on `/admin` (super-admin-only by design). Custom fields are an org-level feature; the backend endpoints already allowed `owner / admin / super_admin / deputy_admin`. Tab moved (well, duplicated) to **Settings → Custom fields** with the same role gate; `/admin` tab kept as a super-admin convenience.
  - **Out of v2** (flagged in scoping replies): list-table column visibility for custom fields, required-field enforcement, number / date / boolean / multi-select field types.
- **Appointment-due reminders + per-user chime mute** — TAKO-native only (Google fires its own reminders). 60s `appointment_reminder` cron in `backend/scheduled/appointments.py` covers two collections:
  - `scheduled_calls` (legacy) — keeps its existing `call_reminder` notification type so the bell's styling/icon mapping stays intact.
  - `calendar_events` — new `reminder_at` / `reminder_minutes` / `reminder_sent` columns. Default lead time 5 min, set on POST/PUT/booking-confirm. PUT recomputes when `date` or `reminder_minutes` change so a rescheduled meeting re-fires.
  - Stale-event guard: events that already started >60s ago skip the chime but still flip `reminder_sent` so we don't keep retrying.
  - Frontend `NotificationBell` synthesises a short two-tone WebAudio chime (C5 → E5, 140 ms each) and pops a sonner toast with an Open action linking to `/calendar?event=…` or `/calls`. `localStorage` tracks the last-chimed notification ID so a reload doesn't re-fire; a baseline pass on first fetch suppresses chimes for historical reminders.
  - **Per-user mute** — `User.appointment_chime_enabled` (default true; missing → true for back-compat). Toggle on Settings → Profile. Held in a ref so a toggle takes effect on the next 30s poll without re-arming the baseline guard. Mute gates the chime only; the toast still appears so muted users don't miss the alert visually.
  - **Reminder lead-time picker** — *Remind me* dropdown on Create + Edit Event dialogs: No reminder · 5 / 10 / 15 / 30 min · 1 hr · 1 day. Defaults to 5; legacy events without the field show 5 in the dropdown but the value only persists on user save. Hidden for Google events.
- **Cron health watchdog** — Single failing scheduled job used to log but never alert. APScheduler event listeners in `jobs.py` stamp every job's last run into `db.job_runs_latest` (status, timestamp, truncated traceback, consecutive-error streak). New 5-min `cron_health_watchdog` checks two signals: `next_run_time` >10 min stale (scheduler wedged) and `consecutive_errors >= 3` (job repeatedly failing). Either fires a `job_health_alert` notification to every super-admin user, with 1-hour dedup per `job_id` so a stuck job doesn't spam the bell. `GET /api/admin/jobs/health` returns per-job health rows for ad-hoc inspection.
- **Constellation usability + several reliability fixes** — The Phase 3 Constellation tab had three real problems that surfaced once Florian's queue grew past a few hundred linked entities:
  - **Crash on bad deal value (#67)** — `_entity_value_label` did `int(value/1000)` on a deal whose `value` was a free-typed string. TypeError → uncaught 500 → "Failed to load constellation". Now `float()`-coerced with try/except, per-entity build loop wrapped so a single bad row doesn't kill the payload, outer endpoint surfaces the actual exception class+message in the error banner instead of a generic placeholder.
  - **Legacy `related_lead_id` invisible (#68)** — The endpoint only walked the Phase-B `links` array; campaign-runner tasks store their linkage in the legacy `related_lead_id` FK. Florian had 28 tasks with zero entities. `lead` is now a 6th entity type, with `related_lead_id` folded into `refs_by_type` alongside the new array. New empty-state copy distinguishes "no tasks" from "tasks but no links" — *Nothing to map yet. You have N tasks, but none are linked to a deal, contact, lead, project, company, or campaign.*
  - **Sort TypeError on mixed datetime/str created_at (#69)** — Cross-entity-link aggregator sorted `tasks` by `created_at`. Some legacy rows store `created_at` as a raw datetime (Motor decoded from BSON `Date`); newer rows are ISO strings. Python's sort blew up on the first cross-type comparison. Sort key coerced through `str(...)`.
  - **Focus filter for 296-entity overload (#70)** — The constellation collapsed into an illegible perimeter blur once entity count crossed ~30. New **Focus** dropdown defaults to *Needs attention* — keeps only entities with `urgency_signal` in `{stalled, win_window, overdue_tasks}`. Hard cap of 24 visible entities even in *All my work* mode (sorted by signal then `task_count` so the busiest rows always make it). Header chip shows "Showing 24 of 296 entities" so the trim is visible. Filter mode persists in localStorage.
- **Calendar task pill click → full task edit dialog (#70 + #71)** — Was: clicking a yellow task pill dropped you on a blank `/tasks` page. Now: deep-links to `/tasks?task=<id>&edit=1`. TasksPage consumes `&edit=1` once and auto-opens the full edit dialog, then strips the param so a refresh doesn't re-pop. Routing centralised inside `openEventDetail` so every click site (timed event, all-day strip, "+N more" overflow popover) routes identically — also catches deal-close pills (→ `/deals?detail=<id>`).
- **Auto-close edit dialogs on save** — Florian: *"When I save a contact, etc. (after editing) it should autoclose."* Save handlers in ContactsPage, LeadsPage, CompaniesPage, DealsPage now `setSelected*(null)` + `setEditData({})` + `setEditMode(false)` on success. Toast confirmation + list refresh already fire, so the operator gets feedback without the stale dialog blocking the next move.
- **Edit email accounts + Settings → Email tab** — Pencil-edit button on every email account row at `/settings/email`. Modal lets the operator change display name, IMAP host/port/SSL, SMTP host/port/TLS, signature HTML, and rotate the password. Password rotation re-encrypts under the same `credentials_id` so the FK on the account row and the email_links rows survive. Backend re-runs the IMAP + SMTP probe before persisting whenever server fields or the password change, surfacing the real IMAP/SMTP error if anything fails to bind. Sync status resets to `ok` on a successful save. Plus: the `/settings` tab strip now has a dedicated **Email** entry (right after Profile) so the panel is reachable without typing the URL. `EmailSettingsPage` body refactored into a reusable `EmailAccountsPanel` (named export) used by both the standalone route and the new tab.
- **Chat-message echo fixed (#67)** — Own messages used to appear twice on first send and vanish on refresh. Root cause: `handleSendMessage` optimistically appended the response, then the next `/chat/messages/new` poll re-pulled the same message because `lastFetchTime` was stamped with the browser clock while the server-side filter compared against `created_at` (server clock). Tiny skew was enough to slip through the `since > ` filter. Now `pollNewMessages` dedupes by `message_id` against current state — idempotent regardless of clock drift.
- **Calendar 07:00 stacked-tasks bug** — Tasks with date-only due_dates (Pydantic's `Optional[datetime]` coerces `"2026-05-14"` to `"2026-05-14T00:00:00"`) were missed by the old all-day heuristic and clamped to `top:0` on the timed grid — they piled up at the 07:00 row in a perimeter blur. New `_is_date_only_or_midnight` helper treats midnight (with or without tz suffix) as all-day. Applied to tasks AND deals.
- **Admin tab strip wraps (#78)** — Adding Custom Fields pushed the `/admin` tab count to 11; *Settings* fell off the right edge at narrow viewports. `flex flex-wrap h-auto gap-1 p-1` on the TabsList so tabs wrap to a second row. Same rule SettingsPage already uses.
- **Recurring calendar events** — Create + Edit event dialogs gain a *Repeats* picker: Does not repeat · Daily · Weekly · Monthly · Yearly, with interval (every N) + end condition (Never / After N occurrences / On a specific date). Backend stores a compact `recurrence` dict on the parent row and expands instances at `GET /calendar/events` time within the requested window (safety cap 200 instances per row per request). Recurring instances carry `series_date` so editing an occurrence edits the series anchor rather than silently moving the whole series. **Reminder v1 only fires for the parent date** — multi-instance reminders flagged as a v2 follow-up.
- **Google Calendar entries block booking availability** — Public booking previously only honoured TAKO-native `calendar_events.blocks_booking == true`. Google-side meetings on the host's primary calendar now also block availability: the `/booking/{user_id}/available` endpoint walks the host's cached `google_calendar_events` for the target day (deleted-excluded, all-day skipped) and merges the busy intervals into the same set used by TAKO events + scheduled calls + existing bookings. No-op if the host hasn't connected Google.
- **Auto-close calendar event edit dialog on save** — `handleSaveEvent` used to drop to read-only view (still inside the dialog) after a successful save. Now closes the dialog entirely + clears `editEventData`, matching the auto-close pattern shipped earlier for contacts / leads / companies / deals.
- **SearchableSelect search fixed inside dialogs** — The Lead / Contact / Company type-ahead pickers (Create + Edit Deal) had a dead search box: clicking it never let you type. Cause: the popover portals to `document.body` to escape the dialog's overflow clipping, but that puts the search input outside the Radix Dialog's FocusScope, whose `trapped` focus-out handler yanked focus straight back into the dialog. Fix: capture-phase `focusin` / `focusout` listeners on `document` that `stopPropagation()` for popover-bound focus events before Radix's bubble-phase trap fires — surgical, only touches focus moves involving the picker.
- **Add Deal from Company** — Company detail dialog gains an "Add Deal" button that opens the Create Deal dialog pre-linked to that company (`/deals?create=true&company_id=<id>`). The deal-create deep-link handler already accepted `lead_id` / `contact_id`; `company_id` joins them.
- **Calendar invites actually deliver — email + bell (#82, #83)** — Adding an invitee in the Create / Edit Event dialog used to store the email on the event silently: no email, no in-app ping (Florian: invited Kate, she got nothing). New `_notify_event_invitees` helper now fires from create, edit (newly-added only), and the standalone `/invite` endpoint. Two independent channels: an in-app **bell notification** (`event_invite`) for every invitee email that matches a user in the org — reliable even if email bounces — plus a best-effort Resend email with an `.ics` attachment for teammates and external guests alike. **#83 follow-up:** the explicit Send-invite action notifies the *full* submitted list (not just newly-added), so re-sending an invite to someone already on the event — the "she didn't get it, send again" case — actually re-fires; and the org-member email match is case-insensitive so a casing mismatch (`Kate@…` vs `kate@…`) can't silently drop the bell.
- **Company typeahead on lead → contact convert (#82)** — The Convert dialog's company field is now a live typeahead over existing companies. Picking a suggestion links the new contact to that exact company (`company_id`) instead of spawning a near-duplicate; editing the free text without picking falls back to resolve-or-create on the typed name. Backend `convert-to-contact` accepts an optional `company_id`.
- **More auto-close on save (#82)** — The Projects task-detail edit now closes the dialog on save (and refreshes the project), matching the auto-close pattern already applied to contacts / leads / companies / deals / calendar events / booking settings.
- **Event RSVP — accept / decline + status (#84)** — Invitees can now respond to a calendar event. The event detail dialog shows each invitee as a status chip (Pending / Accepted / Declined); if the current user is on the invitee list, Accept / Decline buttons appear with the active choice highlighted. `POST /calendar/events/{id}/respond` records the answer in an `invitee_status` map (keyed by normalised email) and bell-pings the event creator (`event_rsvp`) so they see who responded. Status is initialised to `pending` whenever someone is invited (without clobbering an existing answer on re-send). v1 is in-app for TAKO users; external-guest RSVP via tokenised email links is a future follow-up.
- **Detail dialogs no longer re-open after save (#85)** — The `?detail=<id>` deep-link effect on Contacts / Leads / Companies / Deals re-ran on every list change. Saving an edit calls `fetch*()`, which mutated the list, which re-fired the effect and re-opened the just-closed dialog — so a successful save *looked* like the popup refusing to close (Florian: "clicked save and the pop-up stays open"). The save was fine; the dialog was being re-opened. Fix: each deep-link effect now consumes the param exactly once (ref latch) and strips `?detail` from the URL, so a post-save list refresh can't resurrect the dialog. (Calendar `?event` and Tasks `?task` already had this latch; Campaigns strips immediately; Projects runs once on mount.)
- **SearchableSelect option click fixed inside dialogs (#86)** — Sibling to #81. The picker's option click did nothing — the dropdown closed without selecting (Florian: "I can't select the company; it shows now, but clicking it just closes again"). Cause: Radix Dialog's modal pointer-events lock sets `body { pointer-events: none }` and re-enables it only inside the dialog content; our popover is portaled to `document.body`, so it inherited `pointer-events: none` and option clicks passed straight through to the layer behind, tripping the outside-click close. (Search still worked because the input auto-focuses programmatically and keyboard input doesn't need pointer-events — which is why #81 looked complete.) Fix: `pointer-events: auto` on the portaled popover.
- **Deals show their company at a glance (#86)** — Deal kanban cards and the deal detail header now show the linked company name (and contact, when both are set) under the deal name, resolved from the loaded company/contact lists. A deal called "POC" or "Renewal" is finally identifiable without opening it.
- **Partner-application bot defense + review surface (#87)** — A bot wave was flooding the public Founder Tier application form, and every submission fired the full admin cascade (bell + WhatsApp + email). Layered zero-dependency defenses on `POST /partners/agency-application`: **honeypot** (hidden phone field; filled → silently dropped with a fake-success so scripts don't adapt), **timing check** (form stamps `form_started_at` on mount; missing or <4s → spam), **per-IP rate limit** (3/24h → 429), **30-day per-email dedup** (returns the existing id, no re-notification), and **content heuristics** (URLs in name/company, link-stuffed or gibberish text). Heuristic hits are SAVED with `status="spam"` + `spam_reason` but skip the whole notification cascade and the applicant auto-response — a false positive costs a delayed review, never a lost lead. New **Admin → Partners → Founder Tier applications** panel: status chips with counts (New / Spam / Reviewed / All), per-row mark-reviewed / mark-spam / restore, expandable clients-text, submit IP, and a one-click **Purge spam**. Endpoints: `GET/PUT /admin/partner-applications*` + `POST /admin/partner-applications/purge-spam` (super-admin).
- **Support overhaul: training + FAQ comparison & cost-benefit (#89)** — The /support Feature Training was four features behind the product. Rebuilt around a bilingual content module (`supportContent.js`, LandingPage precedent for long-form copy): **15 training modules** (was 8) now covering Inbox/TAKO Mail, Bookings, Team Chat, Contacts & Companies, Projects & Files, Custom Fields, Capture/⌘K and the AI toolkit alongside refreshed versions of the original eight. Every module gains an **"Open in TAKO" deep link**, **best-practice tips** (amber callout), and a **screenshot slot** — drop `<key>.png` into `frontend/public/training/` and it renders automatically (missing files hide silently; see the README in that folder). The FAQ tab gains two researched sections: **TAKO vs. the market leaders** — the landing page's three published comparison tables (CRM vs HubSpot/Salesforce/Pipedrive, Projects vs Asana/Monday/ClickUp, Booking vs Calendly/Chili Piper/SavvyCal) rendered in-app with an honest "where the big suites win" note — and **Self-hosting vs. SaaS: the 3-year math**, a worked 10-user example (≈€25.5–42.5k SaaS stack vs ≈€7.7k TAKO all-in) with assumptions stated, plus five new Q&As covering operations, maintenance lapse behaviour, GDPR posture, and when SaaS is honestly the better choice. Full EN + DE.

### Late May 2026 — Booking v2 Phase 1: named meeting types, caps, idempotency

- **Named meeting types** — booking settings now define a `meeting_types` array replacing the old flat `meeting_durations` list (kept one release for backward-safe migration). Each type carries its own label, duration, buffer, daily cap, minimum notice, booking horizon, video config, and active flag; operators configure them in Settings → Booking.
- **Per-type daily caps** — each meeting type can independently cap the number of bookings accepted per day; the availability endpoint returns `at_capacity: true` when the cap is reached for a given type on a given date.
- **Minimum notice & booking horizon** — per-type controls block bookings made within X minutes from now (minimum notice) or beyond Y days out (booking horizon), preventing last-minute surprises and runaway future availability.
- **Idempotency on book** — `POST /booking/{user_id}/book` now accepts an `Idempotency-Key` request header; a retrying client or agent replaying the same key gets the original booking back instead of creating a duplicate.

### Late May 2026 — TAKO Mail Phase 1, Today's Focus, contact-company auto-promote

The "30 minutes a day" plumbing. Email integration earned its way past the no-features-until-first-sale gate because every sale gets closed over email; without it the pipeline view lies. Plus the smart task prioritisation that turns the morning open-TAKO into "here are your top 3" rather than "here are 200 things, good luck."

- **TAKO Mail Phase 1** — IMAP-read + SMTP-send, personal accounts + admin-only shared mailboxes, manual log-to-CRM, AI summary on every inbound, sent-folder sync, compose with tiptap + variable templates + localStorage drafts, setup wizard for the four major providers (Gmail / Outlook / iCloud / Custom). Strategy memo locked six decisions: IMAP first (OAuth → Phase 3), manual auto-link (the user decides what's a CRM artefact), metadata + summary only (NEVER bodies — fetched on demand from your IMAP server), sent-folder mirror, admin-controlled shared mailboxes, NO tracking pixels ever. New `backend/mail/` package (NOT `backend/email/` — the latter shadows Python's stdlib). 5 new collections (`email_accounts`, `email_credentials_encrypted`, `emails`, `email_links`, `email_assignments`) with 13 indices. PyNaCl secretbox credential encryption gated by `TAKO_EMAIL_CRED_KEY` (32-byte master key — backend refuses to start without it). Master polling worker at 30s interval, per-account cadence 60s active / 5min sleeper, 200-UID cap on cold-start. Production setup walkthrough at [`docs/email/PRODUCTION_SETUP.md`](docs/email/PRODUCTION_SETUP.md). 60+ backend pytest tests + 6 frontend smoke tests covering every public surface.
- **TAKO Mail Phase 1.5–1.7 follow-ups** — Bell pings on inbound from a known CRM contact / lead only (newsletters silent; bell-spam guard). Backfill skips AI summary + bell so cold-start doesn't burn the daily AI cap or dump 200 notifications on day-one connection. Server-side delete reconcile every poll cycle (UID compare; soft-deletes locally any row whose UID is gone from the server) so deletes in Apple Mail / Gmail web propagate to TAKO. Live-fetch 410-on-deleted-UID path: if the user clicks a row that was deleted server-side between cycles, the backend recognises "UID not found" specifically, soft-deletes the row, returns 410 Gone with a friendly message — frontend drops it from the list. Refresh button now triggers an actual IMAP poll (not just a DB re-paint) so the gap closes in 1-2s instead of 60s. Hover-row trash icon (Gmail-style quick-delete) AND detail-header Delete button — both open the same DeleteEmailDialog with optional "also remove from real mailbox" checkbox (IMAP `STORE \Deleted` + EXPUNGE). 410 fix on `/api/email/inbox` 500 (the `asyncio.create_task` on the activity-bump was rejecting Motor's Future-like return).
- **Today's Focus card + daily TaskRanker (Phase B)** — Mon-Fri 06:55 Europe/London cron iterates every user with at least one open task, ranks via Claude under their `priority_mode` (Revenue / Cost / Balanced), writes one row per user to `db.task_rankings`. `/tasks` page renders a "Today's focus" card with the top 3 still-open tasks; rolling client-side promotion as you complete them. Three-mode pill switches mode + instantly re-ranks. Manual Refresh re-ranks on demand (rate-limited 1/min/user). Strict-JSON parse with valid-task-id clamp + rank renumber. Cap-block returns empty rather than crashing. Greyed-out "Include private life" toggle reserved for the future Private Chores project. Spec doc at [`docs/operations/2026-05-03-task-prioritisation-agent-phase-b.md`](docs/operations/2026-05-03-task-prioritisation-agent-phase-b.md).
- **Auto-promote contact.company → Company entity + link** — Florian flagged: Lennart Cornelsen had `company="Pantaenius"` (legacy free-text string) but the Companies multi-link section read "No companies linked yet". Lead-conversion already auto-created the Company entity via `_resolve_or_create_company_from_lead`; every other contact-create path (direct form, manual edit, CSV import) just stored the legacy string. New `_resolve_or_create_company_for_contact` sibling. Wired into `POST /api/contacts` and `PUT /api/contacts/:id`. One-shot `Backfill contact-company links` workflow runs the same logic retroactively (idempotent, dry-run supported) — fixed every existing affected contact in production.
- **Emails on the entity activity feed** — `GET /entity/:type/:id/history` learns a new event source from `email_links` joined to `emails`. Sent emails (composed via the entity-page Email button) show as `Sent email · <subject>` (Send icon, indigo) ; logged inbound emails show as `Received email · <subject>` (Mail icon, rose). Click-through to `/inbox?email=<id>` deep-links to the message. New "Mail" filter chip on the timeline distinguishes from the existing "Campaigns" chip (campaign-recipient rows).
- **Searchable Lead / Contact / Company pickers in Create + Edit Deal** — flat radix Selects became unusable past ~10 entries. New `<SearchableSelect>` component with type-ahead, options carry a subtitle (Lead → company, Contact → job_title, Company → industry), portal + `position: fixed` from the trigger's bbox so the popover escapes ANY ancestor's overflow (the original Edit Deal dialog clipped the popover on the right and labels rendered as "ntaenius" instead of "Pantaenius"). Right-edge collision detection auto-anchors right when near the viewport edge. Min-width 240px. Reusable across the app.
- **Linked entities section on Deal detail** — read-only mode now surfaces who the deal is with: Lead / Contact / Company rows with icon + clickable name (deep-links via `?detail=<id>`) + metadata cluster (job_title / email / industry / website, truncating as one block). Email button auto-pre-fills To from the linked contact (or lead fallback, or company's primary contact); fully editable in the compose modal. Layout uses `max-w-2xl` + flex `min-w-0` truncation so long deal names + linked-contact lines no longer leak past the dialog edge.
- **AdminPage — superadmin can edit org licence + AI trial + demo state** — for hosted-by-us / comp / extended-trial accounts. `PUT /admin/organizations/:id` allow-list extends to `ai_trial_ends_at` + `is_demo` + `demo_status` + `demo_expires_at` (with ISO datetime validation). Frontend gets an Edit pencil per org row that opens a sectioned modal (Identity / Licence / AI Trial / Demo state) with inline copy explaining what each control actually drives.
- **Sidebar Inbox entry + `/inbox?email=<id>` deep-link** — Inbox lives under the Engagement section in the left rail (EN: "Inbox" · DE: "Posteingang") with the Mail icon. Settings → Email Accounts reachable via `/settings/email`. The inbox view honours `?email=<id>` deep-links from the bell-ping notifications and the new HistoryTimeline drill-down.
- **`User.language` field** — added so the email summariser knows which language to write summaries in for inbound mail received while the user isn't online. Frontend pushes the localStorage `tako_lang` value via `PUT /auth/me` on every language toggle. Shared mailboxes inherit the org owner's language.

### Mid May 2026 — Tasks v2 closing chapter + design consolidation across the app

The Tasks v2 redesign closed in three phases (2 / 2.1 / 3), then a breadth-first design pass swept the same editorial vocabulary across nine entity-list pages.

- **Tasks v2 Phase 2 — AI layer** — Sentinel email-derived suggestions cron (Haiku triage → Sonnet extraction → grounded "why_now" reasoning), source-hash dedup against 7-day dismissed window + currently-pending, cap 5/user. Sentinel strip on /tasks with Accept / Dismiss. Context Lens right dock — slides in on task focus, shows linked entities + recent activity (per-entity history union) + Ask TAKO pane with conversation memory. NextSteps AI-suggested actions inside the Lens (Draft email / Add task / Block calendar). Workspace-level AI cap configuration with super-admin UI.
- **Tasks v2 Phase 2.1 — multi-collaborators + sub-nav + critical gaps** — Tasks gain `collaborators[]` (multi-person) alongside `assigned_to` (owner). Reassignment + collaborator-add fires `notify_user_cascade` (in-app + email; WhatsApp opt-in). Default Pulse scope is **"involved"** (owner OR collaborator OR `assigned_to=null`) — closes the pre-2.1 security hole where every user saw every task; orphan agent-created tasks (no `assigned_to`) stay visible to the org via the null-assigned branch. Contextual left **TaskNav** with FOCUS · SURFACES · SMART LISTS · BY OWNER (admin-only). Smart-list registry with 4 outcome-driven lists (Won deals to onboard · Cold leads warming · Pricing-page hits as `available=False` placeholder · Champions gone quiet). **Route-order hot-fix**: pre-2.1 the parameterised `/api/tasks/{task_id}` was declared before the new static `/tasks/smart-list-counts` and `/tasks/suggestions` routes — FastAPI matched the literal paths as task IDs and 404'd. Startup hook now sorts static `/tasks/*` paths ahead of `/tasks/{task_id}` (regex-based, prevents same-class regressions). Plus three smoke-test bug fixes: UserPicker click commits before the autofocused search input blurs, dashboard task → `/tasks?task=<id>` deep-link opens the Lens, NextSteps card hover discoverability.
- **Tasks v2 Phase 3 — closing chapter** — Constellation tab (radial SVG graph: entities arranged on a circle, tasks orbit, halos tinted by urgency, curved arms, cross-entity dashed teal links, pulsing overdue ring; 1h server-side cache). Calendar tab + auto-scheduler (`POST /api/tasks/schedule` reads busy intervals from Google Calendar cache + scheduled_calls + pinned tasks, greedy-fits deep-work to morning + admin to afternoon). Global ⌘K palette (`POST /api/tasks/parse` calls Haiku, 300ms debounce, ~$0.001/call, modifier-gated so plain "k" doesn't trigger in inputs; mounted once at app root inside `<AuthProvider>`). Janitor Sentinel slice (stalled deals; cap 3/user/day; ISO-week-anchored source-hash so persistent staleness gets fresh weekly nudges; same Mon-Fri 08:30 + Mon 06:00 cron). All four marquee features ship with 116 tests across 5 backend + 3 frontend suites.
- **Design consolidation across nine entity pages** — `frontend/src/components/entity/entity-styles.js` lifts the editorial `.tako-pulse-*` vocabulary from Tasks into a shared CSS module: paper-bg list rows with Fraunces titles + smoke meta lines (dot separators) + JetBrains-mono numerics, token-driven filter bars + filter chips, hover-only action menus, bulk-action bar. Per-row AI buttons on Leads (`AI Summary` / `Draft Email` / `Discuss`) hide behind `.tako-entity-row-hover` so they only appear on hover — fixes the "1080 teal pills" problem. Applied across Leads, Contacts, Companies, Deals (Kanban + List), Campaigns, Files, Listeners, Projects, Calls. Detail/edit dialogs across all nine pages intentionally untouched — Phase F (dialog migration) ships next session. PipelineReportPage flagged as analytics-shape (stat cards, stage-comparison grid, comparison tables) — needs its own dedicated session for analytics primitives, not entity-row vocabulary.

### Early May 2026 — launch hardening (Stripe, /demo, CRM graph, ops workflows, system-mail filter)

The "make it actually launchable" pass. Closed every gap between the public marketing surfaces and the backend reality, plus the CRM graph + ops tooling that was due for the outreach push.

- **Stripe pricing fully wired** — Four products live in LIVE-mode Stripe (`tako_selfhost_once` / `_12mo` / `_24mo` / `tako_maintenance_yearly`), all four price IDs in production env, webhook secret (`whsec_…`) verified, and the webhook handler now covers `checkout.session.completed`, `invoice.paid`, **`invoice.payment_failed`** (smart-retry friendly — no licence deactivation on first failure, just stamps `last_payment_failed_at` and emails the customer), and `customer.subscription.deleted`. `setup-stripe-products.yml` probe step reports presence of all six Stripe envs without leaking values.
- **30-day money-back refund** — `POST /api/admin/refund/{org_id}` (super-admin) implements the public-page guarantee. One-time licences refund the payment_intent in full; instalment licences cancel the subscription AND refund every paid invoice. 30-day window enforced; `{"force": true}` overrides (audit-logged). Idempotent. ToS template fixed from "14 days" → "30 days" to match the rest of the site.
- **`/demo` is a real page** — was a `Navigate` redirect to `/signup?demo=1` (which SignupPage ignored), bouncing cold traffic. Now an honest intent-capture page with a "Book a 15-minute walkthrough" primary CTA pointing at the operator's native booking link, plus a "notify me when the sandbox is live" email-capture wired to `/api/newsletter` (`source: 'demo_page'`). The auto-trial spin-up still exists post-signup via SetupOrgPage's "Try TAKO free for 14 days" button.
- **Testimonials removed** — three placeholder quotes (Marcus W. / Sophie L. / James R.) deleted from the landing page along with the orphan i18n keys. Under UK CPUT 2008 / EU 2005/29/EC fabricated testimonials are an unfair commercial practice; the existing `europe` GDPR/EU/self-hosted block carries the positioning instead. Also fixed the "Join hundreds of European sales teams" claim in `ctaDesc` (en + de) — replaced with self-host positioning that doesn't require validation.
- **Partners FAQ cleanup** — removed the "What is alakai.digital?" entry (would have been a fabricated proof point under CPUT 2008) and replaced the "Direct Slack channel with the TAKO team" Agency Partner card bullet with "Direct messaging with the TAKO team in-product" (no Slack channel exists; in-product chat is the actual support path).
- **Contact-Company graph (CRM Phases 2 + 3 + 4)** — typed multi-value email/phone arrays across Lead/Contact/Company (Shape A: `[{value, label, primary}]` with primary mirrored to the legacy scalar field for back-compat), M:N `contact_companies` join collection with `role` + `is_primary`, lead-conversion auto-creates company docs + join rows, smart-search now walks the graph (matched company → its contacts + leads, matched contact → its companies, matched lead → its company) with `_via` badges. Company detail dialogs show their linked Contacts AND Leads. Migration workflow ships the Phase 3 backfill with idempotent dry-run.
- **System-mail / DMARC inbound filter** — inbound webhook now silently swallows DMARC aggregate reports (Microsoft, Google, Yahoo all send daily) and other auto-mailer traffic. Two-layer check: local-part match against a curated list (`mailer-daemon`, `postmaster`, `dmarcreport`, etc.) plus RFC 3834 / 2076 auto-submission headers. No more "could not match a campaign or lead" alerts in the operator inbox for system mail.
- **One-click ops workflows** — `verify-campaign-runner.yml` (APScheduler probe + optional fire), `migrate-contacts-companies.yml` (Phase 3 backfill), `smoke-probe.yml` (generic Python probe inside the prod backend), Stripe-probe extension on `setup-stripe-products.yml`. All gh-Actions-triggered, all idempotent, all honour the shared `deploy-prod` concurrency group so they never race the backend deploy.
- **Booking confirmation flow** — public bookings now go through a `pending_confirmation` → single-use token → `/book/confirm/:token` flow. Side-effects (calendar event creation, host notification cascade) are deferred to confirm so unconfirmed bookings expire harmlessly after 24h. Closes the "ghost bookings" gap where typo'd email addresses left cruft on the calendar.
- **Footer + FAQ launch polish** — newsletter form stacks input + button on small viewports (no more overlap with the GDPR Statement link), legal block uses Companies House + registered-office address + null VAT (hidden), Slack social icon hidden until provisioned.
- **Concurrency safety on deploys** — backend + frontend deploy workflows now share a `deploy-prod` concurrency group with `cancel-in-progress: false`, so two pushes to main queue instead of racing each other. Resolved a 7-minute API outage from the original race.

### Late April 2026 — outbound machine, chat overhaul, AI-native company picker, closed-loop replies

The biggest single push in the project's life. Roughly: turned campaigns from a draft tool into a real outbound machine *with a closed reply loop*, replaced the toy Team Chat with a Slack-shaped one, and made adding/managing companies AI-native end-to-end. Highlights:

- **Campaign per-recipient delivery tracking** — new `campaign_recipients` collection with status vocabulary (`pending` / `queued` / `sending` / `delivered` / `bounced` / `recipient_unknown` / `inbox_full` / `deferred` / `replied` / `opted_out` / `failed`). Aggregate counts denormalised on the campaign for cheap list rendering. Resend send pipeline tags every outgoing message with `campaign_id` / `organization_id` / `recipient_id` and captures the Resend message id back onto the recipient row.
- **Resend webhook ingestion + signature verification** — `/api/webhooks/resend` Svix-style HMAC-SHA256 verification, idempotent on `svix-id`, maps every event type onto our recipient status with **precedence guarding** (out-of-order events can't downgrade terminal states — a delivered arriving after a bounced no longer overwrites the bounce). `email.suppressed` mapped to `bounced`. Concurrent-retry `DuplicateKeyError` tolerated.
- **Inbound replies live** — `/api/webhooks/resend/inbound` running with a separate signing secret, Resend Inbound DNS configured (MX → `inbound-smtp.eu-west-1.amazonaws.com`), `Reply-To: tako@tako.software` on every campaign send, auto-forward of unmatched replies to the operator inbox, two-layer mail-loop guards (sender-domain match + `[TAKO reply]` subject prefix) after a runaway forward incident. Reply content (subject / from / body) is stored on the recipient row and surfaced inline in the campaign detail expand row + the per-entity history timeline.
- **AI suggestions on replies (Phase F)** — When a reply matches, Claude classifies intent (`interested` / `objection` / `not_now` / `out_of_office` / `unsubscribe` / `question` / `other`), drafts a contextual response, and proposes 0–3 follow-up tasks with priority + due dates. Surfaced inline with one-click **Send reply** (Resend send + `In-Reply-To` threading + `chat_audit` mirror) and one-click **Create task** (auto-linked to the underlying lead/deal). Three new endpoints: `POST /campaigns/{cid}/recipients/{rid}/{regenerate-suggestions,send-reply,create-task}`.
- **Lead-by-from-email fallback for replies** — When a reply doesn't match any campaign recipient (e.g. it's a reply to a transactional email sent outside TAKO, like the AIOS Institute field-note confirmation that hits Resend directly), the inbound webhook falls back to matching the from-address against `leads.email`. Match runs across all orgs and picks the most-recent lead by `created_at`, so a stale cross-org `campaign_recipients` hit no longer hides the reply from the operator's actual lead. Stored on a new `inbound_messages` collection (separate from `campaign_recipients` to avoid synthetic no-campaign rows); the per-entity history endpoint surfaces them as `inbound_email` events distinct from campaign-reply events.
- **Inbound body fetch via Resend API** — Resend's `email.received` webhook payload only ships metadata (subject, from, to, message_id) — the actual `text` and `html` body are missing on the basic event. We now follow up the webhook with `GET https://api.resend.com/emails/receiving/{email_id}` to pull the full body and persist it on `inbound_messages` / `campaign_recipients`. Empty-body retry on first miss handles the brief Resend-side propagation delay. Operators see the actual reply text in the lead history drill-down, not just "(no body)".
- **BCC outbound capture** (`docs/BCC-OUTBOUND-CAPTURE.md`) — When an operator replies to a lead from their own mail client (Apple Mail, Outlook, Gmail, etc.) and BCCs `tako@tako.software`, the inbound webhook recognises the BCC, identifies the operator by from-address (matching `users.email` OR `users.email_aliases`), finds the lead by To-address, and logs the message on the lead's history as an **Outbound email** event with the full body. Auto-forward back to the operator's inbox is suppressed for these (you sent it; don't echo it). Bell stays quiet too. Closes the "I replied from my own client and it didn't get logged" gap without a full Gmail/Outlook OAuth integration.
- **Email aliases on users** — New `email_aliases` array on `users`, plus `PUT /auth/me/email-aliases` self-service endpoint. The outbound matcher now does an `$or` against `email` OR `email_aliases`, so an operator whose TAKO login is `florian@floriankrueger.com` but who sends from `florian@fintery.com` gets correctly identified as the sender (rather than being misclassified as an inbound reply).
- **Inbound bell + history polish** — Inbound replies (campaign-reply OR lead-fallback) trigger a single notification in the topbar bell with a tone-coded entry; the history timeline distinguishes `email` (campaign-replies via `campaign_recipients`) from `inbound_email` (transactional replies via `inbound_messages`) from `outbound_email` (BCC capture) so the operator can scan a lead's correspondence at a glance.
- **Email editor with live preview + AI draft** — reusable `<EmailEditor>` component used in campaign create + edit flows. Subject + body + Markdown-ish formatting (`**bold**`, `*italic*`, `[link](url)`, `- list`, `# heading`) + live HTML preview. AI Draft (purpose + tone) embedded in the editor.
- **Email body visible everywhere** — campaign detail dialog read view shows the body in a "exactly what went out" panel; per-entity history drill-down shows the body inline. Single shared `renderEmailMarkdown` module so editor preview / campaign detail / history drill-down all render identically.
- **Per-entity History timeline** — new `GET /api/entity/{type}/{id}/history` aggregates email events (with full body), tasks, deals, conversion events, chat summary, plus creation/update anchors. `<HistoryTimeline>` component drops into Lead / Contact / Deal / Company detail dialogs with filter chips and per-row drill-down.
- **`/v1` write surface complete** — `tako_...` API keys now cover the full programmatic outbound loop: PUT lead, POST/PUT contact, full campaigns CRUD, send, recipients, bulk add-to-campaign. Field whitelists + per-org isolation preserved.
- **Multi-agent `sender_id`** — `/v1/chat/messages` now honors `sender_id` from the body so multiple AI agents (Maestro, Berny, etc.) sharing one API key post under their own user identities. Cross-org `sender_id` rejected with a deliberately ambiguous 400.
- **Chat overhaul** — five channel kinds (`general` / `public` / `private` / `dm` / `context`), explicit `members` + `moderators` roles, deterministic-id 1:1 DMs (`POST /api/chat/dm/{user_id}` is idempotent), promote-DM-to-named-private-group flow (≤ 10 members, one-way), full member/moderator management UI, visibility enforcement on every read path. `chat_audit` collection records every event; super-admin viewer at `/admin/chat/audit` with filters, includes DM content for forensics. `CHAT_AUDIT_ENABLED=false` env flag for restricted regions.
- **Chat unread indicators** — new `chat_read_state` collection (per-user, per-channel `last_read_at`, unique index). `GET /api/chat/channels` decorates each channel with `unread_count`; `POST /api/chat/channels/{cid}/read` marks read up to now; `GET /api/notifications` adds top-level `chat_unread_count`. Sidebar shows a small red pill (capped at 9+) next to every channel with unread, and bolds the name. The topbar bell lights up on new chat traffic and shows a single pinned "N new chat messages" row in its dropdown. Lazy-init on first GET sets `last_read_at = now` so existing users start clean (no flood of "99+" dots on rollout).
- **AI-native company picker** — typeahead in the Add Company dialog merges hits from your own data (existing companies + names referenced on leads/contacts) with Claude lookups (`/api/companies/suggest` + `/api/companies/suggest/ai`). Existing record clicks open the record instead of creating a duplicate.
- **AI Enrich for companies** — single (`POST /api/ai/enrich-company/{id}`) and bulk (`POST /api/bulk/enrich` with `entity_type=company`). Fills only empty fields; full enrichment payload (founded year, HQ, competitors, products, tags) stored under `enrichment` with a "Verify" pill in the UI.
- **Filtering** — Lead and Contact filters now include `tags` (CSV, ANY-match), `q` (regex search across name/email/company/title), `source`, `industry`, `decision_maker`, `has_email`, `campaign_id`, `not_in_campaign_id`. Backend `/leads/tags` and `/contacts/tags` populate dropdowns.
- **Admin user edit** — `PUT /api/admin/users/{id}` lets admins edit name, email, role, organization_id, email_verified flag in one shot. Edit dialog in AdminPage with diff-based partial-update payload.
- **Email verification idempotency** — verification token is no longer `$unset` on first success. Eliminates the prefetch-race where Brave's safe-browsing scanner / Apple Mail's link rendering / email security gateways consumed the token before the human click. Now any later hit (human or bot) hits the existing "already verified — idempotent success" branch.
- **Booking-confirmation emails fixed** — pre-existing `resend.emails.send` call (lowercase) was silently broken on `resend==2.21.0`; the SDK exposes `resend.Emails.send` (capital E). Same fix applied to the campaign send pipeline.
- **Operations** — backup script now does S3 + rclone offsite uploads (env-driven, no-op safe when unset). `scripts/restart-backend.sh` auto-detects host nginx and uses the right compose-file chain. `.github/workflows/deploy-backend.yml` auto-deploys backend to the VPS on push to `main`. First distribution tag cut: `v2026.04.26`. `/api/health` adds `resend_webhooks` status.

### Earlier April 2026

- **Security audit & hardening** — Dual-token JWT flow (15-min access / 7-day refresh) with rotation on refresh, per-IP login rate limiting, single-use 1-hour password-reset tokens, Stripe webhook signature verification with event-id idempotency, HTML escaping on invoices, session termination on org deletion, VAT-aware invoicing (`COMPANY_VAT_NUMBER`). Host-level guide added at `docs/VPS-HARDENING.md`.
- **Operations telemetry** — `GET /api/health` covers Mongo, email, Sentry, and Stripe readiness (200/503). Sentry SDK wired as a soft dependency on both backend and frontend. `scripts/backup-mongo.sh` with 7-day retention runs from cron.
- **In-app support tickets** — `POST /api/support/ticket` surfaces user-submitted issues with env/org/user metadata pre-filled in the super-admin inbox.
- **GDPR endpoints** — Self-service data export, account deletion with grace period, and a public DPA endpoint at `/api/legal/dpa`.
- **Distribution system** — `scripts/build-distribution.sh` produces a clean customer tarball with platform-only code stripped via `DEMO_BEGIN`/`PLATFORM_BEGIN` sentinel regions. Output lands in `DISTRIBUTION_DIR` (`./dist` by default).
- **Self-serve demo** — Landing-page `/demo` spins up a seeded 14-day trial org. APScheduler runs daily expiry; expired tenants soft-lock via middleware (writes blocked, reads open). Super admins extend/expire/purge from the admin panel.
- **Release pipeline** — `.github/workflows/release.yml` builds the tarball, attaches SHA-256, creates the GitHub Release, and calls `/api/admin/releases/notify` so `/changelog` and the update banner pick up new versions automatically.
- **Customer update checker** — `/api/system/update-check` runs once per day from each customer instance with a 24h per-org cache; admins see a banner linking to `/changelog` and a fresh signed download token from `POST /api/license/download` when a newer version ships.

### Apr 2026 — Pre-launch QA highlights

- **Self-hosted licence model** — one-time, 12-month, and 24-month installment plans via Stripe; UNYT token payments on Arbitrum; optional €999/year maintenance renewal. All licences unlock unlimited users; AI features carry no TAKO surcharge — self-hosted instances connect the customer's own Anthropic API key (usage billed by Anthropic at their rates).
- **Public booking page** — now renders host name, avatar, and welcome message from `GET /booking/{user_id}/info`.
- **Profile editing** — `PUT /auth/me` lets users update name, avatar, and timezone from Settings → Profile. IANA timezone picker uses `Intl.supportedValuesOf('timeZone')` with a fallback list and a "use my current timezone" one-click.
- **Password change** — `POST /auth/change-password` with bcrypt verification, 8-char minimum, plus a richer UI: strength meter, show/hide toggles, inline validation for mismatch / too-short / same-as-current, and post-save confirmation.
- **Calendar time picker** — date + time split with 5-minute steps, duration quick-picks (15m/30m/45m/1h/90m/2h), auto-preserved duration when start changes, and a live duration readout.
- **Landing page** — global `ScrollToTop` route listener disables browser scroll restoration and handles hash anchors; `#features`, `#pricing`, `#product` get `scroll-mt-20` so the fixed nav doesn't cover section headers.
- **Admin View badge / Team Summary tab** — hidden for solo admins in Pipeline Reports.
- **Empty states** — Contacts, Leads, Listeners, and Files got richer empty states with explanatory copy and clear CTAs (plus gated upload/form UI).
- **Sidebar + nav** — Deals/Pipeline overlap resolved, sidebar overflow fixed, TAKO logo floating-dot artefact removed, duplicate Sign Out in Settings removed, Settings tabs no longer wrap, Kit.com tab hidden when unconnected.
- **Onboarding** — inline onboarding checklist on Dashboard with one-click copy-to-tasks into an "Onboarding" project.

---

## Pending / Roadmap

### Custom Fields — v3
| Item | Status |
|------|--------|
| Column visibility for custom fields on list views | v2 ships filtering; column toggle is a separate UI surface |
| Required-field enforcement | Backend validation gate + frontend asterisk + submit-disable |
| Number / date / boolean / multi-select field types | v1/v2 ship text + dropdown only |
| Backend-side filtering for very large lists | Client-side AND-filter today; backend `?cf:<key>=value` query support if a partner hits a perf wall |

### TAKO Mail — Phase 2 / 3 / 4
| Item | Status |
|------|--------|
| Multi-folder support (Sent / Junk / Archive / custom) | Phase 1 watches INBOX only; closes the "All Mail vs INBOX" gap vs Apple Mail |
| All-accounts merged inbox view | Today: switch via left rail. Phase 2: interleave |
| Bulk delete (multi-select in centre list) | Phase 2 |
| Trash recovery surface for soft-deleted emails | Soft-delete flag is on every row; needs a `/trash`-style listing + restore |
| "Load more" / pagination on inbox | Cold-start caps at 200 newest UIDs |
| `ai_intent` classification on inbound | Schema field reserved; Sentinel slice in Phase 2 |
| Re-summarise on language change | Existing summaries stay in the language they were generated in |
| Sentinel skip for known-sender auto-replies | OOO from a known contact still pings the bell today |
| Bell-ping fan-out to all admins on shared mailboxes | Currently goes to org owner only |
| Real UserPicker for the Assign-to action on shared mailboxes | `window.prompt` today; UserPicker component exists, just not wired |
| Mark-done UX for shared-mailbox assignees | i18n key reserved; backend endpoint not shipped |
| Activity-feed delete semantics split (inbox vs everywhere) | Today: soft-delete hides from both. Phase 2 if needed |
| Cross-device draft sync | Compose drafts are device-local localStorage today |
| Reply-all + scheduled send | Phase 2 niceties |
| Attachment binaries → TAKO Files bridge | Metadata surfaces today; binary download requires opening in your real client |
| OAuth (Sign in with Google / Microsoft) + push delivery | Phase 3 — Gmail Pub/Sub + Microsoft Graph webhooks replace 30s polling |
| Atlas localisation pass | Phase 4 — full i18n for the email surface, locale-aware date/time |

### Phase B — Smart task prioritisation follow-ups
| Item | Status |
|------|--------|
| Per-task priority badge + override + reason chip on each row | Phase B.2 |
| "Start focus session" walkthrough that opens the top 3 one at a time | Phase B.2 |
| Private Chores integration (toggle currently greyed out) | Schema-side ready; depends on the separate Private Chores project shipping |
| Prompt tuning loop based on operator-flagged ranking surprises | Ongoing |

### Pre-existing
| Item | Status |
|------|--------|
| Encrypt the BYO Anthropic key in `db.org_integrations` | Plaintext today; should use the same `mail.crypto` helper |
| Backend tests in CI | 60+ email-module tests + the existing suites run locally only; deploy-backend.yml smoke-tests `/api/health` only |
| `max_users` per-org seat cap enforcement | Legacy field exists on the model; not enforced |
| Roll out `<SearchableSelect>` to other long-list dropdowns | Add-to-Campaign, etc. |
| Chrome extension (`tako-chrome-extension`) | Not started — backend pairing ready |
| OAuth token encryption at rest | Not done — required before production Meta OAuth |
| Meta app review (Page Public Content Access) | Not submitted — start early, takes weeks |
| Instagram / LinkedIn Listeners | Backend channel stubs ready, poller not wired |
| Settings → Meta OAuth connect button (frontend) | Not done |
| Stripe / Twilio webhook consolidation into generic receiver | Not done |
| Chat Phase B — file sharing + CRM record sharing inside chat | Backend `attachments` field ready; upload pipeline + record picker pending |
| Chat Phase C — PWA push notifications | Service worker + VAPID + permission UX pending |
| Chat Phase D — auto-task detection from chat | LLM pass over `chat_audit` ranges with moderator approval |
| Refresh tokens → httponly cookies | XSS hardening; localStorage today |
| Per-recipient HTML snapshot when personalization lands | Currently every recipient of a campaign gets the same body |
| Per-field changelog on entities | History timeline shows coarse "updated_at" today |
| Companies House (UK) integration | Free API, half-day of work; layer on top of LLM lookup |
| LinkedIn / data-broker enrichment (Apollo, Clearbit, etc.) | Paid; only when needed |
| Sentry activation | DSN registration pending operator action (Stripe webhook now live) |
| DPA legal text for `/api/legal/dpa` | Awaiting counsel review |

---

## License

Proprietary — TAKO by Fintery Ltd. All rights reserved.

Purchasers receive a perpetual licence to use, modify, and deploy TAKO on their own infrastructure. Redistribution or resale of the source code is not permitted.

Canbury Works, Units 6 and 7, Canbury Business Park, Elm Crescent, Kingston upon Thames, Surrey, KT2 6HJ, UK
