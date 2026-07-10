# Questão 01 - Dockerfile para o Lift

## Prompt

Você é uma pessoa engenheira DevOps sênior especialista em Docker, Python, Flask e Kubernetes.

Sua tarefa é criar um Dockerfile de produção para a aplicação Lift, uma API Python/Flask que escuta na porta 8080. O projeto tem esta estrutura:

```text
lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py
```

O arquivo requirements.txt contém:

```text
Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9
```

Em produção o serviço deve subir com:

```bash
gunicorn --bind 0.0.0.0:8080 --workers 4 app:app
```

O runtime precisa receber as variáveis de ambiente DATABASE_URL e API_KEY, mas seus valores não devem ser definidos no Dockerfile. Siga boas práticas: imagem base pequena e versionada, usuário não-root, cache eficiente de dependências, cópia somente do necessário, exposição da porta correta e comando de produção.

Entregue somente o conteúdo do Dockerfile em um bloco de código `Dockerfile`, sem explicações fora do bloco.

## Modelo

GPT-4o. Escolhido por bom desempenho em geração objetiva de artefatos DevOps e aderência a restrições de formato.

## Output

```Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

RUN addgroup --system appgroup \
    && adduser --system --ingroup appgroup appuser

COPY requirements.txt .

RUN pip install --upgrade pip \
    && pip install -r requirements.txt

COPY app.py .
COPY lib ./lib

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

## Justificativa

Role aparece ao definir a IA como engenheira DevOps sênior especialista em Docker, Python, Flask e Kubernetes. Task aparece na solicitação direta de criar um Dockerfile de produção para o Lift com as restrições informadas. Format aparece ao exigir que a resposta contenha somente o Dockerfile em bloco `Dockerfile`.
