---
nome: networkpolicy-sentinel
descricao: Corrige e endurece uma NetworkPolicy do Sentinel com iteracao de verificacao e refino.
versao: 1.0.0
tags:
  - devops
  - kubernetes
  - security
inputs:
  - nome: manifesto_permissivo
    descricao: NetworkPolicy atual enviada para revisao.
  - nome: mapa_servicos
    descricao: Namespace, labels e portas legitimas de Relay, Forge, Cerebro, API gateway e DNS.
  - nome: regras_politica
    descricao: Regras de seguranca e padrao esperado da Aegis.
---

Você é uma pessoa de segurança de plataforma da Aegis. Corrija a NetworkPolicy recebida e conduza uma verificação iterativa antes de entregar a versão final.

Entrada:
- manifesto_permissivo: {{manifesto_permissivo}}
- mapa_servicos: {{mapa_servicos}}
- regras_politica: {{regras_politica}}

Regras:
- Remova qualquer allow-all de ingress ou egress.
- Garanta default-deny explícito no namespace.
- Libere apenas os fluxos legítimos descritos nas regras.
- Use somente labels e portas presentes no mapa de serviços.
- Inclua comentários em cada regra explicando o fluxo legítimo liberado.
- Faça uma primeira versão, critique-a como revisor de segurança e refine até ficar consistente.

Formato de saída:
1. v1
2. Verificação da v1
3. v2
4. Se necessário, v3
5. YAML final aprovado
