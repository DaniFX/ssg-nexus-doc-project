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


## 🗄️ Nexus Repository: Il Wrapper Firestore
L'obiettivo di questo wrapper è trasformare le query HTTP (es. ?status=PAID&sort=-createdAt) direttamente in query Firestore, gestendo al contempo i metadati obbligatori.

### 1. Interfaccia Base del Repository
Ogni entità nel sistema (Socio, Fattura, Documento) deve implementare questa logica tramite l'SDK.

```Go
// pkg/nexus/repository.go
package nexus

import (
	"context"
	"time"

	"cloud.google.com/go/firestore"
)

type Repository struct {
	client     *firestore.Client
	collection string
}

func NewRepository(client *firestore.Client, collection string) *Repository {
	return &Repository{
		client:     client,
		collection: collection,
	}
}

// NexusDoc definisce la struttura minima per la persistenza
type NexusDoc struct {
	ID        string    `firestore:"id"`
	CreatedAt time.Time `firestore:"createdAt"`
	UpdatedAt time.Time `firestore:"updatedAt"`
	CreatedBy string    `firestore:"createdBy"`
}
```

### 2. Il "Navigator": Filtri e Paginazione Automatica
Implementiamo una funzione che traduce i parametri URL in filtri Firestore. Questo permette al frontend di interrogare i servizi in modo standard.

```Go
// ApplyNavigator applica filtri basati sui parametri della richiesta
func (r *Repository) ApplyNavigator(query firestore.Query, filters map[string]string) firestore.Query {
	for key, value := range filters {
		switch key {
		case "sort":
			if value[0] == '-' {
				query = query.OrderBy(value[1:], firestore.Desc)
			} else {
				query = query.OrderBy(value, firestore.Asc)
			}
		case "limit":
			// logica per limit
		default:
			// Filtro di uguaglianza standard (es. status=PAID)
			query = query.Where(key, "==", value)
		}
	}
	return query
}
```

### 3. Logica ERP: Salvataggio con Timestamp e Identity
Ogni volta che creiamo o aggiorniamo un record, l'SDK deve iniettare i metadati di tracciamento estratti dal Nexus Context.

```Go
func (r *Repository) Create(ctx context.Context, id string, data map[string]interface{}) error {
	identity := FromContext(ctx) // Estratto dal middleware NexusGuard
	
	data["id"] = id
	data["createdAt"] = firestore.ServerTimestamp
	data["updatedAt"] = firestore.ServerTimestamp
	data["createdBy"] = identity.UserID

	_, err := r.client.Collection(r.collection).Doc(id).Set(ctx, data)
	return err
}
```