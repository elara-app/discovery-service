[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/elara-app/discovery-service)

# Discovery Service

`discovery-service` is the Eureka Server for the Elara platform. It owns the runtime registry of service instances: gateway, inventory and unit-of-measure instances register with it, send heartbeats and cancel on shutdown, and the gateway resolves those registrations to load-balanced targets. It does not own client configuration, routes, business data, authentication or authorization.

## At a glance

| Item | Value |
|---|---|
| Service ID | `discovery-service` |
| Runtime port | `8761` (`server.port` in `src/main/resources/application.yml`) |
| Stack | Java 21, Spring Boot 3.5.6, Spring Cloud 2025.0.0, `@EnableEurekaServer` |
| Datastore | None; the registry is in memory |
| Runtime dependencies | None: no database, broker, Vault or Config Server client |
| Health endpoint | `GET /actuator/health` |
| Published image | `ghcr.io/elara-app/discovery-service:latest` (also `sha-<short>` tags) |

## Responsibilities and boundaries

- Accept client registrations, heartbeats and cancellations through the standard Eureka REST protocol.
- Serve the Eureka dashboard and the machine-readable registry API.
- Run standalone: `eureka.client.register-with-eureka: false` and `fetch-registry: false`, so it neither registers itself nor downloads the registry.
- Stay registry-only. It does not own:
  - client configuration (registration URLs come from `centralized-configuration` through `config-service`),
  - routing (`api-gateway` owns the route table and `lb://` resolution),
  - business data (`inventory-service`, `unit-of-measure-service`),
  - security (there is no authentication on any surface).

## Runtime architecture

`DiscoveryServiceApplication` is a single Spring Cloud Netflix Eureka Server. The registry is single-node and in memory: there is no replication, failover, persistence or backup. Scaling the container creates a second independent registry, not a replica.

1. Clients read `eureka.client.service-url.defaultZone` from their own config. Host-run clients default to `http://localhost:8761/eureka/`; Compose sets `EUREKA_URI=http://discovery-service:8761/eureka/`.
2. Each instance registers as `${spring.application.name}:${random.uuid}` with `prefer-ip-address: true` and version/environment/region metadata.
3. `api-gateway`, `inventory-service` and `unit-of-measure-service` register and resolve through discovery. `config-service` and `discovery-service` do not register.
4. Docker health-gates the platform: gateway, inventory and UOM wait for discovery to become healthy. Discovery itself has no `depends_on` and does not wait for config, databases or the broker.

Ports, stack layout and the canonical full-stack workflow live in [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme). Client profile values live in [`centralized-configuration`](https://github.com/elara-app/centralized-configuration).

## Interfaces

| Surface | Path | Notes |
|---|---|---|
| Eureka dashboard | `GET /` | Browser UI listing registered applications and instances. |
| Registry listing | `GET /eureka/apps` | Machine-readable registry. Send `Accept: application/json`; XML is also produced. |
| Application / instance reads | `GET /eureka/apps/{app}`, `GET /eureka/apps/{app}/{instanceId}` | Standard Eureka read endpoints. |
| Client protocol | `/eureka/apps/...` | Registration, heartbeat and cancellation. No custom controller or error mapping. |
| Health | `GET /actuator/health` | Actuator exposes `health,info` only; `show-details` is not configured. |
| Info | `GET /actuator/info` | Exposed, but no info contributors are configured. |

```bash
curl -s -H 'Accept: application/json' http://127.0.0.1:8761/eureka/apps | jq
curl -s http://127.0.0.1:8761/actuator/health
```

There is no Swagger/OpenAPI UI and no custom API in this repo. `/eureka/apps` is the useful listing endpoint; the bare `/eureka/` is only a base path. There is no authentication, so anyone who can reach port 8761 can read the registry and attempt registration.

## Run and verify

Prerequisites: Java 21. The Maven wrapper is not executable (mode 644), so always call it as `sh ./mvnw`.

Run the server directly; it has no external dependencies:

```bash
sh ./mvnw spring-boot:run
```

Smoke checks:

```bash
# Application health; expect {"status":"UP"}
curl -s http://127.0.0.1:8761/actuator/health

# Registry listing; expect an empty applications list until clients register
curl -s -H 'Accept: application/json' http://127.0.0.1:8761/eureka/apps
```

Open the dashboard at `http://127.0.0.1:8761/`. A standalone server shows an empty registry; that is expected until clients register.

For the full stack, follow [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme). After the stack is up, the dashboard is at `http://127.0.0.1:8761/` and the registry should list `API-GATEWAY`, `INVENTORY-SERVICE` and `UNIT-OF-MEASURE-SERVICE` instances.

## Configuration

`src/main/resources/application.yml` owns the runtime contract:

- `spring.application.name: discovery-service`
- `server.port: 8761`
- `eureka.client.register-with-eureka: false`, `eureka.client.fetch-registry: false`
- `management.endpoints.web.exposure.include: health,info`

`application-test.yml` disables Spring Cloud Config and repeats the two Eureka flags; CI runs with `-Dspring.profiles.active=test`.

There is no `spring.config.import` and no Config Server client, so the Config Server is not a dependency. The source Compose file sets `SERVER_PORT: ${SERVER_PORT_DISCOVERY:-8761}`, but that is not a supported relocation: `server.port` is a literal 8761 and the container port mapping and healthcheck stay on 8761. Changing the port means updating both Compose files, their healthchecks and every client's `EUREKA_URI`.

Client-side configuration for gateway, inventory and UOM is owned by [`centralized-configuration`](https://github.com/elara-app/centralized-configuration); see its README for the profile matrix.

## Testing and quality gates

The only test is `DiscoveryServiceApplicationTests#contextLoads`, a `@SpringBootTest` with no assertions. It proves the application context starts, nothing more.

```bash
sh ./mvnw test                                          # all tests
sh ./mvnw test -Dtest=DiscoveryServiceApplicationTests  # focused test
sh ./mvnw -B clean verify -Dspring.profiles.active=test # canonical verify (CI)
```

- CI: `.github/workflows/test.yml` runs the reusable org test workflow on pull requests to `develop` and `main`, executing the canonical verify command on Java 21 Temurin.
- Publish: `.github/workflows/publish-image.yml` runs on pushes to `main` (markdown-only changes are ignored). It runs the test job, then builds and pushes multi-arch (`amd64`/`arm64`) `latest` and `sha-<short>` tags to GHCR.
- No coverage or static-analysis gate: the POM declares only `spring-boot-maven-plugin`; there is no JaCoCo, Checkstyle, SpotBugs or Enforcer.

## Current limitations

- Single node, in-memory registry. No replication, failover, persistence or backup; restarting discovery loses all registrations until clients re-register.
- No security. The dashboard, `/eureka/apps` and the registration API are unauthenticated. The source stack publishes `8761:8761` on all host interfaces; the published stack binds loopback by default.
- Default Eureka self-preservation applies; no `eureka.server.*` tuning is configured.
- Health is application health. `/actuator/health` returning `UP` does not prove that expected clients are registered or that `/eureka/apps` is usable.
- No test asserts registration, heartbeat, dashboard or registry behavior, so registry changes are not caught by CI.
- Published image tags are `latest` and `sha-<short>`; there is no image tag matching the POM version `1.3.1`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `./mvnw: Permission denied` | the wrapper is mode 644 | run `sh ./mvnw ...` |
| Dashboard shows no applications | no clients registered yet, or clients cannot reach discovery | check client `defaultZone`/`EUREKA_URI`, then `curl -s -H 'Accept: application/json' http://127.0.0.1:8761/eureka/apps` |
| Clients cannot reach discovery inside Compose | they use `localhost` | use `http://discovery-service:8761/eureka/` |
| Registry empty after restarting discovery | the registry is in memory | wait for clients to re-register; registrations are not restored |
| Health `DOWN` or connection refused | server not started or port conflict | check port 8761 and `curl -s http://127.0.0.1:8761/actuator/health` |
| Compose `up --wait` fails | discovery never became healthy, so health-gated dependents did not start | inspect `docker compose logs discovery-service` |

## Documentation

- [`AGENTS.md`](AGENTS.md) — agent rules, commands, invariants and known traps.
- [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme) — stack layout, ports and canonical Compose verification.
- [`centralized-configuration`](https://github.com/elara-app/centralized-configuration) — client profiles, `EUREKA_URI` and the profile matrix.
- [`api-gateway`](https://github.com/elara-app/api-gateway) — route table, discovery locator and `lb://` resolution.
