# 💰 SSG Finance Service — Specifiche Tecniche

> **Repository:** [`ssg-finance-service`](https://github.com/DaniFX/ssg-finance-service)
> **Stack:** Go 1.21+, Gin, Firestore, `ssg-nexus-sdk`
> **Ruolo:** Modulo ERP di SSG Nexus. Gestisce il ciclo di vita delle fatture (DRAFT → ISSUED → PAID), la registrazione dei pagamenti nel libro giornale e la riconciliazione automatica.

---

## 1. Struttura del Progetto

```
ssg-finance-service/
├── cmd/
│   └── finance-service/
│       └── main.go              # Bootstrap: Firestore, FinanceService, Routes
├── internal/
│   ├── handlers/
│   │   ├── invoices.go          # IssueInvoice (PATCH /invoices/:id/issue)
│   │   ├── ledger.go            # RegisterTransaction (POST /ledger)
│   │   └── discovery.go        # GetDiscovery (GET /_discover)
│   ├── models/
│   │   ├── invoice.go           # Invoice, InvoiceDates, Totals, Entity, InvoiceStatus
│   │   └── ledger.go            # LedgerEntry
│   ├── repository/
│   │   └── firestore.go         # FinanceRepository: GetInvoice, UpdateInvoice, SaveLedgerEntry, GetTotalPaidForInvoice
│   └── services/
│       └── finance_service.go   # FinanceService: UpdateInvoice, IssueInvoice, RegisterPayment, generateDocument
└── Dockerfile
```

---

## 2. Modelli Dati

### 2.1 Invoice — `internal/models/invoice.go`

```go
type InvoiceStatus string

const (
    StatusDraft     InvoiceStatus = "DRAFT"
    StatusIssued    InvoiceStatus = "ISSUED"
    StatusPaid      InvoiceStatus = "PAID"
    StatusCancelled InvoiceStatus = "CANCELLED"
)

type Invoice struct {
    ID          string         `json:"id"          firestore:"id"`
    ExternalID  string         `json:"externalId"  firestore:"externalId"`  // es. SDI-2026-abc123
    Type        string         `json:"type"        firestore:"type"`
    Status      InvoiceStatus  `json:"status"      firestore:"status"`
    Issuer      Entity         `json:"issuer"      firestore:"issuer"`      // snapshot da Registry
    Receiver    Entity         `json:"receiver"    firestore:"receiver"`    // snapshot da Registry
    Totals      Totals         `json:"totals"      firestore:"totals"`
    Dates       InvoiceDates   `json:"dates"       firestore:"dates"`
    DocumentRef string         `json:"documentRef" firestore:"documentRef"` // ID doc nel Document Service
    Metadata    map[string]any `json:"metadata"    firestore:"metadata"`    // "immutable": true dopo ISSUED
}

type InvoiceDates struct {
    Document time.Time  `json:"document" firestore:"document"`
    Due      time.Time  `json:"due"      firestore:"due"`
    Paid     *time.Time `json:"paid"     firestore:"paid"` // nil finché non pagata
}

type Totals struct {
    Gross    float64 `json:"gross"    firestore:"gross"`
    Currency string  `json:"currency" firestore:"currency"`
}

type Entity struct {
    EntityID string `json:"entityId" firestore:"entityId"`
    Name     string `json:"name"     firestore:"name"`
    VAT      string `json:"vat"      firestore:"vat"`
}
```

> ⚠️ **Snapshot fiscale:** `Issuer` e `Receiver` sono copie statiche dei dati Registry al momento dell'emissione. Non si aggiornano se i dati dell'entità cambiano in seguito.

> ⚠️ **Locking ERP:** Dopo `IssueInvoice()`, il campo `metadata["immutable"] = true` viene impostato nel codice Go. `UpdateInvoice()` controlla questo flag ed emette errore se la fattura è già emessa.

### 2.2 LedgerEntry — `internal/models/ledger.go`

```go
type LedgerEntry struct {
    ID        string    `json:"id"        firestore:"id"`
    EntityID  string    `json:"entityId"  firestore:"entityId"`
    InvoiceID string    `json:"invoiceId" firestore:"invoiceId"`
    Amount    float64   `json:"amount"    firestore:"amount"`
    Type      string    `json:"type"      firestore:"type"`   // DEBIT | CREDIT
    Method    string    `json:"method"    firestore:"method"` // STRIPE | BANK_TRANSFER
    Timestamp time.Time `json:"timestamp" firestore:"timestamp"`
}
```

> `Timestamp` è impostato automaticamente dal service layer (`time.Now()`), non dal client.

---

## 3. Logica di Business — `internal/services/finance_service.go`

Il `FinanceService` contiene tutta la business logic. Gli handler sono thin wrapper che delegano a questo layer.

### 3.1 `IssueInvoice` — Ciclo di vita DRAFT → ISSUED

```
DRAFT
  |
  | [1] Verifica: status != ISSUED e metadata["immutable"] != true
  | [2] Genera ExternalID: "SDI-{anno}-{id[:6]}"
  | [3] HTTP POST al Document Service: /api/v1/documents/generate
  |     Payload: { "type": "INVOICE", "data": <invoice> }
  |     Risposta attesa: { "data": { "documentId": "..." } }
  | [4] Imposta DocumentRef, Status = ISSUED, metadata["immutable"] = true
  | [5] UpdateInvoice su Firestore
  v
ISSUED (immutabile)
```

**Errore codice:** `ERP_LOCK_ERROR` (HTTP 400) se la fattura è già emessa o già immutabile.

> ⚠️ **Issue critico:** Il Finance Service chiama `POST /api/v1/documents/generate` sul Document Service, ma questo endpoint **non esiste** nel `ssg-nexus-document-service` attuale. Il Document Service espone solo `upload-url` e `finalize`. Vanno allineati.

### 3.2 `RegisterPayment` — Registrazione e Riconciliazione Automatica

```
POST /ledger
  |
  | [1] Imposta entry.Timestamp = time.Now()
  | [2] SaveLedgerEntry su Firestore (collection: ledger_entries)
  | [3] GetTotalPaidForInvoice: somma tutti i ledger_entries per invoiceId
  | [4] GetInvoice per recuperare totals.gross
  | [5] Se totalPaid >= invoice.Totals.Gross → Status = PAID, Dates.Paid = now
  | [6] UpdateInvoice su Firestore
  v
PAID (se soglia raggiunta)
```

**Riconciliazione atomica lato applicazione:** il passaggio a PAID avviene nello stesso goroutine del pagamento, senza lock distribuiti. In caso di scrittura concorrente possono verificarsi race condition.

### 3.3 `UpdateInvoice` — Protezione DRAFT

Permette modifiche solo se la fattura è in stato `DRAFT` e `metadata["immutable"]` è falso. Usato per aggiornamenti pre-emissione.

---

## 4. Repository — `internal/repository/firestore.go`

| Metodo | Collection | Operazione |
|---|---|---|
| `GetInvoice(ctx, id)` | `invoices` | `.Doc(id).Get()` |
| `UpdateInvoice(ctx, inv)` | `invoices` | `.Doc(inv.ID).Set()` (upsert) |
| `SaveLedgerEntry(ctx, entry)` | `ledger_entries` | `.Doc(entry.ID).Set()` — ID auto-generato da Firestore se vuoto |
| `GetTotalPaidForInvoice(ctx, invoiceID)` | `ledger_entries` | `.Where("invoiceId", "==", id)` + somma Amount |

> **Nota:** `UpdateInvoice` usa `.Set()` (upsert), non `.Update()` (patch). Sovrascrive l'intero documento — attenzione a non perdere campi non inclusi nella struct.

---

## 5. API Endpoints

| Metodo | Path | Handler | Descrizione |
|---|---|---|---|
| `PATCH` | `/api/v1/invoices/:id/issue` | `IssueInvoice` | DRAFT → ISSUED, lock immutabile, genera PDF via Document Service |
| `POST` | `/api/v1/ledger` | `RegisterTransaction` | Registra pagamento + riconciliazione automatica |
| `GET` | `/_discover` | `GetDiscovery` | Service Discovery (fuori dal Guard) |

### 5.1 `PATCH /api/v1/invoices/:id/issue`

**Nessun body richiesto.** L'ID viene letto dal path param.

**Risposta successo (200):**
```json
{
  "success": true,
  "data": {
    "status": "ISSUED",
    "message": "Fattura emessa e bloccata con successo"
  }
}
```

**Risposta errore (400):**
```json
{
  "success": false,
  "error": {
    "code": "ERP_LOCK_ERROR",
    "message": "il documento è già emesso e non può essere modificato"
  }
}
```

### 5.2 `POST /api/v1/ledger`

**Body — `LedgerEntry` (parziale):**

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `invoiceId` | `string` | ✅ | Fattura da riconciliare |
| `amount` | `float64` | ✅ | Deve essere > 0 |
| `entityId` | `string` | No | ID entità pagante |
| `type` | `string` | No | `DEBIT` \| `CREDIT` |
| `method` | `string` | No | `STRIPE` \| `BANK_TRANSFER` |

> `id` e `timestamp` sono impostati automaticamente dal service layer.

**Risposta successo (200):**
```json
{
  "success": true,
  "data": {
    "message": "Pagamento registrato, riconciliazione effettuata"
  }
}
```

---

## 6. Service Discovery

**Definita in:** `internal/handlers/discovery.go`

A differenza degli altri servizi, il Finance Service gestisce la discovery con un **handler Go dedicato** (non tramite `nexus.RegisterDiscovery`). Espone correttamente i 2 endpoint con metodo, path, summary e `authRequired: true`.

```json
{
  "serviceName": "finance-service",
  "version": "1.0.0",
  "endpoints": [
    { "path": "/api/v1/invoices/:id/issue", "method": "PATCH", "authRequired": true },
    { "path": "/api/v1/ledger",             "method": "POST",  "authRequired": true }
  ]
}
```

> ✅ **Questo è l'unico servizio con discovery compilata correttamente.** `ssg-nexus-document-service` ha ancora `Endpoints: []` vuoto.

---

## 7. Dipendenza Inter-servizi

Il Finance Service chiama direttamente il Document Service via HTTP durante `IssueInvoice`:

```
Finance Service
  |
  | POST {DOC_SERVICE_URL}/api/v1/documents/generate
  | Header: Authorization (via nexusClient)
  | Body: { "type": "INVOICE", "data": <invoice> }
  v
Document Service  ← ENDPOINT NON ANCORA IMPLEMENTATO
```

La chiamata usa `nexus.NexusClient.Do()` che gestisce l'autenticazione inter-servizio con il token `INTERNAL_SECRET`.

---

## 8. Variabili d'Ambiente

| Variabile | Descrizione | Obbligatoria |
|---|---|---|
| `GCP_PROJECT_ID` | Project ID GCP per Firestore | ✅ Sì |
| `DOC_SERVICE_URL` | URL base del Document Service (es. `https://doc-service-xxx.run.app`) | ✅ Sì |
| `GATEWAY_URL` | URL del Gateway per l'handshake Discovery | ✅ Sì |
| `SERVICE_URL` | URL Cloud Run di questo servizio | ✅ Sì |
| `INTERNAL_SECRET` | Token condiviso Gateway ↔ Servizio | ✅ Sì |
| `PORT` | Porta HTTP (default `8080`) | No |

---

## 9. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | `POST /api/v1/documents/generate` non esiste nel Document Service — `IssueInvoice` fallirà sempre in produzione | ⏳ Bloccante |
| 🔴 Alta | Nessun endpoint `POST /invoices` (creazione DRAFT) — le fatture non possono essere create via API | ⏳ Mancante |
| 🔴 Alta | Nessun endpoint `GET /invoices` / `GET /invoices/:id` — impossibile leggere le fatture | ⏳ Mancante |
| 🟡 Media | Race condition sulla riconciliazione: doppio pagamento concorrente può settare PAID due volte | ⏳ Aperto |
| 🟡 Media | `UpdateInvoice` usa `.Set()` (upsert totale) — rischio perdita campi non inclusi nella struct | ⏳ Aperto |
| 🟡 Media | `ExternalID` generato come `SDI-{anno}-{id[:6]}` — non è sequenziale e non è univoco garantito | ⏳ Aperto |
| 🟡 Media | `Totals` contiene solo `Gross` — mancano `net`, `tax`, `items` presenti nel vecchio schema doc | ⏳ Aperto |
| 🟢 Bassa | Nessun endpoint `GET /ledger` per consultare i movimenti | ⏳ Mancante |
| 🟢 Bassa | Nessun test unitario o di integrazione nel repo | ⏳ Aperto |

---

## 10. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/DaniFX/ssg-nexus-sdk` | Guard, NexusClient inter-servizio, Response standard |
| `cloud.google.com/go/firestore` | Persistenza fatture e ledger |
| `github.com/gin-gonic/gin` | HTTP framework |
| `github.com/joho/godotenv` | Caricamento `.env` in sviluppo locale |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
