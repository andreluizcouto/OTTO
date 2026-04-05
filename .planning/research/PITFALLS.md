# Pitfalls Research: FinCoach AI

**Research Date:** 2026-04-05
**Domain:** AI-powered personal finance assistant
**Confidence:** MEDIUM

## Critical Pitfalls

### 1. Streamlit Session State Resets

**Severity:** HIGH
**Phase:** 1 (Auth + Foundation)

Streamlit reruns the entire script on every interaction. Auth tokens stored in `st.session_state` vanish on browser refresh or tab switch.

**Warning Signs:**
- Users randomly logged out after switching tabs
- "Not authenticated" errors after idle time
- Auth state inconsistent across pages

**Prevention Strategy:**
- Use `st.session_state` combined with browser cookies (`streamlit-cookies-controller` or `extra-streamlit-components`)
- Store session tokens in Supabase and validate on each rerun
- Implement `st.query_params` for token persistence as fallback
- Test: open app, switch tabs, wait 5 minutes, come back — should still be logged in

### 2. Supabase RLS Silent Empty Results

**Severity:** CRITICAL
**Phase:** 1 (Database Setup)

When Row Level Security (RLS) policies are misconfigured, queries return **empty results** instead of errors. For a first-time Supabase user, this looks like "no data" when data actually exists but is invisible.

**Warning Signs:**
- Dashboard shows zero transactions when data exists in Supabase
- New user sees other users' data (policies too permissive)
- Data visible in Supabase dashboard but not in app

**Prevention Strategy:**
- Enable RLS on EVERY table from day 1 (never leave it for later)
- Create a test script that verifies: User A cannot see User B's transactions
- Use `auth.uid()` in all policies: `USING (user_id = auth.uid())`
- Test with 2+ test users before moving to other features
- Add a "data visibility smoke test" to every phase

### 3. Make.com as Unversionable Black Box

**Severity:** HIGH
**Phase:** All phases using Make.com

Make.com scenarios have no git, no version control, no automated tests, and no rollback. When it's your sole backend, a bad edit can break everything with no undo.

**Warning Signs:**
- "I changed something and now classification doesn't work"
- No way to compare current scenario with previous version
- Debugging by trial and error in the Make.com editor

**Prevention Strategy:**
- Document every Make.com scenario step-by-step in `.planning/` (screenshots + description)
- Export scenario JSON regularly (Make > Export Blueprint) and commit to git
- Keep scenarios small and single-purpose (1 trigger = 1 outcome)
- Use Make.com's scenario versioning feature (limited but better than nothing)
- For critical logic, consider putting it in Python (Supabase Edge Function or webhook endpoint) where it CAN be versioned

### 4. OpenAI Classification Accuracy for Brazilian Financial Data

**Severity:** HIGH
**Phase:** 3 (AI Classification)

Brazilian bank transactions have cryptic merchant names: "RCHLO" (Riachuelo), "Pag*JoseDaSilva" (PagSeguro payment), "PIX TRANSF JOAO M" (Pix transfer). Generic OpenAI prompts trained on US financial data will misclassify frequently.

**Warning Signs:**
- "Pag*" merchants classified as different categories each time
- PIX transfers all lumped into one category
- Classification accuracy below 70% on real-looking data

**Prevention Strategy:**
- Use OpenAI Structured Outputs (response_format with JSON schema) — not free-text
- Build a Brazilian merchant name mapping table in Supabase (abbreviation → full name → category)
- Include Brazilian-specific examples in the classification prompt (at least 20 examples)
- Add a "confidence score" field — low-confidence classifications get flagged for user review
- Test with realistic Brazilian transaction descriptions, not clean English data
- Prompt in Portuguese with Portuguese category names

### 5. Z-API WhatsApp Connection Drops

**Severity:** MEDIUM
**Phase:** 4-5 (WhatsApp Integration)

Z-API sessions disconnect silently when the connected phone changes network, updates WhatsApp, or after extended idle periods. Messages sent during disconnection are silently lost.

**Warning Signs:**
- Scheduled reminders stop arriving with no error in Make.com
- Z-API status shows "disconnected" but Make.com scenario shows "success"
- Users report not receiving messages for days

**Prevention Strategy:**
- Add a health-check step in Make.com before sending: call Z-API status endpoint first
- Implement a "message sent" confirmation table in Supabase
- Set up a daily Make.com scenario that sends a test message to yourself
- Have a reconnection procedure documented (QR code scan)
- Store unsent messages in a queue table for retry

### 6. Multi-User Cache Leakage in Streamlit

**Severity:** HIGH
**Phase:** 2 (Dashboard)

`st.cache_data` and `st.cache_resource` are GLOBAL across all users. If you cache query results without including `user_id` in the cache key, User B can see User A's financial data.

**Warning Signs:**
- Switching between test users shows stale data from previous user
- Dashboard shows correct data on first load but wrong data after navigation
- Numbers that don't match what's in Supabase for a specific user

**Prevention Strategy:**
- NEVER cache user-specific data without `user_id` in the function signature
- Use `st.cache_data(ttl=300)` with short TTL for financial data
- Better: don't cache financial data at all — Supabase queries are fast enough for MVP scale
- If you must cache: `@st.cache_data` with `def get_transactions(user_id: str)` — the `user_id` param becomes part of the cache key
- Add cache-clearing on logout: `st.cache_data.clear()`

### 7. Make.com Webhook Timeouts and Operation Limits

**Severity:** MEDIUM-HIGH
**Phase:** 3+ (Any Make.com scenario)

Make.com webhooks timeout after 40 seconds. If OpenAI classification takes longer (batch of transactions), the webhook returns timeout error to Streamlit. Also: free tier = 1,000 operations/month. A classification scenario with 5 modules × 30 transactions × 10 users = 1,500 ops — already over limit.

**Warning Signs:**
- Streamlit shows "webhook timeout" errors intermittently
- Make.com dashboard shows operation count climbing fast
- Scenarios stop running mid-month ("operation limit reached")

**Prevention Strategy:**
- Use ASYNCHRONOUS pattern: Streamlit inserts unclassified transactions → Make.com polls or uses scheduled trigger to classify in background → Streamlit shows "processing" indicator
- Never make Streamlit wait synchronously for Make.com webhook response
- Calculate operation budget before building: count modules × expected transactions × users × 30 days
- Consider Make.com Pro plan ($9/month, 10,000 ops) from the start
- Batch transactions: classify 10 at once in a single OpenAI call instead of 1-by-1

## Technical Debt Patterns

| Pattern | Risk | When It Bites |
|---------|------|---------------|
| Hardcoded categories | Can't add new categories without code change | When users want custom categories |
| No error logging | Silent failures impossible to debug | First production issue |
| Secrets in code | Supabase keys/Z-API tokens in Python files | First git push |
| No data backup | Supabase data loss = total loss | First database incident |
| Single environment | Dev changes affect "production" | First user-facing bug |

## Integration Gotchas

1. **Supabase Python SDK auth + Streamlit**: `supabase.auth.sign_in_with_password()` returns a session object — you must store `session.access_token` and use it for subsequent queries. Don't create a new client per request.

2. **Make.com + Supabase**: Make.com's Supabase module uses the service_role key (bypasses RLS). This is correct for backend operations but means Make.com scenarios must manually filter by user_id.

3. **Make.com + OpenAI**: Use the "Create a Chat Completion" module with JSON mode. Set max_tokens appropriately (classification responses are small, ~200 tokens). Don't use streaming in Make.com.

4. **Streamlit multipage + auth**: Each page file reruns independently. Auth check must be at the top of EVERY page file, not just the main app.py.

## "Looks Done But Isn't" Checklist

- [ ] Auth works → but does it persist across refresh?
- [ ] Dashboard shows data → but only YOUR data (RLS)?
- [ ] Classification works → but with Brazilian merchant names?
- [ ] WhatsApp sends → but does it reconnect after disconnection?
- [ ] Budget alerts fire → but at the right threshold, not on every transaction?
- [ ] Insights generate → but are they accurate, not hallucinated?
- [ ] Multiple users → but cached data isolated?

## Phase Mapping

| Pitfall | Relevant Phase | Priority |
|---------|---------------|----------|
| Streamlit session state | Phase 1 (Auth) | Must fix before anything else |
| Supabase RLS | Phase 1 (DB setup) | Must test with 2+ users |
| Make.com versioning | All phases | Ongoing discipline |
| OpenAI Brazilian accuracy | Phase 3 (Classification) | Test with realistic data |
| Z-API disconnections | Phase 4-5 (WhatsApp) | Build health checks |
| Cache leakage | Phase 2 (Dashboard) | Test with user switching |
| Webhook timeouts | Phase 3+ (Make.com) | Use async pattern from start |

---
*Research completed: 2026-04-05*
