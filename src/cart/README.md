# Retail Store Sample App — DevOps & Infrastructure Architecture

This repository contains a microservices reference e-commerce application structured specifically for modern cloud-native deployment patterns, CI/CD automation, and multi-tier container runtime orchestration.

## DevOps & Infrastructure Overview

```
                                      Git Commit / PR
                                            │
                                            ▼
                      ┌───────────────────────────────────────────┐
                      │    Azure DevOps Pipelines (.azure-pipelines)│
                      └─────────────────────┬─────────────────────┘
                                            │
                        ┌───────────────────┼───────────────────┐
                        ▼                   ▼                   ▼
                   Lint & Test         Multi-Stage        Security & Compliance
                   - Go test           Docker Build       - Gitleaks / Secret Scan
                   - Maven/JUnit       (Multi-Arch)       - Open Source Tools (ORT)
                   - Yarn / Jest                          - Code Scan Reporting
                                            │
                                            ▼
                                  Container Registry
                                            │
                 ┌──────────────────────────┴──────────────────────────┐
                 ▼                                                     ▼
        Local Container Runtime                                Kubernetes Cluster
     (Docker Compose / Tilt Dev)                             (Helm / Helmfile Deploy)
```

From an infrastructure and DevOps perspective, the repository functions as a modular monorepo:

- **Decoupled Workflows**: Each microservice maintains its own build runtime, dependency definition, container lifecycle, and localized Helm/Compose manifests.
- **Declarative Orchestration**: The top-level `src/app/` serves as the platform layer, consolidating individual service charts and containers into integrated environments via Helmfile, Docker Compose overlays, and Tilt.
- **Ephemeral Testing Capabilities**: Persistence adapters allow services to pivot dynamically between lightweight, in-memory local state (or containerized mocks like DynamoDB Local / MySQL containers) and managed cloud resources (RDS, OpenSearch, DynamoDB) using environment variables.

## Docker & Containerization Patterns

Each service under `src/` packages an optimized Dockerfile tailored to its language ecosystem:

### Service Container Matrix

| Service | Path | Build Strategy | Base Image Strategy | Key Exposing Ports |
|---|---|---|---|---|
| Catalog | `src/catalog/` | Multi-stage Go build | `golang:alpine` builder → minimal distroless/scratch runtime | 8080 (Internal) / 8081 (Host) |
| Cart | `src/cart/` | Multi-stage Maven build | Eclipse Temurin / Corretto OpenJDK runtime | 8080 (Internal) / 8082 (Host) |
| Orders | `src/orders/` | Multi-stage Maven/Gradle build | Minimal JRE base image | 8080 (Internal) / 8083 (Host) |
| Checkout | `src/checkout/` | Multi-stage Node.js build | Node Alpine, dependencies pruned with production flags | 8080 (Internal) / 8084 (Host) |
| UI | `src/ui/` | Multi-stage Vue build | Node build phase → NGINX unprivileged base server | 8080 (Internal) / 8080 (Host) |

### Docker Compose Architecture

The repository provides several compose topology layers:

- **Root Compose** (`docker-compose.yml`): Primary bootstrapping topology that links all microservices and spun-up data containers (MySQL, PostgreSQL, RabbitMQ, DynamoDB Local, Redis).
- **Application Compose** (`src/app/docker-compose.yml`): Explicit platform configuration managing service discovery and environment parameter injection.
- **Observability Overlay** (`src/app/docker-compose.tracing.yml`): Layered onto base compose configurations to inject the OpenTelemetry (OTel) Collector and routing parameters for distributed trace collection.
- **Compose Override** (`src/app/compose.override.yaml`): Overrides networking, local mount points, and live-reload volumes for active development.

## CI/CD Pipeline Architecture (`.azure-pipelines/`)

The repository uses Azure DevOps Pipelines for modular testing, vulnerability auditing, image construction, and package delivery.

### Pipeline Structure

```
.azure-pipelines/
├── cart-ci.yml                 # Java / Spring Boot CI Workflow
├── catalog-ci.yml              # Go CI Workflow
├── checkout-ci.yml             # TypeScript / Node.js CI Workflow
├── orders-ci.yml               # Java / Spring Boot CI Workflow
├── ui-ci.yml                   # Vue.js Frontend CI Workflow
└── templates/
    └── deploy-step.yml         # Shared continuous deployment step template
```

### Pipeline Breakdown

#### `catalog-ci.yml` (Go)

- **Trigger**: Path-filtered to changes in `src/catalog/**` and pipeline configs.
- **Execution**:
  - Sets up Go toolchain.
  - Runs dependency download and security auditing via `go.mod`.
  - Executes automated tests with code coverage outputs (`go test -v -coverprofile=...`).
  - Builds multi-stage production Docker container.
  - Publishes test artifacts and calls `deploy-step.yml` for staging/registry push.

#### `cart-ci.yml` & `orders-ci.yml` (Java / Spring Boot)

- **Trigger**: Path-filtered to changes in `src/cart/**` and `src/orders/**` respectively.
- **Execution**:
  - Configures JDK 17 environment.
  - Executes `./mvnw clean test verify` covering unit tests and slice tests.
  - Runs containerized ephemeral tests against local mock services.
  - Packages Spring Boot executable JAR and triggers Docker image build via Dockerfile.
  - Publishes test results (JUnit format) and code coverage.

#### `checkout-ci.yml` & `ui-ci.yml` (Node.js / Vue.js)

- **Trigger**: Path-filtered to changes in `src/checkout/**` and `src/ui/**` respectively.
- **Execution**:
  - Bootstraps Node environment and Yarn package manager.
  - Runs `yarn install --immutable` for reproducible dependency graphs.
  - Lints codebase and runs Jest/Mocha suites.
  - Bundles static dist assets (`yarn build`).
  - Builds runtime container (distroless or NGINX).

#### `templates/deploy-step.yml` (Shared Pipeline Template)

Centralized step template that abstracts:

- Container image tagging (Git SHA, SemVer, branch name).
- Container registry authentication.
- Image push and Helm chart artifact archiving.



### Bring up Full Stack via Docker Compose

```bash
# Standard local setup with databases
docker compose up -d

# Overlay with OpenTelemetry distributed tracing
docker compose -f src/app/docker-compose.yml -f src/app/docker-compose.tracing.yml up -d
```
