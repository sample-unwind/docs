# Namestitev sistema

Ta dokument vsebuje navodila za namestitev in deployment sistema Parkora na Azure Kubernetes Service (AKS).

## Predpogoji

Pred začetkom namestitve potrebujete naslednja orodja:

| Orodje | Namen | Namestitev |
|--------|-------|------------|
| **Azure CLI** | Upravljanje Azure virov | [Navodila](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) |
| **kubectl** | Upravljanje Kubernetes | [Navodila](https://kubernetes.io/docs/tasks/tools/) |
| **Helm** | Kubernetes package manager | [Navodila](https://helm.sh/docs/intro/install/) |
| **Terraform** | Infrastructure as Code | [Navodila](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) |
| **GitHub CLI** | GitHub operacije | [Navodila](https://cli.github.com/) |

## Omejitve Azure Student Subscription

> **Pomembno:** Azure Student subscription ne omogoča ustvarjanja Service Principalov. To pomeni, da:
> - Terraform se izvaja **ročno, lokalno** (ne preko GitHub Actions)
> - Helm deployment se izvaja **ročno, lokalno**
> - Microservice CI/CD (build, test, push images) deluje normalno preko GitHub Actions

---

## 1. Prijava v Azure

```bash
# Prijava v Azure
az login

# Preveri aktivno subscription
az account show

# Nastavi subscription (če imate več)
az account set --subscription "Azure for Students"
```

---

## 2. Terraform - Infrastruktura

Terraform konfiguracija je v repozitoriju `core_infra`.

### 2.1 Kloniranje repozitorija

```bash
git clone https://github.com/sample-unwind/core_infra.git
cd core_infra
```

### 2.2 Inicializacija Terraform

```bash
# Inicializacija (poveže se na Azure Storage backend za state)
terraform init
```

**Backend konfiguracija** (že nastavljena v `main.tf`):
- Resource Group: `rg-parkora-tfstate`
- Storage Account: `stparkoratfstate`
- Container: `tfstate`

### 2.3 Pregled sprememb

```bash
# Pregled kaj bo ustvarjeno/spremenjeno
terraform plan
```

**Pomembno:** Vedno preglej output `terraform plan` preden izvedeš `apply`!

### 2.4 Ustvarjanje infrastrukture

```bash
# Ustvari vse Azure vire
terraform apply
```

Po uspešnem `apply` boste imeli:
- Resource Group: `rg-parkora`
- AKS Cluster: `aks-parkora`
- Key Vault: `kv-parkora`

### 2.5 Pridobitev AKS credentials

```bash
# Prenesi kubeconfig za AKS
az aks get-credentials --resource-group rg-parkora --name aks-parkora

# Preveri povezavo
kubectl get nodes
```

---

## 3. Kubernetes - Namespaces

Ustvari potrebne namespaces:

```bash
# Parkora namespace (mikrostoritve)
kubectl create namespace parkora

# Keycloak namespace (avtentikacija)
kubectl create namespace keycloak
```

Kong in cert-manager namespaces se ustvarijo avtomatsko ob Helm instalaciji.

---

## 4. Kong Ingress Controller

```bash
# Dodaj Kong Helm repo
helm repo add kong https://charts.konghq.com
helm repo update

# Namesti Kong
helm install kong kong/ingress -n kong --create-namespace

# Preveri namestitev
kubectl get pods -n kong
kubectl get svc -n kong
```

Po namestitvi dobite LoadBalancer IP (npr. `72.144.69.17`). Nastavite DNS A zapis za `parkora.crn.si` na ta IP.

---

## 5. cert-manager (TLS certifikati)

```bash
# Dodaj Jetstack Helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Namesti cert-manager
helm install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Preveri namestitev
kubectl get pods -n cert-manager
```

### Let's Encrypt ClusterIssuer

Ustvari ClusterIssuer za Let's Encrypt:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-http01-key
    solvers:
    - http01:
        ingress:
          class: kong
EOF
```

---

## 6. Keycloak

### 6.1 PostgreSQL za Keycloak

```bash
kubectl apply -f core_infra/k8s/keycloak/postgresql.yaml
```

### 6.2 Keycloak Helm instalacija

```bash
# Dodaj Codecentric Helm repo
helm repo add codecentric https://codecentric.github.io/helm-charts
helm repo update

# Namesti Keycloak
helm install keycloak codecentric/keycloakx \
  -n keycloak \
  -f core_infra/k8s/keycloak/values.yaml
```

### 6.3 Keycloak konfiguracija

Po namestitvi odpri Admin Console (https://keycloak.parkora.crn.si/auth/):

1. **Ustvari realm** `parkora`
2. **Ustvari clienta** `frontend-app`:
   - Access Type: `public`
   - Valid Redirect URIs: `https://parkora.crn.si/*`
   - Web Origins: `https://parkora.crn.si`
3. **Ustvari clienta** `backend-services`:
   - Access Type: `bearer-only`
4. **Ustvari testne uporabnike**:
   - `testuser1` / `password123`
   - `testuser2` / `password123`

---

## 7. Azure Key Vault - Skrivnosti

### 7.1 Dodajanje skrivnosti

```bash
# Supabase credentials za parking-service
az keyvault secret set --vault-name kv-parkora \
  --name "parking-service-supabase-url" \
  --value "https://your-project.supabase.co/"

az keyvault secret set --vault-name kv-parkora \
  --name "parking-service-supabase-key" \
  --value "your-service-role-key"

# OpenWeatherMap API key
az keyvault secret set --vault-name kv-parkora \
  --name "parking-service-openweathermap-key" \
  --value "your-api-key"

# RabbitMQ password
az keyvault secret set --vault-name kv-parkora \
  --name "rabbitmq-password" \
  --value "your-password"
```

### 7.2 Kubernetes secrets

Ustvari K8s secrets iz Key Vault vrednosti:

```bash
# Parking service
SUPABASE_URL=$(az keyvault secret show --vault-name kv-parkora --name "parking-service-supabase-url" --query value -o tsv)
SUPABASE_KEY=$(az keyvault secret show --vault-name kv-parkora --name "parking-service-supabase-key" --query value -o tsv)
OWM_KEY=$(az keyvault secret show --vault-name kv-parkora --name "parking-service-openweathermap-key" --query value -o tsv)

kubectl create secret generic parking-service-secrets \
  --from-literal=SUPABASE_URL="$SUPABASE_URL" \
  --from-literal=SUPABASE_SERVICE_ROLE_KEY="$SUPABASE_KEY" \
  --from-literal=OPENWEATHERMAP_API_KEY="$OWM_KEY" \
  -n parkora --dry-run=client -o yaml | kubectl apply -f -
```

---

## 8. GHCR Image Pull Secret

Za privatne repozitorije ustvarite image pull secret:

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<github-token> \
  -n parkora
```

---

## 9. RabbitMQ

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
  namespace: parkora
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3-management-alpine
        ports:
        - containerPort: 5672
        - containerPort: 15672
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  namespace: parkora
spec:
  selector:
    app: rabbitmq
  ports:
  - name: amqp
    port: 5672
  - name: management
    port: 15672
EOF
```

---

## 10. Mikrostoritve - Helm Deployment

### 10.1 Frontend

```bash
cd frontend

helm upgrade --install frontend ./helm/frontend \
  --namespace parkora \
  --set image.tag=main
```

### 10.2 Parking Service

```bash
cd microservices/parking-service

helm upgrade --install parking-service ./helm/parking-service \
  --namespace parkora \
  --set image.tag=master
```

### 10.3 User Service

```bash
cd microservices/user-service

helm upgrade --install user-service ./helm/user-service \
  --namespace parkora \
  --set image.tag=master
```

### 10.4 Reservation Service

```bash
cd microservices/reservation-service

helm upgrade --install reservation-service ./helm/reservation-service \
  --namespace parkora \
  --set image.tag=master
```

### 10.5 Payment Service

```bash
cd microservices/payment-service

helm upgrade --install payment-service ./helm/payment-service \
  --namespace parkora \
  --set image.tag=master
```

### 10.6 Notification Service

```bash
cd microservices/notification-service

helm upgrade --install notification-service ./helm/notification-service \
  --namespace parkora \
  --set image.tag=master
```

---

## 11. Kong Rate Limiting

Apliciraj rate limiting plugine:

```bash
kubectl apply -f core_infra/k8s/kong/rate-limit-ui.yaml
kubectl apply -f core_infra/k8s/kong/rate-limit-api.yaml
kubectl apply -f core_infra/k8s/kong/rate-limit-auth.yaml
```

---

## 12. Monitoring Stack

### 12.1 Prometheus + Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n parkora
```

### 12.2 Elasticsearch

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch elastic/elasticsearch \
  -n parkora \
  --set replicas=1 \
  --set minimumMasterNodes=1
```

### 12.3 Kibana

```bash
kubectl apply -f core_infra/k8s/monitoring/kibana-deployment.yaml
kubectl apply -f core_infra/k8s/monitoring/kibana-ingress.yaml
```

### 12.4 Filebeat

```bash
helm install filebeat elastic/filebeat \
  -n parkora \
  --set daemonset.enabled=true
```

---

## 13. Preverjanje namestitve

### Preveri pode

```bash
kubectl get pods -n parkora
kubectl get pods -n keycloak
kubectl get pods -n kong
kubectl get pods -n cert-manager
```

### Preveri servise

```bash
kubectl get svc -n parkora
kubectl get ingress -A
```

### Preveri certifikate

```bash
kubectl get certificate -A
```

### Testni klici

```bash
# Health check
curl https://parkora.crn.si/api/v1/parking/health/live

# Parkirišča
curl https://parkora.crn.si/api/v1/parking/analytics/parkings

# GraphQL
curl -X POST https://parkora.crn.si/api/v1/user/graphql \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: 00000000-0000-0000-0000-000000000001" \
  -d '{"query": "{ users { id email } }"}'
```

---

## 14. Posodabljanje mikrostoritev

### Ročni update

```bash
# Restart deployment (pull new image)
kubectl rollout restart deployment/frontend -n parkora
kubectl rollout restart deployment/parking-service -n parkora

# Ali z Helm
helm upgrade frontend ./helm/frontend -n parkora --set image.tag=main
```

### Avtomatski update (CronJob)

Parkora ima nastavljen CronJob za avtomatsko posodabljanje:

```bash
kubectl apply -f core_infra/k8s/image-updater/cronjob.yaml
```

CronJob vsaki 2 minuti restarta deploymente z labelom `auto-update: enabled`.

---

## 15. Azure Function - Scraper

Serverless funkcija za periodično pobiranje podatkov o parkirni zasedenosti.

### 15.1 Predpogoji

```bash
# Namesti Azure Functions Core Tools
npm install -g azure-functions-core-tools@4 --unsafe-perm true

# Preveri namestitev
func --version
```

### 15.2 Ustvarjanje Function App

```bash
# Ustvari Function App (consumption plan - pay per execution)
az functionapp create \
  --resource-group rg-parkora \
  --consumption-plan-location germanywestcentral \
  --runtime node \
  --runtime-version 20 \
  --functions-version 4 \
  --name func-parkora-scraper \
  --storage-account stparkoratfstate

# Nastavi environment variables iz Key Vault
az functionapp config appsettings set \
  --resource-group rg-parkora \
  --name func-parkora-scraper \
  --settings \
    SUPABASE_URL="$(az keyvault secret show --vault-name kv-parkora --name parking-service-supabase-url --query value -o tsv)" \
    SUPABASE_SERVICE_ROLE_KEY="$(az keyvault secret show --vault-name kv-parkora --name parking-service-supabase-key --query value -o tsv)"
```

### 15.3 Struktura projekta

```
lptscraper/
├── ScraperFunction/
│   ├── function.json      # Timer trigger konfiguracija
│   └── index.js           # Funkcija logika
├── host.json              # Azure Functions host konfiguracija
├── package.json           # Node.js dependencies
└── local.settings.json    # Lokalne nastavitve (ni v git)
```

**`ScraperFunction/function.json`:**
```json
{
  "bindings": [
    {
      "name": "myTimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 */10 * * * *"
    }
  ]
}
```

**`host.json`:**
```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

### 15.4 Deployment

```bash
cd lptscraper

# Inicializiraj Azure Function projekt (če še ni)
func init --worker-runtime node --language javascript

# Ustvari funkcijo
func new --name ScraperFunction --template "Timer trigger"

# Deploy funkcije v Azure
func azure functionapp publish func-parkora-scraper
```

### 15.5 Preverjanje

```bash
# Preveri status funkcije
az functionapp show \
  --name func-parkora-scraper \
  --resource-group rg-parkora \
  --query state -o tsv

# Preveri loge v real-time
az functionapp log tail \
  --name func-parkora-scraper \
  --resource-group rg-parkora

# Ročno proži funkcijo (za testiranje)
az functionapp function invoke \
  --resource-group rg-parkora \
  --name func-parkora-scraper \
  --function-name ScraperFunction

# Preveri zadnje izvajanje
az monitor activity-log list \
  --resource-group rg-parkora \
  --caller "func-parkora-scraper" \
  --max-events 5
```

### 15.6 Monitoring

Funkcija avtomatsko pošilja podatke v Application Insights:

```bash
# Omogoči Application Insights
az functionapp update \
  --name func-parkora-scraper \
  --resource-group rg-parkora \
  --set siteConfig.appInsightsInstrumentationKey="<instrumentation-key>"
```

V Azure Portal:
1. Pojdi na Function App → `func-parkora-scraper`
2. Klikni **Monitor** → **Invocations**
3. Preglej zgodovino izvajanj in morebitne napake

---

## 16. Odpravljanje težav

### Pod ne starta

```bash
kubectl describe pod <pod-name> -n parkora
kubectl logs <pod-name> -n parkora
```

### Ingress ne deluje

```bash
kubectl get ingress -A
kubectl describe ingress <ingress-name> -n parkora
kubectl logs -n kong -l app.kubernetes.io/name=kong
```

### Certifikat ni izdan

```bash
kubectl get certificate -A
kubectl describe certificate <cert-name> -n <namespace>
kubectl get challenges -A
kubectl logs -n cert-manager -l app=cert-manager
```

### Database connection

```bash
# Connect to PostgreSQL
kubectl exec -it -n keycloak <postgresql-pod> -- psql -U keycloak -d user_service

# Check RLS policies
\d+ users
SELECT * FROM pg_policies;
```

---

## Povzetek ukazov

| Akcija | Ukaz |
|--------|------|
| Azure prijava | `az login` |
| AKS credentials | `az aks get-credentials --resource-group rg-parkora --name aks-parkora` |
| Terraform apply | `cd core_infra && terraform apply` |
| Helm install | `helm upgrade --install <release> ./helm/<chart> -n parkora` |
| Restart deployment | `kubectl rollout restart deployment/<name> -n parkora` |
| View logs | `kubectl logs -n parkora deployment/<name>` |
| List pods | `kubectl get pods -n parkora` |
| Deploy Azure Function | `func azure functionapp publish func-parkora-scraper` |
| View Function logs | `az functionapp log tail --name func-parkora-scraper --resource-group rg-parkora` |
