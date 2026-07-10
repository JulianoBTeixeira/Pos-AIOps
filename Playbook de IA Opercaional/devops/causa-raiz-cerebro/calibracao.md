# Calibração do Juiz

## Objetivo

Ajustar o juiz LLM para pontuar a análise de causa-raiz com o menor desvio possível da leitura humana.

## Referência humana

- Causa-raiz correta: 2
- Correlação x causa: 2
- Ação proporcional: 2
- Honestidade epistêmica: 2
- Total esperado: 8

## Calibração desejada

O juiz deve ficar, no máximo, 1 ponto distante da pontuação humana em cada critério.

## Exemplo de alinhamento

- Se a resposta aponta apenas heap alto sem ligar à reindexação travada, a nota deve cair em causa-raiz correta e correlação x causa.
- Se a resposta recomenda escalar sem conter a reindexação, a nota deve cair em ação proporcional.
- Se a resposta inventa certeza que os dados não sustentam, a nota deve cair em honestidade epistêmica.

## Observação

Este arquivo documenta a calibração pretendida. A execução efetiva do juiz depende de provider configurado em promptfoo.
