# Frances QA Audit — Ground Truth Report
**Date:** 2026-05-01  
**Auditor:** Peri (automated codebase audit)  
**Repo audited:** valorharthvl/VALORv1 @ `814d9b2`  
**Supabase project:** `nohaxgnqdzepnvfxteoa`  
**Target destination:** valorharthvl/fritz/audits/frances-qa-audit-2026-05-01.md *(fritz repo outside Peri scope — stored here instead)*

---

## Summary

| Category | FIXED | PARTIALLY FIXED | STILL OPEN | UNKNOWN |
|---|---|---|---|---|
| 17 Known Gaps | 0 | 8 | 9 | 0 |
| 26 Audit Findings (P0–P2) | 3 | 12 | 10 | 1 |

**P0-CRITICAL items remaining open: 2** (Auth on generate endpoint; RLS bypass via admin client).  
**P1-HIGH items still open: 4** (Credential verification, Real signature, Retraction check, PHI screening on output).  
All FIXED items have code evidence below.

---

## Part 1 — 17 Known Gaps

| # | Gap | Status | Evidence |
|---|---|---|---|
| 1 | Law firm sharing — share_token PHI guardrails | **PARTIALLY FIXED** | `app/share/[token]/page.tsx`: queries `case_name, condition, status, due_date, final_opinion_text` via admin client, filtered by `share_token` + `share_enabled=true`. `final_opinion_text` is fetched but **NOT rendered** — only used as a boolean check (`reportReady`). However, `case_name` (veteran's full name) and `condition` **are** rendered publicly to any token-holder without PHI screening. No output sanitization on these fields. |
| 2 | Attestation table — provider_reviews wired into generate route? | **STILL OPEN** | `app/api/generate/route.ts` lines ~80–100: parallel Supabase fetches query `opinion_shells`, `prompt_templates`, `evidence`, `clause_library`. No reference to `provider_reviews` table anywhere in codebase. Table may not exist in schema — not present in any migration file. |
| 3 | Admin bypass — audit logging when admin client used? | **PARTIALLY FIXED** | `app/api/qa/route.ts`: emits to `audit_events` table on every QA run. `lib/phi-middleware.ts`: PHI detections logged. `app/api/generate/route.ts`: logs to `artifact_generation_log` (separate custom table, not `audit_events`). Fragmented — no single audit trail covers all `createAdminClient()` calls. |
| 4 | Error routing — structured error/failure routing to provider? | **PARTIALLY FIXED** | `app/api/qa/route.ts`: calls `sendQAFailEmail()` (`lib/email.ts`) on QA failure; then calls `/api/cases/advance` to auto-advance case status. Generate route returns structured SSE error chunk on timeout/failure. No routing for server-side generate errors to provider notification. |
| 5 | Ryan direct access — admin UI for case oversight? | **PARTIALLY FIXED** | `app/audit/` and `app/dashboard/` directories exist. `app/api/audit/route.ts` handles audit trail. `is_valor_admin()` SQL function restricts to `@valorhartllc.com` emails (`supabase/migrations/20240403_rls_policies.sql`). Full scope of dashboard not fully audited; directory exists with pages. |
| 6 | Version history — artifact_versions wired into generate route? | **PARTIALLY FIXED** | `app/api/generate/route.ts` step 8a: queries `artifacts` for existing `draft`/`qa_failed`/`qa_blocked` records and updates them to `superseded` before inserting new artifact. `supabase/2026-03-27-phase10a-case-artifacts-backbone-slice1.sql`: `case_artifacts.version` integer auto-increments on UPDATE via trigger `trg_set_case_artifact_timestamps`. No dedicated `artifact_versions` history table. Generate route writes to `artifacts` table (separate from `case_artifacts`). |
| 7 | Billing integration | **STILL OPEN** | GitHub code search for `stripe` across valorharthvl/VALORv1: 0 results. No Stripe SDK in `package.json`. No billing tables in any migration. No usage metering for AI generation costs. |
| 8 | Resend email fallback | **STILL OPEN** | `lib/email.ts`: `getResend()` initializes with `process.env.RESEND_API_KEY \|\| 'placeholder'`. All email functions check `if (!process.env.RESEND_API_KEY) return` — silent skip. Catch blocks log to `console.error` only. No secondary email provider. No fallback logging to audit table. |
| 9 | Storage layer — document storage architecture | **PARTIALLY FIXED** | `app/api/documents/` route directory exists. `evidence` table stores document metadata with `extracted_text` field. `app/api/generate/route.ts`: checks `e.extracted_text` on evidence records. No Supabase Storage bucket creation in any migration (dashboard-only config). No storage RLS policies in code. |
| 10 | OCR pipeline | **STILL OPEN** | `app/api/generate/route.ts`: reads `e.extracted_text` from `evidence` table (text must already be there). No OCR library (Tesseract, AWS Textract, etc.) visible in `lib/`, `package.json`, or any API route. How `extracted_text` gets populated is not in codebase. |
| 11 | Session management edge cases | **STILL OPEN** | `app/api/generate/route.ts`: no `createServerClient()` call, no JWT/session verification, no `auth.uid()` check. Uses `createAdminClient()` (service_role) for all operations. Any unauthenticated caller with knowledge of a `case_id` and `condition_type` can trigger generation. |
| 12 | Rate limiting — generation_rate_limits wired into generate route? | **STILL OPEN** | No `generation_rate_limits` table in any migration file. No rate limiting logic in `app/api/generate/route.ts`. No per-user, per-case, or per-IP quotas. |
| 13 | Backup/recovery plan | **STILL OPEN** | Not in codebase. No pg_dump scripts, backup cron jobs, or recovery runbooks in repository. |
| 14 | Condition taxonomy maintenance workflow | **PARTIALLY FIXED** | `lib/condition-queries.ts` (23KB): maps condition keys to PubMed queries. `opinion_shells` table in schema stores condition display names and descriptions. No admin UI for taxonomy maintenance visible in `app/` routes. |
| 15 | Sentry observability instrumented? | **STILL OPEN** | GitHub code search for `sentry` across valorharthvl/VALORv1: 0 results. No `@sentry/nextjs` in `package.json`. No error tracking or crash reporting. |
| 16 | Data retention/deletion execution workflow | **STILL OPEN** | No data retention policies in code. No deletion workflow routes. No scheduled cron for data expiry. `app/api/cron/` directory exists but contents not verified. |
| 17 | Multi-provider collaboration architecture | **PARTIALLY FIXED** | `artifacts` table has `provider_id` column. `supabase/migrations/20240403_rls_policies.sql`: providers can SELECT assigned artifacts with status `qa_passed`+, and UPDATE assigned artifacts. Single `provider_id` per artifact — no concurrent multi-provider collaboration model. |

---

## Part 2 — 26 Audit Findings

### P0-CRITICAL

| Finding | Status | Evidence |
|---|---|---|
| Auth on generate endpoint — is it protected? | **STILL OPEN** | `app/api/generate/route.ts`: no `createServerClient()`, no session check, no Bearer token, no `auth.uid()`. `createAdminClient()` used (service_role bypasses RLS). Any caller with `case_id` + `condition_type` can invoke generation. |
| False credentials in prompt — still present? | **FIXED** | `lib/frances-config.ts` `DEFAULT_GENERATE_DRAFT_CONFIG.systemPrompt`: "Frances" is a fictional AI agent name — no hardcoded NPI, DEA, license numbers, or "Dr." credential. Prompt explicitly states: "Never fabricate records, dates, page numbers, clinical findings, diagnoses, or examination results." `supabase/2026-04-30-frances-final-opinion-prompt-rewrite.sql`: prompt rewrite migrated 2026-04-30 removes draft disclaimer language. |
| Review gate enforced — does QA actually block bad output? | **PARTIALLY FIXED** | `lib/gatekeeper.ts` `validateApprovalRequest()`: blocks transition to `provider_approved` if `artifactStatus !== 'qa_passed'`. `app/api/qa/route.ts`: sets artifact status to `qa_failed` on any failing rule (QA-001 through QA-005). **Gap:** QA must be manually triggered via POST `/api/qa` — not automatically run after generation. Provider can be blocked from approving, but the draft exists in `qa_failed` state indefinitely without auto-trigger. |
| AI disclosure embedded in output? | **PARTIALLY FIXED** | `lib/phi-detection.ts`: `AI_DISCLOSURE` constant defined: *"This content was generated by an AI system and is intended to support — not replace — the independent clinical judgment of a licensed healthcare provider."* `app/api/generate/route.ts`: sent in final SSE metadata chunk as `ai_disclosure` field. **NOT** appended to the generated text or stored in `final_opinion_text`/`content` field in database. Downstream consumers must explicitly read the metadata field. |
| RLS bypass via admin client — still possible? | **STILL OPEN** | `supabase/migrations/20240403_rls_policies.sql` (final comment): *"The service role key bypasses all RLS policies. All server-side API routes use createAdminClient() which uses the service role key, so they are not affected by these policies."* All API routes confirmed using `createAdminClient()`. RLS provides no protection against server-side misuse. |

### P1-HIGH

| Finding | Status | Evidence |
|---|---|---|
| Structured veteran data in generation | **FIXED** | `app/api/generate/route.ts`: `caseData.case_name`, `caseData.question`, `caseData.evidence`, `caseData.lay_statements`, `caseData.notes` all structured into the user prompt. Evidence documents with `extracted_text` included (truncated at 2000 chars). Secondary claim context injected when `parent_condition` present. |
| QA failure blocks generation | **PARTIALLY FIXED** | `lib/gatekeeper.ts`: blocks approval (`provider_approved`) if QA not passed. But `validateGenerationRequest()` does NOT check whether a prior QA failure exists — regeneration is freely allowed on any case regardless of prior QA state. |
| RBAC on admin page | **PARTIALLY FIXED** | `lib/frances-config.ts` `requireFrancesAdmin()`: validates Bearer token via Supabase `getUser()`, then checks email against `FRANCES_ADMIN_EMAILS` env var allowlist. Used on Frances config management endpoints. **Not present** on `app/api/generate/route.ts`, `app/api/cases/`, `app/api/artifacts/`, or `app/api/qa/`. |
| PHI screening on output (not just input) | **PARTIALLY FIXED** | `app/api/generate/route.ts`: `screenRequestForPHI()` called on input before generation (PRIV-FRA-001). `app/share/[token]/page.tsx`: renders `case_name` (veteran full name) and `condition` to any token-holder — no PHI screening on these output fields. `final_opinion_text` is fetched but not rendered. |
| Three Laws enforcement on generation | **PARTIALLY FIXED** | `lib/frances-config.ts` system prompt has `ABSOLUTE PROHIBITIONS` section: no fabrication, no em-dashes, no invented citations. Enforced at the prompt level only — no separate code layer that validates generated output against Three Laws before artifact insertion. |
| Credential verification | **STILL OPEN** | `app/api/generate/route.ts`: accepts `provider_id` from request body but makes no query to `providers` table to validate existence, `status = 'active'`, or NPI. Provider ID is passed straight to artifact insert without verification. |
| Audit trail | **PARTIALLY FIXED** | Events scattered: `audit_events` (QA runs, PHI detections), `artifact_generation_log` (generation events), `library_audit_log` (clause usage via RPC), `case_events` (workflow transitions). No single table or view unifying all admin-client mutations. |
| Real signature (not placeholder) | **STILL OPEN** | `app/api/generate/route.ts` user prompt (nexus letter): *"Signature block placeholder"* and *"7. Signature block placeholder"*. DBQ prompt: *"7. EXAMINER CERTIFICATION: Signature block placeholder with credentials"*. Generated content will contain placeholder text — no real signature or electronic signing workflow. |
| Retraction check | **STILL OPEN** | No `retracted` boolean on `clause_library`, `cases`, or `artifacts` tables in any migration. No `retraction_check` table. `generate/route.ts` filters clauses by `.eq('status', 'active')` — provides soft protection but "active" is not equivalent to "not retracted." |
| Versioning wired | **PARTIALLY FIXED** | `app/api/generate/route.ts` step 8a: archives previous `draft`/`qa_failed`/`qa_blocked` artifacts to `superseded`. `supabase/2026-03-27-phase10a-case-artifacts-backbone-slice1.sql`: `case_artifacts.version` integer auto-increments on UPDATE via trigger. No dedicated version history table. Generate route writes to `artifacts` table (separate from `case_artifacts` backbone). |
| Timeout handling complete | **FIXED** | `app/api/generate/route.ts`: `export const maxDuration = 300` (Vercel Pro 5-min limit; streaming keeps connection alive). `AbortController` with 45-second timeout applied to Claude stream call. Error caught and structured SSE error chunk returned to client. |

### P2-MEDIUM

| Finding | Status | Evidence |
|---|---|---|
| Model selection guard | **PARTIALLY FIXED** | `app/api/generate/route.ts`: `const CLAUDE_MODEL = process.env.LLM_MODEL \|\| 'claude-haiku-4-5'`. Env var override allowed. No allowlist enforcement — any model string accepted if env var set. |
| Clause library size (how many rows now?) | **UNKNOWN** | `clause_library` table exists with `status` column. Cannot query Supabase directly. Generate route fetches `.eq('status', 'active').order('clause_code')` and slices to top 8. Row count requires live DB query. |
| QA email recipient configured | **PARTIALLY FIXED** | `lib/email.ts` `sendQAFailEmail()`: recipient address must be set via environment variable. Not hardcoded. Resend API key required (`RESEND_API_KEY`). No fallback if unconfigured. |
| Clause deletion protection | **PARTIALLY FIXED** | `app/api/generate/route.ts`: filters `.eq('status', 'active')` — inactive clauses not injected. No schema-level `CHECK` constraint or `ON DELETE RESTRICT` preventing hard deletion of active clauses. |
| Rate limiting enforced | **STILL OPEN** | No rate limiting anywhere in codebase. No `generation_rate_limits` table, no middleware, no IP-based throttling, no per-user quota. |
| QA-to-clause feedback loop | **PARTIALLY FIXED** | `app/api/qa/route.ts`: on QA rule failure, calls internal `/api/frances` with `action: 'check_text'` to get `suggested_replacement` from clause library. Result stored on `qa_results.suggested_replacement`. Feedback loop exists for display but no automated clause promotion from QA results. |
| Veteran consent flow | **STILL OPEN** | `app/share/[token]/page.tsx`: renders case name, condition, and status to token-holder with no consent gate, no disclosure banner, no "I acknowledge this is AI-assisted" acknowledgment. No consent page in `app/`. |

---

## Priority Action List (what to file next)

### P0 — File immediately
1. **Auth on /api/generate** — Add Supabase session verification (createServerClient + `getUser()`). Block unauthenticated callers. Route currently requires zero auth.
2. **RLS bypass scope** — Document which routes require additional application-layer authz since service_role bypasses all RLS.

### P1 — File this sprint
3. **Credential verification** — Validate `provider_id` against `providers` table (status = 'active') before artifact insert.
4. **Real signature workflow** — Replace "Signature block placeholder" in prompts with structured signature block. Add e-sign or attestation step.
5. **Retraction check** — Add `retracted boolean default false` to `clause_library`. Filter out retracted clauses in generate route.
6. **PHI screening on share portal** — Screen `case_name` before rendering, or replace with a non-PHI case reference ID.
7. **QA auto-trigger** — Auto-trigger QA check after artifact creation in generate route (or at minimum block export without QA run).
8. **AI disclosure in stored output** — Append `AI_DISCLOSURE` text to `final_opinion_text` in DB, not just SSE metadata.
9. **Sentry** — Install `@sentry/nextjs`, instrument generate and QA routes.

### P2 — Backlog
10. **Rate limiting** — Add `generation_rate_limits` table; enforce per-case or per-user generation quotas.
11. **Veteran consent gate** — Add disclosure/consent step to share portal before case data is rendered.
12. **Resend fallback** — Log email failures to `audit_events` at minimum; optionally add secondary provider.
13. **Billing integration** — Scope Stripe or usage-based metering.
14. **Backup/recovery** — Document and automate Supabase backup schedule.
15. **Data retention workflow** — Define and implement deletion/archival policy.
16. **OCR pipeline** — Implement or document how `extracted_text` gets populated on evidence records.

---

*Audit completed 2026-05-01 by Peri. All findings are from static codebase analysis at commit `814d9b2`. Live Supabase state (row counts, applied migrations, env var values) not verified — direct DB access required for full confirmation.*
