# Uporabljeni arhitekturni vzorci

Ta dokument opisuje arhitekturne vzorce, ki so implementirani v sistemu Parkora.

## Pregled vzorcev

| Vzorec | Storitev | Namen |
|--------|----------|-------|
| [CQRS](#cqrs---command-query-responsibility-segregation) | reservation-service | Ločitev read/write operacij |
| [Event Sourcing](#event-sourcing) | reservation-service | Shranjevanje dogodkov namesto stanja |
| [Circuit Breaker](#circuit-breaker) | parking-service | Toleranca napak pri zunanjih API-jih |
| [API Gateway](#api-gateway) | Kong | Centralna vstopna točka |
| [Multitenancy (RLS)](#multitenancy---row-level-security) | user, reservation, payment | Izolacija najemnikov |
| [gRPC](#grpc-komunikacija) | payment, reservation | Sinhrona binarna komunikacija |
| [Message Queue](#message-queue---rabbitmq) | payment, notification | Asinhrono sporočanje |

---

## CQRS - Command Query Responsibility Segregation

![CQRS/Event Sourcing diagram](https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/sample-unwind/docs/main/diagrami/cqrs-event-sourcing.puml)

### Opis

CQRS (Command Query Responsibility Segregation) je vzorec, ki loči operacije branja (queries) od operacij pisanja (commands). V Parkori je CQRS implementiran v `reservation-service`.

### Implementacija

**Command Side (Write Model):**
- Sprejema GraphQL mutations (`createReservation`, `cancelReservation`, ...)
- Validira poslovna pravila
- Zapisuje dogodke v Event Store
- Projicira dogodke v Read Model

**Query Side (Read Model):**
- Sprejema GraphQL queries (`reservations`, `reservationById`, ...)
- Bere podatke iz optimizirane `reservations` tabele
- Hitro in učinkovito branje brez kompleksne logike

### Prednosti

| Prednost | Opis |
|----------|------|
| **Skalabilnost** | Read in write modeli se lahko skalirajo neodvisno |
| **Fleksibilnost** | Query model optimiziran za specifične poizvedbe |
| **Preprostost** | Vsaka stran ima fokusirano odgovornost |
| **Revizijska sled** | Vsi ukazi (commands) so zabeleženi |

### Koda

```python
# Command - GraphQL Mutation
@strawberry.mutation
def create_reservation(self, info, input: ReservationInput) -> Reservation:
    # 1. Validate business rules
    # 2. Create aggregate
    aggregate = ReservationAggregate(...)
    
    # 3. Append event to Event Store
    event = event_store.append(
        aggregate_id=aggregate.id,
        event_type=EventType.RESERVATION_CREATED,
        data=aggregate.to_event_data()
    )
    
    # 4. Project to Read Model
    projector.apply_event(event)
    
    return aggregate.to_graphql()

# Query - GraphQL Query
@strawberry.field
def reservations(self, info, limit: int = 10) -> list[Reservation]:
    # Direct read from optimized table
    return db.query(ReservationModel).limit(limit).all()
```

---

## Event Sourcing

### Opis

Event Sourcing je vzorec, kjer se namesto trenutnega stanja shranjujejo vsi dogodki (events), ki so privedli do tega stanja. Stanje se rekonstruira z replay-anjem dogodkov.

### Implementacija v reservation-service

**Event Store tabela:**
```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,        -- ID rezervacije
    aggregate_type VARCHAR(100),       -- 'Reservation'
    event_type VARCHAR(100) NOT NULL,  -- Tip dogodka
    version INTEGER NOT NULL,          -- Verzija za optimistic locking
    data JSONB NOT NULL,               -- Podatki dogodka
    event_metadata JSONB,              -- Dodatni metapodatki
    tenant_id UUID NOT NULL,           -- Multitenancy
    created_at TIMESTAMPTZ NOT NULL    -- Časovna značka
);
```

**Event tipi:**

| Event | Opis | Sprožilec |
|-------|------|-----------|
| `RESERVATION_CREATED` | Nova rezervacija | `createReservation` mutation |
| `RESERVATION_CONFIRMED` | Potrditev | `confirmReservation` mutation |
| `RESERVATION_CANCELLED` | Preklic | `cancelReservation` mutation |
| `RESERVATION_COMPLETED` | Zaključek | `completeReservation` mutation |
| `PAYMENT_PROCESSED` | Uspešno plačilo | `payReservation` mutation |
| `PAYMENT_FAILED` | Neuspešno plačilo | gRPC response |

### Rekonstrukcija stanja

```python
class EventStore:
    def load_aggregate(self, aggregate_id: UUID) -> ReservationAggregate:
        """Rekonstruiraj stanje iz dogodkov."""
        events = self.get_events(aggregate_id)
        
        if not events:
            return None
            
        aggregate = ReservationAggregate.empty()
        for event in events:
            aggregate.apply(event)
            
        return aggregate
```

### Projekcija (Read Model)

```python
class ReservationProjector:
    def apply_event(self, event: Event) -> None:
        """Posodobi read model glede na event."""
        handlers = {
            EventType.RESERVATION_CREATED: self._handle_created,
            EventType.RESERVATION_CANCELLED: self._handle_cancelled,
            EventType.PAYMENT_PROCESSED: self._handle_payment,
        }
        handler = handlers.get(event.event_type)
        if handler:
            handler(event)
    
    def rebuild_from_events(self, tenant_id: UUID) -> int:
        """Ponovno zgradi celoten read model iz eventov."""
        # Počisti obstoječe podatke
        db.query(ReservationModel).filter_by(tenant_id=tenant_id).delete()
        
        # Replay vseh eventov
        events = event_store.get_all_events(tenant_id)
        for event in events:
            self.apply_event(event)
            
        return len(events)
```

### Admin API za rebuild

```bash
# Statistika event stora
curl https://parkora.crn.si/api/v1/reservation/admin/events/stats \
  -H "X-Tenant-ID: 00000000-0000-0000-0000-000000000001"

# Rebuild read modela
curl -X POST https://parkora.crn.si/api/v1/reservation/admin/rebuild \
  -H "X-Tenant-ID: 00000000-0000-0000-0000-000000000001"
```

### Prednosti Event Sourcing

| Prednost | Opis |
|----------|------|
| **Popolna revizijska sled** | Vsaka sprememba je zabeležena |
| **Time travel** | Možnost rekonstrukcije stanja na poljuben čas |
| **Debugging** | Replay eventov za razumevanje problema |
| **Recovery** | Rebuild read modela če je poškodovan |
| **Analytics** | Mining zgodovinskih podatkov |

---

## Circuit Breaker

### Opis

Circuit Breaker je vzorec za toleranco napak pri klicih zunanjih storitev. Preprečuje kaskadno odpoved sistema, ko zunanja storitev ni dosegljiva.

### Implementacija v parking-service

Parking-service uporablja Circuit Breaker pri klicu OpenWeatherMap API za vremenske podatke.

**Stanja:**

```
                    ┌─────────┐
                    │ CLOSED  │ ◄─── Normalno delovanje
                    └────┬────┘
                         │
            3 zaporedne napake
                         │
                         ▼
                    ┌─────────┐
                    │  OPEN   │ ◄─── Vrača fallback, ne kliče API
                    └────┬────┘
                         │
                    30 sekund
                         │
                         ▼
                    ┌──────────┐
                    │HALF-OPEN │ ◄─── Poskusni klic
                    └────┬─────┘
                         │
            ┌────────────┴────────────┐
            │                         │
         Uspeh                     Napaka
            │                         │
            ▼                         ▼
       ┌─────────┐              ┌─────────┐
       │ CLOSED  │              │  OPEN   │
       └─────────┘              └─────────┘
```

### Go implementacija

```go
import "github.com/sony/gobreaker"

var weatherBreaker *gobreaker.CircuitBreaker

func init() {
    settings := gobreaker.Settings{
        Name:        "weather-api",
        MaxRequests: 1,                    // Zahtev v half-open stanju
        Interval:    0,                    // Ne resetiraj v closed stanju
        Timeout:     30 * time.Second,     // Čas v open stanju
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures >= 3  // Odpri po 3 napakah
        },
    }
    weatherBreaker = gobreaker.NewCircuitBreaker(settings)
}

func getWeather() (*WeatherResponse, error) {
    result, err := weatherBreaker.Execute(func() (interface{}, error) {
        // Klic OpenWeatherMap API
        return callWeatherAPI()
    })
    
    if err != nil {
        // Vrni fallback
        return &WeatherResponse{
            Available: false,
            Message:   "Weather data temporarily unavailable",
        }, nil
    }
    
    return result.(*WeatherResponse), nil
}
```

### Response header

Odgovor vsebuje header `X-Circuit-Breaker-State` z trenutnim stanjem:

```bash
curl -I https://parkora.crn.si/api/v1/parking/weather

# Response headers:
# X-Circuit-Breaker-State: closed
# ali
# X-Circuit-Breaker-State: open
```

### Fallback response

Ko je circuit odprt:

```json
{
    "available": false,
    "message": "Weather data temporarily unavailable"
}
```

---

## API Gateway

![Komunikacija med servisi](https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/sample-unwind/docs/main/diagrami/komunikacija.puml)

### Opis

API Gateway je centralna vstopna točka za vse zahteve v sistem. V Parkori to vlogo opravlja **Kong Ingress Controller**.

### Funkcionalnosti

| Funkcionalnost | Implementacija |
|----------------|----------------|
| **Routing** | Path-based routing do mikrostoritev |
| **Load Balancing** | Porazdelitev prometa med replike |
| **TLS Termination** | HTTPS z Let's Encrypt certifikati |
| **Rate Limiting** | Omejitev zahtevkov na IP |
| **Path Stripping** | Odstranitev `/api/v1/<service>` predpone |

### Kong konfiguracija

**Ingress z rate limiting:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: parking-service
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: "rate-limit-api"
spec:
  ingressClassName: kong
  rules:
  - host: parkora.crn.si
    http:
      paths:
      - path: /api/v1/parking
        pathType: Prefix
        backend:
          service:
            name: parking-service
            port:
              number: 80
```

**Rate Limiting plugin:**
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-api
spec:
  plugin: rate-limiting
  config:
    minute: 20
    hour: 200
    limit_by: ip
    policy: local
```

### Routing tabela

| Path | Servis | Rate Limit |
|------|--------|------------|
| `/` | frontend | 100/min |
| `/api/v1/user` | user-service | 20/min |
| `/api/v1/parking` | parking-service | 20/min |
| `/api/v1/reservation` | reservation-service | 20/min |
| `/api/v1/payment` | payment-service | 20/min |

---

## Multitenancy - Row-Level Security

### Opis

Multitenancy omogoča, da več najemnikov (tenants) uporablja isto aplikacijo z zagotovljeno izolacijo podatkov. Parkora uporablja PostgreSQL Row-Level Security (RLS) za izolacijo na nivoju baze.

### Implementacija

**1. Stolpec `tenant_id` v vsaki tabeli:**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    -- ...
    UNIQUE (tenant_id, email)  -- Unikatnost per-tenant
);
```

**2. RLS politika:**
```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    FOR ALL
    USING (
        CASE 
            WHEN COALESCE(current_setting('app.tenant_id', true), '') = '' 
            THEN false
            ELSE tenant_id = current_setting('app.tenant_id', true)::UUID
        END
    );
```

**3. Nastavitev tenant_id v API-ju:**
```python
def set_tenant_id(db: Session, tenant_id: str) -> None:
    """Nastavi tenant_id za trenutno sejo."""
    db.execute(text(f"SET app.tenant_id = '{tenant_id}'"))

def get_context(request: Request, db: Session = Depends(get_db)):
    tenant_id = request.headers.get("x-tenant-id", DEFAULT_TENANT_ID)
    set_tenant_id(db, tenant_id)
    return {"db": db, "tenant_id": tenant_id}
```

### Uporaba

```bash
# Z tenant headerjem - vrne podatke za ta tenant
curl -X POST https://parkora.crn.si/api/v1/user/graphql \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: 00000000-0000-0000-0000-000000000001" \
  -d '{"query": "{ users { id email } }"}'

# Brez headerja - vrne prazen rezultat (strict mode)
curl -X POST https://parkora.crn.si/api/v1/user/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ users { id email } }"}'
```

### Prednosti RLS

| Prednost | Opis |
|----------|------|
| **Defense in depth** | Izolacija tudi ob aplikacijskih bugih |
| **GDPR compliance** | Stroga ločitev podatkov najemnikov |
| **Preprostost** | Ni potrebe po ločenih bazah/shemah |
| **Transparentnost** | Ista koda za vse najemnike |

---

## gRPC komunikacija

### Opis

gRPC je visoko-zmogljiv RPC framework za sinhrono komunikacijo med servisi. Parkora ga uporablja za komunikacijo med `reservation-service` in `payment-service`.

### Proto definicija

```protobuf
// payment.proto
syntax = "proto3";

package payment;

service PaymentService {
    rpc ProcessPayment(PaymentRequest) returns (PaymentResponse);
}

message PaymentRequest {
    string reservation_id = 1;
    string user_id = 2;
    double amount = 3;
    string currency = 4;
}

message PaymentResponse {
    bool success = 1;
    string transaction_id = 2;
    string message = 3;
}
```

### Python Server (payment-service)

```python
import grpc
from concurrent import futures
import payment_pb2
import payment_pb2_grpc

class PaymentServicer(payment_pb2_grpc.PaymentServiceServicer):
    def ProcessPayment(self, request, context):
        # Procesiraj plačilo
        transaction_id = process_payment(
            reservation_id=request.reservation_id,
            amount=request.amount
        )
        
        # Objavi event v RabbitMQ
        publish_payment_event(transaction_id)
        
        return payment_pb2.PaymentResponse(
            success=True,
            transaction_id=str(transaction_id),
            message="Payment processed successfully"
        )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    payment_pb2_grpc.add_PaymentServiceServicer_to_server(
        PaymentServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
```

### Python Client (reservation-service)

```python
import grpc
import payment_pb2
import payment_pb2_grpc

class PaymentClient:
    def __init__(self, host: str = "payment-service:50051"):
        self.channel = grpc.insecure_channel(host)
        self.stub = payment_pb2_grpc.PaymentServiceStub(self.channel)
    
    def process_payment(self, reservation_id: str, amount: float) -> dict:
        request = payment_pb2.PaymentRequest(
            reservation_id=reservation_id,
            amount=amount,
            currency="EUR"
        )
        
        response = self.stub.ProcessPayment(request)
        
        return {
            "success": response.success,
            "transaction_id": response.transaction_id,
            "message": response.message
        }
```

### Prednosti gRPC

| Prednost | Opis |
|----------|------|
| **Hitrost** | Binarni protokol (Protocol Buffers) |
| **Type safety** | Strogi tipi definirani v .proto |
| **Streaming** | Podpora za streaming (bidi-directional) |
| **Code generation** | Avtomatska generacija client/server kode |

---

## Message Queue - RabbitMQ

### Opis

RabbitMQ je message broker za asinhrono komunikacijo med servisi. Parkora ga uporablja za obveščanje po uspešnem plačilu.

### Arhitektura

```
┌─────────────────┐        ┌───────────────────┐        ┌────────────────────┐
│ payment-service │───────▶│     RabbitMQ      │───────▶│notification-service│
│   (Publisher)   │        │                   │        │    (Consumer)      │
└─────────────────┘        │  Exchange:        │        └────────────────────┘
                           │  parkora_events   │                  │
                           │  (topic)          │                  │
                           │                   │                  ▼
                           │  Queue:           │           ┌────────────┐
                           │  notification_    │           │  ntfy.sh   │
                           │  queue            │           │   Push     │
                           └───────────────────┘           └────────────┘
```

### Publisher (payment-service)

```python
import pika
import json

def publish_payment_event(transaction_id: str, reservation_id: str):
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='rabbitmq')
    )
    channel = connection.channel()
    
    # Declare exchange
    channel.exchange_declare(
        exchange='parkora_events',
        exchange_type='topic',
        durable=True
    )
    
    # Publish message
    message = {
        "event_type": "payment.processed",
        "transaction_id": transaction_id,
        "reservation_id": reservation_id,
        "timestamp": datetime.utcnow().isoformat()
    }
    
    channel.basic_publish(
        exchange='parkora_events',
        routing_key='payment.processed',
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent
            content_type='application/json'
        )
    )
    
    connection.close()
```

### Consumer (notification-service)

```python
import pika
import requests

def callback(ch, method, properties, body):
    message = json.loads(body)
    
    # Pošlji push notification preko ntfy.sh
    requests.post(
        "https://ntfy.sh/parkora-notification-service",
        data=f"Payment processed: {message['transaction_id']}",
        headers={"Title": "Parkora Payment"}
    )
    
    # Acknowledge message
    ch.basic_ack(delivery_tag=method.delivery_tag)

def consume():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='rabbitmq')
    )
    channel = connection.channel()
    
    # Declare queue
    channel.queue_declare(queue='notification_queue', durable=True)
    
    # Bind to exchange
    channel.queue_bind(
        exchange='parkora_events',
        queue='notification_queue',
        routing_key='payment.processed'
    )
    
    # Start consuming
    channel.basic_consume(
        queue='notification_queue',
        on_message_callback=callback
    )
    
    channel.start_consuming()
```

### Prednosti asinhrone komunikacije

| Prednost | Opis |
|----------|------|
| **Decoupling** | Servisi niso odvisni eden od drugega |
| **Reliability** | Sporočila so trajno shranjena |
| **Scalability** | Več consumerjev lahko procesira sporočila |
| **Resilience** | Sistem deluje tudi ko je consumer nedosegljiv |

---

## Povzetek

Parkora implementira več naprednih arhitekturnih vzorcev, ki skupaj zagotavljajo:

- **Zanesljivost**: Circuit Breaker, Message Queue
- **Skalabilnost**: CQRS, API Gateway, Microservices
- **Varnost**: Multitenancy z RLS, JWT validacija
- **Sledljivost**: Event Sourcing, centralizirano beleženje
- **Učinkovitost**: gRPC, optimiziran read model

Ti vzorci so del zahtev predmeta RSO in demonstrirajo razumevanje sodobnih pristopov k razvoju oblačnih aplikacij.
