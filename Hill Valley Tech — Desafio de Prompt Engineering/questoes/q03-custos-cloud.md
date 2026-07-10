# Questão 03 — Relatório de redução de custos cloud

## Prompt

```markdown
# Task
Produza um relatório executivo de oportunidades de redução de custos AWS para a Hill Valley Tech, usando o CSV abaixo como fonte de dados.

servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes

# Action
Calcule o custo total mensal, identifique oportunidades de economia priorizadas por impacto e, para cada oportunidade, informe:
- serviço afetado;
- hipótese de otimização;
- economia estimada em USD;
- percentual da conta total;
- esforço de implementação: baixo, médio ou alto;
- riscos, dependências ou pré-requisitos;
- impacto esperado sobre SLA.

Use premissas explícitas quando a economia exata não puder ser calculada diretamente. Não recomende desligamento ou redução de capacidade sem validação de métricas.

# Goal
Gerar um relatório para Goldie Wilson que permita decidir um plano de redução de 15% no custo cloud no próximo trimestre, sem degradar SLA. O relatório deve mostrar se a meta de 15% é atingível e quais iniciativas deveriam ser priorizadas primeiro.
```

## Modelo

**Gemini 2.5 Pro — Google.** Escolhido por boa capacidade de análise tabular, síntese executiva e raciocínio orientado a metas.

## Output

# Relatório executivo — Redução de custos AWS

## 1. Resumo executivo

O custo mensal analisado é de **US$ 41.700**. A meta de redução de **15%** equivale a **US$ 6.255/mês**.

Com base no CSV, a meta é **atingível**, desde que a empresa combine ações de baixo risco e validação operacional. O plano recomendado prioriza otimização de EC2 on-demand, EKS, RDS, CloudWatch Logs e ElastiCache, pois esses itens concentram alto custo e/ou baixa utilização média.

## 2. Conta total por categoria

| Categoria | Custo mensal | % da conta |
|---|---:|---:|
| Compute | US$ 20.000 | 47,96% |
| Databases | US$ 10.300 | 24,70% |
| Storage | US$ 4.700 | 11,27% |
| Observability | US$ 3.700 | 8,87% |
| Network | US$ 3.100 | 7,43% |
| **Total** | **US$ 41.700** | **100%** |

## 3. Oportunidades priorizadas

| Prioridade | Serviço | Hipótese de otimização | Economia estimada | % da conta | Esforço | Riscos / pré-requisitos | Impacto em SLA |
|---:|---|---|---:|---:|---|---|---|
| 1 | EC2 on-demand | Rightsizing e migração parcial para Savings Plans/Spot em workloads variáveis com uso médio de 45% | US$ 2.460 | 5,90% | Médio | Validar sazonalidade, workloads stateful e tolerância a interrupção | Baixo se aplicado gradualmente |
| 2 | EKS | Rightsizing de nodes, revisão de requests/limits e consolidação dos 3 clusters | US$ 1.340 | 3,21% | Alto | Exige análise de bin packing, HPA, PDBs e isolamento entre ambientes | Médio se consolidação for agressiva |
| 3 | RDS PostgreSQL | Rightsizing controlado, revisão de storage/IOPS e análise de instância Multi-AZ | US$ 820 | 1,97% | Médio | Validar CPU, memória, conexões, IOPS, janela de manutenção e failover | Médio; requer teste de carga |
| 4 | CloudWatch Logs | Reduzir retenção de 90 dias para 30/45 dias e mover histórico para S3 Glacier | US$ 1.120 | 2,69% | Baixo | Validar requisitos de compliance e troubleshooting histórico | Baixo |
| 5 | ElastiCache Redis | Rightsizing ou ajuste de nós em cluster com uso médio de 40% | US$ 630 | 1,51% | Médio | Validar memória usada, eviction, latência e failover | Médio |
| 6 | S3 Standard | Lifecycle para IA/Glacier em dados frios dos 5 buckets principais | US$ 465 | 1,12% | Baixo | Classificação de dados e política de retenção | Baixo |
| 7 | NAT Gateway | Avaliar endpoints privados para S3/ECR/CloudWatch e reduzir tráfego via NAT | US$ 360 | 0,86% | Médio | Mapear fluxos de rede e rotas | Baixo |
| 8 | Lambda | Ajustar memória e timeout para funções com baixa utilização | US$ 180 | 0,43% | Baixo | Exige profiling por função | Baixo |

## 4. Soma potencial

As 5 primeiras iniciativas somam aproximadamente:

**US$ 6.370/mês**, equivalente a **15,28%** da conta total.

Isso atinge a meta de 15%, com uma margem pequena de **US$ 115/mês** acima do alvo. Para reduzir risco, recomenda-se manter S3 Lifecycle e NAT Gateway como iniciativas de contingência, elevando o potencial total para aproximadamente **17,26%**.

## 5. Plano recomendado para o trimestre

### Fase 1 — Baixo risco e retorno rápido
- Reduzir retenção de CloudWatch Logs.
- Implementar lifecycle em S3.
- Fazer tuning de Lambda.

### Fase 2 — Otimização com validação técnica
- Rightsizing de EC2 on-demand.
- Revisão de ElastiCache.
- Redução de custos de NAT Gateway com VPC endpoints.

### Fase 3 — Maior impacto com maior controle
- Otimizar EKS e RDS PostgreSQL após análise de métricas, teste de carga e validação de SLA.

## 6. Recomendação final

A meta de 15% é realista, mas não deve ser buscada com cortes cegos. O caminho recomendado é combinar FinOps com SRE: reduzir desperdício observável, preservar capacidade crítica e validar cada mudança contra métricas de disponibilidade, latência e erro.

## Justificativa

A **Task** solicita explicitamente um relatório executivo de redução de custos baseado no CSV. A **Action** define os cálculos, campos e critérios de priorização que o modelo deve aplicar. O **Goal** amarra a resposta à meta de negócio: reduzir 15% no trimestre sem degradar SLA.
