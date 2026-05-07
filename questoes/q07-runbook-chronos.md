# Questão 07 — Runbook para alerta recorrente

## Prompt

```markdown
# Role
Você é um SRE sênior especializado em Kubernetes/EKS, AIOps, observabilidade, resposta a incidentes e criação de runbooks operacionais para times de plantão.

# Input
Alerta recorrente no Beacon:
[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)

Contexto do ambiente:
- Serviço: Chronos API
- Plataforma: EKS
- Namespace: production
- Réplicas atuais: 6
- HPA: min 4, max 12, CPU target 70%
- Deploy: Argo CD a partir do repositório hvt/chronos-api
- Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS)
- Observabilidade: métricas em /metrics, logs centralizados no Beacon, dashboards em Grafana
- Ferramentas disponíveis: kubectl, aws cli, argocd cli
- Canal de plantão: #oncall-chronos no Slack
- Time sênior de escalação: @chronos-core
- SLA de resposta da escalação: 15 min em horário comercial, 30 min fora

# Steps
Crie um runbook procedural cobrindo:
1. Triagem inicial do alerta e confirmação do impacto.
2. Coleta do estado dos pods, HPA e eventos Kubernetes.
3. Identificação de quais pods estão com maior uso de memória.
4. Análise de logs do Chronos no período do alerta.
5. Verificação de dependências Ledger e Reactor.
6. Avaliação de mitigação segura: restart seletivo, rollback via Argo CD ou ajuste temporário de escala.
7. Critérios objetivos para escalar ao @chronos-core.
8. Critérios objetivos para encerrar o incidente.
9. Evidências que devem ser registradas no canal #oncall-chronos.

# Expectation
O runbook deve ser completo, prático e executável por um plantonista que não conhece profundamente o Chronos. Para cada etapa, inclua:
- objetivo;
- comandos específicos;
- o que verificar;
- resultado esperado;
- próxima ação se o resultado for anormal.

Use apenas comandos realistas com `kubectl`, `aws cli` e `argocd cli`. Inclua limites de tempo por etapa e mantenha o procedimento total dentro de 45 minutos.
```

## Modelo

**Claude Sonnet 4 — Anthropic.** Escolhido pela capacidade de estruturar procedimentos operacionais longos com critérios objetivos e comandos acionáveis.

## Output

# Runbook — High memory usage on Chronos API pods

## 1. Escopo

Este runbook orienta a resposta ao alerta:

`[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`

Serviço afetado: **Chronos API** no namespace **production**.

Tempo alvo de execução: **até 45 minutos**.

## 2. Comunicação inicial — 2 minutos

| Item | Detalhe |
|---|---|
| Objetivo | Registrar início da investigação e evitar trabalho duplicado |
| Comando | Não aplicável |
| O que fazer | Postar no `#oncall-chronos`: horário do alerta, responsável, severidade inicial e link do dashboard |
| Resultado esperado | Incidente assumido por um plantonista |
| Se anormal | Se houver impacto em clientes, declarar incidente e acionar fluxo SEV |

Mensagem sugerida:

```text
Assumindo alerta CRITICAL de memória alta no Chronos API em production. Iniciando triagem. Próxima atualização em até 10 min.
```

## 3. Confirmar estado geral — 5 minutos

| Objetivo | Confirmar pods, réplicas e saúde do deployment |
|---|---|
| Comandos | Ver abaixo |

```bash
kubectl -n production get deploy chronos-api
kubectl -n production get pods -l app.kubernetes.io/name=chronos-api -o wide
kubectl -n production get hpa chronos-api
kubectl -n production get events --sort-by=.lastTimestamp | tail -n 30
```

Resultado esperado:

- Deployment disponível.
- Pods em `Running` e `Ready`.
- HPA dentro do limite configurado.
- Sem eventos repetidos de `OOMKilled`, `Evicted`, `FailedScheduling` ou `BackOff`.

Próxima ação anormal:

- Se houver `OOMKilled`, ir para etapa 4 e coletar logs anteriores.
- Se HPA estiver no máximo, avaliar pressão real antes de escalar.
- Se deployment indisponível, escalar para `@chronos-core` imediatamente.

## 4. Identificar pods com maior memória — 5 minutos

```bash
kubectl -n production top pods -l app.kubernetes.io/name=chronos-api --containers
kubectl -n production describe pods -l app.kubernetes.io/name=chronos-api | grep -E "Name:|State:|Last State:|Reason:|OOMKilled|Limits:|Requests:" -A 3
```

Resultado esperado:

- Identificação dos pods acima de 85% de uso de memória.
- Ausência de reinícios frequentes.
- Requests/limits coerentes com o consumo observado.

Próxima ação anormal:

- Se 1 ou 2 pods concentram o problema, considerar restart seletivo após verificar readiness dos demais pods.
- Se todos os pods estão crescendo de forma uniforme, suspeitar de aumento de tráfego, leak após deploy ou dependência lenta.

## 5. Analisar logs do Chronos — 8 minutos

```bash
kubectl -n production logs deploy/chronos-api --since=30m --all-containers=true | grep -Ei "error|exception|timeout|memory|oom|ledger|reactor|gc|failed" | tail -n 200
```

Se for necessário analisar um pod específico:

```bash
kubectl -n production logs <pod-name> --since=30m --all-containers=true
kubectl -n production logs <pod-name> --previous --all-containers=true
```

Resultado esperado:

- Sem aumento relevante de erros.
- Sem mensagens de exaustão de pool, timeouts em Ledger ou falhas de publicação no Reactor.

Próxima ação anormal:

- Erros de Ledger: verificar conexões e latência do PostgreSQL.
- Erros de Reactor/SQS: verificar backlog e consumer lag.
- Erros após deploy recente: verificar Argo CD e considerar rollback controlado.

## 6. Verificar Argo CD e deploy recente — 5 minutos

```bash
argocd app get chronos-api
argocd app history chronos-api
```

Resultado esperado:

- Aplicação `Synced` e `Healthy`.
- Sem deploy recente correlacionado diretamente ao início do alerta.

Próxima ação anormal:

- Se houver deploy recente correlacionado ao aumento de memória, coletar versão atual e anterior.
- Preparar rollback, mas executar apenas com aprovação do responsável pelo incidente.

## 7. Verificar dependências Ledger e Reactor — 8 minutos

### Ledger

```bash
kubectl -n production logs deploy/chronos-api --since=30m | grep -Ei "ledger|postgres|connection|timeout" | tail -n 100
```

### Reactor / SQS

```bash
aws sqs get-queue-attributes \
  --queue-url <chronos-transactions-queue-url> \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible ApproximateAgeOfOldestMessage \
  --region us-east-1
```

Resultado esperado:

- Sem timeouts recorrentes no Ledger.
- SQS sem crescimento anormal de mensagens ou idade elevada.

Próxima ação anormal:

- Se Ledger lento, escalar para equipe responsável por banco antes de aumentar carga no Chronos.
- Se Reactor acumulando fila, informar no canal e avaliar impacto de processamento assíncrono.

## 8. Mitigações seguras — até 10 minutos

### 8.1 Restart seletivo de pod
Usar quando poucos pods estão com memória alta e os demais estão saudáveis.

```bash
kubectl -n production delete pod <pod-name>
kubectl -n production rollout status deploy/chronos-api --timeout=5m
```

Critério de sucesso:

- Novo pod fica `Ready`.
- Memória média cai abaixo de 80%.
- Erros não aumentam.

### 8.2 Rollback via Argo CD
Usar quando houver correlação clara com deploy recente.

```bash
argocd app history chronos-api
argocd app rollback chronos-api <revision-id>
argocd app wait chronos-api --health --timeout 300
```

Critério de sucesso:

- Aplicação volta para `Healthy`.
- Métrica de memória estabiliza.
- Taxa de erro não piora.

### 8.3 Escala temporária
Usar quando há aumento de tráfego e memória alta uniforme, sem indício de leak.

```bash
kubectl -n production scale deploy chronos-api --replicas=8
kubectl -n production rollout status deploy/chronos-api --timeout=5m
```

Critério de sucesso:

- Novos pods ficam prontos.
- Memória média por pod reduz.
- Latência e erro permanecem estáveis.

## 9. Critérios de escalação para @chronos-core

Escalar imediatamente se qualquer condição ocorrer:

- Pods entram em `OOMKilled` repetidamente.
- Memória permanece acima de 90% por mais de 15 minutos após mitigação.
- HPA chega ao máximo e o uso continua subindo.
- Há correlação forte com deploy recente e rollback não é seguro sem análise do time dono.
- Erro HTTP 5xx aumenta acima de 2% por mais de 5 minutos.
- Ledger ou Reactor apresentam sintomas de saturação.
- O plantonista não consegue identificar causa provável em até 20 minutos.

Mensagem sugerida:

```text
@chronos-core escalando alerta de memória alta no Chronos. Evidências: <resumo>. Mitigações tentadas: <ações>. Impacto atual: <métricas>. Precisamos de apoio em até 15/30 min conforme SLA.
```

## 10. Critérios de encerramento

Encerrar o incidente apenas quando:

- Memória média dos pods permanecer abaixo de 80% por pelo menos 15 minutos.
- Nenhum pod estiver em restart loop ou OOMKilled.
- Deployment estiver `Available` e Argo CD `Synced/Healthy`.
- Taxa de erro e latência estiverem dentro do baseline.
- Ledger e Reactor não mostrarem backlog ou timeouts anormais.
- Evidências, causa provável e ação aplicada forem registradas no `#oncall-chronos`.

## 11. Registro final obrigatório

```text
Resumo: <o que aconteceu>
Impacto: <serviço/usuários afetados>
Causa provável: <hipótese mais forte>
Ação tomada: <mitigação>
Validação: <métricas pós-ação>
Follow-up: <issue/runbook/ajuste definitivo>
```

## Justificativa

O **Role** define a expertise SRE/Kubernetes/AIOps necessária. O **Input** traz alerta, ambiente, ferramentas, dependências e canais. Os **Steps** determinam a sequência de diagnóstico, mitigação e encerramento. A **Expectation** exige comandos, verificações, resultados esperados, ações anormais e limite de 45 minutos.
