# Cadeia de Migração do Forge

## Objetivo

Migrar o Forge de batch para evento contínuo sem big-bang, mantendo dependentes funcionando durante a transição.

## Cadeia

1. `01-diagnostico.md` - entende o estado atual e os pontos frágeis.
2. `02-estrategia.md` - transforma o diagnóstico em estratégia de migração faseada.
3. `03-plano-executavel.md` - converte a estratégia em plano reversível e acionável.

## Saída esperada por elo

- Elo 1: lista de fragilidades do batch atual e dependências críticas.
- Elo 2: fases pequenas, coexistência batch/eventos e pontos de rollback.
- Elo 3: plano executável com etapas, responsáveis e critérios de conclusão.

## Curadoria

A cadeia foi desenhada para reduzir a carga cognitiva do modelo. Em vez de um prompt monolítico, cada etapa estreita o espaço da resposta e passa contexto útil para a seguinte.
