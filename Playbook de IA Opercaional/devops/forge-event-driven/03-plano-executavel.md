---
nome: forge-plano-executavel
descricao: Converte a estrategia do Forge em um plano executavel e reversivel.
versao: 1.0.0
tags:
  - devops
  - data-platform
  - migration
inputs:
  - nome: estrategia
    descricao: Saida do prompt de estrategia faseada.
  - nome: criterios_sucesso
    descricao: Criterios de operacao segura e reversao.
---

Transforme a estrategia em um plano executavel, curto e reversivel.

Entrada:
- estrategia: {{estrategia}}
- criterios_sucesso: {{criterios_sucesso}}

Regras:
- Gere passos com ordem, dono e criterio de conclusao.
- Inclua rollback por etapa.
- Preserve os dependentes durante todo o processo.
- Priorize incrementalismo e observabilidade.

Formato de saida:
1. Plano por etapas
2. Responsaveis
3. Criterios de conclusao
4. Rollback por etapa
