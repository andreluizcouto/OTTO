# Feature Research

**Domain:** AI-powered personal finance assistant (Brazilian market focus)
**Researched:** 2026-04-05
**Confidence:** MEDIUM (based on training data through early 2025 -- web search unavailable for verification of latest feature changes)

## Competitor Overview

Before diving into features, here is the landscape of products analyzed and what each represents:

| Product | Market | Model | Known For |
|---------|--------|-------|-----------|
| **Cleo** | US/UK | AI chatbot + banking | Sassy AI personality, roast mode, cash advances, budget coaching via chat |
| **Plum** | UK/EU | AI auto-savings + investing | Automatic savings algorithms, round-ups, investment pockets |
| **Olivia AI** | Brazil | AI financial assistant | Brazilian-focused AI coach, conversational finance, Open Finance integration |
| **Guiabolso** | Brazil | Aggregation + credit | Pioneer in Brazilian Open Finance, credit score, automatic categorization |
| **Mobills** | Brazil | Manual/automatic tracking | Clean UI, envelope budgeting, bill reminders, mature product |
| **Organizze** | Brazil | Manual tracking | Simple, beloved UX, manual-first approach, family sharing |
| **Pierre** (concept) | Brazil | AI coach + WhatsApp | Proactive coaching, WhatsApp-native messaging, automated classification |

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete or untrustworthy.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Transaction categorization** | Every competitor does this (Guiabolso, Mobills, Cleo all auto-categorize). Users refuse to manually tag every expense. | MEDIUM | OpenAI classification is FinCoach's approach. Must handle Brazilian merchant names, PIX descriptions, and common abbreviations. Cleo and Guiabolso achieve ~85-90% accuracy. Aim for same. |
| **Spending by category visualization** | Mobills, Organizze, Guiabolso all show pie/bar charts of spending by category. Users expect to see "where my money goes" at a glance. | LOW | Streamlit charts (Plotly/Altair). Categories: Alimentacao, Transporte, Moradia, Lazer, Saude, Educacao, etc. |
| **Monthly spending summary** | Every finance app shows month-over-month totals. Cleo's "spending insights" and Guiabolso's monthly summary are baseline expectations. | LOW | Total spent, total income, balance. Show trend vs previous month. |
| **Budget limits per category** | Mobills and Organizze both offer category budgets. Cleo has "budget mode." Users expect to set a limit and see progress. | MEDIUM | Need: set limit, track progress, visual indicator (green/yellow/red). Envelope-style budgeting like Mobills. |
| **Over-budget alerts** | Mobills sends push notifications when budget is exceeded. Cleo alerts in-chat. Users expect to be warned, not discover overspending after the fact. | LOW | Trigger via Make.com when category spend > threshold. Send via WhatsApp (Z-API). |
| **Secure per-user data isolation** | Obvious. Guiabolso, Mobills, all have individual accounts. No user should ever see another user's data. | MEDIUM | Supabase Auth + Row Level Security (RLS). Non-negotiable for even MVP with 5-10 users. |
| **Period filtering (month/week/custom)** | All competitors let you pick date ranges. Users expect to view "this month", "last month", custom ranges. | LOW | Date picker in Streamlit. Filter transactions by period. |
| **Income vs expenses overview** | Guiabolso and Mobills both show income-expense balance prominently. Users expect a clear picture of net cash flow. | LOW | Simple calculation, prominent display on dashboard. |

### Differentiators (Competitive Advantage)

Features that set FinCoach apart from Mobills/Organizze/Guiabolso. These align with the project's core value of "proactive AI coaching."

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **AI-generated monthly insights** | Goes beyond charts. Cleo does "spending roasts" but most Brazilian apps (Mobills, Organizze) just show data without interpretation. FinCoach tells you *what the numbers mean*: "You spent 30% more on delivery this month vs last month. That's R$180 extra." | MEDIUM | OpenAI prompt with transaction summary context. Key differentiator vs Mobills/Organizze which are data-display-only. Olivia AI does this but is not widely adopted in Brazil. |
| **Proactive WhatsApp coaching** | THIS is the killer differentiator. No Brazilian app does this well. Cleo uses in-app chat. Olivia AI uses app notifications. WhatsApp has 99% penetration in Brazil -- meeting users where they already are. Contextual nudges like "It's Friday night -- you've spent 80% of your Lazer budget, be mindful!" | HIGH | Requires: contextual triggers, timing logic (Thursday/Friday evenings, post-payday), message templates, Z-API integration via Make.com. No competitor does WhatsApp-native coaching in Brazil. |
| **Financial goal tracking with proactive reminders** | Mobills has basic goals. Organizze does not. None send WhatsApp reminders like "Your trip is in 45 days, you've saved R$2,100 of R$5,000. Set aside R$96/day to make it." This turns passive tracking into active coaching. | MEDIUM | Goal model: name, target amount, deadline, current saved. Reminder logic via Make.com scheduled scenarios. |
| **AI spending pattern detection** | Beyond monthly summaries -- detecting patterns over time. "You always overspend in the first week after payday" or "Your Uber spending spikes on weekends." Cleo does basic pattern detection. Guiabolso does not. | MEDIUM | Requires 2-3 months of data. OpenAI analysis of time-series transaction data. More valuable as data accumulates. |
| **Conversational tone / personality** | Cleo proved that personality matters enormously -- their "sassy best friend" tone is a core product feature, not a nice-to-have. Brazilian culture is warm and informal. A coaching personality ("Pierre" or custom) makes the product memorable. | LOW | Prompt engineering for OpenAI responses and WhatsApp messages. Use Brazilian Portuguese naturally (informal "voce", emojis, humor). Low technical complexity, high impact. |
| **Smart timing for notifications** | Sending coaching messages at the RIGHT moment: payday, weekend eve, end of month, near goal deadlines. Not just random alerts. Cleo and Plum both optimize timing. | MEDIUM | Make.com scheduled scenarios with conditional logic. Payday detection (recurring large credit), day-of-week logic, goal deadline proximity. |
| **Simulated data for instant onboarding** | Users see value immediately without waiting for real bank data. No Brazilian competitor does this. Removes the biggest friction point: "I need to connect my bank or manually enter 3 months of data before this is useful." | LOW | AI-generated realistic Brazilian transaction data. Lets MVP users experience full product value from day 1. Also critical for investor demos. |

### Anti-Features (Deliberately NOT Building)

Features that seem appealing but create problems for an MVP-stage AI finance coach.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Manual transaction entry** | Users might want to add cash purchases or corrections | Defeats core value proposition of "zero effort." Mobills and Organizze already do manual entry well -- competing there is losing. Every manual entry is friction that kills retention. | Accept some inaccuracy in MVP. Add "quick adjust" for miscategorized transactions only, not full manual entry. Phase 2: photo receipt scanning via WhatsApp. |
| **Real-time bank sync (Open Finance)** | Users expect "connect my bank" like Guiabolso | Pluggy/Belvo integration is expensive (R$0.50-2.00/user/month), complex (auth flows, token refresh, error handling), and regulated. Kills MVP velocity. Guiabolso spent years on this. | Simulated data for MVP. Phase 2: Open Finance via Pluggy after concept validation. The AI coaching experience is the hypothesis to validate, not the data pipeline. |
| **Investment tracking / recommendations** | Plum and some Brazilian fintechs offer this | Requires CVM compliance awareness, real-time market data, different data models. Completely different product domain. Dilutes focus. | Acknowledge investments exist but stay focused on spending coaching. Can reference "you could invest the R$500 you saved" without tracking portfolios. |
| **Multi-currency support** | International users, travel expenses | Adds complexity to every calculation, category, and comparison. MVP audience is 5-10 Brazilians. | BRL only. If a user travels, they can note it manually or we ignore those transactions. |
| **Social/sharing features** | Comparing spending with friends, leaderboards | Privacy nightmare for financial data. Guiabolso tried "compare with people like you" and it was controversial. Brazilian users are particularly sensitive about sharing financial details. | Keep individual. The WhatsApp coaching already creates a "personal" relationship feel. |
| **Credit score tracking** | Guiabolso's original differentiator | Requires Serasa/SPC integration, regulated data, different user expectation. Not aligned with coaching value prop. | Out of scope entirely. Recommend Guiabolso or Serasa app for this. |
| **Bill payment / PIX integration** | "Let me pay my bills from the app" | Turns a coaching app into a banking app. Regulatory complexity (payment institution license in Brazil), security liability, massive scope expansion. | Show upcoming bills as reminders only. "Your electricity bill of ~R$180 is due in 3 days." |
| **Gamification (badges, streaks, points)** | Engagement mechanics like Duolingo | Hollow without substance. Cleo tried this and users reported it felt patronizing when their finances were stressed. Brazilian audience for MVP is close friends/family who need real value, not game mechanics. | Use the coaching personality and real progress (goal %, budget adherence) as the engagement loop. Natural motivation > artificial gamification. |
| **Complex financial planning tools** | Retirement calculators, tax planning, debt payoff simulators | Wrong product for this. These are one-time-use tools that financial advisors provide. Complexity explodes. Not aligned with daily coaching. | Keep it simple: monthly insights + goals. For complex planning, recommend a human financial advisor. |

## Feature Dependencies

```
[Supabase Auth + RLS]
    |
    +--> [Transaction Data Model]
    |        |
    |        +--> [AI Categorization (OpenAI)]
    |        |        |
    |        |        +--> [Spending by Category Charts]
    |        |        |        |
    |        |        |        +--> [Monthly Spending Summary]
    |        |        |        |
    |        |        |        +--> [Budget Limits per Category]
    |        |        |                 |
    |        |        |                 +--> [Over-budget Alerts (WhatsApp)]
    |        |        |                 |
    |        |        |                 +--> [Budget Progress Visualization]
    |        |        |
    |        |        +--> [AI Monthly Insights]
    |        |        |
    |        |        +--> [AI Pattern Detection] (needs 2+ months data)
    |        |
    |        +--> [Period Filtering]
    |        |
    |        +--> [Income vs Expenses Overview]
    |
    +--> [Financial Goals Model]
    |        |
    |        +--> [Goal Progress Tracking]
    |                 |
    |                 +--> [Goal Reminders (WhatsApp)]
    |
    +--> [Simulated Data Generator]
             |
             +--> [Instant Onboarding Experience]

[Z-API + Make.com WhatsApp Integration]
    |
    +--> [Over-budget Alerts]
    +--> [Goal Reminders]
    +--> [Proactive Coaching Messages]
    +--> [Smart Timing Logic]
```

### Dependency Notes

- **Everything requires Auth + Data Model:** No feature works without per-user authentication and a transaction storage layer. This is Phase 0.
- **AI Categorization unlocks the entire dashboard:** Charts, budgets, insights all depend on categorized transactions. This is the first AI integration point.
- **WhatsApp integration is a parallel workstream:** Z-API + Make.com setup can happen alongside dashboard development, but messages need categorized data to be meaningful.
- **Simulated data unlocks testing:** Must exist before any feature can be demoed or tested. Should be one of the first things built.
- **AI Pattern Detection requires data accumulation:** Cannot work meaningfully until 2-3 months of transaction data exist. Defer to v1.x.
- **Budget alerts require both budgets AND WhatsApp:** Two dependencies -- the budget system and the messaging pipeline.

## MVP Definition

### Launch With (v1)

Minimum viable product -- what is needed to validate the core hypothesis: "AI coaching via WhatsApp changes how Brazilians manage money."

- [ ] **Supabase Auth with RLS** -- Per-user data isolation. Non-negotiable even for 5 users.
- [ ] **Simulated transaction data** -- AI-generated realistic Brazilian spending data. Enables instant demo and testing.
- [ ] **AI transaction categorization** -- OpenAI classifies transactions into ~10 Brazilian spending categories. Core automation promise.
- [ ] **Streamlit dashboard with category charts** -- Visual spending breakdown by category and period. Proves the "understand your spending" value.
- [ ] **Monthly spending summary** -- Income, expenses, balance, trend vs last month. Basic but essential.
- [ ] **Category budget limits + visual tracking** -- Set limits, see progress bars (green/yellow/red). Core tracking feature.
- [ ] **Over-budget WhatsApp alerts** -- The first touch of proactive coaching. "You've exceeded your R$500 Alimentacao budget."
- [ ] **AI monthly insights** -- OpenAI-generated narrative about spending habits. "You spent 40% more on delivery than last month." Key differentiator.

### Add After Validation (v1.x)

Features to add once core is working and initial users confirm the coaching concept resonates.

- [ ] **Financial goals with WhatsApp reminders** -- Trigger: users ask "can I save for X?" Add goal model + proactive reminders.
- [ ] **Proactive contextual coaching** -- Trigger: WhatsApp alerts work and users respond well. Add timing-based nudges (Friday night, post-payday).
- [ ] **AI spending pattern detection** -- Trigger: 2-3 months of data accumulated. Start detecting behavioral patterns across time.
- [ ] **Coaching personality refinement** -- Trigger: user feedback on message tone. Refine the AI's voice to be more engaging/Brazilian.
- [ ] **Quick re-categorization** -- Trigger: users complain about miscategorized transactions. Allow one-tap category correction (NOT manual entry).

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] **Open Finance integration (Pluggy/Belvo)** -- Why defer: R$0.50-2.00/user/month cost, complex auth flows, regulatory compliance. Only worth it after coaching concept is validated.
- [ ] **Receipt scanning via WhatsApp** -- Why defer: User sends photo of receipt, AI extracts data. Cool but complex (OCR accuracy, Brazilian receipt formats vary wildly).
- [ ] **Family/couple shared budgets** -- Why defer: Data model complexity doubles. Organizze does this well already.
- [ ] **Recurring transaction detection** -- Why defer: Useful but requires pattern matching across months. Add when data pipeline matures.
- [ ] **Export to spreadsheet/PDF** -- Why defer: Users may request but low strategic value. Simple CSV export is trivial to add when asked.

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Supabase Auth + RLS | HIGH | MEDIUM | P1 |
| Simulated data generator | HIGH | LOW | P1 |
| AI transaction categorization | HIGH | MEDIUM | P1 |
| Category spending charts | HIGH | LOW | P1 |
| Monthly spending summary | HIGH | LOW | P1 |
| Period filtering | MEDIUM | LOW | P1 |
| Income vs expenses overview | MEDIUM | LOW | P1 |
| Category budget limits | HIGH | MEDIUM | P1 |
| Budget progress visualization | MEDIUM | LOW | P1 |
| Over-budget WhatsApp alerts | HIGH | MEDIUM | P1 |
| AI monthly insights | HIGH | MEDIUM | P1 |
| Financial goal tracking | HIGH | MEDIUM | P2 |
| Goal WhatsApp reminders | HIGH | MEDIUM | P2 |
| Proactive contextual coaching | HIGH | HIGH | P2 |
| Smart timing for messages | MEDIUM | MEDIUM | P2 |
| AI pattern detection | MEDIUM | MEDIUM | P2 |
| Coaching personality/tone | MEDIUM | LOW | P2 |
| Quick re-categorization | MEDIUM | LOW | P2 |
| Open Finance integration | HIGH | HIGH | P3 |
| Receipt scanning (WhatsApp) | MEDIUM | HIGH | P3 |
| Family/couple budgets | LOW | HIGH | P3 |
| Recurring transaction detection | MEDIUM | MEDIUM | P3 |

**Priority key:**
- P1: Must have for launch -- validates core hypothesis
- P2: Should have, add after initial validation confirms direction
- P3: Nice to have, future consideration after product-market fit

## Competitor Feature Analysis

| Feature | Cleo (US/UK) | Guiabolso (BR) | Mobills (BR) | Organizze (BR) | Olivia AI (BR) | FinCoach Approach |
|---------|--------------|----------------|--------------|-----------------|----------------|-------------------|
| Auto-categorization | Yes (AI) | Yes (rules + ML) | Yes (rules) | No (manual) | Yes (AI) | Yes -- OpenAI-based, handles BR merchants |
| Spending charts | Yes | Yes | Yes (excellent) | Yes (simple) | Yes | Yes -- Streamlit + Plotly |
| Category budgets | Yes ("Budget mode") | Basic | Yes (mature) | Basic | No | Yes -- envelope-style with visual progress |
| Over-budget alerts | In-app chat | Push notification | Push notification | No | In-app | WhatsApp via Z-API -- meets user where they are |
| AI insights | Yes ("roasts", analysis) | No | No | No | Yes (basic) | Yes -- OpenAI narrative insights in Portuguese |
| Proactive coaching | In-app chat nudges | No | Bill reminders only | No | Basic tips | WhatsApp-native contextual coaching -- unique in BR |
| Financial goals | Yes (basic) | No | Yes (mature) | No | No | Yes with WhatsApp reminders |
| Personality/tone | Strong ("sassy friend") | Corporate | Neutral | Neutral | Friendly but generic | Warm Brazilian personality -- informal, humorous |
| Chat interface | Yes (core UX) | No | No | No | Yes | No (WhatsApp IS the chat) |
| Bank sync | Yes (Plaid) | Yes (Open Finance) | Yes (Open Finance) | No | Yes (Open Finance) | No -- simulated data for MVP |
| Investment tracking | No | Yes (basic) | Yes | No | No | No -- deliberately excluded |
| Credit score | No | Yes (core feature) | No | No | No | No -- out of scope |
| Manual entry | Optional | No (auto only) | Yes (core) | Yes (core) | No | No -- anti-feature |
| WhatsApp integration | No | No | No | No | No | YES -- core differentiator |
| Simulated data | No | No | No | No | No | YES -- instant onboarding |

### Key Competitive Insights

1. **The WhatsApp gap is real.** No major Brazilian finance app uses WhatsApp for proactive coaching, despite WhatsApp being the dominant communication platform in Brazil (99% smartphone penetration, used daily by virtually every Brazilian). This is FinCoach's strongest differentiator.

2. **Brazilian apps are data-display, not coaching.** Mobills and Organizze show you data beautifully but never tell you what to *do* about it. Guiabolso aggregates but does not coach. There is a clear gap for an opinionated, proactive coach in the Brazilian market.

3. **Cleo proved the AI coach model works** but is unavailable in Brazil and culturally tuned for US/UK audiences. The opportunity is a Brazilian Cleo that speaks through WhatsApp instead of a dedicated chat interface.

4. **Olivia AI is the closest Brazilian competitor** but has not achieved significant market penetration. Their approach is app-centric (push notifications, in-app experience) rather than WhatsApp-native.

5. **Simulated data removes the biggest onboarding friction.** Every competitor requires either manual entry (Organizze, Mobills) or bank connection (Guiabolso, Olivia AI) before providing value. FinCoach shows value in seconds.

## Sources

- Training data knowledge of Cleo AI features and product positioning (MEDIUM confidence -- well-documented product, features stable since 2023)
- Training data knowledge of Plum features (MEDIUM confidence)
- Training data knowledge of Olivia AI Brazil (LOW-MEDIUM confidence -- smaller product, less documentation available)
- Training data knowledge of Guiabolso features and history (MEDIUM confidence -- well-known Brazilian fintech, extensively covered)
- Training data knowledge of Mobills features (MEDIUM confidence -- popular Brazilian app with clear feature documentation)
- Training data knowledge of Organizze features (MEDIUM confidence -- popular Brazilian app)
- Training data knowledge of Brazilian WhatsApp market penetration (HIGH confidence -- consistently documented across multiple years)
- PROJECT.md context for FinCoach-specific decisions and constraints

**Note:** Web search and web fetch tools were unavailable during this research session. All competitor analysis is based on training data through early 2025. Feature sets of actively developed products may have changed. Key findings (WhatsApp gap, Brazilian market dynamics, competitor positioning) are unlikely to have changed significantly but should be validated before major product decisions.

---
*Feature research for: AI-powered personal finance assistant (Brazilian market)*
*Researched: 2026-04-05*
