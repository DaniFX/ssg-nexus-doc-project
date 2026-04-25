## 1. Missione
Gestire il ciclo di vita dei file (upload, archiviazione, metadatazione e sicurezza) fungendo da ponte tra Google Cloud Storage (fisico) e Firestore (logico).

## 2. Il Concetto di "Attachment"
In SSG Nexus, un documento non esiste quasi mai da solo. È sempre "agganciato" a un'entità (Registry) o a una transazione (Finance).

Proprietario (Owner): L'ID dell'entità che ha caricato il file.

Parent: L'ID della risorsa a cui è collegato (es. ID di un Socio o ID di una Fattura).

## 3. Modello Dati (Firestore - Collection documents)
```JSON
{
  "id": "UUID",
  "fileName": "carta_identita_rossi.pdf",
  "mimeType": "application/pdf",
  "size": 102456,
  "storagePath": "entities/{entityId}/docs/{docId}.pdf",
  "category": "IDENTITY_DOC | CONTRACT | INVOICE_ATTACHMENT | LOGO",
  "metadata": {
    "expiryDate": "2030-01-01T00:00:00Z", // Critico per documenti d'identità
    "isVerified": false,
    "version": 1
  },
  "relation": {
    "parentType": "ENTITY | INVOICE | PROJECT",
    "parentId": "ID_DELLA_RISORSA"
  },
  "accessControl": {
    "isPublic": false,
    "allowedRoles": ["admin", "hr"]
  },
  "nexusMetadata": {
    "createdAt": "timestamp",
    "createdBy": "userId"
  }
}
```
## 4. Flusso di Caricamento (Secure Upload)
Per non sovraccaricare il microservizio Go, useremo i Signed URLs:

Request: Il Frontend chiede al Document Service: "Voglio caricare un PDF per il socio X".

Grant: Il servizio verifica i permessi e restituisce un Signed URL temporaneo di Google Cloud Storage.

Direct Upload: Il Frontend carica il file direttamente su GCS.

Finalize: Il Frontend conferma l'upload al servizio, che crea il record su Firestore e triggera eventuali analisi (es. OCR se necessario).

## 5. Integrazione con Registry Service
Nel registry-service, l'anagrafica di un socio non conterrà il file, ma un array di riferimenti o una query dinamica:

Esempio Query: GET /documents?relation.parentId={memberId}

Risultato: Lista di tutti i documenti (ID, nome, categoria) pronti per essere visualizzati o scaricati.

## 📂 Struttura Cartelle per il tuo Repo Documentale
Ti consiglio di organizzare il repo ssg-nexus-doc-project così:

```Plaintext
/architecture
  ├── global-standards.md      # Standard API, Errori, Headers
  └── auth-flow.md             # Dettaglio Firebase Auth + Gateway
/services
  ├── registry-service.md      # Anagrafiche e CRM
  ├── document-service.md      # Gestione file (questo file)
  └── finance-service.md       # (Prossimamente) Logica ERP/Fatture
/contracts
  └── schemas.json             # JSON Schemas per la validazione
  ```