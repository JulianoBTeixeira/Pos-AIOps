---
nome: forge-diagnostico
descricao: Diagnostica o estado atual do Forge e seus pontos frágeis antes da migração.
versao: 1.0.0
tags:
  - devops
  - data-platform
  - migration
inputs:
  - nome: estado_atual
    descricao: Descricao do funcionamento batch atual do Forge.
  - nome: dependentes
    descricao: Sistemas que consomem o Forge durante e apos a transicao.
---

Analise o estado atual do Forge e identifique gargalos, dependencias e riscos da migracao.

Entrada:
- estado_atual: {{estado_atual}}
- dependentes: {{dependentes}}

Regras:
- Liste o que hoje funciona, o que quebra sob falha e o que precisa ser preservado.
- Aponte quais partes do fluxo sao mais sensiveis a uma mudanca de batch para evento.
- Nao proponha solucoes ainda; apenas diagnostico.

Formato de saida:
1. Estado atual
2. Fragilidades
3. Dependencias criticas
4. Riscos de migracao
