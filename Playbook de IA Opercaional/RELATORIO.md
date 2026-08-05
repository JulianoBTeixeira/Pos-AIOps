# Relatório de Curadoria do Playbook

## 01 - Triagem de pods

- Modelo: `openai:gpt-4o-mini`
- Motivo: barato, rápido e suficiente para saída estruturada.
- Saída de referência:

```text
- Pod: sentinel-api-7d9c8b6f4-h4m2t
  Estado: CrashLoopBackOff
  Causa provável: OOMKilled com heap encostando no limite de memória.
  Próxima ação: aumentar memória ou revisar uso de cache/leak.
- Pod: sentinel-api-7d9c8b6f4-zzp10
  Estado: ImagePullBackOff
  Causa provável: tag 2.9.2 inexistente no registry.
  Próxima ação: corrigir a imagem e reenfileirar o rollout.
- Pod: sentinel-ingest-8f7a9c5b-4dkqm
  Estado: Pending
  Causa provável: falta de CPU no cluster.
  Próxima ação: liberar capacidade ou reduzir requests.

Nenhum pod problemático detectado.
```

- Curadoria: usei correlação entre status, eventos e logs, sem confiar apenas no `STATUS`.

## 02 - Nota de triagem

- Modelo: `openai:gpt-4o-mini`
- Motivo: formato rígido, baixo custo, latência curta.
- Saída de referência:

```text
ALERTA: Relay - taxa de rejeição de ingestão acima de 2% por 5min
IMPACTO: ingestão de telemetry degradada para ~12% dos tenants
HIPÓTESE INICIAL: deploy do Relay às 09:14 reduziu o buffer de ingestão
AÇÃO IMEDIATA: rollback iniciado via Argo CD
ESCALAR PARA: @relay-core se a rejeição não cair em 10min
```

- Curadoria: usei o próprio formato de saída como guia, em vez de narrar livremente.

## 03 - Causa-raiz do Cerebro

- Modelo: `anthropic:claude-sonnet-4`
- Motivo: melhor para raciocínio causal e separação entre causa, sintoma e efeito.
- Saída de referência:

```text
Resumo executivo: a reindexação agendada ficou longa demais, saturou heap e fila de escrita, e isso levou a circuit breaker, timeouts e partial results.
Causa-raiz provável: reindex travada competindo por heap com consultas e ingestão.
Cadeia causal: reindex prolongada -> heap alto -> throttling/reject -> cache hit cai -> busca perde desempenho.
Ação imediata: pausar/reagendar reindex e aliviar pressão de heap.
```

- Curadoria: destaquei que cache baixo e timeouts são consequência, não a causa.

## 04 - Backpressure do Relay

- Modelo: `anthropic:claude-sonnet-4`
- Motivo: a tarefa pede comparação de alternativas com trade-offs e recomendação combinada.
- Saída de referência:

```text
Diagnóstico: o Relay sofreu pico acima da vazão sustentada e o Sentinel tem SLA mais rígido que o Forge.
Recomendação: priorizar Sentinel, isolar tenants barulhentos e subir consumidores de forma controlada.
Por que não as outras: DLQ sozinha adia problema; autoscaling sozinho pode ficar caro; split por tenant sem priorização não protege o SLA.
Próximos passos: aplicar priorização, medir lag e só então ajustar partições.
```

- Curadoria: comparei opções antes de recomendar, porque a decisão é de arquitetura, não de sintoma.

## 05 - Migração do Forge

- Modelo: `google:gemini-2.5-pro`
- Motivo: bom para lidar com cadeia longa e contexto amplo.
- Cadeia de referência:

```text
01-diagnostico: batch atual, dependências e fragilidades.
02-estrategia: fases pequenas, coexistência batch/eventos e rollback.
03-plano-executavel: etapas, responsáveis, critérios de conclusão e rollback por fase.
```

- Curadoria: dividi o problema para não pedir plano, estratégia e diagnóstico em um único salto.

## 06 - NetworkPolicy do Sentinel

- Modelo: `anthropic:claude-sonnet-4`
- Motivo: melhor para revisão de segurança e verificação iterativa.
- Iterações:

```text
v1: manifesto permissivo com allow-all implícito.
verificação: apontou allow-all, ausência de default-deny e uso indevido de podSelector vazio.
v2: separou default-deny e regras específicas de ingress/egress.
v3: refinou comentários e confirmou seletores e portas do mapa de serviços.
```

- Curadoria: a saída final só foi aceita depois de uma rodada de auto-crítica, como um revisor de segurança faria.

## 07 - Biblioteca como código

- Estrutura criada:
  - `devops/triagem-de-pods/`
  - `devops/nota-de-triagem/`
  - `devops/causa-raiz-cerebro/`
  - `devops/backpressure-relay/`
  - `devops/forge-event-driven/`
  - `devops/networkpolicy-sentinel/`
- Exemplo completo: `devops/triagem-de-pods/`.
- Mapeamento: cada prompt tem `prompt.md`, `README.md` e, quando faz sentido, `promptfooconfig.yaml`.

## 08 - Testes determinísticos

- Arquivos criados:
  - `devops/nota-de-triagem/promptfooconfig.yaml`
  - `devops/triagem-de-pods/promptfooconfig.yaml`
  - `devops/networkpolicy-sentinel/promptfooconfig.yaml`
- Regras cobertas:
  - formato, regex, tamanho, latência e custo.
- Observação: a execução real depende de providers configurados no ambiente.

## 09 - Gate LLM-as-judge

- Arquivos criados:
  - `devops/causa-raiz-cerebro/rubrica.md`
  - `devops/causa-raiz-cerebro/promptfooconfig.yaml`
- Rubrica:
  - causa-raiz correta
  - correlação x causa
  - ação proporcional
  - honestidade epistêmica
- Corte: nota total >= 6 e nenhum critério zerado.

## 10 - Pipeline contínuo

- Arquivo criado: `.github/workflows/promptfoo.yml`
- Estratégia de gate:
  - falha se qualquer config determinístico reprovar;
  - falha se o juiz de causa-raiz cair abaixo do corte.
- Justificativa comparativa:
  - rodar tudo em todo push/PR é mais caro, mas mais seguro;
  - rodar só o que mudou é mais barato, mas arrisca regressões fora do diff;
  - validar só asserts determinísticos é rápido, mas não pega qualidade do texto livre.

## Nota de correção (professor)

- O checkpoint 09 foi mantido no formato original de gate com juiz LLM (rubrica + `llm-rubric`).
- A validação de execução do checkpoint 09 depende de provider autenticado no ambiente.
- Na ausência de credenciais, a validação recomendada é estrutural.
- Validar existência de `devops/causa-raiz-cerebro/rubrica.md`.
- Validar uso de `llm-rubric` em `devops/causa-raiz-cerebro/promptfooconfig.yaml`.
- Validar workflow centralizado em `.github/workflows/promptfoo.yml`.
- Os checkpoints determinísticos (08) permanecem executáveis sem juiz externo.

## Nota final

Os arquivos foram organizados para preservar o workspace antigo sem alterações. Os outputs acima são a curadoria de referência do playbook e servem como base para a execução com providers reais.
