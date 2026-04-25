## 🚨 Protocollo Gestione Errori
Ogni microservizio deve mappare i propri errori interni nei seguenti codici Nexus standard:

| Codice Nexus | HTTP Status | Descrizione |
| :--- | :--- | :--- |
| `ERR_UNAUTHORIZED` | 401 | Token mancante o non valido. |
| `ERR_FORBIDDEN` | 403 | Permessi insufficienti per la risorsa/azione. |
| `ERR_NOT_FOUND` | 404 | Risorsa non trovata su Firestore. |
| `ERR_IMMUTABLE_RECORD` | 409 | Tentativo di modifica di un record locked (Logica ERP). |
| `ERR_VALIDATION_FAILED` | 400 | Payload non conforme allo schema JSON. |
| `ERR_INTERNAL` | 500 | Errore generico lato server o GCP. |

## 📡 Headers Obbligatori
Oltre a `X-Nexus-User-ID` e `X-Nexus-Role`, aggiungiamo:
- `X-Nexus-Trace-ID`: Per il logging distribuito (Cloud Trace).

## 📋 Standard per il ssg-nexus-doc-project
Aggiungi queste regole nella sezione global-standards.md:

- Immutabilità: Se un documento ha il flag immutable: true (tipico del Finance Service), il wrapper Update dell'SDK deve restituire un errore ERR_IMMUTABLE_RECORD.

- Soft Delete: Non cancelliamo mai fisicamente i dati ERP. L'SDK deve implementare un campo deletedAt.

- Audit Log: Ogni operazione di scrittura deve essere loggata con l'ID utente (X-Nexus-User-ID) per scopi di auditing.