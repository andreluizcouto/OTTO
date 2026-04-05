# Phase 1: Foundation - Context

**Gathered:** 2026-04-05
**Status:** Ready for planning

<domain>
## Phase Boundary

Supabase Auth (cadastro, login, sessão persistente, logout), schema do banco PostgreSQL com RLS, e esqueleto do app Streamlit com navegação. Usuários podem criar conta, acessar com segurança, e ver apenas seus próprios dados numa estrutura de app funcional.

</domain>

<decisions>
## Implementation Decisions

### Estrutura do app
- **D-01:** Navegação principal via sidebar com ícones usando streamlit-option-menu — menu lateral com ícones para cada seção (Dashboard, Transações, Orçamentos, Metas, Config)
- **D-02:** Esqueleto mínimo funcional na Phase 1 — apenas Dashboard (placeholder) e página de Configurações. Demais páginas (Transações, Orçamentos, Metas) chegam nas fases seguintes conforme suas funcionalidades são implementadas
- **D-03:** Tema visual dark mode do Streamlit — visual moderno, combina com apps financeiros

### Detalhes do schema
- **D-04:** Tabela transactions com campos extras além do mínimo: merchant_name (nome real da loja), payment_method (crédito/débito/pix), is_recurring (gasto fixo), notes (observação do usuário) — além dos obrigatórios amount, date, description, category, confidence_score, user_id
- **D-05:** Tabela goals completa: name, target_amount, current_amount, deadline, status (active/completed/cancelled), user_id, created_at
- **D-06:** RLS strict por usuário em todas as tabelas — políticas separadas para SELECT, INSERT, UPDATE, DELETE. Cada usuário só vê e edita seus próprios dados

### Taxonomia de categorias
- **D-07:** Categorias pré-populadas com taxonomia brasileira: Alimentação, Transporte, Moradia, Saúde, Lazer, Educação, Compras, Assinaturas, Delivery, Outros — usuário pode customizar depois (Phase 3)
- **D-08:** Cada categoria com cor hex (para gráficos Plotly) e emoji associado — visual rico no dashboard desde o início

### Claude's Discretion
- Design da tela de login/cadastro (layout, mensagens de erro, fluxo após autenticação)
- Loading states e skeleton screens durante carregamento
- Tipos de dados exatos e constraints do PostgreSQL (varchar lengths, check constraints, etc.)
- Estrutura exata da tabela budgets (será usada na Phase 4)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

No external specs — requirements fully captured in decisions above and in:

### Project requirements
- `.planning/REQUIREMENTS.md` — AUTH-01 through AUTH-04, DATA-01, DATA-03 requirements for this phase
- `.planning/PROJECT.md` — Constraints (tech stack, Supabase experience level, demo quality)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Nenhum código existente — projeto greenfield. Apenas CLAUDE.md e artefatos de planejamento.

### Established Patterns
- Tech stack definido em CLAUDE.md: Python 3.11, Streamlit >=1.56.0, supabase-py >=2.28.3, streamlit-option-menu >=0.4.0, plotly >=6.6.0, pandas >=3.0.2
- Pattern de auth: supabase-py `client.auth.sign_in_with_password()` + `st.session_state` para sessão
- Pattern de query: `client.table("x").select("*").eq("user_id", uid).execute()`
- Env vars: python-dotenv para dev local, st.secrets para produção

### Integration Points
- Streamlit app (main entry point) → Supabase Auth (sign up/in/out)
- Streamlit pages → Supabase PostgREST queries (via supabase-py)
- RLS policies filtram automaticamente por `auth.uid()`

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation*
*Context gathered: 2026-04-05*
