# 📘 SSG Nexus — Documento di Riferimento (Source of Truth)

> **Versione:** 1.2.0
> **Data:** 2026-05-06
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

| Repository | Ruolo | Documentazione |
|---|---|---|
| [`ssg-gateway`](https://github.com/DaniFX/ssg-gateway) | Gateway unico di ingresso, autenticazione e routing | [`services/gateway.md`](./services/gateway.md) |
| [`ssg-registry-service`](https://github.com/DaniFX/ssg-registry-service) | Anagrafica polimorfica (soci, clienti, fornitori) | [`services/registry-service.md`](./services/registry-service.md) |
| [`ssg-nexus-document-service`](https://github.com/DaniFX/ssg-nexus-document-service) | Gestione file, metadati e allegati | [`services/nexus-document-service.md`](./services/nexus-document-service.md) |
| [`ssg-finance-service`](https://github.com/DaniFX/ssg-finance-service) | Modulo ERP: fatture, ledger, riconciliazione | [`services/finance-service.md`](./services/finance-service.md) |
| [`ssg-mail-reader-service`](https://github.com/DaniFX/ssg-mail-reader-service) | Lettura e parsing email | [`services/mail-reader-service.md`](./services/mail-reader-service.md) |
| [`ssg-nexus-sdk`](https://github.com/DaniFX/ssg-nexus-sdk) | Libreria Go condivisa (middleware, wrapper, SDK) | — |
| [`ssg-db`](https://github.com/DaniFX/ssg-db) | Modelli Firestore e configurazioni database | — |
| [`ssg-admin`](https://github.com/DaniFX/ssg-admin) | Interfaccia di amministrazione | — |
| [`ssg-project-definition`](https://github.com/DaniFX/ssg-project-definition) | Definizione del progetto e configurazioni globali | — |
| [`ssg-nexus-doc-project`](https://github.com/DaniFX/ssg-nexus-doc-project) | **Questo repo** — Documentazione e source of truth | — |

---

## 2. Principi Architetturali

### 2.1 Design Principles

- **Gateway-first**: Nessun servizio è esposto direttamente a internet. Tutto il traffico esterno transita dal Gateway.
- **Identity Propagation**: L'identità utente viene propagata tra i servizi tramite header HTTP standard Nexus, non tramite token ripetuti.
- **Contract-First**: Le API sono definite prima dell'implementazione. I JSON Schema in `/contracts` sono la fonte di verità per i payload.
- **ERP-Grade Integrity**: I dati fiscali e contabili sono immutabili una volta emessi. Non si cancella mai fisicamente un record.
- **SDK as Guardrail**: L'SDK condiviso non è opzionale; ogni microservizio deve adottarlo per garantire uniformità.

### 2.2 Pattern Trasversali

- **Soft Delete**: Nessun dato viene mai rimosso fisicamente. Si imposta `deletedAt` tramite `SoftDelete()` dell'SDK.
- **Audit Log**: Ogni scrittura registra `createdBy` e `updatedAt` estratti dal `NexusContext`.
- **Immutabilità ERP**: Se `immutable: true` o `status == ISSUED`, qualsiasi modifica restituisce `ERR_IMMUTABLE_RECORD`.
- **Navigator Pattern**: Tutti i listing endpoint supportano `?status=PAID&sort=-createdAt&limit=20`.
- **NexusClient**: Le chiamate inter-servizio propagano automaticamente `X-Nexus-User-ID` e `X-Nexus-Role` tramite `NexusClient.Do(ctx, req)`.

---

## 3. Infrastruttura GCP

| Componente | Servizio GCP | Note |
|---|---|---|
| Microservizi | **Cloud Run** | Stateless, auto-scaling, ingress solo interno |
| Database logico | **Firestore** | Collection-based, no-SQL |
| Storage fisico file | **Google Cloud Storage** | Signed URL V4 per upload/download sicuro |
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
         |                    |                      |
   Guard() verifica     Guard() verifica       Guard() verifica
   X-Nexus-User-ID      X-Nexus-User-ID        X-Nexus-User-ID
```

### Header Standard Nexus

| Header | Contenuto | Obbligatorio |
|---|---|---|
| `X-Nexus-User-ID` | Firebase UID dell'utente autenticato | ✅ Sì |
| `X-Nexus-Role` | Ruolo estratto dai custom claims Firebase | ✅ Sì |
| `X-Nexus-Trace-ID` | ID per il logging distribuito (Cloud Trace) | ⚠️ Raccomandato |

> **Regola**: Ogni microservizio verifica `X-Nexus-User-ID` tramite `nexus.Guard()`. Assenza header -> `401 ERR_UNAUTHORIZED`.

---

## 5. Nexus SDK (Go)

Il **Nexus SDK** è la libreria interna Go 1.21+ nel repo [`ssg-nexus-sdk`](https://github.com/DaniFX/ssg-nexus-sdk). È **obbligatoria** per tutti i microservizi.

> Dettaglio completo di tutti i pattern SDK: **[`architecture/global-standards.md`](./architecture/global-standards.md)**

### 5.1 `nexus.Guard()` — Middleware

Verifica `X-Nexus-User-ID` e inietta `UserID`, `Role`, `TraceID` nel `context.Context`.
Vedere [Global Standards §5](./architecture/global-standards.md#5-nexus-guard--middleware-obbligatorio).

### 5.2 `nexus.FromContext()` + `HasRole()`

```go
identity := nexus.FromContext(c.Request.Context())
if !nexus.HasRole(c.Request.Context(), "admin") { ... }
```

> `HasRole()` include **admin bypass** automatico: il ruolo `admin` supera qualsiasi controllo.
Vedere [Global Standards §2.2](./architecture/global-standards.md#22-controllo-ruoli-rbac).

### 5.3 `nexus.Success()` / `nexus.Failure()` — Response Standard

**Successo:** `{ "success": true, "data": { ... }, "meta": { ... } }`

**Errore:** `{ "success": false, "error": { "code": "ERR_CODE", "message": "..." } }`

Vedere [Global Standards §3](./architecture/global-standards.md#3-formato-standard-delle-risposte).

### 5.4 `repository.Repository` — Firestore ORM

- `Create()` — inietta `createdAt`, `updatedAt`, `createdBy` automaticamente.
- `Update()` — verifica `IsLocked()` prima di procedere.
- `SoftDelete()` — imposta `deletedAt` invece di rimuovere il documento.
- `ApplyNavigator()` — traduce query string HTTP in query Firestore (esclude soft-deleted).

Vedere [Global Standards §8.2](./architecture/global-standards.md#82-repository-layer--repositoryrepository) e [§9](./architecture/global-standards.md#9-navigator-pattern--filtri--ordinamento-firestore).

### 5.5 `NexusClient` — Chiamate Inter-Servizio

```go
nc := &nexus.NexusClient{}
resp, err := nc.Do(c.Request.Context(), req) // propaga X-Nexus-User-ID e X-Nexus-Role
```

Vedere [Global Standards §10](./architecture/global-standards.md#10-chiamate-inter-servizio--nexusclient).

### 5.6 `storage.GCSClient` — Signed URL V4

```go
gcs := storage.NewGCSClient(os.Getenv("GCS_BUCKET_NAME"))
uploadURL, _ := gcs.GenerateUploadURL(objectName, mimeType, 15)
downloadURL, _ := gcs.GenerateDownloadURL(objectName, 60)
```

Vedere [Global Standards §11](./architecture/global-standards.md#11-storage-gcs--signed-url-v4).

### 5.7 Service Discovery

Ogni microservizio espone `GET /_discover` e si registra al Gateway all'avvio tramite `nexus.StartGatewayHandshake()`.
Vedere [Global Standards §6](./architecture/global-standards.md#6-service-discovery--pattern-di-registrazione).

---

## 6. Servizi di Dominio

### 6.1 Registry Service

> Documentazione completa: [`services/registry-service.md`](./services/registry-service.md)

**Missione**: Custodire le anagrafiche polimorfiche (PERSON / ORGANIZATION) con subTypes flessibili.
**Collezione Firestore**: `entities`

| Metodo | Path | Auth |
|---|---|---|
| `GET` | `/api/v1/registry/entities` | ✅ |
| `POST` | `/api/v1/registry/entities` | ✅ |
| `GET` | `/api/v1/registry/entities/:id` | ✅ |
| `PATCH` | `/api/v1/registry/entities/:id` | ✅ |
| `DELETE` | `/api/v1/registry/entities/:id` | ✅ |

**Business rules chiave:**
- `vatNumber` obbligatorio se `type == ORGANIZATION`.
- `taxCode` univoco nella collezione.

---

### 6.2 Document Service

> Documentazione completa: [`services/nexus-document-service.md`](./services/nexus-document-service.md)

**Missione**: Gestire file e metadati, ponte tra GCS (fisico) e Firestore (logico).
**Collezione Firestore**: `documents`

**Flusso Upload (Signed URL):**
```
Frontend → POST /upload-url → Document Service → GCS (Signed URL PUT) ← Frontend → PUT GCS
```

---

### 6.3 Finance Service

> Documentazione completa: [`services/finance-service.md`](./services/finance-service.md)

**Missione**: Modulo ERP per ciclo attivo/passivo e riconciliazione.
**Collezioni Firestore**: `invoices`, `ledger_entries`

**Ciclo stati fattura**: `DRAFT → ISSUED (lock) → PAID`

---

### 6.4 Mail Reader Service

> Documentazione completa: [`services/mail-reader-service.md`](./services/mail-reader-service.md)

**Missione**: Lettura, parsing e archiviazione email da caselle monitorate.

---

### 6.5 Gateway

> Documentazione completa: [`services/gateway.md`](./services/gateway.md)

**Missione**: Unico punto di ingresso — autenticazione Firebase, routing dinamico, service discovery.

---

## 7. Standard Globali

> 📌 **Documentazione completa e definitiva:** [`architecture/global-standards.md`](./architecture/global-standards.md)

Il documento copre:

| Sezione | Contenuto |
|---|---|
| [§1 Principi](./architecture/global-standards.md#1-principi-fondamentali) | Gateway-first, Soft Delete, Immutabilità ERP, Identity Propagation |
| [§2 Header & Identity](./architecture/global-standards.md#2-header-standard-nexus) | `X-Nexus-*`, `Guard()`, `FromContext()`, `HasRole()`, admin bypass |
| [§3 Response Format](./architecture/global-standards.md#3-formato-standard-delle-risposte) | `StandardResponse`, `Success()`, `Failure()` |
| [§4 Codici Errore](./architecture/global-standards.md#4-codici-errore-standard) | Tutti gli 11 `nexus.Err*` con HTTP status |
| [§5 Guard Middleware](./architecture/global-standards.md#5-nexus-guard--middleware-obbligatorio) | Setup route pubbliche vs private |
| [§6 Service Discovery](./architecture/global-standards.md#6-service-discovery--pattern-di-registrazione) | `ServiceDefinition`, handshake, retry |
| [§7 Firebase Auth](./architecture/global-standards.md#7-autenticazione-firebase--solo-gateway) | Solo Gateway, ADC, Custom Claims |
| [§8 NexusDoc & Repository](./architecture/global-standards.md#8-modello-dati-base--nexusdoc) | ORM Firestore, `Create/Update/SoftDelete/IsLocked` |
| [§9 Navigator Pattern](./architecture/global-standards.md#9-navigator-pattern--filtri--ordinamento-firestore) | `ApplyNavigator()`, filtri, sort, soft-delete automatico |
| [§10 NexusClient](./architecture/global-standards.md#10-chiamate-inter-servizio--nexusclient) | Propagazione identità inter-servizio |
| [§11 GCS Signed URL](./architecture/global-standards.md#11-storage-gcs--signed-url-v4) | Upload/download V4, ADC, flussi |
| [§12 Stack Tecnologico](./architecture/global-standards.md#12-stack-tecnologico-standard) | Go 1.21+, Gin, Firestore, Cloud Run |
| [§13 Variabili d'Ambiente](./architecture/global-standards.md#13-variabili-dambiente--schema-completo) | Schema completo di tutte le env var |
| [§14 Checklist Deploy](./architecture/global-standards.md#14-checklist-nuovo-microservizio) | 16 punti obbligatori prima del deploy |

> 🗂️ **Modello Dati Condiviso (Frontend + BFF):** [`architecture/data-model.md`](./architecture/data-model.md)
> TypeScript types, relazioni tra entità, state machine, convenzioni API e validazioni Zod per il portal frontend.

---

## 8. Contratti e JSON Schema

> Contratti formali: [`contracts/schemas.json`](./contracts/schemas.json)

| Schema | Collezione Firestore | Stato |
|---|---|---|
| `NexusDoc` | (struct base) | ✅ Disponibile |
| `NexusEntity` | `entities` | ✅ Disponibile |
| `NexusDocument` | `documents` | ✅ Disponibile |
| `NexusInvoice` | `invoices` | ✅ Disponibile |
| `NexusLedgerEntry` | `ledger_entries` | ✅ Disponibile |

---

## 9. Open Issues e Roadmap

### Issue risolti

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | `inance-service.md` — typo nel nome file | ✅ Rinominato `finance-service.md` |
| 🟡 Media | `contracts/schemas.json` — schema incompleto | ✅ Aggiornato con tutti i modelli |
| 🟡 Media | `README.md` — vuoto | ✅ Source of truth completo |
| 🟡 Media | `global-standards.md` — mancavano NexusClient, Navigator, GCS | ✅ Sezioni integrate |

### Issue aperti

| Priorità | Issue | Assegnato a |
|---|---|---|
| 🔴 Alta | `PORT` hardcoded in alcuni servizi (non legge `os.Getenv`) | ⏳ Aperto |
| 🔴 Alta | Validazione `vatNumber` mancante nel Registry Service | ⏳ Aperto |
| 🟡 Media | `X-Nexus-Trace-ID` non propagato da `NexusClient` | ⏳ Aperto |
| 🟡 Media | Endpoint GET/PATCH/DELETE commentati in Registry Service | ⏳ Aperto |
| 🟢 Bassa | `/_discover` non testato e2e | ⏳ Aperto |

### Roadmap Documentale

- [x] `services/registry-service.md`
- [x] `services/nexus-document-service.md`
- [x] `services/finance-service.md`
- [x] `services/mail-reader-service.md`
- [x] `architecture/global-standards.md`
- [x] `contracts/schemas.json`
- [x] `architecture/data-model.md` — TypeScript types, relazioni, state machine, API BFF
- [ ] `services/gateway.md` — routing rules e discovery
- [ ] `architecture/deployment.md` — diagramma infrastruttura GCP
- [ ] `architecture/data-flows.md` — flussi inter-servizio
- [ ] `adr/` — Architecture Decision Records

---

*Documento aggiornato il 2026-05-06 v1.2.0 — da aggiornare ad ogni decisione architetturale rilevante.*
