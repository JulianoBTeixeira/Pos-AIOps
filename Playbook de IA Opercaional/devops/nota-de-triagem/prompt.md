---
nome: nota-de-triagem
descricao: Gera uma nota padronizada de triagem a partir de um alerta cru.
versao: 1.0.0
tags:
  - devops
  - incident-response
  - scribe
inputs:
  - nome: alerta_cru
    descricao: Texto bruto do alerta emitido pelo Sentinel, Relay, Forge ou Cerebro.
---

Você é uma pessoa plantonista da Aegis. Transforme o alerta bruto em uma nota padronizada, concisa e reutilizável.

Entrada:
- alerta_cru: {{alerta_cru}}

Regras:
- Produza exatamente os cinco campos: ALERTA, IMPACTO, HIPÓTESE INICIAL, AÇÃO IMEDIATA, ESCALAR PARA.
- Mantenha a nota objetiva, com no máximo 8 linhas.
- O campo de escalonamento deve conter um handle no formato @palavra.
- Não explique o processo; escreva apenas a nota final.
- Se houver mais de uma leitura possível, escolha a hipótese mais provável com base no alerta.

Formato de saída:
ALERTA: ...
IMPACTO: ...
HIPÓTESE INICIAL: ...
AÇÃO IMEDIATA: ...
ESCALAR PARA: @..."}},{"recipient_name":"functions.create_file","parameters":{"filePath":"d:\Juliano\Treinamentos\POS\Pós AIOps e IA na Engenharia de Cloud\Desafios\Frameworks Prompt Engineering\Playbook de IA Opercaional\devops\nota-de-triagem\README.md","content":"# Nota de Triagem\n\n## Objetivo\n\nConverter alertas brutos em uma nota curta, uniforme e fácil de assumir no turno seguinte.\n\n## Casos de uso\n\n- Alerta único de saturação, rejeição ou lag.\n- Registro padronizado para handoff de plantão.\n- Resposta rápida em incidentes recorrentes.\n\n## Exemplo\n\nEntrada:\n\n```text\n{{alerta_cru}}\n```\n\nSaída esperada:\n\n```text\nALERTA: Relay - taxa de rejeição de ingestão acima de 2% por 5min\nIMPACTO: ingestão de telemetry degradada para ~12% dos tenants\nHIPÓTESE INICIAL: deploy do Relay às 09:14 reduziu o buffer de ingestão\nAÇÃO IMEDIATA: rollback iniciado via Argo CD\nESCALAR PARA: @relay-core se a rejeição não cair em 10min\n```\n\n## Limitações\n\n- Não inventa contexto fora do alerta.\n- Assume que o alerta já está sanitizado quando houver dado sensível.\n- Não produz narrativa longa; é uma nota operacional.\n\n## Curadoria\n\nA estrutura foi curada por exemplares e reforço de formato. O padrão funciona melhor com few-shot interno implícito, porque os cinco campos e a ordem são rígidos."}}]}]}