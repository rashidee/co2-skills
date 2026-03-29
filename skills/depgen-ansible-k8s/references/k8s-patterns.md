# Kubernetes Manifest Patterns

Complete Kubernetes manifest patterns for deploying containerized applications and their
dependent services.

## Application Manifests

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {namespace}
  labels:
    app.kubernetes.io/part-of: {project_name}
```

### ConfigMap

Generated from all non-sensitive environment variables detected in the application config.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {app_name}-config
  namespace: {namespace}
  labels:
    app.kubernetes.io/name: {app_name}
    app.kubernetes.io/version: "{app_version}"
data:
  # Server
  SERVER_PORT: "{server_port}"
  CORS_ALLOWED_ORIGINS: "https://{app_domain}"

  # Database
  MONGODB_URI: "mongodb://mongo-svc:27017/{db_name}"

  # Authentication
  KEYCLOAK_SERVER_URL: "http://keycloak-svc:8080"
  KEYCLOAK_REALM: "{realm}"
  KEYCLOAK_ISSUER_URI: "http://keycloak-svc:8080/realms/{realm}"
  KEYCLOAK_CLIENT_ID: "{client_id}"

  # Messaging
  RABBITMQ_HOST: "rabbitmq-svc"
  RABBITMQ_PORT: "5672"

  # Mail
  MAIL_HOST: "smtp-svc"
  MAIL_PORT: "587"

  # Logging
  LOG_LEVEL_ROOT: "INFO"
  LOG_LEVEL_APP: "INFO"

  # Stack-specific (examples)
  # Spring Boot:
  JTE_DEV_MODE: "false"
  JTE_PRECOMPILED: "true"
  DEVTOOLS_RESTART: "false"
  DEVTOOLS_LIVERELOAD: "false"
  # Laravel:
  # APP_ENV: "production"
  # APP_DEBUG: "false"
  # Node.js:
  # NODE_ENV: "production"
```

### Secret

Generated from all sensitive environment variables. Values are base64-encoded.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {app_name}-secret
  namespace: {namespace}
  labels:
    app.kubernetes.io/name: {app_name}
type: Opaque
stringData:
  KEYCLOAK_ADMIN_USERNAME: "{admin_username}"
  KEYCLOAK_ADMIN_PASSWORD: "{admin_password}"
  RABBITMQ_USERNAME: "{rmq_username}"
  RABBITMQ_PASSWORD: "{rmq_password}"
  # DB_PASSWORD: "{db_password}"  # If database requires auth
```

**Production note**: For production environments, use external secret management:
- **HashiCorp Vault** with the Vault Secrets Operator
- **AWS Secrets Manager** with External Secrets Operator
- **Azure Key Vault** with Azure Key Vault Provider for Secrets Store CSI Driver

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {app_name}
  namespace: {namespace}
  labels:
    app.kubernetes.io/name: {app_name}
    app.kubernetes.io/version: "{app_version}"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: {app_name}
  template:
    metadata:
      labels:
        app: {app_name}
        app.kubernetes.io/name: {app_name}
        app.kubernetes.io/version: "{app_version}"
    spec:
      serviceAccountName: {app_name}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: {app_name}
          image: {registry}/{image_name}:{app_version}
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: {server_port}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {app_name}-config
            - secretRef:
                name: {app_name}-secret
          resources:
            requests:
              cpu: {cpu_request}
              memory: {mem_request}
            limits:
              cpu: {cpu_limit}
              memory: {mem_limit}
          livenessProbe:
            httpGet:
              path: {health_endpoint}
              port: http
            initialDelaySeconds: {liveness_delay}
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: {health_endpoint}
              port: http
            initialDelaySeconds: {readiness_delay}
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: {health_endpoint}
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 30
      imagePullSecrets:
        - name: registry-credentials
```

**Resource defaults by stack:**

| Stack | CPU Request | CPU Limit | Memory Request | Memory Limit | Liveness Delay | Readiness Delay |
|---|---|---|---|---|---|---|
| Spring Boot | 250m | 1000m | 512Mi | 1024Mi | 60s | 30s |
| Laravel | 100m | 500m | 128Mi | 512Mi | 15s | 10s |
| Node.js | 100m | 500m | 128Mi | 512Mi | 15s | 10s |

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {app_name}-svc
  namespace: {namespace}
  labels:
    app.kubernetes.io/name: {app_name}
spec:
  type: ClusterIP
  selector:
    app: {app_name}
  ports:
    - name: http
      port: {server_port}
      targetPort: http
      protocol: TCP
```

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {app_name}
  namespace: {namespace}
  labels:
    app.kubernetes.io/name: {app_name}
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {app_name}-ingress
  namespace: {namespace}
  labels:
    app.kubernetes.io/name: {app_name}
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  rules:
    - host: {app_domain}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {app_name}-svc
                port:
                  number: {server_port}
  tls:
    - hosts:
        - {app_domain}
      secretName: {app_name}-tls
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {app_name}-hpa
  namespace: {namespace}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {app_name}
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {app_name}-netpol
  namespace: {namespace}
spec:
  podSelector:
    matchLabels:
      app: {app_name}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - port: {server_port}
          protocol: TCP
  egress:
    # Allow DNS
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # Allow database
    - to:
        - podSelector:
            matchLabels:
              app: mongodb
      ports:
        - port: 27017
    # Allow Keycloak
    - to:
        - podSelector:
            matchLabels:
              app: keycloak
      ports:
        - port: 8080
    # Allow RabbitMQ
    - to:
        - podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - port: 5672
    # Allow SMTP
    - to:
        - podSelector:
            matchLabels:
              app: mailcatcher
      ports:
        - port: 1025
```

---

## Dependent Service Manifests

For each dependency, provide both Option A (deploy in K8s) and Option B (external reference).

### MongoDB

#### Option A: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: {namespace}
spec:
  serviceName: mongo-svc
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:{mongodb_version}
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 2Gi
  volumeClaimTemplates:
    - metadata:
        name: mongo-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

#### Option B: External Reference

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-svc
  namespace: {namespace}
spec:
  type: ExternalName
  externalName: {external_mongodb_host}
```

### MySQL

#### Option A: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: {namespace}
spec:
  serviceName: mysql-svc
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:{mysql_version}
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 2Gi
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

### Keycloak

#### Option A: Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: {namespace}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:{keycloak_version}
          args: ["start-dev"]
          ports:
            - containerPort: 8080
          env:
            - name: KEYCLOAK_ADMIN
              value: "admin"
            - name: KEYCLOAK_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-secret
                  key: admin-password
            - name: KC_DB
              value: "mysql"
            - name: KC_DB_URL
              value: "jdbc:mysql://mysql-svc:3306/{keycloak_db}"
            - name: KC_DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: keycloak-secret
                  key: db-username
            - name: KC_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-secret
                  key: db-password
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: keycloak
  ports:
    - port: 8080
      targetPort: 8080
```

### RabbitMQ

#### Option A: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: {namespace}
spec:
  serviceName: rabbitmq-svc
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:{rabbitmq_version}-management-alpine
          ports:
            - containerPort: 5672
              name: amqp
            - containerPort: 15672
              name: management
          env:
            - name: RABBITMQ_DEFAULT_USER
              valueFrom:
                secretKeyRef:
                  name: rabbitmq-secret
                  key: username
            - name: RABBITMQ_DEFAULT_PASS
              valueFrom:
                secretKeyRef:
                  name: rabbitmq-secret
                  key: password
          volumeMounts:
            - name: rabbitmq-data
              mountPath: /var/lib/rabbitmq
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            exec:
              command: ["rabbitmq-diagnostics", "-q", "ping"]
            initialDelaySeconds: 20
            periodSeconds: 10
  volumeClaimTemplates:
    - metadata:
        name: rabbitmq-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: rabbitmq
  ports:
    - name: amqp
      port: 5672
      targetPort: 5672
    - name: management
      port: 15672
      targetPort: 15672
```

### Redis

#### Option A: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: {namespace}
spec:
  serviceName: redis-svc
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:{redis_version}-alpine
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          readinessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 5
            periodSeconds: 10
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
```

### Meilisearch

#### Option A: Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meilisearch
  namespace: {namespace}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: meilisearch
  template:
    metadata:
      labels:
        app: meilisearch
    spec:
      containers:
        - name: meilisearch
          image: getmeili/meilisearch:{meilisearch_version}
          ports:
            - containerPort: 7700
          env:
            - name: MEILI_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: meilisearch-secret
                  key: master-key
          volumeMounts:
            - name: meili-data
              mountPath: /meili_data
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
      volumes:
        - name: meili-data
          persistentVolumeClaim:
            claimName: meilisearch-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: meilisearch-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: meilisearch
  ports:
    - port: 7700
      targetPort: 7700
```

### Mailcatcher (Dev/Staging Only)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mailcatcher
  namespace: {namespace}
  labels:
    app: mailcatcher
    environment: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mailcatcher
  template:
    metadata:
      labels:
        app: mailcatcher
    spec:
      containers:
        - name: mailcatcher
          image: schickling/mailcatcher:latest
          ports:
            - containerPort: 1025
              name: smtp
            - containerPort: 1080
              name: http
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: mailcatcher-svc
  namespace: {namespace}
spec:
  type: ClusterIP
  selector:
    app: mailcatcher
  ports:
    - name: smtp
      port: 1025
      targetPort: 1025
    - name: http
      port: 1080
      targetPort: 1080
```

**Note**: Mailcatcher should ONLY be deployed in dev/staging environments. In production,
use a real SMTP service (e.g., AWS SES, SendGrid, Mailgun) and point `MAIL_HOST`/`MAIL_PORT`
to it via ConfigMap.
