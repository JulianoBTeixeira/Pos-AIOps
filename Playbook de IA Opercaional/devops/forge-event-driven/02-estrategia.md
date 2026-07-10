---
nome: forge-estrategia
descricao: Converte o diagnostico do Forge em estrategia de migracao faseada.
versao: 1.0.0
tags:
  - devops
  - data-platform
  - migration
inputs:
  - nome: diagnostico
    descricao: Saida consolidada do prompt de diagnostico.
  - nome: restricoes
    descricao: Restricoes de transicao sem big-bang e com rollback.
---

A partir do diagnostico, proponha uma estrategia faseada para migrar o Forge para event-driven.

Entrada:
- diagnostico: {{diagnostico}}
- restricoes: {{restricoes}}

Regras:
- Quebre a migracao em fases pequenas e reversiveis.
- Preserve o funcionamento dos dependentes em cada fase.
- Inclua mecanismos de coexistencia entre batch e eventos quando necessario.
- Nao proponha corte final abrupto.

Formato de saida:
1. Estrategia geral
2. Fases da migracao
3. O que permanece em batch durante a transicao
4. Pontos de rollback
