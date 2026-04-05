# Phase 1: Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-05
**Phase:** 01-foundation
**Areas discussed:** Estrutura do app, Detalhes do schema, Taxonomia de categorias

---

## Estrutura do app

### Navegação principal

| Option | Description | Selected |
|--------|-------------|----------|
| Sidebar com ícones | Menu lateral com streamlit-option-menu — visual profissional, ícones para cada seção | :heavy_check_mark: |
| Tabs no topo | Abas horizontais nativas do Streamlit (st.tabs) — mais simples, sem dependência extra | |
| Multipage nativo | Streamlit multipage (pages/) — cada seção é um arquivo separado, sidebar automática | |

**User's choice:** Sidebar com ícones (Recomendado)
**Notes:** Nenhuma nota adicional

### Páginas do esqueleto

| Option | Description | Selected |
|--------|-------------|----------|
| Mínimo funcional | Dashboard (placeholder) + Configurações — só o que a Phase 1 precisa | :heavy_check_mark: |
| Esqueleto completo | Dashboard + Transações + Orçamentos + Metas + Config — todas com placeholder | |
| Só Dashboard | Apenas a página principal de Dashboard | |

**User's choice:** Mínimo funcional (Recomendado)
**Notes:** Nenhuma nota adicional

### Tema visual

| Option | Description | Selected |
|--------|-------------|----------|
| Dark mode | Tema escuro do Streamlit — visual moderno, combina com apps financeiros | :heavy_check_mark: |
| Light mode | Tema claro padrão do Streamlit — mais limpo, melhor para apresentações | |
| Escolha do usuário | Toggle dark/light nas configurações | |
| Você decide | Claude escolhe o melhor para demo | |

**User's choice:** Dark mode (Recomendado)
**Notes:** Nenhuma nota adicional

---

## Detalhes do schema

### Campos da tabela transactions

| Option | Description | Selected |
|--------|-------------|----------|
| Campos práticos extras | merchant_name, payment_method, is_recurring, notes além do mínimo | :heavy_check_mark: |
| Mínimo dos requirements | Só amount, date, description, category, confidence_score, user_id | |
| Você decide | Claude define os campos para o MVP | |

**User's choice:** Campos práticos extras (Recomendado)
**Notes:** Nenhuma nota adicional

### Estrutura da tabela goals

| Option | Description | Selected |
|--------|-------------|----------|
| Completa | name, target_amount, current_amount, deadline, status, user_id, created_at | :heavy_check_mark: |
| Mínima | name, target_amount, deadline, user_id — current_amount calculado | |
| Você decide | Claude define a estrutura ideal | |

**User's choice:** Completa (Recomendado)
**Notes:** Nenhuma nota adicional

### Políticas RLS

| Option | Description | Selected |
|--------|-------------|----------|
| Strict por usuário | RLS ativado, políticas separadas para SELECT, INSERT, UPDATE, DELETE | :heavy_check_mark: |
| RLS básico | RLS só no SELECT, INSERT/UPDATE sem restrição | |
| Você decide | Claude define o nível de segurança | |

**User's choice:** Strict por usuário (Recomendado)
**Notes:** Nenhuma nota adicional

---

## Taxonomia de categorias

### Categorias pré-definidas

| Option | Description | Selected |
|--------|-------------|----------|
| Pré-populadas BR | Seed com categorias brasileiras: Alimentação, Transporte, Moradia, etc. | :heavy_check_mark: |
| Vazio | Sem categorias pré-definidas — IA cria conforme classifica | |
| Poucas genéricas | 5 categorias básicas (Essencial, Lazer, Transporte, Moradia, Outros) | |

**User's choice:** Pré-populadas BR (Recomendado)
**Notes:** Nenhuma nota adicional

### Ícones e cores

| Option | Description | Selected |
|--------|-------------|----------|
| Cor + emoji | Cada categoria com cor hex + emoji — visual rico no dashboard | :heavy_check_mark: |
| Só cor | Apenas cor hex por categoria — suficiente para gráficos | |
| Você decide | Claude define o melhor para o dashboard | |

**User's choice:** Cor + emoji (Recomendado)
**Notes:** Nenhuma nota adicional

---

## Claude's Discretion

- Experiência de login/cadastro (design da tela, fluxo de auth) — usuário não selecionou esta área para discussão
- Loading states e error handling
- Tipos de dados exatos e constraints do PostgreSQL
- Estrutura da tabela budgets

## Deferred Ideas

None — discussion stayed within phase scope
