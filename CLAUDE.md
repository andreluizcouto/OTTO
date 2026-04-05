<!-- GSD:project-start source:PROJECT.md -->
## Project

**FinCoach AI**

Um assistente financeiro pessoal com IA que automatiza o controle de gastos, classifica transações inteligentemente, gera insights sobre hábitos de consumo e atua como um coach financeiro proativo — enviando lembretes e cutucadas via WhatsApp para ajudar o usuário a atingir suas metas. Diferente de planilhas e apps genéricos, o objetivo é ser um "Pierre" acessível para amigos, família e, eventualmente, um público maior.

**Core Value:** Classificação automática de gastos com IA + coach financeiro proativo que cutuca o usuário via WhatsApp para atingir metas — sem esforço manual.

### Constraints

- **Tech Stack**: Python (Streamlit) + Make.com (webhooks/automação) + Supabase (DB/Auth) + Z-API (WhatsApp) + OpenAI (IA) — escolha do usuário, não negociável
- **Custo**: MVP deve ter custo mínimo — dados simulados, sem APIs pagas de Open Finance
- **Experiência Supabase**: Usuário tem conhecimento mínimo — setup deve ser guiado e simples
- **Público MVP**: 5-10 pessoas (usuário + amigos + família)
- **Qualidade demo**: Precisa estar apresentável para potenciais investidores
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Technologies
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Python | 3.11.x | Runtime | Already installed (3.11.9). Stable, fast, excellent library support. 3.11 has 10-60% speed improvement over 3.10. Avoid 3.13 for now -- Streamlit and supabase-py ecosystem fully tested on 3.11. |
| Streamlit | >=1.56.0 | Web UI / Dashboard | Latest stable. Built-in `st.connection` for database integration, native charting, session state for auth, multipage app support. Fastest path from Python to interactive dashboard. |
| Supabase (supabase-py) | >=2.28.3 | Database + Auth | Official Python SDK. Wraps PostgREST, Auth, Storage, Realtime. Provides typed query builder for PostgreSQL. Includes Row Level Security (RLS) support out of the box. |
| OpenAI | >=2.30.0 | AI classification + insights | Latest v2 SDK with Structured Outputs (response_format with Pydantic models), streaming, function calling. Critical for reliable JSON extraction from financial data. |
| Make.com | N/A (SaaS) | Backend orchestration | Webhook-based automation platform. Handles: email parsing, OpenAI calls, Supabase writes, Z-API messaging, scheduled triggers. No server to manage. |
| Z-API | N/A (SaaS) | WhatsApp messaging | Brazilian WhatsApp Business API. REST-based, excellent Make.com integration via HTTP module. Handles proactive notifications. |
### Supporting Libraries
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| pydantic | >=2.12.5 | Data validation + OpenAI Structured Outputs | Always. Define transaction schemas, API response models, configuration. OpenAI SDK v2 uses Pydantic models directly for Structured Outputs. |
| plotly | >=6.6.0 | Interactive charts | Primary charting library. Use via `st.plotly_chart()`. Superior to Streamlit native charts for financial dashboards (candlestick, sunburst, treemap for categories). |
| pandas | >=3.0.2 | Data manipulation | Always. Transform Supabase query results into DataFrames for aggregation, time-series analysis, category grouping. Streamlit has native DataFrame rendering. |
| httpx | >=0.28.1 | HTTP client | For direct Z-API calls from Streamlit if needed (bypassing Make.com). Already a supabase-py dependency, so no extra install. Prefer over requests for async support. |
| python-dotenv | >=1.2.2 | Environment variables (local dev) | Local development only. Load `.env` file with Supabase URL, keys, OpenAI key. In production (Streamlit Cloud), use `st.secrets` instead. |
| Faker | >=40.12.0 | Simulated financial data | MVP phase only. Generate realistic Brazilian financial transactions (pt_BR locale). Use for seeding Supabase with test data without Open Finance integration. |
| streamlit-option-menu | >=0.4.0 | Navigation sidebar | Better sidebar navigation than default Streamlit multipage. Professional-looking icon-based menu for dashboard sections. |
### Development Tools
| Tool | Purpose | Notes |
|------|---------|-------|
| uv | Package management | Faster than pip. Use `uv pip install` or `uv venv`. Drop-in replacement. If not installed, pip works fine. |
| Streamlit Cloud | Deployment | Free tier for public repos. Built-in secrets management (`st.secrets`). Auto-deploy from GitHub. Best zero-cost deployment for Streamlit apps. |
| Supabase Dashboard | DB management | Web UI for schema design, RLS policies, auth configuration. No SQL client needed for MVP. |
| Make.com Scenario Editor | Automation design | Visual workflow builder. Use "Watch Webhooks" trigger for Streamlit-to-Make communication. |
## Installation
# Create virtual environment
# .venv\Scripts\activate   # Windows
# Core dependencies
# Dashboard & data
# MVP data generation
# Development
# HTTP client (already installed as supabase dependency, but pin explicitly)
### requirements.txt
## Alternatives Considered
| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Streamlit | Dash (Plotly) | When you need deeply custom layouts, callback-based interactivity, or enterprise features. Streamlit is faster to build; Dash gives more control. For an MVP, Streamlit wins. |
| Streamlit | Gradio | Only if the app were purely an AI chat interface. Gradio lacks dashboard/charting capabilities needed for financial visualization. |
| supabase-py | SQLAlchemy + psycopg2 | When you need complex ORM patterns, raw SQL control, or migrations. supabase-py is simpler and includes auth. For MVP, supabase-py is sufficient. |
| supabase-py | Firebase (Python Admin SDK) | Never for this project. Firebase lacks PostgREST query builder, has weaker SQL capabilities, and the user already chose Supabase. |
| plotly | Altair (via st.altair_chart) | When you want declarative grammar-of-graphics style. Plotly is more mature for financial charts (candlestick, waterfall, gauges). |
| plotly | matplotlib | Never for interactive dashboards. Matplotlib produces static images. Plotly is interactive, zoomable, tooltips. |
| OpenAI API | Local LLM (Ollama) | When privacy is paramount or API costs are prohibitive. For MVP with simulated data, OpenAI is fine. Local LLMs add deployment complexity. |
| Make.com | n8n (self-hosted) | When you need full control over automation server, or Make.com pricing becomes an issue at scale. For MVP, Make.com's visual editor and zero-infra approach is ideal. |
| httpx | requests | When the team is more familiar with requests. httpx is already a transitive dependency (supabase-py uses it), supports async, and has a near-identical API. No reason to add requests separately. |
| Faker | Custom seed script | When you need highly domain-specific data (e.g., realistic Brazilian bank statement patterns). Start with Faker, customize as needed. |
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| streamlit-authenticator | Adds YAML-based auth, password hashing complexity. Doesn't leverage Supabase Auth (which is already in the stack). Creates auth fragmentation. | Supabase Auth via supabase-py (`client.auth.sign_in_with_password()`). Use `st.session_state` to store the session. |
| st-supabase-connection | Convenience wrapper but adds abstraction layer over supabase-py, may lag behind SDK updates, and makes debugging harder. | Direct supabase-py client. `create_client(url, key)` is already simple enough. |
| Flask/FastAPI as backend | Make.com IS the backend. Adding a Python server defeats the no-server architecture choice. Creates deployment complexity and cost. | Make.com webhooks for all backend logic. Streamlit calls Make.com via HTTP. |
| openai v1.x (legacy) | Completely different API surface. No Structured Outputs, no Pydantic integration, deprecated patterns. | openai >=2.0.0. The v2 SDK has typed responses, Structured Outputs, and better error handling. |
| pandas <2.0 | Missing Copy-on-Write, Arrow-backed dtypes, and performance improvements. | pandas >=3.0.2 (or at minimum >=2.0). |
| SQLAlchemy for Supabase | Over-engineering for this use case. supabase-py already provides a query builder. SQLAlchemy adds migration complexity and doesn't use PostgREST. | supabase-py query builder: `client.table("transactions").select("*").eq("user_id", uid).execute()` |
| Streamlit native charts only | `st.line_chart`, `st.bar_chart` are limited. No tooltips, no custom colors, no financial chart types. | plotly for all charts beyond the most basic. Use `st.plotly_chart(fig, use_container_width=True)`. |
## Stack Patterns by Architecture
- Streamlit sends data to Make.com via `httpx.post(MAKE_WEBHOOK_URL, json=payload)`
- Make.com processes (calls OpenAI, writes to Supabase, sends WhatsApp)
- Streamlit reads results directly from Supabase via supabase-py
- Pattern: Write via Make.com, Read via supabase-py
- Use Supabase Auth (`client.auth.sign_in_with_password()`)
- Store JWT in `st.session_state["access_token"]`
- Create authenticated Supabase client per session
- RLS policies on Supabase tables filter data by `auth.uid()`
- Make.com's HTTP module calls OpenAI API directly
- Use Structured Outputs (JSON Schema / response_format) for reliable classification
- Define category taxonomy as enum in the prompt
- Model recommendation: `gpt-4o-mini` for classification (cheap, fast, accurate enough)
- Model recommendation: `gpt-4o` for monthly insight generation (needs reasoning)
- Z-API provides REST endpoints for sending WhatsApp messages
- Make.com calls Z-API HTTP endpoints with instance ID + token
- Endpoint pattern: `https://api.z-api.io/instances/{INSTANCE_ID}/token/{TOKEN}/send-text`
- Scheduled scenarios in Make.com trigger proactive messages
- Python script using Faker (pt_BR locale) + custom financial templates
- Generate: transactions, categories, merchants, dates, amounts
- Seed directly into Supabase via supabase-py
- Run once to populate, then use the app normally
## Version Compatibility
| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| streamlit >=1.56.0 | Python 3.9-3.12 | Verified 3.11.x works. Streamlit dropped 3.8 support. |
| supabase >=2.28.3 | Python 3.9+ | Uses httpx internally. Async support available via `acreate_client()`. |
| openai >=2.30.0 | pydantic >=2.0 | Structured Outputs require pydantic v2 models. Do NOT mix with pydantic v1. |
| plotly >=6.0.0 | streamlit >=1.40.0 | Plotly 6.x requires `st.plotly_chart()`. Works with `use_container_width=True`. |
| pandas >=3.0.0 | Python 3.10+ | pandas 3.x dropped Python 3.9. Fine for 3.11. |
| Faker >=40.0 | Python 3.9+ | pt_BR locale included by default. |
## Supabase Schema Considerations
## Environment Variables
# .env (local development)
## Sources
- pip index (verified 2026-04-05) -- All version numbers confirmed against PyPI
- supabase-py installed locally (2.28.3) -- Dependencies verified via `pip show`
- Training data (May 2025 cutoff) -- Architecture patterns, Z-API endpoints, Make.com patterns
- Confidence note: Z-API endpoint patterns and Make.com specifics are from training data (MEDIUM confidence). Verify Z-API docs at https://developer.z-api.io/ before implementation.
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, or `.github/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
