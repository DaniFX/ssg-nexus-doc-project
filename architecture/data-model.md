# SSG Nexus — Modello Dati Condiviso

> **Versione:** 1.0.0 | **Data:** 2026-05-06
> Documento operativo derivato da `ssg-nexus-doc-project` v1.1.0.
> Destinato al team frontend (Portal) e BFF per allineamento su entità, relazioni, convenzioni API e TypeScript types.

---

## 1. Principi fondamentali

Il modello dati Nexus è fondato su quattro regole non negoziabili:

- **Soft Delete only** — nessun record viene mai eliminato fisicamente. Ogni entità ha `deletedAt: string | null`. Il BFF e i servizi filtrano automaticamente i record con `deletedAt != null`.
- **Immutabilità ERP** — se `immutable: true` oppure `status == ISSUED` (per le fatture), qualsiasi PATCH o DELETE restituisce `ERR_IMMUTABLE_RECORD`. Il frontend deve disabilitare i controlli di modifica in questi stati.
- **Audit automatico** — `createdAt`, `updatedAt`, `createdBy` (Firebase UID) sono iniettati dall'SDK lato backend; il frontend non li invia mai in POST/PATCH.
- **`NexusDoc` è la base di tutto** — ogni entità Firestore estende questa struttura.

---

## 2. Gerarchia delle entità

```
NexusDoc (base)
├── NexusEntity          → collezione: entities         (Registry Service)
├── NexusDocument        → collezione: documents        (Document Service)
├── NexusInvoice         → collezione: invoices         (Finance Service)
└── NexusLedgerEntry     → collezione: ledger_entries   (Finance Service)
```

Le relazioni tra entità sono **reference by ID** (non embedding), con unica eccezione: `FiscalParty` dentro `NexusInvoice` è uno **snapshot immutabile** dei dati fiscali al momento dell'emissione — non un riferimento live all'entità.

---

## 3. TypeScript Types — Portal Frontend

Questi tipi sono da inserire in `src/types/nexus.ts` nel repository `ssg-nexus-portal-frontend`.

```typescript
// ─── BASE ────────────────────────────────────────────────────────────────────

export interface NexusDoc {
  id: string;
  createdAt: string;       // ISO 8601
  updatedAt: string;       // ISO 8601
  createdBy: string;       // Firebase UID
  deletedAt: string | null;
  immutable: boolean;
}

// ─── ENTITY (Anagrafica) ──────────────────────────────────────────────────────

export type EntityType    = 'PERSON' | 'ORGANIZATION';
export type EntitySubType = 'MEMBER' | 'CUSTOMER' | 'SUPPLIER' | 'EMPLOYEE' | 'PARTNER';
export type EntityStatus  = 'ACTIVE' | 'INACTIVE' | 'SUSPENDED';

export interface EntityAddress {
  street?:  string;
  city?:    string;
  zip?:     string;
  country:  string; // default: "IT"
}

export interface EntityCoreData {
  displayName: string;          // Nome o ragione sociale — obbligatorio
  email:       string;          // obbligatorio
  phone?:      string;
  taxCode?:    string;          // Codice fiscale — univoco
  vatNumber?:  string;          // Partita IVA — obbligatoria se type == ORGANIZATION
  address?:    EntityAddress;
}

export interface NexusEntity extends NexusDoc {
  type:      EntityType;
  subType:   EntitySubType;
  status:    EntityStatus;
  coreData:  EntityCoreData;
  metadata?: Record<string, unknown>; // Campi extra per subType
}

// Regola di validazione frontend:
// if (entity.type === 'ORGANIZATION') => vatNumber obbligatoria

// ─── DOCUMENT ────────────────────────────────────────────────────────────────

export type DocumentStatus     = 'PENDING_UPLOAD' | 'ACTIVE' | 'ARCHIVED' | 'DELETED';
export type DocumentParentType = 'ENTITY' | 'INVOICE' | 'LEDGER_ENTRY';

export interface DocumentRelation {
  parentType: DocumentParentType;
  parentId:   string;
}

export interface NexusDocument extends NexusDoc {
  name:        string;          // Nome file con estensione
  mimeType:    string;          // Es. "application/pdf"
  size?:       number;          // Byte
  storageUrl:  string;          // URL GCS (non usare direttamente — usare signedUrl)
  signedUrl?:  string | null;   // Signed URL temporaneo per download — non persistito
  status:      DocumentStatus;
  relation:    DocumentRelation;
  tags?:       string[];
  ocrText?:    string | null;
}

// ─── INVOICE (Fattura) ────────────────────────────────────────────────────────

export type InvoiceType      = 'INVOICE' | 'CREDIT_NOTE' | 'RECEIPT';
export type InvoiceDirection = 'INBOUND' | 'OUTBOUND';
export type InvoiceStatus    = 'DRAFT' | 'ISSUED' | 'PAID' | 'OVERDUE' | 'CANCELLED';
export type PaymentMethod    = 'STRIPE' | 'BANK_TRANSFER' | 'CASH' | 'OTHER';

export interface FiscalParty {
  entityId: string;    // Riferimento a NexusEntity.id
  name:     string;    // Snapshot al momento emissione
  vat?:     string;
  taxCode?: string;
}

export interface InvoiceItem {
  description: string;
  quantity:    number;
  unitPrice:   number;
  vatRate:     number; // Percentuale, es. 22
  total:       number;
}

export interface InvoiceTotals {
  net:      number;
  tax:      number;
  gross:    number;
  currency: string; // default: "EUR"
}

export interface InvoiceDates {
  document: string;        // ISO 8601 — data documento
  due?:     string | null; // Scadenza
  paid?:    string | null; // Data pagamento effettivo
}

export interface NexusInvoice extends NexusDoc {
  externalId?:  string;         // SDI ID o riferimento sistema esterno
  type:         InvoiceType;
  direction:    InvoiceDirection;
  status:       InvoiceStatus;
  issuer:       FiscalParty;
  receiver:     FiscalParty;
  items:        InvoiceItem[];
  totals:       InvoiceTotals;
  dates:        InvoiceDates;
  documentRef?: string | null;  // ID NexusDocument allegato (PDF fattura)
}

// ─── LEDGER ENTRY (Movimento contabile) ──────────────────────────────────────

export type LedgerEntryType = 'DEBIT' | 'CREDIT';

export interface NexusLedgerEntry extends NexusDoc {
  entityId:  string;           // ID NexusEntity
  invoiceId: string;           // ID NexusInvoice
  amount:    number;           // >= 0.01
  type:      LedgerEntryType;
  method:    PaymentMethod;
  timestamp: string;           // ISO 8601
  note?:     string | null;
}
```

---

## 4. Relazioni tra entità — Diagramma

```
NexusEntity (entities)
    │
    ├──[1:N]──► NexusDocument  (relation.parentType = "ENTITY",        relation.parentId = entity.id)
    │
    ├──[1:N]──► NexusInvoice   (issuer.entityId | receiver.entityId)
    │               │
    │               ├──[1:1]──► NexusDocument  (documentRef = document.id)  [PDF allegato]
    │               │
    │               └──[1:N]──► NexusLedgerEntry  (invoiceId = invoice.id)
    │                               │
    │                               ├──[N:1]──► NexusEntity  (entityId = entity.id)
    │                               └──[1:N]──► NexusDocument  (relation.parentType = "LEDGER_ENTRY")
    │
    └──[relazione indiretta via FiscalParty — snapshot immutabile]
```

### Regole di relazione

| Relazione | Chiave | Note |
|---|---|---|
| Documento → Entità | `relation.parentId` | Documento allegato a un soggetto |
| Documento → Fattura | `relation.parentId` | Documento allegato a una fattura (diverso da `documentRef`) |
| Documento → Movimento | `relation.parentId` | Documento allegato a un movimento contabile |
| Fattura → PDF | `documentRef` | Un solo documento PDF principale per fattura |
| Fattura → Emittente | `issuer.entityId` | Snapshot fiscale — non aggiornato se l'entità cambia |
| Fattura → Ricevente | `receiver.entityId` | Idem |
| Movimento → Entità | `entityId` | Soggetto del pagamento |
| Movimento → Fattura | `invoiceId` | Fattura a cui il movimento si riferisce |

---

## 5. State machine dei documenti

### NexusEntity — Status

```
ACTIVE ──► SUSPENDED ──► INACTIVE
  ▲              │
  └──────────────┘
```

| Stato | Operazioni consentite |
|---|---|
| `ACTIVE` | Modifica, collegamento documenti/fatture, soft delete |
| `SUSPENDED` | Solo lettura, nessuna nuova fattura |
| `INACTIVE` | Solo lettura, soft deleted de facto |

### NexusDocument — Status

```
PENDING_UPLOAD ──► ACTIVE ──► ARCHIVED
                      │
                      └──► DELETED (soft)
```

| Stato | Note |
|---|---|
| `PENDING_UPLOAD` | Upload GCS in corso — `signedUrl` presente, `storageUrl` definitivo non ancora confermato |
| `ACTIVE` | File confermato su GCS, accessibile tramite nuovo signed URL |
| `ARCHIVED` | Conservato ma non in uso attivo |
| `DELETED` | Soft delete — `deletedAt` impostato, non visibile nelle liste |

### NexusInvoice — Status

```
DRAFT ──► ISSUED (lock) ──► PAID
                │
                ├──► OVERDUE
                └──► CANCELLED
```

| Stato | `immutable` | Operazioni consentite |
|---|---|---|
| `DRAFT` | `false` | Modifica completa, eliminazione |
| `ISSUED` | `true` ← bloccato | Solo lettura, aggiunta pagamenti (LedgerEntry) |
| `PAID` | `true` | Solo lettura |
| `OVERDUE` | `true` | Solo lettura, eventuale sollecito |
| `CANCELLED` | `true` | Solo lettura |

> **Regola UI critica**: se `invoice.status !== 'DRAFT'` oppure `invoice.immutable === true`, tutti i controlli di modifica devono essere `disabled`. Il pulsante "Modifica" non appare o appare disabilitato con tooltip esplicativo.

---

## 6. Convenzioni API — BFF

### Formato risposta standard

```typescript
// Successo (lista)
{
  "success": true,
  "data": [ ...NexusEntity[] ],
  "meta": { "total": 42, "limit": 20, "offset": 0, "hasMore": true }
}

// Successo (singolo)
{ "success": true, "data": { ...NexusEntity } }

// Errore
{ "success": false, "error": { "code": "ERR_NOT_FOUND", "message": "Entity not found" } }
```

### Navigator Pattern — parametri di query universali

| Param | Tipo | Esempio | Note |
|---|---|---|---|
| `status` | string | `?status=ACTIVE` | Filtro per stato |
| `sort` | string | `?sort=-createdAt` | `-` prefix = descending |
| `limit` | number | `?limit=20` | Default: 20, Max: 100 |
| `offset` | number | `?offset=40` | Paginazione offset-based |
| `q` | string | `?q=mario` | Ricerca testuale (dove supportata) |

I record con `deletedAt != null` sono **sempre esclusi automaticamente**.

### Endpoint BFF per modulo

**Anagrafiche (Registry)**
```
GET    /api/v1/registry/entities
POST   /api/v1/registry/entities
GET    /api/v1/registry/entities/:id
PATCH  /api/v1/registry/entities/:id
DELETE /api/v1/registry/entities/:id
```

**Documentale (Document Service)**
```
GET    /api/v1/documents
POST   /api/v1/documents/upload-url
POST   /api/v1/documents/confirm/:id
GET    /api/v1/documents/:id
PATCH  /api/v1/documents/:id
DELETE /api/v1/documents/:id
```

**Finance (Finance Service)**
```
GET    /api/v1/finance/invoices
POST   /api/v1/finance/invoices
GET    /api/v1/finance/invoices/:id
PATCH  /api/v1/finance/invoices/:id
POST   /api/v1/finance/invoices/:id/issue

GET    /api/v1/finance/ledger
POST   /api/v1/finance/ledger
GET    /api/v1/finance/ledger/:id
```

---

## 7. Validazioni frontend — riepilogo Zod

| Campo | Regola | Messaggio |
|---|---|---|
| `NexusEntity.coreData.displayName` | minLength: 1 | "Nome obbligatorio" |
| `NexusEntity.coreData.email` | formato email | "Email non valida" |
| `NexusEntity.coreData.vatNumber` | obbligatorio se `type == ORGANIZATION` | "P.IVA obbligatoria per aziende" |
| `NexusInvoice.items` | minItems: 1 | "Aggiungere almeno una riga" |
| `NexusInvoice.totals.gross` | coerente con somma items | "Totale non coerente con le righe" |
| `NexusLedgerEntry.amount` | >= 0.01 | "Importo non valido" |
| `NexusLedgerEntry.invoiceId` | fattura in stato ISSUED | "Il pagamento richiede fattura emessa" |
| Qualsiasi modifica su `immutable: true` | blocco UI preventivo | "Documento non modificabile" |

---

## 8. Campi che il frontend NON invia mai

| Campo | Perché |
|---|---|
| `id` | Generato da Firestore |
| `createdAt` / `updatedAt` | Iniettati dall'SDK |
| `createdBy` | Estratto da `X-Nexus-User-ID` |
| `deletedAt` | Gestito da `SoftDelete()` |
| `immutable` | Impostato dal servizio al cambio di stato |
| `signedUrl` | Generato on-demand, mai persistito |

---

## 9. Issue aperti rilevanti per il frontend

| Issue | Impatto UI | Workaround |
|---|---|---|
| Validazione `vatNumber` mancante nel Registry Service | Backend non rifiuta ORGANIZATION senza P.IVA | **Validare obbligatoriamente lato frontend con Zod** |
| Endpoint GET/PATCH/DELETE commentati in Registry Service | Dettaglio e modifica anagrafica non disponibili | Coordinare con team backend — usare mock in staging |
| `X-Nexus-Trace-ID` non propagato da `NexusClient` | Nessun impatto UI diretto | Ignorare per ora |

---

*Documento generato il 2026-05-06 — da aggiornare ad ogni modifica ai contratti in `contracts/schemas.json`.*
