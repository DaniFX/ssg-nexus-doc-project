# 📂 SSG Nexus Document Service — Specifiche Tecniche

> **Repository:** [`ssg-nexus-document-service`](https://github.com/DaniFX/ssg-nexus-document-service)
> **Stack:** Go 1.21+, Gin, Google Cloud Storage (GCS), Firestore, `ssg-nexus-sdk`
> **Ruolo:** Gestisce il ciclo di vita dei file (upload, archiviazione, metadatazione). Ponte tra GCS (fisico) e Firestore (logico). Ogni documento è sempre associato a un'entità padre (`ENTITY`, `INVOICE`, `PROJECT`) tramite il campo `relation`.

---

## 1. Principio Architetturale: Secure Upload via Signed URL

Il servizio **non riceve mai il file binario** direttamente. Usa un pattern a due step che scarica il traffico binario dal microservizio e lo dirige direttamente su GCS:

```
Frontend
  │
  │ [1] POST /api/v1/documents/upload-url
  │     { fileName, mimeType, category, parentType, parentId }
  ▼
┌──────────────────────────────────┐
│  Document Service                │
│  HandleGetUploadURL()            │
│  → genera docId (UUID v4)        │
│  → path: entities/{parentId}/    │
│           docs/{docId}.ext       │
│  → GCS.GenerateUploadURL()       │
│    (Signed URL V4, 15 min, PUT)  │
└──────────────────────────────────┘
  │
  │ [2] Risposta: { docId, uploadUrl, storagePath }
  ▼
Frontend
  │
  │ [3] PUT {uploadUrl}  ← file binario diretto su GCS
  │     (senza passare per il servizio)
  ▼
Google Cloud Storage
  │
  │ [4] POST /api/v1/documents/finalize
  │     { docId, fileName, mimeType, size, storagePath, ... }
  ▼
┌──────────────────────────────────┐
│  Document Service                │
│  HandleFinalizeUpload()          │
│  → scrive metadati su Firestore  │
│    collezione "documents"        │
└──────────────────────────────────┘
```

---

## 2. Struttura del Progetto

```
ssg-nexus-document-service/
├── cmd/
│   └── document-service/
│       └── main.go             # Bootstrap: Firestore, GCS, Routes, Discovery, Guard
├── internal/
│   ├── api/
│   │   └── handlers.go         # HandleGetUploadURL, HandleFinalizeUpload
│   ├── models/
│   │   └── document.go         # Struct Document + Relation
│   └── storage/
│       └── gcs.go              # GCSClient: NewGCSClient(), GenerateUploadURL()
└── Dockerfile
```

---

## 3. Modello Dati

**File:** `internal/models/document.go`

```go
type Document struct {
    repository.NexusDoc // createdAt, updatedAt, deletedAt, createdBy

    FileName    string                 `firestore:"fileName"    json:"fileName"`
    MimeType    string                 `firestore:"mimeType"    json:"mimeType"`
    Size        int64                  `firestore:"size"        json:"size"`
    StoragePath string                 `firestore:"storagePath" json:"storagePath"`
    Category    string                 `firestore:"category"    json:"category"`
    Metadata    map[string]interface{} `firestore:"metadata"    json:"metadata"`
    Relation    Relation               `firestore:"relation"    json:"relation"`
}

type Relation struct {
    ParentType string `firestore:"parentType" json:"parentType"` // ENTITY | INVOICE | PROJECT
    ParentID   string `firestore:"parentId"   json:"parentId"`
}
```

### 3.1 Documento Firestore completo (post-finalize)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "fileName": "carta_identita_rossi.pdf",
  "mimeType": "application/pdf",
  "size": 102456,
  "storagePath": "entities/entity-abc/docs/550e8400.pdf",
  "category": "IDENTITY_DOC",
  "metadata": {},
  "relation": {
    "parentType": "ENTITY",
    "parentId": "entity-abc"
  },
  "accessControl": {
    "isPublic": false,
    "allowedRoles": ["admin"]
  },
  "createdAt": "2026-05-02T10:00:00Z",
  "createdBy": "uid_firebase_xyz"
}
```

### 3.2 Categorie documento (convenzione — non validato da enum)

| Category | Descrizione |
|---|---|
| `IDENTITY_DOC` | Documenti d'identità (CI, Passaporto) |
| `CONTRACT` | Contratti firmati |
| `INVOICE_ATTACHMENT` | Allegati a fatture |
| `LOGO` | Logo azienda/entità |

### 3.3 Path GCS

```
entities/{parentId}/docs/{docId}{ext}
```

Esempio: `entities/entity-abc/docs/550e8400.pdf`

---

## 4. GCS Client

**File:** `internal/storage/gcs.go`

Usa la libreria ufficiale `cloud.google.com/go/storage` con **Signed URL V4**.

```go
func (g *GCSClient) GenerateUploadURL(objectName, mimeType string, expiresInMinutes int) (string, error) {
    opts := &storage.SignedURLOptions{
        Scheme:      storage.SigningSchemeV4,
        Method:      "PUT",
        Expires:     time.Now().Add(time.Duration(expiresInMinutes) * time.Minute),
        ContentType: mimeType,
    }
    return g.client.Bucket(g.bucketName).SignedURL(objectName, opts)
}
```

**Parametri di firma:**

| Parametro | Valore | Note |
|---|---|---|
| Schema | `V4` | Più sicuro di V2 |
| Metodo HTTP | `PUT` | Il frontend carica con PUT diretto su GCS |
| Scadenza | **15 minuti** | Hardcoded nell'handler — non configurabile da env |
| Content-Type | Dinamico | Verificato da GCS, impedisce sostituzioni di tipo |

> ⚠️ **Nota produzione:** La firma V4 con ADC (Application Default Credentials) richiede che il Service Account del Cloud Run abbia il ruolo `roles/iam.serviceAccountTokenCreator` su se stesso, oppure che sia presente un JSON key file.

---

## 5. Handlers

**File:** `internal/api/handlers.go`

### 5.1 `HandleGetUploadURL` — Step 1

Genera un UUID come `docId`, costruisce il path GCS e restituisce il Signed URL al frontend.

**Request `UploadURLRequest`:**

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `fileName` | `string` | ✅ | Usato per estrarre l'estensione |
| `mimeType` | `string` | ✅ | Verificato da GCS all'upload |
| `category` | `string` | ✅ | Es. `IDENTITY_DOC`, `CONTRACT` |
| `parentType` | `string` | ✅ | `ENTITY` \| `INVOICE` \| `PROJECT` |
| `parentId` | `string` | ✅ | ID della risorsa padre |

**Risposta:**
```json
{
  "success": true,
  "data": {
    "docId": "550e8400-e29b-41d4-a716-446655440000",
    "uploadUrl": "https://storage.googleapis.com/bucket/entities/...?X-Goog-Signature=...",
    "storagePath": "entities/entity-abc/docs/550e8400.pdf"
  }
}
```

### 5.2 `HandleFinalizeUpload` — Step 2

Dopo che il frontend ha completato l'upload diretto su GCS, salva i metadati su Firestore nella collezione `documents`.

**Request `FinalizeRequest`:**

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `docId` | `string` | ✅ | UUID ricevuto da `upload-url` |
| `fileName` | `string` | ✅ | |
| `mimeType` | `string` | ✅ | |
| `size` | `int64` | ✅ | Dimensione in byte |
| `storagePath` | `string` | ✅ | Path GCS ricevuto da `upload-url` |
| `category` | `string` | ✅ | |
| `parentType` | `string` | ✅ | |
| `parentId` | `string` | ✅ | |

**Risposta:**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "FINALIZED"
  }
}
```

> ⚠️ **`accessControl` default:** Al momento della finalizzazione viene impostato `isPublic: false`, `allowedRoles: ["admin"]`. Non è ancora personalizzabile via API.

---

## 6. API Endpoints

Base path: `/api/v1` — tutti protetti da `nexus.Guard()`.

| Metodo | Path | Handler | Auth |
|---|---|---|---|
| `POST` | `/api/v1/documents/upload-url` | `HandleGetUploadURL` | ✅ Guard |
| `POST` | `/api/v1/documents/finalize` | `HandleFinalizeUpload` | ✅ Guard |
| `GET` | `/_discover` | `nexus.RegisterDiscovery` | ❌ Pubblico |

---

## 7. Bootstrap (`main.go`)

Sequenza di inizializzazione:

1. `godotenv.Load()` — carica `.env` (no-op in Cloud Run)
2. `firestore.NewClient(ctx, projectID)` — connessione Firestore
3. `repository.NewRepository(fsClient, "documents")` — ORM SDK sulla collezione `documents`
4. `storage.NewGCSClient(ctx, bucketName)` — client GCS reale
5. `api.NewDocumentHandler(docRepo, gcsClient)` — iniezione dipendenze
6. `gin.Default()` + `nexus.RegisterDiscovery()` — setup router
7. `apiGroup.Use(nexus.Guard())` — middleware auth su tutte le rotte `/api/v1`
8. `nexus.StartGatewayHandshake(serviceDef)` — registrazione al Gateway

### Mock GCS residuo

In `main.go` è presente un mock non utilizzato:

```go
type mockGCS struct{}
func (m *mockGCS) GenerateUploadURL(...) (string, error) {
    return "https://storage.googleapis.com/test-bucket/" + objectName + "?signed=true", nil
}
```

Il mock **non viene usato** — il `main()` istanzia correttamente il client reale. Va rimosso o spostato nei test.

---

## 8. Service Discovery

```go
serviceDef := nexus.ServiceDefinition{
    ServiceName: "document-service",
    Version:     "1.0.0",
    Endpoints:   []nexus.Endpoint{},  // ← VUOTO
}
```

> 🔴 **Issue critico:** `Endpoints` è un array vuoto. Il Gateway registra il servizio ma non conosce i suoi endpoint. Va completata con i due endpoint reali.

---

## 9. Variabili d'Ambiente

| Variabile | Descrizione | Obbligatoria |
|---|---|---|
| `GCP_PROJECT_ID` | Project ID GCP per Firestore | ✅ Sì |
| `FIREBASE_STORAGE_BUCKET` | Nome bucket GCS (es. `ssg-nexus.firebasestorage.app`) | ✅ Sì |
| `GATEWAY_URL` | URL del Gateway per l'handshake Discovery | ✅ Sì |
| `SERVICE_URL` | URL Cloud Run di questo servizio | ✅ Sì |
| `INTERNAL_SECRET` | Token condiviso Gateway ↔ Servizio | ✅ Sì |
| `PORT` | Porta HTTP (default `8080`) | No |

---

## 10. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | `ServiceDefinition.Endpoints` vuoto — il Gateway non riceve la mappa degli endpoint | ⏳ Aperto |
| 🔴 Alta | Mancano `GET /documents?parentId=` (lista) e `GET /documents/:id/download-url` | ⏳ Mancante |
| 🟡 Media | `accessControl.allowedRoles` fisso a `["admin"]` — non personalizzabile via API | ⏳ Aperto |
| 🟡 Media | Nessuna verifica che l'upload GCS sia avvenuto prima della finalizzazione | ⏳ Aperto |
| 🟡 Media | Mock GCS inutilizzato in `main.go` — va rimosso o spostato nei test | ⏳ Aperto |
| 🟢 Bassa | Scadenza Signed URL hardcoded a 15 min — non configurabile da env | ⏳ Aperto |
| 🟢 Bassa | Nessun soft-delete implementato (`deletedAt` previsto da `NexusDoc`) | ⏳ Aperto |
| 🟢 Bassa | Nessun test unitario o di integrazione presente | ⏳ Aperto |

---

## 11. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/DaniFX/ssg-nexus-sdk` | Guard, Repository ORM, Discovery, Response standard |
| `cloud.google.com/go/storage` | Client GCS per Signed URL V4 |
| `cloud.google.com/go/firestore` | Persistenza metadati documenti |
| `github.com/gin-gonic/gin` | HTTP framework |
| `github.com/google/uuid` | Generazione `docId` univoci |
| `github.com/joho/godotenv` | Caricamento `.env` in sviluppo locale |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
