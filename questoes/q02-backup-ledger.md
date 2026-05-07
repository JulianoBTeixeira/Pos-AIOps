# Questão 02 — Script de backup do Ledger

## Prompt

```markdown
# Role
Você é um DBA sênior e engenheiro SRE especializado em PostgreSQL, Linux, AWS CLI, automação com Bash e rotinas de backup em produção.

# Task
Crie um script Bash robusto para backup diário do banco PostgreSQL Ledger, com as seguintes características do ambiente:

- Host: ledger-db.internal.hvt.io
- Porta: 5432
- Banco: ledger_prod
- Usuário de backup: backup_user
- Senha: variável de ambiente PGPASSWORD, populada pelo AWS Secrets Manager via IAM role da instância
- Região AWS: us-east-1
- SO: Ubuntu 22.04 LTS
- Diretório de trabalho: /var/backups/ledger
- Espaço disponível: 80 GB
- Tamanho médio do dump compactado: ~12 GB
- Bucket S3: hvt-ledger-backups

O script deve:
- Executar `pg_dump`.
- Compactar com gzip.
- Fazer upload para S3 usando `aws s3 cp`.
- Manter retenção de 30 dias no S3, removendo backups mais antigos.
- Registrar logs em `/var/log/ledger-backup.log` com timestamp.
- Validar pré-requisitos: PGPASSWORD, pg_dump, gzip, aws cli e diretório local.
- Usar `set -Eeuo pipefail`.
- Retornar exit code adequado em caso de falha.
- Evitar expor senha em log.

# Format
Retorne:
1. Script Bash completo, comentado e pronto para salvar como `/usr/local/bin/ledger-backup.sh`.
2. Exemplo de entrada no cron para execução diária às 02:00 UTC.
3. Observações operacionais de segurança e monitoramento em até 5 bullets.
```

## Modelo

**Claude Sonnet 4 — Anthropic.** Escolhido pela boa aderência a tarefas longas com requisitos operacionais, tratamento de erros e documentação inline.

## Output

### `/usr/local/bin/ledger-backup.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"
AWS_REGION="us-east-1"
BACKUP_DIR="/var/backups/ledger"
LOG_FILE="/var/log/ledger-backup.log"
S3_BUCKET="hvt-ledger-backups"
S3_PREFIX="ledger_prod"
RETENTION_DAYS="30"
TIMESTAMP="$(date -u +%Y%m%dT%H%M%SZ)"
BACKUP_BASE="${DB_NAME}_${TIMESTAMP}"
DUMP_FILE="${BACKUP_DIR}/${BACKUP_BASE}.sql"
GZIP_FILE="${DUMP_FILE}.gz"
S3_URI="s3://${S3_BUCKET}/${S3_PREFIX}/${BACKUP_BASE}.sql.gz"

log() {
  echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') [ledger-backup] $*" | tee -a "$LOG_FILE"
}

fail() {
  log "ERROR: $*"
  exit 1
}

cleanup_partial() {
  local exit_code=$?
  if [[ $exit_code -ne 0 ]]; then
    log "ERROR: backup failed with exit code ${exit_code}. Removing partial local files."
    rm -f "$DUMP_FILE" "$GZIP_FILE"
  fi
  exit "$exit_code"
}

trap cleanup_partial EXIT

validate_prereqs() {
  [[ -n "${PGPASSWORD:-}" ]] || fail "PGPASSWORD is not set"
  command -v pg_dump >/dev/null 2>&1 || fail "pg_dump not found"
  command -v gzip >/dev/null 2>&1 || fail "gzip not found"
  command -v aws >/dev/null 2>&1 || fail "aws cli not found"
  mkdir -p "$BACKUP_DIR" || fail "cannot create backup directory: $BACKUP_DIR"
  [[ -w "$BACKUP_DIR" ]] || fail "backup directory is not writable: $BACKUP_DIR"
  [[ -w "$(dirname "$LOG_FILE")" ]] || fail "log directory is not writable: $(dirname "$LOG_FILE")"
}

run_backup() {
  log "Starting PostgreSQL backup for ${DB_NAME} at ${DB_HOST}:${DB_PORT}"

  pg_dump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --format=plain \
    --no-owner \
    --no-privileges \
    --file="$DUMP_FILE"

  log "pg_dump completed: ${DUMP_FILE}"

  gzip -9 "$DUMP_FILE"
  log "Compression completed: ${GZIP_FILE}"

  aws s3 cp "$GZIP_FILE" "$S3_URI" \
    --region "$AWS_REGION" \
    --only-show-errors

  log "Upload completed: ${S3_URI}"
}

apply_s3_retention() {
  local cutoff_epoch
  cutoff_epoch="$(date -u -d "${RETENTION_DAYS} days ago" +%s)"

  log "Applying S3 retention policy: removing backups older than ${RETENTION_DAYS} days"

  aws s3 ls "s3://${S3_BUCKET}/${S3_PREFIX}/" --region "$AWS_REGION" | while read -r file_date file_time file_size file_name; do
    [[ -n "${file_name:-}" ]] || continue

    local file_epoch
    file_epoch="$(date -u -d "${file_date} ${file_time}" +%s)"

    if [[ "$file_epoch" -lt "$cutoff_epoch" ]]; then
      log "Removing old backup from S3: ${file_name} (${file_size} bytes)"
      aws s3 rm "s3://${S3_BUCKET}/${S3_PREFIX}/${file_name}" \
        --region "$AWS_REGION" \
        --only-show-errors
    fi
  done
}

main() {
  validate_prereqs
  run_backup
  apply_s3_retention
  rm -f "$GZIP_FILE"
  log "Backup finished successfully"
}

main "$@"
```

### Cron diário às 02:00 UTC

```cron
0 2 * * * /usr/local/bin/ledger-backup.sh >> /var/log/ledger-backup.log 2>&1
```

### Observações operacionais

- A senha não é escrita em arquivo nem log; o script depende de `PGPASSWORD` populado externamente.
- O bucket deve ter criptografia, versionamento e policy de acesso restrita à IAM role da instância.
- O ideal é complementar a retenção no script com Lifecycle Policy no S3 para defesa em profundidade.
- O log deve ser coletado pelo agente de observabilidade para alertar falhas de backup.
- Recomenda-se teste periódico de restore, não apenas validação de geração do dump.

## Justificativa

O **Role** posiciona o modelo como DBA/SRE com domínio de PostgreSQL, Bash e AWS. A **Task** detalha o ambiente, requisitos funcionais e requisitos de robustez. O **Format** exige script completo, entrada de cron e observações operacionais, evitando uma resposta genérica.
