# Questão 06 — Módulo Terraform no padrão interno

## Prompt

```markdown
# Context
A Hill Valley Tech usa Terraform na AWS e possui um padrão interno de compliance definido por Strickland:

- Tags obrigatórias em todo recurso: Owner, CostCenter, Environment.
- Prefixo `hvt-` nos nomes de recursos.
- Todo bucket S3 deve ter:
  - encryption habilitada, com SSE-S3 no mínimo;
  - versioning ativo;
  - block public access total;
  - logging configurado.
- Variáveis de entrada em `variables.tf` devem ter `description` e `type` obrigatórios.
- O módulo será reutilizado por vários times, então precisa incluir exemplo de uso.

# Action
Crie um módulo Terraform reutilizável para provisionar buckets S3 aderentes ao padrão interno da empresa.

# Result
Retorne a estrutura do módulo com os arquivos:
- `main.tf`
- `variables.tf`
- `outputs.tf`
- `examples/basic/main.tf`

O módulo deve:
- criar bucket com nome prefixado por `hvt-`;
- aplicar tags obrigatórias;
- habilitar versionamento;
- habilitar server-side encryption SSE-S3;
- bloquear completamente acesso público;
- configurar access logging para um bucket de logs informado por variável;
- expor outputs úteis: nome, ARN e região do bucket;
- seguir estilo claro e semelhante ao exemplo.

# Example
Referência de estilo do módulo de VPC já existente:

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

## Modelo

**GPT-4o — OpenAI.** Escolhido por boa geração de IaC estruturado, com múltiplos arquivos e aderência a padrões de naming.

## Output

### `main.tf`

```hcl
locals {
  bucket_name = "hvt-${var.bucket_name}-${var.environment}"

  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "this" {
  bucket = local.bucket_name

  tags = merge(local.common_tags, var.additional_tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.logging_bucket_name
  target_prefix = "${local.bucket_name}/"
}
```

### `variables.tf`

```hcl
variable "bucket_name" {
  description = "Nome lógico do bucket sem o prefixo hvt e sem o sufixo de ambiente. Exemplo: ledger-backups."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{1,50}[a-z0-9]$", var.bucket_name))
    error_message = "bucket_name deve usar letras minúsculas, números e hífens, sem iniciar ou terminar com hífen."
  }
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)."
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment deve ser dev, staging ou production."
  }
}

variable "owner" {
  description = "Responsável pelo recurso. Usado na tag obrigatória Owner."
  type        = string
}

variable "cost_center" {
  description = "Centro de custo do recurso. Usado na tag obrigatória CostCenter."
  type        = string
}

variable "logging_bucket_name" {
  description = "Nome do bucket S3 que receberá os access logs. Deve existir previamente."
  type        = string
}

variable "additional_tags" {
  description = "Tags adicionais opcionais para complementar as tags obrigatórias."
  type        = map(string)
  default     = {}
}
```

### `outputs.tf`

```hcl
output "bucket_name" {
  description = "Nome do bucket S3 criado."
  value       = aws_s3_bucket.this.bucket
}

output "bucket_arn" {
  description = "ARN do bucket S3 criado."
  value       = aws_s3_bucket.this.arn
}

output "bucket_region" {
  description = "Região do bucket S3 criado."
  value       = aws_s3_bucket.this.region
}
```

### `examples/basic/main.tf`

```hcl
module "ledger_backups_bucket" {
  source = "../../"

  bucket_name         = "ledger-backups"
  environment         = "production"
  owner               = "sre"
  cost_center         = "platform"
  logging_bucket_name = "hvt-s3-access-logs-production"

  additional_tags = {
    Application = "ledger"
    ManagedBy   = "terraform"
  }
}
```

## Justificativa

O **Context** apresenta o padrão interno de IaC, requisitos de compliance e restrições do módulo. A **Action** solicita a criação do módulo S3 reutilizável. O **Result** define arquivos, recursos obrigatórios e outputs esperados. O **Example** fornece o estilo do módulo VPC, incluindo `locals`, `common_tags` e `merge` de tags.
