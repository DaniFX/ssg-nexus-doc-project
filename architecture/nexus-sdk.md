# 🛠️ Nexus SDK (Go)
Il Nexus SDK è una libreria interna sviluppata in Go 1.21+ che standardizza il comportamento di tutti i microservizi SSG Nexus. Il suo scopo è astrarre la complessità dell'infrastruttura GCP e garantire che ogni servizio rispetti il contratto di comunicazione.

## 1. Funzionalità Core
### A. Nexus Context & Auth Middleware
Gestisce l'estrazione degli header iniettati dal Gateway (X-Nexus-User-ID, X-Nexus-Role) e li inserisce nel context.Context di Go.

Standard: Ogni richiesta downstream deve avere il contesto arricchito per tracciare chi sta compiendo l'azione.

Validazione: Fornisce helper per verificare se l'utente ha i permessi necessari (es. nexus.HasRole(ctx, "admin")).

### B. Standardized Responder
Garantisce che ogni microservizio risponda con lo stesso formato JSON definito nel progetto:

Successo: { "success": true, "data": { ... }, "meta": { ... } }.

Errore: { "success": false, "error": { "code": "ERR_CODE", "message": "..." } }.

### C. Firestore Wrapper (Nexus Repository)
Un wrapper attorno al client Firestore ufficiale di Google per:

Gestire automaticamente i timestamp createdAt e updatedAt.

Implementare il "Navigator" (filtri, paginazione e ordinamento standardizzati via URL query).

Gestione delle transazioni ERP-style.

### D. Service Discovery Helper
Implementa automaticamente l'endpoint /_discover richiesto dal Gateway.

Il microservizio deve solo passare una struct con la definizione dei propri endpoint all'SDK, che si occupa di esporre il JSON corretto.