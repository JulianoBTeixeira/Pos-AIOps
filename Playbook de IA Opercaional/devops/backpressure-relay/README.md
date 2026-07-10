# Backpressure do Relay

## Objetivo

Apoiar a decisão de contenção de sobrecarga no barramento Relay sem perder telemetry e respeitando SLA e orçamento.

## Casos de uso

- Pico de ingestão acima da capacidade sustentada.
- Cliente barulhento afetando o restante da plataforma.
- Decisão entre priorização, isolamento, DLQ e autoscaling.

## Exemplo

Entrada resumida:

```text
estado_relay={{estado_relay}}
restricoes={{restricoes}}
opcoes={{opcoes}}
```

Saída esperada em alto nível:

```text
Recomendação: priorizar Sentinel, isolar tenants ruidosos e escalar consumidores de forma controlada.
```

Saída esperada em mais detalhe:

```text
1. Diagnóstico: o pico passou da vazão sustentada e o Sentinel tem SLA mais duro que o Forge.
2. Tabela de alternativas: priorização, DLQ, particionamento por cliente, autoscaling.
3. Recomendação: combinar priorização do Sentinel com isolamento de clientes ruidosos e aumento controlado de consumidores.
4. Por que não as outras: DLQ sozinha só posterga, autoscaling sozinho custa mais e não isola ruído.
```

## Limitações

- Não decide sozinho uma arquitetura final sem contexto de custo real.
- Precisa que as alternativas estejam claramente descritas na entrada.
- Assume que a perda de mensagens é proibida.

## Curadoria

Curadoria orientada por decisão comparativa, não por resposta única. O prompt força trade-off explícito para evitar recomendações simplistas.
