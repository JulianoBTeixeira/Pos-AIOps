# Questão 02 - Script de backup do Ledger

## Prompt

Você é uma pessoa SRE sênior responsável por rotinas confiáveis de backup em Linux, PostgreSQL e AWS.

Sua tarefa é criar um script Bash de produção para backup diário do PostgreSQL Ledger com estes parâmetros:

- Host: ledger-db.internal.hvt.io
- Porta: 5432
- Banco: ledger_prod
- Usuário: backup_user
- Senha: variável de ambiente PGPASSWORD, já populada via AWS Secrets Manager e IAM role da instância
- Região AWS: us-east-1
- SO: Ubuntu 22.04 LTS
- Diretório de trabalho: /var/backups/ledger, com 80 GB livres
- Tamanho médio do dump compactado: cerca de 12 GB
- Bucket S3: hvt-ledger-backups

O script deve executar `pg_dump`, compactar com `gzip`, enviar o arquivo para o S3 com `aws s3 cp`, manter retenção de 30 dias no S3 removendo backups antigos, registrar cada etapa em `/var/log/ledger-backup.log` com timestamp e sair com exit code adequado em caso de falha. O script deve validar pré-requisitos, evitar expor senha em log e usar nomes de arquivo com timestamp UTC.

Entregue somente o conteúdo do script em um bloco de código `bash`, pronto para salvar como `/usr/local/bin/ledger-backup.sh`.

## Modelo

GPT-4o. Escolhido por gerar scripts shell claros, defensivos e alinhados a operações Linux/AWS.

## Output

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"
AWS_REGION="us-east-1"
BACKUP_DIR="/var/backups/ledger"
S3_BUCKET="hvt-ledger-backups"
LOG_FILE="/var/log/ledger-backup.log"
RETENTION_DAYS=30

timestamp() {
  date -u +"%Y-%m-%dT%H:%M:%SZ"
}

log() {
  echo "$(timestamp) $*" >> "$LOG_FILE"
}

fail() {
  log "ERROR: $*"
  exit 1
}

require_command() {
  command -v "$1" >/dev/null 2>&1 || fail "Required command not found: $1"
}

main() {
  require_command pg_dump
  require_command gzip
  require_command aws
  require_command find

  [[ -n "${PGPASSWORD:-}" ]] || fail "PGPASSWORD is not set"
  [[ -d "$BACKUP_DIR" ]] || fail "Backup directory does not exist: $BACKUP_DIR"
  [[ -w "$BACKUP_DIR" ]] || fail "Backup directory is not writable: $BACKUP_DIR"

  local started_at
  local backup_name
  local dump_file
  local s3_uri

  started_at="$(date -u +"%Y%m%dT%H%M%SZ")"
  backup_name="ledger_prod_${started_at}.sql.gz"
  dump_file="${BACKUP_DIR}/${backup_name}"
  s3_uri="s3://${S3_BUCKET}/${backup_name}"

  log "INFO: Starting Ledger backup: ${backup_name}"

  if ! pg_dump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --format=plain \
    --no-owner \
    --no-privileges \
    | gzip -9 > "$dump_file"; then
    rm -f "$dump_file"
    fail "pg_dump or gzip failed"
  fi

  log "INFO: Dump created: ${dump_file}"

  aws s3 cp "$dump_file" "$s3_uri" --region "$AWS_REGION" \
    || fail "Failed to upload backup to ${s3_uri}"

  log "INFO: Backup uploaded to ${s3_uri}"

  local objects
  objects="$(aws s3 ls "s3://${S3_BUCKET}/" --region "$AWS_REGION" \
    | awk '{print $4}' \
    | grep '^ledger_prod_[0-9]\{8\}T[0-9]\{6\}Z\.sql\.gz$' || true)"

  while read -r object; do
    [[ -n "$object" ]] || continue
        object_date="${object#ledger_prod_}"
        object_date="${object_date%.sql.gz}"
        object_epoch="$(date -u -d "${object_date:0:8} ${object_date:9:2}:${object_date:11:2}:${object_date:13:2}" +%s)"
        cutoff_epoch="$(date -u -d "${RETENTION_DAYS} days ago" +%s)"
        if [[ "$object_epoch" -lt "$cutoff_epoch" ]]; then
          aws s3 rm "s3://${S3_BUCKET}/${object}" --region "$AWS_REGION" \
            && log "INFO: Removed expired S3 backup: ${object}"
        fi
  done <<< "$objects"

  find "$BACKUP_DIR" -type f -name 'ledger_prod_*.sql.gz' -mtime +2 -delete

  log "INFO: Ledger backup completed successfully"
}

main "$@"
```

## Justificativa

Role aparece ao posicionar a IA como SRE sênior responsável por backup em Linux, PostgreSQL e AWS. Task aparece na criação do script Bash com dump, gzip, upload, retenção, logs e exit codes. Format aparece na exigência de retornar somente um bloco `bash` pronto para uso.
