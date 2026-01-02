# Arhitektura sistema

Ta dokument opisuje arhitekturo sistema Parkora - pametnega parkirnega sistema, implementiranega z mikrostoritvami.

## Pregled arhitekture

![Arhitektura sistema](https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/sample-unwind/docs/main/diagrami/arhitektura.puml)

Parkora je sestavljena iz več neodvisnih mikrostoritev, ki komunicirajo preko REST/GraphQL API-jev, gRPC in sporočilne vrste RabbitMQ. Vse storitve so nameščene v Azure Kubernetes Service (AKS) in dostopne preko Kong API Gateway-a.

## Mikrostoritve

### user-service

| Lastnost | Vrednost |
|----------|----------|
| **Jezik** | Python 3.11+ |
| **Framework** | FastAPI + Strawberry GraphQL |
| **Baza** | PostgreSQL (v Kubernetes) |
| **Port** | 8000 |
| **API** | GraphQL |

**Funkcionalnosti:**
- Upravljanje uporabniških profilov
- CRUD operacije za uporabnike
- Multitenancy z PostgreSQL RLS
- JWT token validacija preko Keycloak

**GraphQL operacije:**
- Queries: `users`, `user(id)`, `userByEmail`, `userByKeycloakId`
- Mutations: `createUser`, `updateUser`, `deleteUser`

---

### parking-service

| Lastnost | Vrednost |
|----------|----------|
| **Jezik** | Go 1.22 |
| **Framework** | net/http (standard library) |
| **Baza** | Supabase (PostgreSQL) |
| **Port** | 4000 |
| **API** | REST |

**Funkcionalnosti:**
- Pregled parkirišč in lokacij
- Trenutna zasedenost parkirišč
- Zgodovina zasedenosti
- Vremenske podatke (OpenWeatherMap API)
- Circuit Breaker za toleranco napak

**Endpointi:**
- `GET /analytics/parkings` - Seznam parkirišč
- `GET /analytics/availability/current` - Trenutna zasedenost
- `GET /analytics/availability/history?id=<id>` - Zgodovina
- `GET /weather` - Vremenske razmere (z Circuit Breaker)

---

### reservation-service

| Lastnost | Vrednost |
|----------|----------|
| **Jezik** | Python 3.11+ |
| **Framework** | FastAPI + Strawberry GraphQL |
| **Baza** | PostgreSQL (v Kubernetes) |
| **Port** | 8000 |
| **API** | GraphQL |

**Funkcionalnosti:**
- Upravljanje rezervacij parkirnih mest
- CQRS (Command Query Responsibility Segregation)
- Event Sourcing - shranjevanje vseh sprememb kot dogodkov
- Multitenancy z PostgreSQL RLS
- gRPC klic na payment-service za plačila

**GraphQL operacije:**
- Queries: `reservations`, `reservationById`, `reservationsByUser`, `checkAvailability`, `eventsByReservation`
- Mutations: `createReservation`, `confirmReservation`, `cancelReservation`, `completeReservation`, `payReservation`

**Event tipi:**
- `RESERVATION_CREATED`, `RESERVATION_CONFIRMED`, `RESERVATION_CANCELLED`
- `RESERVATION_COMPLETED`, `RESERVATION_EXPIRED`
- `PAYMENT_PROCESSED`, `PAYMENT_FAILED`

---

### payment-service

| Lastnost | Vrednost |
|----------|----------|
| **Jezik** | Python 3.11+ |
| **Framework** | FastAPI + gRPC |
| **Baza** | PostgreSQL (v Kubernetes) |
| **Porti** | 8000 (REST), 50051 (gRPC) |
| **API** | REST + gRPC |

**Funkcionalnosti:**
- Procesiranje plačil
- gRPC server za sinhrono komunikacijo
- Objava dogodkov v RabbitMQ po uspešnem plačilu
- Multitenancy z PostgreSQL RLS

**gRPC operacije:**
- `ProcessPayment` - Procesira plačilo za rezervacijo

**RabbitMQ:**
- Exchange: `parkora_events` (topic)
- Routing key: `payment.processed`

---

### notification-service

| Lastnost | Vrednost |
|----------|----------|
| **Jezik** | Python 3.11+ |
| **Framework** | FastAPI |
| **Port** | 8000 |
| **API** | REST (health checks) |

**Funkcionalnosti:**
- Posluša RabbitMQ za `payment.processed` dogodke
- Pošilja push obvestila preko ntfy.sh
- Obveščanje uporabnikov o uspešnih plačilih

**RabbitMQ:**
- Queue: `notification_queue`
- Binding: `payment.processed`

---

## Podatkovna plast

### PostgreSQL (v Kubernetes)

Deljena PostgreSQL instanca (v `keycloak` namespace) gosti več baz:

| Baza | Uporaba |
|------|---------|
| `keycloak` | Keycloak podatki |
| `user_service` | Uporabniški profili |
| `reservation_service` | Rezervacije in event store |
| `payment_service` | Plačila |

**Multitenancy:**
Vse tabele uporabljajo PostgreSQL Row-Level Security (RLS) za izolacijo najemnikov:
- Stolpec `tenant_id` v vsaki tabeli
- RLS policy: `USING (tenant_id = current_setting('app.tenant_id')::UUID)`
- API nastavi `app.tenant_id` iz `X-Tenant-ID` headerja

### Supabase

Parking-service uporablja Supabase za podatke o parkiriščih:
- Baza: PostgreSQL na Supabase platformi
- API: Supabase REST API z service role key
- Podatki: Seznam parkirišč, zgodovina zasedenosti

---

## Avtentikacija (Keycloak)

| Lastnost | Vrednost |
|----------|----------|
| **URL** | https://keycloak.parkora.crn.si/auth/ |
| **Realm** | `parkora` |
| **Verzija** | 26.4 |

### Clienti

| Client ID | Tip | Uporaba |
|-----------|-----|---------|
| `frontend-app` | Public | SvelteKit frontend (PKCE flow) |
| `backend-services` | Bearer-only | Mikrostoritve (token validacija) |

### Testni uporabniki

| Uporabnik | Geslo |
|-----------|-------|
| `testuser1` | `password123` |
| `testuser2` | `password123` |

### OIDC Flow

1. Uporabnik klikne "Prijava" na frontendu
2. Frontend preusmeri na Keycloak `/auth/realms/parkora/protocol/openid-connect/auth`
3. Uporabnik se prijavi
4. Keycloak preusmeri nazaj z authorization code
5. Frontend zamenja code za tokene (access, refresh, id)
6. Tokeni shranjeni v HTTP-only secure cookies
7. API klici vsebujejo `Authorization: Bearer <token>` header
8. Mikrostoritve validirajo token preko Keycloak JWKS

---

## API Gateway (Kong)

| Lastnost | Vrednost |
|----------|----------|
| **IP** | 72.144.69.17 |
| **Namespace** | `kong` |
| **Ingress Class** | `kong` |

### Routing

| Path | Servis | Namespace |
|------|--------|-----------|
| `/` | frontend | parkora |
| `/api/v1/user` | user-service | parkora |
| `/api/v1/parking` | parking-service | parkora |
| `/api/v1/reservation` | reservation-service | parkora |
| `/api/v1/payment` | payment-service | parkora |

### Rate Limiting

| Servis | Zahtevkov/minuto | Zahtevkov/uro |
|--------|------------------|---------------|
| Frontend (UI) | 100 | 1000 |
| API servisi | 20 | 200 |
| Keycloak (Auth) | 10 | 100 |

### TLS

- Certifikati: Let's Encrypt (letsencrypt-http01)
- Upravljanje: cert-manager
- Veljavnost: Avtomatsko podaljšanje

---

## Monitoring in beleženje

### Prometheus + Grafana

| Storitev | URL | Uporabnik | Geslo |
|----------|-----|-----------|-------|
| Grafana | https://grafana.parkora.crn.si/ | admin | admin |

**Metrike:**
- CPU, memory, network usage
- Kubernetes pod status
- API latency in error rates

### EFK Stack (Elasticsearch + Filebeat + Kibana)

| Storitev | URL | Uporabnik | Geslo |
|----------|-----|-----------|-------|
| Kibana | https://kibana.parkora.crn.si/ | elastic | NILXDRm5Em3FOFOG |

**Funkcionalnosti:**
- Centralizirano zbiranje logov iz vseh podov
- Iskanje in filtriranje logov
- Dashboardi za analizo

---

## Kubernetes namespaces

![Deployment diagram](https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/sample-unwind/docs/main/diagrami/deployment.puml)

| Namespace | Komponente |
|-----------|------------|
| `parkora` | frontend, user-service, parking-service, reservation-service, payment-service, notification-service, RabbitMQ, Prometheus, Grafana, Elasticsearch, Kibana |
| `keycloak` | Keycloak, PostgreSQL |
| `kong` | Kong Ingress Controller |
| `cert-manager` | cert-manager za TLS certifikate |

---

## Azure viri

| Vir | Ime | Opis |
|-----|-----|------|
| Resource Group | `rg-parkora` | Vsebuje vse Parkora vire |
| AKS Cluster | `aks-parkora` | Kubernetes cluster (Free tier, 1x B2ps_v2 ARM node) |
| Key Vault | `kv-parkora` | Centralno shranjevanje skrivnosti |
| Storage Account | `stparkoratfstate` | Terraform state backend |

**Regija:** `germanywestcentral` (zahteva Azure Student subscription)

---

## Omrežni diagram

```
                         Internet
                             │
                             ▼
                    ┌─────────────────┐
                    │  DNS A Record   │
                    │ parkora.crn.si  │
                    │  72.144.69.17   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Kong Ingress   │
                    │   Controller    │
                    │  (LoadBalancer) │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐        ┌─────▼─────┐       ┌─────▼─────┐
    │Frontend │        │ /api/v1/* │       │ Keycloak  │
    │ :3000   │        │  Services │       │  :8080    │
    └─────────┘        └─────┬─────┘       └───────────┘
                             │
         ┌─────────┬─────────┼─────────┬─────────┐
         │         │         │         │         │
    ┌────▼────┐ ┌──▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼────┐
    │  user   │ │parking│ │reserv.│ │payment│ │notific.│
    │ :8000   │ │ :4000 │ │ :8000 │ │ :8000 │ │ :8000  │
    └────┬────┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬────┘
         │         │         │         │         │
         │         │         │    gRPC │         │
         │         │         └────────►│         │
         │         │                   │         │
         │         │          RabbitMQ │         │
         │         │         ┌─────────┘         │
         │         │         │                   │
         ▼         ▼         ▼                   ▼
    ┌─────────────────────────────┐         ┌────────┐
    │       PostgreSQL            │         │RabbitMQ│
    │  (user_service, payment_    │         │        │
    │   service, reservation_     │         │        │
    │   service, keycloak)        │         │        │
    └─────────────────────────────┘         └────────┘
```

---

## Naslednji koraki

Za navodila o namestitvi sistema glej [Namestitev](namestitev.md).

Za opis uporabljenih arhitekturnih vzorcev glej [Vzorci](vzorci.md).
