# Nota de Triagem

## Objetivo

Converter alertas brutos em uma nota curta, uniforme e fácil de assumir no turno seguinte.

## Casos de uso

- Alerta único de saturação, rejeição ou lag.
- Registro padronizado para handoff de plantão.
- Resposta rápida em incidentes recorrentes.

## Exemplo

Entrada:

```text
{{alerta_cru}}
```

Saída esperada:

```text
ALERTA: Relay - taxa de rejeição de ingestão acima de 2% por 5min
IMPACTO: ingestão de telemetry degradada para ~12% dos tenants
HIPÓTESE INICIAL: deploy do Relay às 09:14 reduziu o buffer de ingestão
AÇÃO IMEDIATA: rollback iniciado via Argo CD
ESCALAR PARA: @relay-core se a rejeição não cair em 10min
```

## Limitações

- Não inventa contexto fora do alerta.
- Assume que o alerta já está sanitizado quando houver dado sensível.
- Não produz narrativa longa; é uma nota operacional.

## Curadoria

A estrutura foi curada por exemplares e reforço de formato. O padrão funciona melhor com few-shot implícito, porque os cinco campos e a ordem são rígidos.