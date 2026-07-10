# Questão 08 - Postmortem técnico de incidente em produção

## Framework escolhido

R-I-S-E.

## Prompt

Você é uma pessoa staff SRE conduzindo análise técnica de incidente em produção durante pico de tráfego. Doc Brown precisa, em até 20 minutos, de um postmortem técnico objetivo para decidir entre rollback do deploy `chronos-api v2.48.0` e scaling emergencial de RDS/pool de conexões.

Input:

Deploy anterior:

```text
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s
```

Métricas do Beacon nos últimos 30 minutos:

```text
timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7
```

Log do pod:

```text
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error
```

Reactor:

```text
50.127 mensagens acumuladas, crescendo a ~800/min.
Consumer lag atual: 18 minutos e aumentando.
```

Cluster:

```text
Chronos: 12/12 pods running (HPA no máximo).
CPU médio dos pods: 62%.
Memória média dos pods: 71%.
Conexões ativas ao Ledger: 240/250 (limite do RDS).
```

Steps:

1. Reconstruir a linha do tempo do deploy e da degradação.
2. Separar sintomas, evidências e hipóteses.
3. Avaliar se o gargalo principal é compute do Chronos, pool de conexões, RDS ou mudança funcional do deploy.
4. Recomendar uma decisão imediata entre rollback e scaling emergencial.
5. Explicar riscos da decisão e próximos passos de estabilização.
6. Listar dados adicionais que devem ser coletados depois da estabilização.

Expectation: entregue um postmortem técnico em português, objetivo e acionável, com recomendação explícita. Use as seções: resumo executivo, impacto, linha do tempo, evidências, causa provável, decisão recomendada, riscos, próximos passos imediatos e ações pós-incidente. Não invente dados ausentes; quando fizer inferência, identifique como inferência.

## Modelo

Claude Sonnet 4. Escolhido por forte capacidade de análise causal, síntese executiva e comparação de hipóteses sob pressão.

## Output

## Resumo executivo

O incidente parece estar relacionado ao deploy `chronos-api v2.48.0`, especialmente à refatoração do cliente Ledger, ao novo endpoint batch, ao pool de conexões limitado a 20 por pod e à redução do timeout para 2s. A recomendação imediata é **rollback para v2.47.0**, não scaling emergencial como primeira ação.

Scaling de RDS pode aliviar temporariamente a saturação de conexões, mas os sinais apontam que a aplicação passou a pressionar o Ledger de forma diferente após o deploy. Como o HPA já está no máximo, CPU e memória não estão saturadas, e o RDS está em 240/250 conexões, aumentar capacidade sem reverter o comportamento novo pode apenas deslocar o gargalo.

## Impacto

- Latência p99 subiu de 420 ms para 8100 ms em 50 minutos.
- Taxa de erro subiu de 0,2% para 11,7%.
- Reactor acumula 50.127 mensagens e cresce cerca de 800/min.
- Consumer lag chegou a 18 minutos e continua aumentando.
- Circuit breaker do Ledger abriu com erro corrente de 87%.

## Linha do tempo

| Horário UTC | Evento |
|---|---|
| 2026-04-23 18:42:11 | Deploy `chronos-api v2.48.0` sincronizado via Argo CD |
| 2026-04-24 13:30 | p99 420 ms, erro 0,2%, tráfego 1200 req/s |
| 2026-04-24 14:00 | p99 780 ms, erro 0,8%, tráfego 1780 req/s |
| 2026-04-24 14:10 | degradação forte: p99 2400 ms, erro 4,5% |
| 2026-04-24 14:20 | p99 8100 ms, erro 11,7%, tráfego 2650 req/s |
| 2026-04-24 14:19-14:20 | logs mostram pool esgotado, timeouts, circuit breaker aberto e falha no Reactor |

## Evidências

- `connection pool exhausted (max=20, active=20, waiting=147)` indica contenção no cliente Ledger.
- `query timeout after 2000ms` coincide com a mudança de timeout de 5s para 2s.
- `POST /v2/transactions/batch failed` conecta o erro ao endpoint novo ou ao seu caminho de execução.
- Chronos está com 12/12 pods, mas CPU média é 62% e memória 71%, então o gargalo principal não parece ser compute.
- Ledger está em 240/250 conexões, próximo do limite do RDS.
- Reactor está acumulando mensagens como efeito secundário da falha upstream do Chronos.

## Causa provável

Inferência: a versão `v2.48.0` introduziu uma combinação ruim entre novo padrão de acesso ao Ledger, pool de conexões na nova biblioteca interna, endpoint batch e timeout reduzido. Sob pico de tráfego, isso esgota pools por pod, pressiona o limite de conexões do RDS, aumenta timeouts, abre circuit breaker e impede publicação consistente no Reactor.

## Decisão recomendada

Executar **rollback imediato para `v2.47.0`**.

Motivos:

- Há forte correlação temporal e técnica com mudanças do deploy.
- O HPA já atingiu o máximo, então escalar Chronos horizontalmente não está disponível.
- CPU e memória não justificam scaling de compute como resposta primária.
- A saturação de conexões do Ledger parece consequência do comportamento novo da aplicação.
- Rollback reduz a pressão introduzida pela nova versão e é reversível.

Scaling emergencial de RDS/pool deve ficar como plano B se, após rollback, conexões e latência não começarem a cair em 10 a 15 minutos.

## Riscos

- Rollback pode não aliviar imediatamente o backlog do Reactor; será necessário acompanhar drenagem.
- Se parte do tráfego já depende do endpoint batch, haverá perda temporária dessa funcionalidade.
- Se houve migração acoplada ao deploy, validar compatibilidade antes do rollback.
- Scaling de RDS durante o incidente pode mascarar o bug e aumentar custo sem corrigir causa.

## Próximos passos imediatos

1. Validar se `v2.47.0` é compatível com o estado atual de banco e configuração.
2. Executar rollback via Argo CD.
3. Monitorar por 15 minutos: p99, erro, conexões do Ledger, circuit breaker e backlog do Reactor.
4. Se conexões continuarem acima de 230/250 e erro acima de 5%, acionar plano B: aumento temporário de limite RDS/pool com `@chronos-core`.
5. Comunicar no canal de incidente a decisão, horário e métrica esperada de recuperação.

## Ações pós-incidente

- Revisar implementação do pool na nova biblioteca Ledger.
- Testar `POST /v2/transactions/batch` com carga realista antes de novo deploy.
- Reavaliar redução de timeout de 5s para 2s.
- Adicionar limite de concorrência por pod para chamadas ao Ledger.
- Criar alerta para pool waiting count e conexões RDS acima de 85%.
- Rodar canary release para reintroduzir `v2.48.0` corrigida.

## Justificativa estendida

Escolhi R-I-S-E porque o cenário exige resposta operacional estruturada sob pressão. Role aparece ao posicionar a IA como staff SRE; Input aparece nos artefatos de deploy, métricas, logs, Reactor e cluster; Steps aparecem na sequência de análise causal e decisão; Expectation aparece no formato exigido para o postmortem e na recomendação explícita.

Comparando com R-T-F: ele ajudaria a definir papel, tarefa e formato de saída rapidamente, o que seria útil para produzir um documento bem formatado. Porém, perderia força na decomposição do raciocínio investigativo, porque não obriga a IA a seguir passos de análise antes da conclusão.

Comparando com T-A-G: ele seria bom para manter foco na ação e no objetivo, por exemplo "decidir rollback ou scaling". A perda seria menor controle sobre a sequência de investigação e sobre o uso disciplinado das evidências, que é crítico em postmortem técnico durante incidente.

Comparando com B-A-B: ele poderia enquadrar bem o antes do deploy, o depois degradado e a ponte para estabilização. O ganho seria uma narrativa clara de transformação do estado, mas a perda seria que o framework favorece persuasão e contraste, não necessariamente diagnóstico técnico com hipóteses, evidências e critérios de decisão.
