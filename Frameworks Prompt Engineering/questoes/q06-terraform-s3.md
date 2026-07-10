# Questão 06 - Módulo Terraform no padrão interno

## Prompt

Contexto: a Hill Valley Tech tem um padrão interno de IaC definido por segurança e compliance. Todo recurso deve ter tags `Owner`, `CostCenter` e `Environment`; nomes de recursos devem usar prefixo `hvt-`; buckets S3 devem ter encryption SSE-S3 no mínimo, versioning ativo, bloqueio total de acesso público e logging configurado. Variáveis de entrada devem estar em `variables.tf` com `description` e `type`.

Ação: crie um módulo Terraform reutilizável para provisionar buckets S3 aderentes a esse padrão. O módulo deve conter `main.tf`, `variables.tf`, `outputs.tf` e um exemplo de uso. Use AWS provider moderno e recursos separados quando apropriado.

Resultado esperado: código Terraform claro, reutilizável por múltiplos times, com nomes padronizados, tags comuns, versionamento, criptografia, public access block, logging e outputs úteis. Não use valores sensíveis hardcoded.

Exemplo de estilo interno:

```hcl
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

Entregue a resposta em seções com os caminhos dos arquivos e blocos `hcl` para cada arquivo.

## Modelo

GPT-4o. Escolhido por boa geração de módulos Terraform com estrutura repetível e legível.

## Output

### main.tf

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

  tags = merge(local.common_tags, {
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
  target_prefix = "s3/${local.bucket_name}/"
}
```

### variables.tf

```hcl
variable "bucket_name" {
  description = "Nome lógico do bucket sem o prefixo hvt e sem o ambiente"
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

variable "owner" {
  description = "Time ou pessoa responsável pelo bucket"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo associado ao bucket"
  type        = string
}

variable "logging_bucket_name" {
  description = "Nome do bucket S3 que receberá logs de acesso"
  type        = string
}
```

### outputs.tf

```hcl
output "bucket_name" {
  description = "Nome do bucket criado"
  value       = aws_s3_bucket.this.bucket
}

output "bucket_arn" {
  description = "ARN do bucket criado"
  value       = aws_s3_bucket.this.arn
}

output "bucket_id" {
  description = "ID do bucket criado"
  value       = aws_s3_bucket.this.id
}
```

### examples/basic/main.tf

```hcl
module "ledger_exports_bucket" {
  source = "../../"

  bucket_name         = "ledger-exports"
  environment         = "production"
  owner               = "sre"
  cost_center         = "platform"
  logging_bucket_name = "hvt-s3-access-logs-production"
}
```

## Justificativa

Context aparece nas regras internas de compliance e no padrão de estilo existente. Action aparece ao pedir a criação do módulo reutilizável com arquivos específicos. Result aparece na definição do resultado esperado, incluindo segurança, tags, versionamento, logging e outputs. Example aparece no trecho do módulo VPC usado como referência de estilo.
