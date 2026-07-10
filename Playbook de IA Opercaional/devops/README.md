# Índice DevOps

## Convenções

- Um prompt por pasta.
- `prompt.md` contém o prompt parametrizável.
- `README.md` documenta objetivo, uso, exemplo e limitações.
- `promptfooconfig.yaml` vive ao lado do prompt quando houver teste.
- Versões começam em `1.0.0` e evoluem por mudança semântica.

## Prompts

- `triagem-de-pods` - triagem de pods do Sentinel
- `nota-de-triagem` - padronização de notas de alerta
- `causa-raiz-cerebro` - análise causal de degradação no Cerebro
- `backpressure-relay` - decisão de backpressure para o Relay
- `forge-event-driven` - cadeia de migração do Forge
- `networkpolicy-sentinel` - endurecimento de NetworkPolicy

## Exemplo canônico

O prompt `triagem-de-pods` é o exemplo completo do padrão do registry: contém `prompt.md` parametrizável, `README.md` documentado e `promptfooconfig.yaml` para teste.
