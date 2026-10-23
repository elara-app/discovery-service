# AGENTS — discovery-service

Guidance for coding agents changing this repository. Source files and Compose are authoritative; prose is not.

## Read first

- [`README.md`](README.md) — purpose, interfaces, run/verify, limitations, troubleshooting.
- [`src/main/resources/application.yml`](src/main/resources/application.yml) — the runtime contract: service ID, port 8761, Eureka flags, actuator exposure.
- [`src/main/resources/application-test.yml`](src/main/resources/application-test.yml) — the test profile CI activates.
- [`infra/README.md`](../infra/README.md) — stack layout, ports and the canonical Compose workflow.
- [`centralized-configuration`](https://github.com/elara-app/centralized-configuration) — client `EUREKA_URI` and profile matrix (not owned here).

## Scope and boundaries

This repo owns the Eureka Server registry only.

- In scope: Eureka server configuration, actuator exposure, Dockerfile/Compose wiring for this service, and tests of the application context.
- Out of scope: business logic, routing, persistence, authentication, client configuration, and a Config Server client.
- Do not add databases, brokers, Vault, or a Config Server client/dependency without an explicit platform decision (see "Stop and ask").
- Do not enable self-registration or registry fetch. `register-with-eureka: false` and `fetch-registry: false` are the contract, in both `application.yml` and `application-test.yml`.
- Preserve the stable service IDs and registry contract: clients register as `api-gateway`, `inventory-service`, `unit-of-measure-service`; discovery does not register and `config-service` does not either.

## Quick facts

| Fact | Value |
|---|---|
| Package / main class | `com.elara.app.discovery_service`, `DiscoveryServiceApplication` |
| Service ID | `discovery-service` |
| Port | `8761`, literal in `application.yml` (no placeholder) |
| Java / Boot / Cloud | 21 / 3.5.6 / 2025.0.0 |
| Runtime dependencies | none: no DB, broker, Vault or Config Server |
| Server annotation | `@EnableEurekaServer` |
| Eureka clients | `api-gateway`, `inventory-service`, `unit-of-measure-service` |
| Non-clients | `config-service`, `discovery-service` |

## Commands

Always call the wrapper as `sh ./mvnw`. Both wrappers are mode 644, so `./mvnw` fails with `Permission denied`; never instruct `chmod +x`.

```bash
sh ./mvnw test                                            # all tests
sh ./mvnw test -Dtest=DiscoveryServiceApplicationTests    # focused test
sh ./mvnw -B clean verify -Dspring.profiles.active=test   # canonical verify (CI)
sh ./mvnw spring-boot:run                                 # standalone; no external deps
docker compose -f ../infra/compose.yml config --quiet
docker compose -f ../infra/compose.published.yml config --quiet
```

Full-stack Compose verification: use the block in [`infra/README.md#verify-the-stack`](../infra/README.md#verify-the-stack). Do not paste or duplicate it here.

## Repository map

| Path | Role |
|---|---|
| `src/main/java/.../DiscoveryServiceApplication.java` | boot class + `@EnableEurekaServer` |
| `src/main/resources/application.yml` | service ID, port, Eureka flags, actuator exposure |
| `src/main/resources/application-test.yml` | test profile: config client disabled, Eureka flags repeated |
| `src/test/java/.../DiscoveryServiceApplicationTests.java` | only test: `contextLoads` |
| `Dockerfile` | multi-stage build; JRE runtime, uid 10001, `EXPOSE 8761`; no baked healthcheck |
| `.github/workflows/test.yml` | PR test workflow (reusable org workflow) |
| `.github/workflows/publish-image.yml` | image publish on `main` |

## Contracts and invariants

- Service ID stays `discovery-service`; port stays 8761.
- `eureka.client.register-with-eureka: false` and `eureka.client.fetch-registry: false` stay false.
- Actuator exposes only `health,info`; `/actuator/health` is the container health contract.
- The registry is single-node and in-memory: no replication, failover, persistence or backup exists or may be implied.
- Clients register as `${spring.application.name}:${random.uuid}` with `prefer-ip-address: true`; the gateway resolves `lb://inventory-service` and `lb://unit-of-measure-service`.
- There is no security layer. Do not document or assume authentication on the dashboard, `/eureka/apps`, or the registration API.
- Compose healthcheck contract: `curl -f http://localhost:8761/actuator/health`, interval 15s, timeout 5s, retries 10, start period 60s.
- `/eureka/apps` is the listing endpoint; `/eureka/` is only a base path. JSON requires `Accept: application/json`.

## Configuration and runtime

- `application.yml` is the single runtime config source in this repo. There is no `spring.config.import` and no Config Server client; the Config Server is not a dependency.
- `SERVER_PORT_DISCOVERY` in Compose is not a supported relocation mechanism. The container mapping (source `8761:8761`; published `${ELARA_BIND:-127.0.0.1}:${ELARA_PORT_DISCOVERY:-8761}:8761`) and healthchecks stay on 8761. Moving the port requires coordinated changes to both Compose files, their healthchecks and every client `EUREKA_URI`.
- Host-run clients default to `http://localhost:8761/eureka/`; inside Compose clients use `http://discovery-service:8761/eureka/`.
- `SERVER_PORT` raises precedence questions but the YAML literal is 8761 and the healthcheck probes 8761 regardless; treat 8761 as fixed.
- The Dockerfile packages with tests skipped, so `docker compose build` does not run the test suite.

## Testing and quality gates

- One test: `DiscoveryServiceApplicationTests#contextLoads`. It asserts the context starts and nothing else. Add tests when you add behavior; do not rename existing tests or imply more coverage.
- Canonical verify: `sh ./mvnw -B clean verify -Dspring.profiles.active=test`. PR CI to `develop`/`main` runs exactly this on Java 21 Temurin.
- No coverage or static-analysis gate: no JaCoCo, Checkstyle, SpotBugs, Enforcer or Failsafe. Do not claim a gate exists; do not add one without asking.
- Publish workflow on `main` runs the tests before the multi-arch image push; markdown-only pushes are ignored.

## Definition of done

- [ ] `sh ./mvnw -B clean verify -Dspring.profiles.active=test` passes locally.
- [ ] Service ID, port 8761 and the two Eureka flags are unchanged.
- [ ] Compose/healthcheck impact considered and consumer `EUREKA_URI` contract not broken.
- [ ] README/AGENTS stay consistent with source; no unqualified `./mvnw` commands or stale claims.
- [ ] Documentation-only tasks touch documentation only; no application code, resources, Compose or workflow changes unless the task explicitly asks.
- [ ] `git status` shows only the files the task allows.

## Stop and ask

Stop and get an explicit decision before:

- Adding replication, HA, a Eureka peer/service-url list, or any multi-node registry topology. Scaling a single-node Eureka server is not replication.
- Adding Spring Security or any authentication/authorization for the dashboard or registry API.
- Adding persistence, a database, backup or restore for the registry.
- Moving the port or changing container mappings/healthchecks.
- Adding a Config Server client or any new runtime dependency.
- Changing client registration semantics (instance-id format, `prefer-ip-address`, metadata, `defaultZone`).

## Known traps

- `./mvnw` fails (mode 644); use `sh ./mvnw`. Never instruct `chmod +x`.
- `SERVER_PORT_DISCOVERY` does not make port relocation supported; the mapping and healthcheck stay on 8761.
- The registry is in memory: restarting discovery empties it until clients re-register. Never claim persistence or HA.
- No security: the dashboard and `/eureka/apps` are unauthenticated. The source stack publishes 8761 on all host interfaces; the published stack binds loopback by default.
- `/actuator/health` returning `UP` is not registry-readiness proof. It does not show registered clients; check `/eureka/apps`.
- `/eureka/` is not the listing; use `/eureka/apps` with `Accept: application/json` for JSON (XML is returned otherwise).
- Published image tags are `latest` and `sha-<short>`; there is no `1.3.1` tag.
- `config-service` and `discovery-service` are not Eureka clients; only gateway, inventory and UOM are.
- Self-preservation is on by default; no `eureka.server.*` tuning exists.
- Do not duplicate the README interfaces table or the infra Compose block here; link instead.

## Documentation maintenance

- `README.md` owns human onboarding: purpose, interfaces, run/verify, limitations, troubleshooting.
- `AGENTS.md` (this file) owns agent rules: commands, invariants, gates, stop conditions, traps.
- `infra/README.md` owns the stack/port overview and canonical Compose commands; `centralized-configuration/README.md` owns profiles, property ownership and secrets.
- When behavior changes, update the owning document, then fix links. Do not copy facts between documents; change the owner and point to it.
- Keep both files within their line targets and free of stale `./mvnw`, `localhost`-inside-Compose, Config-Server-client and HA/persistence claims.
