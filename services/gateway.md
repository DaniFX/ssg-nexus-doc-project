# 🚪 SSG Gateway — Specifiche Tecniche

> **Repository:** [`ssg-gateway`](https://github.com/DaniFX/ssg-gateway)  
> **Stack:** Go 1.21+, Gin, Firebase Auth, Firestore (`ssg-db`), GCP Cloud Run, OIDC (`google/idtoken`)  
> **Ruolo:** Unico punto di ingresso (API Gateway) per il progetto SSG Nexus. Gestisce autenticazione Firebase JWT, autorizzazione RBAC, routing dinamico verso i microservizi e propagazione dell'identità utente tramite header Nexus.

---

## 1. Responsabilità del Gateway

| Responsabilità | Implementazione |
|---|---|
| Autenticazione JWT Firebase | `FirebaseAuthMiddleware.Authenticate()` |
| Autorizzazione RBAC per role | `FirebaseAuthMiddleware.RequireRole(...)` |
| Routing dinamico microservizi | `RouteConfigurator` + Firestore (`services`, `service_endpoints`) |
| Service Discovery (Push) | `POST /internal/register` + `DiscoveryService.RegisterService()` |
| Proxy HTTP verso Cloud Run | `createProxyHandler()` con OIDC token injection |
| Propagazione identità utente | Header `X-Nexus-User-ID`, `X-Nexus-Role`, `X-Nexus-Trace-ID` |
| Rate Limiting | `RateLimiter` token bucket (10 rps, burst 20) |
| Migrations database | `ssg-db` migrator al boot |
| Logging strutturato | `slog.JSONHandler` + GCP Cloud Logging |
| CORS | Whitelist origini configurate staticamente |

---

## 2. Struttura del Progetto

```
ssg-gateway/
├── cmd/
│   └── gateway/
│       └── main.go                  # Bootstrap: config, Firebase, migrator, router, handlers
├── internal/
│   ├── config/
│   │   └── config.go                # Load() — tutte le env var in una struct Config
│   ├── handlers/
│   │   ├── auth.go                  # Login (POST /auth/login)
│   │   ├── health.go                # /health, /ready, /live
│   │   ├── user.go                  # CRUD utenti (solo admin)
│   │   ├── role.go                  # CRUD ruoli (solo admin)
│   │   ├── app.go                   # CRUD app (solo admin)
│   │   ├── communicator.go          # POST /admin/send-email
│   │   └── logs.go                  # GET /admin/logs (GCP Cloud Logging)
│   ├── middleware/
│   │   ├── auth.go                  # FirebaseAuthMiddleware: Authenticate(), RequireRole()
│   │   ├── logger.go                # LoggerContext() — trace ID nel context Gin
│   │   └── ratelimit.go             # RateLimiter token bucket + CleanupRateLimiters()
│   ├── models/                      # Eventuale typing locale
│   └── services/
│       ├── discovery.go             # DiscoveryService: RegisterService(), ServiceDiscoveryResponse
│       ├── routing.go               # RouteConfigurator: Start(), RefreshRoutes(), proxy handler
│       ├── firebase.go              # FirebaseService: VerifyIDToken(), CreateUser(), ...
│       ├── logging.go               # LoggingService: GCP Cloud Logging
│       └── communicator.go          # CommunicatorClient: gRPC verso ssg-mail-reader-service
└── ssg-db/                          # Submodule: client Firestore condiviso, models, migrations
```

---

## 3. Bootstrap (main.go)

Sequenza di avvio in ordine esatto:

```
1. slog.JSONHandler → logger strutturato
2. godotenv.Load()  → carica .env (warn se assente)
3. config.Load()    → tutti i valori env in Config struct
4. services.NewFirebaseService()         → verifica JWT Firebase Admin SDK
5. db.NewMigrator() + migrator.Migrate() → esegue migrazioni Firestore al boot (FATALE se fallisce)
6. firestore.NewClientWithClient()       → client condiviso (User, Role, App repos)
7. middleware.NewFirebaseAuthMiddleware() → istanza auth (appID = "ssg-admin")
8. services.NewCommunicatorClient()      → gRPC mail (Warning se fallisce, non fatale)
9. services.NewLoggingService()          → GCP Cloud Logging (Warning se fallisce)
10. var routeConfigurator *RouteConfigurator  ← predichiarato (closure callback)
11. services.NewDiscoveryService(callback)    → callback chiama routeConfigurator.RefreshRoutes()
12. gin.Default() + middleware.LoggerContext()
13. services.NewRouteConfigurator() → carica rotte attive da Firestore
14. go routeConfigurator.Start()    → goroutine avvia proxy routes
15. cors.New()       → whitelist CORS
16. RateLimiter 10 rps / burst 20 + goroutine cleanup ogni 5m
17. Registrazione handler statici (health, auth, admin, me)
18. r.Run(":PORT")
```

> **Nota critica:** La pre-dichiarazione di `routeConfigurator` (step 10) prima del `DiscoveryService` (step 11) è intenzionale: la callback di discovery deve poter chiamare `routeConfigurator.RefreshRoutes()`, che al momento della costruzione del DiscoveryService non è ancora inizializzato. La closure cattura il puntatore.

---

## 4. Middleware di Autenticazione

**File:** `internal/middleware/auth.go`

### 4.1 `Authenticate()` — Flusso JWT Firebase

```
HTTP Request
  |
  | [1] Authorization header presente?
  |     → No  → 401 UNAUTHORIZED (Missing authorization header)
  |
  | [2] Ha prefisso "Bearer "?
  |     → No  → 401 UNAUTHORIZED (Invalid authorization format)
  |
  | [3] firebaseService.VerifyIDToken(idToken)
  |     → Errore → 401 UNAUTHORIZED (Invalid or expired token)
  |
  | [4] Estrai userID (token.UID) e userEmail (claims["email"])
  |     → c.Set("userID"), c.Set("userEmail")
  |
  | [5] userRepo.GetUserAppRole(userID, appID="ssg-admin")
  |     → Errore/nil → checkAutoProvision(userID, userEmail)
  |         → Email in AdminConfig.Emails? → crea/aggiorna user con role="admin"
  |         → else → role="viewer" (default)
  |
  | [6] c.Set("userRole", role)
  |
  | [7] PROPAGAZIONE NEXUS:
  |     → c.Request.Header.Set("X-Nexus-User-ID", userID)
  |     → c.Request.Header.Set("X-Nexus-Role", role)
  |     → X-Nexus-Trace-ID = X-Cloud-Trace-Context (GCP) oppure ssg-trace-{UnixNano}
  |
  v
c.Next() → handler/proxy
```

### 4.2 `RequireRole(allowedRoles...)` — RBAC

Verifica che `c.Get("userRole")` sia incluso nella lista `allowedRoles`. Se assente o non autorizzato: `403 FORBIDDEN`. Usato esclusivamente sul gruppo `/api/v1/admin`.

### 4.3 Auto-Provisioning Admin

Se un utente Firebase non è presente in Firestore **ma** la sua email è in `ADMIN_EMAILS`, viene creato automaticamente con ruolo `admin`. Utile per il primo accesso senza setup manuale del DB.

---

## 5. Service Discovery (Push Model)

**File:** `internal/services/discovery.go`

Il Gateway non fa polling verso i microservizi. I microservizi **si auto-registrano** al boot tramite handshake.

### 5.1 Endpoint di registrazione

```
POST /internal/register
Header: X-Internal-Secret: <INTERNAL_SECRET>
Header: X-Service-Url: https://mio-servizio.run.app  (oppure metadata.targetUrl)
Body: ServiceDiscoveryResponse
```

**`ServiceDiscoveryResponse` (payload di registrazione):**
```json
{
  "serviceName": "finance-service",
  "description": "Gestione fatture e pagamenti",
  "version": "1.0.0",
  "metadata": { "targetUrl": "https://..." },
  "endpoints": [
    {
      "path": "/api/v1/finance/invoices/:id/issue",
      "method": "PATCH",
      "summary": "Emette una fattura",
      "authRequired": true,
      "rateLimit": { "requestsPerMinute": 60, "burst": 10 }
    }
  ]
}
```

### 5.2 Flusso `RegisterService()`

```
[1] serviceRepo.GetAll() → cerca servizio per nome
[2] Se non esiste → serviceRepo.Create() con IsActive=true
    Se esiste     → serviceRepo.Update() (URL, Version, IsActive=true)
[3] endpointRepo.GetByServiceID() → deactivate tutti gli endpoint esistenti
[4] Per ogni EndpointSpec → endpointRepo.Create() con IsActive=true
    EndpointID = "{serviceID}-{METHOD}-{path_safe}" (slash→underscore, colon rimossi)
[5] updateCallback() → RouteConfigurator.RefreshRoutes()
```

> **Idempotenza:** Una re-registrazione disattiva gli endpoint vecchi e crea quelli nuovi. Non ci sono duplicati.

---

## 6. Routing Dinamico

**File:** `internal/services/routing.go`

### 6.1 `RouteConfigurator`

```go
type RouteConfigurator struct {
    router         *gin.Engine
    serviceRepo    repository.ServiceRepository
    endpointRepo   repository.ServiceEndpointRepository
    httpClient     *http.Client        // timeout 30s
    activeRoutes   map[string]*routeInfo  // thread-safe con RWMutex
    authMiddleware gin.HandlerFunc     // iniettato da main.go
    tokenSources   map[string]oauth2.TokenSource  // cache OIDC per audience
}
```

### 6.2 Registrazione rotta con middleware chain

Per ogni endpoint attivo in Firestore:

```
fullPath = "/" + service.Name + endpoint.Path

handlersChain:
  Se endpoint.AuthRequired == true:
    → [authMiddleware.Authenticate()]  ← JWT Firebase validato qui
  → [createProxyHandler(service, endpoint)]  ← proxy HTTP

router.{METHOD}(fullPath, handlersChain...)
```

**Deduplication:** Se una route con la stessa chiave `{serviceID}:{endpointID}:{method}` esiste già in `activeRoutes`, viene saltata (Gin non permette registrazioni duplicate).

### 6.3 Proxy Handler — Flusso Completo

```
Richiesta in arrivo su /{service.Name}/api/v1/...
  |
  | [1] Verifica activeRoutes[routeKey] → se non attiva: 404
  |
  | [2] Path rewriting:
  |     requestPath = c.Request.URL.Path
  |     requestPath = TrimPrefix(requestPath, "/"+service.Name)
  |     targetURL = service.URL + requestPath + "?" + RawQuery
  |
  | [3] Copia headers originali → rimuove hop-by-hop
  |     (Connection, Keep-Alive, Transfer-Encoding, ...)
  |
  | [4] OIDC Token Injection:
  |     audience = service.URL (trim trailing slash)
  |     getTokenSource(ctx, audience)  → cache per audience
  |     tokenSource.Token()  → OIDC JWT firmato con SA Gateway
  |     proxyReq.Header.Set("Authorization", "Bearer "+oidcToken)
  |
  | [5] User Identity Forwarding:
  |     X-User-Id    = c.Get("userID")
  |     X-User-Email = c.Get("userEmail")
  |     X-User-Role  = c.Get("userRole")
  |     (già presenti X-Nexus-* iniettati da Authenticate() — step 7)
  |
  | [6] httpClient.Do(proxyReq) → risposta del microservizio
  |     Errore → 502 Bad Gateway
  |
  | [7] Copia headers risposta → c.Writer.WriteHeader(statusCode)
  |     io.Copy(c.Writer, resp.Body)
```

> **Sicurezza IAM:** Il token OIDC sostituisce il Bearer JWT dell'utente nel trasporto verso Cloud Run. Il microservizio riceve il token di servizio del Gateway (autorizzato IAM), **non** il token Firebase dell'utente. L'identità dell'utente viene propagata separatamente tramite `X-Nexus-User-ID` e `X-Nexus-Role`.

---

## 7. Endpoint Statici del Gateway

Questa è la lista degli endpoint gestiti direttamente dal Gateway (non proxyati).

| Metodo | Path | Auth | Ruolo richiesto | Handler |
|---|---|---|---|---|
| `GET` | `/health` | ❌ | — | `healthHandler.Health` |
| `GET` | `/ready` | ❌ | — | `healthHandler.Ready` |
| `GET` | `/live` | ❌ | — | `healthHandler.Live` |
| `POST` | `/auth/login` | ❌ | — | `authHandler.Login` |
| `POST` | `/internal/register` | `X-Internal-Secret` | — | Discovery handshake |
| `GET` | `/api/v1/public` | ❌ | — | Endpoint di test |
| `GET` | `/api/v1/me` | ✅ Firebase JWT | any | Profilo utente corrente |
| `GET` | `/api/v1/admin/stats` | ✅ | `admin` | Statistiche mock |
| `GET/POST/PUT/DELETE` | `/api/v1/admin/users[/:id]` | ✅ | `admin` | CRUD utenti |
| `PUT` | `/api/v1/admin/users/:id/role` | ✅ | `admin` | Aggiornamento ruolo |
| `GET/POST/PUT/DELETE` | `/api/v1/admin/roles[/:id]` | ✅ | `admin` | CRUD ruoli |
| `GET/POST/PUT/DELETE` | `/api/v1/admin/apps[/:id]` | ✅ | `admin` | CRUD applicazioni |
| `POST` | `/api/v1/admin/send-email` | ✅ | `admin` | Invio email via Communicator |
| `GET` | `/api/v1/admin/logs` | ✅ | `admin` | Log GCP Cloud Logging |

---

## 8. Rate Limiting

**File:** `internal/middleware/ratelimit.go`

- **Algoritmo:** Token bucket per IP
- **Limiti:** 10 request/s (rate), burst 20
- **Cleanup:** goroutine elimina bucket inattivi ogni 5 minuti
- Applicato **globalmente** su tutto il router (incluse le rotte proxy dinamiche)

---

## 9. CORS

Configurato staticamente in `main.go`. Origini consentite:

```
http://localhost:5173
http://localhost:3000
https://ssg-api-99ac0.web.app
https://ssg-api-99ac0.firebaseapp.com
```

Metodi: `GET, POST, PUT, DELETE, OPTIONS` | Headers: `Origin, Content-Type, Authorization` | `AllowCredentials: true`

> ⚠️ **TODO:** Le origini CORS sono hardcodate nel sorgente. In produzione andrebbero spostate in variabile d'ambiente.

---

## 10. Variabili d'Ambiente

| Variabile | Descrizione | Comportamento se assente |
|---|---|---|
| `PORT` | Porta HTTP | Default `8080` |
| `ENVIRONMENT` | `development` / `production` | — |
| `FIREBASE_PROJECT_ID` | Project ID Firebase | Fatale (Firebase init) |
| `FIREBASE_PRIVATE_KEY` | Chiave privata SA Firebase Admin | Fatale |
| `FIREBASE_CLIENT_EMAIL` | Email SA Firebase Admin | Fatale |
| `FIRESTORE_PROJECT_ID` | Project ID Firestore (ssg-db) | Fatale (migrator) |
| `FIREBASE_WEB_API_KEY` | API Key Web Firebase (per login REST) | `authHandler.Login` non funziona |
| `INTERNAL_SECRET` | Token condiviso per `/internal/register` | Tutti i register → `401` |
| `ADMIN_EMAILS` | Lista email admin auto-provisionate | Nessun auto-provisioning |

> **`credentials.json`** è presente nella root del repo (service account GCP). In Cloud Run viene sostituita dall'identità del service account IAM associato al servizio.

---

## 11. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/ssg/ssg-db` | Client Firestore condiviso, models, migrations, repositories |
| `firebase.google.com/go/v4` | Firebase Admin SDK (verifica JWT) |
| `google.golang.org/api/idtoken` | Generazione token OIDC per Cloud Run IAM |
| `golang.org/x/oauth2` | TokenSource caching OIDC |
| `github.com/gin-gonic/gin` | HTTP framework + router dinamico |
| `github.com/gin-contrib/cors` | Middleware CORS |
| `github.com/joho/godotenv` | Caricamento `.env` |
| `log/slog` | Logging strutturato JSON (stdlib Go 1.21+) |

---

## 12. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | CORS origini hardcodate in `main.go` — non configurabili senza rebuild | ⚠️ Da spostare in env |
| 🔴 Alta | `GET /api/v1/admin/stats` restituisce dati mock statici (`totalUsers: 100`) | ⏳ Non implementato |
| 🟡 Media | `credentials.json` committato nella root — rischio esposizione accidentale | ⚠️ Aggiungere a `.gitignore` |
| 🟡 Media | `RouteConfigurator.RefreshRoutes()` non gestisce route rimosse (solo aggiunta) — un servizio deregistrato mantiene la rotta Gin | ⏳ Aperto |
| 🟡 Media | `X-Nexus-Trace-ID` generato con `UnixNano` se non presente — non UUID conforme W3C TraceContext | ⏳ Da migliorare |
| 🟡 Media | Nessun timeout configurabile per il proxy `httpClient` (hardcoded 30s) | ⏳ Aperto |
| 🟢 Bassa | Nessun test unitario o di integrazione nel repo | ⏳ Aperto |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
