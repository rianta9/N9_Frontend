# Deployment & Operations Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Platform Engineering Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document defines the **deployment architecture, processes, and operational procedures** for the N9 platform across all environments.

### 2.2 Deployment Principles

| Principle | Description |
|-----------|-------------|
| **Infrastructure as Code** | All infrastructure defined in Terraform/Helm |
| **Immutable Infrastructure** | Replace, don't modify |
| **Blue-Green/Canary** | Zero-downtime deployments |
| **Automated Rollback** | Automatic on health check failure |
| **GitOps** | Git as single source of truth |

---

## 3. Infrastructure Architecture

### 3.1 Cloud Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      N9 CLOUD ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                        ┌──────────────────┐                         │
│                        │   CloudFront     │                         │
│                        │   (CDN)          │                         │
│                        └────────┬─────────┘                         │
│                                 │                                    │
│           ┌─────────────────────┼─────────────────────┐             │
│           │                     │                     │             │
│           ▼                     ▼                     ▼             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   S3 (Static)   │  │   ALB (API)     │  │   ALB (WS)      │     │
│  └─────────────────┘  └────────┬────────┘  └────────┬────────┘     │
│                                │                     │              │
│                                ▼                     ▼              │
│                       ┌────────────────────────────────────┐        │
│                       │         EKS Cluster                │        │
│                       │                                    │        │
│                       │  ┌──────────┐  ┌──────────┐       │        │
│                       │  │ API Pods │  │ WS Pods  │       │        │
│                       │  │ (3-10)   │  │ (2-5)    │       │        │
│                       │  └──────────┘  └──────────┘       │        │
│                       │                                    │        │
│                       │  ┌──────────┐  ┌──────────┐       │        │
│                       │  │ Worker   │  │ Scheduler│       │        │
│                       │  │ Pods     │  │ Pods     │       │        │
│                       │  └──────────┘  └──────────┘       │        │
│                       └────────────────────────────────────┘        │
│                                │                                     │
│           ┌────────────────────┼────────────────────┐               │
│           │                    │                    │               │
│           ▼                    ▼                    ▼               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   RDS Aurora    │  │   ElastiCache   │  │   OpenSearch    │     │
│  │   (PostgreSQL)  │  │   (Redis)       │  │                 │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES NAMESPACE LAYOUT                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Namespace: n9-production                                       │ │
│  │                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────┐   │ │
│  │  │ Deployments                                              │   │ │
│  │  │                                                          │   │ │
│  │  │  n9-api (3-10 replicas, HPA)                            │   │ │
│  │  │  n9-websocket (2-5 replicas, HPA)                       │   │ │
│  │  │  n9-worker (2-5 replicas, HPA)                          │   │ │
│  │  │  n9-scheduler (1 replica, leader election)              │   │ │
│  │  └─────────────────────────────────────────────────────────┘   │ │
│  │                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────┐   │ │
│  │  │ Services                                                 │   │ │
│  │  │                                                          │   │ │
│  │  │  n9-api-svc (ClusterIP) ──▶ Ingress                     │   │ │
│  │  │  n9-websocket-svc (ClusterIP) ──▶ Ingress               │   │ │
│  │  └─────────────────────────────────────────────────────────┘   │ │
│  │                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────┐   │ │
│  │  │ ConfigMaps & Secrets                                     │   │ │
│  │  │                                                          │   │ │
│  │  │  n9-config (app configuration)                          │   │ │
│  │  │  n9-secrets (external secrets operator)                 │   │ │
│  │  └─────────────────────────────────────────────────────────┘   │ │
│  │                                                                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Environment Configuration

### 4.1 Environment Overview

| Environment | Purpose | Infrastructure | Data |
|-------------|---------|----------------|------|
| **Development** | Local dev | Docker Compose | Fixtures |
| **CI** | Automated tests | Ephemeral | Generated |
| **Staging** | Pre-production | EKS (reduced) | Sanitized |
| **Production** | Live service | EKS (full) | Real |
| **DR** | Disaster recovery | EKS (standby) | Replicated |

### 4.2 Environment Specifications

| Aspect | Staging | Production |
|--------|---------|------------|
| **API Instances** | 2 | 3-10 (HPA) |
| **WebSocket Instances** | 1 | 2-5 (HPA) |
| **Worker Instances** | 1 | 2-5 (HPA) |
| **DB Size** | db.r6g.large | db.r6g.2xlarge |
| **Redis Size** | cache.r6g.large | cache.r6g.xlarge |
| **Region** | Single | Multi-AZ |

### 4.3 Configuration Management

```yaml
# Helm values-production.yaml
replicaCount:
  api: 3
  websocket: 2
  worker: 2

resources:
  api:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

env:
  - name: SPRING_PROFILES_ACTIVE
    value: production
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: n9-secrets
        key: database-url
```

---

## 5. Deployment Strategy

### 5.1 Deployment Types

| Strategy | Use Case | Downtime | Rollback Speed |
|----------|----------|----------|----------------|
| **Rolling** | Standard deploys | Zero | Fast |
| **Blue-Green** | Major versions | Zero | Instant |
| **Canary** | High-risk changes | Zero | Instant |
| **Recreate** | Database migrations | Yes | Slow |

### 5.2 Rolling Deployment

```yaml
# Kubernetes Deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: n9-api
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
```

### 5.3 Canary Deployment Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CANARY DEPLOYMENT FLOW                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: Deploy Canary (5% traffic)                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Production v1.0 (95%) ◀────┬────▶ Canary v1.1 (5%)        │    │
│  └─────────────────────────────┼────────────────────────────────┘    │
│                                │                                     │
│  Step 2: Monitor (15 minutes)                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ✓ Error rate < 1%                                          │    │
│  │  ✓ Latency P99 < baseline + 20%                            │    │
│  │  ✓ No critical alerts                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                │                                     │
│  Step 3: Increase traffic (25%)                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Production v1.0 (75%) ◀────┬────▶ Canary v1.1 (25%)       │    │
│  └─────────────────────────────┼────────────────────────────────┘    │
│                                │                                     │
│  Step 4: Full rollout                                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Production v1.1 (100%)                                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. CI/CD Pipeline

### 6.1 Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CI/CD PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  PR Created                                                          │
│      │                                                               │
│      ▼                                                               │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐            │
│  │  Lint   │──▶│  Build  │──▶│  Test   │──▶│  SAST   │            │
│  │         │   │         │   │ (Unit)  │   │  Scan   │            │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘            │
│                                                  │                   │
│  PR Merged to main                               │                   │
│      │◀──────────────────────────────────────────┘                   │
│      │                                                               │
│      ▼                                                               │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐            │
│  │  Build  │──▶│  Image  │──▶│  Push   │──▶│ Deploy  │            │
│  │         │   │  Build  │   │   ECR   │   │ Staging │            │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘            │
│                                                  │                   │
│                                                  ▼                   │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐            │
│  │  E2E    │──▶│ Approve │──▶│ Canary  │──▶│  Full   │            │
│  │  Tests  │   │ (Manual)│   │  Deploy │   │ Rollout │            │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 GitHub Actions Workflow

```yaml
name: Deploy Production

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Build with Maven
        run: ./mvnw clean package -DskipTests
      
      - name: Run tests
        run: ./mvnw verify
      
      - name: Build Docker image
        run: docker build -t n9-api:${{ github.sha }} .
      
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker tag n9-api:${{ github.sha }} $ECR_REGISTRY/n9-api:${{ github.sha }}
          docker push $ECR_REGISTRY/n9-api:${{ github.sha }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging
        run: |
          helm upgrade n9-api ./helm/n9-api \
            --namespace n9-staging \
            --set image.tag=${{ github.sha }} \
            -f ./helm/values-staging.yaml

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Canary Deploy (5%)
        run: |
          helm upgrade n9-api-canary ./helm/n9-api \
            --namespace n9-production \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=1 \
            --set canary.enabled=true \
            --set canary.weight=5
      
      - name: Monitor Canary
        run: ./scripts/monitor-canary.sh --duration 15m
      
      - name: Full Rollout
        run: |
          helm upgrade n9-api ./helm/n9-api \
            --namespace n9-production \
            --set image.tag=${{ github.sha }} \
            -f ./helm/values-production.yaml
```

---

## 7. Container Configuration

### 7.1 Dockerfile

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline
COPY src src
RUN ./mvnw package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Security: Run as non-root user
RUN addgroup -g 1001 n9 && adduser -u 1001 -G n9 -s /bin/sh -D n9
USER n9

COPY --from=build /app/target/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O /dev/null http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 7.2 JVM Configuration

```yaml
# ConfigMap for JVM options
apiVersion: v1
kind: ConfigMap
metadata:
  name: n9-jvm-config
data:
  JAVA_OPTS: |
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis=200
    -XX:+UseStringDeduplication
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/tmp/heapdump.hprof
    -Xms1g
    -Xmx3g
    -XX:MaxMetaspaceSize=256m
    -Djava.security.egd=file:/dev/./urandom
```

---

## 8. Database Operations

### 8.1 Migration Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DATABASE MIGRATION FLOW                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Backward Compatible Changes (Online)                             │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ ADD COLUMN (with default), ADD INDEX CONCURRENTLY       │     │
│     │ → Deploy first, then migrate                             │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. Breaking Changes (Staged)                                        │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ Phase 1: Add new column/table                            │     │
│     │ Phase 2: Deploy code that writes to both                 │     │
│     │ Phase 3: Backfill data                                   │     │
│     │ Phase 4: Deploy code that reads from new                 │     │
│     │ Phase 5: Remove old column/table                         │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. Migrations Run As                                                │
│     - Kubernetes Job (pre-deploy hook)                              │
│     - Flyway/Liquibase with baseline                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 Flyway Configuration

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    out-of-order: false
    table: flyway_schema_history
    clean-disabled: true  # Prevent accidental clean in prod
```

### 8.3 Backup Strategy

| Type | Frequency | Retention | Storage |
|------|-----------|-----------|---------|
| **Automated Snapshot** | Daily | 35 days | RDS |
| **Point-in-Time** | Continuous | 7 days | RDS |
| **Manual Snapshot** | Pre-release | 90 days | RDS + S3 |
| **Logical Backup** | Weekly | 1 year | S3 (encrypted) |

---

## 9. Monitoring & Observability

### 9.1 Monitoring Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Grafana Dashboard                        │    │
│  └──────────────────────────────┬──────────────────────────────┘    │
│                                 │                                    │
│           ┌─────────────────────┼─────────────────────┐             │
│           │                     │                     │             │
│           ▼                     ▼                     ▼             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   Prometheus    │  │      Loki       │  │     Jaeger      │     │
│  │   (Metrics)     │  │     (Logs)      │  │   (Tracing)     │     │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘     │
│           │                    │                     │              │
│           └────────────────────┼─────────────────────┘              │
│                                │                                     │
│                                ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                  Application Pods                            │    │
│  │                                                              │    │
│  │  Metrics (/actuator/prometheus)                             │    │
│  │  Logs (stdout/stderr → Fluentd → Loki)                      │    │
│  │  Traces (OpenTelemetry → Jaeger)                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Key Metrics

| Category | Metric | Alert Threshold |
|----------|--------|-----------------|
| **Availability** | Uptime | < 99.9% |
| **Latency** | P99 response time | > 500ms |
| **Errors** | 5xx error rate | > 1% |
| **Saturation** | CPU usage | > 80% |
| **Saturation** | Memory usage | > 85% |
| **Saturation** | DB connections | > 80% |

### 9.3 Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: n9-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) /
          sum(rate(http_server_requests_seconds_count[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P99 latency is {{ $value }}s"
```

---

## 10. Incident Response

### 10.1 Severity Levels

| Level | Impact | Response Time | Examples |
|-------|--------|---------------|----------|
| **SEV1** | Service down | 15 min | Complete outage |
| **SEV2** | Major degradation | 30 min | Payment failures |
| **SEV3** | Minor degradation | 2 hours | Slow search |
| **SEV4** | Minimal impact | 24 hours | UI glitches |

### 10.2 Incident Response Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INCIDENT RESPONSE FLOW                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Alert Triggered                                                     │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────┐                                                │
│  │ PagerDuty/OpsGenie                                               │
│  │ On-call notified                                                 │
│  └────────┬────────┘                                                │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐     ┌─────────────────┐                        │
│  │ Acknowledge     │────▶│ Create Incident │                        │
│  │ Alert           │     │ Slack Channel   │                        │
│  └────────┬────────┘     └─────────────────┘                        │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │ Triage          │                                                │
│  │ - Assess impact │                                                │
│  │ - Assign severity                                                │
│  └────────┬────────┘                                                │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐     ┌─────────────────┐                        │
│  │ Investigate     │────▶│ Rollback?       │                        │
│  │ & Mitigate      │     │ Feature flag?   │                        │
│  └────────┬────────┘     └─────────────────┘                        │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐     ┌─────────────────┐                        │
│  │ Resolve         │────▶│ Postmortem      │                        │
│  │                 │     │ (within 48h)    │                        │
│  └─────────────────┘     └─────────────────┘                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.3 Runbooks

| Runbook | Trigger | Actions |
|---------|---------|---------|
| `high-error-rate` | 5xx > 1% | Check logs, recent deploys, rollback |
| `database-connection-pool` | Pool exhausted | Scale up, check long queries |
| `memory-pressure` | Memory > 90% | Restart pods, check for leaks |
| `certificate-expiry` | < 14 days | Renew certificates |
| `disk-space-low` | < 20% | Clean logs, expand volume |

---

## 11. Disaster Recovery

### 11.1 DR Strategy

| Component | RPO | RTO | Strategy |
|-----------|-----|-----|----------|
| **Database** | 5 min | 1 hour | Cross-region replica |
| **Cache** | N/A | 15 min | Rebuild from DB |
| **Storage** | 24 hours | 4 hours | Cross-region replication |
| **Application** | N/A | 30 min | Multi-region deployment |

### 11.2 Failover Process

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DR FAILOVER PROCESS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Primary Region (us-east-1)           DR Region (us-west-2)         │
│  ┌─────────────────────┐              ┌─────────────────────┐       │
│  │     Active          │    Sync     │     Standby         │       │
│  │                     │────────────▶│                     │       │
│  │  - EKS Cluster      │              │  - EKS Cluster      │       │
│  │  - RDS Primary      │              │  - RDS Read Replica │       │
│  │  - ElastiCache      │              │  - ElastiCache      │       │
│  │  - S3 Bucket        │              │  - S3 Bucket        │       │
│  └─────────────────────┘              └─────────────────────┘       │
│           │                                    │                     │
│           │  Disaster                          │                     │
│           ▼                                    ▼                     │
│  ┌─────────────────────┐              ┌─────────────────────┐       │
│  │     Down            │              │     Active          │       │
│  └─────────────────────┘              │                     │       │
│                                       │  1. Promote RDS     │       │
│                                       │  2. Update DNS      │       │
│                                       │  3. Scale up EKS    │       │
│                                       └─────────────────────┘       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. Security Operations

### 12.1 Secret Management

| Secret Type | Storage | Rotation |
|-------------|---------|----------|
| Database credentials | AWS Secrets Manager | 90 days |
| API keys | AWS Secrets Manager | 30 days |
| JWT signing keys | AWS Secrets Manager | 180 days |
| TLS certificates | AWS Certificate Manager | Auto |

### 12.2 External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: n9-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: n9-secrets
  data:
    - secretKey: database-url
      remoteRef:
        key: n9/production/database
        property: url
    - secretKey: jwt-secret
      remoteRef:
        key: n9/production/jwt
        property: secret
```

---

## 13. Maintenance Procedures

### 13.1 Maintenance Windows

| Activity | Window | Frequency | Duration |
|----------|--------|-----------|----------|
| Patching | Sunday 02:00-06:00 UTC | Monthly | 2-4 hours |
| DB maintenance | Sunday 04:00-06:00 UTC | Weekly | 30 min |
| Certificate renewal | Any time | As needed | 5 min |
| Major upgrades | Sunday 02:00-08:00 UTC | Quarterly | 4-6 hours |

### 13.2 Pre-Deployment Checklist

- [ ] All tests passing
- [ ] Security scan clean
- [ ] Database migration tested
- [ ] Rollback plan documented
- [ ] Monitoring dashboards ready
- [ ] On-call team notified
- [ ] Feature flags configured
- [ ] Load test passed (if applicable)

---

## 14. References

### 14.1 Related Specifications
- [10_TESTING_STRATEGY.md](10_TESTING_STRATEGY.md)
- [05_SECURITY_AND_COMPLIANCE.md](05_SECURITY_AND_COMPLIANCE.md)
- [07_INTEGRATIONS.md](07_INTEGRATIONS.md)

### 14.2 External References
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Charts](https://helm.sh/docs/)
- [AWS Well-Architected](https://aws.amazon.com/architecture/well-architected/)
