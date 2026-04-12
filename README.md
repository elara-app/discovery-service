# Discovery Service

Service registry for the Elara microservices platform, built with **Netflix Eureka Server**.

This service is the runtime source of truth for service instance discovery. It allows clients and edge components to resolve backend locations dynamically instead of relying on static host/port wiring.

## What this service does

`discovery-service` is a Spring Boot application enabled with `@EnableEurekaServer`.

At runtime it:

- receives registrations from microservice instances,
- maintains the active service registry,
- exposes discovery metadata used by clients and infrastructure components.

Current local server settings (`src/main/resources/application.yml`):
- `spring.application.name: discovery-service`
- `server.port: 8761`
- `eureka.client.register-with-eureka: false`
- `eureka.client.fetch-registry: false`

## How it interacts with the rest of the platform

Primary consumers of service discovery:
- [inventory-service](https://github.com/elara-app/inventory-service.git)
- [unit-of-measure-service](https://github.com/elara-app/unit-of-measure-service.git)
- [api-gateway](https://github.com/elara-app/api-gateway.git)

### Interaction flow

1. `discovery-service` starts on port `8761`.
2. Service instances boot and register against `http://localhost:8761/eureka/`.
3. API Gateway and peer services resolve logical service IDs to healthy instances through Eureka.
4. Routing and service-to-service communication use registry data instead of fixed endpoints.

In this platform, service registration URLs are typically distributed through `config-service` from `centralized-configuration`, while this repository remains focused on Eureka Server behavior.

## Key endpoints

- Eureka dashboard: `http://localhost:8761/`
- Registry API base: `http://localhost:8761/eureka/`

Common inspection endpoint:

```bash
curl http://localhost:8761/eureka/apps
```

## Build, test, and run

Requirements:
- Java 21
- Maven Wrapper (`./mvnw`)

Commands:

```bash
./mvnw clean install
./mvnw test
./mvnw spring-boot:run
```

Single test execution:

```bash
./mvnw test -Dtest=DiscoveryServiceApplicationTests
./mvnw test -Dtest=DiscoveryServiceApplicationTests#contextLoads
```

## Operational best practices

- Start `discovery-service` before services that depend on registration/discovery.
- Keep service IDs stable (`spring.application.name`) to avoid route or lookup regressions.
- Avoid hardcoded downstream URLs in clients when discovery is the expected runtime contract.
- Monitor registry churn (rapid register/deregister cycles), as it usually indicates unhealthy instances or network instability.
- Keep this repository limited to discovery concerns; manage environment-specific client config in `centralized-configuration`.
