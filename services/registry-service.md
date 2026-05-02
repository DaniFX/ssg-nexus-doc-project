# 🏢 SSG Registry Service — Specifiche Tecniche

> **Repository:** [`ssg-registry-service`](https://github.com/DaniFX/ssg-registry-service)
> **Stack:** Go 1.21+, Gin, Firestore (GCP), `ssg-nexus-sdk`
> **Ruolo:** Custode delle **Anagrafiche Polimorfiche** (Soci, Clienti, Prospect, Fornitori, Dipendenti). Servizio core dell’ERP: ogni entità che interagisce con SSG Nexus viene registrata qui.

---

## 1. Principio Architetturale: Anagrafica Polimorfica

Un singolo documento Firestore descrive qualsiasi tipo di soggetto nel sistema. Il campo `type` (`PERSON` | `ORGANIZATION`) definisce la struttura rigida; `subTypes` (es. `["MEMBER", "CUSTOMER"]`) aggiunge capacità multiple alla stessa entità. I dati obbligatori fiscali stanno in `coreData`; i dati specifici per sottotipo (es. `membershipDate`, `billingAddress`) vanno in `extData` (schema-flexible).

```
                        +-------------------------+
                        |      entities           |  <-- Collezione Firestore
                        |-------------------------|  
  PERSON + MEMBER  -->  |  id: uuid               |
  ORGANIZATION +   -->  |  type: PERSON|ORG       |
    CUSTOMER          |  subTypes: [...]          |
                        |  status: ACTIVE|INACTIVE |
                        |  coreData: { ... }      |  <-- dati fiscali (strict)
                        |  extData:  { ... }      |  <-- dati per subType (flex)
                        |  createdAt, updatedAt   |  <-- iniettati da NexusDoc
                        |  createdBy              |  <-- identity.UserID dal Guard
                        +-------------------------+
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
│   │   └── entity.go           # Struct Entity che embedding NexusDoc
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

### 3.1 Struttura `coreData` (convenzione, non validata da struct Go)

```json
{
  "displayName": "Mario Rossi / Acme S.r.l.",
  "email": "mario@example.com",
  "taxCode": "RSSMRA80A01H501U",
  "vatNumber": "IT01234567890"
}
```

> **Regola ERP:** Se `type == ORGANIZATION`, il campo `coreData.vatNumber` è obbligatorio. La validazione non è ancora implementata a livello di struct (vedi Issue 🔴 in sezione 9).

### 3.2 Esempi `extData` per subType

| SubType | Campi tipici in `extData` |
|---|---|
| `MEMBER` | `membershipDate`, `membershipNumber`, `tshirtSize` |
| `CUSTOMER` | `billingAddress`, `paymentTerms`, `creditLimit` |
| `SUPPLIER` | `iban`, `defaultPaymentDays`, `category` |
| `EMPLOYEE` | `hireDate`, `jobTitle`, `department` |

### 3.3 `NexusDoc` — Metadati ERP (ereditati dall’SDK)

| Campo | Tipo | Iniettato da |
|---|---|---|
| `createdAt` | `time.Time` | `nexusRepo.Create()` |
| `updatedAt` | `time.Time` | `nexusRepo.Update()` |
| `deletedAt` | `*time.Time` | Soft-delete (non ancora implementato) |
| `createdBy` | `string` | `identity.UserID` da `nexus.FromContext()` |

---

## 4. Autenticazione e Guard

Il servizio usa **solo `nexus.Guard()`** — nessun middleware aggiuntivo oltre allo standard Nexus. La catena è:

```
Gateway
  |
  | X-Nexus-User-ID: <firebase_uid>
  | X-Nexus-Role:    <role>
  | X-Nexus-Trace-ID: <trace>
  v
+-------------------------------+
|     Registry Service          |
|  nexus.Guard()                |
|    -> nexus.FromContext(ctx)  |
|    -> identity.UserID         |
|  EntityHandler.Create()       |
|    -> nexusRepo.Create()      |
|    -> Firestore: "entities"   |
+-------------------------------+
```

`nexus.FromContext(ctx)` estrae l’identità iniettata dal Guard e la rende disponibile all’handler per valorizzare `createdBy` automaticamente. [cite:96]

---

## 5. Repository Layer: Nexus ORM

Il servizio usa **`nexusRepo.Repository`** dall’SDK invece di chiamare Firestore direttamente. Questo garantisce l’iniezione automatica dei metadati standard (`createdAt`, `updatedAt`, `createdBy`) su ogni write.

```go
// main.go — bootstrap
entityRepo := nexusRepo.NewRepository(firestoreClient, "entities")
// La collezione Firestore target è "entities"
```

```go
// handler — create
err := h.Repo.Create(ctx, entityID, data)
// Automaticamente aggiunge: createdAt, updatedAt, createdBy (da context)
```

**Operazioni disponibili nell’SDK** (da `ssg-nexus-sdk`):

| Metodo | Descrizione |
|---|---|
| `Create(ctx, id, data)` | Crea documento con metadati ERP auto-iniettati |
| `Update(ctx, id, data)` | Aggiorna con `updatedAt` automatico |
| `GetByID(ctx, id)` | Fetch singolo documento |
| `List(ctx, filters)` | Query con filtri (Navigator pattern) |
| `SoftDelete(ctx, id)` | Imposta `deletedAt` senza rimuovere il documento |

---

## 6. Service Discovery

**Definita in:** `cmd/registry/main.go`

Servizio: `registry-service` | Versione: `1.0.0`

Endpoint registrati nel discovery (contratto verso il Gateway):

| Metodo | Path | Auth | Summary |
|---|---|---|---|
| `POST` | `/api/v1/registry/entities` | ✅ Richiesta | Crea una nuova entità nel registro |
| `GET` | `/api/v1/registry/entities` | ✅ | Lista filtrabile *(commentato, non ancora attivo)* |
| `GET` | `/api/v1/registry/entities/:id` | ✅ | Dettaglio entità *(commentato)* |
| `PATCH` | `/api/v1/registry/entities/:id` | ✅ | Aggiornamento parziale *(commentato)* |

> ⚠️ Solo `POST /entities` è attualmente attivo. Gli altri endpoint sono commentati in `main.go`.

Endpoint esposto per ispezione: `GET /_discover` (fuori dal Guard)

---

## 7. API Endpoints

### `POST /api/v1/registry/entities`

Crea una nuova entità nel registro anagrafiche.

**Header richiesti:**
- `X-Nexus-User-ID` (iniettato dal Gateway)

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

**Risposta `201 Created`:**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "type": "ORGANIZATION",
    "subTypes": ["CUSTOMER", "SUPPLIER"],
    "status": "ACTIVE",
    "coreData": { "..." },
    "extData": { "..." }
  },
  "meta": {
    "insertedBy": "uid_firebase_xyz"
  }
}
```

**Errori:**

| Codice | Chiave | Causa |
|---|---|---|
| `400` | `VALIDATION_FAILED` | JSON malformato o campi obbligatori mancanti |
| `500` | `INTERNAL` | Errore scrittura Firestore |

---

## 8. Variabili d’Ambiente

| Variabile | Descrizione | Obbligatoria |
|---|---|---|
| `GCP_PROJECT_ID` | Project ID GCP per la connessione Firestore | ✅ Sì (fatal se assente) |
| `GATEWAY_URL` | URL del Gateway per l’handshake Discovery | ✅ Sì |
| `SERVICE_URL` | URL Cloud Run di questo servizio | ✅ Sì |
| `INTERNAL_SECRET` | Token condiviso Gateway ↔ Servizio | ✅ Sì |
| `PORT` | Porta HTTP (default `8080`) | No |

---

## 9. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | Solo `POST /entities` attivo — `GET`, `PATCH`, soft-delete commentati in `main.go` | ⏳ Aperto |
| 🔴 Alta | Validazione `vatNumber` obbligatorio se `type == ORGANIZATION` non implementata a runtime | ⏳ Aperto |
| 🔴 Alta | Unicità `coreData.taxCode` non verificata prima della scrittura (possibili duplicati) | ⏳ Aperto |
| 🟡 Media | Prima del soft-delete, verificare con Finance Service assenza di fatture `PENDING` | ⏳ Non implementato |
| 🟡 Media | Logica permessi a "volumi": visibilità pubblica (solo `displayName`) vs gestionale (dati fiscali) | ⏳ Progettata, non implementata |
| 🟢 Bassa | Nessun test unitario o di integrazione presente nel repo | ⏳ Aperto |

---

## 10. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/DaniFX/ssg-nexus-sdk` | Guard, Repository ORM, Discovery, Response standard |
| `cloud.google.com/go/firestore` | Persistenza dati |
| `github.com/gin-gonic/gin` | HTTP framework |
| `github.com/google/uuid` | Generazione ID univoci per le entità |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
