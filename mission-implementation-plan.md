# Mission Implementation Plan: Enterprise-Grade AI Lending Platform
## Granular Engineering Specifications for Production Deployment

---

# TABLE OF CONTENTS

- [Mission 0: Monorepo Structure + Infrastructure-as-Code](#mission-0)
- [Mission 1: Shared Platform Libraries + Cross-Cutting Services](#mission-1)
- [Mission 2: Data Layer Services (S1-S5)](#mission-2)
- [Mission 3: Decisioning Engine (S6-S8)](#mission-3)
- [Mission 4: Distribution Layer (S9-S12)](#mission-4)
- [Mission 5: Agent Intelligence Layer (S13-S16)](#mission-5)
- [Mission 6: Context Graph + Monitoring (S17-S18)](#mission-6)
- [Mission 7: Lender Dashboard + Copilot](#mission-7)
- [Mission 8: Agent Lifecycle Pipeline (SRAL)](#mission-8)
- [Mission 9: Integration Testing + Load Testing + Security](#mission-9)
- [Mission 10: Production Deployment + First Lender Onboarding](#mission-10)

---

# MISSION 0: MONOREPO STRUCTURE + INFRASTRUCTURE-AS-CODE

**Objective:** Establish the unified monorepo, CI/CD, and all infrastructure components so every subsequent mission deploys into a known-good environment.

**Dependencies:** None (this is the foundation).

**Duration:** Weeks 1-4

## 0.1 Monorepo Directory Layout

Base from `voiceai-agent-services-platform` (14-service architecture), extended for lending domain:

```
lending-platform/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                          # lint + test + security scan on every PR
│   │   ├── cd-staging.yml                  # auto-deploy to staging on merge to main
│   │   ├── cd-production.yml               # manual approval gate -> deploy to prod
│   │   └── model-deploy.yml                # MLflow model promotion pipeline
│   └── CODEOWNERS
│
├── infrastructure/
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── eks/                        # EKS cluster (3-node HA control plane)
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   ├── node_groups.tf          # CPU pool + GPU pool (g5.xlarge for inference)
│   │   │   │   └── iam.tf
│   │   │   ├── rds/                        # PostgreSQL 16 with RLS
│   │   │   │   ├── main.tf                 # Multi-AZ, encrypted storage, parameter groups
│   │   │   │   └── security_groups.tf
│   │   │   ├── elasticache/                # Redis 7 Cluster (6-node)
│   │   │   ├── msk/                        # Kafka (MSK 3-broker, schema registry)
│   │   │   ├── neo4j/                      # Neo4j on EC2 (causal cluster, 3 core)
│   │   │   ├── temporal/                   # Temporal server on EKS
│   │   │   ├── s3/                         # Buckets: models, voice-recordings, documents, backups
│   │   │   ├── vault/                      # HashiCorp Vault on EKS
│   │   │   └── networking/                 # VPC, subnets, NAT, security groups, VPN
│   │   ├── environments/
│   │   │   ├── staging/
│   │   │   │   ├── main.tf
│   │   │   │   ├── terraform.tfvars
│   │   │   │   └── backend.tf
│   │   │   └── production/
│   │   │       ├── main.tf
│   │   │       ├── terraform.tfvars
│   │   │       └── backend.tf
│   │   └── global/
│   │       ├── ecr.tf                      # Container registry
│   │       ├── route53.tf                  # DNS
│   │       └── acm.tf                      # TLS certificates
│   │
│   ├── helm/
│   │   ├── platform-base/                  # Shared Helm chart for all Python services
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml                 # Default values
│   │   │   ├── values-staging.yaml
│   │   │   ├── values-production.yaml
│   │   │   └── templates/
│   │   │       ├── deployment.yaml
│   │   │       ├── service.yaml
│   │   │       ├── hpa.yaml                # Horizontal Pod Autoscaler
│   │   │       ├── pdb.yaml                # Pod Disruption Budget
│   │   │       ├── configmap.yaml
│   │   │       ├── secret.yaml             # Vault-injected
│   │   │       ├── service-monitor.yaml    # Prometheus scrape config
│   │   │       └── network-policy.yaml     # Service mesh policies
│   │   │
│   │   ├── observability/
│   │   │   ├── prometheus/
│   │   │   ├── grafana/
│   │   │   │   └── dashboards/
│   │   │   │       ├── platform-overview.json
│   │   │   │       ├── loan-lifecycle.json
│   │   │   │       ├── agent-performance.json
│   │   │   │       ├── data-quality.json
│   │   │   │       └── per-lender.json
│   │   │   ├── loki/
│   │   │   └── jaeger/
│   │   │
│   │   └── istio/                          # Service mesh
│   │       ├── gateway.yaml
│   │       ├── virtual-services/
│   │       └── destination-rules/
│   │
│   └── docker/
│       ├── Dockerfile.python               # Base image for all Python services
│       ├── Dockerfile.node                  # Base image for dashboard
│       └── docker-compose.dev.yml          # Local development (all services)
│
├── packages/                               # Shared libraries (installed as local packages)
│   ├── platform-common/                    # Auth, logging, config, error handling
│   │   ├── pyproject.toml
│   │   └── src/platform_common/
│   │       ├── __init__.py
│   │       ├── auth/
│   │       │   ├── jwt.py                  # JWT validation + generation
│   │       │   ├── rbac.py                 # Role-based access control
│   │       │   ├── api_key.py              # Partner API key validation
│   │       │   └── middleware.py           # FastAPI auth middleware
│   │       ├── config/
│   │       │   ├── loader.py               # YAML config loader with env override
│   │       │   ├── vault.py                # HashiCorp Vault client
│   │       │   └── segment.py              # Segment config registry (38 segments)
│   │       ├── db/
│   │       │   ├── postgres.py             # SQLAlchemy async engine + session factory
│   │       │   ├── redis.py                # Redis async client + caching decorators
│   │       │   ├── models.py               # Base model with lender_id, created_at, audit
│   │       │   └── rls.py                  # Row-level security enforcement middleware
│   │       ├── events/
│   │       │   ├── kafka.py                # Kafka producer + consumer with Avro
│   │       │   ├── schemas.py              # Event schema definitions (Avro)
│   │       │   └── handlers.py             # Base event handler with error handling
│   │       ├── logging/
│   │       │   ├── structured.py           # JSON structured logging
│   │       │   ├── tracing.py              # OpenTelemetry trace context propagation
│   │       │   └── pii_filter.py           # PII redaction in logs (Aadhaar, PAN)
│   │       ├── errors/
│   │       │   ├── exceptions.py           # Platform exception hierarchy
│   │       │   └── handlers.py             # FastAPI exception handlers
│   │       ├── crypto/
│   │       │   ├── pii.py                  # AES-256-GCM encrypt/decrypt for PII fields
│   │       │   ├── hashing.py              # Aadhaar/PAN hashing for dedup
│   │       │   └── consent.py              # DPDP consent token management
│   │       ├── audit/
│   │       │   ├── trail.py                # Immutable audit log writer
│   │       │   └── decorators.py           # @audited decorator for service methods
│   │       ├── health/
│   │       │   ├── checks.py               # Liveness + readiness probes
│   │       │   └── metrics.py              # Prometheus metrics helpers
│   │       └── testing/
│   │           ├── fixtures.py             # Shared test fixtures (db, redis, kafka)
│   │           ├── factories.py            # Test data factories (borrower, loan, lender)
│   │           └── assertions.py           # Custom assertions for lending domain
│   │
│   ├── platform-mcp/                       # MCP server/client library
│   │   ├── pyproject.toml
│   │   └── src/platform_mcp/
│   │       ├── server.py                   # MCP server base (JSON-RPC 2.0)
│   │       ├── client.py                   # MCP client for inter-service calls
│   │       ├── tools.py                    # Tool registration decorators
│   │       ├── permissions.py              # Per-tool RBAC
│   │       └── transport.py                # HTTP/SSE + stdio transport
│   │
│   └── platform-temporal/                  # Temporal workflow utilities
│       ├── pyproject.toml
│       └── src/platform_temporal/
│           ├── client.py                   # Temporal client wrapper
│           ├── workflows.py                # Base workflow patterns
│           ├── activities.py               # Activity decorators with retry
│           └── signals.py                  # Signal/query definitions
│
├── services/                               # 18 core microservices
│   ├── uli-orchestrator/                   # S1
│   ├── data-quality-engine/                # S2
│   ├── bankconnect-svc/                    # S3
│   ├── deviceconnect-svc/                  # S4
│   ├── identity-svc/                       # S5
│   ├── sentinel-core/                      # S6
│   ├── product-engine/                     # S7
│   ├── fraud-engine/                       # S8
│   ├── prism-matcher/                      # S9
│   ├── los-engine/                         # S10
│   ├── journey-studio/                     # S11
│   ├── notification-svc/                   # S12
│   ├── agent-runtime/                      # S13
│   ├── voice-pipeline/                     # S14
│   ├── constellation/                      # S15
│   ├── knowledge-svc/                      # S16
│   ├── context-graph/                      # S17
│   └── monitoring-agent/                   # S18
│
├── cross-cutting/                          # 7 cross-cutting services
│   ├── api-gateway/                        # Kong config or custom gateway
│   ├── auth-svc/
│   ├── audit-trail/
│   ├── compliance-reporter/
│   ├── ml-platform/
│   ├── observability/
│   └── mcp-server/                         # Unified MCP endpoint (49+ tools)
│
├── dashboard/                              # Lender dashboard (React + TypeScript)
│   ├── package.json
│   ├── src/
│   │   ├── pages/
│   │   │   ├── portfolio/
│   │   │   ├── applications/
│   │   │   ├── policy-builder/
│   │   │   ├── shadow-test/
│   │   │   ├── copilot/
│   │   │   ├── compliance/
│   │   │   ├── monitoring/
│   │   │   └── settings/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── store/
│   └── tsconfig.json
│
├── sdk/                                    # External SDKs
│   ├── python/                             # pip-installable platform SDK
│   ├── typescript/                         # npm-installable for partners
│   └── android/                            # DeviceConnect mobile SDK
│
├── configs/
│   ├── segments/                           # 38 borrower segment YAML configs
│   │   ├── dairy_farmer.yaml
│   │   ├── street_vendor.yaml
│   │   ├── gig_worker_delivery.yaml
│   │   ├── gig_worker_ridehail.yaml
│   │   ├── asha_worker.yaml
│   │   ├── kirana_ondc.yaml
│   │   ├── shg_member.yaml
│   │   ├── fpo_member.yaml
│   │   ├── salaried_ntc.yaml
│   │   ├── crop_farmer_rice.yaml
│   │   ├── crop_farmer_cotton.yaml
│   │   ├── crop_farmer_sugarcane.yaml
│   │   ├── fisherman.yaml
│   │   ├── ev_driver.yaml
│   │   ├── gold_loan.yaml
│   │   ├── warehouse_receipt.yaml
│   │   └── ... (38 total)
│   ├── policies/                           # Sentinel policy templates
│   │   ├── dairy_animal_purchase_v1.yaml
│   │   ├── vendor_working_capital_v1.yaml
│   │   └── gig_emergency_v1.yaml
│   ├── journeys/                           # Journey Studio flow definitions
│   │   ├── dairy_voice_gujarati.json
│   │   ├── vendor_whatsapp_hindi.json
│   │   └── gig_web_english.json
│   ├── lenders/                            # Lender Prism profiles
│   │   └── example_sfb.yaml
│   └── compliance/
│       ├── rbi_digital_lending.yaml
│       ├── dpdp_act.yaml
│       ├── psl_classification.yaml
│       └── rbi_cla_2026.yaml
│
├── migrations/
│   └── alembic/
│       ├── alembic.ini
│       ├── env.py
│       └── versions/
│           ├── 001_foundation_tables.py
│           ├── 002_data_layer_tables.py
│           ├── 003_decisioning_tables.py
│           ├── 004_distribution_tables.py
│           ├── 005_agent_tables.py
│           └── 006_monitoring_tables.py
│
├── tests/
│   ├── unit/                               # Per-service unit tests
│   ├── integration/                        # Cross-service integration tests
│   ├── e2e/                                # Full loan lifecycle end-to-end
│   ├── load/                               # Locust load test scenarios
│   ├── security/                           # OWASP + PII leak + RBAC tests
│   └── fixtures/                           # Shared test data
│       ├── borrower_profiles/
│       ├── bank_statements/                # Anonymized real statements
│       ├── segment_test_data/
│       └── mock_uli_responses/
│
├── docs/
│   ├── adr/                                # Architecture Decision Records
│   ├── runbooks/
│   └── api/                                # Auto-generated OpenAPI specs
│
├── pyproject.toml                          # Root project config (monorepo tooling)
├── Makefile                                # Build/test/deploy commands
└── .pre-commit-config.yaml                 # Pre-commit hooks (ruff, mypy, security)
```

## 0.2 Standard Service Template

Every service in `services/` follows this internal structure:

```
services/<service-name>/
├── pyproject.toml                          # Service dependencies
├── Dockerfile                              # Extends infrastructure/docker/Dockerfile.python
├── helm/
│   └── values.yaml                         # Service-specific Helm overrides
├── src/<service_name>/
│   ├── __init__.py
│   ├── main.py                             # FastAPI app factory
│   ├── config.py                           # Pydantic Settings (env + Vault)
│   ├── api/
│   │   ├── __init__.py
│   │   ├── router.py                       # FastAPI router
│   │   ├── schemas.py                      # Pydantic request/response models
│   │   └── dependencies.py                 # FastAPI Depends (auth, db session, etc.)
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── models.py                       # SQLAlchemy ORM models
│   │   ├── entities.py                     # Domain entities (pure Python)
│   │   ├── services.py                     # Business logic (no framework deps)
│   │   ├── repository.py                   # Data access layer
│   │   └── events.py                       # Kafka event definitions
│   ├── integrations/
│   │   ├── __init__.py                     # External API clients
│   │   └── ...
│   └── mcp/
│       ├── __init__.py
│       └── tools.py                        # MCP tool implementations for this service
├── tests/
│   ├── conftest.py
│   ├── test_api.py
│   ├── test_domain.py
│   └── test_integrations.py
└── alembic/                                # Service-specific migrations (if needed)
```

## 0.3 CI/CD Pipeline Specification

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff mypy
      - run: ruff check .
      - run: mypy services/ packages/ --strict

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: test }
        options: >-
          --health-cmd pg_isready --health-interval 10s
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping" --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e "packages/platform-common[test]"
      - run: pytest tests/unit/ -v --cov=services --cov-report=xml
      - run: pytest tests/integration/ -v --timeout=120

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      - run: pip install bandit safety
      - run: bandit -r services/ -ll
      - run: safety check

  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [uli-orchestrator, data-quality-engine, bankconnect-svc, ...]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: services/${{ matrix.service }}/Dockerfile
          push: true
          tags: ${{ env.ECR }}/${{ matrix.service }}:${{ github.sha }}
```

## 0.4 Database Schema -- Foundation Tables (Migration 001)

```sql
-- migrations/alembic/versions/001_foundation_tables.py

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- LENDER MANAGEMENT
-- ============================================================
CREATE TABLE lenders (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL CHECK (type IN ('PSB', 'SFB', 'NBFC', 'MFI', 'LSP')),
    status          VARCHAR(20) NOT NULL DEFAULT 'ONBOARDING'
                    CHECK (status IN ('ONBOARDING', 'SHADOW', 'PILOT', 'ACTIVE', 'SUSPENDED')),
    config_json     JSONB NOT NULL DEFAULT '{}',       -- Prism profile, PSL targets, risk appetite
    api_key_hash    VARCHAR(255),                       -- bcrypt hash of API key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ROW-LEVEL SECURITY
-- ============================================================
-- Every data table includes lender_id and enforces isolation

ALTER TABLE lenders ENABLE ROW LEVEL SECURITY;
CREATE POLICY lender_isolation ON lenders
    USING (id = current_setting('app.current_lender_id')::UUID);

-- ============================================================
-- RBAC
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id       UUID NOT NULL REFERENCES lenders(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL
                    CHECK (role IN ('CRO', 'RISK_ANALYST', 'UNDERWRITER', 'OPS', 'COMPLIANCE', 'ADMIN')),
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_lender ON users(lender_id);

-- ============================================================
-- AUDIT LOG (immutable, append-only, partitioned by month)
-- ============================================================
CREATE TABLE audit_log (
    id              BIGSERIAL,
    lender_id       UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,               -- 'LOAN', 'POLICY', 'AGENT', 'USER'
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,               -- 'CREATED', 'STATE_CHANGE', 'DECISION', etc.
    actor_type      VARCHAR(20) NOT NULL,               -- 'USER', 'AGENT', 'SYSTEM'
    actor_id        VARCHAR(255) NOT NULL,
    payload_hash    VARCHAR(64) NOT NULL,               -- SHA-256 of payload for tamper detection
    payload         JSONB NOT NULL,
    prev_hash       VARCHAR(64),                        -- Chain hash for tamper evidence
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions (automated via pg_partman or cron)
CREATE TABLE audit_log_2026_04 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_lender_ts ON audit_log(lender_id, timestamp);

-- ============================================================
-- CONSENT LEDGER (DPDP Act compliance)
-- ============================================================
CREATE TABLE consent_ledger (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id       UUID NOT NULL REFERENCES lenders(id),
    borrower_id     UUID NOT NULL,
    data_element    VARCHAR(100) NOT NULL,              -- 'AADHAAR', 'PAN', 'BANK_STATEMENT', etc.
    purpose         VARCHAR(100) NOT NULL,              -- 'KYC', 'CREDIT_SCORING', 'MONITORING'
    consent_token   VARCHAR(255) NOT NULL,              -- Opaque token from consent flow
    granted_at      TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'EXPIRED', 'REVOKED'))
);
CREATE INDEX idx_consent_borrower ON consent_ledger(borrower_id, status);
```

## 0.5 Kafka Topic Registry

```yaml
# infrastructure/kafka/topics.yaml
topics:
  # Data Layer
  - name: uli.fetch.requested
    partitions: 12
    replication: 3
    retention_ms: 604800000     # 7 days
    schema: ULIFetchRequest.avsc

  - name: uli.fetch.completed
    partitions: 12
    replication: 3
    schema: ULIFetchResponse.avsc

  - name: dq.enriched
    partitions: 12
    replication: 3
    schema: EnrichedProfile.avsc

  - name: bc.analyzed
    partitions: 6
    replication: 3
    schema: BankConnectResult.avsc

  - name: dc.scored
    partitions: 6
    replication: 3
    schema: DeviceConnectScore.avsc

  - name: id.verified
    partitions: 6
    replication: 3
    schema: IdentityResult.avsc

  # Decisioning
  - name: sentinel.decision.requested
    partitions: 12
    replication: 3
    schema: DecisionRequest.avsc

  - name: sentinel.decision.completed
    partitions: 12
    replication: 3
    schema: DecisionResult.avsc

  - name: fraud.alert
    partitions: 6
    replication: 3
    schema: FraudAlert.avsc

  - name: product.designed
    partitions: 6
    replication: 3
    schema: ProductDesign.avsc

  # Distribution
  - name: prism.matched
    partitions: 6
    replication: 3
    schema: LenderMatch.avsc

  - name: los.event.created
    partitions: 12
    replication: 3
    schema: LOSEvent.avsc

  - name: los.event.disbursed
    partitions: 6
    replication: 3
    schema: LOSEvent.avsc

  # Agent
  - name: agent.interaction
    partitions: 12
    replication: 3
    schema: AgentInteraction.avsc

  - name: voice.stream
    partitions: 24            # High throughput for voice
    replication: 3
    schema: VoiceChunk.avsc

  # Monitoring
  - name: monitor.trigger.detected
    partitions: 6
    replication: 3
    schema: TriggerEvent.avsc

  - name: monitor.trigger.intervened
    partitions: 6
    replication: 3
    schema: InterventionEvent.avsc

  # Cross-cutting
  - name: graph.ingest
    partitions: 12
    replication: 3
    schema: GraphEvent.avsc

  - name: audit.event
    partitions: 12
    replication: 3
    retention_ms: 2592000000  # 30 days (also persisted to audit_log table)
    schema: AuditEvent.avsc

  - name: rl.signal
    partitions: 6
    replication: 3
    schema: RLSignal.avsc

  - name: notify.dispatch
    partitions: 6
    replication: 3
    schema: NotificationRequest.avsc
```

## 0.6 Acceptance Criteria (Gate 0)

```
[ ] Monorepo created with all directories, CI/CD green on empty services
[ ] Terraform apply succeeds for staging environment
[ ] PostgreSQL: RLS verified (cross-lender query returns empty)
[ ] Redis: cluster healthy, TLS enforced
[ ] Kafka: all topics created, produce/consume verified
[ ] Neo4j: causal cluster up, governance graph schema loaded
[ ] Temporal: test workflow completes end-to-end
[ ] Vault: secrets injected into test pod
[ ] Observability: Grafana shows all infra dashboards
[ ] CI pipeline: PR triggers lint + test + security + build
[ ] Docker images build and push to ECR
[ ] Helm deploy to staging succeeds for platform-base chart
[ ] Pre-commit hooks: ruff + mypy + bandit configured
```

---

# MISSION 1: SHARED PLATFORM LIBRARIES + CROSS-CUTTING SERVICES

**Objective:** Build the shared libraries that every service depends on, plus the cross-cutting services (auth, audit, gateway).

**Dependencies:** Mission 0

**Duration:** Weeks 2-4 (overlaps with Mission 0 infrastructure)

## 1.1 `platform-common` Library -- Key Implementation Details

### Auth Middleware

```python
# packages/platform-common/src/platform_common/auth/middleware.py

from fastapi import Request, HTTPException
from fastapi.security import HTTPBearer
import jwt

class LenderAuthMiddleware:
    """Validates JWT, extracts lender_id, sets PostgreSQL session variable for RLS."""

    async def __call__(self, request: Request):
        token = await HTTPBearer()(request)
        payload = jwt.decode(token.credentials, settings.jwt_secret, algorithms=["HS256"])

        request.state.lender_id = payload["lender_id"]
        request.state.user_id = payload["sub"]
        request.state.role = payload["role"]

        # SET PostgreSQL session variable for row-level security
        async with db_session() as session:
            await session.execute(
                text(f"SET app.current_lender_id = '{payload['lender_id']}'")
            )

        return payload
```

### PII Encryption

```python
# packages/platform-common/src/platform_common/crypto/pii.py

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

class PIIEncryptor:
    """AES-256-GCM encryption for PII fields. Key from Vault."""

    def __init__(self, key: bytes):
        assert len(key) == 32  # 256 bits
        self._aesgcm = AESGCM(key)

    def encrypt(self, plaintext: str, associated_data: bytes = b"") -> str:
        nonce = os.urandom(12)
        ct = self._aesgcm.encrypt(nonce, plaintext.encode(), associated_data)
        return (nonce + ct).hex()

    def decrypt(self, ciphertext_hex: str, associated_data: bytes = b"") -> str:
        data = bytes.fromhex(ciphertext_hex)
        nonce, ct = data[:12], data[12:]
        return self._aesgcm.decrypt(nonce, ct, associated_data).decode()


# SQLAlchemy type decorator for transparent PII encryption
class EncryptedString(TypeDecorator):
    impl = String
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            return pii_encryptor.encrypt(value)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            return pii_encryptor.decrypt(value)
        return value
```

### Structured Logging with PII Redaction

```python
# packages/platform-common/src/platform_common/logging/structured.py

import structlog
import re

PII_PATTERNS = {
    "aadhaar": re.compile(r"\b\d{4}\s?\d{4}\s?\d{4}\b"),
    "pan": re.compile(r"\b[A-Z]{5}\d{4}[A-Z]\b"),
    "phone": re.compile(r"\b[6-9]\d{9}\b"),
    "email": re.compile(r"\b[\w.-]+@[\w.-]+\.\w+\b"),
}

def redact_pii(_, __, event_dict):
    """Processor that redacts PII from all log fields."""
    for key, value in event_dict.items():
        if isinstance(value, str):
            for pii_type, pattern in PII_PATTERNS.items():
                value = pattern.sub(f"[REDACTED_{pii_type.upper()}]", value)
            event_dict[key] = value
    return event_dict

def configure_logging(service_name: str):
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            redact_pii,
            structlog.processors.JSONRenderer(),
        ],
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
    )
```

### Audit Trail Decorator

```python
# packages/platform-common/src/platform_common/audit/decorators.py

import functools
import hashlib
import json
from platform_common.events.kafka import kafka_producer

def audited(entity_type: str, action: str):
    """Decorator that emits audit event for every service method call."""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            result = await func(*args, **kwargs)

            payload = {
                "args": str(args[1:]),  # skip self
                "result_type": type(result).__name__,
            }
            payload_json = json.dumps(payload, sort_keys=True)

            await kafka_producer.send("audit.event", {
                "lender_id": str(get_current_lender_id()),
                "entity_type": entity_type,
                "entity_id": str(getattr(result, "id", "unknown")),
                "action": action,
                "actor_type": get_current_actor_type(),
                "actor_id": str(get_current_actor_id()),
                "payload_hash": hashlib.sha256(payload_json.encode()).hexdigest(),
                "payload": payload,
            })
            return result
        return wrapper
    return decorator
```

## 1.2 API Gateway Configuration

```yaml
# cross-cutting/api-gateway/kong.yaml
services:
  # Data Layer
  - name: uli-orchestrator
    url: http://uli-orchestrator.platform.svc:8000
    routes:
      - paths: ["/v1/data/fetch"]
        methods: ["POST"]
    plugins:
      - name: jwt
      - name: rate-limiting
        config: { minute: 100, policy: local }
      - name: request-transformer
        config:
          add:
            headers: ["X-Request-ID:$(uuid)"]

  # Decisioning
  - name: sentinel-core
    url: http://sentinel-core.platform.svc:8000
    routes:
      - paths: ["/v1/decision/*"]
        methods: ["POST", "GET"]
    plugins:
      - name: jwt
      - name: rate-limiting
        config: { minute: 500, policy: local }

  # MCP
  - name: mcp-server
    url: http://mcp-server.platform.svc:8443
    routes:
      - paths: ["/mcp"]
        methods: ["POST"]
    plugins:
      - name: jwt
      - name: rate-limiting
        config: { minute: 1000, policy: local }

  # Dashboard API
  - name: dashboard-api
    url: http://dashboard-api.platform.svc:8000
    routes:
      - paths: ["/api/v1/*"]
    plugins:
      - name: jwt
      - name: cors
        config:
          origins: ["https://dashboard.lending-platform.ai"]
```

## 1.3 Acceptance Criteria

```
[ ] platform-common installed as editable dependency in all services
[ ] Auth middleware: JWT validation + RLS session variable setting verified
[ ] PII encryption: encrypt -> persist -> retrieve -> decrypt roundtrip
[ ] Structured logging: JSON output, PII redacted, trace ID propagated
[ ] Audit decorator: method call -> Kafka audit.event -> audit_log table
[ ] API Gateway: routes all service endpoints, JWT enforced, rate-limited
[ ] Auth service: login -> JWT issued, role verified, refresh token works
[ ] RBAC: CRO can promote policy, ANALYST cannot (verified end-to-end)
[ ] Health checks: /healthz and /readyz on every service return 200
[ ] Prometheus metrics: request_count, latency_seconds exposed per service
```

---

# MISSION 2: DATA LAYER SERVICES (S1-S5)

**Objective:** Build all 5 data services that fetch, parse, validate, and enrich borrower data from DPI/ULI sources.

**Dependencies:** Mission 0, Mission 1

**Duration:** Weeks 5-12

## 2.1 Database Schema -- Data Layer (Migration 002)

```sql
-- migrations/alembic/versions/002_data_layer_tables.py

-- ============================================================
-- BORROWER PROFILES (PII encrypted)
-- ============================================================
CREATE TABLE borrower_profiles (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    aadhaar_hash        VARCHAR(64) NOT NULL,           -- SHA-256 for dedup (never store raw)
    aadhaar_encrypted   TEXT,                            -- AES-256-GCM encrypted
    pan_encrypted       TEXT,
    name_encrypted      TEXT,
    dob_encrypted       TEXT,
    phone_encrypted     TEXT,
    address_encrypted   TEXT,
    pincode             VARCHAR(6),                      -- Not PII, needed for geo queries
    segment             VARCHAR(50) NOT NULL,
    data_quality_score  DECIMAL(3,2),                   -- 0.00 - 1.00
    profile_json        JSONB,                           -- Enriched profile (non-PII features)
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_borrower_aadhaar_lender
    ON borrower_profiles(aadhaar_hash, lender_id);
CREATE INDEX idx_borrower_segment ON borrower_profiles(lender_id, segment);
ALTER TABLE borrower_profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY borrower_rls ON borrower_profiles
    USING (lender_id = current_setting('app.current_lender_id')::UUID);

-- ============================================================
-- DATA FETCH REQUESTS (ULI orchestrator tracking)
-- ============================================================
CREATE TABLE data_fetch_requests (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    borrower_id         UUID NOT NULL REFERENCES borrower_profiles(id),
    segment             VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                        CHECK (status IN ('PENDING', 'IN_PROGRESS', 'COMPLETE', 'PARTIAL', 'FAILED')),
    sources_requested   JSONB NOT NULL,                 -- [{source, required, timeout_ms}]
    sources_completed   JSONB DEFAULT '[]',             -- [{source, status, latency_ms, data_ref}]
    confidence_score    DECIMAL(3,2),
    data_quality_flags  TEXT[],
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fetch_borrower ON data_fetch_requests(borrower_id);

-- ============================================================
-- BANK STATEMENT ANALYSIS
-- ============================================================
CREATE TABLE bank_statements (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    borrower_id         UUID NOT NULL REFERENCES borrower_profiles(id),
    bank_name           VARCHAR(100),
    account_type        VARCHAR(20),                    -- SAVINGS, CURRENT
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    statement_s3_key    VARCHAR(500),                   -- Encrypted PDF in S3
    fbcs_score          INTEGER,                        -- 0-900
    monthly_income      JSONB,                          -- [{month, amount, source_breakdown}]
    income_stability    DECIMAL(3,2),                   -- stddev/mean
    fraud_flags         TEXT[],                         -- CIRCULAR_TRANSFER, ROUND_TRIP, etc.
    cluster_analysis    JSONB,                          -- Cluster intelligence results
    transaction_summary JSONB,                          -- Category totals, counts
    parsed_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bankstmt_borrower ON bank_statements(borrower_id);

-- ============================================================
-- DEVICE BEHAVIORAL DATA
-- ============================================================
CREATE TABLE device_scores (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    borrower_id         UUID NOT NULL REFERENCES borrower_profiles(id),
    device_id_hash      VARCHAR(64),                    -- Hashed device identifier
    fis_score           INTEGER,                        -- 0-1000
    feature_vector      JSONB,                          -- Extracted features (not raw data)
    fraud_flags         TEXT[],
    consent_token       VARCHAR(255) NOT NULL,
    raw_data_deleted_at TIMESTAMPTZ,                    -- Must be set within retention window
    scored_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- IDENTITY VERIFICATION RESULTS
-- ============================================================
CREATE TABLE identity_verifications (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    borrower_id         UUID NOT NULL REFERENCES borrower_profiles(id),
    verification_type   VARCHAR(30) NOT NULL
                        CHECK (verification_type IN (
                            'AADHAAR_OTP', 'AADHAAR_BIOMETRIC', 'PAN',
                            'DIGILOCKER', 'AA_CONSENT', 'VIDEO_KYC'
                        )),
    status              VARCHAR(20) NOT NULL
                        CHECK (status IN ('PENDING', 'SUCCESS', 'FAILED', 'EXPIRED')),
    result_json         JSONB,                          -- Verification result (PII encrypted within)
    consent_id          UUID REFERENCES consent_ledger(id),
    verified_at         TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_identity_borrower ON identity_verifications(borrower_id);
```

## 2.2 S1: `uli-orchestrator` -- Implementation Detail

### API Contract

```python
# services/uli-orchestrator/src/uli_orchestrator/api/schemas.py

class DataFetchRequest(BaseModel):
    borrower_id: UUID
    aadhaar_token: str                      # Encrypted consent token
    aa_consent_handle: Optional[str]
    segment_hint: str                       # DAIRY_FARMER, GIG_WORKER, etc.
    lender_id: UUID
    request_id: UUID                        # Idempotency key

class SourceResult(BaseModel):
    source: str                             # CREDIT_BUREAU_CIBIL, DAIRY_COOPERATIVE, etc.
    status: Literal["SUCCESS", "TIMEOUT", "FAILED", "SKIPPED"]
    latency_ms: int
    data_ref: Optional[str]                 # Reference to fetch result in DB

class DataFetchResponse(BaseModel):
    request_id: UUID
    status: Literal["COMPLETE", "PARTIAL", "FAILED"]
    sources_fetched: list[SourceResult]
    confidence_score: float                 # 0.0 - 1.0
    data_quality_flags: list[str]
    enriched_profile: dict                  # Output of data-quality-engine
```

### Parallel Fetch Engine

```python
# services/uli-orchestrator/src/uli_orchestrator/domain/services.py

import asyncio
from circuitbreaker import circuit

class ULIFetchService:
    """Parallel ULI data fetcher with circuit breakers and graceful degradation."""

    def __init__(self, segment_registry, source_clients, kafka_producer):
        self.segment_registry = segment_registry
        self.source_clients = source_clients
        self.kafka = kafka_producer

    @audited(entity_type="DATA_FETCH", action="INITIATED")
    async def fetch_borrower_data(self, request: DataFetchRequest) -> DataFetchResponse:
        # 1. Load segment config to determine which sources to fetch
        segment_config = self.segment_registry.get(request.segment_hint)
        required_sources = segment_config.required_sources
        optional_sources = segment_config.optional_sources

        # 2. Validate consent before any fetch
        await self._validate_consent(request.aadhaar_token, request.borrower_id)

        # 3. Create async tasks for all sources (parallel execution)
        tasks = {}
        for source in required_sources + optional_sources:
            client = self.source_clients[source.name]
            timeout = source.timeout_ms / 1000.0
            tasks[source.name] = asyncio.create_task(
                self._fetch_with_circuit_breaker(
                    client, request.borrower_id, request.aadhaar_token, timeout
                )
            )

        # 4. Gather results (don't fail on individual source failures)
        results = {}
        for source_name, task in tasks.items():
            try:
                results[source_name] = await asyncio.wait_for(
                    task, timeout=segment_config.max_total_timeout_s
                )
            except (asyncio.TimeoutError, Exception) as e:
                results[source_name] = SourceResult(
                    source=source_name,
                    status="TIMEOUT" if isinstance(e, asyncio.TimeoutError) else "FAILED",
                    latency_ms=int(segment_config.max_total_timeout_s * 1000),
                    data_ref=None,
                )

        # 5. Send to data-quality-engine for enrichment
        await self.kafka.send("dq.enriched", {
            "request_id": str(request.request_id),
            "borrower_id": str(request.borrower_id),
            "segment": request.segment_hint,
            "source_results": {k: v.dict() for k, v in results.items()},
        })

        return self._build_response(request.request_id, results, required_sources)

    @circuit(failure_threshold=5, recovery_timeout=30)
    async def _fetch_with_circuit_breaker(self, client, borrower_id, token, timeout):
        """Individual source fetch with circuit breaker pattern."""
        import time
        start = time.monotonic()
        try:
            data = await asyncio.wait_for(
                client.fetch(borrower_id, token), timeout=timeout
            )
            latency = int((time.monotonic() - start) * 1000)
            return SourceResult(source=client.name, status="SUCCESS",
                              latency_ms=latency, data_ref=data.ref_id)
        except asyncio.TimeoutError:
            latency = int((time.monotonic() - start) * 1000)
            return SourceResult(source=client.name, status="TIMEOUT",
                              latency_ms=latency, data_ref=None)
```

### Segment Config Format (1 of 38)

```yaml
# configs/segments/dairy_farmer.yaml
segment:
  id: DAIRY_FARMER
  display_name: "Dairy Farmer"
  languages: [gu, hi, mr, kn]
  geography: [GJ, MH, RJ, KA, UP]

  data_sources:
    required:
      - name: CREDIT_BUREAU_CIBIL
        timeout_ms: 3000
        retry: 1
      - name: BANK_ACCOUNT_AA
        timeout_ms: 5000
        retry: 1
      - name: DAIRY_COOPERATIVE
        timeout_ms: 3000
        retry: 0
      - name: LAND_RECORDS
        timeout_ms: 5000
        retry: 1
    optional:
      - name: SATELLITE_NDVI
        timeout_ms: 3000
      - name: IMD_WEATHER
        timeout_ms: 2000
      - name: PMFBY_INSURANCE
        timeout_ms: 3000
    max_total_timeout_s: 8

  scoring:
    model_id: DairyScore_v1
    features:
      - name: cooperative_production_liters_per_day
        source: DAIRY_COOPERATIVE
        weight: 0.25
        fallback: null
      - name: bank_balance_6m_stability
        source: BANK_ACCOUNT_AA
        weight: 0.25
        fallback: 0.5
      - name: land_ownership_acres
        source: LAND_RECORDS
        weight: 0.15
        fallback: null
      - name: bureau_score
        source: CREDIT_BUREAU_CIBIL
        weight: 0.15
        fallback: null     # Required, no fallback
      - name: device_inclusion_score
        source: DEVICECONNECT
        weight: 0.10
        fallback: 500
      - name: cooperative_membership_years
        source: DAIRY_COOPERATIVE
        weight: 0.10
        fallback: 1

  product:
    types: [SEASONAL_DYNAMIC, MORATORIUM_WITH_TRIGGER]
    amount_range: {min: 25000, max: 500000}
    tenure_range_months: {min: 6, max: 36}
    insurance: PASHU_BIMA
    collection_strategy: COOPERATIVE_DEDUCTION

  monitoring:
    watchers:
      - type: COOPERATIVE_PRODUCTION
        poll_interval: DAILY
        threshold: {metric: pouring_liters, drop_percent: 40}
      - type: WEATHER
        poll_interval: HOURLY
        threshold: {metric: temperature_celsius, above: 38}
      - type: REPAYMENT
        trigger: ON_DUE_DATE

  voice:
    primary_language: gu
    fallback_language: hi
    greeting_template: "નમસ્તે {name}, હું તમારો લોન સહાયક છું"
    domain_vocab: [loan, EMI, buffalo, cooperative, harvest, monsoon, milk, veterinary]
```

## 2.3 S3: `bankconnect-svc` -- Statement Parser Architecture

```python
# services/bankconnect-svc/src/bankconnect_svc/domain/services.py

class BankStatementAnalyzer:
    """Multi-bank statement parser with cluster intelligence."""

    TRANSACTION_CATEGORIES = [
        "SALARY", "UPI_CREDIT", "UPI_DEBIT", "CASH_DEPOSIT", "CASH_WITHDRAWAL",
        "LOAN_EMI", "COOPERATIVE_CREDIT", "PLATFORM_PAYOUT", "RENT", "UTILITY",
        "TRANSFER_IN", "TRANSFER_OUT", "BOUNCE", "INTEREST", "OTHER"
    ]

    async def analyze(self, statement_ref: str, borrower_id: UUID) -> BankConnectResult:
        # 1. Parse statement (PDF or image)
        raw_txns = await self._parse_statement(statement_ref)

        # 2. Categorize each transaction
        categorized = await self._categorize_transactions(raw_txns)

        # 3. Compute income metrics
        monthly_income = self._compute_monthly_income(categorized)
        income_stability = self._compute_stability(monthly_income)

        # 4. Run cluster intelligence (fraud detection)
        fraud_flags = await self._detect_clusters(categorized)

        # 5. Compute FBCS (FinBox Credit Score) 0-900
        fbcs = self._compute_fbcs(monthly_income, income_stability, fraud_flags, categorized)

        # 6. Compute FOIR
        existing_emis = self._extract_emi_obligations(categorized)
        foir = sum(existing_emis) / (sum(m["amount"] for m in monthly_income) / len(monthly_income))

        return BankConnectResult(
            fbcs_score=fbcs,
            monthly_income=monthly_income,
            income_stability=income_stability,
            fraud_flags=fraud_flags,
            foir=foir,
            transaction_summary=self._summarize(categorized),
        )

    async def _detect_clusters(self, txns: list[CategorizedTxn]) -> list[str]:
        """Graph-based cluster detection for fraud patterns."""
        flags = []

        # Circular transfer: A->B->C->A within 48h window
        transfer_graph = self._build_transfer_graph(txns)
        cycles = self._find_cycles(transfer_graph, max_length=5, window_hours=48)
        if cycles:
            flags.append("CIRCULAR_TRANSFER")

        # Round-trip: out + in same amount within 48h
        round_trips = self._find_round_trips(txns, window_hours=48, tolerance=0.05)
        if round_trips:
            flags.append("ROUND_TRIP")

        # Income inflation: sudden spike in deposits before application
        if self._detect_income_inflation(txns, lookback_months=2, spike_threshold=2.0):
            flags.append("INCOME_INFLATION")

        # Bounce frequency
        bounces = [t for t in txns if t.category == "BOUNCE"]
        if len(bounces) / max(len(txns), 1) > 0.05:
            flags.append("BOUNCE_FREQUENCY")

        return flags
```

## 2.4 Acceptance Criteria (Gate 1)

```
[ ] ULI orchestrator: parallel fetch for dairy (5 sources) completes in <5s p95
[ ] ULI orchestrator: circuit breaker opens after 5 failures, half-open after 30s
[ ] ULI orchestrator: idempotency key prevents duplicate fetches
[ ] Data quality: entity resolution matches across 3+ sources (Jaro-Winkler)
[ ] Data quality: confidence score correctly weights sources/freshness/anomaly
[ ] Data quality: graceful degradation reduces amount when sources missing
[ ] BankConnect: parses SBI, HDFC, Kotak, BoB PDF statements correctly
[ ] BankConnect: FBCS 0-900 score computed, validated against known profiles
[ ] BankConnect: cluster intelligence detects planted circular transfers
[ ] DeviceConnect: Android SDK collects, uploads, FIS score returned
[ ] DeviceConnect: raw data deleted after feature extraction
[ ] Identity: Aadhaar OTP flow completes in UIDAI sandbox
[ ] Identity: AA consent flow completes with Setu/OneMoney sandbox
[ ] All PII encrypted in PostgreSQL (verified via pg_catalog query)
[ ] Consent ledger: entry created for every data element fetched
[ ] Kafka events: uli.fetch.completed emitted with correct schema
[ ] Load test: 100 concurrent dairy fetches, all complete, no OOM
[ ] Unit tests: >90% coverage on all 5 services
[ ] Integration test: full fetch -> enrich -> persist pipeline
```

---

# MISSION 3: DECISIONING ENGINE (S6-S8)

**Objective:** Build Sentinel policy engine, product design engine, and fraud detection.

**Dependencies:** Mission 2 (data layer must provide enriched profiles)

**Duration:** Weeks 9-16

## 3.1 Database Schema -- Decisioning (Migration 003)

```sql
-- migrations/alembic/versions/003_decisioning_tables.py

-- ============================================================
-- POLICY DEFINITIONS (versioned, shadow-capable)
-- ============================================================
CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id       UUID NOT NULL REFERENCES lenders(id),
    policy_id       VARCHAR(100) NOT NULL,              -- e.g., DAIRY_ANIMAL_PURCHASE
    version         INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT'
                    CHECK (status IN ('DRAFT', 'SHADOW', 'CANARY', 'PRODUCTION', 'RETIRED')),
    segment         VARCHAR(50) NOT NULL,
    definition_yaml TEXT NOT NULL,                      -- Full policy YAML
    definition_hash VARCHAR(64) NOT NULL,               -- SHA-256 for diff detection
    promoted_by     UUID REFERENCES users(id),
    promoted_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_policy_version
    ON policies(lender_id, policy_id, version);
CREATE INDEX idx_policy_active
    ON policies(lender_id, segment, status) WHERE status IN ('SHADOW', 'CANARY', 'PRODUCTION');

-- ============================================================
-- CREDIT DECISIONS
-- ============================================================
CREATE TABLE credit_decisions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id       UUID NOT NULL REFERENCES lenders(id),
    borrower_id     UUID NOT NULL REFERENCES borrower_profiles(id),
    loan_application_id UUID,                           -- FK added in migration 004
    policy_id       UUID NOT NULL REFERENCES policies(id),
    policy_version  INTEGER NOT NULL,
    is_shadow       BOOLEAN NOT NULL DEFAULT false,     -- Shadow decisions don't affect state

    -- Scoring
    raw_score       INTEGER NOT NULL,                   -- 0-900
    grade           VARCHAR(5) NOT NULL,                -- A1, A2, B1, B2
    feature_values  JSONB NOT NULL,                     -- Input features with values
    feature_weights JSONB NOT NULL,                     -- Weights used

    -- Decision
    decision        VARCHAR(20) NOT NULL
                    CHECK (decision IN ('APPROVE', 'REFER', 'REJECT')),
    reason_codes    TEXT[] NOT NULL,                     -- Human-readable codes
    shap_values     JSONB,                              -- SHAP explainability

    -- Product (if approved)
    proposed_amount DECIMAL(12,2),
    proposed_rate   DECIMAL(5,2),
    proposed_tenure INTEGER,                            -- months
    schedule_type   VARCHAR(30),

    -- Fraud
    fraud_flags     TEXT[],
    fraud_score     DECIMAL(3,2),

    -- Confidence
    data_confidence DECIMAL(3,2) NOT NULL,

    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_decision_borrower ON credit_decisions(borrower_id);
CREATE INDEX idx_decision_policy ON credit_decisions(policy_id, is_shadow);
CREATE INDEX idx_decision_lender_date ON credit_decisions(lender_id, decided_at);

-- ============================================================
-- REPAYMENT SCHEDULES (generated by product-engine)
-- ============================================================
CREATE TABLE repayment_schedules (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    loan_id         UUID NOT NULL,                      -- FK to loan_applications (migration 004)
    installment_no  INTEGER NOT NULL,
    due_date        DATE NOT NULL,
    principal       DECIMAL(12,2) NOT NULL,
    interest        DECIMAL(12,2) NOT NULL,
    total_emi       DECIMAL(12,2) NOT NULL,
    season_tag      VARCHAR(20),                        -- FLUSH, LEAN, TRANSITION (for seasonal)
    status          VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED'
                    CHECK (status IN ('SCHEDULED', 'PAID', 'PARTIAL', 'OVERDUE', 'MORATORIUM', 'SKIPPED')),
    paid_amount     DECIMAL(12,2) DEFAULT 0,
    paid_at         TIMESTAMPTZ,
    moratorium_reason VARCHAR(100)
);
CREATE INDEX idx_schedule_loan ON repayment_schedules(loan_id, due_date);
```

## 3.2 S6: `sentinel-core` -- Policy Evaluation Engine

```python
# services/sentinel-core/src/sentinel_core/domain/engine.py

class SentinelEngine:
    """3-layer policy evaluation: structural -> behavioral -> supervisor."""

    def __init__(self, policy_repo, model_registry, fraud_engine, feature_store):
        self.policy_repo = policy_repo
        self.model_registry = model_registry
        self.fraud = fraud_engine
        self.features = feature_store

    @audited(entity_type="DECISION", action="EVALUATED")
    async def evaluate(self, request: DecisionRequest) -> DecisionResult:
        # 1. Load active policy for segment + lender
        policy = await self.policy_repo.get_active(
            request.lender_id, request.segment
        )

        # Also load shadow policies for parallel evaluation
        shadow_policies = await self.policy_repo.get_shadows(
            request.lender_id, request.segment
        )

        # 2. LAYER 1: Structural checks (hard rules, no ML)
        structural_result = self._evaluate_structural(policy, request)
        if structural_result.is_terminal:
            return structural_result  # Hard reject (velocity, blacklist)

        # 3. LAYER 2: ML scoring
        features = await self.features.compute(
            request.borrower_id, policy.scoring.features
        )
        model = self.model_registry.get(policy.scoring.model)
        raw_score = model.predict(features)
        grade = self._score_to_grade(raw_score, policy.scoring.grade_bands)

        # 4. LAYER 2: Fraud check (parallel with scoring for latency)
        fraud_result = await self.fraud.check(request.borrower_id, features)

        # 5. LAYER 3: Decision rules (configurable per policy)
        decision = self._apply_decision_rules(
            policy.decision_rules, raw_score, fraud_result, request.data_confidence
        )

        # 6. Explainability
        shap_values = model.explain(features) if decision != "APPROVE" else None
        reason_codes = self._generate_reason_codes(
            decision, raw_score, fraud_result, request.data_confidence
        )

        # 7. Product design (if approved)
        product = None
        if decision == "APPROVE":
            product = await self._design_product(
                policy, raw_score, grade, features, request
            )

        # 8. Run shadow policies in parallel (non-blocking)
        for shadow in shadow_policies:
            asyncio.create_task(self._evaluate_shadow(shadow, request, features))

        result = DecisionResult(
            borrower_id=request.borrower_id,
            policy_id=policy.id,
            raw_score=raw_score,
            grade=grade,
            decision=decision,
            reason_codes=reason_codes,
            shap_values=shap_values,
            fraud_flags=fraud_result.flags,
            proposed_amount=product.amount if product else None,
            proposed_rate=product.rate if product else None,
            data_confidence=request.data_confidence,
        )

        # 9. Emit decision event
        await self.kafka.send("sentinel.decision.completed", result.dict())

        # 10. Write to context graph
        await self.kafka.send("graph.ingest", {
            "type": "DECISION_TRACE",
            "borrower_id": str(request.borrower_id),
            "input": features,
            "reasoning": {"score": raw_score, "grade": grade, "rules": policy.decision_rules},
            "output": result.dict(),
        })

        return result

    def _apply_decision_rules(self, rules, score, fraud, confidence):
        """Configurable decision logic from policy YAML."""
        if fraud.has_hard_reject_flags(rules.auto_reject.or_fraud_flags):
            return "REJECT"
        if score <= rules.auto_reject.max_score:
            return "REJECT"
        if (score >= rules.auto_approve.min_score
            and fraud.flag_count <= rules.auto_approve.max_fraud_flags
            and confidence >= rules.auto_approve.min_confidence):
            return "APPROVE"
        return "REFER"
```

## 3.3 S7: `product-engine` -- Schedule Generator

```python
# services/product-engine/src/product_engine/domain/generators.py

class SeasonalDynamicGenerator:
    """Scales EMI by flush/lean/transition months from cooperative calendar."""

    async def generate(self, params: ProductParams) -> RepaymentSchedule:
        calendar = await self.coop_client.get_seasonal_calendar(
            params.segment, params.pincode
        )

        # Map months to season weights
        # FLUSH (high production) = 1.5x, TRANSITION = 1.0x, LEAN (low) = 0.5x
        month_weights = {}
        for month in range(1, 13):
            if month in calendar.flush_months:
                month_weights[month] = 1.5
            elif month in calendar.lean_months:
                month_weights[month] = 0.5
            else:
                month_weights[month] = 1.0

        # Calculate total weight sum for tenure period
        start_month = params.start_date.month
        total_weight = sum(
            month_weights[(start_month + i - 1) % 12 + 1]
            for i in range(params.tenure_months)
        )

        # Calculate per-month EMI such that sum(EMIs) = total_repayment
        total_repayment = self._compute_total_repayment(
            params.amount, params.rate, params.tenure_months
        )
        base_emi = total_repayment / total_weight

        schedule = []
        remaining_principal = params.amount
        for i in range(params.tenure_months):
            month = (start_month + i - 1) % 12 + 1
            weight = month_weights[month]
            emi = round(base_emi * weight, 2)

            interest = round(remaining_principal * (params.rate / 100 / 12), 2)
            principal = emi - interest
            remaining_principal -= principal

            schedule.append(Installment(
                installment_no=i + 1,
                due_date=params.start_date + relativedelta(months=i),
                principal=principal,
                interest=interest,
                total_emi=emi,
                season_tag="FLUSH" if weight > 1.2 else ("LEAN" if weight < 0.8 else "TRANSITION"),
            ))

        # Verify: sum of all EMIs ≈ total_repayment (within INR 1 rounding)
        assert abs(sum(s.total_emi for s in schedule) - total_repayment) < 1.0

        return RepaymentSchedule(installments=schedule)
```

## 3.4 Acceptance Criteria (Gate 2)

```
[ ] Sentinel: dairy policy evaluates in <200ms p95 (500 concurrent)
[ ] Sentinel: shadow mode produces results without state mutation
[ ] Sentinel: champion/challenger splits 90/10 correctly
[ ] Sentinel: decision rules match policy YAML (20 test profiles)
[ ] Product engine: seasonal schedule sum(EMI) = principal + interest
[ ] Product engine: flush months EMI > lean months EMI
[ ] Product engine: all 8 generators produce valid schedules
[ ] Fraud: velocity check blocks 3rd app from same Aadhaar in 48h
[ ] Fraud: circular transfer detected in planted bank statement
[ ] MCP: all 8 decisioning tools callable via JSON-RPC
[ ] Explainability: SHAP values returned for rejected decisions
[ ] KFS: RBI-compliant Key Fact Statement generated (all mandatory fields)
[ ] Risk-based pricing: score 780 -> 14%, score 580 -> 17%
[ ] Drift detection: PSI alert fires on planted distribution shift
[ ] Shadow comparison: production vs shadow side-by-side report generated
[ ] All decisions persisted with full feature values + weights
[ ] Decision traces written to context graph via Kafka
```

---

# MISSION 4: DISTRIBUTION LAYER (S9-S12)

**Objective:** Build lender matching (Prism), loan origination (LOS), journey studio, and notification service.

**Dependencies:** Mission 3

**Duration:** Weeks 13-20

## 4.1 Database Schema -- Distribution (Migration 004)

```sql
-- migrations/alembic/versions/004_distribution_tables.py

-- ============================================================
-- LOAN APPLICATIONS (full lifecycle state machine)
-- ============================================================
CREATE TABLE loan_applications (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    borrower_id         UUID NOT NULL REFERENCES borrower_profiles(id),
    segment             VARCHAR(50) NOT NULL,
    channel             VARCHAR(20) NOT NULL
                        CHECK (channel IN ('VOICE', 'WHATSAPP', 'WEB', 'PARTNER', 'DASHBOARD')),

    -- State machine
    status              VARCHAR(30) NOT NULL DEFAULT 'CREATED'
                        CHECK (status IN (
                            'CREATED', 'KYC_CHECK', 'KYC_FAILED',
                            'DATA_FETCH', 'DECISIONING',
                            'APPROVED', 'REFERRED', 'REJECTED',
                            'PRODUCT_DESIGN', 'LENDER_MATCHING',
                            'OFFER_PRESENTED', 'ACCEPTED', 'DECLINED',
                            'DISBURSING', 'ACTIVE',
                            'CLOSED', 'GRADUATED', 'WRITTEN_OFF'
                        )),
    sub_status          VARCHAR(30)
                        CHECK (sub_status IS NULL OR sub_status IN (
                            'CURRENT', 'MORATORIUM', 'ADJUSTED',
                            'DELINQUENT', 'RESTRUCTURED', 'EMERGENCY_LOAN'
                        )),

    -- Decision
    decision_id         UUID REFERENCES credit_decisions(id),
    matched_lender_id   UUID REFERENCES lenders(id),    -- Selected by Prism
    co_lend_structure   JSONB,                          -- Co-lending split details

    -- Product
    approved_amount     DECIMAL(12,2),
    approved_rate       DECIMAL(5,2),
    approved_tenure     INTEGER,
    schedule_type       VARCHAR(30),

    -- Disbursement
    disbursed_amount    DECIMAL(12,2),
    disbursed_at        TIMESTAMPTZ,
    disbursement_ref    VARCHAR(255),                   -- UTR/NEFT reference
    disbursement_mode   VARCHAR(20),                    -- UPI, NEFT, IMPS

    -- Insurance
    insurance_policy_id VARCHAR(255),
    insurance_type      VARCHAR(30),

    -- Temporal workflow
    temporal_workflow_id VARCHAR(255),

    -- Metadata
    psl_category        VARCHAR(50),                    -- AUTO-CLASSIFIED
    psl_sub_category    VARCHAR(50),
    rejection_reasons   TEXT[],
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_loan_borrower ON loan_applications(borrower_id);
CREATE INDEX idx_loan_lender_status ON loan_applications(lender_id, status);
CREATE INDEX idx_loan_segment ON loan_applications(lender_id, segment);
ALTER TABLE loan_applications ENABLE ROW LEVEL SECURITY;
CREATE POLICY loan_rls ON loan_applications
    USING (lender_id = current_setting('app.current_lender_id')::UUID);

-- ============================================================
-- LENDER PROFILES (Prism configuration)
-- ============================================================
CREATE TABLE lender_profiles (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    segments            TEXT[] NOT NULL,                 -- Eligible segments
    geography           TEXT[] NOT NULL,                 -- State codes
    ticket_range        JSONB NOT NULL,                  -- {min, max}
    psl_targets         JSONB NOT NULL,                  -- {agriculture: {current, target}, ...}
    risk_appetite       JSONB NOT NULL,                  -- {max_npa, fldg_required, co_lending}
    disbursement_api    JSONB,                           -- {url, auth_type, credentials_vault_path}
    co_lending_config   JSONB,                           -- {share, partner_lender_id}
    active              BOOLEAN NOT NULL DEFAULT true,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- JOURNEY DEFINITIONS (A/B testing)
-- ============================================================
CREATE TABLE journey_definitions (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lender_id           UUID NOT NULL REFERENCES lenders(id),
    name                VARCHAR(255) NOT NULL,
    segment             VARCHAR(50) NOT NULL,
    channel             VARCHAR(20) NOT NULL,
    language            VARCHAR(5) NOT NULL,
    definition_json     JSONB NOT NULL,                  -- Flow definition
    ab_variant          VARCHAR(10) DEFAULT 'control',   -- 'control', 'variant_a', 'variant_b'
    traffic_pct         INTEGER DEFAULT 100,
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT'
                        CHECK (status IN ('DRAFT', 'ACTIVE', 'PAUSED', 'RETIRED')),
    conversion_stats    JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 4.2 S10: `los-engine` -- Temporal Workflow Implementation

```python
# services/los-engine/src/los_engine/workflows/origination.py

from temporalio import workflow, activity
from datetime import timedelta

@workflow.defn
class LoanOriginationWorkflow:
    """Durable loan origination workflow. Survives service restarts."""

    @workflow.run
    async def run(self, params: OriginationParams) -> OriginationResult:
        # 1. Create application record
        app = await workflow.execute_activity(
            create_application,
            params,
            start_to_close_timeout=timedelta(seconds=10),
        )

        # 2. KYC verification
        kyc_result = await workflow.execute_activity(
            run_kyc_verification,
            KYCParams(borrower_id=params.borrower_id, aadhaar_token=params.aadhaar_token),
            start_to_close_timeout=timedelta(minutes=5),
            retry_policy=RetryPolicy(maximum_attempts=2),
        )
        if not kyc_result.passed:
            await workflow.execute_activity(
                update_status, UpdateStatusParams(app.id, "KYC_FAILED"),
                start_to_close_timeout=timedelta(seconds=5),
            )
            return OriginationResult(status="KYC_FAILED", application_id=app.id)

        # 3. Data fetch (parallel ULI)
        fetch_result = await workflow.execute_activity(
            fetch_borrower_data,
            FetchParams(borrower_id=params.borrower_id, segment=params.segment),
            start_to_close_timeout=timedelta(seconds=15),
        )

        # 4. Credit decision
        decision = await workflow.execute_activity(
            run_decisioning,
            DecisionParams(borrower_id=params.borrower_id, segment=params.segment,
                          data_confidence=fetch_result.confidence_score),
            start_to_close_timeout=timedelta(seconds=5),
        )

        if decision.decision == "REJECT":
            await self._handle_rejection(app.id, decision)
            return OriginationResult(status="REJECTED", application_id=app.id)

        if decision.decision == "REFER":
            # Wait for human decision (up to 48 hours)
            human_decision = await workflow.execute_activity(
                route_to_human_queue,
                ReferParams(app_id=app.id, decision=decision),
                start_to_close_timeout=timedelta(hours=48),
            )
            if human_decision.outcome != "APPROVE":
                return OriginationResult(status="REJECTED", application_id=app.id)

        # 5. Product design
        product = await workflow.execute_activity(
            design_product,
            ProductParams(decision=decision, segment=params.segment),
            start_to_close_timeout=timedelta(seconds=5),
        )

        # 6. Lender matching (Prism)
        matched = await workflow.execute_activity(
            match_lender,
            MatchParams(application=app, decision=decision, product=product),
            start_to_close_timeout=timedelta(seconds=5),
        )

        # 7. Present offer to borrower (wait for acceptance)
        offer_response = await workflow.wait_condition(
            lambda: self.offer_accepted is not None,
            timeout=timedelta(hours=24),
        )
        if not self.offer_accepted:
            return OriginationResult(status="DECLINED", application_id=app.id)

        # 8. E-sign + Insurance + Disbursement (sequential, each must succeed)
        await workflow.execute_activity(
            execute_esign,
            ESignParams(borrower_id=params.borrower_id, loan_id=app.id),
            start_to_close_timeout=timedelta(minutes=10),
        )

        await workflow.execute_activity(
            enroll_insurance,
            InsuranceParams(borrower_id=params.borrower_id, type=product.insurance_type),
            start_to_close_timeout=timedelta(minutes=5),
        )

        disbursement = await workflow.execute_activity(
            initiate_disbursement,
            DisbursementParams(
                loan_id=app.id,
                amount=product.amount,
                lender_api=matched.disbursement_api,
            ),
            start_to_close_timeout=timedelta(minutes=15),
            retry_policy=RetryPolicy(maximum_attempts=3, backoff_coefficient=2.0),
        )

        # 9. Activate loan + start monitoring
        await workflow.execute_activity(
            activate_loan,
            ActivateParams(loan_id=app.id, disbursement_ref=disbursement.utr),
            start_to_close_timeout=timedelta(seconds=10),
        )

        await workflow.execute_activity(
            start_monitoring,
            MonitorParams(loan_id=app.id, segment=params.segment),
            start_to_close_timeout=timedelta(seconds=10),
        )

        return OriginationResult(status="ACTIVE", application_id=app.id)

    # Signal handler for offer acceptance
    @workflow.signal
    async def on_offer_response(self, accepted: bool):
        self.offer_accepted = accepted
```

## 4.3 Acceptance Criteria (Gate 3)

```
[ ] Prism: routes to highest PSL-gap bank (10 test applications)
[ ] Prism: co-lending structure generated for FPO > 2L (80/20 split)
[ ] LOS: Temporal workflow completes full lifecycle in staging
[ ] LOS: human referral queue works (48h timeout, signal-based acceptance)
[ ] LOS: idempotent disbursement (retry doesn't double-disburse)
[ ] Journey Studio: GenAI generates valid journey from NL prompt
[ ] Journey Studio: A/B traffic split verified (chi-squared on conversion)
[ ] Notification: SMS, WhatsApp template, email dispatch verified
[ ] Dashboard: portfolio overview renders with correct AUM/DPD/NPA
[ ] Dashboard: policy builder create -> shadow -> results flow works
[ ] Dashboard: copilot answers via MCP (verified tool invocation)
[ ] Dashboard: RBAC enforced (analyst cannot promote without CRO)
[ ] PSL auto-classification: dairy loan classified as AGRICULTURE
[ ] Full audit trail: single application has 15+ events logged
[ ] Temporal: workflow survives service restart (durability test)
```

---

# MISSION 5: AGENT INTELLIGENCE LAYER (S13-S16)

**Objective:** Build the multi-agent runtime, voice pipeline, model routing, and knowledge service.

**Dependencies:** Mission 4 (LOS + journey must exist for agents to call)

**Duration:** Weeks 17-24

## 5.1 S13: `agent-runtime` -- 16-Step Turn Processor

```python
# services/agent-runtime/src/agent_runtime/domain/turn_processor.py

class TurnProcessor:
    """16-step turn processor from voiceai-platform, adapted for lending."""

    STEPS = [
        "receive_input",           # 1. Raw audio/text from channel
        "detect_language",         # 2. Language identification
        "transcribe",              # 3. ASR (if voice)
        "extract_context",         # 4. Load graph memory for returning borrowers
        "classify_intent",         # 5. NLU: 8 lending intents
        "extract_entities",        # 6. Amount, tenure, segment, asset type
        "check_policy",            # 7. Policy engine: is this action allowed?
        "route_to_agent",          # 8. Select specialized agent for this intent
        "execute_fast_brain",      # 9. Parallel: cached response for simple queries
        "execute_slow_brain",      # 10. Parallel: MCP tool call for complex queries
        "merge_responses",         # 11. Combine fast/slow brain results
        "check_empathy",           # 12. Distress detection -> human escalation
        "apply_compliance",        # 13. Filter response through compliance rules
        "generate_response",       # 14. LLM generates natural language response
        "synthesize_speech",       # 15. TTS (if voice)
        "log_and_learn",           # 16. Audit trail + graph memory + RL signal
    ]

    async def process_turn(self, input: ChannelInput) -> ChannelOutput:
        context = TurnContext(input=input)

        for step_name in self.STEPS:
            step = self.step_registry[step_name]
            context = await step.execute(context)

            if context.should_escalate:
                return await self._escalate_to_human(context)
            if context.has_response:
                break

        # Log complete turn
        await self._log_turn(context)
        return context.output
```

## 5.2 S14: `voice-pipeline` -- Indian Language ASR/TTS

```python
# services/voice-pipeline/src/voice_pipeline/domain/asr.py

class IndianLanguageASR:
    """Whisper large-v3 fine-tuned for Indian languages + lending domain vocab."""

    LENDING_VOCAB_BOOST = {
        "gu": ["લોન", "EMI", "ભેંસ", "સહકારી", "લણણી", "ચોમાસું", "દૂધ", "આધાર"],
        "hi": ["लोन", "EMI", "ब्याज", "किस्त", "आधार", "भैंस", "सहकारी", "मानसून"],
        "ta": ["கடன்", "வட்டி", "தவணை", "ஆதார்"],
        # ... 12 languages
    }

    async def transcribe(self, audio_chunk: bytes, language: str) -> TranscriptionResult:
        # Primary: Deepgram with vocab boost
        try:
            result = await self.deepgram.transcribe(
                audio_chunk,
                language=language,
                keywords=self.LENDING_VOCAB_BOOST.get(language, []),
                model="nova-2",
            )
            if result.confidence > 0.7:
                return result
        except Exception:
            pass

        # Fallback: Whisper large-v3 fine-tuned
        result = await self.whisper.transcribe(
            audio_chunk,
            language=language,
            task="transcribe",
        )
        return result


class IndianLanguageTTS:
    """Bhashini (government-backed Indian TTS) with ElevenLabs fallback."""

    async def synthesize(self, text: str, language: str, voice_config: dict) -> bytes:
        try:
            return await self.bhashini.synthesize(
                text=text,
                language=language,
                gender=voice_config.get("gender", "female"),
                speed=voice_config.get("speed", 1.0),
            )
        except Exception:
            return await self.elevenlabs.synthesize(
                text=text,
                voice_id=voice_config.get("elevenlabs_voice_id"),
            )
```

## 5.3 Acceptance Criteria (Gate 4)

```
[ ] Voice agent: dairy journey in Gujarati end-to-end <10 min
[ ] ASR: WER <15% on 200 labeled Gujarati lending utterances
[ ] WhatsApp agent: vendor journey in Hindi with photo OCR
[ ] Intent classification: >90% accuracy across 8 intents, 3 languages
[ ] Empathy detection: distress keywords trigger human escalation
[ ] Fast brain: FAQ response <200ms (cached)
[ ] Slow brain: credit decision via MCP <2s
[ ] Returning borrower: graph memory context loaded ("Welcome back")
[ ] Conversation log: full transcript + audio stored in audit trail
[ ] Constellation: routes FAQ to fast model, decisioning to capable model
[ ] Agent-to-LOS: voice agent triggers Temporal workflow, offer presented
[ ] 50 concurrent voice calls: no audio degradation, <1.5s response
[ ] All channel adapters: voice, WhatsApp, web, partner SDK tested
```

---

# MISSION 6: CONTEXT GRAPH + MONITORING (S17-S18)

**Objective:** Build the Neo4j context graph and continuous monitoring agent.

**Dependencies:** Mission 4 (loans must be active), Mission 5 (agents must be deployed)

**Duration:** Weeks 21-30

## 6.1 S17: `context-graph` -- Neo4j Schema + Query API

```cypher
// Context Graph Schema (from contextgraphs-platform-fintech)

// Core constraints
CREATE CONSTRAINT FOR (b:Borrower) REQUIRE b.id IS UNIQUE;
CREATE CONSTRAINT FOR (l:Loan) REQUIRE l.id IS UNIQUE;
CREATE CONSTRAINT FOR (p:Policy) REQUIRE (p.id, p.version) IS UNIQUE;
CREATE CONSTRAINT FOR (c:Cooperative) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT FOR (pc:Pincode) REQUIRE pc.code IS UNIQUE;

// Relationship types with properties
// (:Borrower)-[:HAS_LOAN {originated_at, channel}]->(:Loan)
// (:Borrower)-[:MEMBER_OF {since, role}]->(:Cooperative)
// (:Borrower)-[:OWNS {area_acres, verified}]->(:Land)
// (:Borrower)-[:LIVES_IN]->(:Pincode)
// (:Borrower)-[:HAD_CONVERSATION {channel, date, summary_embedding}]->(:Interaction)
// (:Loan)-[:DECIDED_BY {score, grade}]->(:Policy)
// (:Loan)-[:MATCHED_TO]->(:Lender)
// (:Loan)-[:HAS_REPAYMENT {amount, on_time}]->(:RepaymentEvent)
// (:Loan)-[:HAS_MORATORIUM {trigger, duration_months}]->(:MoratoriumEvent)
// (:Loan)-[:HAS_INTERVENTION {type, outcome}]->(:InterventionEvent)
// (:Loan)-[:GRADUATED_TO]->(:Loan)
// (:Pincode)-[:HAD_WEATHER {type, severity, date}]->(:WeatherEvent)
// (:Cooperative)-[:RECORDED_PRODUCTION {date, volume}]->(:ProductionRecord)

// Decision Traces (first-class objects)
// (:DecisionTrace)-[:FOR_BORROWER]->(:Borrower)
// (:DecisionTrace)-[:USED_POLICY]->(:Policy)
// (:DecisionTrace)-[:PRODUCED_DECISION {decision, score}]->(:Loan)
// (:DecisionTrace {input_features, reasoning, output, timestamp})
```

```python
# services/context-graph/src/context_graph/domain/queries.py

class ContextGraphQueries:
    """Pre-built query patterns for agent and lender use."""

    async def get_borrower_full_context(self, borrower_id: UUID) -> dict:
        """Returns complete borrower subgraph for agent greeting."""
        result = await self.neo4j.execute("""
            MATCH (b:Borrower {id: $id})
            OPTIONAL MATCH (b)-[r1:HAS_LOAN]->(l:Loan)
            OPTIONAL MATCH (b)-[r2:MEMBER_OF]->(c:Cooperative)
            OPTIONAL MATCH (l)-[r3:HAS_REPAYMENT]->(rep:RepaymentEvent)
            OPTIONAL MATCH (l)-[r4:HAS_INTERVENTION]->(int:InterventionEvent)
            RETURN b, collect(DISTINCT l) as loans,
                   collect(DISTINCT c) as cooperatives,
                   collect(DISTINCT rep) as repayments,
                   collect(DISTINCT int) as interventions
        """, {"id": str(borrower_id)})
        return self._format_context(result)

    async def detect_fraud_ring(self, borrower_id: UUID, hops: int = 3) -> list:
        """Multi-hop graph query for shared reference persons / devices."""
        return await self.neo4j.execute("""
            MATCH (b:Borrower {id: $id})-[:REFERENCED_BY]->(ref:ReferencePerson)
            MATCH (ref)<-[:REFERENCED_BY]-(other:Borrower)
            WHERE other.id <> $id
            OPTIONAL MATCH (other)-[:HAS_LOAN]->(l:Loan)
            WHERE l.status IN ['DELINQUENT', 'WRITTEN_OFF']
            RETURN other.id as borrower_id, count(l) as bad_loans,
                   ref.name as shared_reference
            ORDER BY bad_loans DESC
        """, {"id": str(borrower_id)})

    async def find_similar_defaulters(self, borrower_id: UUID) -> list:
        """Find borrowers in same cooperative/pincode who defaulted."""
        return await self.neo4j.execute("""
            MATCH (b:Borrower {id: $id})-[:MEMBER_OF]->(c:Cooperative)
            MATCH (c)<-[:MEMBER_OF]-(b2:Borrower)-[:HAS_LOAN]->(l:Loan)
            WHERE l.sub_status = 'DELINQUENT' AND b2.id <> $id
            RETURN b2.id, l.id as loan_id, l.sub_status,
                   count(*) as default_count
        """, {"id": str(borrower_id)})
```

## 6.2 S18: `monitoring-agent` -- Watcher Implementation

```python
# services/monitoring-agent/src/monitoring_agent/domain/watchers.py

class CooperativeProductionWatcher:
    """Polls dairy cooperative API daily, detects production drops."""

    async def watch(self, loan: ActiveLoan) -> Optional[TriggerEvent]:
        current = await self.coop_client.get_daily_production(
            cooperative_id=loan.cooperative_id,
            borrower_id=loan.borrower_id,
        )

        # Get baseline (30-day rolling average)
        baseline = await self.repo.get_production_baseline(loan.borrower_id)

        if baseline and current:
            drop_pct = (baseline - current.liters) / baseline * 100
            threshold = loan.policy.monitoring.cooperative_threshold

            if drop_pct >= threshold:
                return TriggerEvent(
                    loan_id=loan.id,
                    trigger_type="PRODUCTION_DROP",
                    severity="HIGH" if drop_pct > 60 else "MEDIUM",
                    data={"baseline": baseline, "current": current.liters, "drop_pct": drop_pct},
                    recommended_intervention="AUTO_MORATORIUM" if drop_pct > 60 else "SCHEDULE_ADJUST",
                )
        return None


class WeatherWatcher:
    """Polls IMD API per pincode, detects extreme weather events."""

    async def watch(self, loan: ActiveLoan) -> Optional[TriggerEvent]:
        weather = await self.imd_client.get_current(loan.pincode)

        triggers = loan.policy.monitoring.weather_triggers
        for trigger in triggers:
            if trigger.type == "WEATHER_HEAT" and weather.temp_celsius > trigger.threshold:
                return TriggerEvent(
                    loan_id=loan.id,
                    trigger_type="WEATHER_HEAT",
                    severity="HIGH",
                    data={"temp": weather.temp_celsius, "threshold": trigger.threshold},
                    recommended_intervention="AUTO_MORATORIUM",
                    moratorium_months=trigger.duration_months,
                )
            if trigger.type == "FLOOD" and weather.flood_alert:
                return TriggerEvent(
                    loan_id=loan.id,
                    trigger_type="FLOOD",
                    severity="CRITICAL",
                    data={"alert_level": weather.flood_alert_level},
                    recommended_intervention="AUTO_MORATORIUM",
                    moratorium_months=2,
                )
        return None


class InterventionExecutor:
    """Executes interventions based on trigger events."""

    async def execute(self, trigger: TriggerEvent):
        intervention_type = trigger.recommended_intervention

        if intervention_type == "AUTO_MORATORIUM":
            await self._apply_moratorium(trigger)
        elif intervention_type == "SCHEDULE_ADJUST":
            await self._adjust_schedule(trigger)
        elif intervention_type == "EMERGENCY_MICROLOAN":
            await self._offer_emergency_loan(trigger)
        elif intervention_type == "GRADUATION_OFFER":
            await self._generate_graduation_offer(trigger)

        # Notify borrower via preferred channel
        await self.notification_svc.send(
            borrower_id=trigger.loan.borrower_id,
            template=f"intervention_{intervention_type.lower()}",
            data=trigger.data,
        )

        # Log to audit + graph
        await self.kafka.send("monitor.trigger.intervened", trigger.dict())
        await self.kafka.send("graph.ingest", {
            "type": "INTERVENTION",
            "loan_id": str(trigger.loan_id),
            "trigger_type": trigger.trigger_type,
            "intervention": intervention_type,
            "outcome": "PENDING",
        })

        # Emit RL signal
        await self.kafka.send("rl.signal", {
            "loan_id": str(trigger.loan_id),
            "signal_type": f"INTERVENTION_{intervention_type}",
            "trigger_data": trigger.data,
        })
```

## 6.3 Acceptance Criteria (Gate 5)

```
[ ] Context graph: borrower full context returned in <100ms
[ ] Context graph: fraud ring detected (5 apps sharing reference person)
[ ] Context graph: similar-defaulter query returns relevant results
[ ] Context graph: decision traces persisted for all decisions
[ ] Monitoring: weather trigger fires moratorium within 1 hour
[ ] Monitoring: 40% production drop triggers intervention
[ ] Monitoring: graduation offer at 12 months clean repayment
[ ] Monitoring: all 7 watchers running for active test loans
[ ] RL pipeline: signals collected, model retrained, new version in MLflow
[ ] Intervention: moratorium recalculates schedule correctly
[ ] Intervention: borrower notified via voice/WhatsApp
[ ] Audit trail: all monitoring events persisted with context
[ ] Load: 10,000 active loans monitored without performance degradation
```

---

# MISSION 7: LENDER DASHBOARD + COPILOT

**Objective:** Build the React dashboard and AI copilot that lenders use daily.

**Dependencies:** Missions 2-6 (all backend services must expose APIs)

**Duration:** Weeks 13-22 (parallel with Missions 4-6)

## 7.1 Dashboard Pages -- Implementation Specs

```
dashboard/src/pages/
├── portfolio/
│   ├── PortfolioOverview.tsx       # AUM, DPD buckets, NPA, segment chart
│   │   API: GET /api/v1/portfolio/summary?lender_id=X
│   │   Charts: recharts (bar: DPD distribution, line: origination trend)
│   │   Real-time: WebSocket for live disbursement feed
│   │
│   └── LoanDetail.tsx              # Single loan full lifecycle view
│       API: GET /api/v1/loans/{id}
│       Includes: state machine visualization, repayment timeline,
│                 intervention history, decision trace link
│
├── applications/
│   ├── ApplicationQueue.tsx        # Pending + referred cases
│   │   API: GET /api/v1/applications?status=REFERRED&lender_id=X
│   │   Features: AI assessment badge, recommended action, sort by urgency
│   │
│   └── ApplicationDetail.tsx       # Full application with decision context
│       API: GET /api/v1/applications/{id}
│       Includes: enriched profile, decision breakdown, SHAP waterfall chart,
│                 borrower conversation transcript, approve/reject buttons
│
├── policy-builder/
│   ├── PolicyList.tsx              # All policies: draft, shadow, canary, production
│   ├── PolicyEditor.tsx            # Visual editor for policy YAML
│   │   Features: drag-drop feature weights, threshold sliders,
│   │             decision rule configurator, moratorium trigger builder
│   │   API: POST /api/v1/policies (create), PUT /api/v1/policies/{id} (update)
│   │
│   └── ShadowTestResults.tsx       # Side-by-side comparison
│       API: GET /api/v1/policies/{id}/shadow-results
│       Charts: score distribution overlay, approval rate comparison,
│               projected NPA, sample decisions with explanations
│
├── copilot/
│   └── CopilotChat.tsx             # Natural language interface to MCP
│       API: POST /api/v1/copilot/query
│       Backend: Routes to mcp-server, executes tool, formats response
│       Examples: "What is dairy NPA in Gujarat?"
│                 "What if I lower minimum score to 500?"
│                 "Show me top 10 graduated borrowers"
│                 "Run what-if: expand to Rajasthan dairy"
│
├── compliance/
│   ├── PSLReport.tsx               # Auto-generated PSL classification report
│   ├── AuditTrail.tsx              # Searchable audit log per entity
│   ├── BiasReport.tsx              # Fair lending bias analysis
│   └── ConsentDashboard.tsx        # DPDP consent status per borrower
│
├── monitoring/
│   ├── MonitoringDashboard.tsx     # Real-time triggers, interventions, alerts
│   └── EarlyWarning.tsx            # Borrowers at risk (predictive)
│
└── settings/
    ├── LenderConfig.tsx            # Prism profile editor
    ├── UserManagement.tsx          # RBAC: roles, permissions
    └── IntegrationConfig.tsx       # Disbursement API, insurance, BSP
```

## 7.2 Copilot Backend -- MCP Query Router

```python
# cross-cutting/mcp-server/src/mcp_server/copilot.py

class CopilotQueryRouter:
    """Routes natural language queries to appropriate MCP tools."""

    QUERY_PATTERNS = {
        "portfolio_health": {
            "patterns": ["npa", "dpd", "portfolio", "aum", "disbursed"],
            "tool": "check_portfolio_health",
        },
        "what_if": {
            "patterns": ["what if", "what-if", "simulate", "projected"],
            "tool": "run_what_if",
        },
        "model_performance": {
            "patterns": ["gini", "ks", "model", "accuracy", "performance"],
            "tool": "fetch_model_performance",
        },
        "explain_decision": {
            "patterns": ["why", "explain", "rejected", "reason"],
            "tool": "explain_decision",
        },
        "compliance_report": {
            "patterns": ["psl", "compliance", "audit", "bias", "report"],
            "tool": "generate_compliance_report",
        },
    }

    async def route(self, query: str, lender_id: UUID) -> CopilotResponse:
        # 1. Classify query intent
        tool_name = await self._classify_intent(query)

        # 2. Extract parameters from natural language
        params = await self._extract_params(query, tool_name)
        params["lender_id"] = str(lender_id)

        # 3. Execute MCP tool
        result = await self.mcp_client.call_tool(tool_name, params)

        # 4. Format response in natural language
        formatted = await self._format_response(query, tool_name, result)

        return CopilotResponse(
            answer=formatted,
            tool_used=tool_name,
            raw_data=result,
            charts=self._generate_charts(tool_name, result),
        )
```

---

# MISSION 8: AGENT LIFECYCLE PIPELINE (SRAL)

**Objective:** Implement the eval/shadow/canary deployment pipeline for all agents.

**Dependencies:** Mission 5 (agents must be deployed)

**Duration:** Weeks 20-26

## 8.1 Eval Framework (from caas-casestudy + voiceai-platform)

```python
# packages/platform-common/src/platform_common/lifecycle/eval.py

class AgentEvalSuite:
    """tau-bench + pass^k evaluation framework for lending agents."""

    SCORERS = {
        "task_completion": TaskCompletionScorer(),      # Did agent complete the task?
        "policy_compliance": PolicyComplianceScorer(),   # Did agent follow lending policy?
        "information_accuracy": AccuracyScorer(),        # Were facts correct?
        "empathy": EmpathyScorer(),                      # Was response empathetic?
        "latency": LatencyScorer(),                      # Response time within SLA?
        "language_quality": LanguageQualityScorer(),     # Grammar, fluency in target language?
    }

    async def evaluate(self, agent_id: str, eval_dataset: str) -> EvalReport:
        dataset = await self.load_dataset(eval_dataset)
        results = []

        for scenario in dataset.scenarios:
            # Run agent on scenario
            conversation = await self.agent_runner.run_scenario(agent_id, scenario)

            # Score with all scorers
            scores = {}
            for scorer_name, scorer in self.SCORERS.items():
                scores[scorer_name] = await scorer.score(conversation, scenario.expected)

            results.append(ScenarioResult(
                scenario_id=scenario.id,
                scores=scores,
                pass_all=all(s >= scorer.threshold for s, scorer in zip(scores.values(), self.SCORERS.values())),
            ))

        # pass^k metric (from voiceai-platform)
        pass_at_k = self._compute_pass_at_k(results, k=3)

        return EvalReport(
            agent_id=agent_id,
            total_scenarios=len(results),
            pass_rate=sum(r.pass_all for r in results) / len(results),
            pass_at_k=pass_at_k,
            per_scorer_avg={name: np.mean([r.scores[name] for r in results]) for name in self.SCORERS},
            failures=[r for r in results if not r.pass_all],
        )
```

## 8.2 Shadow/Canary Pipeline (from caas-casestudy)

```python
# packages/platform-common/src/platform_common/lifecycle/canary.py

class CanaryPipeline:
    """7-stage canary deployment with auto-graduate and auto-rollback."""

    STAGES = [
        CanaryStage(traffic_pct=1,  duration_hours=4,  min_requests=100),
        CanaryStage(traffic_pct=5,  duration_hours=8,  min_requests=500),
        CanaryStage(traffic_pct=10, duration_hours=12, min_requests=1000),
        CanaryStage(traffic_pct=25, duration_hours=24, min_requests=2500),
        CanaryStage(traffic_pct=50, duration_hours=24, min_requests=5000),
        CanaryStage(traffic_pct=75, duration_hours=24, min_requests=7500),
        CanaryStage(traffic_pct=100, duration_hours=48, min_requests=10000),
    ]

    ROLLBACK_CRITERIA = {
        "error_rate_threshold": 0.05,        # >5% errors -> rollback
        "latency_p95_threshold_ms": 2000,    # >2s p95 -> rollback
        "task_completion_threshold": 0.85,   # <85% completion -> rollback
        "policy_violation_threshold": 0.01,  # >1% policy violations -> rollback
    }

    async def promote(self, agent_id: str, from_stage: str, to_stage: str):
        """Advance agent through canary stages."""
        for stage in self.STAGES:
            # Set traffic split
            await self.router.set_canary_traffic(agent_id, stage.traffic_pct)

            # Wait for minimum duration and requests
            await self._wait_for_stage(stage)

            # Check metrics
            metrics = await self.metrics.get_agent_metrics(agent_id, stage.duration_hours)

            if self._should_rollback(metrics):
                await self._rollback(agent_id)
                return CanaryResult(status="ROLLED_BACK", stage=stage, metrics=metrics)

        return CanaryResult(status="PROMOTED", stage=self.STAGES[-1])
```

## 8.3 Trust Engine (from lia-openclaw)

```python
# packages/platform-common/src/platform_common/lifecycle/trust.py

class ProgressiveTrustEngine:
    """4-level autonomy with time-based decay."""

    LEVELS = {
        "L0": {"name": "Fully Supervised", "requires_human": True, "can_execute": False},
        "L1": {"name": "Human-in-Loop", "requires_human": True, "can_execute": True},
        "L2": {"name": "Supervised Autonomy", "requires_human": False, "can_execute": True, "async_review": True},
        "L3": {"name": "Full Autonomy", "requires_human": False, "can_execute": True, "async_review": False},
        "L4": {"name": "Self-Improving", "requires_human": False, "can_execute": True, "can_tune": True},
    }

    async def check_autonomy(self, agent_id: str, action: str, lender_id: UUID) -> TrustDecision:
        current_level = await self.repo.get_trust_level(agent_id, lender_id)
        level_config = self.LEVELS[current_level]

        # Check for trust decay
        metrics = await self.metrics.get_recent(agent_id, days=7)
        if metrics.accuracy < self.decay_threshold:
            new_level = self._decrease_level(current_level)
            await self.repo.set_trust_level(agent_id, lender_id, new_level)
            await self._alert_cro(agent_id, lender_id, current_level, new_level, metrics)
            level_config = self.LEVELS[new_level]

        return TrustDecision(
            allowed=level_config["can_execute"],
            requires_human=level_config["requires_human"],
            requires_async_review=level_config.get("async_review", False),
            current_level=current_level,
        )
```

---

# MISSION 9: INTEGRATION TESTING + LOAD TESTING + SECURITY

**Objective:** Comprehensive test coverage ensuring production readiness.

**Dependencies:** Missions 0-8

**Duration:** Weeks 22-30 (continuous, intensifies before pilot)

## 9.1 End-to-End Test Scenarios

```python
# tests/e2e/test_full_loan_lifecycle.py

class TestDairyFarmerJourney:
    """Complete loan lifecycle: voice call -> disbursement -> monitoring -> graduation."""

    async def test_happy_path(self):
        # 1. Borrower calls toll-free number
        call = await self.voice_simulator.initiate_call(
            language="gu",
            audio_fixtures=["dairy_farmer_greeting.wav"],
        )

        # 2. Agent detects intent: NEW_LOAN
        assert call.detected_intent == "NEW_LOAN"
        assert call.detected_segment == "DAIRY_FARMER"

        # 3. Aadhaar OTP flow
        otp_response = await self.voice_simulator.send_audio(
            call.id, "aadhaar_number_spoken.wav"
        )
        assert "OTP" in otp_response.transcript

        # 4. Data fetch completes
        await self.wait_for_status(call.application_id, "DATA_FETCH", timeout=15)

        # 5. Decision: APPROVED
        await self.wait_for_status(call.application_id, "APPROVED", timeout=5)
        decision = await self.api.get_decision(call.application_id)
        assert decision.raw_score >= 600
        assert decision.decision == "APPROVE"

        # 6. Offer presented via voice
        offer_audio = await self.voice_simulator.get_next_response(call.id)
        assert "25,000" in offer_audio.transcript or "રૂપિયા" in offer_audio.transcript

        # 7. Borrower accepts
        await self.voice_simulator.send_audio(call.id, "acceptance_yes.wav")
        await self.wait_for_status(call.application_id, "DISBURSING", timeout=60)

        # 8. Disbursement confirmed
        await self.wait_for_status(call.application_id, "ACTIVE", timeout=120)
        loan = await self.api.get_loan(call.application_id)
        assert loan.disbursed_amount > 0

        # 9. Verify audit trail completeness
        audit = await self.api.get_audit_trail(call.application_id)
        assert len(audit.events) >= 15

        # 10. Verify graph memory
        context = await self.graph.get_borrower_context(loan.borrower_id)
        assert len(context["loans"]) == 1

    async def test_weather_moratorium(self):
        """Active loan + weather event -> auto-moratorium."""
        loan = await self.create_active_loan(segment="DAIRY_FARMER")

        # Inject heat wave event
        await self.imd_mock.set_temperature(loan.pincode, 42)

        # Wait for monitoring agent to detect
        await asyncio.sleep(5)

        # Verify moratorium triggered
        loan = await self.api.get_loan(loan.id)
        assert loan.sub_status == "MORATORIUM"

        # Verify schedule recalculated
        schedule = await self.api.get_schedule(loan.id)
        paused = [s for s in schedule if s.status == "MORATORIUM"]
        assert len(paused) >= 1

        # Verify borrower notified
        notifications = await self.notification_mock.get_sent(loan.borrower_id)
        assert any("moratorium" in n.template for n in notifications)
```

## 9.2 Load Test Scenarios (Locust)

```python
# tests/load/locustfile.py

from locust import HttpUser, task, between

class LoanOriginationUser(HttpUser):
    wait_time = between(1, 5)

    @task(3)
    def submit_application(self):
        """Simulate full application submission."""
        self.client.post("/v1/data/fetch", json={
            "borrower_id": str(uuid4()),
            "aadhaar_token": "test_encrypted_token",
            "segment_hint": random.choice(["DAIRY_FARMER", "GIG_WORKER", "KIRANA_ONDC"]),
            "lender_id": self.lender_id,
            "request_id": str(uuid4()),
        })

    @task(5)
    def trigger_decision(self):
        """Simulate credit decision."""
        self.client.post("/v1/decision/evaluate", json={
            "borrower_id": str(uuid4()),
            "segment": "DAIRY_FARMER",
            "lender_id": self.lender_id,
            "data_confidence": 0.85,
        })

    @task(1)
    def check_portfolio(self):
        """Simulate dashboard portfolio query."""
        self.client.get(f"/api/v1/portfolio/summary?lender_id={self.lender_id}")

# Target load per gate:
# Gate 1: 100 concurrent (data fetch)
# Gate 2: 500 concurrent (decisioning)
# Gate 3: 200 concurrent (LOS workflow)
# Gate 4: 50 concurrent (voice calls)
# Gate 5: 10,000 active loans monitored
# Gate 6: 1,000 concurrent applications, p95 < 500ms
```

## 9.3 Security Test Checklist

```
OWASP TOP 10 COVERAGE:
[ ] Injection: SQL injection tested on all API parameters
[ ] Broken auth: JWT expiry, invalid tokens, role escalation
[ ] Sensitive data: PII encrypted at rest, never in logs, never in URLs
[ ] XXE: XML parsing disabled or hardened
[ ] Broken access: RBAC enforced (cross-lender access impossible)
[ ] Security misconfig: default passwords removed, headers set
[ ] XSS: dashboard input sanitization
[ ] Deserialization: JSON only, no pickle/YAML unsafe load
[ ] Components: all dependencies scanned (Trivy, Safety)
[ ] Logging: sufficient for forensics, PII redacted

LENDING-SPECIFIC:
[ ] Cross-lender data isolation: RLS verified at DB level
[ ] Aadhaar data: encrypted, consent-gated, auto-purge on revocation
[ ] Consent revocation: all data deleted within 24h (DPDP Act)
[ ] Audit trail: tamper-evident (hash chain verified)
[ ] API keys: bcrypt hashed, never logged, rotation supported
[ ] Voice recordings: encrypted in S3, access-controlled
[ ] Disbursement: idempotent (no double-disburse even with retry)
```

---

# MISSION 10: PRODUCTION DEPLOYMENT + FIRST LENDER ONBOARDING

**Objective:** Deploy to production, onboard first lender, execute 90-day pilot.

**Dependencies:** All missions (0-9)

**Duration:** Weeks 28-40

## 10.1 Production Deployment Runbook

```
STEP 1: INFRASTRUCTURE (Day 1-3)
  terraform apply -var-file=production/terraform.tfvars
  Verify: EKS cluster, RDS, Redis, MSK, Neo4j, Temporal, S3, Vault
  Run: infrastructure health check suite

STEP 2: DATABASE (Day 3-4)
  Run all Alembic migrations against production PostgreSQL
  Load: governance graph (RBI rules) into Neo4j
  Load: 38 segment configs into segment_configs table
  Verify: RLS policies active, PII encryption working

STEP 3: SERVICES (Day 4-7)
  Deploy in dependency order:
    1. platform-common (shared library)
    2. auth-svc, audit-trail, api-gateway
    3. uli-orchestrator, data-quality-engine, identity-svc, bankconnect-svc
    4. sentinel-core, product-engine, fraud-engine
    5. prism-matcher, los-engine, journey-studio, notification-svc
    6. agent-runtime, voice-pipeline, constellation, knowledge-svc
    7. context-graph, monitoring-agent
    8. mcp-server (unified)
    9. dashboard
  Each service: helm upgrade --install, verify health check, run smoke test

STEP 4: LENDER ONBOARDING (Day 7-14)
  1. Create lender record in lenders table
  2. Create users with RBAC roles (CRO, risk_analyst, UW, ops, compliance)
  3. Configure Prism profile (segments, geography, ticket range, PSL targets)
  4. Load lender-specific Sentinel policies (start in SHADOW)
  5. Configure disbursement API integration (sandbox first)
  6. Configure voice: toll-free number, language, greeting template
  7. Run 2-week shadow test with anonymized historical data

STEP 5: PILOT LAUNCH (Day 14+)
  Follow First 90 Days playbook from unified-platform-specification.md
  Week 1-2:  50-100 borrowers, all REFER
  Week 3-4:  500-1000 borrowers, auto-approve score > 700
  Week 5-8:  Full geography, autonomy L2
  Week 9-12: Steady state, autonomy L3, first graduations
```

## 10.2 Monitoring Dashboard Post-Launch

```
REAL-TIME METRICS (Grafana dashboards):

Platform Health:
  - Service uptime (target: 99.9%)
  - API latency p50/p95/p99 per service
  - Error rate per service
  - Kafka consumer lag per topic
  - PostgreSQL connection pool utilization
  - Redis hit rate

Lending Metrics:
  - Applications per hour (by segment, channel)
  - Decision latency p95 (target: <500ms)
  - Approval rate (by segment, policy)
  - Data fetch success rate per ULI source
  - Disbursement success rate
  - Average time: application to disbursement

Agent Metrics:
  - Voice agent: calls per hour, avg call duration, completion rate
  - ASR accuracy (real-time WER estimate)
  - Intent classification confidence distribution
  - Human escalation rate (target: <15%)
  - Returning borrower recognition rate

Portfolio Metrics:
  - AUM growth
  - DPD distribution (0, 1-30, 31-60, 61-90, 90+)
  - NPA rate (target: <3%)
  - Monitoring triggers fired per day
  - Intervention success rate
  - Graduation pipeline (eligible borrowers)

ALERTS (PagerDuty):
  - P1: Any service down > 5 min
  - P1: Disbursement failure
  - P1: Data breach indicator (unusual query pattern)
  - P2: Decision latency > 2s p95
  - P2: ASR accuracy drop > 10%
  - P2: Kafka consumer lag > 1000 messages
  - P3: NPA rate exceeding threshold
  - P3: Model drift detected (PSI > 0.2)
```

---

# DEPENDENCY GRAPH

```
Mission 0 (Infra) ──────────┐
                             v
Mission 1 (Shared Libs) ────┐
                             v
Mission 2 (Data Layer) ─────┐
                             v
Mission 3 (Decisioning) ────┤
                             v
Mission 4 (Distribution) ───┤──> Mission 7 (Dashboard) ──┐
                             v                             │
Mission 5 (Agents) ─────────┤                             │
                             v                             │
Mission 6 (Graph+Monitor) ──┤                             │
                             v                             v
Mission 8 (Lifecycle) ──────┴──> Mission 9 (Testing) ──> Mission 10 (Deploy)
```

**Critical path:** M0 -> M1 -> M2 -> M3 -> M4 -> M5 -> M9 -> M10

**Parallelizable:** M7 (Dashboard) can start with M4 and run through M6. M8 (Lifecycle) can start with M5.

---

# EFFORT ESTIMATION

| Mission | Duration | Team | Parallel With |
|---------|----------|------|---------------|
| M0: Infrastructure | 4 weeks | 1 DevOps + 1 Backend | -- |
| M1: Shared Libraries | 3 weeks | 2 Backend | M0 (overlap weeks 2-4) |
| M2: Data Layer | 8 weeks | 3 Backend + 1 ML | -- |
| M3: Decisioning | 8 weeks | 2 Backend + 1 ML | M2 (overlap weeks 9-12) |
| M4: Distribution | 8 weeks | 2 Backend | -- |
| M5: Agents | 8 weeks | 2 Backend + 1 ML + 1 Voice | M4 (overlap weeks 17-20) |
| M6: Graph + Monitoring | 6 weeks | 2 Backend | M5 (overlap weeks 21-24) |
| M7: Dashboard | 8 weeks | 2 Frontend | M4-M6 (parallel) |
| M8: Agent Lifecycle | 4 weeks | 1 Backend + 1 ML | M5-M6 (parallel) |
| M9: Testing | 4 weeks | 2 QA + 1 Security | M8 (after) |
| M10: Production | 4 weeks | Full team | -- |

**Total timeline:** ~40 weeks (10 months) with a team of 8-10 engineers.
**First pilot:** Week 24-26 (controlled, 500-1000 loans).
**Production:** Week 30+ (first lender fully live).
