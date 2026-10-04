---
id: CIC-001-001
title: Repository Constitution and Engineering Standards
version: 1.0.0
status: normative
authority: Chief Platform Architect
corpus: CORPUS-001
dependencies: [CIC-000-001, CIC-000-002, CIC-000-003]
traces: [PRIN-003, PRIN-009, PRIN-016, PRIN-022]
certification: [CR-001-001-01, CR-001-001-02, CR-001-001-03, CR-001-001-04, CR-001-001-05, CR-001-001-06]
validation: [VR-001-001-01, VR-001-001-02]
---

# CIC-001-001 — Repository Constitution and Engineering Standards

## 1. Reference technology baseline

Specifications are language-neutral. The **reference implementation** (CORPUS-030) uses:

| Concern | Reference choice | Port / open standard (PRIN-022) |
|---------|------------------|----------------------------------|
| Service language | Kotlin 2.x on JVM 21 | Any language passing contract + certification suites |
| UI / SDK (web) | TypeScript 5.x | — |
| Build | Gradle 8 (Kotlin DSL), pnpm for TS | — |
| Relational store | PostgreSQL 16 | SQL:2016 + JSONB-equivalent |
| Graph projection | Any openCypher / ISO GQL store | openCypher, GQL |
| Search | OpenSearch 2.x | `SearchIndexPort` |
| Event log | Apache Kafka 3.x protocol | CloudEvents 1.0 binary/structured mode |
| Cache | Redis protocol 7 | `CachePort` |
| Policy engine | Embedded CEL evaluator + OPA-compatible bundle adapter | CEL spec, Rego |
| Identity | OIDC 1.0 / OAuth 2.1 provider | OIDC |
| Telemetry | OpenTelemetry SDK | OTLP |
| Packaging | OCI images, Helm charts | OCI, Kubernetes API |

## 2. Repository topology

A single **monorepo** `ucos` holds kernel, domain services, SDKs, and the corpus. Type Packs and
plugins live in separate repositories that depend only on published SDK artifacts.

```
ucos/
├── corpus/                         # This CIC (copied from spec repo, versioned with code)
├── platform/                       # Cross-cutting runtime libraries (no domain logic)
│   ├── ucos-kernel-api/            # Kernel ports & value objects: Urn, Money, LocalizedText, ContextVector …
│   ├── ucos-eventing/              # CloudEvents envelope, outbox, consumer framework
│   ├── ucos-persistence/           # Event store, repository base, migrations runner
│   ├── ucos-policy-runtime/        # PEP client, CEL evaluator, decision cache
│   ├── ucos-context-runtime/       # Context resolution engine (CIC-011-001)
│   ├── ucos-security/              # AuthN filter, PDP client, crypto-shred, field-level encryption
│   ├── ucos-observability/         # OTel wiring, standard metrics, audit emitter
│   └── ucos-testkit/               # Contract-test harness, fixtures, certification annotations
├── kernel/                         # Kernel bounded contexts (tier: kernel)
│   ├── identity/  party/  ontology/  commercial-object/  graph/  context/  policy/  lifecycle/  evidence/
├── domains/                        # Core & extended bounded contexts
│   ├── catalog/  pricing/  offer/  entitlement/  contract/  order/  billing/  …
├── adapters/                       # Port adapters, one module per (port, technology)
│   ├── persistence-postgres/  graph-neo4j/  graph-age/  search-opensearch/  bus-kafka/  …
├── sdk/                            # Published client SDKs (CORPUS-027)
│   ├── kotlin/  typescript/  python/  go/
├── apps/                           # Deployable composition roots (CORPUS-020)
├── ui/                             # Metadata-driven UI shell + renderers (CORPUS-012)
├── packs/std/                      # ucos.std Type Pack (standard commercial types)
├── deploy/                         # Helm charts, environment overlays (CORPUS-024)
└── tools/                          # Linters (UCOS-Lxxx), generators, validators
```

## 3. Module structure (every bounded context module)

```
domains/<context>/
├── <context>-api/            # Public contract: OpenAPI, proto, AsyncAPI, published event schemas, DTOs
├── <context>-domain/         # Aggregates, entities, value objects, domain services, domain events. Pure: no I/O imports.
├── <context>-application/    # Command handlers, query handlers, process managers, ports (interfaces)
├── <context>-infrastructure/ # Port implementations bound to platform adapters; persistence mappings; migrations
├── <context>-service/        # Composition root: HTTP/gRPC endpoints, consumers, config
└── <context>-tests/          # Contract tests, certification tests, scenario tests
```

Dependency rule (enforced by ArchUnit / dependency-cruiser, CR-001-001-01):
`api ← domain ← application ← infrastructure ← service`; `domain` depends only on `ucos-kernel-api`.
A module MUST NOT depend on another context's `domain`, `application`, or `infrastructure`; it MAY
depend on another context's `-api` module only (published language).

## 4. Package structure (JVM reference; other languages mirror segments)

`org.ucos.<context>.<layer>.<subpackage>`, layer ∈ {`api`, `domain`, `application`, `infrastructure`, `service`}.
Inside `domain`: `model` (aggregates/entities/VOs), `event`, `service`, `policy` (policy-input builders), `error`.
Inside `application`: `command`, `query`, `port`, `process`.

## 5. Coding standards

1. Aggregates are immutable value-in/value-out: `fun handle(cmd): Result<List<DomainEvent>>` and
   `fun apply(event): Aggregate`. No setters.
2. No nulls across module boundaries; use `Option`/nullable-with-contract only inside a function.
3. Money is `Money(amountMinor: BigInteger, currency: CurrencyCode, scale: Int)`; floating point for
   money, quantities, or rates is forbidden (lint `UCOS-L002`).
4. Time is `Instant` (UTC) plus explicit `ZoneId` when calendrical; `LocalDateTime` forbidden in
   persisted or published models (`UCOS-L003`).
5. Every externally variable behavior is a policy call (`PolicyClient.decide`) — PRIN-005.
6. No literals for industry/market/channel/category/currency/type names (`UCOS-L001`).
7. Errors are typed sealed hierarchies mapped to RFC 9457 problem types (CIC-005-001 §7).
8. Max cyclomatic complexity 15 per function; max 400 lines per file (excluding generated).
9. All public functions in `-api` and `-domain` carry KDoc/TSDoc including `@traces` IDs.

## 6. Build manifest (`ucos-module.yaml`, mandatory per deployable module)

```yaml
module: pricing-service
context: pricing
domains: [DOM-PRC]
traces: [CIC-011-004, PRIN-005, PRIN-006]
ports:  {consumes: [EventBusPort, CachePort, PolicyPort], provides: [PricingQueryApi]}
api:    {openapi: pricing-api/openapi.yaml, asyncapi: pricing-api/asyncapi.yaml}
certificationLevel: L2
```

## 7. Naming standards

| Element | Convention | Example |
|---------|-----------|---------|
| Bounded context | lower-kebab singular noun | `commercial-object` |
| Aggregate class | PascalCase noun | `PriceList` |
| Command | Imperative verb + noun | `PublishOffer` |
| Event | Noun + past participle | `OfferPublished` → type `ucos.offer.offer.published.v1` |
| Query | `Get`/`Find`/`Resolve` + noun | `ResolvePrice` |
| REST resource | plural lower-kebab | `/v1/price-lists/{id}` |
| JSON fields | lowerCamelCase | `effectiveFrom` |
| DB schema | `ucos_<context>` snake_case | `ucos_pricing` |
| DB table / column | snake_case plural / singular | `price_entries.amount_minor` |
| Metric | `ucos_<context>_<noun>_<unit>` | `ucos_pricing_resolution_duration_seconds` |
| Policy | `urn:ucos:<ns>:policy:<context>.<name>` | `urn:ucos:ucos.std:policy:pricing.rounding` |
| Env var | `UCOS_<CONTEXT>_<KEY>` | `UCOS_PRICING_CACHE_TTL` |

## 8. Documentation standards

- Every module has `README.md` with: purpose, owned domains, API links, events produced/consumed, runbook link.
- Architecture decisions are ADRs (`docs/adr/NNNN-title.md`, MADR 3.0 format) and trace to CIC artifacts.
- API docs are generated from contracts; hand-written API docs are forbidden.

## 9. Testing standards (detailed in CORPUS-023)

Minimum per module: unit (domain ≥ 90 % line, ≥ 80 % mutation score on `-domain`), contract tests
for every published API and event, property tests for every resolution algorithm, scenario tests for
every state machine transition.

## 10. Build standards

- Reproducible builds: pinned toolchains, lockfiles committed, `--offline` build possible from cache.
- Every build produces: artifact, SBOM (CycloneDX 1.5), SLSA L3 provenance, test + coverage reports,
  `ucos-module.yaml` resolved trace report.
- Static gates: compile with warnings-as-errors, UCOS-L001…L0xx, ArchUnit rules, secret scan, license scan
  (allow-list: Apache-2.0, MIT, BSD-2/3, EPL-2.0, MPL-2.0).

## 11. Release and versioning standards

- SemVer per module (CIC-000-003 §4). API MAJOR = URL path segment `/v{N}`; event MAJOR = type suffix `.v{N}`.
- Trunk-based development; release branches only for supported MAJOR lines.
- Conventional Commits (`feat:`, `fix:`, `feat!:`) drive automated version computation.
- Release train: weekly MINOR/PATCH; MAJOR only with a migration artifact in CORPUS-026.

## 12. Certification Rules

- **CR-001-001-01** Architecture tests enforce §3 dependency rule with zero violations.
- **CR-001-001-02** Lint rules UCOS-L001 (hardcoded commercial literals), L002 (float money), L003 (local datetime) report zero violations.
- **CR-001-001-03** Every deployable module has a valid `ucos-module.yaml` whose `traces` resolve in the corpus.
- **CR-001-001-04** Domain-layer mutation score ≥ 80 % (PIT / Stryker).
- **CR-001-001-05** Every build publishes SBOM and SLSA provenance verifiable with `cosign verify-attestation`.
- **CR-001-001-06** Dependency license scan reports zero non-allow-listed licenses.

## 13. Validation Rules

- **VR-001-001-01** Every bounded context in the domain registry maps to exactly one module path under `kernel/` or `domains/`.
- **VR-001-001-02** Every name in published contracts matches §7 conventions (contract linter).
