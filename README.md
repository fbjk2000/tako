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
- **Projects** — Group tasks under deals, progress tracking, clickable task detail from project context, auto-created chat channels, team members
- **Companies** — Target company management with industry, size, and contact tracking. **AI-native typeahead** suggests existing org records + Claude lookups when adding a new company; **AI Enrich** fills missing fields (industry, size, location, founded year, HQ, competitors, key products) on existing records. Single-record and bulk-up-to-20 enrich.
- **Custom Fields** — Every org can extend the core schema with their own taxonomy via Settings → Custom fields (org admins; gated to `owner` / `admin` / `super_admin` / `deputy_admin`). Super-admins additionally see the same panel at Admin → Custom Fields. Text and dropdown field types, per `(org, entity_type)` namespace, applies to contacts / leads / companies / deals. Values live in a `custom_fields` dict on each entity row; values stay on rows when a definition is deleted (recreating the same key restores them). Filterable on every list page (Any / value / (Empty) for dropdowns, substring contains for text; multiple filters AND together).
- **Calendar** — Month/Week views with scheduled calls, task due dates, deal closes, custom events with entity linking, Google Calendar sync. **Per-event reminder lead-time** (None · 5 / 10 / 15 / 30 min · 1 hr · 1 day) — 60s cron fires an in-app `appointment_due` notification at the configured offset with a two-tone WebAudio chime + sonner toast (chime mutable per-user under Settings → Profile). Covers `calendar_events` + `scheduled_calls` + booking-confirm rows. **Recurring events** — Daily / Weekly / Monthly / Yearly + interval + end condition (Never / After N / On date) on the Create + Edit dialogs; instances expand at read time, edits to any occurrence update the series anchor. Clicking a task pill on the calendar deep-links into the full task edit dialog. Tasks/deals with date-only due_dates now render correctly in the all-day strip instead of piling up at the 07:00 row.
- **Campaigns** — Multi-channel (Email via Resend + Kit.com, Facebook, Instagram, LinkedIn), rich email editor (subject + Markdown body + live HTML preview + AI draft), AI-powered drafting (purpose + tone), channel picker on create, social campaigns linked to Listeners. **Per-recipient delivery tracking** with status pills (delivered, bounced, recipient_unknown, inbox_full, deferred, failed, replied, opted_out, pending) driven by Resend webhooks; aggregate counts denormalised on the campaign for cheap list rendering. **Send-batch semantics**: a "Send" only mails recipients whose row status is `pending` — already-delivered/bounced/replied rows are skipped, so adding new people to a sent campaign and pressing Send again only mails the new ones (no re-spam path). **Pending vs Deferred are separate aggregates**: the UI exposes a "Retry N deferred" button (orange band) distinct from "Send to N pending" so operator intent stays explicit. **Provider-safe pacing** — `RESEND_SEND_DELAY_S` (default 250ms ≈ 4 sends/sec) keeps batches under Resend's request-per-second limit. **Add to campaign from Lead detail** — single-lead button on the Lead detail dialog plus the existing bulk-list flow. **Closed-loop replies** — inbound replies are received via Resend Inbound, matched back to the recipient row OR to the underlying lead (via from-address) for transactional sends that bypassed campaigns, and surfaced inline with the full body (fetched from Resend's `/emails/receiving/{id}` API; HTML-only replies render sanitized via DOMPurify when no plain-text part is present). **Self-sourced loop guard** drops inbounds from our own sender domain (bounces, auto-replies, forwarded copies) so they don't overwrite `last_reply_from`. **BCC outbound capture** logs operator replies sent from their own mail client when they BCC `tako@tako.software`, with sender identification via user email or `email_aliases`. **AI suggestions on replies** classify intent (interested / objection / not_now / out_of_office / unsubscribe / question / other), draft a contextual response, and propose follow-up tasks; one-click **Send reply** (with `In-Reply-To` threading) and **Create task** (auto-linked to the lead/deal). **Soft delete + restore** — deleted campaigns + per-recipient history move to /trash; restoring brings the entire send history back intact. **Filter recipients by status** in the detail view. Sent campaigns can be renamed but content stays frozen post-send so history isn't rewritten.
- **Per-entity History** — Every Lead / Contact / Deal / Company detail dialog has a unified activity timeline aggregating emails sent (with full body + delivery status + provider error), tasks, deals, conversion events, chat activity, and creation/update anchors. Each row drills down to the granular trace; email rows link directly to the relevant Campaign detail.
- **Listeners** — AI social listening agents: keyword monitoring, Claude-powered hit classification (buying signal / complaint / question / mention / noise), confidence scoring, sentiment, suggested replies, auto task creation, digest reports, Meta Graph poller, Chrome extension device-code pairing
- **Files** — File upload with AI summarisation (PDF, DOCX, text, images), linked to any entity, one-click task creation from AI suggestions

### AI Features (Claude)
- **Lead Scoring** — AI assigns a 1–100 quality score based on profile completeness and signals
- **Lead Enrichment** — Fills in company info, tech stack, interests, recommended sales approach
- **Company Suggest + Enrich** — Typeahead while creating a company merges hits from your own data (existing companies + names referenced on leads/contacts) with Claude-powered "what is this company" lookups. Operator-confirmation required (AI suggestions tagged "Verify"). Existing company records get a one-click **AI Enrich** that fills empty fields without overwriting operator-set ones; bulk variant up to 20 at a time.
- **Email Drafting** — Personalised sales emails with tone/purpose selection, embedded in the campaign editor with live HTML preview
- **Reply Triage** — Inbound replies are auto-classified by Claude (intent + summary), get a drafted contextual response, and propose 0–3 follow-up tasks. Visible inline in the campaign detail expand row with one-click Send / Create Task.
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
- **Outbound Calling** — Twilio integration for direct calls from the CRM
- **Inbound Calls** — Auto-greeting, voicemail recording, caller identification
- **Call Scheduling** — Calendar-based scheduling with configurable reminders
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
| AI | Anthropic Claude (`claude-sonnet-4-20250514`) via `anthropic` Python SDK |
| Email | Resend (primary, configurable `SENDER_EMAIL` — founder-led `florian@tako.software` is the production default; `REPLY_TO_EMAIL` routes replies to a shared mailbox), Kit.com (optional subscriber lists). Per-message pacing via `RESEND_SEND_DELAY_S` (default 250ms ≈ 4 sends/sec) keeps batches under Resend's request-per-second cap. |
| Payments | Stripe + UNYT Token (Arbitrum) |
| Calling | Twilio Voice API |
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

`scripts/build-distribution.sh` produces `dist/tako-crm-<version>.tar.gz` plus an accompanying `.sha256` and `VERSION` file. What it ships:

- Full backend and frontend source, minus stripped regions
- `docker-compose.yml` and Dockerfiles
- Customer-facing `backend/.env.example` (no platform keys, no Stripe price IDs, no `DISTRIBUTION_DIR`)
- `scripts/backup-mongo.sh`
- `scripts/distribution-README.md` (install guide)
- `docs/VPS-HARDENING.md`

What it strips:

- **UNYT / crypto payment UI and routes** — customers see Stripe only
- **Self-serve demo** — `/demo` endpoints, demo seeder, middleware soft-lock, admin demo management
- **Platform-only pages** — `/changelog`, `/download`, partner onboarding admin UI
- **Super-admin surfaces** — analytics, data explorer, release notify endpoint, partner payout admin
- **Distribution tooling itself** — `scripts/build-distribution.sh`, release workflow, `DISTRIBUTION_DIR` handling
- **Internal configs** — `.emergent/`, platform-only migrations, internal test fixtures

Stripping is driven by sentinel-wrapped regions using the generalized markers `DEMO_BEGIN`/`DEMO_END` and `PLATFORM_BEGIN`/`PLATFORM_END` (matched as `(?P<kind>DEMO|PLATFORM)_BEGIN` … `(?P=kind)_END`). Touch them carefully — the regex is paired.

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

### Leads

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/leads` | List leads. Params: `status`, `source`, `tags` (csv, ANY-match), `q` (search across name/email/company/title), `assigned_to`, `has_email`, `campaign_id`, `not_in_campaign_id` |
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

- **Self-hosted licence model** — one-time, 12-month, and 24-month installment plans via Stripe; UNYT token payments on Arbitrum; optional €999/year maintenance renewal. All licences unlock unlimited users and unlimited AI.
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
