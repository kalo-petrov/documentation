# Backlog

Single source of truth for all pending and planned work.

## Platform Integrations

### Everflow Integration
- Status: Config defined in `lib/platforms/`, not yet connected to real API
- Stat types planned: By Affiliate, By Campaign, By Date
- URL format: `{base_url}/api/affiliate/stats?api_key={key}&...`
- Needs: Platform-specific parameter forms, response transformer testing

### Binom Integration
- Status: Config defined in `lib/platforms/`, not yet connected to real API
- Stat types planned: By Traffic Source, By Campaign, By Date
- URL format: `{base_url}/api/stats/traffic?token={key}&...`
- Needs: Same as Everflow

### Cake Admin API
- Currently only Affiliate API is implemented
- Admin API endpoints needed for platform clients (network-level stats)

### Platform Detection (Phase 2)
- Add `platform` field to connections for dynamic stat type loading
- Platform-specific parameter forms auto-rendered from config
- Platform-specific UI adaptations

## Google Docs Integration

- **Goal**: Let users view/edit invoice templates in Google Docs, saved to their Drive
- **Approach**: OAuth 2.0 with user's own Google account (not service account)
- **Permissions needed**: `documents` + `drive.file` scopes
- **Package**: `googleapis` (already installed)
- **OAuth flow partially exists** (Google Drive connection is implemented, can extend)
- **Steps remaining**:
  1. Enable Google Docs API in Cloud project
  2. Extend OAuth scopes for Docs
  3. Implement document creation from uploaded PDF/DOCX
  4. Add Docs viewer/editor integration in invoice templates UI

## New Automation Step Types

- **Filter step**: Conditional logic (if revenue > X, continue)
- **Transform step**: Data manipulation before email
- **Invoice step**: Generate and send invoices
- **Conditional branching**: If/else logic in workflows
- ~~**Google Sheets step**: Write data to user's spreadsheet~~ ✅ Implemented with three write modes: Replace (in-place placeholder replacement), Append (template row preservation + below-data strategy), and Match & Fill (row matching by criteria + targeted cell writes). See `docs/automations/steps/write-to-google-sheets.mdx`.

## Email Enhancements

- HTML template editor (visual email builder)
- Attachment support (PDF reports, CSV exports)
- Email open/click tracking
- A/B testing for email content
- Dynamic recipients (use affiliate emails from stats data)
- Batch sending optimization for high volume
- Email scheduling (delay sending by hours/days)

## Automation Engine

- Retry logic for failed steps (configurable max retries + backoff)
- Execution history viewer in the UI (currently SQL-only)
- Monitoring dashboard with charts
- Migrate to Inngest for production scheduler (better reliability, monitoring)
- Drag-and-drop step reordering in visual builder
- Step duplication within a workflow
- Aggregate variable selectors (SUM, AVG, MAX, MIN across rows)
- Position-based selectors (top performer by revenue)
- Filter-based selectors (all affiliates with revenue > $1000)

## Admin Panel

- Export activity logs to CSV/JSON
- Real-time activity monitoring (WebSocket)
- Email notifications for critical events (failed automations, suspicious logins)
- Two-factor authentication for admin accounts
- IP whitelisting for admin access
- Admin roles and permissions (read-only vs full access)
- User impersonation for support
- Automated stale session cleanup (cron for `cleanupStaleSessions()`)
- Log retention policy (archive logs older than 90 days)

## Stats System

- Data visualization (charts and graphs alongside tables)
- Advanced table sorting and filtering
- More stat types per platform (sub-affiliate, creative, etc.)
- Cake XML response support (currently JSON only)

## Security & Infrastructure

- Database-level encryption for sensitive fields (currently UI/API-level masking only)
- Rate limiting on API endpoints
- IP-based rate limiting on admin login
- HTTPS enforcement in production (admin cookies already secure-flagged)
- Proper secret rotation procedures

## Technical Debt

- `MOCK_USER_ID` in `lib/mock-user.ts` — replace with real Supabase Auth when re-enabling auth
- Auth was simplified/removed for development (see SIMPLIFICATION_NOTES context); re-add auth context, middleware, and user profile when going to production
- Some migration files reference old table names (`automation_runs` vs `automation_executions`)
- Consider moving from hardcoded Supabase config (`lib/config.ts`) to env-var-only approach
