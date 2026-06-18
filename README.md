# platform-shared-contracts

Single source of truth for **Hubinity cross-service contracts**:

- **REST APIs** as OpenAPI 3.1 specs (one YAML per bounded context)
- **Domain events** as JSON Schema 2020-12 documents

Backend services depend on the relevant module via Maven and get a set of
Jackson-annotated Java DTOs generated at build time. There are **no hand-coded
duplicate DTOs** across the platform.

## Module layout

```
platform-shared-contracts/
  pom.xml                       <- parent (packaging: pom)
  settings.xml                  <- Maven settings template (GitHub Packages)
  contracts-catalog/
    pom.xml
    openapi/catalog.yaml        <- Product, Category, StockItem, StockMovement
  contracts-order/
    pom.xml
    openapi/order.yaml          <- Order, OrderItem, Payment
  contracts-support/
    pom.xml
    openapi/support.yaml        <- Ticket, TicketItem, Customer, Service, Technician
  contracts-cashier/
    pom.xml
    openapi/cashier.yaml        <- LedgerEntry, LedgerCategory, CashSession
  contracts-events/
    pom.xml
    events/*.schema.json        <- OrderPaid, OrderCancelled,
                                   ServiceRevenueGenerated, StockChanged,
                                   PriceChanged, ProductCreated,
                                   ProductUpdated, ProductDeactivated
```

The four `contracts-<domain>` modules use the
[`openapi-generator-maven-plugin`](https://github.com/OpenAPITools/openapi-generator)
in **model-only** mode (no server stubs, no client invoker). The
`contracts-events` module uses
[`jsonschema2pojo-maven-plugin`](https://github.com/joelittlejohn/jsonschema2pojo)
because event payloads are pure JSON Schema, not OpenAPI.

Generated sources land in `target/generated-sources/openapi/src/main/java/`
and are added to the compile path by `build-helper-maven-plugin`. The output
classes live under `com.hubinity.contracts.<domain>.dto` (or
`com.hubinity.contracts.events` for events).

## Requirements

- **JDK 21** (Temurin recommended)
- **Maven 3.9.x**

## Build

```bash
mvn clean install
```

That triggers `generate-sources` -> `compile` -> `install` per module. After a
clean install, the artifacts are available in your local repo as:

| Artifact id           | Coordinate                                           |
| --------------------- | ---------------------------------------------------- |
| contracts-catalog     | `com.hubinity:contracts-catalog:0.1.0-SNAPSHOT`      |
| contracts-order       | `com.hubinity:contracts-order:0.1.0-SNAPSHOT`        |
| contracts-support     | `com.hubinity:contracts-support:0.1.0-SNAPSHOT`      |
| contracts-cashier     | `com.hubinity:contracts-cashier:0.1.0-SNAPSHOT`      |
| contracts-events      | `com.hubinity:contracts-events:0.1.0-SNAPSHOT`       |

To regenerate sources without running tests:

```bash
mvn -DskipTests generate-sources
```

## Consuming from a service repository

Each service depends on **only** the modules it needs. Example for the
Catalog backend:

```xml
<dependency>
    <groupId>com.hubinity</groupId>
    <artifactId>contracts-catalog</artifactId>
    <version>0.1.0-SNAPSHOT</version>
</dependency>
```

A service that publishes domain events also pulls in `contracts-events`:

```xml
<dependency>
    <groupId>com.hubinity</groupId>
    <artifactId>contracts-events</artifactId>
    <version>0.1.0-SNAPSHOT</version>
</dependency>
```

While these artifacts are `-SNAPSHOT` and not yet published, services pick
them up from the developer's local Maven repository after a `mvn install`
here. Once GitHub Packages publishing is enabled (see below), services
authenticate via `settings.xml` and resolve from the remote.

## Publishing (future)

Publishing to GitHub Packages is **not enabled** in this initial slice — the
parent POM contains a commented-out `<distributionManagement>` block. To
enable it later:

1. Uncomment the `<distributionManagement>` block in `./pom.xml`.
2. Copy `./settings.xml` to `~/.m2/settings.xml` (or merge the `<servers>`
   entry) and export `GITHUB_USERNAME` and `GITHUB_TOKEN`.
3. Run `mvn -DskipTests deploy`.

The `ci.yml` workflow only builds and verifies — it does not publish.

## Versioning policy

This repo follows **SemVer** at the module level. Until v1.0.0, breaking
changes can land in minor bumps. After v1.0.0:

- **Patch (0.0.x)** — backward-compatible additions to optional fields, doc
  changes.
- **Minor (0.x.0)** — backward-compatible additions of new schemas, new
  endpoints, new event types.
- **Major (x.0.0)** — breaking changes (removed/renamed fields, type
  changes, required field added without default). When shipping a major,
  the previous major **must continue to be published in parallel for at
  least one release cycle** so consumers can migrate.

## Cookbook: add a new contract module

1. Pick a domain name, e.g. `inventory`.
2. Create `contracts-inventory/` with the same layout as a sibling module
   (copy `contracts-catalog/` as a template).
3. Add `<module>contracts-inventory</module>` to the parent `pom.xml`.
4. Create `contracts-inventory/openapi/inventory.yaml` (or
   `events/*.schema.json` for an events-only module).
5. Update the openapi-generator execution `inputSpec`, `modelPackage` and
   `apiPackage` to reference the new domain.
6. Run `mvn -DskipTests generate-sources -pl contracts-inventory` and
   inspect `contracts-inventory/target/generated-sources/openapi/` to
   confirm the expected POJOs are produced.
7. `mvn -B verify` from the repo root to confirm the whole build still
   passes.
