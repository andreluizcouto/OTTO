# Project Research Summary

**Project:** FinCoach AI
**Domain:** AI-powered personal finance assistant (Brazilian market)
**Researched:** 2026-04-05
**Confidence:** MEDIUM

## Executive Summary

FinCoach AI is an AI-powered personal finance coach targeting Brazilian users, differentiated by proactive WhatsApp-based coaching rather than the passive data-display approach of existing Brazilian apps like Mobills, Organizze, and Guiabolso. The expert-recommended approach for this class of product is a low-code orchestration architecture: Streamlit as the frontend dashboard, Supabase as the central data hub (PostgreSQL + Auth + Row Level Security), Make.com as the serverless backend orchestrator, and OpenAI for transaction classification and insight generation. This eliminates the need for a traditional backend server, dramatically reducing time-to-MVP. The stack is Python-only on the application side, with all external service integrations routed through Make.com scenarios rather than custom API code.

The core product hypothesis -- "AI coaching via WhatsApp changes how Brazilians manage money" -- can be validated with simulated data, removing the largest onboarding friction (bank connection) that plagues every competitor. No major Brazilian finance app currently uses WhatsApp for proactive financial coaching, despite WhatsApp having 99% smartphone penetration in Brazil. This is the strongest competitive gap identified in the research. The MVP should focus exclusively on validating this coaching experience, deferring Open Finance integration, manual transaction entry, and investment tracking.

The primary risks are: (1) Streamlit's session state resets causing auth instability, (2) Supabase RLS misconfigurations silently hiding data instead of throwing errors, (3) Make.com's lack of version control making the backend a fragile black box, and (4) OpenAI's classification accuracy on cryptic Brazilian merchant names (e.g., "RCHLO", "Pag*JoseDaSilva"). All four risks have documented prevention strategies, but they require deliberate attention during the earliest build phases -- not as afterthoughts. The Make.com operation limit (1,000 ops/month on free tier) is a hard constraint that will require a paid plan even for MVP-scale usage.

## Key Findings

### Recommended Stack

The stack is built around four pillars: Streamlit (>=1.56.0) for rapid Python-to-dashboard development, Supabase (supabase-py >=2.28.3) for managed PostgreSQL with built-in auth and RLS, Make.com for webhook-driven backend orchestration, and OpenAI (>=2.30.0) for AI-powered classification and insight generation. All libraries are verified compatible with Python 3.11.x, which is already installed.

**Core technologies:**
- **Python 3.11.x**: Runtime -- 10-60% faster than 3.10, fully tested ecosystem compatibility
- **Streamlit >=1.56.0**: Frontend dashboard -- fastest path from Python to interactive UI, built-in auth support via session state
- **Supabase (supabase-py >=2.28.3)**: Data hub + auth -- PostgreSQL with RLS for multi-tenancy, no ORM needed
- **Make.com**: Backend orchestrator -- replaces a custom API server, handles all AI and WhatsApp integrations
- **OpenAI >=2.30.0**: AI engine -- Structured Outputs with Pydantic models for reliable JSON classification
- **Z-API**: WhatsApp messaging -- REST-based Brazilian WhatsApp Business API, integrates with Make.com
- **Plotly >=6.6.0**: Financial charts -- interactive, supports candlestick/treemap/sunburst, superior to Streamlit native charts
- **Pandas >=3.0.2**: Data manipulation -- transform Supabase results into DataFrames for aggregation

**Critical version requirements:** OpenAI v2 SDK requires Pydantic v2 (not v1). Pandas 3.x requires Python 3.10+. Plotly 6.x requires Streamlit >=1.40.0.

### Expected Features

**Must have (table stakes):**
- Transaction categorization via AI (OpenAI) -- every competitor auto-categorizes; ~85-90% accuracy target
- Spending by category visualization -- pie/bar charts showing "where my money goes"
- Monthly spending summary -- income, expenses, balance, trend vs previous month
- Budget limits per category with visual tracking -- envelope-style, green/yellow/red indicators
- Over-budget WhatsApp alerts -- first proactive coaching touchpoint
- Secure per-user data isolation -- Supabase Auth + RLS, non-negotiable even for 5 users
- Period filtering and income vs expenses overview -- basic navigation and cash flow view

**Should have (differentiators):**
- AI-generated monthly narrative insights -- "You spent 30% more on delivery this month" (key differentiator vs data-display-only Brazilian apps)
- Proactive WhatsApp coaching with smart timing -- Friday nights, post-payday, near goal deadlines (no Brazilian competitor does this)
- Financial goal tracking with WhatsApp reminders -- active coaching, not passive tracking
- Simulated data for instant onboarding -- users see value in seconds, critical for demos
- Coaching personality in Brazilian Portuguese -- warm, informal, humorous (Cleo proved personality matters)

**Defer (v2+):**
- Open Finance integration (Pluggy/Belvo) -- R$0.50-2.00/user/month, complex auth, regulatory burden
- Receipt scanning via WhatsApp -- OCR accuracy issues with Brazilian receipt formats
- Manual transaction entry -- anti-feature that defeats the zero-effort value proposition
- Investment tracking -- different product domain, regulatory complexity
- Family/couple shared budgets -- doubles data model complexity

### Architecture Approach

The system follows a hub-and-spoke architecture with Supabase as the central data hub, Streamlit as the user-facing spoke (reads directly from Supabase), and Make.com as the automation spoke (handles all writes that require AI processing). This creates three distinct execution contexts that never blend: Streamlit renders UI and reads data, Make.com orchestrates AI and messaging workflows, and Supabase stores everything with RLS enforcing data isolation. The critical design principle is "write via Make.com, read via Supabase" -- the Streamlit frontend never calls OpenAI or Z-API directly.

**Major components:**
1. **Streamlit Frontend** -- Auth gate, dashboard rendering, data visualization, user input capture. Multipage app with pages for dashboard, transactions, budgets, goals, and insights.
2. **Supabase Data Hub** -- PostgreSQL with 7 core tables (profiles, transactions, categories, budgets, goals, insights, whatsapp_messages). RLS on every table. Auth via email/password with JWT.
3. **Make.com Orchestrator** -- 4 scenarios: (a) Transaction Classifier (webhook), (b) Monthly Insight Generator (scheduled), (c) Goal Reminder (daily schedule), (d) Simulated Data Generator (webhook, MVP only).
4. **External Services** -- OpenAI (gpt-4o-mini for classification, gpt-4o for insights) and Z-API (WhatsApp messaging), both called exclusively by Make.com.

### Critical Pitfalls

1. **Supabase RLS silent empty results** -- Misconfigured RLS returns empty data instead of errors. Test with 2+ users from day 1; verify User A cannot see User B's data before building anything else.
2. **Streamlit session state resets** -- Auth tokens vanish on browser refresh. Combine `st.session_state` with cookie persistence or Supabase session validation on every rerun.
3. **Multi-user cache leakage** -- `st.cache_data` is global across all users. Always include `user_id` in function signatures used as cache keys, or avoid caching user-specific financial data entirely.
4. **Make.com as unversionable black box** -- No git, no rollback, no tests. Export scenario blueprints to git regularly, keep scenarios small and single-purpose, document every step.
5. **OpenAI classification accuracy on Brazilian data** -- Cryptic merchant names like "RCHLO" and "Pag*JoseDaSilva" will confuse generic prompts. Include 20+ Brazilian examples in prompts, build a merchant name mapping table, prompt in Portuguese.

## Implications for Roadmap

Based on combined research, the architecture has clear dependency chains that dictate a 6-phase build order. Each phase delivers a testable increment.

### Phase 1: Foundation (Auth + Database + App Skeleton)
**Rationale:** Everything depends on Supabase schema and auth. The two most critical pitfalls (RLS silent failures, session state resets) must be solved here before any feature work.
**Delivers:** User can sign up, log in, stay logged in across refreshes, and see an empty but functional dashboard shell. RLS verified with 2+ test users.
**Addresses:** Secure per-user data isolation (table stakes), period filtering skeleton
**Avoids:** RLS silent empty results (Pitfall #2), session state resets (Pitfall #1), cache leakage (Pitfall #3 -- establish patterns early)

### Phase 2: Data Layer (Simulated Data + Dashboard)
**Rationale:** No feature can be demoed or tested without data. Simulated data removes the Open Finance dependency entirely and enables instant onboarding -- a key differentiator.
**Delivers:** User clicks "Generate Sample Data," realistic Brazilian transactions appear, dashboard displays spending by category, monthly summary, income vs expenses.
**Addresses:** Simulated data for instant onboarding (differentiator), spending by category charts (table stakes), monthly spending summary (table stakes), income vs expenses overview (table stakes)
**Uses:** Make.com Scenario 4 (data generator), Plotly charts, Pandas aggregation
**Avoids:** Make.com webhook timeouts (Pitfall #7 -- use async pattern from start)

### Phase 3: AI Classification
**Rationale:** Categorized transactions unlock the entire value chain -- budgets, insights, and coaching all depend on accurate categories. This is the first real AI integration and the riskiest technical phase.
**Delivers:** Transactions are automatically classified into Brazilian spending categories by OpenAI. Dashboard shows classified data with confidence indicators.
**Addresses:** AI transaction categorization (table stakes), category spending visualization (enhanced with real categories)
**Uses:** Make.com Scenario 1 (classifier), OpenAI gpt-4o-mini, Structured Outputs
**Avoids:** Brazilian merchant name misclassification (Pitfall #4 -- build example-rich prompts, test with realistic data)

### Phase 4: Budgets and WhatsApp Alerts
**Rationale:** Budget tracking is table stakes, and over-budget alerts are the first WhatsApp touchpoint -- the core of the product hypothesis. These belong together because alerts require both the budget system AND the messaging pipeline.
**Delivers:** Users set category budgets, see visual progress (green/yellow/red), and receive WhatsApp messages when budgets are exceeded.
**Addresses:** Budget limits per category (table stakes), budget progress visualization (table stakes), over-budget WhatsApp alerts (table stakes + differentiator)
**Uses:** Z-API via Make.com, budget check logic piggybacking on classification scenario
**Avoids:** Z-API disconnection drops (Pitfall #5 -- build health checks and message queue), Make.com versioning drift (Pitfall #3 -- export blueprints)

### Phase 5: AI Insights and Goals
**Rationale:** With classification working and WhatsApp proven, layer on the premium coaching features. Insights and goals are independent of each other and can be developed in parallel.
**Delivers:** Monthly AI-generated spending narrative ("You spent 30% more on delivery"), financial goals with WhatsApp reminders, coaching personality in Brazilian Portuguese.
**Addresses:** AI monthly insights (differentiator), financial goal tracking with reminders (differentiator), coaching personality (differentiator)
**Uses:** Make.com Scenarios 2 and 3, OpenAI gpt-4o for insight quality, Z-API for reminders

### Phase 6: Polish and Demo Readiness
**Rationale:** After all features work, refine the experience for real users and investor demos. Error handling, onboarding flow, and visual polish turn a prototype into a product.
**Delivers:** First-time user onboarding, graceful error handling, demo-quality UI, documented deployment to Streamlit Cloud.
**Addresses:** Production readiness, investor demo capability
**Avoids:** Technical debt accumulation (hardcoded categories, no error logging, missing data backups)

### Phase Ordering Rationale

- **Phases 1-3 are strictly sequential**: Auth/DB must exist before data, data must exist before classification. No parallelism possible.
- **Phases 4 and 5 can partially overlap**: Both depend on Phase 3 (classified data), but are independent of each other. Budget alerts (Phase 4) should come first because they validate the WhatsApp pipeline that Phase 5 also needs.
- **Phase 6 is deliberately last**: Polish is wasted effort if the core hypothesis fails. Build ugly-but-functional first, then refine.
- **Simulated data in Phase 2 (not later)**: This unblocks ALL subsequent testing and demos. Without it, every phase requires manual database seeding.
- **Make.com operations budget**: With 4 scenarios and MVP-scale usage, the free tier (1,000 ops/month) will be exhausted quickly. Budget for the $9/month Pro plan from Phase 2 onward.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (AI Classification):** Brazilian merchant name patterns are poorly documented. Needs hands-on prompt engineering and a test suite of realistic transaction descriptions. Consider `/gsd:research-phase` for OpenAI Structured Outputs best practices and Brazilian financial vocabulary.
- **Phase 4 (WhatsApp Integration):** Z-API documentation is niche with low confidence in research. Needs live API exploration, connection stability testing, and Make.com module verification. Consider `/gsd:research-phase` for Z-API endpoints and error handling patterns.
- **Phase 5 (AI Insights):** Prompt design for generating useful (not hallucinated) financial insights in Portuguese needs experimentation. The quality bar is high -- bad insights erode trust.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Foundation):** Supabase Auth + RLS and Streamlit app structure are well-documented with established patterns.
- **Phase 2 (Data Layer):** Streamlit dashboard rendering, Plotly charts, and Pandas aggregation are thoroughly documented.
- **Phase 6 (Polish):** Standard UI/UX refinement and deployment patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All package versions verified against PyPI on 2026-04-05. Compatibility matrix confirmed. Python 3.11.x ecosystem is mature. |
| Features | MEDIUM | Competitor analysis based on training data (cutoff early 2025). WhatsApp gap and Brazilian market dynamics are stable findings, but individual competitor feature sets may have evolved. |
| Architecture | MEDIUM | Hub-and-spoke pattern with Supabase/Streamlit/Make.com is established, but Make.com-specific module behavior and Z-API integration details are from training data, not live-verified. |
| Pitfalls | MEDIUM | Pitfalls are derived from well-known framework limitations (Streamlit rerun model, Supabase RLS behavior) and community-reported issues. Z-API stability concerns are lower confidence. |

**Overall confidence:** MEDIUM -- The stack is solid and verified. The architecture pattern is sound. The main uncertainty lies in Z-API reliability, Make.com operational limits at scale, and OpenAI classification accuracy on real Brazilian financial data. All three will need empirical validation during implementation.

### Gaps to Address

- **Z-API stability and reconnection**: Low confidence in training data. Must test connection persistence, disconnection behavior, and reconnection procedures with a live Z-API instance before committing to Phase 4 timeline.
- **Make.com operation budget**: Need precise calculation of operations per scenario per month at MVP scale (5-10 users). Free tier is almost certainly insufficient; confirm Pro plan ($9/month) covers projected usage.
- **OpenAI classification accuracy on Brazilian data**: No benchmark exists. Must build a test set of 50-100 realistic Brazilian transaction descriptions and measure classification accuracy before relying on it for the dashboard.
- **Streamlit Cloud deployment limits**: Free tier supports 1 public app. Confirm this is sufficient for MVP, including secrets management and custom domain (if needed).
- **Supabase free tier limits**: 500MB database, 50K monthly auth requests. Should be sufficient for MVP but needs monitoring as simulated data accumulates.

## Sources

### Primary (HIGH confidence)
- PyPI package index (verified 2026-04-05) -- All library versions and compatibility confirmed
- OpenAI API documentation -- Structured Outputs, JSON mode, model capabilities

### Secondary (MEDIUM confidence)
- Streamlit documentation (training data) -- Caching, session state, multipage, secrets management
- Supabase documentation (training data) -- RLS, Auth, Python client, schema patterns
- Make.com documentation (training data) -- Webhook patterns, Supabase module, scheduling
- Competitor analysis (Cleo, Guiabolso, Mobills, Organizze, Olivia AI) -- Feature sets and market positioning from training data through early 2025

### Tertiary (LOW confidence)
- Z-API documentation and patterns -- Niche Brazilian API, limited training data coverage. Verify at https://developer.z-api.io/ before implementation.
- Make.com operational limits and pricing -- May have changed since training data cutoff. Verify current plans.

---
*Research completed: 2026-04-05*
*Ready for roadmap: yes*
