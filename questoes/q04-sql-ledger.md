# Questão 04 — Relatório mensal de transações do Ledger

## Prompt

```markdown
# Task
Crie uma query SQL PostgreSQL para gerar o relatório mensal de transações concluídas do Ledger nos últimos 6 meses corridos a partir de 2026-04-24.

Esquema:

CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);

CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

Categorias em produção: subscription, one_time, refund e credit_adjustment.

# Action
A query deve:
- considerar apenas `status = 'completed'`;
- filtrar os últimos 6 meses corridos a partir de 2026-04-24;
- agrupar por mês no formato `YYYY-MM` e por categoria;
- trazer `transaction_count` e `total_amount_brl`;
- converter `amount_cents` para reais com 2 casas decimais;
- ordenar por mês crescente e categoria crescente;
- evitar função sobre coluna no `WHERE`, para favorecer uso de índice em `created_at`.

# Goal
Gerar uma query segura e pronta para Jennifer usar como base de relatório executivo de crescimento de transações por categoria, com saída legível e consistente para apresentação à diretoria.
```

## Modelo

**GPT-4o — OpenAI.** Escolhido por gerar SQL PostgreSQL com boa aderência a requisitos de formato, datas e agrupamento.

## Output

```sql
SELECT
    TO_CHAR(DATE_TRUNC('month', t.created_at), 'YYYY-MM') AS month,
    t.category,
    COUNT(*) AS transaction_count,
    ROUND((SUM(t.amount_cents)::numeric / 100), 2) AS total_amount_brl
FROM transactions t
WHERE
    t.status = 'completed'
    AND t.category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
    AND t.created_at >= TIMESTAMPTZ '2025-10-24 00:00:00+00'
    AND t.created_at <  TIMESTAMPTZ '2026-04-25 00:00:00+00'
GROUP BY
    DATE_TRUNC('month', t.created_at),
    t.category
ORDER BY
    DATE_TRUNC('month', t.created_at) ASC,
    t.category ASC;
```

### Observações

- O intervalo cobre os últimos 6 meses corridos considerando a data de referência `2026-04-24`, incluindo o próprio dia 24 até o fim do dia em UTC.
- O filtro em `created_at` usa comparação direta, sem aplicar função na coluna, preservando a possibilidade de uso do índice `idx_transactions_created_at`.
- A tabela `customers` não foi incluída porque nenhuma métrica solicitada depende de segmento, país ou data de cadastro.
- `amount_cents` é convertido para reais por `SUM(amount_cents)::numeric / 100` e arredondado para 2 casas decimais.

## Justificativa

A **Task** define a criação da query PostgreSQL para o relatório dos últimos 6 meses. A **Action** lista as regras de filtro, agrupamento, conversão monetária e otimização por índice. O **Goal** conecta a query ao objetivo de negócio da Jennifer: gerar base confiável para apresentação executiva.
