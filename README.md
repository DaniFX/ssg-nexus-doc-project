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
