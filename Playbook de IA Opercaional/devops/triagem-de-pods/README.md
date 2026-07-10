# Triagem de Pods

## Objetivo

Triar rapidamente a saúde dos pods do Sentinel e apontar causa provável e próxima ação.

## Casos de uso

- CrashLoopBackOff por memória.
- ImagePullBackOff por imagem ausente.
- Pending por falta de capacidade do cluster.
- Ambiente saudável sem anomalias relevantes.

## Exemplo

Entrada:

```text
snapshot_pods={{snapshot_pods}}
eventos_pod={{eventos_pod}}
logs_pod={{logs_pod}}
```

Saída esperada:

```text
- Pod: sentinel-api-7d9c8b6f4-h4m2t
  Estado: CrashLoopBackOff
  Causa provável: OOMKilled; os logs mostram heap no limite e shutdown por falta de memória.
  Próxima ação: aumentar memória ou revisar consumo/cache.
```

## Limitações

- Não consulta o cluster por conta própria.
- Depende de snapshot, eventos e logs completos.
- Não substitui investigação profunda quando há causas concorrentes.

## Curadoria

A estrutura é curta porque o uso esperado é plantão. O prompt privilegia correlação causal, não repetição de status.