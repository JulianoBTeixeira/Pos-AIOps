# Decisão de estrutura do repositório

A estrutura escolhida separa cada questão em um arquivo Markdown independente dentro de `questoes/`.

## Justificativa

1. **Rastreabilidade:** cada questão pode evoluir separadamente, facilitando versionamento por commit.
2. **Leitura acadêmica:** o avaliador consegue abrir a questão exata sem navegar por um documento longo.
3. **Reuso profissional:** cada arquivo funciona como exemplo reutilizável de framework aplicado a Cloud, SRE, DevOps, FinOps ou IaC.
4. **Escalabilidade:** novas versões dos prompts ou novos outputs podem ser adicionados posteriormente sem alterar a estrutura base.

## Convenção adotada

Cada arquivo segue o mesmo contrato:

```markdown
# Questão XX — Título

## Prompt
## Modelo
## Output
## Justificativa
```

A Q08 é a única exceção parcial, pois inclui uma justificativa estendida e comparação entre frameworks.
