# 💰 Finance Service (Nexus Ledger) - Specifiche Tecniche
Il Finance Service è il modulo ERP di SSG Nexus. Gestisce il ciclo attivo e passivo, la conformità fiscale e l'integrità dei saldi.

## 1. Modelli Dati Avanzati (Firestore)

### Collection: `invoices`
Questa collezione gestisce documenti fiscali (Fatture, Note di Credito, Ricevute).

```json
{
  "id": "INV-2024-0001",
  "externalId": "SDI-123456",
  "type": "INVOICE | CREDIT_NOTE | RECEIPT",
  "direction": "INBOUND | OUTBOUND",
  "status": "DRAFT | ISSUED | PAID | OVERDUE | CANCELLED",
  "issuer": {
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
  "documentRef": "DOC-ID-FROM-VAULT",
  "metadata": {
    "immutable": false
  }
}
```

> ⚠️ `issuer` e `receiver` sono **snapshot statici** copiati dal Registry al momento dell'emissione. Non vengono mai aggiornati.

> ⚠️ `metadata.immutable` diventa `true` quando `status = ISSUED`. Da quel momento il Nexus SDK blocca UPDATE e DELETE.

### Collection: `ledger_entries` (Libro Giornale)
Ogni transazione finanziaria genera un record qui per la riconciliazione.

```json
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

---

## 2. Logica di Business (Regole ERP)

- **Numerazione Sequenziale**: Le fatture OUTBOUND devono avere numeri univoci e sequenziali per anno fiscale.
- **Locking**: Quando una fattura passa a `ISSUED`, il Nexus SDK blocca qualsiasi operazione di UPDATE o DELETE tramite `IsLocked()`.
- **Riconciliazione**: Quando la somma dei `ledger_entries` per una fattura copre `totals.gross`, lo stato passa automaticamente a `PAID`.
- **Snapshot fiscale**: I dati di `issuer` e `receiver` vengono copiati staticamente dal Registry al momento dell'emissione.

---

## 3. Nexus SDK — Componenti usati da questo servizio

### Nexus Context

```go
// pkg/nexus/context.go
type NexusIdentity struct {
    UserID string
    Role   string
}

func FromContext(ctx context.Context) NexusIdentity { ... }
```

### Standard Responder

```go
// Successo
nexus.SuccessResponse(c, data)

// Errore
nexus.ErrorResponse(c, 409, "ERR_IMMUTABLE_RECORD", "Fattura già emessa")
```

### Nexus Guard Middleware

```go
func NexusGuard() gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetHeader("X-Nexus-User-ID")
        if userID == "" {
            nexus.ErrorResponse(c, 401, "ERR_UNAUTHORIZED", "Missing Nexus Identity")
            c.Abort()
            return
        }
        ctx := context.WithValue(c.Request.Context(), nexus.UserIDKey, userID)
        ctx = context.WithValue(ctx, nexus.RoleKey, c.GetHeader("X-Nexus-Role"))
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}
```

---

## 4. Endpoint Principali

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/invoices` | Lista fatture via Navigator |
| POST | `/invoices` | Creazione fattura (DRAFT) |
| GET | `/invoices/:id` | Dettaglio fattura |
| PATCH | `/invoices/:id/issue` | Emissione (DRAFT -> ISSUED, lock) |
| POST | `/invoices/:id/payments` | Registrazione pagamento su ledger |
| GET | `/ledger` | Lista movimenti contabili |

---

*Parte del progetto SSG Nexus — vedere README.md per gli standard globali.*
