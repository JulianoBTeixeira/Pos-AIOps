# Questão 07 - Runbook para alerta recorrente

## Prompt

Você é uma pessoa SRE líder escrevendo documentação operacional para plantonistas de diferentes níveis de experiência.

Input do cenário:

- Alerta: `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`
- Frequência: cerca de 4 vezes por semana
- Chronos roda no EKS, namespace `production`
- 6 réplicas normalmente
- HPA configurado com min 4, max 12, CPU target 70%
- Deploy via Argo CD a partir do repositório `hvt/chronos-api`
- Dependências diretas: Ledger PostgreSQL e Reactor SQS
- Métricas expostas em `/metrics`
- Logs centralizados no Beacon
- Dashboards em Grafana
- Ferramentas disponíveis: `kubectl`, `aws cli`, `argocd cli`
- Canal de plantão: `#oncall-chronos`
- Time sênior de escalação: `@chronos-core`
- SLA de resposta do time sênior: 15 minutos em horário comercial, 30 minutos fora

Steps esperados no runbook:

1. Confirmar o alerta e o escopo do impacto.
2. Identificar pods afetados e uso atual de memória.
3. Verificar eventos, restarts, OOMKilled e pressão de recursos no namespace.
4. Conferir HPA, réplicas e se o HPA chegou ao máximo.
5. Analisar logs recentes do Chronos no Beacon.
6. Verificar dependências Ledger e Reactor.
7. Executar mitigações seguras e reversíveis.
8. Definir critérios objetivos de escalação.
9. Definir critério de encerramento do incidente.

Expectation: produza um runbook procedural completo, em português, com comandos específicos, resultado esperado ao final de cada passo, decisões condicionais claras, critérios objetivos para escalar e critério de encerramento. O texto deve ser direto, utilizável durante incidente real e sem depender de conhecimento tribal.

## Modelo

Claude Sonnet 4. Escolhido por boa estruturação de documentação operacional e procedimentos passo a passo.

## Output

# Runbook - High memory usage on Chronos API pods

## 1. Confirmar alerta e impacto

Comandos:

```bash
kubectl -n production get pods -l app=chronos-api -o wide
kubectl -n production top pods -l app=chronos-api
```

Verificação esperada: identificar quantos pods estão acima de 85% de memória e se há pods reiniciando ou indisponíveis.

Se mais de 50% dos pods estiverem acima de 85% por mais de 10 minutos, tratar como incidente ativo e avisar `#oncall-chronos`.

## 2. Verificar restarts, OOMKilled e eventos

Comandos:

```bash
kubectl -n production describe pods -l app=chronos-api
kubectl -n production get events --sort-by=.lastTimestamp | tail -50
```

Verificação esperada: procurar `OOMKilled`, `BackOff`, falhas de readiness/liveness e pressão de memória no node.

Escalar para `@chronos-core` se houver OOMKilled repetido em 2 ou mais pods nos últimos 15 minutos.

## 3. Conferir HPA e réplicas

Comandos:

```bash
kubectl -n production get hpa
kubectl -n production describe hpa chronos-api
kubectl -n production get deploy chronos-api
```

Verificação esperada: confirmar número atual de réplicas, alvo de CPU e se o HPA atingiu o máximo de 12 pods.

Se HPA estiver em 12/12 e a memória continuar acima de 85%, escalar para `@chronos-core`, pois o autoscaling horizontal já chegou ao limite configurado.

## 4. Analisar logs recentes

Comandos de apoio:

```bash
kubectl -n production logs deploy/chronos-api --since=15m --all-containers=true | tail -200
```

No Beacon, filtrar por:

```text
service=chronos-api severity>=warn timestamp>=now-15m
```

Verificação esperada: identificar aumento de erros, timeouts para Ledger, falhas de Reactor, vazamento aparente ou endpoint específico concentrando tráfego.

Escalar se houver erro sistêmico de dependência ou crescimento contínuo de erros 5xx.

## 5. Verificar Ledger e Reactor

Comandos:

```bash
QUEUE_URL="$(aws sqs get-queue-url \
  --queue-name chronos-transactions \
  --query QueueUrl \
  --output text)"

aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage
```

No Grafana, abrir dashboards de Chronos, Ledger e Reactor e comparar os últimos 30 minutos.

Verificação esperada: Ledger sem saturação de conexões/latência anormal e Reactor sem crescimento acelerado de fila.

Se Ledger ou Reactor estiver degradado, abrir incidente dependente e marcar `@chronos-core` no `#oncall-chronos`.

## 6. Mitigações seguras

Se o problema for pico de tráfego e o HPA ainda não estiver no máximo:

```bash
kubectl -n production scale deploy/chronos-api --replicas=8
```

Se houver suspeita de deploy problemático:

```bash
argocd app history chronos-api
argocd app rollback chronos-api <REVISION_ID>
```

Verificação esperada: em até 10 minutos, memória média abaixo de 80%, queda de erros e readiness estável.

Não aumentar limites de memória sem validação do time sênior se houver indício de vazamento.

## 7. Critérios de escalação

Escalar para `@chronos-core` quando qualquer condição ocorrer:

- HPA em 12/12 e memória acima de 85% por mais de 10 minutos.
- OOMKilled em 2 ou mais pods nos últimos 15 minutos.
- Erro 5xx acima de 2% por 10 minutos.
- Ledger ou Reactor degradado junto com o alerta.
- Necessidade de rollback ou alteração de limites em produção.
- Dúvida operacional que possa prolongar o incidente por mais de 15 minutos.

## 8. Critério de encerramento

Encerrar somente quando, por pelo menos 20 minutos:

- Memória média dos pods estiver abaixo de 80%.
- Nenhum pod novo tiver OOMKilled.
- Readiness estiver estável.
- Erros 5xx estiverem dentro do baseline.
- Ledger e Reactor estiverem sem backlog ou latência crescente.

Registrar no `#oncall-chronos`: causa provável, ações tomadas, horário de início/fim e se houve escalação.

## Justificativa

Role aparece ao definir a IA como SRE líder escrevendo documentação operacional. Input aparece na lista completa de ambiente, alerta, ferramentas, dependências e canais. Steps aparece na sequência obrigatória de diagnóstico, mitigação, escalação e encerramento. Expectation aparece nos critérios de saída: runbook procedural, comandos específicos, verificações esperadas e decisões condicionais claras.
