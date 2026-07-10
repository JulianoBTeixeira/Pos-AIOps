# Iterações da NetworkPolicy

## v1

- Permitiria tráfego demais.
- Usava allow-all implícito.
- Não tinha default-deny explícito.

## Verificação da v1

- Revisor de segurança apontou que `podSelector: {}` junto com `- {}` em ingress e egress equivalia a política aberta.
- Também faltavam comentários por fluxo legítimo.

## v2

- Separou a política default-deny de regras específicas.
- Limitou ingress ao Relay e ao API gateway.
- Limitou egress ao Forge, Cerebro e DNS interno.

## v3

- Refinou comentários por regra.
- Confirmou labels e portas com o mapa de serviços.
- Resultado final sem allow-all.
