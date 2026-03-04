# Logiforma

**Multi-region cloud infrastructure on AWS, managed as code with OpenTofu.**

Logiforma is a modular infrastructure platform built on AWS, orchestrated through multiple purpose-specific repositories. Each repository owns a single layer of the stack — from networking foundations to databases and observability — enabling independent deployments with clear dependency boundaries.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Topology                                      │
│              Single source of truth for all configuration               │
│         regions · databases · observability · domains · dashboards       │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
              ┌──────────────▼──────────────┐
              │          Networking          │
              │   VPCs · Subnets · Endpoints │
              │   API Gateway · Cloudflare   │
              └──────┬───────────────┬──────┘
                     │               │
          ┌──────────▼───┐   ┌───────▼──────────┐
          │   Databases  │   │  Observability    │
          │ Aurora PostgreSQL│   │  LGTM + Alloy   │
          │ Secrets Manager │   │  Grafana Proxy   │
          └──────────────┘   │  S3 Dashboards    │
                             └──────────────────┘
```

Repositories are deployed in order: **Topology → Networking → Databases → Observability**. Each layer reads upstream configuration via OpenTofu remote state.

## Repositories

| Repository | Description |
|------------|-------------|
| **[Topology](https://github.com/Logiforma/Topology)** | Central configuration — regions, database sizing, observability toggles, domain mappings, and dashboard settings. No resources are created; it only publishes outputs for downstream repos. |
| **[Networking](https://github.com/Logiforma/Networking)** | VPCs with public/private subnets, VPC gateway endpoints (S3, DynamoDB), per-region Platform API Gateways with ACM certificates and Cloudflare DNS. |
| **[Databases](https://github.com/Logiforma/Databases)** | Aurora PostgreSQL clusters — serverless v2 for DEV, provisioned instances for PROD. Encrypted storage, managed passwords via Secrets Manager, automated backups. |
| **[Observability](https://github.com/Logiforma/Observability)** | Full LGTM stack (Loki, Grafana, Tempo, Mimir) plus Grafana Alloy as a unified telemetry collector. Per-region Lightsail instances, S3 storage backends, API Gateway proxy for Grafana, and Cloudflare DNS. |

## Technology Stack

| Layer | Technology |
|-------|------------|
| **IaC** | OpenTofu >= 1.9.0, S3 + DynamoDB backend |
| **Compute** | AWS Lightsail (LGTM stack), Lambda (application) |
| **Database** | Aurora PostgreSQL 16.4 (serverless v2 / provisioned) |
| **Networking** | VPC, API Gateway v2 (HTTP), VPC Endpoints |
| **Observability** | Grafana, Loki, Tempo, Mimir, Alloy |
| **DNS** | Cloudflare (automated via OpenTofu) |
| **CI/CD** | GitHub Actions — branch-based deployment |
| **Secrets** | AWS Secrets Manager, GitHub Actions Secrets |

## Regional Architecture

Infrastructure is deployed per-region with friendly subdomain prefixes:

| Region | Prefix | API Endpoint | Grafana |
|--------|--------|-------------|---------|
| ap-south-1 (Mumbai) | `mumbai` | `mumbai.{domain}` | `mumbai.grafana.{domain}` |
| eu-west-1 (Ireland) | `ireland` | `ireland.{domain}` | `ireland.grafana.{domain}` |

Each region gets its own VPC, Aurora cluster, LGTM+Alloy stack, S3 buckets, and API Gateway — fully isolated.

### Telemetry with Grafana Alloy

Grafana Alloy serves as the single external entry point for all telemetry data per region:

```
Applications ──OTLP──▶ Alloy (4317/4318) ──▶ Tempo (traces)
                                            ──▶ Mimir (metrics)
             ──Loki──▶ Alloy (3100)        ──▶ Loki  (logs)
Docker logs ────────▶ Alloy (discovery)     ──▶ Loki  (logs)
```

Backend services (Loki, Tempo, Mimir) are internal-only — Alloy is the only publicly exposed collector.

## Environments

| Environment | Branch | Domain | Regions | Purpose |
|-------------|--------|--------|---------|---------|
| **DEV** | `dev` | `logiforma.dev` | 1 (Mumbai) | Development, cost-optimised |
| **PROD** | `main` | `logiforma.com` | 2 (Mumbai + Ireland) | Production, high availability |

Pushing to `dev` auto-deploys to DEV. Merging to `main` deploys to PROD.

## Getting Started

Each repository can be deployed locally with OpenTofu:

```bash
# 1. Deploy Topology first
cd Topology
tofu init -backend-config="key=topology/dev/terraform.tfstate"
tofu apply -var-file="environments/dev.tfvars"

# 2. Then Networking
cd ../Networking
tofu init -backend-config="key=networking/dev/terraform.tfstate"
tofu apply -var-file="environments/dev.tfvars" -var="cloudflare_api_token=$CF_TOKEN"

# 3. Then Databases
cd ../Databases
tofu init -backend-config="key=databases/dev/terraform.tfstate"
tofu apply -var-file="environments/dev.tfvars"

# 4. Finally Observability
cd ../Observability
tofu init -backend-config="key=observability/dev/terraform.tfstate"
tofu apply -var-file="environments/dev.tfvars" \
  -var="grafana_admin_password=$GRAFANA_PW" \
  -var="cloudflare_api_token=$CF_TOKEN"
```

See individual repositories for detailed documentation.
