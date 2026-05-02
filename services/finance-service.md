# 💰 SSG Finance Service — Specifiche Tecniche

> **Repository:** [`ssg-finance-service`](https://github.com/DaniFX/ssg-finance-service)
> **Stack:** Go 1.21+, Gin, Firestore (GCP), `ssg-nexus-sdk`
> **Ruolo:** Motore finanziario di SSG Nexus. Gestisce il ciclo di vita delle fatture (DRAFT → ISSUED → PAID), il libro giornale dei pagamenti e la riconciliazione automatica. Interagisce con `ssg-nexus-document-service` per la generazione dei PDF.

---

## 1. Principi Architetturali

### 1.1 Locking ERP (Immutabilità)

Una fattura emessa (`ISSUED`) è **immutabile**. Il servizio implementa un doppio controllo:

- `invoice.Status == StatusIssued` → già emessa
- `invoice.Metadata["immutable"] == true` → flag di lock esplicito

Qualsiasi tentativo di modifica restituisce `ERP_LOCK_ERROR`. Il flag `immutable` viene impostato dal servizio stesso al momento dell’emissione, mai dal client.

### 1.2 Riconciliazione Automatica

Ogni pagamento registrato nel ledger triggera una riconciliazione automatica: si sommano tutti i `LedgerEntry.Amount` per la fattura e, se il totale `>= invoice.Totals.Gross`, la fattura passa automaticamente a `PAID` con `Dates.Paid` valorizzato.

### 1.3 Integrazione Document Service

Al momento dell’emissione (`IssueInvoice`), il Finance Service chiama via HTTP `ssg-nexus-document-service` all’endpoint `POST /api/v1/documents/generate`. Il `documentId` restituito viene salvato nel campo `Invoice.DocumentRef`.

---

## 2. Struttura del Progetto

```
ssg-finance-service/
├── cmd/
│   └── finance/
│       └── main.go                  # Bootstrap, discovery, guard, routes
├── internal/
│   ├── handlers/
│   │   ├── discovery.go             # Handler /_discover
│   │   ├── invoices.go              # IssueInvoice handler
│   │   └── ledger.go                # RegisterTransaction handler
│   ├── models/
│   │   ├── invoice.go               # Invoice, InvoiceDates, Totals, Entity
│   │   └── ledger.go                # LedgerEntry
│   ├── repository/
│   │   └── firestore.go             # FinanceRepository: invoices + ledger_entries
│   └── services/
│       └── finance_service.go       # Business logic: UpdateInvoice, IssueInvoice, RegisterPayment
└── Dockerfile
```

---

## 3. Modelli Dati

### 3.1 `Invoice` — `internal/models/invoice.go`

```go
type InvoiceStatus string

const (
    StatusDraft     InvoiceStatus = "DRAFT"
    StatusIssued    InvoiceStatus = "ISSUED"
    StatusPaid      InvoiceStatus = "PAID"
    StatusCancelled InvoiceStatus = "CANCELLED"
)

type Invoice struct {
    ID          string         // UUID fattura
    ExternalID  string         // Numero sequenziale SDI (es. "SDI-2026-abc123")
    Type        string         // INVOICE | CREDIT_NOTE | ...
    Status      InvoiceStatus  // DRAFT | ISSUED | PAID | CANCELLED
    Issuer      Entity         // Snapshot emittente (da Registry)
    Receiver    Entity         // Snapshot destinatario (da Registry)
    Totals      Totals         // Gross + Currency
    Dates       InvoiceDates   // document, due, paid
    DocumentRef string         // ID documento in ssg-nexus-document-service
    Metadata    map[string]any // immutable: true dopo emissione
}

type InvoiceDates struct {
    Document time.Time  // Data documento
    Due      time.Time  // Scadenza pagamento
    Paid     *time.Time // Null finché non pagata
}

type Entity struct {
    EntityID string // ID da Registry Service
    Name     string
    VAT      string
}

type Totals struct {
    Gross    float64
    Currency string // es. "EUR"
}
```

> **Design:** `Issuer` e `Receiver` sono **snapshot** copiati da Registry al momento della creazione della fattura. Sono immutabili rispetto alle modifiche successive all’anagrafica.

### 3.2 `LedgerEntry` — `internal/models/ledger.go`

```go
type LedgerEntry struct {
    ID        string    // Auto-generato da Firestore se vuoto
    EntityID  string    // Riferimento all'entità pagante
    InvoiceID string    // Fattura di riferimento (obbligatorio)
    Amount    float64   // Importo pagamento (> 0)
    Type      string    // DEBIT | CREDIT
    Method    string    // STRIPE | BANK_TRANSFER
    Timestamp time.Time // Impostato dal service layer al momento del salvataggio
}
```

---

## 4. Service Layer

**File:** `internal/services/finance_service.go`

```go
type FinanceService struct {
    repo          repository.FinanceRepository
    nexusClient   *nexus.NexusClient  // per chiamate inter-service
    docServiceURL string              // URL ssg-nexus-document-service
}
```

### 4.1 `UpdateInvoice` — Aggiornamento DRAFT con Lock

```
UpdateInvoice(ctx, inv)
  |
  | [1] GetInvoice(ctx, inv.ID) -> existing
  | [2] Se existing.Status == ISSUED || existing.Metadata["immutable"] == true
  |     -> return error "cannot update an issued invoice" (ERP Lock)
  | [3] repo.UpdateInvoice(ctx, inv)
```

### 4.2 `IssueInvoice` — Emissione + Documento + Lock

```
IssueInvoice(ctx, invoiceID)
  |
  | [1] repo.GetInvoice(ctx, invoiceID)
  | [2] Verifica Lock: Status==ISSUED || Metadata["immutable"]==true
  |     -> ERP_LOCK_ERROR
  | [3] Genera ExternalID: "SDI-{year}-{id[:6]}"
  | [4] generateDocument(ctx, invoice)
  |     -> POST {DOCUMENT_SERVICE_URL}/api/v1/documents/generate
  |     -> nexusClient.Do(ctx, req)   (chiamata inter-service autenticata)
  |     -> Decode result.Data.DocumentID
  | [5] invoice.DocumentRef = documentID
  |     invoice.Status = "ISSUED"
  |     invoice.Metadata["immutable"] = true
  | [6] repo.UpdateInvoice(ctx, invoice)
```

### 4.3 `RegisterPayment` — Pagamento + Riconciliazione

```
RegisterPayment(ctx, entry)
  |
  | [1] entry.Timestamp = time.Now()
  | [2] repo.SaveLedgerEntry(ctx, entry)
  | [3] repo.GetTotalPaidForInvoice(ctx, entry.InvoiceID)
  |     -> SUM di tutti i LedgerEntry.Amount per invoiceID
  | [4] repo.GetInvoice(ctx, entry.InvoiceID)
  | [5] Se totalPaid >= invoice.Totals.Gross && Status != PAID
  |     -> invoice.Status = "PAID"
  |     -> invoice.Dates.Paid = &now
  |     -> repo.UpdateInvoice(ctx, invoice)
```

---

## 5. Repository Layer

**File:** `internal/repository/firestore.go`

Collezioni Firestore usate:

| Collezione | Contenuto |
|---|---|
| `invoices` | Documenti `Invoice` (1 doc per fattura, ID = Invoice.ID) |
| `ledger_entries` | Documenti `LedgerEntry` (ID auto-generato se vuoto) |

| Metodo | Operazione Firestore |
|---|---|
| `GetInvoice(ctx, id)` | `invoices/{id}` → `DataTo(&Invoice)` |
| `UpdateInvoice(ctx, inv)` | `invoices/{id}.Set(ctx, inv)` (upsert) |
| `SaveLedgerEntry(ctx, entry)` | `ledger_entries/{id}.Set(ctx, entry)` |
| `GetTotalPaidForInvoice(ctx, invoiceID)` | `ledger_entries.Where("invoiceId","==",id)` + sum |

> **Nota:** `NewFinanceRepository` restituisce `error` (non `log.Fatalf`) — comportamento più robusto rispetto al Registry Service. L’errore viene propagato e gestito in `main.go`.

---

## 6. Service Discovery

**Definita in:** `cmd/finance/main.go`

Servizio: `finance-service` | Versione: `1.0.0`

| Metodo | Path | Auth | Stato |
|---|---|---|---|
| `PATCH` | `/api/v1/finance/invoices/:id/issue` | ✅ | ✅ Attivo |
| `POST` | `/api/v1/finance/ledger` | ✅ | ✅ Attivo |
| `GET` | `/_discover` | ❌ No | ✅ Fuori dal Guard |

---

## 7. API Endpoints

### `PATCH /api/v1/finance/invoices/:id/issue`

Emette una fattura DRAFT, genera il documento PDF e la blocca in stato immutabile.

**Nessun body richiesto.** L’ID fattura passa come path parameter.

**Risposta `200 OK`:**
```json
{
  "success": true,
  "data": {
    "status": "ISSUED",
    "message": "Fattura emessa e bloccata con successo"
  },
  "meta": null
}
```

**Errori:**

| Codice | Chiave | Causa |
|---|---|---|
| `400` | `ERP_LOCK_ERROR` | Fattura già emessa o flag `immutable: true` |
| `400` | `ERP_LOCK_ERROR` | Errore chiamata a Document Service |

---

### `POST /api/v1/finance/ledger`

Registra un pagamento nel libro giornale e innesca la riconciliazione automatica.

**Body:**
```json
{
  "entityId":  "uuid-entita",
  "invoiceId": "uuid-fattura",
  "amount":    1500.00,
  "type":      "CREDIT",
  "method":    "BANK_TRANSFER"
}
```

> **Validazione handler:** `invoiceId` obbligatorio, `amount` deve essere `> 0`. Il campo `timestamp` viene impostato dal service layer — ignorare eventuali valori inviati dal client.

**Risposta `200 OK`:**
```json
{
  "success": true,
  "data": {
    "message": "Pagamento registrato, riconciliazione effettuata"
  },
  "meta": null
}
```

**Errori:**

| Codice | Chiave | Causa |
|---|---|---|
| `400` | `INVALID_PAYLOAD` | JSON malformato |
| `400` | `VALIDATION_ERROR` | `invoiceId` vuoto o `amount <= 0` |
| `500` | `LEDGER_ERROR` | Errore salvataggio Firestore o riconciliazione |

---

## 8. Chiamate Inter-Service

Il Finance Service chiama direttamente `ssg-nexus-document-service` tramite `nexus.NexusClient`.

```
Finance Service
  |
  | POST {DOCUMENT_SERVICE_URL}/api/v1/documents/generate
  | Body: { "type": "INVOICE", "data": <Invoice> }
  | Auth: nexusClient.Do() (header INTERNAL_SECRET iniettato dall'SDK)
  |
  v
Document Service
  |
  | Response: { "data": { "documentId": "doc-uuid" } }
  v
Finance Service salva Invoice.DocumentRef = documentId
```

> `DOCUMENT_SERVICE_URL` deve essere impostato come variabile d’ambiente. Se vuota, `generateDocument` fallisce con errore HTTP.

---

## 9. Variabili d’Ambiente

| Variabile | Descrizione | Comportamento se assente |
|---|---|---|
| `GOOGLE_CLOUD_PROJECT` | Project ID GCP per Firestore | Warning + fallback `"ssg-nexus-dev"` |
| `DOCUMENT_SERVICE_URL` | URL base di `ssg-nexus-document-service` | Errore a runtime su `IssueInvoice` |
| `GATEWAY_URL` | URL Gateway per handshake discovery | Richiesta dall’SDK |
| `SERVICE_URL` | URL Cloud Run di questo servizio | Richiesta dall’SDK |
| `INTERNAL_SECRET` | Token condiviso Gateway ↔ Servizio | Richiesta dall’SDK |
| `PORT` | Porta HTTP | Default `8080` |

> **Differenza dal Registry:** `GOOGLE_CLOUD_PROJECT` è un warning (non fatal) con fallback su `ssg-nexus-dev`. Il servizio si avvia comunque, utile per sviluppo locale.

---

## 10. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | `POST /invoices` (creazione fattura DRAFT) non implementato — non c’è handler per creare una nuova fattura | ⏳ Mancante |
| 🔴 Alta | `GET /invoices/:id` non implementato — impossibile leggere una fattura via API | ⏳ Mancante |
| 🔴 Alta | `ExternalID` generato con `id[:6]` — non garantisce unicità in produzione | ⚠️ Da sostituire |
| 🔴 Alta | `GetTotalPaidForInvoice` somma tutti i `LedgerEntry` senza filtrare per `Type=CREDIT` — potrebbe contare DEBIT | ⏳ Bug potenziale |
| 🟡 Media | Nessun controllo se la fattura esiste prima di chiamare `generateDocument` | ⏳ Aperto |
| 🟡 Media | `nexus.NexusClient{}` inizializzato vuoto in `main.go` — verificare se l’SDK richiede configurazione aggiuntiva | ⏳ Da verificare |
| 🟡 Media | Nessuna gestione del caso `StatusCancelled` nel ciclo di vita | ⏳ Non implementato |
| 🟢 Bassa | Nessun test unitario o di integrazione nel repo | ⏳ Aperto |

---

## 11. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/DaniFX/ssg-nexus-sdk` | Guard, NexusClient, Discovery, Response standard |
| `cloud.google.com/go/firestore` | Persistenza fatture e ledger |
| `github.com/gin-gonic/gin` | HTTP framework |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
