# 📧 SSG Mail Reader Service — Specifiche Tecniche

> **Repository:** [`ssg-mail-reader-service`](https://github.com/DaniFX/ssg-mail-reader-service)
> **Stack:** Go 1.21+, Gin, `go-imap` (emersion), TLS 1.2+
> **Ruolo:** Microservizio stateless per la lettura e gestione di caselle email via protocollo IMAP.
> **Caso d'uso primario:** Lettura di caselle **PEC** (Posta Elettronica Certificata) per l'identificazione e il recupero di **Fatture Elettroniche** (allegati `.xml` e `.p7m`).

---

## 1. Caratteristica Architetturale Chiave: Stateless IMAP

A differenza degli altri microservizi Nexus, il Mail Reader **non persiste credenziali** su Firestore. Le credenziali IMAP vengono passate **per ogni richiesta** tramite header HTTP dedicati (`X-Imap-Host`, `X-Imap-User`, `X-Imap-Pass`).

Questo design permette a un singolo servizio di gestire caselle email di **clienti diversi** senza configurazione preventiva, rendendolo ideale per integrarsi con le PEC aziendali di diverse ditte nel flusso ERP.

```
Client
  │
  │ X-Imap-Host: imaps.pec-provider.it:993
  │ X-Imap-User: azienda@pec.it
  │ X-Imap-Pass: <password>
  ▼
+---------------------------------------+
|           SSG Gateway                 |
|  [1] FirebaseAuthMiddleware (JWT)     |
|  [2] Proxy con OIDC Token             |
+---------------------------------------+
             │
             ▼
+---------------------------------------+
|      Mail Reader Service              |
|  [3] nexus.Guard() → X-Nexus-User-ID |
|  [4] RequireImapHeaders() middleware  |
|      → c.Set("imapCreds")            |
|  [5] Handler → MailService           |
|  [6] client.DialTLS() → IMAP server  |
+---------------------------------------+
             │
             ▼
    Server IMAP/PEC (TLS 1.2+)
```

---

## 2. Struttura del Progetto

```
ssg-mail-reader-service/
├── cmd/server/
│   └── main.go              # Bootstrap: router, middleware, handshake Gateway
├── internal/
│   ├── handlers/
│   │   ├── mail.go             # Handler HTTP per tutte le operazioni IMAP
│   │   ├── discovery.go        # GetDiscoveryDoc() — contratto API completo
│   │   └── health.go           # GET /_health
│   ├── middleware/
│   │   └── imap_auth.go        # RequireImapHeaders() — estrae credenziali dagli header
│   ├── models/
│   │   ├── mail.go             # SearchCriteria, EmailPreview, EmailDetail
│   │   └── response.go         # NewSuccessResponse, NewErrorResponse
│   └── service/
│       ├── imap_client.go      # MailService — client IMAP reale
│       └── imap_integration_test.go
├── .env.example
└── Dockerfile
```

---

## 3. Bootstrap (`main.go`)

Sequenza di avvio:

1. `gin.SetMode(gin.ReleaseMode)` se `ENV=production`
2. `registerToGateway()` — lanciato in goroutine background
3. `router.Group("/api/v1/mail")` + `middleware.RequireImapHeaders()`
4. Registrazione dei 6 endpoint
5. `router.Run(":" + port)`

### 3.1 `registerToGateway()` — Handshake con retry

Il servizio usa un meccanismo di registrazione **custom** (non `nexus.StartGatewayHandshake`), con logica di retry esplicita:

```go
func registerToGateway() {
    // Lancia la registrazione in goroutine (non blocca il boot)
    go func() {
        client := &http.Client{Timeout: 10 * time.Second}

        for i := 0; i < 5; i++ {
            req, _ := http.NewRequest("POST", gatewayURL+"/internal/register", body)
            req.Header.Set("Content-Type", "application/json")
            req.Header.Set("X-Internal-Secret", internalSecret)
            req.Header.Set("X-Service-Url", serviceURL)  // URL del servizio per il Gateway

            resp, err := client.Do(req)
            if err == nil && resp.StatusCode == http.StatusOK {
                log.Println("✅ Handshake completato")
                return
            }
            time.Sleep(5 * time.Second)  // attende 5 sec tra tentativi
        }
        log.Println("❌ Impossibile registrarsi dopo 5 tentativi")
    }()
}
```

**Header inviati al Gateway:**

| Header | Valore |
|---|---|
| `Content-Type` | `application/json` |
| `X-Internal-Secret` | `$INTERNAL_SECRET` |
| `X-Service-Url` | `$SERVICE_URL` (URL Cloud Run del servizio) |

**Payload:** JSON della `GetDiscoveryDoc()` (contratto API completo con tutti gli endpoint).

> **Nota:** A differenza degli altri servizi che usano `nexus.StartGatewayHandshake()`, questo servizio implementa la registrazione manualmente con 5 retry e backoff fisso da 5 secondi. Valutare allineamento all'SDK.

---

## 4. Autenticazione e Middleware Chain

La catena di middleware per le route `/api/v1/mail` è composta da **due livelli** in serie:

### 4.1 `nexus.Guard()` — Identità Nexus

Primo middleware. Verifica che `X-Nexus-User-ID` sia presente (iniettato dal Gateway). Blocca con `401` se assente.

### 4.2 `RequireImapHeaders()` — Credenziali IMAP

**File:** `internal/middleware/imap_auth.go`

Secondo middleware. Verifica la presenza di tutti e tre gli header IMAP. In caso di assenza risponde `400 MISSING_IMAP_CREDENTIALS` e chiama `c.Abort()`.

```go
func RequireImapHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        host := c.GetHeader("X-Imap-Host")
        user := c.GetHeader("X-Imap-User")
        pass := c.GetHeader("X-Imap-Pass")

        if host == "" || user == "" || pass == "" {
            c.JSON(400, models.NewErrorResponse("MISSING_IMAP_CREDENTIALS", "...", nil))
            c.Abort()
            return
        }
        c.Set("imapCreds", ImapCredentials{Host: host, Username: user, Password: pass})
        c.Next()
    }
}
```

### 4.3 Header richiesti per ogni richiesta

| Header | Tipo | Esempio | Note |
|---|---|---|---|
| `X-Nexus-User-ID` | `string` | `uid_firebase_xyz` | Iniettato dal Gateway (obbligatorio) |
| `X-Imap-Host` | `string` | `imaps.pec.it:993` | Host + porta del server IMAP/PEC |
| `X-Imap-User` | `string` | `azienda@pec.it` | Username casella email |
| `X-Imap-Pass` | `string` | `<password>` | Password casella email |

> ⚠️ **Sicurezza**: Le credenziali IMAP viaggiano in chiaro negli header HTTP. La connessione client → Gateway **deve** essere HTTPS. In produzione valutare l'uso di token temporanei o vault secrets invece della password diretta.

---

## 5. Client IMAP — `MailService`

**File:** `internal/service/imap_client.go`
**Libreria:** [`github.com/emersion/go-imap`](https://github.com/emersion/go-imap)

### 5.1 Connessione TLS

Ogni operazione apre una nuova connessione TLS dedicata e la chiude con `defer c.Logout()`. Il servizio è **completamente stateless** a livello di connessioni.

```go
func (s *MailService) connect() (*client.Client, error) {
    // Estrae solo l'host (senza porta) per ServerName TLS
    // necessario per i provider PEC (Aruba, Namirial, Legalmail)
    tlsConfig := &tls.Config{
        ServerName: hostOnly,
        MinVersion: tls.VersionTLS12,
    }
    c, err := client.DialTLS(s.Host, tlsConfig)
    c.Login(s.Username, s.Password)
}
```

### 5.2 Operazioni disponibili

| Metodo | Comando IMAP | Descrizione |
|---|---|---|
| `ListFolders()` | `LIST "" *` | Recupera tutte le cartelle della casella |
| `Search(criteria)` | `UID SEARCH` + `UID FETCH ENVELOPE` | Ricerca per Subject/From/Body, ritorna preview |
| `GetMessage(folder, uid)` | `UID FETCH BODY[] ENVELOPE` | Fetch completo: body text, HTML, allegati |
| `CreateFolder(name)` | `CREATE` | Crea nuova cartella IMAP |
| `MoveMessage(src, uid, dst)` | `UID MOVE` | Sposta email tra cartelle |
| `DeleteMessage(folder, uid)` | `UID STORE +FLAGS \\Deleted` | Marca per eliminazione (non rimuove fisicamente) |

> ⚠️ **`DeleteMessage`**: aggiunge il flag `\Deleted` IMAP. La rimozione fisica richiede `EXPUNGE`, non implementato. Valutare se aggiungere `EXPUNGE` esplicito dopo il `UidStore`.

### 5.3 Identificazione Fatture Elettroniche

In `GetMessage()`, il parser MIME (`go-message/mail`) scansiona gli allegati e identifica le Fatture Elettroniche per estensione:

```go
case *mail.AttachmentHeader:
    filename, _ := h.Filename()
    fnLower := strings.ToLower(filename)
    // Fattura XML standard SDI o firmata CAdES (.p7m)
    if strings.HasSuffix(fnLower, ".xml") || strings.HasSuffix(fnLower, ".p7m") {
        // TODO: salvataggio su Cloud Storage o analisi XML SDI
    }
```

> 🔴 **TODO aperto**: La logica di salvataggio su GCS e analisi XML SDI è predisposta ma non implementata. È il prossimo step naturale per integrare il flusso Fattura Elettronica con il Finance Service.

---

## 6. API Endpoints

Base path: `/api/v1/mail` — tutti protetti da `RequireImapHeaders()` + `nexus.Guard()`.

| Metodo | Path | Handler | Descrizione |
|---|---|---|---|
| `GET` | `/api/v1/mail/folders` | `GetFolders` | Lista cartelle IMAP |
| `POST` | `/api/v1/mail/folders` | `CreateFolder` | Crea nuova cartella |
| `POST` | `/api/v1/mail/search` | `SearchEmails` | Ricerca email con filtri |
| `GET` | `/api/v1/mail/messages/:uid` | `GetMessage` | Dettaglio email completo |
| `PUT` | `/api/v1/mail/messages/:uid/move` | `MoveMessage` | Sposta email in altra cartella |
| `DELETE` | `/api/v1/mail/messages/:uid` | `DeleteMessage` | Marca email come eliminata |
| `GET` | `/_discover` | `GetDiscovery` | Contratto API (pubblico) |

### Esempi Richiesta/Risposta

**`POST /api/v1/mail/search`**
```json
// Request
{
  "folder": "INBOX",
  "subject": "Fattura",
  "from": "sdi@pec.fatturapa.it"
}
// Response
{
  "success": true,
  "data": {
    "emails": [
      { "uid": 1042, "subject": "Fattura n. 123/2026", "from": "sdi@pec.fatturapa.it", "date": "2026-04-15T10:30:00Z" }
    ]
  }
}
```

**`GET /api/v1/mail/messages/1042?folder=INBOX`**
```json
{
  "success": true,
  "data": {
    "message": {
      "uid": 1042,
      "subject": "Fattura n. 123/2026",
      "from": "sdi@pec.fatturapa.it",
      "date": "2026-04-15T10:30:00Z",
      "body": "Testo della email...",
      "htmlBody": "<html>...</html>"
    }
  }
}
```

**`PUT /api/v1/mail/messages/1042/move?folder=INBOX`**
```json
{ "destinationFolder": "Fatture/2026" }
```

---

## 7. Modelli di Dati

**File:** `internal/models/mail.go`

```go
// SearchCriteria — payload per POST /search
type SearchCriteria struct {
    Folder       string `json:"folder"`               // default "INBOX"
    Subject      string `json:"subject,omitempty"`
    From         string `json:"from,omitempty"`
    BodyContains string `json:"bodyContains,omitempty"`
}

// EmailPreview — metadati, usato nelle liste (senza body)
type EmailPreview struct {
    UID     uint32    `json:"uid"`
    Subject string    `json:"subject"`
    From    string    `json:"from"`
    Date    time.Time `json:"date"`
}

// EmailDetail — email completa (estende EmailPreview)
type EmailDetail struct {
    EmailPreview
    Body     string `json:"body"`
    HTMLBody string `json:"htmlBody,omitempty"`
}
```

---

## 8. Service Discovery

**File:** `internal/handlers/discovery.go`

A differenza degli altri servizi che usano `nexus.ServiceDefinition`, questo servizio costruisce il discovery doc come `gin.H` direttamente nel codice. Il contratto è completo e include `inputSchema` per gli endpoint con body.

- Servizio: `mail-reader-service` | Versione: `1.0.0`
- Tutti e 6 gli endpoint dichiarano `authRequired: true`
- Endpoint di ispezione esposto: `GET /_discover`

> **Nota:** Il formato di discovery è diverso dallo standard `nexus.ServiceDefinition` degli altri servizi. Valutare allineamento all'SDK per uniformità.

---

## 9. Variabili d'Ambiente

| Variabile | Descrizione | Obbligatoria |
|---|---|---|
| `GATEWAY_URL` | URL del Gateway per la registrazione push | ✅ Sì |
| `SERVICE_URL` | URL Cloud Run di questo servizio | ✅ Sì |
| `INTERNAL_SECRET` | Segreto condiviso per l'handshake con il Gateway | ✅ Sì |
| `ENV` | Se `production` abilita `gin.ReleaseMode` | No |
| `PORT` | Porta HTTP (default `8080`) | No |

---

## 10. Dipendenze Principali

| Package | Scopo |
|---|---|
| `github.com/emersion/go-imap` | Client IMAP RFC 3501 |
| `github.com/emersion/go-message` | Parser MIME multipart (body + allegati) |
| `github.com/gin-gonic/gin` | HTTP framework |
| `github.com/DaniFX/ssg-nexus-sdk` | Guard, Response standard |

---

## 11. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | Credenziali IMAP in chiaro negli header — password diretta senza cifratura aggiuntiva | ⚠️ Design by choice, documentare rischio |
| 🔴 Alta | Logica salvataggio allegati XML/P7M su GCS non implementata (TODO nel codice) | ⏳ Aperto |
| 🟡 Media | `DeleteMessage` non esegue `EXPUNGE` — le email non vengono rimosse fisicamente | ⏳ Aperto |
| 🟡 Media | Discovery doc in formato `gin.H` custom invece di `nexus.ServiceDefinition` | ⏳ Da allineare |
| 🟡 Media | `registerToGateway()` custom con 5 retry invece di `nexus.StartGatewayHandshake()` | ⏳ Da allineare all'SDK |
| 🟡 Media | Ogni operazione apre e chiude una connessione TLS — costoso per operazioni batch | ⏳ Da valutare connection pooling |
| 🟢 Bassa | Nessun rate limiting sulle ricerche IMAP | ⏳ Aperto |
| 🟢 Bassa | Test di integrazione presenti ma richiedono server IMAP reale (non eseguibili in CI) | ⏳ Aperto |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
