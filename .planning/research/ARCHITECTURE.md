# Architecture Patterns

**Domain:** AI-powered personal finance assistant (FinCoach AI)
**Researched:** 2026-04-05
**Confidence:** MEDIUM (based on established patterns for Streamlit/Supabase/Make.com; no web verification available during research)

## System Overview

FinCoach AI is a **hub-and-spoke architecture** where Supabase is the central data hub, Streamlit is the user-facing spoke, and Make.com is the automation spoke. OpenAI and Z-API are external services called exclusively by Make.com -- the Streamlit frontend never calls them directly.

This is NOT a traditional client-server architecture. It is a **low-code orchestration architecture** where Make.com replaces a conventional backend API server, and Supabase replaces both the database layer and the auth layer. Streamlit reads from Supabase directly (no backend API needed for reads). Writes that require AI processing go through Make.com webhooks.

```
+-------------------+        +-------------------+        +-------------------+
|                   |  reads  |                   | writes  |                   |
|   Streamlit UI    |-------->|     Supabase      |<--------|    Make.com       |
|   (Frontend)      |  direct |   (Data Hub)      |  direct |  (Orchestrator)  |
|                   |-------->|   PostgreSQL +    |         |                   |
|                   |  auth   |   Auth + RLS      |         |                   |
+-------------------+        +-------------------+        +-------------------+
        |                                                        |       |
        | webhook trigger                              calls     |       | calls
        | (new transaction,                                      |       |
        |  budget update)                                        v       v
        +----------------------------------------------->  +-------+ +-------+
                                                           |OpenAI | | Z-API |
                                                           |  API  | | Whats |
                                                           +-------+ +-------+
```

## Recommended Architecture

### The "Supabase as Central Hub" Pattern

Every component reads from or writes to Supabase. This is the correct pattern because:

1. **Single source of truth** -- no data duplication between services
2. **Supabase RLS (Row Level Security)** handles multi-tenancy natively -- each user sees only their data without application-level filtering
3. **Real-time subscriptions** available if needed later (Supabase Realtime)
4. **Make.com has native Supabase modules** -- no custom API needed

### Three Execution Contexts

This system operates in three distinct execution contexts that never blend:

| Context | Runtime | Trigger | Responsibilities |
|---------|---------|---------|------------------|
| **Streamlit App** | Python on Streamlit Cloud / local | User opens browser | Display data, capture user input, trigger webhooks |
| **Make.com Scenarios** | Make.com cloud | Webhooks, schedules, Supabase triggers | AI classification, insight generation, WhatsApp messaging |
| **Supabase** | Managed PostgreSQL | Direct queries, RLS policies | Data storage, auth, row-level security, database functions |

This separation is critical. Streamlit does NOT call OpenAI. Make.com does NOT render UI. Supabase does NOT orchestrate workflows.

## Component Boundaries

### Component 1: Streamlit Frontend

| Aspect | Detail |
|--------|--------|
| **Responsibility** | User authentication, dashboard rendering, data visualization, user input capture |
| **Communicates With** | Supabase (direct reads via supabase-py), Make.com (webhook POST for write operations) |
| **Does NOT do** | AI calls, WhatsApp messaging, heavy data processing |
| **State Management** | `st.session_state` for auth tokens and UI state; no persistent state outside Supabase |

**Internal Structure (recommended page layout):**

```
app.py                      # Entry point, auth gate, navigation
pages/
  01_dashboard.py           # Main spending overview
  02_transactions.py        # Transaction list with categories
  03_budgets.py             # Budget management per category
  04_goals.py               # Financial goals (e.g., trip savings)
  05_insights.py            # AI-generated insights display
components/
  auth.py                   # Login/signup forms, session management
  charts.py                 # Reusable chart components (Plotly)
  filters.py                # Date range, category filters
utils/
  supabase_client.py        # Singleton Supabase client
  webhook.py                # Make.com webhook caller
  formatters.py             # Currency formatting (BRL), date formatting
config.py                   # Environment variables, constants
```

**Auth Flow in Streamlit:**

```python
# Streamlit auth pattern with Supabase
# 1. User enters email/password
# 2. supabase.auth.sign_in_with_password() returns session
# 3. Store access_token in st.session_state
# 4. Create authenticated Supabase client with user's token
# 5. All subsequent queries use this client (RLS enforced)
# 6. On page refresh: check st.session_state, re-auth if expired
```

**Key constraint:** Streamlit re-runs the entire script on every interaction. This means:
- Supabase client must be cached (`@st.cache_resource`) but auth tokens must be in `st.session_state`
- Heavy queries should use `@st.cache_data(ttl=300)` with a reasonable TTL
- Webhook calls should be behind `st.button` to avoid accidental re-triggers

### Component 2: Supabase (Data Hub + Auth)

| Aspect | Detail |
|--------|--------|
| **Responsibility** | Data persistence, user authentication, row-level security, database functions |
| **Communicates With** | Streamlit (serves queries), Make.com (receives writes, serves reads) |
| **Does NOT do** | Business logic beyond RLS, external API calls |

**Database Schema (recommended):**

```sql
-- Core tables
users                       -- Managed by Supabase Auth (auth.users)

profiles                    -- Extended user info
  id (uuid, FK auth.users)
  name (text)
  whatsapp_number (text)    -- For Z-API messaging
  monthly_income (numeric)
  created_at (timestamptz)

transactions
  id (uuid, PK)
  user_id (uuid, FK auth.users)
  date (date)
  description (text)        -- Original transaction description
  amount (numeric)          -- Always positive, use type for sign
  type (text)               -- 'expense' | 'income'
  category (text)           -- AI-classified category
  subcategory (text)        -- Optional finer classification
  source (text)             -- 'simulated' | 'manual' | 'open_finance'
  ai_confidence (numeric)   -- 0-1, how confident was the AI classification
  classified_at (timestamptz)
  created_at (timestamptz)

categories
  id (uuid, PK)
  user_id (uuid, FK auth.users)
  name (text)               -- e.g., 'Alimentacao', 'Transporte'
  icon (text)               -- Emoji or icon name
  color (text)              -- Hex color for charts
  is_default (boolean)      -- System-provided vs user-created

budgets
  id (uuid, PK)
  user_id (uuid, FK auth.users)
  category (text)
  monthly_limit (numeric)
  period (text)             -- 'monthly' default
  created_at (timestamptz)

goals
  id (uuid, PK)
  user_id (uuid, FK auth.users)
  name (text)               -- e.g., 'Viagem para Europa'
  target_amount (numeric)
  current_amount (numeric)
  target_date (date)
  reminder_frequency (text) -- 'weekly' | 'biweekly' | 'monthly'
  reminder_day (int)        -- Day of week (0=Mon) or day of month
  is_active (boolean)
  created_at (timestamptz)

insights
  id (uuid, PK)
  user_id (uuid, FK auth.users)
  period (text)             -- '2026-03' (monthly)
  insight_type (text)       -- 'spending_trend' | 'budget_alert' | 'comparison'
  content (text)            -- AI-generated text in Portuguese
  data (jsonb)              -- Supporting data (percentages, amounts)
  sent_via_whatsapp (boolean)
  created_at (timestamptz)

whatsapp_messages
  id (uuid, PK)
  user_id (uuid, FK auth.users)
  message_type (text)       -- 'insight' | 'budget_alert' | 'goal_reminder'
  content (text)
  sent_at (timestamptz)
  delivery_status (text)    -- 'sent' | 'delivered' | 'read' | 'failed'
```

**Row Level Security (RLS) -- CRITICAL:**

```sql
-- Every table with user_id MUST have RLS enabled
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;

-- Users can only see their own data
CREATE POLICY "Users see own transactions"
  ON transactions FOR SELECT
  USING (auth.uid() = user_id);

-- Users can only insert their own data
CREATE POLICY "Users insert own transactions"
  ON transactions FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Service role (Make.com) can read/write all data
-- Make.com uses the service_role key, which bypasses RLS
-- This is intentional: Make.com is a trusted backend
```

**Important: Make.com uses `service_role` key (bypasses RLS) because it needs to process data across users (e.g., generate insights for all users on a schedule). Streamlit uses the `anon` key + user JWT (RLS enforced).**

### Component 3: Make.com (Backend Orchestrator)

| Aspect | Detail |
|--------|--------|
| **Responsibility** | AI classification, insight generation, WhatsApp messaging, scheduled jobs |
| **Communicates With** | Supabase (reads/writes), OpenAI API (classification/insights), Z-API (WhatsApp) |
| **Does NOT do** | UI rendering, direct user interaction |

**Scenario Architecture (recommended 4 scenarios):**

#### Scenario 1: Transaction Classifier (Webhook-triggered)

```
Trigger: Custom Webhook (receives transaction data)
  |
  v
[HTTP Module] Call OpenAI API
  - Prompt: "Classify this Brazilian transaction: '{description}' of R${amount}.
    Categories: Alimentacao, Transporte, Moradia, Saude, Lazer, Educacao,
    Compras, Servicos, Investimentos, Outros.
    Return JSON: {category, subcategory, confidence}"
  |
  v
[JSON Parse] Extract category, subcategory, confidence
  |
  v
[Supabase Module] UPDATE transactions
  SET category = ..., subcategory = ..., ai_confidence = ..., classified_at = NOW()
  WHERE id = transaction_id
  |
  v
[Router] Check if budget exceeded
  |-- YES --> [Supabase] Read budget limit
  |           [Supabase] Sum month spending in category
  |           [Z-API] Send WhatsApp alert
  |-- NO  --> End
```

#### Scenario 2: Monthly Insight Generator (Scheduled)

```
Trigger: Schedule (1st of each month, 9:00 AM BRT)
  |
  v
[Supabase] SELECT DISTINCT user_id FROM transactions
  WHERE date >= first_day_of_previous_month
  |
  v
[Iterator] For each user_id:
  |
  v
  [Supabase] SELECT category, SUM(amount), COUNT(*)
    FROM transactions WHERE user_id = ... AND date IN previous_month
    GROUP BY category
  |
  v
  [Supabase] SELECT * FROM insights
    WHERE user_id = ... AND period = two_months_ago (for comparison)
  |
  v
  [HTTP Module] Call OpenAI API
    - Prompt: "You are a friendly financial coach speaking Portuguese.
      Analyze this spending data: {data}.
      Previous month comparison: {previous}.
      Generate 3-5 actionable insights. Be specific with amounts in R$.
      Tone: encouraging but honest, like a friend who's good with money."
  |
  v
  [Supabase] INSERT INTO insights (user_id, period, content, data, ...)
  |
  v
  [Supabase] SELECT whatsapp_number FROM profiles WHERE id = user_id
  |
  v
  [Z-API] Send WhatsApp message with insights summary
  |
  v
  [Supabase] INSERT INTO whatsapp_messages (delivery tracking)
```

#### Scenario 3: Goal Reminder (Scheduled)

```
Trigger: Schedule (Every day at 8:00 AM BRT)
  |
  v
[Supabase] SELECT goals.*, profiles.whatsapp_number
  FROM goals JOIN profiles ON goals.user_id = profiles.id
  WHERE is_active = true
  AND should_remind_today(reminder_frequency, reminder_day) -- DB function
  |
  v
[Iterator] For each goal:
  |
  v
  [HTTP Module] Call OpenAI API
    - Prompt: "Generate a short, motivating WhatsApp reminder in Portuguese.
      Goal: '{name}', Progress: R${current} of R${target} ({percentage}%).
      Deadline: {target_date}. Be creative, use emojis, max 3 lines."
  |
  v
  [Z-API] Send WhatsApp message
  |
  v
  [Supabase] Log message in whatsapp_messages
```

#### Scenario 4: Simulated Data Generator (Webhook-triggered, MVP only)

```
Trigger: Custom Webhook (called from Streamlit "Generate Sample Data" button)
  |
  v
[HTTP Module] Call OpenAI API
  - Prompt: "Generate 30 realistic Brazilian financial transactions for one month.
    Include: date, description (real Brazilian merchant names like iFood, Uber,
    Carrefour, Drogaria Sao Paulo), amount in BRL, type (expense/income).
    Mix of categories. Return as JSON array."
  |
  v
[JSON Parse] Parse transaction array
  |
  v
[Iterator] For each transaction:
  [Supabase] INSERT INTO transactions (with source = 'simulated')
  |
  v
[Webhook Response] Return success + count to Streamlit
```

### Component 4: External Services

#### OpenAI API
| Aspect | Detail |
|--------|--------|
| **Called by** | Make.com only (never Streamlit) |
| **Model** | gpt-4o-mini for classification (cheaper, fast), gpt-4o for insight generation (better quality) |
| **Prompt strategy** | JSON mode for classification, free-text for insights |
| **Token budget** | Classification: ~200 tokens out. Insights: ~500 tokens out |

#### Z-API (WhatsApp)
| Aspect | Detail |
|--------|--------|
| **Called by** | Make.com only |
| **Message types** | Budget alerts (immediate), insights (monthly), goal reminders (scheduled) |
| **Rate limiting** | Z-API has sending limits; batch messages with delays between sends |
| **Instance** | Requires a connected WhatsApp number on Z-API dashboard |

## Data Flow

### Flow 1: User Views Dashboard (Read Path)

```
User opens Streamlit
  --> st.session_state checks for auth token
  --> If no token: show login form
  --> If token: create authenticated Supabase client
  --> Query Supabase directly (RLS filters to user's data)
  --> Render charts with Plotly
  --> Cache with @st.cache_data(ttl=300)
```

**No Make.com involvement.** Reads are direct Streamlit-to-Supabase. This is fast and simple.

### Flow 2: New Transaction Needs Classification (Write + AI Path)

```
Simulated data arrives in Supabase (via Make.com Scenario 4)
  OR future: Open Finance data arrives
  --> Transactions exist in Supabase WITHOUT category
  --> Make.com Scenario 1 triggered (webhook or Supabase trigger)
  --> OpenAI classifies each transaction
  --> Make.com writes category back to Supabase
  --> Next time user opens dashboard, classified data is visible
```

**Asynchronous pattern.** User does NOT wait for classification. Data appears unclassified first, then gets classified in the background. Dashboard shows a "pending classification" indicator if `category IS NULL`.

### Flow 3: Budget Alert (Event-Driven Path)

```
Make.com classifies a transaction (Flow 2)
  --> After writing category, checks budget
  --> Queries: SUM(amount) WHERE category = X AND month = current
  --> Compares to budget.monthly_limit
  --> If exceeded: sends WhatsApp alert via Z-API
  --> Logs the alert in whatsapp_messages
```

**Piggybacks on classification.** Budget checks happen as part of the classification scenario, not as a separate scheduled job. This gives near-real-time alerts.

### Flow 4: Monthly Insights (Scheduled Path)

```
1st of month, 9:00 AM BRT
  --> Make.com scheduled scenario triggers
  --> Reads all users' previous month data from Supabase
  --> Sends spending summary to OpenAI for analysis
  --> Writes insights to Supabase
  --> Sends WhatsApp summary to each user
  --> User can also view insights in Streamlit dashboard
```

### Flow 5: Goal Reminder (Scheduled Path)

```
Daily at 8:00 AM BRT
  --> Make.com checks which goals need reminders today
  --> For each: generates personalized message via OpenAI
  --> Sends via Z-API WhatsApp
  --> Logs delivery
```

## Patterns to Follow

### Pattern 1: Webhook Request/Response for Streamlit-to-Make.com

**What:** When Streamlit needs to trigger a Make.com action (e.g., "generate sample data"), use a synchronous webhook with response.

**When:** User clicks a button that requires Make.com processing.

**Example:**

```python
# In Streamlit
import requests

def trigger_data_generation(user_id: str, month: str):
    webhook_url = st.secrets["MAKE_WEBHOOK_GENERATE_DATA"]
    response = requests.post(
        webhook_url,
        json={
            "user_id": user_id,
            "month": month,
            "transaction_count": 30
        },
        timeout=30  # Make.com scenarios can take time
    )
    if response.status_code == 200:
        st.success("Dados gerados com sucesso!")
        st.cache_data.clear()  # Force dashboard refresh
    else:
        st.error("Erro ao gerar dados. Tente novamente.")
```

**Important:** Make.com webhook responses have a ~40 second timeout. For long operations, use a fire-and-forget webhook (no response) and poll Supabase for completion.

### Pattern 2: Service Role vs Anon Key Separation

**What:** Strict separation between Supabase keys used by Streamlit (anon + user JWT) and Make.com (service_role).

**When:** Always. This is a security boundary.

```python
# Streamlit: uses anon key + user's JWT (RLS enforced)
supabase = create_client(SUPABASE_URL, SUPABASE_ANON_KEY)
supabase.auth.sign_in_with_password({"email": email, "password": password})
# All queries now filtered by RLS

# Make.com: uses service_role key (bypasses RLS)
# Configured in Make.com's Supabase module connection settings
# Can read/write any user's data (needed for batch processing)
```

### Pattern 3: Idempotent Webhook Design

**What:** Make.com webhooks should handle duplicate calls gracefully.

**When:** Always. Network retries and Streamlit re-runs can cause duplicate webhook calls.

```
# In Make.com scenario:
# Before processing, check if already processed
[Supabase] SELECT COUNT(*) FROM transactions
  WHERE user_id = X AND source = 'simulated' AND
  date_trunc('month', date) = target_month

# If count > 0: skip (already generated)
# If count = 0: proceed with generation
```

### Pattern 4: Streamlit Caching Strategy

**What:** Use Streamlit's caching decorators to avoid redundant Supabase queries.

**When:** Every Supabase read in Streamlit.

```python
@st.cache_data(ttl=300)  # Cache for 5 minutes
def get_monthly_spending(user_id: str, month: str):
    response = supabase.table("transactions") \
        .select("category, amount") \
        .eq("user_id", user_id) \
        .gte("date", f"{month}-01") \
        .lt("date", f"{next_month}-01") \
        .execute()
    return response.data

# Clear cache when user triggers data changes
if st.button("Atualizar dados"):
    st.cache_data.clear()
    st.rerun()
```

### Pattern 5: OpenAI Prompt with JSON Mode for Classification

**What:** Use structured output (JSON mode) for transaction classification to get reliable, parseable results.

**When:** Every classification call.

```
# Make.com HTTP module to OpenAI:
POST https://api.openai.com/v1/chat/completions
{
  "model": "gpt-4o-mini",
  "response_format": { "type": "json_object" },
  "messages": [
    {
      "role": "system",
      "content": "You classify Brazilian financial transactions. Always respond in JSON with keys: category, subcategory, confidence. Categories: Alimentacao, Transporte, Moradia, Saude, Lazer, Educacao, Compras, Servicos, Investimentos, Salario, Outros."
    },
    {
      "role": "user",
      "content": "Classify: 'IFOOD *RESTAURANTE SUSHI' R$87.50 expense"
    }
  ]
}

# Expected response:
{"category": "Alimentacao", "subcategory": "Delivery", "confidence": 0.95}
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Streamlit Calling OpenAI Directly

**What:** Putting OpenAI API calls in Streamlit code.

**Why bad:** Streamlit re-runs the entire script on every widget interaction. An OpenAI call in the main flow would fire on every click, slider change, or page navigation. This wastes tokens, adds latency, and makes costs unpredictable.

**Instead:** All AI calls go through Make.com. Streamlit only reads AI results from Supabase.

### Anti-Pattern 2: Storing Secrets in Streamlit Code

**What:** Hardcoding API keys, Supabase URLs, or webhook URLs in Python files.

**Why bad:** Security risk, especially if code is in a Git repo.

**Instead:** Use `st.secrets` (Streamlit's built-in secrets management) via `.streamlit/secrets.toml` locally, and Streamlit Cloud's secrets UI for deployment. Add `.streamlit/secrets.toml` to `.gitignore`.

### Anti-Pattern 3: One Giant Make.com Scenario

**What:** Putting all logic (classification + insights + WhatsApp + data generation) in a single Make.com scenario.

**Why bad:** Make.com scenarios have execution limits, become hard to debug, and cannot be independently scheduled or triggered.

**Instead:** Separate scenarios by trigger type and responsibility (see the 4 scenarios above). Use webhooks for on-demand actions, schedules for periodic jobs.

### Anti-Pattern 4: Synchronous Classification on Transaction Insert

**What:** Requiring transactions to be classified before they appear in the dashboard.

**Why bad:** Adds latency to data ingestion. If OpenAI is slow or down, no data appears.

**Instead:** Insert transactions immediately (unclassified), classify asynchronously via Make.com, display with a "processing" indicator in the UI.

### Anti-Pattern 5: No RLS on Supabase Tables

**What:** Relying on application-level filtering (`WHERE user_id = X`) without enabling PostgreSQL Row Level Security.

**Why bad:** If RLS is not enabled, any authenticated user could potentially access other users' data by manipulating queries. With 5-10 MVP users this seems low-risk, but it is a fundamental security flaw that becomes catastrophic at scale.

**Instead:** Enable RLS on EVERY table with user_id. Test by logging in as different users and verifying isolation.

## Scalability Considerations

| Concern | At 5-10 Users (MVP) | At 100 Users | At 1,000+ Users |
|---------|---------------------|--------------|-----------------|
| **Supabase free tier** | Sufficient (500MB DB, 50K monthly auth requests) | Still OK if data is modest | Likely need Pro plan ($25/mo) |
| **Make.com operations** | Free plan has 1,000 ops/month -- tight with 30 tx/user/month | Need paid plan (~$9/mo for 10K ops) | Need Team plan or custom backend |
| **OpenAI costs** | ~$0.50/month (gpt-4o-mini classification) | ~$5/month | Consider batch API or fine-tuned model |
| **Streamlit Cloud** | Free tier (1 app, public) | Still free if single app | Consider Streamlit Community Cloud limits |
| **Z-API** | Basic plan sufficient | Check message volume limits | May need Business plan |
| **Make.com bottleneck** | Not an issue | Scenario execution time limits | Make.com becomes the limiting factor; consider migrating orchestration to Supabase Edge Functions or a Python backend |

**Key scaling inflection point:** At ~100+ users, Make.com's operation limits and execution time constraints become the bottleneck. The migration path is: Make.com scenarios --> Supabase Edge Functions (Deno/TypeScript) or a FastAPI backend. Design the Supabase schema now so this migration is non-breaking.

## Deployment Architecture

```
+---------------------------+
|    Streamlit Cloud         |
|    (Free Community)        |
|    - Hosts app.py          |
|    - Manages secrets       |
|    - Auto-deploys from Git |
+---------------------------+
           |
           v
+---------------------------+     +---------------------------+
|    Supabase (Free tier)   |     |    Make.com (Free/Paid)   |
|    - Region: South America|     |    - 4 scenarios          |
|    - PostgreSQL + Auth    |     |    - Webhook endpoints    |
|    - RLS enabled          |     |    - Scheduled triggers   |
+---------------------------+     +---------------------------+
                                         |          |
                                         v          v
                                  +---------+  +---------+
                                  | OpenAI  |  |  Z-API  |
                                  |   API   |  | WhatsApp|
                                  +---------+  +---------+
```

**Deployment notes:**
- Streamlit Cloud deploys from GitHub repo (push to main = auto-deploy)
- Supabase project created via dashboard (schema managed via SQL migrations in repo)
- Make.com scenarios configured via UI (export as JSON blueprint for version control)
- Secrets NEVER in the repo; managed per-platform

## Suggested Build Order (Dependencies)

The architecture has clear dependency chains that dictate build order:

```
Phase 1: Foundation (no dependencies)
  [Supabase Setup] --> Schema, RLS, Auth
  [Streamlit Skeleton] --> App structure, auth flow, Supabase connection
  Result: User can log in and see empty dashboard

Phase 2: Data Layer (depends on Phase 1)
  [Make.com Scenario 4] --> Simulated data generator
  [Streamlit Dashboard] --> Read and display transactions
  Result: User can generate fake data and see it in charts

Phase 3: Intelligence (depends on Phase 2)
  [Make.com Scenario 1] --> Transaction classifier
  [Streamlit Categories] --> Show classified transactions
  Result: Transactions get AI-classified categories

Phase 4: Budgets & Alerts (depends on Phase 3)
  [Supabase Budget Tables] --> Budget schema
  [Streamlit Budget UI] --> Set and track budgets
  [Make.com Budget Check] --> Alert logic in Scenario 1
  [Z-API Integration] --> WhatsApp alerts
  Result: Budget tracking with WhatsApp alerts

Phase 5: Insights & Goals (depends on Phase 3)
  [Make.com Scenario 2] --> Monthly insight generator
  [Make.com Scenario 3] --> Goal reminders
  [Streamlit Insights Page] --> Display insights
  [Streamlit Goals Page] --> Manage goals
  Result: Full AI coaching experience

Phase 6: Polish (depends on Phase 5)
  [UI/UX refinement] --> Demo-quality visuals
  [Error handling] --> Graceful failures
  [Onboarding flow] --> First-time user experience
  Result: Investor-ready demo
```

**Critical dependency:** Supabase schema and auth MUST be first. Everything else reads from or writes to Supabase. Getting the schema right early prevents painful migrations later.

**Parallel opportunities:**
- Phase 1: Supabase setup and Streamlit skeleton can be built in parallel
- Phase 5: Insights and Goals are independent of each other
- Phase 4 and 5: Can be developed in parallel if Phase 3 is complete

## Security Architecture

| Layer | Mechanism | Notes |
|-------|-----------|-------|
| **Auth** | Supabase Auth (email/password) | JWT-based, handles session refresh |
| **Data isolation** | PostgreSQL RLS | Every table, every operation |
| **Streamlit secrets** | `.streamlit/secrets.toml` + Cloud secrets UI | Never in code |
| **Make.com access** | Service role key | Bypasses RLS intentionally |
| **API keys** | Stored in Make.com connections | Never exposed to frontend |
| **WhatsApp** | Z-API instance with connected number | Separate from personal WhatsApp |

**The Streamlit frontend NEVER holds the Supabase service_role key, OpenAI key, or Z-API key.** It only has the Supabase anon key and webhook URLs.

## Sources

- Streamlit official documentation (caching, session state, secrets management patterns) -- MEDIUM confidence (from training data, not live-verified)
- Supabase official documentation (RLS, Auth, Python client) -- MEDIUM confidence
- Make.com webhook and Supabase module documentation -- MEDIUM confidence
- OpenAI API documentation (JSON mode, chat completions) -- HIGH confidence (well-established API)
- Z-API documentation patterns -- LOW confidence (niche Brazilian API, limited training data)
- Architecture patterns derived from established multi-service integration patterns -- MEDIUM confidence
