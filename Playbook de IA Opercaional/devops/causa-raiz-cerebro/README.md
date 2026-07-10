# Causa-Raiz do Cerebro

## Objetivo

Investigar degradação de busca/indexação no Cerebro e chegar à causa raiz sem confundir sintomas com origem.

## Casos de uso

- Busca lenta e resultados parciais.
- Heap alto e circuit breaker acionando.
- Reindexação travada ou competindo com queries.

## Exemplo

Entrada resumida:

```text
configuracao_cluster={{configuracao_cluster}}
metricas={{metricas}}
logs={{logs}}
```

Saída esperada em alto nível:

```text
Resumo executivo: a reindexação prolongada está saturando heap e filas de escrita, o que dispara circuit breaker e degrada a busca.
```

Saída esperada em mais detalhe:

```text
1. Resumo executivo: a degradação vem da reindexação agendada que não terminou e está disputando heap com consultas.
2. Causa-raiz provável: reindex travada/longa demais, levando a saturação de heap, throttling e circuit breaker.
3. Cadeia causal: reindex -> heap alto -> filas de escrita -> cache cai -> busca fica lenta e parcial.
4. Ação imediata: pausar ou reagendar a reindexação e aliviar a pressão de memória.
```

## Limitações

- Não substitui análise com acesso ao cluster.
- Depende de janela temporal coerente entre métricas e logs.
- Se os artefatos vierem incompletos, o modelo deve explicitar a incerteza.

## Curadoria

Curado para raciocínio causal com separação explícita entre evidência direta e inferência. Esse prompt pede síntese longa e justificável, então privilegia consistência sobre concisão extrema.