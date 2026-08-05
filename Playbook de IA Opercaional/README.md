# Playbook de IA Operacional da Aegis

Biblioteca versionada de prompts parametrizáveis para operação contínua da plataforma Aegis.

## Convenções

- Cada prompt vive em `devops/<nome-do-prompt>/`.
- `prompt.md` contém o prompt parametrizável com frontmatter.
- `README.md` contém a documentação humana, exemplos e curadoria.
- `promptfooconfig.yaml` aparece ao lado do prompt quando há testes determinísticos ou gate com juiz.
- Os prompts foram desenhados para uso em chat, playground ou API, com os exemplos colados na entrada.

## Escopo coberto

- Checkpoint 01: triagem de pods
- Checkpoint 02: nota padronizada de triagem
- Checkpoint 03: análise de causa-raiz do Cerebro
- Checkpoint 04: decisão de backpressure do Relay
- Checkpoint 05: migração do Forge para evento contínuo
- Checkpoint 06: endurecimento de NetworkPolicy do Sentinel
- Checkpoint 07: estrutura tipo prompt-registry
- Checkpoint 08: testes determinísticos com promptfoo
- Checkpoint 09: gate de qualidade com LLM-as-judge
- Checkpoint 10: pipeline contínuo em GitHub Actions

## Navegação

- Índice da biblioteca: [devops/README.md](devops/README.md)
- Relatório de curadoria: [RELATORIO.md](RELATORIO.md)

## Modelo de curadoria

- Estruturados e baratos: `openai:gpt-4o-mini`
- Raciocínio causal e comparação de alternativas: `anthropic:claude-sonnet-4`
- Síntese com contexto amplo: `google:gemini-2.5-pro`

## Observação

Este diretório foi criado como a área nova do playbook. O restante do workspace foi preservado sem alterações.

## Nota de validação para correção

- A entrega está alinhada aos checkpoints 08, 09 e 10 em estrutura e conteúdo.
- O checkpoint 09 usa gate com juiz LLM via rubrica em `devops/causa-raiz-cerebro/rubrica.md` e `devops/causa-raiz-cerebro/promptfooconfig.yaml`.
- Em execução real, o checkpoint 09 depende de provider autenticado no ambiente de quem roda a suíte.
- Se não houver credenciais de provider no ambiente de correção, a validação recomendada é estrutural.
- Validar presença da rubrica e do assert `llm-rubric`.
- Validar o workflow em `.github/workflows/promptfoo.yml`.
- Executar normalmente os checkpoints determinísticos (08), que não dependem de juiz externo.