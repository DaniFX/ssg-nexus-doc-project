# 🌐 SSG Nexus — Standard Globali di Architettura

> Questo documento è la **fonte di verità** per tutti gli standard tecnici trasversali al sistema SSG Nexus.
> Ogni microservizio **deve** conformarsi a queste specifiche. Le deroghe vanno discusse e documentate.

---

## 1. Principi Fondamentali

| Principio | Regola |
|---|---|
| **Gateway-First** | Nessun microservizio è accessibile direttamente da internet. Tutto transita da `ssg-gateway`. |
| **Soft Delete** | I record non vengono mai eliminati fisicamente. Si usa `deletedAt: timestamp` su Firestore. |
| **Immutabilità ERP** | I documenti fiscali (fatture, movimenti contabili) sono immutabili dopo emissione. Modifiche tramite documenti rettificativi. |
| **Identity propagation** | Il Gateway risolve l'identità utente una sola volta e la propaga via header standard. I microservizi la leggono, non la ri-verificano. |
| **Contract-first** | I modelli di dati (Firestore) sono definiti in `ssg-db`; i contratti API e l'SDK condiviso in `ssg-nexus-sdk`. |
| **Audit Log** | Ogni operazione di scrittura deve essere loggata con `X-Nexus-User-ID` per scopi di auditing. |

---

## 2. Header Standard Nexus

Ogni richiesta che attraversa il Gateway verso un microservizio contiene questi header. I microservizi **non devono mai** accettare richieste senza `X-Nexus-User-ID` su route private.

| Header | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `X-Nexus-User-ID` | `string` | ✅ Sì | Firebase UID dell'utente autenticato |
| `X-Nexus-Role` | `string` | ✅ Sì | Ruolo utente (`admin`, `viewer`, ...) |
| `X-Nexus-Trace-ID` | `string` | ⚠️ Raccomandato | Trace ID per correlazione log distribuiti (`X-Cloud-Trace-Context` o generato runtime) |
| `X-User-Email` | `string` | No | Email utente (inoltrata dal Gateway se disponibile) |

### 2.1 Come leggere gli header (SDK)

Usare sempre `nexus.FromContext()` — **mai** leggere gli header raw direttamente nel codice business:

```go
import "github.com/DaniFX/ssg-nexus-sdk/pkg/nexus"

func (h *Handler) GetEntity(c *gin.Context) {
    identity := nexus.FromContext(c.Request.Context())
    // identity.UserID  -> Firebase UID
    // identity.Role    -> Ruolo utente
    // identity.TraceID -> Trace ID per logging
}
```

> `FromContext()` è alimentato dal middleware `nexus.Guard()` che legge gli header e li inietta nel `context.Context` della request.

### 2.2 Controllo ruoli (RBAC)

```go
// HasRole rispetta il bypass automatico per admin
if !nexus.HasRole(c.Request.Context(), "editor") {
    nexus.Failure(c, http.StatusForbidden, nexus.ErrForbidden, "Permesso negato", nil)
    return
}
```

**Regola admin bypass**: un utente con ruolo `admin` supera automaticamente qualsiasi check `HasRole()`, indipendentemente dal ruolo richiesto. Questa logica risiede nell'SDK (`context.go`) ed è centralizzata.

---

## 3. Formato Standard delle Risposte

Tutte le API Nexus rispondono con `StandardResponse`. Non sono ammesse strutture JSON custom.

### 3.1 Struttura

```go
// Da: ssg-nexus-sdk/pkg/nexus/response.go
type StandardResponse struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Meta    interface{} `json:"meta,omitempty"`
    Error   *NexusError `json:"error,omitempty"`
}
```

### 3.2 Risposta di successo

```json
{
  "success": true,
  "data": { ... },
  "meta": { "total": 42, "page": 1, "perPage": 20 }
}
```

```go
nexus.Success(c, entities, gin.H{"total": len(entities)})
```

### 3.3 Risposta di errore

```json
{
  "success": false,
  "error": {
    "code": "ERR_NOT_FOUND",
    "message": "Entità non trovata",
    "details": { "id": "abc123" }
  }
}
```

```go
nexus.Failure(c, http.StatusNotFound, nexus.ErrNotFound, "Entità non trovata", gin.H{"id": id})
```

---

## 4. Codici Errore Standard

Definiti in `ssg-nexus-sdk/pkg/nexus/errors.go`. Usare sempre le costanti SDK — mai stringhe letterali.

| Costante SDK | Valore stringa | HTTP | Quando usarlo |
|---|---|---|---|
| `nexus.ErrUnauthorized` | `ERR_UNAUTHORIZED` | `401` | Token mancante o non valido |
| `nexus.ErrForbidden` | `ERR_FORBIDDEN` | `403` | Ruolo insufficiente |
| `nexus.ErrNotFound` | `ERR_NOT_FOUND` | `404` | Risorsa non esistente o soft-deleted |
| `nexus.ErrImmutableRecord` | `ERR_IMMUTABLE_RECORD` | `409` | Tentativo di modifica su record immutabile |
| `nexus.ErrValidationFailed` | `ERR_VALIDATION_FAILED` | `422` | Dati di input non validi |
| `nexus.ErrInternal` | `ERR_INTERNAL` | `500` | Errore generico non gestibile |
| `nexus.ErrFileNotFound` | `ERR_FILE_NOT_FOUND` | `404` | File GCS non trovato |
| `nexus.ErrInvalidFileType` | `ERR_INVALID_FILE_TYPE` | `400` | MIME type non accettato |
| `nexus.ErrFileTooLarge` | `ERR_FILE_TOO_LARGE` | `413` | File supera dimensione massima |
| `nexus.ErrStorageUpload` | `ERR_STORAGE_UPLOAD_FAILED` | `500` | Errore upload su Cloud Storage |
| `nexus.ErrSignedURLError` | `ERR_SIGNED_URL_GENERATION` | `500` | Errore generazione Signed URL GCS |

---

## 5. Nexus Guard — Middleware Obbligatorio

**File SDK:** `ssg-nexus-sdk/pkg/nexus/middleware.go`

Ogni microservizio **deve** applicare `nexus.Guard()` su tutte le route private. Il Guard:

1. Verifica la presenza di `X-Nexus-User-ID` (iniettato dal Gateway)
2. Rifiuta con `401 ERR_UNAUTHORIZED` se assente
3. Inietta `UserID`, `Role`, `TraceID` nel `context.Context` della request

```go
router := gin.New()

// Route pubbliche (senza Guard)
router.GET("/_health",   handlers.Health)
router.GET("/_discover", nexus.DiscoveryHandler(serviceDefinition))

// Route private: Guard obbligatorio
private := router.Group("/")
private.Use(nexus.Guard())
{
    private.GET("/entities",     handlers.ListEntities)
    private.POST("/entities",    handlers.CreateEntity)
    private.GET("/entities/:id", handlers.GetEntity)
}
```

> Il Guard verifica solo la presenza dell'identità. Per logiche RBAC aggiuntive, usare `nexus.HasRole()` all'interno del singolo handler.

---

## 6. Service Discovery — Pattern di Registrazione

**File SDK:** `ssg-nexus-sdk/pkg/nexus/discovery.go`

### 6.1 Struttura `ServiceDefinition`

```go
var Definition = nexus.ServiceDefinition{
    ServiceName: "my-service",
    Version:     "1.0.0",
    Metadata:    map[string]interface{}{"region": "europe-west8"},
    Endpoints: []nexus.Endpoint{
        {
            Path:         "/items",
            Method:       "GET",
            Summary:      "Lista items",
            AuthRequired: true,
            RateLimit:    &nexus.RateLimit{RequestsPerMinute: 60, Burst: 10},
            // InputSchema e OutputSchema opzionali (JSON Schema)
        },
    },
}
```

### 6.2 Avvio standard

```go
func main() {
    router := gin.New()
    nexus.RegisterDiscovery(router, Definition)     // espone GET /_discover
    nexus.StartGatewayHandshake(Definition)         // push asincrono (5 retry x 5s)
    router.Run(":8080")
}
```

### 6.3 Variabili d'ambiente richieste

| Variabile | Descrizione | Esempio |
|---|---|---|
| `GATEWAY_URL` | URL interno del Gateway | `https://ssg-gateway-xyz.run.app` |
| `SERVICE_URL` | URL Cloud Run di questo servizio | `https://my-service-xyz.run.app` |
| `INTERNAL_SECRET` | Segreto condiviso per autenticare la registrazione | `<valore da Secret Manager>` |

> In assenza di queste variabili l'handshake viene disabilitato silenziosamente con log di warning.

---

## 7. Autenticazione Firebase — Solo Gateway

**File SDK:** `ssg-nexus-sdk/pkg/nexus/auth.go`

L'`Authenticator` Firebase è usato **solo dal Gateway**. I microservizi non validano JWT Firebase.

- In produzione (Cloud Run): usa Application Default Credentials automaticamente
- In locale: usa `credentialsJSON` da file
- Il ruolo viene letto dal **Custom Claim** `"role"` del JWT (default `"user"` se assente)

```go
// Impostare il ruolo via Firebase Admin SDK:
claims := map[string]interface{}{"role": "admin"}
firebaseClient.SetCustomUserClaims(ctx, uid, claims)
```

---

## 8. Modello Dati Base — `NexusDoc`

Ogni documento Firestore nel sistema deve incorporare `NexusDoc` (definito in `ssg-db`):

```go
type NexusDoc struct {
    ID        string     `firestore:"id"`
    CreatedAt time.Time  `firestore:"createdAt"`
    UpdatedAt time.Time  `firestore:"updatedAt"`
    DeletedAt *time.Time `firestore:"deletedAt,omitempty"` // nil = attivo
    Immutable bool       `firestore:"immutable"`           // true = sola lettura
    TenantID  string     `firestore:"tenantId"`
}
```

### 8.1 Regole di integrità

- **Soft Delete**: `deletedAt = now()`. Mai `Delete()` su Firestore.
- **Query liste**: filtrare sempre con `where("deletedAt", "==", null)`.
- **Immutabilità**: se `immutable == true` → ritornare `ERR_IMMUTABLE_RECORD` su qualsiasi PUT/PATCH.
- **TenantID**: valorizzato con l'`AppID` del progetto (predisposto per multi-tenant futuro).

### 8.2 Repository Layer — `repository.Repository`

**File SDK:** `ssg-nexus-sdk/pkg/nexus/repository/firestore.go`

Ogni microservizio usa `repository.NewRepository()` come unico punto di accesso a Firestore. **Non usare mai `firestoreClient.Collection().Doc()` direttamente** nel codice business.

```go
// Inizializzazione in main.go
repo := repository.NewRepository(firestoreClient, "entities")

// Create — inietta automaticamente id, createdAt, updatedAt, createdBy
err := repo.Create(ctx, uuid, dataMap)

// Update — verifica IsLocked() prima di procedere; appende updatedAt
err := repo.Update(ctx, id, []firestore.Update{
    {Path: "status", Value: "ACTIVE"},
})

// SoftDelete — imposta deletedAt; verifica IsLocked()
err := repo.SoftDelete(ctx, id)

// IsLocked — true se immutable==true OR status=="ISSUED"
locked, err := repo.IsLocked(ctx, id)
```

**Ciclo di vita di un record:**

```
Create()      → inietta: id, createdAt, updatedAt, createdBy (da context)
Update()      → IsLocked()? ERR_IMMUTABLE_RECORD : appende updatedAt
SoftDelete()  → IsLocked()? ERR_IMMUTABLE_RECORD : imposta deletedAt
```

> Il campo `createdBy` viene estratto automaticamente da `nexus.FromContext(ctx).UserID` — non passarlo manualmente nel `dataMap`.

---

## 9. Navigator Pattern — Filtri & Ordinamento Firestore

**File SDK:** `ssg-nexus-sdk/pkg/nexus/repository/navigator.go`

`ApplyNavigator()` traduce i query parameter URL in filtri Firestore standardizzati. **Esclude sempre i soft-deleted** (`deletedAt == nil`) come comportamento di default.

```go
// In ogni handler di tipo LIST
filters := map[string]string{}
for k, v := range c.Request.URL.Query() {
    filters[k] = v[0]
}
query := repo.ApplyNavigator(
    firestoreClient.Collection("entities").Query,
    filters,
)
```

### Parametri Supportati

| Query Param | Comportamento | Esempio |
|---|---|---|
| `sort=createdAt` | Ordina ASC per il campo | `?sort=createdAt` |
| `sort=-createdAt` | Ordina DESC (prefisso `-`) | `?sort=-createdAt` |
| `status=PAID` | Filtro uguaglianza | `?status=PAID` |
| `type=PERSON` | Filtro uguaglianza | `?type=PERSON` |
| `limit=N` | ⚠️ Placeholder — da implementare come `query.Limit(n)` | `?limit=20` |

> **Nota:** Il filtro `deletedAt == nil` è sempre applicato prima di qualsiasi altro filtro. Per esporre i record eliminati (es. admin panel) è necessario costruire la query manualmente senza `ApplyNavigator`.

---

## 10. Chiamate Inter-Servizio — `NexusClient`

**File SDK:** `ssg-nexus-sdk/pkg/nexus/client.go`

Quando un microservizio chiama un altro microservizio, **deve** usare `NexusClient.Do()` per propagare automaticamente `X-Nexus-User-ID` e `X-Nexus-Role` dal contesto corrente. Non impostare mai questi header manualmente.

```go
nc := &nexus.NexusClient{}

// La chiamata propaga automaticamente l'identità dal ctx
req, _ := http.NewRequest("GET", os.Getenv("REGISTRY_SERVICE_URL")+"/api/v1/registry/entities/"+entityID, nil)
resp, err := nc.Do(c.Request.Context(), req)
if err != nil {
    nexus.Failure(c, 500, nexus.ErrInternal, "Errore chiamata registry", nil)
    return
}
```

**Flusso di propagazione:**

```
Client  →  Gateway  →  Service A  →  Service B
             │               │             │
         verifica JWT    Guard()       Guard()
         crea Identity   FromContext() FromContext()
         inietta header  usa UserID    usa UserID
                         NexusClient   ...
                         propaga →
```

> **Regola:** `TraceID` non è ancora propagato da `NexusClient`. Per la tracciabilità distribuita completa, aggiungere manualmente `req.Header.Set("X-Nexus-Trace-ID", identity.TraceID)` fino a quando l'SDK non lo gestirà automaticamente.

---

## 11. Storage GCS — Signed URL V4

**File SDK:** `ssg-nexus-sdk/pkg/nexus/storage/gcs.go`

Il pattern di upload/download su GCS usa esclusivamente **Signed URL V4** (firma via Cloud Run Service Account ADC). Il file non transita mai attraverso il microservizio.

### Flusso Upload

```
Client → POST /documents/upload-url → Microservizio
                                           │
                               GCSClient.GenerateUploadURL(
                                 objectName,   // es. "documents/{uuid}/file.pdf"
                                 mimeType,     // es. "application/pdf"
                                 expiresMin,   // es. 15
                               )
                                           │
                               { "uploadUrl": "https://storage.googleapis.com/..." }
                                           │
Client ←──────────────────────────────────┘
  │
  └──► PUT signed URL ──► GCS  (upload diretto, senza passare dal backend)
```

### Flusso Download

```
Client → GET /documents/{id}/download-url → Microservizio
                                                 │
                                 GCSClient.GenerateDownloadURL(
                                   objectName,  // percorso GCS
                                   expiresMin,  // es. 60
                                 )
                                                 │
                                 { "downloadUrl": "https://storage.googleapis.com/..." }
                                                 │
Client ←─────────────────────────────────────────┘
  │
  └──► GET signed URL ──► GCS  (download diretto, link temporaneo)
```

### Utilizzo

```go
// Inizializzazione in main.go
gcs := storage.NewGCSClient(os.Getenv("GCS_BUCKET_NAME"))

// Generare URL upload (PUT, scade in 15 minuti)
uploadURL, err := gcs.GenerateUploadURL(
    "documents/"+docID+"/"+fileName,
    "application/pdf",
    15,
)
if err != nil {
    nexus.Failure(c, 500, nexus.ErrStorageUpload, "Errore generazione upload URL", nil)
    return
}

// Generare URL download (GET, scade in 60 minuti)
downloadURL, err := gcs.GenerateDownloadURL(
    "documents/"+docID+"/"+fileName,
    60,
)
if err != nil {
    nexus.Failure(c, 500, nexus.ErrSignedURLError, "Errore generazione download URL", nil)
    return
}
```

> La firma avviene tramite le **Application Default Credentials** del Service Account Cloud Run. In locale è necessario autenticarsi con `gcloud auth application-default login` o usare una chiave JSON tramite `GOOGLE_APPLICATION_CREDENTIALS`.

---

## 12. Stack Tecnologico Standard

| Componente | Tecnologia | Note |
|---|---|---|
| **Linguaggio** | Go 1.21+ | Tutti i microservizi |
| **HTTP Framework** | Gin (`github.com/gin-gonic/gin`) | Standard di progetto |
| **Database** | Cloud Firestore | NoSQL document store |
| **Storage file** | Google Cloud Storage | Upload documenti, fatture |
| **Autenticazione** | Firebase Authentication | JWT + Custom Claims |
| **Auth inter-service** | Google OIDC (`idtoken`) | Gateway → Cloud Run (IAM) |
| **Secret management** | GCP Secret Manager | Credenziali e config sensitive |
| **Infrastruttura** | Cloud Run (serverless) | Deploy di tutti i servizi |
| **SDK condiviso** | [`ssg-nexus-sdk`](https://github.com/DaniFX/ssg-nexus-sdk) | Contratti, guard, discovery, errors |
| **Modelli dati** | [`ssg-db`](https://github.com/DaniFX/ssg-db) | Structs Go + Repository pattern |

---

## 13. Variabili d'Ambiente — Schema Completo

| Variabile | Servizi | Obbligatoria | Note |
|---|---|---|---|
| `GCP_PROJECT_ID` | Tutti | ✅ | ID progetto GCP. Assenza → `log.Fatal` |
| `PORT` | Tutti | ✅ | Porta HTTP. **Mai hardcoded**: usare `os.Getenv("PORT")` |
| `GATEWAY_URL` | Tutti | ✅ | URL Gateway per handshake discovery |
| `SERVICE_URL` | Tutti | ✅ | URL Cloud Run del servizio stesso |
| `INTERNAL_SECRET` | Tutti | ✅ | Shared secret per autenticazione inter-servizio |
| `GCS_BUCKET_NAME` | Document, Mail Reader | ✅ | Nome bucket GCS upload/download |
| `FIREBASE_CREDENTIALS_JSON` | Gateway | ⚠️ Opzionale | Se assente usa ADC (Cloud Run) |
| `GOOGLE_CLOUD_PROJECT` | Finance Service | ⚠️ Alternativa | Alias usato da alcune librerie GCP. Preferire `GCP_PROJECT_ID` |
| `GOOGLE_APPLICATION_CREDENTIALS` | Locale | ⚠️ Solo dev | Path al JSON delle credenziali GCP per sviluppo locale |

---

## 14. Checklist Nuovo Microservizio

Prima del deploy di qualsiasi nuovo servizio nel sistema Nexus:

- [ ] Dipendenza da `ssg-nexus-sdk` nel `go.mod`
- [ ] `nexus.Guard()` applicato su tutte le route private
- [ ] `nexus.RegisterDiscovery()` + `nexus.StartGatewayHandshake()` in `main()`
- [ ] Tutti i modelli Firestore estendono `NexusDoc`
- [ ] Accesso Firestore solo via `repository.NewRepository()` — mai chiamate dirette
- [ ] Liste Firestore usano `ApplyNavigator()` — escludono soft-deleted per default
- [ ] Chiamate inter-servizio usano `NexusClient.Do()` — non impostare header manualmente
- [ ] Upload/download file usano `storage.NewGCSClient()` con Signed URL V4
- [ ] Tutte le risposte usano `nexus.Success()` / `nexus.Failure()`
- [ ] Tutti gli errori usano le costanti `nexus.Err*`
- [ ] `PORT` letta da `os.Getenv("PORT")` — **mai hardcoded**
- [ ] `GCP_PROJECT_ID` verificata con `log.Fatal` se assente
- [ ] `GATEWAY_URL`, `SERVICE_URL`, `INTERNAL_SECRET` configurati in env / Secret Manager
- [ ] Nessuna credenziale committata nel repo
- [ ] Ingress Cloud Run impostato su **interno** (no accesso pubblico diretto)
- [ ] Health check `GET /_health` esposto senza Guard
- [ ] Operazioni di scrittura loggano `X-Nexus-User-ID` (audit)

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
