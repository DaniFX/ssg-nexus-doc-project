# 🏢 SSG Registry Service — Specifiche Tecniche

> **Repository:** [`ssg-registry-service`](https://github.com/DaniFX/ssg-registry-service)
> **Stack:** Go 1.21+, Gin, Firestore (GCP), `ssg-nexus-sdk`
> **Ruolo:** Custode delle **Anagrafiche Polimorfiche** (Soci, Clienti, Prospect, Fornitori, Dipendenti). Servizio core dell'ERP: ogni entità che interagisce con SSG Nexus viene registrata qui.

---

## 1. Principio Architetturale: Anagrafica Polimorfica

Un singolo documento Firestore descrive qualsiasi tipo di soggetto nel sistema. Il campo `type` (`PERSON` | `ORGANIZATION`) definisce la struttura rigida; `subTypes` (es. `["MEMBER", "CUSTOMER"]`) aggiunge capacità multiple alla stessa entità. I dati obbligatori fiscali stanno in `coreData`; i dati specifici per sottotipo (es. `membershipDate`, `billingAddress`) vanno in `extData` (schema-flexible).

```
                        +----------------------------+
                        |       entities             |  <-- Collezione Firestore
                        |----------------------------|  
  PERSON + MEMBER  -->  |  id: uuid                  |
  ORGANIZATION +   -->  |  type: PERSON | ORGANIZATION|
    CUSTOMER           |  subTypes: [MEMBER, ...]    |
                        |  status: ACTIVE | INACTIVE  |
                        |  coreData: { ... }          |  <-- dati fiscali (strict)
                        |  extData:  { ... }          |  <-- dati per subType (flex)
                        |  createdAt, updatedAt       |  <-- iniettati da NexusDoc SDK
                        |  createdBy                  |  <-- identity.UserID dal Guard
                        +----------------------------+
```

---

## 2. Struttura del Progetto

```
ssg-registry-service/
├── cmd/
│   └── registry/
│       └── main.go             # Bootstrap: Firestore, Repository, Routes, Discovery, Guard
├── internal/
│   ├── handlers/
│   │   └── entity.go           # EntityHandler: Create (altri handler commentati)
│   ├── models/
│   │   └── entity.go           # Struct Entity con embedding NexusDoc
│   └── repository/
│       └── firestore.go        # InitFirestore() — connessione Firestore via GCP_PROJECT_ID
└── Dockerfile
```

---

## 3. Modello Dati

**File:** `internal/models/entity.go`

```go
type Entity struct {
    repository.NexusDoc                        // createdAt, updatedAt, deletedAt, createdBy
    Type     string                 `json:"type"     firestore:"type"`     // PERSON | ORGANIZATION
    SubTypes []string               `json:"subTypes" firestore:"subTypes"` // MEMBER, CUSTOMER, SUPPLIER...
    Status   string                 `json:"status"   firestore:"status"`   // ACTIVE | INACTIVE | PROSPECT
    CoreData map[string]interface{} `json:"coreData" firestore:"coreData"` // dati fiscali obbligatori
    ExtData  map[string]interface{} `json:"extData"  firestore:"extData"`  // dati specifici per subType
}
```

### 3.1 `coreData` — Dati fiscali (convenzione, non validata da struct Go)

```json
{
  "displayName": "Mario Rossi",
  "email":       "mario@example.com",
  "taxCode":     "RSSMRA80A01H501U",
  "vatNumber":   "IT01234567890"
}
```

> **Regola ERP:** Se `type == ORGANIZATION`, il campo `coreData.vatNumber` è obbligatorio. La validazione **non è ancora implementata** a livello di codice (vedi Issue 🔴 in sezione 9).

### 3.2 `extData` per subType — Esempi

| SubType | Campi tipici in `extData` |
|---|---|
| `MEMBER` | `membershipDate`, `membershipNumber`, `tshirtSize` |
| `CUSTOMER` | `billingAddress`, `paymentTerms`, `creditLimit` |
| `SUPPLIER` | `iban`, `defaultPaymentDays`, `category` |
| `EMPLOYEE` | `hireDate`, `jobTitle`, `department` |

### 3.3 `NexusDoc` — Metadati ERP iniettati dall'SDK

| Campo | Tipo | Chi lo imposta |
|---|---|---|
| `createdAt` | `time.Time` | `nexusRepo.Create()` automaticamente |
| `updatedAt` | `time.Time` | `nexusRepo.Update()` automaticamente |
| `deletedAt` | `*time.Time` | Soft-delete (non ancora usato nel registry) |
| `createdBy` | `string` | `identity.UserID` estratto da `nexus.FromContext(ctx)` |

---

## 4. Flusso Completo: Create Entity

**File:** `internal/handlers/entity.go`

```
POST /api/v1/registry/entities
  |
  | [1] nexus.Guard() verifica X-Nexus-User-ID
  |     nexus.FromContext(ctx) -> identity.UserID
  |
  | [2] c.ShouldBindJSON(&payload) -> models.Entity
  |     Errore -> nexus.Failure(400, ErrValidationFailed)
  |
  | [3] entityID = uuid.New().String()
  |
  | [4] data = map[string]interface{}{
  |       "type", "subTypes", "status", "coreData", "extData"
  |     }
  |
  | [5] h.Repo.Create(ctx, entityID, data)
  |     -> SDK inietta createdAt, updatedAt, createdBy
  |     -> Firestore.Set("entities", entityID, data)
  |     Errore -> nexus.Failure(500, ErrInternal)
  |
  | [6] data["id"] = entityID
  |     nexus.Success(c, data, gin.H{"insertedBy": identity.UserID})
  v
200 OK
```

---

## 5. Repository Layer: Nexus ORM

Il servizio usa **`nexusRepo.Repository`** dall'SDK invece di chiamare Firestore direttamente. Questo garantisce l'iniezione automatica dei metadati standard su ogni write.

```go
// main.go — bootstrap
entityRepo := nexusRepo.NewRepository(firestoreClient, "entities")
```

| Metodo SDK | Descrizione |
|---|---|
| `Create(ctx, id, data)` | Crea documento con `createdAt`, `updatedAt`, `createdBy` auto-iniettati |
| `Update(ctx, id, data)` | Aggiorna con `updatedAt` automatico |
| `GetByID(ctx, id)` | Fetch singolo documento |
| `List(ctx, filters)` | Query con filtri (Navigator pattern) |
| `SoftDelete(ctx, id)` | Imposta `deletedAt` senza rimuovere il documento |

> **Nota:** `InitFirestore()` in `internal/repository/firestore.go` effettua `log.Fatalf` se `GCP_PROJECT_ID` non è impostata — il servizio non si avvia senza questa variabile.

---

## 6. Service Discovery

**Definita in:** `cmd/registry/main.go` tramite `nexus.RegisterDiscovery()` e `nexus.StartGatewayHandshake()`.

Servizio: `registry-service` | Versione: `1.0.0`

| Metodo | Path | Auth | Stato |
|---|---|---|---|
| `POST` | `/api/v1/registry/entities` | ✅ | ✅ Attivo |
| `GET` | `/api/v1/registry/entities` | ✅ | 💤 Commentato in `main.go` |
| `GET` | `/api/v1/registry/entities/:id` | ✅ | 💤 Commentato in `main.go` |
| `PATCH` | `/api/v1/registry/entities/:id` | ✅ | 💤 Commentato in `main.go` |
| `GET` | `/_discover` | ❌ No | ✅ Fuori dal Guard |

> ⚠️ La `ServiceDefinition` in `main.go` dichiara solo `POST /entities`. Gli endpoint commentati non sono comunicati al Gateway.

---

## 7. API: `POST /api/v1/registry/entities`

**Body:**
```json
{
  "type": "ORGANIZATION",
  "subTypes": ["CUSTOMER", "SUPPLIER"],
  "status": "ACTIVE",
  "coreData": {
    "displayName": "Acme S.r.l.",
    "email": "info@acme.it",
    "taxCode": "01234567890",
    "vatNumber": "IT01234567890"
  },
  "extData": {
    "billingAddress": "Via Roma 1, Milano",
    "paymentTerms": "30gg"
  }
}
```

**Risposta `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "type": "ORGANIZATION",
    "subTypes": ["CUSTOMER", "SUPPLIER"],
    "status": "ACTIVE",
    "coreData": { "..." },
    "extData":  { "..." }
  },
  "meta": {
    "insertedBy": "uid_firebase_xyz"
  }
}
```

> **Nota HTTP status:** L'handler usa `nexus.Success()` che restituisce `200` (non `201`). Questo è allineato allo standard Nexus SDK per tutti i servizi.

**Errori:**

| Codice HTTP | Chiave Nexus | Causa |
|---|---|---|
| `400` | `VALIDATION_FAILED` | JSON malformato o struct binding fallito |
| `500` | `INTERNAL` | Errore scrittura Firestore |

---

## 8. Variabili d'Ambiente

| Variabile | Descrizione | Obbligatoria |
|---|---|---|
| `GCP_PROJECT_ID` | Project ID GCP per Firestore | ✅ Fatal se assente |
| `GATEWAY_URL` | URL del Gateway per l'handshake Discovery | ✅ Sì |
| `SERVICE_URL` | URL Cloud Run di questo servizio | ✅ Sì |
| `INTERNAL_SECRET` | Token condiviso Gateway ↔ Servizio | ✅ Sì |
| `PORT` | Porta HTTP (hardcoded `8080` in `r.Run`) | No |

> **Nota `PORT`:** Il Registry usa `r.Run(":8080")` senza leggere `os.Getenv("PORT")`, comportamento non uniforme rispetto agli altri servizi Nexus.

---

## 9. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | Solo `POST /entities` attivo — `GET`, `PATCH`, soft-delete commentati in `main.go` | ⏳ Aperto |
| 🔴 Alta | Validazione `vatNumber` obbligatorio se `type == ORGANIZATION` non implementata a runtime | ⏳ Aperto |
| 🔴 Alta | Unicità `coreData.taxCode` non verificata prima della scrittura (possibili duplicati anagrafica) | ⏳ Aperto |
| 🟡 Media | `PORT` hardcoded a `8080` in `r.Run()` — non legge `os.Getenv("PORT")` come gli altri servizi | ⏳ Aperto |
| 🟡 Media | Prima del soft-delete, verificare con Finance Service assenza di fatture `PENDING` | ⏳ Non implementato |
| 🟡 Media | Logica permessi a "volumi": visibilità pubblica (solo `displayName`) vs gestionale (dati fiscali) | ⏳ Progettata, non implementata |
| 🟢 Bassa | Nessun test unitario o di integrazione nel repo | ⏳ Aperto |

---

## 10. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/DaniFX/ssg-nexus-sdk` | Guard, Repository ORM, Discovery, Response standard, `nexus.FromContext` |
| `cloud.google.com/go/firestore` | Persistenza dati entità |
| `github.com/gin-gonic/gin` | HTTP framework |
| `github.com/google/uuid` | Generazione ID univoci per le entità |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
