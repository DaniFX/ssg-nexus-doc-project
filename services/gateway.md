# 🔄 SSG Gateway — Specifiche Tecniche

> **Repository:** [`ssg-gateway`](https://github.com/DaniFX/ssg-gateway)
> **Stack:** Go 1.21+, Gin, Firebase Admin SDK, Google OIDC (`idtoken`)
> **Ruolo:** Unico punto di ingresso pubblico dell’ecosistema SSG Nexus.

---

## 1. Responsabilità

Il Gateway svolge **quattro funzioni fondamentali** in sequenza per ogni richiesta in arrivo:

1. **Autenticazione** — Verifica il Firebase JWT e risolve il ruolo utente.
2. **Propagazione Nexus** — Inietta gli header standard (`X-Nexus-User-ID`, `X-Nexus-Role`, `X-Nexus-Trace-ID`) nella richiesta da inoltrare.
3. **Routing Dinamico** — Individua il microservizio target tramite tabella di routing caricata da Firestore.
4. **Proxy OIDC** — Ottiene un token OIDC (Google Identity) e lo usa come `Authorization: Bearer` verso il Cloud Run del microservizio.

> **Regola fondamentale**: Nessun microservizio accetta traffico diretto da internet. Tutto transita dal Gateway.

---

## 2. Struttura del Progetto

```
ssg-gateway/
├── cmd/                        # Entrypoint (main.go)
├── internal/
│   ├── config/                 # Config da env (GCP Project, Firebase, etc.)
│   ├── handlers/               # Handler HTTP (health, discovery push)
│   ├── middleware/
│   │   ├── auth.go             # FirebaseAuthMiddleware (JWT + Role)
│   │   ├── ratelimit.go        # Rate limiting per IP / per utente
│   │   └── logger.go           # Request logging middleware
│   └── services/
│       ├── discovery.go        # DiscoveryService — gestione handshake Push
│       ├── routing.go          # RouteConfigurator — proxy handler dinamico
│       ├── firebase.go         # Wrapper firebase-admin SDK
│       ├── communicator.go     # Client HTTP verso gli altri servizi
│       └── logging.go          # Wrapper Cloud Logging
├── .env.example
└── Dockerfile
```

---

## 3. Flusso di una Richiesta

```
Client
  |
  | Authorization: Bearer <Firebase JWT>
  v
+--------------------------------------------------------+
|                    SSG Gateway (Gin)                   |
|                                                        |
|  [1] FirebaseAuthMiddleware.Authenticate()             |
|      ├─ Verifica JWT con firebaseService.VerifyIDToken |
|      ├─ Estrae uid          -> c.Set("userID")        |
|      ├─ Estrae email        -> c.Set("userEmail")     |
|      ├─ Risolve ruolo       -> c.Set("userRole")      |
|      └─ Inietta negli Header:                         |
|            X-Nexus-User-ID  = uid                      |
|            X-Nexus-Role     = ruolo                    |
|            X-Nexus-Trace-ID = X-Cloud-Trace-Context    |
|                               (o generato runtime)     |
|                                                        |
|  [2] RouteConfigurator.createProxyHandler()            |
|      ├─ Risolve URL target da Firestore               |
|      ├─ Copia headers + rimuove hop-by-hop            |
|      ├─ Ottiene OIDC Token (idtoken.NewTokenSource)    |
|      └─ Invia richiesta al microservizio Cloud Run     |
+--------------------------------------------------------+
             |
             | Authorization: Bearer <OIDC Token IAM>
             | X-Nexus-User-ID, X-Nexus-Role, X-Nexus-Trace-ID
             v
    +--------------------+    +--------------------+
    | Registry Service   |    | Finance Service    |  ...
    | (Cloud Run)        |    | (Cloud Run)        |
    +--------------------+    +--------------------+
```

---

## 4. Autenticazione — `FirebaseAuthMiddleware`

**File:** `internal/middleware/auth.go`

### 4.1 Pipeline di verifica

```go
func (m *FirebaseAuthMiddleware) Authenticate() gin.HandlerFunc {
    // 1. Legge header "Authorization: Bearer <token>"
    // 2. Verifica con m.firebaseService.VerifyIDToken()
    // 3. Risolve ruolo via m.userRepo.GetUserAppRole()
    // 4. Se ruolo non trovato -> checkAutoProvision() per admin-email predefinite
    // 5. Inietta X-Nexus-User-ID, X-Nexus-Role, X-Nexus-Trace-ID nella Request
    // 6. c.Next() -> proxy handler
}
```

### 4.2 Auto-provisioning Admin

Se un utente non ha ancora un ruolo nel database ma la sua email è presente nella lista `adminEmails` (configurata via env), viene **auto-provisionato** come `admin` al primo accesso. Questo permette di inizializzare il sistema senza intervento manuale su Firestore.

### 4.3 Ruoli disponibili

| Ruolo | Accesso |
|---|---|
| `admin` | Tutti gli endpoint |
| `viewer` | Default per utenti autenticati senza ruolo esplicito |

> I ruoli sono **estensibili**: il sistema legge il valore stringa da Firestore senza validazione hard-coded. Ogni microservizio può implementare logiche di autorizzazione aggiuntive tramite il Nexus SDK (`HasRole()`).

### 4.4 RequireRole — RBAC

Alcuni endpoint del Gateway stesso usano `RequireRole` per il controllo accessi:

```go
r.POST("/admin/register", authMiddleware.RequireRole("admin"), handlers.RegisterService)
```

---

## 5. Service Discovery — Pattern Push

**File:** `internal/services/discovery.go`

Il Gateway usa un **pattern Push-based**: i microservizi si registrano attivamente all’avvio chiamando l’endpoint del Gateway. Non c’è polling.

### 5.1 Endpoint di registrazione

```
POST /nexus/discover
Authorization: Bearer <OIDC Token>   (richiesto)
Content-Type: application/json
```

### 5.2 Payload di registrazione (`ServiceDiscoveryResponse`)

```json
{
  "serviceName": "registry-service",
  "description": "Anagrafica polimorfica SSG Nexus",
  "version": "1.2.0",
  "metadata": {
    "env": "production",
    "region": "europe-west8"
  },
  "endpoints": [
    {
      "path": "/entities",
      "method": "GET",
      "summary": "Lista entità",
      "authRequired": true,
      "rateLimit": { "requestsPerMinute": 60, "burst": 10 }
    },
    {
      "path": "/entities",
      "method": "POST",
      "summary": "Crea entità",
      "authRequired": true,
      "rateLimit": { "requestsPerMinute": 30, "burst": 5 }
    }
  ]
}
```

### 5.3 Logica di `RegisterService()`

```
1. Cerca il servizio per nome in Firestore (collection: services)
2. Se NON esiste  -> crea nuovo documento Service con IsActive: true
3. Se esiste      -> aggiorna URL, version, metadata, IsActive: true
4. Deactiva tutti gli endpoint esistenti per questo servizio
5. Crea nuovi endpoint dalla lista nel payload
6. Chiama updateCallback() -> RouteConfigurator.RefreshRoutes()
```

### 5.4 Generazione ID endpoint

Gli endpoint ID sono generati deterministicamente per evitare duplicati su Firestore:

```go
// Pattern: {serviceID}-{METHOD}-{path_sanitized}
// Esempio: registry-service-GET-_entities
func generateEndpointID(serviceID, path, method string) string {
    safePath := strings.ReplaceAll(path, "/", "_")
    safePath = strings.ReplaceAll(safePath, ":", "")
    return fmt.Sprintf("%s-%s-%s", serviceID, method, safePath)
}
```

---

## 6. Routing Dinamico — `RouteConfigurator`

**File:** `internal/services/routing.go`

### 6.1 Startup

All’avvio del Gateway, `RouteConfigurator.Start()` legge da Firestore tutti i servizi e gli endpoint attivi (`isActive: true`) e registra dinamicamente le route sul router Gin.

### 6.2 Registrazione route con middleware condizionale

```go
// Se l'endpoint dichiara authRequired: true,
// il middleware JWT viene inserito PRIMA del proxy handler
if endpoint.AuthRequired {
    handlersChain = append(handlersChain, r.authMiddleware)
}
handlersChain = append(handlersChain, handler)

// Pattern: /{serviceName}/{endpoint.Path}
fullPath := servicePath + endpointPath
r.router.GET(fullPath, handlersChain...)
```

### 6.3 Costruzione URL target

```
URL target = service.URL + requestPath (strippato del prefisso serviceName) + ?queryString
```

Esempio:
```
Richiesta in arrivo:  GET /registry-service/entities?status=ACTIVE
service.URL:          https://registry-service-xyz.run.app
URL target risultante: https://registry-service-xyz.run.app/entities?status=ACTIVE
```

### 6.4 Refresh Route

Quando un microservizio si registra tramite Push Discovery, `RefreshRoutes()` viene invocato automaticamente tramite callback, rileggendo Firestore e aggiungendo le nuove route al router Gin.

> ⚠️ **Attenzione**: Gin non supporta la rimozione di route a runtime. Le route vengono aggiunte ma mai eliminate. Un servizio deregistrato non sarà più proxiato (endpoint deactivato su Firestore), ma la route Gin resterà registrata e risponderà con `404 Service endpoint not found or no longer active`.

---

## 7. Proxy OIDC verso Cloud Run

**File:** `internal/services/routing.go` — `createProxyHandler()`

### 7.1 Perché OIDC

I microservizi su Cloud Run sono configurati con **ingress interno + autenticazione IAM**. Per chiamarli, il Gateway deve presentare un Google OIDC Token con `audience` uguale all’URL base del Cloud Run target.

### 7.2 Caching dei TokenSource

Per evitare di creare un nuovo `TokenSource` ad ogni richiesta (operazione costosa), il `RouteConfigurator` mantiene una cache `map[string]oauth2.TokenSource` protetta da `sync.RWMutex`.

```go
// Double-checked locking pattern per thread safety
func (r *RouteConfigurator) getTokenSource(ctx context.Context, audience string) (oauth2.TokenSource, error) {
    r.tsMu.RLock()
    ts, exists := r.tokenSources[audience]
    r.tsMu.RUnlock()
    if exists { return ts, nil }

    r.tsMu.Lock()
    defer r.tsMu.Unlock()
    // ... crea e mette in cache il nuovo TokenSource
}
```

### 7.3 Header inoltrati al microservizio

| Header | Valore | Fonte |
|---|---|---|
| `Authorization` | `Bearer <OIDC Token>` | Generato da `idtoken.NewTokenSource` |
| `X-Nexus-User-ID` | Firebase UID utente | Iniettato da `Authenticate()` |
| `X-Nexus-Role` | Ruolo utente | Iniettato da `Authenticate()` |
| `X-Nexus-Trace-ID` | Trace ID GCP | `X-Cloud-Trace-Context` o generato runtime |
| `X-User-Email` | Email utente | Da `c.Get("userEmail")` |

**Header hop-by-hop rimossi** prima di inviare al microservizio:
`Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`, `Te`, `Trailer`, `Transfer-Encoding`, `Upgrade`

---

## 8. Rate Limiting

**File:** `internal/middleware/ratelimit.go`

Il rate limit è configurato **per endpoint** nel payload di discovery (`rateLimit.requestsPerMinute`, `rateLimit.burst`). Il middleware applica il token bucket algorithm con i valori dichiarati dal microservizio.

---

## 9. Variabili d’Ambiente

> Riferimento: `.env.example` nel repo `ssg-gateway`

| Variabile | Descrizione | Esempio |
|---|---|---|
| `GCP_PROJECT_ID` | ID Progetto GCP | `ssg-prod-123` |
| `FIREBASE_CREDENTIALS_FILE` | Path al JSON Service Account Firebase | `/secrets/firebase.json` |
| `APP_ID` | Identificatore applicazione per i ruoli Firestore | `nexus` |
| `ADMIN_EMAILS` | Email auto-provisionate come `admin` (CSV) | `admin@ssg.it,dev@ssg.it` |
| `PORT` | Porta HTTP del Gateway | `8080` |
| `GIN_MODE` | `debug` o `release` | `release` |

---

## 10. Implementazione in un Nuovo Microservizio

Per registrare un nuovo microservizio nel Gateway, il servizio deve:

1. **Implementare `/_discover`** (fornito automaticamente dall’SDK Nexus).
2. **All’avvio**, chiamare `POST /nexus/discover` sul Gateway con il payload `ServiceDiscoveryResponse`.
3. **Esporre gli endpoint** con il flag `authRequired: true` per tutti gli endpoint protetti.
4. **Usare il Nexus SDK** per leggere `X-Nexus-User-ID` e `X-Nexus-Role` dalle richieste in arrivo.

Esempio di registrazione all’avvio (Go):

```go
func registerWithGateway(gatewayURL string) error {
    payload := ServiceDiscoveryResponse{
        ServiceName: "my-new-service",
        Version:     "1.0.0",
        Endpoints: []EndpointSpec{
            {Path: "/items", Method: "GET",  AuthRequired: true,  RateLimit: RateLimit{RequestsPerMinute: 60, Burst: 10}},
            {Path: "/items", Method: "POST", AuthRequired: true,  RateLimit: RateLimit{RequestsPerMinute: 30, Burst: 5}},
        },
    }
    body, _ := json.Marshal(payload)
    resp, err := http.Post(gatewayURL+"/nexus/discover", "application/json", bytes.NewReader(body))
    if err != nil || resp.StatusCode != 200 {
        return fmt.Errorf("gateway registration failed")
    }
    return nil
}
```

---

## 11. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | Route Gin non rimuovibili a runtime (servizi deregistrati lasciano route zombie) | ⏳ Aperto |
| 🟡 Media | Trace ID generato con `time.Now().UnixNano()` — non UUID standard | ⏳ Aperto |
| 🟡 Media | `credentials.json` committato nel repo (da spostare in Secret Manager) | ⚠️ Urgente |
| 🟢 Bassa | Test e2e per il flusso discovery + proxy non implementati | ⏳ Aperto |
| 🟢 Bassa | `RefreshRoutes()` non thread-safe su Gin (aggiunta route durante traffico live) | ⏳ Aperto |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per gli standard globali.*
