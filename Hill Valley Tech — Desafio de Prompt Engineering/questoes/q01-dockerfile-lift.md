# Questão 01 — Dockerfile para o Lift

## Prompt

```markdown
# Role
Você é um engenheiro DevOps sênior especializado em Docker, Python, Flask, Gunicorn e boas práticas de containers para Kubernetes em produção.

# Task
Crie um Dockerfile de produção para a aplicação Lift, uma API Python/Flask que escuta na porta 8080. A aplicação possui a seguinte estrutura:

lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py

Conteúdo de requirements.txt:
Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9

Em produção, o serviço deve iniciar com:
gunicorn --bind 0.0.0.0:8080 --workers 4 app:app

Requisitos obrigatórios:
- Usar imagem Python slim adequada para produção.
- Não executar a aplicação como root.
- Instalar dependências de forma reprodutível.
- Expor a porta 8080.
- Definir DATABASE_URL e API_KEY como variáveis esperadas no runtime, sem hardcoding de segredos.
- Usar boas práticas de cache de camadas.
- Evitar copiar arquivos desnecessários como testes para a imagem final, se possível.

# Format
Retorne apenas:
1. Um Dockerfile completo em bloco de código.
2. Um arquivo .dockerignore recomendado em bloco de código.
3. Uma explicação curta, em até 6 bullets, das principais boas práticas aplicadas.
```

## Modelo

**GPT-4o — OpenAI.** Escolhido por ter boa capacidade de geração de artefatos técnicos curtos, com sintaxe correta e explicação objetiva.

## Output

### Dockerfile

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PORT=8080

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY lib/ ./lib/

RUN addgroup --system appgroup \
    && adduser --system --ingroup appgroup appuser \
    && chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

### .dockerignore

```dockerignore
.git
.gitignore
__pycache__/
*.pyc
*.pyo
*.pyd
.pytest_cache/
.coverage
htmlcov/
.env
.venv/
venv/
tests/
Dockerfile
README.md
```

### Boas práticas aplicadas

- Usa `python:3.12-slim`, reduzindo superfície da imagem em relação a imagens completas.
- Instala dependências antes de copiar o código para aproveitar cache de camadas.
- Não inclui segredos no Dockerfile; `DATABASE_URL` e `API_KEY` devem ser injetadas no runtime via Kubernetes Secret/Env.
- Executa o processo como usuário não-root.
- Expõe explicitamente a porta 8080 usada pelo Gunicorn.
- Usa `.dockerignore` para evitar arquivos desnecessários na imagem final.

## Justificativa

O **Role** define o modelo como engenheiro DevOps sênior especializado em Docker, Python, Flask e Kubernetes. A **Task** descreve exatamente a criação do Dockerfile com requisitos técnicos e comando de produção. O **Format** delimita a saída em Dockerfile, `.dockerignore` e bullets explicativos, garantindo previsibilidade.
