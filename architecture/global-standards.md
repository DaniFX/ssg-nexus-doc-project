definisce come i servizi devono rispondere (formato JSON, gestione errori, header X-Nexus-*).

Da implementare

## 📋 Standard per il ssg-nexus-doc-project
Aggiungi queste regole nella sezione global-standards.md:

- Immutabilità: Se un documento ha il flag immutable: true (tipico del Finance Service), il wrapper Update dell'SDK deve restituire un errore ERR_IMMUTABLE_RECORD.

- Soft Delete: Non cancelliamo mai fisicamente i dati ERP. L'SDK deve implementare un campo deletedAt.

- Audit Log: Ogni operazione di scrittura deve essere loggata con l'ID utente (X-Nexus-User-ID) per scopi di auditing.