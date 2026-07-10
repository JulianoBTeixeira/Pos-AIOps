# Questão 03 - Relatório de redução de custos cloud

## Prompt

Analise o CSV abaixo e produza um relatório executivo de oportunidades de redução de custos cloud para a Hill Valley Tech.

Primeiro, calcule o custo mensal total. Depois, identifique oportunidades priorizadas por impacto financeiro, estime quanto cada uma representa em percentual da conta total, classifique o esforço de implementação como baixo, médio ou alto e descreva riscos ou pré-requisitos. Considere que a meta do próximo trimestre é reduzir 15% do custo cloud sem degradar SLA.

CSV:

```csv
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
```

O objetivo é entregar para Goldie, CEO da empresa, uma recomendação objetiva para atingir a meta de 15% com o menor risco operacional possível. Formate a resposta com: resumo executivo, tabela priorizada, plano sugerido em ondas e observações de risco.

## Modelo

Claude Sonnet 4. Escolhido por boa qualidade em análise executiva, priorização e síntese de trade-offs.

## Output

### Resumo executivo

O custo mensal total analisado é de **US$ 41.800**. A meta de redução de 15% equivale a **US$ 6.270/mês**. Há oportunidades suficientes para atingir a meta sem mexer primeiro nos componentes de maior risco, desde que as ações sejam feitas com validação de capacidade e monitoramento de SLA.

### Oportunidades priorizadas

| Prioridade | Oportunidade | Economia estimada | % da conta total | Esforço | Riscos e pré-requisitos |
|---|---:|---:|---:|---|---|
| 1 | Reduzir EC2 on-demand subutilizada e migrar cargas previsíveis para Savings Plans/Reserved Instances | US$ 2.460 | 5,9% | Médio | Requer análise por workload; risco de comprometer picos se remover capacidade sem observar sazonalidade. |
| 2 | Otimizar EKS: rightsizing de node groups, ajuste de requests/limits e consolidação dos 3 clusters | US$ 1.675 | 4,0% | Alto | Exige validação por namespace, HPA e workloads críticos; risco de pressão de CPU/memória se requests estiverem incorretos. |
| 3 | Revisar RDS PostgreSQL Multi-AZ, tamanho da instância e storage | US$ 1.230 | 2,9% | Médio | Não remover Multi-AZ sem análise de SLA; pré-requisito de métricas de CPU, IOPS, conexões e janela de manutenção. |
| 4 | Reduzir retenção e volume de CloudWatch Logs de 90 para 30/45 dias com exportação seletiva para S3 | US$ 1.120 | 2,7% | Baixo | Confirmar exigências de auditoria; manter logs críticos de segurança pelo prazo obrigatório. |
| 5 | Aplicar lifecycle em S3 Standard para IA/Glacier em dados frios | US$ 775 | 1,9% | Baixo | Mapear padrões de acesso; risco de aumento de latência/custo de restore para dados consultados com frequência. |
| 6 | ElastiCache rightsizing | US$ 525 | 1,3% | Médio | Validar memória usada, evictions e conexões; risco de degradação se cache estiver absorvendo picos. |
| 7 | Reduzir NAT Gateway e Data Transfer Out com endpoints privados e revisão de tráfego entre regiões | US$ 620 | 1,5% | Médio | Depende de análise de rotas; risco de indisponibilidade se endpoints forem aplicados sem teste. |

### Plano em ondas

**Onda 1 - Baixo risco e retorno rápido:** CloudWatch Logs, lifecycle de S3 e análise de EC2 on-demand. Potencial aproximado: **US$ 4.355/mês**, ou **10,4%**.

**Onda 2 - Rightsizing controlado:** EKS, RDS e ElastiCache com baseline de métricas e mudanças graduais. Potencial acumulado aproximado: **US$ 7.785/mês**, ou **18,6%**.

**Onda 3 - Otimização de rede:** NAT Gateway, endpoints privados e tráfego entre regiões. Usar após mapear dependências, porque erros de rota podem afetar produção.

### Recomendação

Para atingir 15% sem degradar SLA, priorizar Onda 1 inteira e selecionar ações de EKS/RDS com menor risco após validação de métricas. A meta é viável sem remover resiliência crítica, especialmente sem desligar Multi-AZ de produção como primeira medida.

## Justificativa

Task aparece no pedido para analisar o CSV e produzir relatório executivo de redução de custos. Action aparece nos passos explícitos de calcular total, priorizar oportunidades, estimar percentual, esforço e riscos. Goal aparece na meta de reduzir 15% no trimestre sem degradar SLA e entregar recomendação objetiva para a CEO.
