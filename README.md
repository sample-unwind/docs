# Parkora - Tehnična dokumentacija

Parkora je pametni parkirni sistem, implementiran z mikrostoritveno arhitekturo. Omogoča pregled zasedenosti parkirišč v realnem času, upravljanje uporabnikov, rezervacije, plačila in obveščanje.

## Kazalo dokumentacije

| Dokument | Opis |
|----------|------|
| [Arhitektura](arhitektura.md) | Pregled arhitekture sistema, mikrostoritve, podatkovne baze |
| [Namestitev](namestitev.md) | Navodila za Terraform, Helm, ročni deployment |
| [Vzorci](vzorci.md) | Uporabljeni arhitekturni vzorci (CQRS, Event Sourcing, Circuit Breaker, ...) |

## Diagrami

| Diagram | Opis |
|---------|------|
| [Arhitektura sistema](diagrami/arhitektura.puml) | Celotna arhitektura z vsemi komponentami |
| [Kubernetes deployment](diagrami/deployment.puml) | Namestitev v AKS z namespaces |
| [CQRS/Event Sourcing](diagrami/cqrs-event-sourcing.puml) | Vzorec v reservation-service |
| [Komunikacija](diagrami/komunikacija.puml) | Sinhrona in asinhrona komunikacija med servisi |

## Produkcijski URL-ji

| Storitev | URL | Opis |
|----------|-----|------|
| **Frontend** | https://parkora.crn.si/ | Spletna aplikacija |
| **Keycloak** | https://keycloak.parkora.crn.si/auth/ | Avtentikacija (OAuth2/OIDC) |
| **Grafana** | https://grafana.parkora.crn.si/ | Monitoring in metrike |
| **Kibana** | https://kibana.parkora.crn.si/ | Centralizirano beleženje |
| **Scraper Function** | Azure Portal | Serverless scraper (timer 10min) |

## API Endpoints

| Servis | URL | Tip |
|--------|-----|-----|
| User Service | https://parkora.crn.si/api/v1/user/graphql | GraphQL |
| Parking Service | https://parkora.crn.si/api/v1/parking/ | REST |
| Reservation Service | https://parkora.crn.si/api/v1/reservation/graphql | GraphQL |
| Payment Service | https://parkora.crn.si/api/v1/payment/ | REST + gRPC |

## Tehnologije

### Frontend
- **SvelteKit 2** - Full-stack framework
- **Svelte 5** - UI framework (runes, snippets)
- **Tailwind CSS v4** - Utility-first CSS
- **DaisyUI v5** - Tailwind component library

### Backend
- **Python/FastAPI** - user-service, reservation-service, payment-service, notification-service
- **Go 1.22** - parking-service
- **Strawberry GraphQL** - GraphQL API
- **gRPC** - Binarna komunikacija med servisi
- **RabbitMQ** - Asinhrono sporočanje

### Infrastruktura
- **Azure AKS** - Kubernetes cluster
- **Azure Functions** - Serverless izvajanje scraperja
- **Terraform** - Infrastructure as Code
- **Helm** - Kubernetes package manager
- **Kong** - API Gateway / Ingress Controller
- **cert-manager** - TLS certifikati (Let's Encrypt)

### Avtentikacija in podatki
- **Keycloak** - Identity and Access Management
- **PostgreSQL** - Relacijska baza
- **Supabase** - PostgreSQL hosting z REST API

### Monitoring
- **Prometheus** - Zbiranje metrik
- **Grafana** - Vizualizacija metrik
- **Elasticsearch + Filebeat + Kibana (EFK)** - Centralizirano beleženje

## GitHub repozitoriji

| Repozitorij | Opis |
|-------------|------|
| [frontend](https://github.com/sample-unwind/frontend) | SvelteKit spletna aplikacija |
| [core_infra](https://github.com/sample-unwind/core_infra) | Terraform infrastruktura |
| [user-service](https://github.com/sample-unwind/user-service) | Upravljanje uporabnikov (GraphQL) |
| [analytics-parkora](https://github.com/sample-unwind/analytics-parkora) | Parking service (Go) |
| [reservation-service](https://github.com/sample-unwind/reservation-service) | Rezervacije (CQRS/ES) |
| [payment-service](https://github.com/sample-unwind/payment-service) | Plačila (gRPC) |
| [notification-service](https://github.com/sample-unwind/notification-service) | Obveščanje (RabbitMQ) |

## Člani ekipe

- Žane Bučan
- Stjepan Sazonov
- Gregor Bučar

---

*RSO Projekt 2025/2026 - Fakulteta za računalništvo in informatiko, Univerza v Ljubljani*
