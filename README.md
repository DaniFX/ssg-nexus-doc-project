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

### Repository del Progetto

| Repository | Ruolo |
|---|---|
| `ssg-gateway` | Gateway unico di ingresso, auth, routing |
| `ssg-registry-service` | Anagrafica polimorfica (soci, clienti, fornitori) |
| `ssg-nexus-document-service` | Gestione file, metadati e allegati |
| `ssg-finance-service` | Modulo ERP: fatture, ledger, riconciliazione |
| `ssg-nexus-sdk` | Libreria Go condivisa (middleware, SDK, wrapper) |
| `ssg-nexus-doc-project` | **Questo repo** — Documentazione e source of truth |
| `ssg-admin` | Interfaccia di amministrazione |
| `ssg-db` | Configurazioni database/Firestore rules |
| `ssg-mail-reader-service` | Servizio lettura e parsing email |
| `ssg-project-definition` | Definizione progetto e configurazioni globali |

---

## 2. Principi Architetturali

- **Gateway-first**: Nessun servizio è esposto direttamente a internet.
- **Identity Propagation**: Identità utente propagata via header Nexus, non token ripetuti.
- **Contract-First**: JSON Schema in `/contracts` sono la fonte di verità per i payload.
- **ERP-Grade Integrity**: Dati fiscali immutabili post-emissione. Niente cancellazione fisica.
- **SDK as Guardrail**: L'SDK è obbligatorio per tutti i microservizi.

### Pattern Trasversali

- **Soft Delete**: Nessun dato rimosso fisicamente. Campo `deletedAt` via `SoftDelete()` SDK.
- **Audit Log**: Ogni scrittura registra `createdBy` e `updatedAt` da `NexusContext`.
- **Immutabilità ERP**: `immutable: true` o `status: ISSUED` → `ERR_IMMUTABLE_RECORD`.
- **Navigator Pattern**: Tutti i listing supportano `?status=PAID&sort=-createdAt&limit=20`.

---

## 3. Infrastruttura GCP

| Componente | Servizio GCP | Note |
|---|---|---|
| Microservizi | **Cloud Run** | Stateless, auto-scaling, ingress interno |
| Database logico | **Firestore** | Collection-based, no-SQL |
| Storage file | **Google Cloud Storage** | Signed URL per upload sicuro |
| Autenticazione | **Firebase Authentication** | JWT con custom claims per i ruoli |
| Rete interna | **VPC Shared** | Comunicazione solo interna |
| Logging & Tracing | **Cloud Logging + Cloud Trace** | Via `X-Nexus-Trace-ID` |
| Secrets | **Secret Manager** | Credenziali GCP, chiavi Firebase |

---

## 4. Flusso di Autenticazione
Client --> [Authorization: Bearer <JWT>] --> SSG Gateway
|
1. Verifica JWT (firebase-admin)
2. Estrae uid -> X-Nexus-User-ID
3. Estrae role -> X-Nexus-Role
4. Genera X-Nexus-Trace-ID
|
[VPC interna, header iniettati]
|
+-----------------+-----------------+-----------------+
| Registry Service| Document Service| Finance Service |
### Header Standard Nexus

| Header | Contenuto | Obbligatorio |
|---|---|---|
| `X-Nexus-User-ID` | Firebase UID | ✅ Sì |
| `X-Nexus-Role` | Ruolo da custom claims | ✅ Sì |
| `X-Nexus-Trace-ID` | ID tracing distribuito | ✅ Sì |

> **Regola**: Ogni microservizio verifica `X-Nexus-User-ID` via middleware `NexusGuard`. Assenza → `401 ERR_UNAUTHORIZED`.

---

## 5. Nexus SDK (Go)

Libreria interna Go 1.21+ in `ssg-nexus-sdk`. Obbligatoria per tutti i microservizi.

### 5.1 NexusGuard Middleware
Verifica `X-Nexus-User-ID` e inietta identità nel `context.Context`.

### 5.2 Standard Responder

**Successo:** `{ "success": true, "data": {...}, "meta": {...} }`  
**Errore:** `{ "success": false, "error": { "code": "ERR_CODE", "message": "..." } }`

### 5.3 Firestore Repository Wrapper

- Iniezione automatica `createdAt`, `updatedAt`, `createdBy`
- `SoftDelete()` → imposta `deletedAt`
- `IsLocked()` → controlla `immutable` o `status == ISSUED`
- `ApplyNavigator()` → traduce query string in query Firestore

### 5.4 NexusDoc (Struct Base)

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

### 5.5 Service Discovery
Ogni microservizio espone `/_discover` implementato dall'SDK per il routing dinamico del Gateway.

---

## 6. Servizi di Dominio

### 6.1 Registry Service — `ssg-registry-service`
**Collezione Firestore**: `entities`  
Gestisce anagrafiche polimorfiche: PERSON / ORGANIZATION con subTypes (MEMBER, CUSTOMER, SUPPLIER).

**Business Rules:**
- `taxCode` univoco nella collezione
- `vatNumber` obbligatorio se `type == ORGANIZATION`
- Prima del soft-delete verificare assenza fatture `PENDING` nel Finance Service

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/entities` | Lista via Navigator |
| POST | `/entities` | Creazione con validazione fiscale |
| GET | `/entities/:id` | Dettaglio completo |
| PATCH | `/entities/:id` | Aggiornamento parziale |
| DELETE | `/entities/:id` | Soft delete |

---

### 6.2 Document Service — `ssg-nexus-document-service`
**Collezione Firestore**: `documents`  
Ponte tra GCS (fisico) e Firestore (logico). Ogni file è un "attachment" collegato a ENTITY, INVOICE o PROJECT.

**Flusso Upload (Signed URL):**
1. Frontend richiede Signed URL al servizio
2. Verifica permessi → restituzione URL temporaneo GCS
3. Upload diretto del file su GCS
4. Conferma → creazione record Firestore + eventuali analisi

**Query documenti di un'entità:**

GET /documents?relation.parentType=ENTITY&relation.parentId={id}


---

### 6.3 Finance Service — `ssg-finance-service`
**Collezioni Firestore**: `invoices`, `ledger_entries`  
Modulo ERP per ciclo attivo/passivo, conformità fiscale e riconciliazione.

**Business Rules ERP:**
- Numerazione sequenziale fatture OUTBOUND per anno fiscale
- `status → ISSUED` → `immutable: true` → blocco UPDATE/DELETE
- Riconciliazione automatica: somma `ledger_entries` >= `totals.gross` → `status = PAID`
- Snapshot fiscale immutabile di issuer/receiver al momento dell'emissione

---

## 7. Standard Globali

### Codici Errore Nexus

| Codice | HTTP | Quando |
|---|---|---|
| `ERR_UNAUTHORIZED` | 401 | Header `X-Nexus-User-ID` assente |
| `ERR_FORBIDDEN` | 403 | Ruolo insufficiente |
| `ERR_NOT_FOUND` | 404 | Risorsa non trovata in Firestore |
| `ERR_IMMUTABLE_RECORD` | 409 | Record locked (ERP) |
| `ERR_VALIDATION_FAILED` | 400 | Payload non conforme allo schema |
| `ERR_INTERNAL` | 500 | Errore server/GCP |

### Regole di Integrità
- Mai `Delete()` diretto su Firestore per dati ERP → solo `SoftDelete()`
- Ogni scrittura include `createdBy` da `X-Nexus-User-ID`
- `taxCode` e `vatNumber` univoci in `entities`

### Navigator — Query Standard

| Parametro | Esempio | Comportamento |
|---|---|---|
| `sort` | `?sort=-createdAt` | Ordine discendente |
| `sort` | `?sort=status` | Ordine ascendente |
| `limit` | `?limit=20` | Max risultati |
| `<campo>` | `?status=PAID` | Filtro uguaglianza |

---

## 8. Contratti e JSON Schema

I contratti formali sono in `/contracts/schemas.json`.

> ⚠️ **TODO**: Espandere con schemi per `Invoice`, `Document` e `LedgerEntry`.

---

## 9. Open Issues e Roadmap

| Priorità | Issue | Azione |
|---|---|---|
| 🔴 Alta | `services/inance-service.md` — nome errato | Rinominare in `finance-service.md` |
| 🟡 Media | `contracts/schemas.json` — incompleto | Aggiungere Invoice, Document, LedgerEntry |
| 🟡 Media | `ssg-mail-reader-service` — non documentato | Aggiungere `services/mail-reader-service.md` |
| 🟢 Bassa | `/_discover` — non testato e2e | Definire test di integrazione |

### Roadmap Documentale
- [ ] Rinominare `inance-service.md` → `finance-service.md`
- [ ] Espandere `/contracts/schemas.json`
- [ ] Aggiungere `services/gateway.md`
- [ ] Aggiungere `architecture/deployment.md`
- [ ] Aggiungere `architecture/data-flows.md`
- [ ] Introdurre sezione `adr/` per Architecture Decision Records

---

*Documento creato il 2026-05-02 — da aggiornare ad ogni decisione architetturale rilevante.*