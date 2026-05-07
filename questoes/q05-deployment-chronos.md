# Questão 05 — Modernizar deployment legado

## Prompt

```markdown
# Before
O deployment atual do Chronos API em produção é legado, tem apenas 1 réplica, usa imagem `chronos-api:latest`, expõe segredos diretamente no manifest, não possui resource requests/limits, não possui liveness/readiness probes e não define securityContext.

Manifest atual:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"

# After
Quero uma versão moderna, segura e adequada para produção Kubernetes, contendo:
- alta disponibilidade;
- imagem versionada, sem `latest`;
- segredos referenciados via Kubernetes Secret, sem valores hardcoded;
- resource requests e limits;
- readinessProbe e livenessProbe;
- securityContext não-root no pod e no container;
- rollout seguro;
- labels consistentes;
- boas práticas adicionais de produção quando aplicáveis.

# Bridge
Transforme o manifest legado em uma versão modernizada. Retorne o YAML completo do Deployment e, se necessário, um exemplo separado de Secret apenas com placeholders, sem valores reais. Explique brevemente quais problemas do estado anterior foram corrigidos.
```

## Modelo

**Claude Sonnet 4 — Anthropic.** Escolhido pela boa capacidade de transformar manifests com foco em segurança, legibilidade e práticas de produção.

## Output

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: chronos
    app.kubernetes.io/managed-by: argocd
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: chronos-api
      app.kubernetes.io/component: api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: chronos-api
        app.kubernetes.io/component: api
        app.kubernetes.io/part-of: chronos
    spec:
      serviceAccountName: chronos-api
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: chronos-api:v2.48.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: db-password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: jwt-secret
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 10001
            runAsGroup: 10001
            capabilities:
              drop:
                - ALL
```

### Secret com placeholders

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: chronos-api-secrets
  namespace: production
  labels:
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/component: api
type: Opaque
stringData:
  db-password: "REPLACE_WITH_SECRET_MANAGER_VALUE"
  jwt-secret: "REPLACE_WITH_SECRET_MANAGER_VALUE"
```

### Problemas corrigidos

- A disponibilidade melhora com 3 réplicas e estratégia RollingUpdate.
- A imagem passa a ser versionada, reduzindo risco de deploy não determinístico.
- Segredos deixam de ficar expostos no manifest do Deployment.
- Requests e limits permitem melhor scheduling, autoscaling e controle de uso de recursos.
- Probes permitem separar disponibilidade de inicialização e detectar falhas de runtime.
- SecurityContext reduz privilégios, bloqueia privilege escalation e remove capabilities Linux.

## Justificativa

O **Before** descreve o estado legado e inclui o manifest vulnerável. O **After** define o estado desejado com alta disponibilidade, segurança e boas práticas de produção. O **Bridge** pede explicitamente a transformação do YAML, com Secret separado e explicação das melhorias.
