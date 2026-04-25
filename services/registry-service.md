## 1. Definizione
Il Registry Service è il custode delle "Anagrafiche Polimorfiche". Gestisce ogni entità che interagisce con SSG Nexus (Soci, Clienti, Prospect, Fornitori, Dipendenti) garantendo l'integrità dei dati core necessari alla logica ERP.

## 2. Modello Dati (Firestore - Collection entities)
Useremo un approccio Schema-flexible per i dettagli, ma Schema-strict per i metadati ERP.

```JSON
{
  "id": "UUID_OR_FIREBASE_ID",
  "type": "PERSON | ORGANIZATION",
  "subTypes": ["MEMBER", "CUSTOMER"], 
  "status": "ACTIVE | INACTIVE | PROSPECT",
  "coreData": {
    "displayName": "string",
    "email": "string (unique)",
    "taxCode": "string (CF o P.IVA)",
    "vatNumber": "string (optional)"
  },
  "extData": { 
    // Campi dinamici basati sui subTypes
    // Esempio per MEMBER: "membershipDate", "tshirtSize"
    // Esempio per CUSTOMER: "billingAddress", "paymentTerms"
  },
  "nexusMetadata": {
    "createdAt": "timestamp",
    "updatedAt": "timestamp",
    "createdBy": "userId"
  }
}
```
## 3. Logica a "Volumi" e Permessi
Per gestire l'accesso ai dati in modo sicuro tramite il Gateway:

Visibilità Pubblica/Social: Solo displayName e photoURL.

Visibilità Gestionale: Accesso completo ai dati fiscali solo per utenti con ruolo admin o operator.

Self-Service: L'utente può modificare solo i propri extData tramite l'Header X-Nexus-User-ID iniettato dal Gateway.

## 4. Endpoint Principali (/_discover)
GET /entities: Lista filtrabile per type e status (Navigator).

POST /entities: Creazione con validazione rigorosa dei dati fiscali (Logica ERP).

GET /entities/:id: Dettaglio completo.

PATCH /entities/:id: Aggiornamento parziale (senza alterare lo storico critico).

#### 🚀 Azioni suggerite per il repo ssg-nexus-doc-project:
Crea una cartella /architecture: Inserisci un file global-standards.md che definisca come i servizi devono rispondere (formato JSON, gestione errori, header X-Nexus-*).

Crea una cartella /services: Inserisci il file registry-service.md sopra descritto.

File README.md: Definisci la visione di SSG Nexus come sistema unificato.