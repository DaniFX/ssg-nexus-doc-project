# 📘 SSG Nexus — Documento di Riferimento (Source of Truth)

> **Versione:** 1.0.0
> **Data:** 2026-05-02
> **Stato:** Attivo — Documento normativo per tutti i contributori del progetto Nexus.

---

## Indice

1. [Introduzione e Scopo](#1-introduzione-e-scopo)
2. [Principi Architetturali](#2-principi-architetturali)
3. [Infrastruttura GCP](#3-infrastruttura-gcp)
4. [Flusso di Autenticazione](#4-flusso-di-autenticazione)
5. [Nexus SDK (Go)](#5-nexus-sdk-go)
6. [Servizi di Dominio](#6-servizi-di-dominio)
7. [Standard Globali](#7-standard-globali)
8. [Contratti e JSON Schema](#8-contratti-e-json-schema)
9. [Open Issues e Roadmap](#9-open-issues-e-roadmap)

---

## 1. Introduzione e Scopo

**SSG Nexus** è il sottoprogetto della piattaforma SSG dedicato alla gestione di processi gestionali/ERP, flussi documentali e anagrafiche. È implementato come sistema a **microservizi in Go** deployati su **Google Cloud Platform (GCP)**.

L'obiettivo architetturale è fornire un ecosistema coerente e scalabile dove:
- Ogni microservizio è **autonomo e stateless**.
- La **comunicazione** tra i servizi avviene tramite rete VPC interna.
- Un **SDK condiviso** garantisce la conformità a standard di sicurezza, risposta e persistenza.
- Ogni operazione di scrittura è **tracciabile e auditabile**.

### Repository del Progetto

| Repository | Ruolo |
|---|---|
| `ssg-gateway` | Gateway unico di ingresso, autenticazione e routing |
| `ssg-registry-service` | Anagrafica polimorfica (soci, clienti, fornitori) |
| `ssg-nexus-document-service` | Gestione file, metadati e allegati |
| `ssg-finance-service` | Modulo ERP: fatture, ledger, riconciliazione |
| `ssg-nexus-sdk` | Libreria Go condivisa (middleware, wrapper, SDK) |
| `ssg-nexus-doc-project` | **Questo repo** — Documentazione e source of truth |
| `ssg-admin` | Interfaccia di amministrazione |
| `ssg-db` | Configurazioni database e Firestore rules |
| `ssg-mail-reader-service` | Servizio di lettura e parsing email |
| `ssg-project-definition` | Definizione del progetto e configurazioni globali |

---

## 2. Principi Architetturali

### 2.1 Design Principles

- **Gateway-first**: Nessun servizio è esposto direttamente a internet. Tutto il traffico esterno transita dal Gateway.
- **Identity Propagation**: L’identità utente viene propagata tra i servizi tramite header HTTP standard Nexus, non tramite token ripetuti.
- **Contract-First**: Le API sono definite prima dell’implementazione. I JSON Schema in `/contracts` sono la fonte di verità per i payload.
- **ERP-Grade Integrity**: I dati fiscali e contabili sono immutabili una volta emessi. Non si cancella mai fisicamente un record.
- **SDK as Guardrail**: L’SDK condiviso non è opzionale; ogni microservizio deve adottarlo per garantire uniformità.

### 2.2 Pattern Trasversali

- **Soft Delete**: Nessun dato viene mai rimosso fisicamente. Si imposta `deletedAt` tramite `SoftDelete()` dell’SDK.
- **Audit Log**: Ogni scrittura registra `createdBy` e `updatedAt` estratti dal `NexusContext`.
- **Immutabilità ERP**: Se `immutable: true` o `status == ISSUED`, qualsiasi modifica restituisce `ERR_IMMUTABLE_RECORD`.
- **Navigator Pattern**: Tutti i listing endpoint supportano `?status=PAID&sort=-createdAt&limit=20`.

---

## 3. Infrastruttura GCP

| Componente | Servizio GCP | Note |
|---|---|---|
| Microservizi | **Cloud Run** | Stateless, auto-scaling, ingress solo interno |
| Database logico | **Firestore** | Collection-based, no-SQL |
| Storage fisico file | **Google Cloud Storage** | Signed URL per upload sicuro |
| Autenticazione | **Firebase Authentication** | JWT con custom claims per i ruoli |
| Rete interna | **VPC Shared** | I servizi comunicano solo internamente |
| Logging & Tracing | **Cloud Logging + Cloud Trace** | Tracciamento distribuito via `X-Nexus-Trace-ID` |
| Secrets | **Secret Manager** | Credenziali GCP, chiavi Firebase |

---

## 4. Flusso di Autenticazione

```
Client (App/Browser)
    |
    |  Authorization: Bearer <Firebase JWT>
    v
+-----------------------------------------------+
|                  SSG Gateway                  |
|  1. Verifica JWT con firebase-admin SDK        |
|  2. Estrae uid          --> X-Nexus-User-ID    |
|  3. Estrae custom claim --> X-Nexus-Role       |
|  4. Genera              --> X-Nexus-Trace-ID   |
+-----------------------------------------------+
    |
    |  Header iniettati (rete VPC interna)
    v
+------------------+  +------------------+  +------------------+
| Registry Service |  | Document Service |  | Finance Service  |
|   (Cloud Run)    |  |   (Cloud Run)    |  |   (Cloud Run)    |
+------------------+  +------------------+  +------------------+
```

### Header Standard Nexus

| Header | Contenuto | Obbligatorio |
|---|---|---|
| `X-Nexus-User-ID` | Firebase UID dell’utente autenticato | ✅ Sì |
| `X-Nexus-Role` | Ruolo estratto dai custom claims Firebase | ✅ Sì |
| `X-Nexus-Trace-ID` | ID per il logging distribuito (Cloud Trace) | ✅ Sì |

> **Regola**: Ogni microservizio verifica `X-Nexus-User-ID` tramite il middleware `NexusGuard`. Assenza header -> `401 ERR_UNAUTHORIZED`.

---

## 5. Nexus SDK (Go)

Il **Nexus SDK** è la libreria interna Go 1.21+ nel repo `ssg-nexus-sdk`. È **obbligatoria** per tutti i microservizi.

### 5.1 NexusGuard Middleware

Verifica la presenza di `X-Nexus-User-ID` e inietta identità e ruolo nel `context.Context` di Go.

```go
func NexusGuard() gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetHeader("X-Nexus-User-ID")
        if userID == "" {
            nexus.ErrorResponse(c, 401, "ERR_UNAUTHORIZED", "Missing Nexus Identity")
            c.Abort()
            return
        }
        ctx := context.WithValue(c.Request.Context(), nexus.UserIDKey, userID)
        ctx = context.WithValue(ctx, nexus.RoleKey, c.GetHeader("X-Nexus-Role"))
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}
```

### 5.2 Nexus Context & Identity

```go
type NexusIdentity struct {
    UserID string
    Role   string
}

func FromContext(ctx context.Context) NexusIdentity { ... }
func HasRole(ctx context.Context, role string) bool { ... }
```

### 5.3 Standard Responder

**Successo:** `{ "success": true, "data": { ... }, "meta": { "page": 1, "total": 42 } }`

**Errore:** `{ "success": false, "error": { "code": "ERR_CODE", "message": "..." } }`

### 5.4 Firestore Repository Wrapper

- `Create()` — inietta `createdAt`, `updatedAt`, `createdBy` automaticamente.
- `SoftDelete()` — imposta `deletedAt` invece di rimuovere il documento.
- `IsLocked()` — controlla `immutable` o `status == ISSUED` prima di ogni update.
- `ApplyNavigator()` — traduce query string HTTP in query Firestore.

### 5.5 NexusDoc (Struct Base)

```go
type NexusDoc struct {
    ID        string     `firestore:"id"`
    CreatedAt time.Time  `firestore:"createdAt"`
    UpdatedAt time.Time  `firestore:"updatedAt"`
    CreatedBy string     `firestore:"createdBy"`
    DeletedAt *time.Time `firestore:"deletedAt,omitempty"`
    Immutable bool       `firestore:"immutable"`
}
```

### 5.6 Service Discovery

Ogni microservizio espone `/_discover` implementato dall’SDK per il routing dinamico del Gateway.

---

## 6. Servizi di Dominio

### 6.1 Registry Service (`ssg-registry-service`)

> Documentazione completa: [`services/registry-service.md`](./services/registry-service.md)

**Missione**: Custodire le anagrafiche polimorfiche (PERSON / ORGANIZATION) con subTypes flessibili.

**Collezione Firestore**: `entities`

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/entities` | Lista filtrabile via Navigator |
| POST | `/entities` | Creazione con validazione fiscale |
| GET | `/entities/:id` | Dettaglio completo |
| PATCH | `/entities/:id` | Aggiornamento parziale |
| DELETE | `/entities/:id` | Soft delete |

**Business Rules:**
- `coreData.taxCode` univoco nella collezione.
- `vatNumber` obbligatorio se `type == ORGANIZATION`.
- Verificare assenza fatture `PENDING` prima del soft-delete.

---

### 6.2 Document Service (`ssg-nexus-document-service`)

> Documentazione completa: [`services/document-service.md`](./services/document-service.md)

**Missione**: Gestire file e metadati, ponte tra GCS (fisico) e Firestore (logico).

**Collezione Firestore**: `documents`

**Flusso Upload:**
```
1. Frontend -> Document Service: richiesta Signed URL
2. Verifica permessi -> URL temporaneo GCS
3. Frontend -> GCS: upload diretto
4. Frontend -> Document Service: conferma
5. Record su Firestore + eventuali analisi
```

---

### 6.3 Finance Service (`ssg-finance-service`)

> Documentazione completa: [`services/finance-service.md`](./services/finance-service.md)

**Missione**: Modulo ERP per ciclo attivo/passivo e riconciliazione.

**Collezioni Firestore**: `invoices`, `ledger_entries`

**Business Rules ERP:**
- Numerazione sequenziale per anno fiscale.
- `status -> ISSUED` -> lock immutabile via SDK.
- Riconciliazione automatica: somma pagamenti >= gross -> `PAID`.

---

## 7. Standard Globali

> Documentazione completa: [`architecture/global-standards.md`](./architecture/global-standards.md)

### Codici Errore Nexus

| Codice Nexus | HTTP | Quando |
|---|---|---|
| `ERR_UNAUTHORIZED` | 401 | Header `X-Nexus-User-ID` assente |
| `ERR_FORBIDDEN` | 403 | Ruolo insufficiente |
| `ERR_NOT_FOUND` | 404 | Risorsa non trovata |
| `ERR_IMMUTABLE_RECORD` | 409 | Record locked ERP |
| `ERR_VALIDATION_FAILED` | 400 | Payload non conforme |
| `ERR_INTERNAL` | 500 | Errore server/GCP |

### Regole di Integrità

- **Soft Delete Only** — mai `Delete()` diretto su Firestore.
- **Audit Log** — ogni scrittura include `createdBy` da `X-Nexus-User-ID`.
- **Unicità fiscale** — `taxCode` e `vatNumber` univoci in `entities`.

### Navigator — Query Standard

| Parametro | Esempio | Comportamento |
|---|---|---|
| `sort` | `?sort=-createdAt` | Discendente |
| `sort` | `?sort=status` | Ascendente |
| `limit` | `?limit=20` | Max risultati |
| `<campo>` | `?status=PAID` | Filtro uguaglianza |

---

## 8. Contratti e JSON Schema

> Contratti formali: [`contracts/schemas.json`](./contracts/schemas.json)

| Schema | Stato |
|---|---|
| `NexusEntity` | ✅ Disponibile |
| `NexusInvoice` | ✅ Disponibile |
| `NexusDocument` | ✅ Disponibile |
| `NexusLedgerEntry` | ✅ Disponibile |

---

## 9. Open Issues e Roadmap

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | `inance-service.md` — typo nel nome | ✅ Rinominato `finance-service.md` |
| 🟡 Media | `contracts/schemas.json` — incompleto | ✅ Aggiornato con tutti i modelli |
| 🟡 Media | `README.md` — vuoto | ✅ Questo documento |
| 🟢 Bassa | `ssg-mail-reader-service` non documentato | ⏳ Aperto |
| 🟢 Bassa | `/_discover` non testato e2e | ⏳ Aperto |

### Roadmap Documentale

- [ ] `services/gateway.md` — routing rules e discovery
- [ ] `services/mail-reader-service.md`
- [ ] `architecture/deployment.md` — diagramma infrastruttura GCP
- [ ] `architecture/data-flows.md` — flussi inter-servizio
- [ ] `adr/` — Architecture Decision Records

---

*Documento creato il 2026-05-02 — da aggiornare ad ogni decisione architetturale rilevante.*
