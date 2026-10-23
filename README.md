<p align="right"><b>EN</b> | <a href="README.fr.md">FR</a></p>

# K8s Microservices

An e-commerce application built as Spring Cloud microservices, secured with Keycloak, deployed on Kubernetes through a Jenkins pipeline and monitored with the Grafana stack (Prometheus, Loki, Tempo).

![Java](https://img.shields.io/badge/Java-17-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.3-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Spring Cloud](https://img.shields.io/badge/Spring_Cloud-2023.0-6DB33F?style=flat-square&logo=spring&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-17-DD0031?style=flat-square&logo=angular&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-OIDC-4D4D4D?style=flat-square&logo=keycloak&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Minikube-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=flat-square&logo=jenkins&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-observability-F46800?style=flat-square&logo=grafana&logoColor=white)

## Architecture

```mermaid
flowchart LR
    U[Angular web client] -->|JWT| GW[API Gateway<br/>:9999]
    U -.->|OIDC login| KC[Keycloak]
    KC --- PG[(PostgreSQL)]

    GW -->|/customers/**| CS[customer-service]
    GW -->|/products/**| IS[inventory-service]
    GW -->|/orders/**| OS[order-service]
    OS -->|OpenFeign + token relay| CS
    OS -->|OpenFeign + token relay| IS

    CFG[config-service] -->|Git| REPO[(config-repo)]
    CS & IS & OS & GW -.-> CFG
    CS & IS & OS & GW -.-> CON[Consul<br/>service discovery]
```

| Component | Role |
|---|---|
| **geteway-service** | Spring Cloud Gateway: single entry point, routing, CORS, JWT validation |
| **customer-service** | Customer management (Spring Data REST, projections) |
| **inventory-service** | Product catalog |
| **order-service** | Orders; calls the customer and inventory services through OpenFeign, forwarding the user's OAuth2 token |
| **config-service** | Spring Cloud Config Server, backed by the [config-repo](https://github.com/saber0amine/config-repo) Git repository |
| **Consul** | Service registry and discovery (StatefulSet on Kubernetes) |
| **Keycloak** | Identity provider (OpenID Connect), PostgreSQL storage |
| **ecom-web-app** | Angular 17 front end with `keycloak-angular`, served by Nginx |

## Security

Users authenticate against Keycloak from the Angular app. Each request carries a JWT that the gateway and every service validate as an OAuth2 resource server; Keycloak roles are mapped to Spring authorities by a custom `JwtAuthConverter`. For service-to-service calls, an `OAuth2FeignRequestInterceptor` propagates the token from `order-service` to the downstream services.

## Observability

```mermaid
flowchart LR
    SVC[Microservices<br/>Actuator + Micrometer] -->|metrics| P[Prometheus]
    SVC -->|traces| T[Tempo]
    SVC -->|logs| L[Loki]
    C[cAdvisor] -->|container metrics| P
    P & T & L --> G[Grafana]
```

- **Metrics:** `/actuator/prometheus` endpoints scraped by Prometheus, with HTTP latency histograms.
- **Distributed tracing:** Micrometer Tracing (Brave), 100% sampling; `traceId` and `spanId` injected into every log line.
- **Dashboards:** Grafana with a preconfigured dashboard and data sources ([docker/grafana](docker/grafana)).

## CI/CD

The [Jenkinsfile](Jenkinsfile) automates the whole delivery chain:

```mermaid
flowchart LR
    A[Checkout] --> B[Build Docker images] --> C[Push to Docker Hub] --> D[kubectl apply on Minikube]
```

The Kubernetes manifests are in [kubernetes-Manifests](kubernetes-Manifests):

| Folder | Content |
|---|---|
| `Springboot-k8s-main/` | Deployments and Services for each microservice, the gateway, Consul, Keycloak (with Ingress), PostgreSQL, MySQL, Adminer and the web client |
| `observability-manifests/` | Prometheus, Grafana, Loki, Tempo and cAdvisor |

Services wait for their dependencies (Config Server, Consul) with init containers before starting.

## Getting started

### Kubernetes (Minikube)

```bash
minikube start
kubectl apply -f kubernetes-Manifests/Springboot-k8s-main/
kubectl apply -f kubernetes-Manifests/observability-manifests/
```

| Service | NodePort |
|---|---|
| Web client | 32542 |
| API Gateway | 31769 |
| Keycloak | 31095 |
| Adminer | 31085 |

### Docker Compose

```bash
docker compose up -d
```

Starts the microservices, Consul and Vault locally (gateway on port 9999).

## Project structure

```text
.
├── config-service/          Spring Cloud Config Server
├── customer-service/        Customer microservice
├── inventory-service/       Product microservice
├── order-service/           Order microservice (OpenFeign)
├── geteway-service/         Spring Cloud Gateway
├── ecom-web-app/            Angular front end
├── kubernetes-Manifests/    Kubernetes manifests
├── docker/                  Prometheus, Grafana and Tempo configuration
├── docker-compose.yml
└── Jenkinsfile
```
