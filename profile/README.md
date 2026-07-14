# Logiforma

**A multi-tenant SaaS platform on AWS, managed entirely as code.**

Logiforma is built as two complementary layers, each a set of purpose-specific
repositories with clear dependency boundaries and independent deployments:

1. **Core cloud platform** — regional foundations: networking, databases, observability (OpenTofu).
2. **Application & tenant platform** — the serverless services that onboard tenants and run the products (OpenTofu + Terragrunt, all serverless).

---

## Repositories

### Core cloud platform

| Repository | Description |
|------------|-------------|
| **[Topology](https://github.com/Logiforma/Topology)** | Central configuration — regions, database sizing, observability toggles, domain mappings, and dashboard settings. No resources are created; it only publishes outputs for downstream repos. |
| **[Networking](https://github.com/Logiforma/Networking)** | VPCs with public/private subnets, VPC gateway endpoints (S3, DynamoDB), per-region Platform API Gateways with ACM certificates and Cloudflare DNS. |
| **[Databases](https://github.com/Logiforma/Databases)** | Aurora PostgreSQL clusters — serverless v2 for DEV, provisioned instances for PROD. Encrypted storage, managed passwords via Secrets Manager, automated backups. |
| **[Observability](https://github.com/Logiforma/Observability)** | Full LGTM stack (Loki, Grafana, Tempo, Mimir) plus Grafana Alloy as a unified telemetry collector. Per-region Lightsail instances, S3 storage backends, API Gateway proxy for Grafana, and Cloudflare DNS. |

### Application & tenant platform

| Repository | Description |
|------------|-------------|
| **[Atlas](https://github.com/Logiforma/atlas)** | Central infrastructure-as-code (OpenTofu + Terragrunt) for the serverless application platform — DynamoDB, Cognito (one user pool per tenant), S3/CloudFront, API Gateway, Lambda runtime platforms, SES, SQS, and IAM. Config-driven per environment. |
| **[Bootstrap](https://github.com/Logiforma/bootstrap)** | Run-once stack that creates the remote OpenTofu state backend (encrypted, versioned S3 + lock table) per environment. |
| **[sentinel-middleware](https://github.com/Logiforma/sentinel-middleware)** | Sentinel admin API — tenant onboarding, per-tenant Cognito pools, user management, notification templates. FastAPI + Mangum on Lambda. |
| **[sentinel-ui](https://github.com/Logiforma/sentinel-ui)** | Sentinel admin console for staff — tenants, users, notifications, system health. Vue 3 SPA on S3 + CloudFront. |
| **[notification-engine](https://github.com/Logiforma/notification-engine)** | Async, SQS-driven Lambda worker that renders templates and sends transactional email via Amazon SES. |

---

## Architecture

### Core cloud platform

```
┌──────────────────────────────────────────────────────────────────┐
│                            Topology                                │
│         Single source of truth for all configuration               │
│      regions · databases · observability · domains · dashboards    │
└────────────────────────────┬───────────────────────────────────────┘
                             │
              ┌──────────────▼──────────────┐
              │          Networking          │
              │   VPCs · Subnets · Endpoints │
              │   API Gateway · Cloudflare   │
              └──────┬───────────────┬───────┘
                     │               │
          ┌──────────▼─────┐   ┌──────▼───────────┐
          │   Databases    │   │  Observability   │
          │ Aurora Postgres│   │  LGTM + Alloy    │
          │ Secrets Manager│   │  Grafana Proxy   │
          └────────────────┘   └──────────────────┘
```

Deployed in order **Topology → Networking → Databases → Observability**; each layer
reads upstream configuration via OpenTofu remote state.

### Application & tenant platform (serverless)

```
Staff ─▶ Sentinel UI ─▶ Sentinel API ─┬─▶ DynamoDB (tenants · users · templates · logs)
 (Entra SSO / Cognito)  (Lambda + API GW)│
                                         ├─▶ Cognito (one user pool per tenant)
                                         └─▶ SQS ─▶ Notification Engine ─▶ SES ✉
```

**Atlas** provisions the shared resources and publishes each app's configuration to
SSM Parameter Store; the application repos read that contract and **deploy themselves**
via GitHub Actions (OIDC — no long-lived AWS keys), independently of Atlas. Everything
here is serverless and free-tier-first.

---

## Technology stack

| Layer | Technology |
|-------|------------|
| **IaC** | OpenTofu (≥ 1.9), S3 remote state; Terragrunt for the application platform |
| **Compute** | Lightsail (LGTM stack), AWS Lambda (applications) |
| **Data** | Aurora PostgreSQL 16.4 (core platform) · DynamoDB (application platform) |
| **Identity** | Amazon Cognito + Microsoft Entra SSO (per-tenant pools) |
| **Networking / edge** | VPC, API Gateway v2 (HTTP), CloudFront, Cloudflare DNS |
| **Messaging** | Amazon SQS + SES (transactional email) |
| **Observability** | Grafana, Loki, Tempo, Mimir, Alloy |
| **CI/CD** | GitHub Actions — branch-based deployment, OIDC federation |
| **Secrets / config** | AWS Secrets Manager, SSM Parameter Store, GitHub Environments |

---

## Environments

| Environment | Branch | Domain | Regions | Purpose |
|-------------|--------|--------|---------|---------|
| **DEV** | `dev` | `logiforma.dev` | Mumbai | Development, cost-optimised |
| **PROD** | `main` | `logiforma.com` | Mumbai + Ireland | Production, high availability |

Pushing to `dev` auto-deploys to DEV; merging to `main` deploys to PROD. See each
repository's README for details.
