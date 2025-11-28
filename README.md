# digital-bubble-api
# Digital Bubble Tenant API

API REST per gestire tenant DigitalBubble tramite ArgoCD su cluster Kubernetes.

## Caratteristiche

- ✅ Creazione automatica tenant con ArgoCD Application
- ✅ Generazione automatica password PostgreSQL e Redis (10 caratteri alfanumerici)
- ✅ Gestione Secret Kubernetes per le credenziali
- ✅ Autenticazione JWT (token statico configurabile)
- ✅ CRUD completo su tenant (Create, Read, Update, Delete, List)
- ✅ Validazione input con Pydantic
- ✅ Logging strutturato
- ✅ Containerizzato con Docker

## Architettura

```
[Frontend] --HTTP--> [REST API] --crea direttamente--> [ArgoCD Application]
                                                              |
                                                              v
                                                          [ArgoCD]
                                                              |
                                                              v
                                                    [Deploy Helm Chart DigitalBubble]
```

**Flusso Operativo:**
1. Frontend invia richiesta HTTP all'API con `tenantName` e `apiKey`
2. API genera password PostgreSQL/Redis casuali
3. API crea Secret Kubernetes con le password
4. API crea ArgoCD Application con valori Helm completi
5. ArgoCD sincronizza e deploya l'Helm Chart sul cluster

## Prerequisiti

- Python 3.10+ (testato con Python 3.13.7)
- Accesso a cluster Kubernetes (locale o AKS)
- ArgoCD installato nel cluster (namespace `argocd`)
- `kubectl` configurato
- Docker (opzionale, per esecuzione containerizzata)

## Installazione e Esecuzione

### Opzione 1: Esecuzione Python Locale (Sviluppo)

#### 1. Clona/prepara il progetto

```bash
cd digital-bubble-api
```

#### 2. Crea ambiente virtuale

```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
# oppure
venv\Scripts\activate  # Windows
```

#### 3. Installa dipendenze

```bash
pip install -r requirements.txt
```

#### 4. Configura variabili d'ambiente

```bash
cp .env.example .env
# Modifica .env con i tuoi valori se necessario
```

#### 5. Verifica connessione Kubernetes

```bash
kubectl get nodes
kubectl get ns argocd
```

#### 6. Avvia l'API

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

L'API sarà disponibile su: `http://localhost:8000`

Documentazione interattiva: `http://localhost:8000/docs`

---

### Opzione 2: Esecuzione Docker su Cluster Kubernetes Locale

Per testare l'API containerizzata su un cluster Kubernetes locale (es. Docker Desktop, Minikube, Kind):

#### 1. Prepara il file kubeconfig

**IMPORTANTE:** L'API containerizzata necessita di accedere al kubeconfig per comunicare con il cluster.

```bash
# Copia il tuo kubeconfig nella directory del progetto
# Il file DEVE chiamarsi "kubeconfig" (senza estensione)
cp ~/.kube/config kubeconfig

# Verifica che il file esista
ls -la kubeconfig
```

⚠️ **Nota Sicurezza:** Il file `kubeconfig` è già incluso nel `.gitignore` per evitare commit accidentali.

#### 2. Build dell'immagine Docker

```bash
# Assicurati di essere nella directory del progetto
cd /path/to/digital-bubble-api

# Build dell'immagine
docker build -t tenant-api:1.2 .
```

#### 3. Run del container

```bash
# Monta il kubeconfig e esponi la porta 8000
docker run -it \
  -v ${PWD}/kubeconfig:/root/.kube/config:ro \
  --network=host \
  -p 8000:8000 \
  tenant-api:1.2
```

**Parametri spiegati:**
- `-v ${PWD}/kubeconfig:/root/.kube/config:ro`: Monta il kubeconfig in read-only
- `--network=host`: Permette al container di accedere al cluster locale
- `-p 8000:8000`: Espone la porta 8000
- `:ro`: Read-only mount per sicurezza

#### 4. Test del container

```bash
# Health check
curl http://localhost:8000/health

# Verifica documentazione API
open http://localhost:8000/docs  # Mac
# oppure
xdg-open http://localhost:8000/docs  # Linux
```

**Esempio completo (Mac):**
```bash
# Dalla directory del progetto
cd /Users/enricomostacci/Documents/code/Plansol/digitalbubble/digital-bubble-admin-feature-k8s-operator/digital-bubble-api

# Copia kubeconfig
cp ~/.kube/config kubeconfig

# Build immagine
docker build -t tenant-api:1.2 .

# Run container
docker run -it -v ${PWD}/kubeconfig:/root/.kube/config:ro --network=host -p 8000:8000 tenant-api:1.2
```

## Utilizzo API

### Autenticazione

Tutte le richieste (tranne `/health`) richiedono il token Bearer nell'header:

```bash
Authorization: Bearer digital-bubble-static-token-2025
```

Il token di default può essere modificato nel file `.env`:
```
API_STATIC_TOKEN=il-tuo-token-personalizzato
```

### Struttura Tenant

Ogni tenant creato genera automaticamente:

1. **ArgoCD Application** (namespace: `argocd`)
   - Nome: `{tenantName}`
   - Progetto: `dal-bubble-admin`
   - Repository: `https://github.com/Rinascitech/StudioHype`
   - Path: `helm/app`

2. **Namespace Kubernetes**: `{tenantName}`

3. **Secret Kubernetes**: `tenant-secrets-{tenantName}`
   - `postgres_password`: Password PostgreSQL (10 caratteri)
   - `redis_password`: Password Redis (10 caratteri)

4. **Valori Helm configurati automaticamente:**
   - Ingress: `{tenantName}.digitalbubble.prod.flow.digitalbubble.it`
   - TLS Secret: `{tenantName}-tls`
   - Environment variables (APP_NAME, APP_URL, ecc.)
   - Database credentials (auto-generate)
   - API_KEY (fornita dal client)

### Endpoint API

#### 1. Health Check (senza autenticazione)

```bash
curl http://localhost:8000/health
```

**Risposta:**
```json
{
  "status": "healthy",
  "service": "digital-bubble-api",
  "version": "1.0.0"
}
```

---

#### 2. CREATE - Crea Nuovo Tenant

```bash
curl -X POST http://localhost:8000/api/v1/tenants \
  -H "Authorization: Bearer digital-bubble-static-token-2025" \
  -H "Content-Type: application/json" \
  -d '{
    "tenantName": "studio-esempio",
    "apiKey": "my-api-key-12345"
  }'
```

**Parametri:**
- `tenantName` (obbligatorio): Nome univoco tenant, solo lowercase/numeri/trattini
- `apiKey` (obbligatorio): API Key da inserire in `secrets.API_KEY`

**Risposta (201 Created):**
```json
{
  "name": "studio-esempio",
  "namespace": "studio-esempio",
  "host": "studio-esempio.digitalbubble.prod.flow.digitalbubble.it",
  "repoURL": "https://github.com/Rinascitech/StudioHype",
  "status": "Created",
  "syncStatus": null,
  "healthStatus": null
}
```

**Cosa viene creato:**
- ✅ ArgoCD Application
- ✅ Secret con password PostgreSQL e Redis
- ✅ Namespace (via ArgoCD syncOption CreateNamespace=true)

---

#### 3. READ - Lista Tutti i Tenant

```bash
curl -X GET http://localhost:8000/api/v1/tenants \
  -H "Authorization: Bearer digital-bubble-static-token-2025"
```

**Risposta (200 OK):**
```json
{
  "tenants": [
    {
      "name": "studio-esempio",
      "namespace": "studio-esempio",
      "host": "studio-esempio.digitalbubble.prod.flow.digitalbubble.it",
      "repoURL": "https://github.com/Rinascitech/StudioHype",
      "syncStatus": "Synced",
      "healthStatus": "Healthy"
    }
  ],
  "count": 1
}
```

---

#### 4. READ - Dettagli Singolo Tenant

```bash
curl -X GET http://localhost:8000/api/v1/tenants/studio-esempio \
  -H "Authorization: Bearer digital-bubble-static-token-2025"
```

**Risposta (200 OK):**
```json
{
  "name": "studio-esempio",
  "namespace": "studio-esempio",
  "host": "studio-esempio.digitalbubble.prod.flow.digitalbubble.it",
  "repoURL": "https://github.com/Rinascitech/StudioHype",
  "syncStatus": "Synced",
  "healthStatus": "Healthy"
}
```

**Errore se non trovato (404):**
```json
{
  "detail": "Tenant 'studio-esempio' non trovato"
}
```

---

#### 5. UPDATE - Aggiorna Tenant (API Key)

```bash
curl -X PUT http://localhost:8000/api/v1/tenants/studio-esempio \
  -H "Authorization: Bearer digital-bubble-static-token-2025" \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "nuova-api-key-67890"
  }'
```

**Parametri:**
- `apiKey` (opzionale): Nuova API Key

**Risposta (200 OK):**
```json
{
  "name": "studio-esempio",
  "namespace": "studio-esempio",
  "host": "studio-esempio.digitalbubble.prod.flow.digitalbubble.it",
  "repoURL": "https://github.com/Rinascitech/StudioHype",
  "status": "Updated"
}
```

**Note importanti:**
- ⚠️ Il `tenantName` **NON può essere modificato** (immutabile)
- ✅ Le password PostgreSQL/Redis vengono **mantenute** (non rigenerate)
- ✅ Solo i campi specificati vengono aggiornati

---

#### 6. DELETE - Cancella Tenant

```bash
curl -X DELETE http://localhost:8000/api/v1/tenants/studio-esempio \
  -H "Authorization: Bearer digital-bubble-static-token-2025"
```

**Risposta (204 No Content):**
```
(nessun body nella risposta)
```

**Cosa viene cancellato:**
1. ✅ ArgoCD Application `studio-esempio`
2. ✅ Secret `tenant-secrets-studio-esempio`
3. ✅ Namespace `studio-esempio` (con tutte le risorse al suo interno)

⚠️ **Attenzione:** Questa operazione è **irreversibile**!

---

### Test Completo - Ciclo di Vita Tenant

Sequenza completa per testare tutti gli endpoint:

```bash
# 1. Health check
curl http://localhost:8000/health

# 2. Crea tenant
curl -X POST http://localhost:8000/api/v1/tenants \
  -H "Authorization: Bearer digital-bubble-static-token-2025" \
  -H "Content-Type: application/json" \
  -d '{
    "tenantName": "studio-test",
    "apiKey": "prima-api-key-123"
  }'

# 3. Verifica che esista
curl -X GET http://localhost:8000/api/v1/tenants/studio-test \
  -H "Authorization: Bearer digital-bubble-static-token-2025"

# 4. Lista tutti i tenant
curl -X GET http://localhost:8000/api/v1/tenants \
  -H "Authorization: Bearer digital-bubble-static-token-2025"

# 5. Aggiorna API Key
curl -X PUT http://localhost:8000/api/v1/tenants/studio-test \
  -H "Authorization: Bearer digital-bubble-static-token-2025" \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "seconda-api-key-456"
  }'

# 6. Verifica le modifiche su Kubernetes
kubectl get application studio-test -n argocd -o yaml

# 7. Cancella tenant
curl -X DELETE http://localhost:8000/api/v1/tenants/studio-test \
  -H "Authorization: Bearer digital-bubble-static-token-2025"

# 8. Verifica cancellazione (dovrebbe dare 404)
curl -X GET http://localhost:8000/api/v1/tenants/studio-test \
  -H "Authorization: Bearer digital-bubble-static-token-2025"
```

## Containerizzazione e Deployment

### Dockerfile

Il progetto include un Dockerfile per containerizzare l'API:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Build dell'Immagine

```bash
docker build -t digital-bubble-api:latest .
```

### Test Locale con Docker

```bash
# Esegui il container (usa la tua kubeconfig locale)
docker run -p 8000:8000 \
  -v ~/.kube/config:/root/.kube/config \
  -e API_STATIC_TOKEN=your-token \
  digital-bubble-api:latest
```

### Deployment su Kubernetes

L'API è stata testata con successo su Docker Desktop locale con cluster Kubernetes integrato. 
Per il deployment su AKS di produzione, sono necessari i manifest YAML per:
- Deployment
- Service
- ServiceAccount + RBAC

(Questi manifest saranno aggiunti nella directory `k8s/` nei prossimi step)

## Logging

L'applicazione usa il modulo `logging` di Python. I log includono:

- Info su creazione/update/delete tenant
- Errori di autenticazione
- Problemi di connessione Kubernetes
- Exception non gestite

## Sicurezza

⚠️ **Importante per produzione:**

1. Cambia `API_STATIC_TOKEN` in `.env`
2. Cambia `JWT_SECRET_KEY` in `.env`
3. Configura CORS con domini specifici (non `*`)
4. Usa HTTPS
5. Implementa rate limiting
6. Si può pensare ad un'autenticazione più robusta (OAuth2, etc.)

## Test Effettuati

L'API è stata testata con successo in ambiente locale:

**Setup di Test:**
- Docker Desktop con Kubernetes integrato
- ArgoCD deployato nel cluster locale (namespace `argocd`)
- API containerizzata e in esecuzione nello stesso cluster
- Test eseguiti tramite chiamate CURL da terminale

**Test Completati:**

1. ✅ **CREATE Tenant**
   - Creazione ArgoCD Application
   - Generazione automatica password PostgreSQL e Redis
   - Creazione Secret Kubernetes con credenziali
   - Configurazione Helm values con ingress, env vars, secrets

2. ✅ **UPDATE Tenant**
   - Modifica API_KEY mantenendo password esistenti
   - Preservazione metadata.resourceVersion per aggiornamenti K8s
   - Merge corretto dei valori Helm

3. ✅ **DELETE Tenant**
   - Rimozione ArgoCD Application
   - Cancellazione Secret con password
   - Rimozione completa Namespace

4. ✅ **LIST Tenants**
   - Recupero di tutti i tenant gestiti dall'API
   - Estrazione status sync e health da ArgoCD

5. ✅ **GET Tenant Singolo**
   - Dettagli completi di un tenant specifico
   - Verifica 404 per tenant inesistenti

**Validazioni:**
- Autenticazione JWT con token statico
- Validazione input (tenantName: solo lowercase, numeri, trattini)
- Gestione errori con status code HTTP appropriati
- Logging strutturato di tutte le operazioni

### Errore: "Impossibile inizializzare Kubernetes client"

- Verifica che `kubectl` funzioni
- Controlla `~/.kube/config`

### Errore: "Tenant esiste già"

- Il nome tenant deve essere univoco
- Usa DELETE prima di ricreare

### Errore 401 su tutte le richieste

- Verifica l'header `Authorization: Bearer <token>`
- Token di default: `digital-bubble-static-token-2025`

## Contatti

enrico@newesis.com

## Verifica Risorse Kubernetes

Dopo aver creato un tenant, verifica le risorse su Kubernetes:

```bash
# Verifica ArgoCD Application
kubectl get application studio-esempio -n argocd

# Visualizza manifest completo
kubectl get application studio-esempio -n argocd -o yaml

# Verifica Secret con password
kubectl get secret tenant-secrets-studio-esempio -n studio-esempio

# Decodifica le password (per debug)
kubectl get secret tenant-secrets-studio-esempio -n studio-esempio -o jsonpath='{.data.postgres_password}' | base64 -d
kubectl get secret tenant-secrets-studio-esempio -n studio-esempio -o jsonpath='{.data.redis_password}' | base64 -d

# Verifica risorse nel namespace del tenant
kubectl get all -n studio-esempio
```


```

## FAQ

**Q: Posso modificare il nome di un tenant?**  
A: No, il `tenantName` è immutabile. Per cambiare nome, fai DELETE e CREATE (attenzione: perderai i dati).

**Q: Le password vengono rigenerate ad ogni UPDATE?**  
A: No, le password sono mantenute. Vengono generate solo alla creazione e salvate nel Secret.

**Q: Come espongo l'API all'esterno del cluster?**  
A: Serve un Ingress o LoadBalancer (non incluso in questa versione).

**Q: Supporta multi-tenancy?**  
A: Sì, ogni tenant ha il proprio namespace Kubernetes isolato.

**Q: Il file kubeconfig è sicuro nel progetto?**  
A: No, il file `kubeconfig` è già incluso nel `.gitignore` e NON deve essere committato. È solo per test locali.

**Q: Dove vengono salvate le password PostgreSQL/Redis?**  
A: In un Secret Kubernetes chiamato `tenant-secrets-{tenantName}` nel namespace del tenant.

**Q: Cosa succede se cancello il Secret con le password?**  
A: Al prossimo UPDATE del tenant, l'API tenterà di recuperare il Secret. Se non esiste, ne genererà uno nuovo con nuove password (attenzione: questo resetterà le credenziali del database!).
