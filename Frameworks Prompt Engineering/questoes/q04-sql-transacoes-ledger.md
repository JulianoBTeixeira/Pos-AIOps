# Questão 04 - Relatório mensal de transações do Ledger

## Prompt

Crie uma query SQL PostgreSQL para consolidar transações mensais do Ledger.

Use as tabelas abaixo, filtre somente `status = 'completed'`, considere os últimos 6 meses corridos a partir de `2026-04-24`, agrupe por mês no formato `YYYY-MM` e por categoria, e retorne por linha a quantidade de transações e o volume total em reais com 2 casas decimais. O campo `amount_cents` está em centavos de real. As categorias existentes são `subscription`, `one_time`, `refund` e `credit_adjustment`. Ordene por mês crescente e categoria crescente.

Schema:

```sql
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
```

O objetivo é entregar uma SQL pronta para Jennifer usar na apresentação executiva, com colunas nomeadas claramente. Retorne apenas a query em um bloco `sql`.

## Modelo

GPT-4o. Escolhido por precisão em SQL PostgreSQL e boa obediência a formato curto.

## Output

```sql
SELECT
  TO_CHAR(DATE_TRUNC('month', created_at), 'YYYY-MM') AS mes,
  category AS categoria,
  COUNT(*) AS quantidade_transacoes,
  ROUND(SUM(amount_cents)::numeric / 100, 2) AS volume_total_reais
FROM transactions
WHERE status = 'completed'
  AND created_at >= DATE '2026-04-24' - INTERVAL '6 months'
  AND created_at < DATE '2026-04-24' + INTERVAL '1 day'
  AND category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
GROUP BY
  DATE_TRUNC('month', created_at),
  category
ORDER BY
  DATE_TRUNC('month', created_at) ASC,
  category ASC;
```

## Justificativa

Task aparece no pedido direto para criar uma query SQL PostgreSQL. Action aparece na lista de operações: filtrar, recortar período, agrupar, converter centavos para reais, nomear colunas e ordenar. Goal aparece ao dizer que a SQL deve estar pronta para Jennifer usar na apresentação executiva.
