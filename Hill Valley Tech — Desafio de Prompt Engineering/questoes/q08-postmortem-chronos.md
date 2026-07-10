# Questão 08 — Postmortem técnico de incidente em produção

## Framework escolhido

Framework escolhido: **R-I-S-E — Role, Input, Steps, Expectation**.

## Prompt

```markdown
# Role
Você é um Staff SRE e Incident Commander especializado em Kubernetes/EKS, PostgreSQL/RDS, sistemas distribuídos, AIOps, análise de incidentes e tomada de decisão sob pressão em ambiente de produção.

# Input
Incidente em andamento durante pico de tráfego. Doc Brown precisa de um postmortem técnico em 20 minutos para decidir entre:

1. Rollback do deploy v2.48.0, que subiu ontem.
2. Scaling emergencial, com aumento de limits do RDS e do pool de conexões.

Artefatos disponíveis:

## Evento do deploy anterior
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s

## Métricas do Beacon nos últimos 30 minutos

timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7

## Log do pod chronos-api-79c4d8b9-xk2jp
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error

## Estado do Reactor
- Fila chronos-transactions: 50.127 mensagens acumuladas.
- Crescimento: ~800 mensagens/min.
- Consumer lag atual: 18 minutos e aumentando.

## Estado do cluster
- Chronos: 12/12 pods running.
- HPA no máximo.
- CPU médio dos pods: 62%.
- Memória média dos pods: 71%.
- Conexões ativas ao Ledger: 240/250, próximo do limite do RDS.

# Steps
Analise o incidente nesta sequência:
1. Criar um resumo executivo em linguagem direta para Doc Brown.
2. Construir uma linha do tempo dos eventos e métricas.
3. Identificar hipóteses de causa raiz e classificá-las por probabilidade.
4. Relacionar cada hipótese às evidências disponíveis.
5. Comparar rollback vs scaling emergencial, com prós, contras, riscos e tempo esperado de efeito.
6. Recomendar uma decisão operacional imediata.
7. Listar validações obrigatórias nos próximos 10 minutos.
8. Listar follow-ups preventivos para depois da estabilização.

# Expectation
Retorne um postmortem técnico preliminar, adequado para tomada de decisão em 20 minutos, com as seções:
- Resumo executivo
- Severidade e impacto
- Linha do tempo
- Hipóteses classificadas
- Evidências técnicas
- Decisão recomendada: rollback ou scaling emergencial
- Plano de ação imediato
- Riscos da decisão
- Dados adicionais necessários
- Follow-ups preventivos

Não invente dados. Diferencie evidência, inferência e suposição. Priorize a opção com maior chance de estabilizar o serviço com menor risco operacional.
```

## Modelo

**Claude Sonnet 4 — Anthropic.** Escolhido por boa capacidade de análise estruturada de incidente, comparação entre alternativas e redação de documento técnico sob restrição de tempo.

## Output

# Postmortem técnico preliminar — Chronos API

## 1. Resumo executivo

O incidente apresenta forte evidência de saturação na camada de acesso ao Ledger, agravada por mudanças introduzidas no deploy `v2.48.0`. A aplicação está com HPA no máximo, mas CPU média está em apenas 62% e memória em 71%, o que indica que o gargalo principal não é capacidade computacional dos pods. Os logs mostram exaustão do pool de conexões (`max=20`, `waiting=147`), timeouts de 2s no Ledger, circuit breaker aberto e falhas no fluxo assíncrono para o Reactor.

A recomendação imediata é **rollback para v2.47.0**, não scaling emergencial como primeira ação. O scaling pode piorar a pressão no Ledger, pois as conexões já estão em 240/250 no RDS. A mudança mais suspeita é a refatoração do cliente Ledger, combinada com redução de timeout de 5s para 2s e novo endpoint batch.

## 2. Severidade e impacto

- Severidade sugerida: **SEV1**, pois há aumento progressivo de latência p99 e erro durante pico de tráfego.
- p99 subiu de **420 ms para 8.100 ms** em 50 minutos.
- Taxa de erro subiu de **0,2% para 11,7%**.
- Reactor acumula **50.127 mensagens**, crescendo **~800/min**.
- Consumer lag está em **18 minutos e aumentando**.

## 3. Linha do tempo

| Horário UTC | Evento |
|---|---|
| 2026-04-23 18:42 | Deploy `chronos-api v2.47.0 -> v2.48.0` via Argo CD |
| 2026-04-24 13:30 | p99 420 ms, req/s 1200, erro 0,2% |
| 2026-04-24 13:45 | p99 510 ms, req/s 1450, erro 0,3% |
| 2026-04-24 14:00 | p99 780 ms, req/s 1780, erro 0,8% |
| 2026-04-24 14:10 | p99 2400 ms, req/s 2100, erro 4,5% |
| 2026-04-24 14:15 | p99 5200 ms, req/s 2400, erro 8,2% |
| 2026-04-24 14:19 | Logs indicam pool do Ledger exausto, timeouts e circuit breaker aberto |
| 2026-04-24 14:20 | p99 8100 ms, req/s 2650, erro 11,7%; HPA em 12/12 pods |

## 4. Hipóteses classificadas

| Probabilidade | Hipótese | Evidências | Observação |
|---|---|---|---|
| Alta | Regressão no cliente Ledger introduzida no deploy v2.48.0 | Refatoração do pool, bump de psycopg, timeout reduzido, log de pool exhausted | Melhor hipótese para explicar mudança comportamental após deploy |
| Alta | Novo endpoint batch aumentou concorrência e pressão sobre Ledger | Endpoint `/v2/transactions/batch` falha com deadline exceeded; RDS em 240/250 conexões | Pode ter ampliado número de queries/conexões por request |
| Média | RDS atingiu limite de conexões durante pico | 240/250 conexões ativas; timeouts e reset by peer | Pode ser efeito, não causa primária |
| Média | Timeout de 2s é agressivo para carga de pico | Logs mostram query timeout after 2000ms | Pode transformar lentidão parcial em erro sistêmico |
| Baixa | Falta de CPU/memória nos pods Chronos | CPU 62%, memória 71% | HPA no máximo, mas sem saturação local evidente |

## 5. Evidências técnicas principais

### Evidências diretas

- `connection pool exhausted (max=20, active=20, waiting=147)` indica filas internas aguardando conexão ao Ledger.
- `query timeout after 2000ms` correlaciona com mudança de timeout de 5s para 2s no deploy.
- `circuit-breaker ledger-client OPEN` mostra falha sistemática de dependência.
- `POST /v2/transactions/batch failed` aponta o novo endpoint do deploy como participante do incidente.
- RDS com **240/250 conexões** mostra risco de saturação global do banco.

### Inferências

- Aumentar pods ou pool sem controle pode elevar ainda mais conexões concorrentes ao Ledger.
- O gargalo parece estar na integração Chronos-Ledger, não na capacidade bruta de CPU/memória do Chronos.
- O Reactor está sofrendo efeito cascata por falhas upstream no Chronos.

## 6. Comparação: rollback vs scaling emergencial

| Critério | Rollback para v2.47.0 | Scaling emergencial RDS/pool |
|---|---|---|
| Tempo de efeito | Rápido após rollout saudável | Pode exigir alteração em RDS, app config e novo deploy |
| Risco imediato | Médio: reverter código em pico exige cuidado | Alto: pode aumentar pressão no banco e amplificar fila |
| Alvo da causa provável | Alto: remove mudanças do cliente Ledger, timeout e endpoint batch | Parcial: trata sintoma de capacidade/conexão |
| Evidência favorável | Deploy alterou pool, psycopg, timeout e endpoint batch | RDS próximo do limite de conexão |
| Risco de piora | Rollback mal executado pode causar instabilidade temporária | Mais conexões podem levar RDS a 250/250 e degradar todos os clientes |
| Reversibilidade | Alta com Argo CD | Média; mudanças em RDS podem ter efeitos colaterais |

## 7. Decisão recomendada

**Recomendo rollback imediato para `v2.47.0` como primeira ação.**

Justificativa:

1. O deploy `v2.48.0` alterou diretamente a área em falha: cliente Ledger, pool de conexões, versão do driver e timeout.
2. O cluster já está no máximo do HPA, mas CPU e memória não indicam saturação local.
3. O Ledger está próximo do limite de conexões; aumentar pool ou pods pode piorar o gargalo.
4. O novo endpoint batch aparece diretamente nos logs de falha.
5. Rollback é mais reversível e ataca a hipótese de causa mais provável.

## 8. Plano de ação imediato

### Ação 1 — Congelar mudanças

```text
Declarar freeze temporário de deploys no Chronos até estabilização.
```

### Ação 2 — Executar rollback via Argo CD

```bash
argocd app history chronos-api
argocd app rollback chronos-api <revision-v2.47.0>
argocd app wait chronos-api --health --timeout 300
```

### Ação 3 — Monitorar rollout

```bash
kubectl -n production rollout status deploy/chronos-api --timeout=5m
kubectl -n production get pods -l app.kubernetes.io/name=chronos-api -o wide
```

### Ação 4 — Validar estabilização

Nos 10 minutos após rollback, acompanhar:

- p99 latency.
- error rate.
- conexões ativas no Ledger.
- backlog e consumer lag do Reactor.
- abertura/fechamento do circuit breaker.

## 9. Riscos da decisão

- Se o tráfego atual for muito superior ao baseline, rollback pode não ser suficiente.
- Se o banco já estiver degradado por acúmulo de conexões, a recuperação pode levar alguns minutos.
- Se o schema ou comportamento do novo endpoint tiver efeitos persistentes, pode haver impacto em mensagens já acumuladas.
- O backlog do Reactor pode continuar alto mesmo após estabilizar o Chronos, exigindo plano separado de drenagem.

## 10. Dados adicionais necessários

- Gráfico de conexões RDS antes e depois do deploy `v2.48.0`.
- Taxa de uso do endpoint `/v2/transactions/batch`.
- Queries mais lentas no Ledger durante a janela do incidente.
- Configuração efetiva do pool por pod e número total de pods.
- Diferença de comportamento entre psycopg `3.1.18` e `3.2.0` no uso interno.
- Métricas de circuit breaker e retry rate.

## 11. Follow-ups preventivos

1. Adicionar teste de carga específico para `/v2/transactions/batch`.
2. Definir limite de concorrência por endpoint batch.
3. Revisar configuração de pool por pod considerando limite total do RDS.
4. Implementar métrica de pool waiting/active por pod.
5. Criar alerta para conexões RDS acima de 80%.
6. Exigir canary ou progressive delivery para mudanças em clientes de dependências críticas.
7. Revisar timeout de Ledger com base em SLO real, não apenas valor fixo agressivo.
8. Criar runbook específico para saturação de pool Chronos-Ledger.

## Justificativa estendida

### Por que R-I-S-E foi escolhido

O R-I-S-E é o framework mais adequado porque o cenário exige análise procedural sob pressão, usando múltiplas entradas concretas: changelog, métricas, logs, estado de fila, HPA e conexões RDS. O **Role** define a expertise de Staff SRE/Incident Commander. O **Input** concentra os artefatos de diagnóstico. Os **Steps** forçam uma sequência analítica: resumo, linha do tempo, hipóteses, evidências, comparação e decisão. A **Expectation** define o formato do postmortem e exige separação entre evidência, inferência e suposição.

### Comparação com T-A-G

O T-A-G também seria defensável porque há um objetivo claro: decidir em 20 minutos entre rollback e scaling. Ele ajudaria a manter foco no resultado de negócio e na decisão operacional. O ganho seria uma resposta mais executiva e orientada ao objetivo. A perda seria menor controle sobre a sequência investigativa, aumentando o risco de o modelo pular evidências ou não construir linha do tempo e hipóteses de forma disciplinada.

### Comparação com B-A-B

O B-A-B poderia enquadrar o incidente como transformação de um estado ruim para um estado estabilizado: Before com sistema degradado, After com serviço recuperado e Bridge com plano de estabilização. O ganho seria clareza sobre o caminho de recuperação. A perda é que o framework é melhor para transformação planejada, migração ou refatoração, não para análise de incidente com múltiplas evidências e tomada de decisão comparativa em tempo real.

### Comparação adicional com R-T-F

O R-T-F seria simples e rápido para pedir “gere um postmortem neste formato”. O ganho seria baixo overhead. A perda seria grande: o cenário tem dados operacionais densos, hipótese concorrente e decisão crítica. Sem Steps e Expectation, o modelo poderia produzir um postmortem superficial ou bonito, mas pouco confiável para decisão durante SEV.
