# Validação das Questões

Validação feita por questão, separando cada artefato como uma tarefa. Quando a execução real dependia de infraestrutura externa, foi feita validação estática e o bloqueio foi registrado.

## Q01 - Dockerfile para o Lift

Status: validado parcialmente.

Tarefa executada:

- Extração do Dockerfile para `.validation/q01/Dockerfile`.
- Tentativa de `docker build`.
- Validação estática dos requisitos principais.

Resultado:

- O Docker CLI está instalado, mas o Docker daemon não estava em execução.
- Erro encontrado: `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`.
- Validação estática aprovada: imagem base versionada, usuário não-root, porta 8080, `CMD` com gunicorn e ausência de valores de segredo no Dockerfile.

## Q02 - Script de backup do Ledger

Status: validado e ajustado.

Tarefa executada:

- Extração do script Bash.
- Execução de `bash -n .validation/q02/ledger-backup.sh`.
- Revisão lógica do comportamento com `set -Eeuo pipefail`.

Resultado:

- Sintaxe Bash aprovada.
- Ajuste aplicado: o trecho de retenção no S3 agora tolera lista vazia de backups sem encerrar como falha.

## Q03 - Relatório de redução de custos cloud

Status: validado e ajustado.

Tarefa executada:

- Recálculo do custo total do CSV.
- Recálculo da meta de 15%.
- Conferência dos percentuais e ondas de economia.

Resultado:

- Total correto: US$ 41.800.
- Meta correta de 15%: US$ 6.270/mês.
- Ajuste aplicado no output da questão, que antes indicava US$ 41.700 e US$ 6.255.

## Q04 - Relatório mensal de transações do Ledger

Status: validado estaticamente.

Tarefa executada:

- Extração da query SQL.
- Verificação de disponibilidade de `psql`.
- Revisão estática da query PostgreSQL.

Resultado:

- `psql` não está instalado no ambiente.
- Query aprovada estaticamente: usa `DATE_TRUNC`, `TO_CHAR`, filtro por `completed`, recorte de seis meses, agrupamento por mês/categoria, conversão de centavos para reais e ordenação conforme pedido.

## Q05 - Modernizar deployment legado

Status: validado parcialmente.

Tarefa executada:

- Extração do YAML.
- Tentativa de `kubectl apply --dry-run=client --validate=false`.
- Parsing local com PyYAML.
- Checagem dos campos obrigatórios.

Resultado:

- `kubectl` tentou acessar API em `localhost:8080`, então a validação Kubernetes completa ficou bloqueada por ausência de cluster/contexto.
- Parsing YAML aprovado.
- Checagens aprovadas: `Deployment`, namespace `production`, 3 réplicas, imagem sem `latest`, secrets por `secretKeyRef`, resources, probes, securityContext não-root e `automountServiceAccountToken: false`.

## Q06 - Módulo Terraform no padrão interno

Status: validado estaticamente.

Tarefa executada:

- Extração dos arquivos `main.tf`, `variables.tf`, `outputs.tf` e `examples/basic/main.tf`.
- Verificação de disponibilidade de `terraform` e parser HCL.
- Checagem textual dos requisitos do enunciado.

Resultado:

- `terraform` não está instalado.
- Parser HCL Python não está instalado.
- Checagens aprovadas: prefixo `hvt-`, tags obrigatórias, versioning, SSE-S3, public access block, logging, variáveis com `description` e `type`, outputs e exemplo de uso.

## Q07 - Runbook para alerta recorrente

Status: validado e ajustado.

Tarefa executada:

- Verificação de comandos específicos.
- Verificação de critérios de escalação e encerramento.
- Revisão da etapa de Reactor/SQS.

Resultado:

- Ajuste aplicado: o runbook agora obtém a URL da fila `chronos-transactions` com `aws sqs get-queue-url` antes de consultar atributos.
- Checagens aprovadas: comandos `kubectl`, HPA, eventos, logs, SQS, rollback via Argo CD, escalação para `@chronos-core`, verificações esperadas e critério de encerramento.

## Q08 - Postmortem técnico de incidente em produção

Status: validado.

Tarefa executada:

- Verificação da escolha do framework.
- Verificação da recomendação explícita.
- Verificação das seções obrigatórias.
- Verificação da comparação com frameworks alternativos.

Resultado:

- Q08 aprovada.
- O output recomenda rollback, separa evidências e inferências, inclui linha do tempo, riscos, próximos passos e compara R-I-S-E com R-T-F, T-A-G e B-A-B.
