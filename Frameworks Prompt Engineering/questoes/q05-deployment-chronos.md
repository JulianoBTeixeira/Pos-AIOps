# Questão 05 - Modernizar deployment legado

## Prompt

Antes, o Chronos roda em produção com este Deployment Kubernetes legado, sem alta disponibilidade, usando `latest`, com secrets hardcoded, sem probes, sem recursos definidos e sem securityContext:

```yaml
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
```

Depois, o manifest precisa representar o padrão moderno de produção da Hill Valley Tech: alta disponibilidade, imagem versionada, secrets fora do manifest, requests e limits, liveness e readiness probes, securityContext não-root e demais boas práticas de produção para Kubernetes.

Faça a ponte entre o estado atual e o desejado gerando uma versão modernizada do Deployment. Use valores realistas quando o enunciado não trouxer números explícitos. Referencie secrets por `secretKeyRef`, não inclua valores sensíveis. Inclua estratégia de rolling update, labels consistentes, automountServiceAccountToken desabilitado e configurações de segurança no pod e no container.

Entregue somente o YAML final em bloco `yaml`.

## Modelo

Claude Sonnet 4. Escolhido por boa capacidade de refatorar manifests mantendo requisitos operacionais e segurança.

## Output

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    component: api
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
      app: chronos-api
      component: api
  template:
    metadata:
      labels:
        app: chronos-api
        component: api
    spec:
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
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 2
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
```

## Justificativa

Before aparece na descrição do Deployment legado e seus problemas. After aparece no estado desejado: HA, imagem versionada, secrets externos, probes, recursos e execução não-root. Bridge aparece no pedido para gerar o YAML modernizado conectando o estado atual ao padrão de produção.
