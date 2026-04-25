# 💰 Finance Service (Nexus Ledger) - Specifiche Tecniche
Il Finance Service è il modulo ERP di SSG Nexus. Gestisce il ciclo attivo e passivo, la conformità fiscale e l'integrità dei saldi.

## 1. Modelli Dati Avanzati (Firestore)
Collection: invoices
Questa collezione gestisce documenti fiscali (Fatture, Note di Credito, Ricevute).

```JSON
{
  "id": "INV-2024-0001",
  "externalId": "SDI-123456", // ID Sistema di Interscambio (opzionale)
  "type": "INVOICE | CREDIT_NOTE | RECEIPT",
  "direction": "INBOUND | OUTBOUND", 
  "status": "DRAFT | ISSUED | PAID | OVERDUE | CANCELLED",
  "issuer": { // Copia statica dal Registry al momento dell'emissione
    "entityId": "nexus-id-azienda",
    "name": "SSG Global Srl",
    "vat": "IT1234567890"
  },
  "receiver": {
    "entityId": "nexus-id-cliente",
    "name": "Mario Rossi",
    "taxCode": "RSSMRA80A01H501U"
  },
  "items": [
    {
      "description": "Consulenza Sviluppo Software",
      "quantity": 10,
      "unitPrice": 75.00,
      "vatRate": 22,
      "total": 750.00
    }
  ],
  "totals": {
    "net": 750.00,
    "tax": 165.00,
    "gross": 915.00,
    "currency": "EUR"
  },
  "dates": {
    "document": "2024-05-20T00:00:00Z",
    "due": "2024-06-20T00:00:00Z",
    "paid": null
  },
  "documentRef": "DOC-ID-FROM-VAULT", // Link al file fisico nel Document Service
  "metadata": {
    "immutable": false // Diventa true quando status = ISSUED
  }
}
```
### Collection: ledger_entries (Libro Giornale)
Ogni transazione finanziaria genera un record qui per la riconciliazione.

```JSON
{
  "id": "TX-999",
  "entityId": "nexus-id-cliente",
  "invoiceId": "INV-2024-0001",
  "amount": 915.00,
  "type": "DEBIT | CREDIT",
  "method": "STRIPE | BANK_TRANSFER | CASH",
  "timestamp": "2024-05-21T10:30:00Z"
}
```
## 2. Logica di Business (Regole ERP)
Numerazione Sequenziale: Il servizio garantisce che le fatture OUTBOUND abbiano numeri univoci e sequenziali per anno fiscale.

Locking: Quando una fattura passa a ISSUED, il Nexus SDK deve bloccare qualsiasi operazione di UPDATE o DELETE sul documento.

Riconciliazione: Un pagamento registrato nel ledger_entries fa scattare un controllo: se la somma dei pagamenti copre il totals.gross della fattura, lo stato passa automaticamente a PAID.

# 🛠️ Nexus SDK - Struttura Iniziale (Go)
Ora che abbiamo il piano, iniziamo a scrivere il Nexus SDK. Questo codice risiederà in un repository separato o in una cartella /pkg condivisa.

## 1. Definizione del Nexus Context
Il primo passo è creare un modo standard per gestire l'identità utente tra i servizi.

```Go
// pkg/nexus/context.go
package nexus

import (
	"context"
)

type contextKey string

const (
	UserIDKey contextKey = "X-Nexus-User-ID"
	RoleKey   contextKey = "X-Nexus-Role"
)

type NexusIdentity struct {
	UserID string
	Role   string
}

// FromContext estrae l'identità dagli header iniettati dal Gateway
func FromContext(ctx context.Context) NexusIdentity {
	return NexusIdentity{
		UserID: ctx.Value(UserIDKey).(string),
		Role:   ctx.Value(RoleKey).(string),
	}
}
```
## 2. Standard Responder (Middleware Gin)
Tutti i servizi devono rispondere con lo stesso formato.

```Go
// pkg/nexus/response.go
package nexus

import (
	"github.com/gin-gonic/gin"
)

type Response struct {
	Success bool        `json:"success"`
	Data    interface{} `json:"data,omitempty"`
	Error   *NexusError `json:"error,omitempty"`
}

type NexusError struct {
	Code    string      `json:"code"`
	Message string      `json:"message"`
	Details interface{} `json:"details,omitempty"`
}

func ErrorResponse(c *gin.Context, httpCode int, errCode string, msg string) {
	c.JSON(httpCode, Response{
		Success: false,
		Error: &NexusError{
			Code:    errCode,
			Message: msg,
		},
	})
}

func SuccessResponse(c *gin.Context, data interface{}) {
	c.JSON(200, Response{
		Success: true,
		Data:    data,
	})
}
```

## 3. Middleware di Sicurezza (Nexus Guard)
Questo middleware verifica che il Gateway abbia effettivamente passato l'autenticazione.

```Go
// pkg/nexus/middleware.go
func NexusGuard() gin.HandlerFunc {
	return func(c *gin.Context) {
		userID := c.GetHeader("X-Nexus-User-ID")
		if userID == "" {
			ErrorResponse(c, 401, "UNAUTHORIZED", "Missing Nexus Identity")
			c.Abort()
			return
		}
		// Inietta nel contesto Go per l'uso nei servizi
		ctx := context.WithValue(c.Request.Context(), UserIDKey, userID)
		ctx = context.WithValue(ctx, RoleKey, c.GetHeader("X-Nexus-Role"))
		c.Request = c.Request.WithContext(ctx)
		c.Next()
	}
}
```