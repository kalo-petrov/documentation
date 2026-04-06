# Architecture

## High-Level Overview

Aff-Zero is a Next.js 15 App Router application with Supabase as the backend. It connects to affiliate tracking platforms (Affise, Cake, Everflow, Binom), pulls stats via their APIs, and lets users build automated workflows that send personalized emails with fresh data.

```
┌──────────────────────────────────────────────────────────────┐
│                        Next.js App                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  Dashboard   │  │  Admin Panel │  │  API Routes        │ │
│  │  (dashboard) │  │  (admin)     │  │  /api/cron         │ │
│  │  Stats       │  │  Users       │  │  /api/send-email   │ │
│  │  Automations │  │  Sessions    │  │  /api/execute-auto │ │
│  │  Connections │  │  Logs        │  │  /api/proxy-stats  │ │
│  │  Invoices    │  │  Admins      │  │  /api/admin/*      │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────┘ │
│         │                 │                    │             │
│  ┌──────┴─────────────────┴────────────────────┴───────────┐ │
│  │                     lib/                                │ │
│  │  platforms/    stats/    steps/    automation-executor   │ │
│  │  email-sender  email-encryption  admin/data-masking     │ │
│  └──────────────────────┬──────────────────────────────────┘ │
└─────────────────────────┼────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │      Supabase         │
              │  PostgreSQL + Auth    │
              │  RLS policies         │
              └───────────────────────┘
```

## App Structure

The app uses Next.js route groups to separate concerns:

- `app/(dashboard)/` — Client-facing pages (stats, automations, connections, invoices, email templates, settings, runs-logs, actions)
- `app/(admin)/` — Admin panel pages (dashboard, users, sessions, logs, activity, manage-admins)
- `app/api/` — Server-side API routes
- `app/login/`, `app/signup/` — Authentication pages
- `app/admin-portal-xyz/` — Admin login (hidden route)

## Database Schema

### Core Tables

| Table | Purpose |
|-------|---------|
| `organizations` | Company/team data, owns resources |
| `profiles` | User profiles with roles (client/admin) |
| `connections` | API platform connections (base URL, API key, platform, connection_type) |
| `automations` | Workflow definitions with `workflow` JSONB column |
| `automation_executions` | Execution history (status, timing, context) |
| `automation_execution_logs` | Step-by-step execution logs |
| `api_endpoint_templates` | Platform-specific API endpoint configs |

### Email Tables

| Table | Purpose |
|-------|---------|
| `email_service_config` | Provider config (Resend/SendGrid per user) |
| `email_templates` | Reusable email templates |
| `email_send_history` | All sent emails with delivery status |

### Admin Tables

| Table | Purpose |
|-------|---------|
| `admin_users` | Separate admin accounts (bcrypt hashed) |
| `user_sessions` | Login/logout tracking with IP and user agent |
| `activity_logs` | User CRUD audit trail (before/after states) |
| `admin_activity_logs` | Admin panel action audit trail |
| `subscription_plans` | Plan limits (connections, automations, integrations) |

### User Settings

| Table | Purpose |
|-------|---------|
| `user_settings` | JSONB blob for Gmail SMTP accounts, Google Drive tokens, ChatGPT config |

## Platform System

The platform system is designed to be platform-agnostic. Each platform is a configuration object that defines stat types, URL building, response transformation, and parameter schemas.

### Key Files

- `lib/platforms/index.ts` — Platform registry
- `lib/platforms/platform-manager.ts` — Centralized service for config retrieval, URL building, response transformation
- `lib/platforms/cake.ts` — Cake platform config
- (Affise config is the default, loaded from `api_endpoint_templates`)

### Platform Configuration Shape

Each platform defines:

- **Stat Types**: e.g., `by_affiliate`, `by_offer`, `by_date`
- **URL Builder**: Constructs API request URLs with platform-specific parameter formats
- **Response Transformer**: Normalizes API responses into a standard table format
- **Parameter Config**: Defines available filters and their types

### Adding a New Platform

1. Create `lib/platforms/{platform}.ts` with the platform config
2. Register it in `lib/platforms/index.ts`
3. Add to `platform-manager.ts`
4. Insert `api_endpoint_templates` rows for its stat types
5. The UI auto-discovers stat types and renders parameter forms

### Current Platforms

| Platform | Status | Connection Types | Stat Types |
|----------|--------|-----------------|------------|
| Affise | Implemented | Admin Portal, Affiliate Portal | By Affiliate, By Advertiser, By Date, By Offer (affiliate) |
| Cake | Implemented | Affiliate API | Daily Summary, Campaign Summary |
| Everflow | Config only | — | By Affiliate, By Campaign, By Date |
| Binom | Config only | — | By Traffic Source, By Campaign, By Date |

## Automation Engine

### Visual Builder

The automation builder (`components/automations/visual-automation-builder.tsx`) uses a horizontal canvas layout with step cards (240x400px). Steps flow left-to-right with data-passing indicators between them.

Step types are color-coded:
- Blue: Pull Stats
- Green: Send Email
- Purple: Invoice
- Orange: Filter

The workflow is stored as a JSONB array in `automations.workflow`.

### Step Types

**Pull Stats** (`components/automations/enhanced-pull-stats-step.tsx`):
- Selects a connection, stat type, date preset, and timezone
- User performs a test pull and clicks cells to select data
- Selected cells are stored as templates (position + entity ID), not values
- Produces variables like `{affiliate_name_1}`, `{payout_2}`

**Send Email** (`components/automations/send-email-step.tsx`):
- Consumes variables from all previous Pull Stats steps
- Supports multiple recipients
- Variables are inserted into subject and body
- Can select which email account (Gmail/Resend/SendGrid) to send from

### Execution Lifecycle

```
Trigger (cron or "Run Now")
  │
  ├─ Load automation + workflow from DB
  │
  ├─ For each step:
  │   ├─ Pull Stats:
  │   │   ├─ Calculate fresh dates from preset + timezone
  │   │   ├─ Build API URL via platform manager
  │   │   ├─ Call platform API directly (server-side)
  │   │   ├─ Match entities by immutable ID (affiliate_id, etc.)
  │   │   ├─ Extract fresh metric values from matched rows
  │   │   └─ Store variables in execution context
  │   │
  │   └─ Send Email:
  │       ├─ Load email config (Gmail/Resend/SendGrid)
  │       ├─ Substitute variables in subject + body
  │       ├─ Send via appropriate provider
  │       └─ Log to email_send_history
  │
  ├─ Write execution record to automation_executions
  └─ Write step logs to automation_execution_logs
```

Key file: `lib/automation-executor.ts`

## Variable System

Variables flow globally across all steps in an automation. Each step can consume variables from ANY previous step.

### Variable Lifecycle

**Configuration time** (test pull):
1. User pulls stats and clicks cells to select
2. Each cell stores: `entityId` (immutable), `entityName`, `entityType`, `column`, `value` (test), `originalRowIndex`
3. Saved to `step.config.selected_data_fields[]`

**Execution time** (real run):
1. Fresh dates calculated from preset + timezone
2. Fresh stats pulled from platform API (same call as test, different dates)
3. Entities matched by immutable ID (e.g., `affiliate_id = 123`)
4. Fresh metric values extracted from matched rows
5. If entity not found, falls back to `originalRowIndex`, then test value

### Variable Naming

Variables are named `{column_name_row_index}` where `row_index` is 1-based:
- `{affiliate_name_1}`, `{payout_1}` — first selected entity
- `{affiliate_name_2}`, `{revenue_2}` — second selected entity

### Entity Targeting

The system stores immutable entity IDs (not row positions) to prevent sending data to the wrong recipient when table order changes between runs. Supported identifiers:
- Affiliate ID, Campaign ID, Offer ID, Advertiser ID, Traffic Source ID

### Multi-Step Variable Collection

```typescript
// visual-automation-builder.tsx collects from ALL previous Pull Stats steps
const pullStatsSteps = previousSteps.filter(s => s.type === 'pull_stats');
const allSelectedFields = [];
pullStatsSteps.forEach(step => {
  if (step.config.selected_data_fields) {
    allSelectedFields.push(...step.config.selected_data_fields);
  }
});
```

## Email System

### Architecture

Three providers supported, each with a different sending mechanism:

| Provider | Mechanism | Library | Best For |
|----------|-----------|---------|----------|
| Gmail SMTP | SMTP via nodemailer | `nodemailer` | Small users, testing |
| Resend | HTTP API | `fetch` | Production, professional |
| SendGrid | HTTP API | `fetch` | Enterprise, high volume |

Key files:
- `lib/email-sender.ts` — Unified send interface for all providers
- `lib/email-encryption.ts` — AES-256-GCM encryption for Gmail passwords (PBKDF2 key derivation, per-encryption salt)

### Multi-Account Gmail

Users can configure up to 3 Gmail SMTP accounts (plan-dependent), stored in `user_settings.settings.gmail_smtp_accounts[]`. The Send Email step shows a grouped dropdown to pick which account to send from. Account IDs prefixed with `gmail_` distinguish them from `email_service_config` table entries.

### Email Config Resolution

1. Check `email_config_id` from step config
2. If starts with `gmail_` → load from `user_settings.gmail_smtp_accounts`
3. If regular UUID → load from `email_service_config` table
4. If no ID → use active/primary config (backward compatibility)

## Stats System

The Stats page (`components/shared/stats-query-interface.tsx`) is a shared component used in two modes:
- **View mode**: Stats page — pull and display stats, download CSV
- **Select mode**: Pull Stats automation step — pull stats and click cells to select variables

### Shared Utilities

- `lib/stats/date-presets.ts` — Timezone-aware date calculations using `Intl.DateTimeFormat().formatToParts()`. Fixes timezone bugs (e.g., "yesterday" showing wrong day in non-UTC zones).
- `lib/stats/url-builder.ts` — Platform-specific URL construction with API key masking for display.
- `lib/stats/entity-extractor.ts` — Entity ID extraction, entity lookup in API responses, variable name generation.

## Admin Panel

Completely separate authentication from the main app. Uses custom JWT (jose) with bcrypt password hashing.

### Pages

| Page | Route | Purpose |
|------|-------|---------|
| Dashboard | `/admin-dashboard` | System metrics, active users, recent activity |
| Users | `/admin-users` | User list with usage gauges vs plan limits |
| User Activity | `/admin-user-activity` | CRUD operation audit logs |
| Admin Activity | `/admin-admin-activity` | Admin action audit logs |
| Sessions | `/admin-sessions` | Login/logout tracking, active sessions |
| System Logs | `/admin-logs` | Automation runs + audit trail (tabbed) |
| Manage Admins | `/admin-manage-admins` | Create/delete admin users |

### Security

- Hidden login route (`/admin-portal-xyz`, configurable via env var)
- Separate JWT secret and cookie (`admin-session`)
- Middleware protection on all admin routes
- All admin actions logged to `admin_activity_logs`
- Data masking in admin views: API keys show `[REDACTED]`, variable values show `***`

### Data Masking

`lib/admin/data-masking.ts` masks sensitive data before sending to admin UI:
- `maskConnectionData()` — API keys and tokens
- `maskExecutionContext()` — Variable values (names visible for debugging)
- `maskAutomationWorkflow()` — Credentials in workflow configs
- `maskUserSettings()` — OAuth tokens

## Scheduler

### Vercel Cron (production)

Configured in `vercel.json` to hit `/api/cron/check-automations` every hour. The endpoint:
1. Queries enabled automations where `next_run_at <= now`
2. Executes each via `automation-executor.ts`
3. Updates `next_run_at` for the next scheduled run

Protected by `CRON_SECRET` env var in production.

### node-cron (local development)

`scripts/run-local-scheduler.js` runs a local cron job checking every minute. Start with `npm run scheduler`.

## Debugging

### Terminal logs

The executor logs extensively with prefixes: `[Executor]`, `[PullStats]`, `[SendEmail]`. Key things to watch:
- Connection and template loading
- API request URLs (keys masked)
- Response status and row counts
- Variable extraction results
- Email delivery confirmation

### Database logs

```sql
-- Recent executions
SELECT ae.id, a.name, ae.status, ae.started_at, ae.error_message
FROM automation_executions ae
JOIN automations a ON ae.automation_id = a.id
ORDER BY ae.created_at DESC LIMIT 10;

-- Step-by-step logs for a run
SELECT level, message, data
FROM automation_execution_logs
WHERE execution_id = 'YOUR_EXECUTION_ID'
ORDER BY created_at;
```

### Common issues

- **404 on Run Now**: The executor makes direct API calls (not through `/api/proxy-stats`). Verify the connection's base URL is correct.
- **Variables mixed up**: Check that `entityIds` are stored with selected cells. The executor matches by entity ID, not row position.
- **Empty stats**: Verify the date range has data and the connection's API key is valid by testing manually on the Stats page first.
- **Email not sent**: Check `email_service_config` table has an active row. For Gmail, verify the encryption key is set in `.env.local`.
