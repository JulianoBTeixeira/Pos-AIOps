# NetworkPolicy do Sentinel

## Objetivo

Endurecer a política de rede do namespace do Sentinel sem criar allow-all e sem inventar labels ou portas.

## Casos de uso

- Revisão de segurança de manifesto permissivo.
- Geração de política com comentários e default-deny.
- Iteração de verificação e refino antes de aprovar a saída.

## Exemplo

Entrada resumida:

```text
manifesto_permissivo={{manifesto_permissivo}}
mapeamento_servicos={{mapa_servicos}}
regras_politica={{regras_politica}}
```

Saída esperada em alto nível:

```yaml
kind: NetworkPolicy
policyTypes:
  - Ingress
  - Egress
```

Saída esperada em mais detalhe:

```yaml
# default-deny do namespace
kind: NetworkPolicy
metadata:
  name: sentinel-deny-all

# libera tráfego do Relay para o Sentinel
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: relay

# libera saída para Forge, Cerebro e DNS interno
egress:
  - to:
      - podSelector:
          matchLabels:
            app: forge
```

## Limitações

- Só deve usar o mapa de serviços fornecido.
- Não pode inventar seletores ou portas.
- Precisa registrar as iterações de verificação para justificar o refinamento.

## Curadoria

A técnica de verificação e refino foi usada porque um artefato de segurança precisa de crítica explícita da própria saída. O prompt é parametrizável e força feedback interno antes da versão final.
