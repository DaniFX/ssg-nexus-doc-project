# 📂 SSG Document Service — Specifiche Tecniche

> **Repository:** [`ssg-nexus-document-service`](https://github.com/DaniFX/ssg-nexus-document-service)
> **Stack:** Go 1.21+, Gin, Google Cloud Storage (GCS), Firestore, `ssg-nexus-sdk`
> **Ruolo:** Gestisce il ciclo di vita dei file (upload, archiviazione, metadatazione, sicurezza). Ponte tra GCS (fisico) e Firestore (logico). Ogni documento è sempre associato a un’entità (Registry) o a una transazione (Finance) tramite il campo `relation`.

---

## 1. Principio Architetturale: Secure Upload via Signed URL

Il servizio non riceve mai il file binario direttamente. Il flusso usa un pattern a due step che scarica il traffico binario dal microservizio e lo dirige direttamente su GCS:

```
Frontend
  |
  | [1] POST /documents/upload-url
  |     { fileName, mimeType, category, parentType, parentId }
  v
+----------------------------------+
|   Document Service               |
|   HandleGetUploadURL()           |
|   -> genera docId (UUID)         |
|   -> path: entities/{parentId}/  |
|            docs/{docId}.ext      |
|   -> GCS.GenerateUploadURL()     |
|      (Signed URL V4, 15 min)     |
+----------------------------------+
  |
  | [2] Risposta: { docId, uploadUrl, storagePath }
  v
Frontend
  |
  | [3] PUT {uploadUrl} <file binario>
  |     (direttamente su GCS, senza passare per il servizio)
  v
Google Cloud Storage
  |
  | [4] POST /documents/finalize
  |     { docId, fileName, mimeType, size, storagePath, ... }
  v
+----------------------------------+
|   Document Service               |
|   HandleFinalizeUpload()         |
|   -> salva metadati su Firestore |
|   -> collezione "documents"      |
+----------------------------------+
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

### 3.2 Categorie documento (convenzione)

| Category | Descrizione |
|---|---|
| `IDENTITY_DOC` | Documenti d’identità (CI, Passaporto) |
| `CONTRACT` | Contratti firmati |
| `INVOICE_ATTACHMENT` | Allegati a fatture |
| `LOGO` | Logo azienda/entità |

> Il campo `category` non è validato da un enum nel codice — è una convenzione documentale.

### 3.3 Path GCS

Il percorso su Cloud Storage è costruito in `HandleGetUploadURL()` con questo schema:

```
entities/{parentId}/docs/{docId}{ext}
```

Esempio: `entities/entity-abc/docs/550e8400.pdf`

---

## 4. GCS Client

**File:** `internal/storage/gcs.go`

Usa la libreria ufficiale `cloud.google.com/go/storage` con **Signed URL V4** (metodo `PUT`, scadenza configurabile).

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
- Schema: **V4** (più sicuro di V2)
- Metodo HTTP: **PUT** (il frontend carica con PUT diretto su GCS)
- Scadenza: **15 minuti** (hardcoded nella chiamata dall’handler)
- Content-Type: verificato da GCS al momento dell’upload (impedisce sostituzioni di tipo)

> ⚠️ **Nota produzione:** La firma V4 richiede che il Service Account del Cloud Run abbia il ruolo `roles/iam.serviceAccountTokenCreator` su se stesso, oppure che venga usato un JSON key file. In Cloud Run con ADC (Application Default Credentials) questo funziona automaticamente se il SA ha i permessi corretti.

---

## 5. Handlers

**File:** `internal/api/handlers.go`

### 5.1 `HandleGetUploadURL` — Step 1

Riceve i metadati del file, genera un UUID per il documento, costruisce il path GCS e restituisce il Signed URL.

**Request struct `UploadURLRequest`:**

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `fileName` | `string` | ✅ | Usato per estrarre l’estensione |
| `mimeType` | `string` | ✅ | Verificato da GCS all’upload |
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

Dopo che il frontend ha completato l’upload diretto su GCS, chiama questo endpoint per salvare i metadati su Firestore. Il `docId` deve corrispondere a quello ottenuto nello step 1.

**Request struct `FinalizeRequest`:**

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `docId` | `string` | ✅ | UUID ricevuto da upload-url |
| `fileName` | `string` | ✅ | |
| `mimeType` | `string` | ✅ | |
| `size` | `int64` | ✅ | Dimensione in byte |
| `storagePath` | `string` | ✅ | Path GCS ricevuto da upload-url |
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

> **Nota `accessControl`:** Al momento della finalizzazione, il campo `accessControl` viene impostato con un default prudenziale: `isPublic: false`, `allowedRoles: ["admin"]`. Non è ancora possibile personalizzarlo via API.

---

## 6. API Endpoints

Base path: `/api/v1` (senza prefisso `document` nel path, a differenza degli altri servizi)

Tutti gli endpoint richiedono `nexus.Guard()` attivo.

| Metodo | Path | Handler | Descrizione |
|---|---|---|---|
| `POST` | `/api/v1/documents/upload-url` | `HandleGetUploadURL` | Genera Signed URL GCS per upload diretto |
| `POST` | `/api/v1/documents/finalize` | `HandleFinalizeUpload` | Salva metadati su Firestore post-upload |
| `GET` | `/_discover` | `nexus.RegisterDiscovery` | Service Discovery (fuori dal Guard) |

> ⚠️ **Attenzione:** la `ServiceDefinition` in `main.go` ha `Endpoints: []nexus.Endpoint{}` — lista vuota. Il Gateway non riceve descrizione degli endpoint tramite handshake. Va completata.

---

## 7. Service Discovery

**Definita in:** `cmd/document-service/main.go`

Servizio: `document-service` | Versione: `1.0.0`

> 🔴 **Issue critico:** `Endpoints` è un array vuoto (`[]nexus.Endpoint{}`). Il Gateway registra il servizio ma non conosce i suoi endpoint. La discovery va completata con i due endpoint reali.

---

## 8. Mock GCS in `main.go`

In `main.go` è presente un **mock temporaneo** del GCS client:

```go
type mockGCS struct{}
func (m *mockGCS) GenerateUploadURL(objectName, mimeType string, expiresIn int) (string, error) {
    return "https://storage.googleapis.com/test-bucket/" + objectName + "?signed=true", nil
}
```

Il mock **non viene più usato in produzione** — il `main()` istanzia correttamente `storage.NewGCSClient()`. Il mock rimane nel codice ma è inutilizzato. Va rimosso o spostato nei test.

---

## 9. Variabili d’Ambiente

| Variabile | Descrizione | Obbligatoria |
|---|---|---|
| `GCP_PROJECT_ID` | Project ID GCP per Firestore | ✅ Sì |
| `FIREBASE_STORAGE_BUCKET` | Nome bucket GCS (es. `ssg-nexus.firebasestorage.app`) | ✅ Sì |
| `GATEWAY_URL` | URL del Gateway per l’handshake Discovery | ✅ Sì |
| `SERVICE_URL` | URL Cloud Run di questo servizio | ✅ Sì |
| `INTERNAL_SECRET` | Token condiviso Gateway ↔ Servizio | ✅ Sì |
| `PORT` | Porta HTTP (default `8080`) | No |

---

## 10. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | `ServiceDefinition.Endpoints` vuoto — il Gateway non riceve la mappa degli endpoint | ⏳ Aperto |
| 🔴 Alta | Mancano `GET /documents` (lista per `parentId`) e `GET /documents/:id/download-url` | ⏳ Mancante |
| 🟡 Media | `accessControl.allowedRoles` fisso a `["admin"]` — non personalizzabile via API | ⏳ Aperto |
| 🟡 Media | Nessuna verifica che l’upload GCS sia avvenuto prima della finalizzazione | ⏳ Aperto |
| 🟡 Media | Mock GCS inutilizzato in `main.go` — va rimosso o spostato nei test | ⏳ Aperto |
| 🟢 Bassa | Scadenza Signed URL hardcoded a 15 min — non configurabile da env | ⏳ Aperto |
| 🟢 Bassa | Nessun soft-delete implementato | ⏳ Aperto |
| 🟢 Bassa | Nessun test unitario o di integrazione presente nel repo | ⏳ Aperto |

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
