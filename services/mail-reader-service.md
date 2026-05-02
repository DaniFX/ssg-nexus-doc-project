# 📧 SSG Mail Reader Service — Specifiche Tecniche

> **Repository:** [`ssg-mail-reader-service`](https://github.com/DaniFX/ssg-mail-reader-service)
> **Stack:** Go 1.21+, Gin, `go-imap` (emersion), TLS 1.2+
> **Ruolo:** Microservizio stateless per la lettura e gestione di caselle email via protocollo IMAP.
> **Caso d’uso primario:** Lettura di caselle **PEC** (Posta Elettronica Certificata) per l’identificazione e il recupero di **Fatture Elettroniche** (allegati `.xml` e `.p7m`).

---

## 1. Caratteristica Architetturale Chiave: Stateless IMAP

A differenza degli altri microservizi Nexus, il Mail Reader **non persiste credenziali** su Firestore. Le credenziali IMAP vengono passate **per ogni richiesta** tramite header HTTP dedicati (`X-Imap-Host`, `X-Imap-User`, `X-Imap-Pass`).

Questo design permette a un singolo servizio di gestire caselle email di **clienti diversi** senza configurazione preventiva, rendendolo ideale per integrarsi con le PEC aziendali di diverse ditte nel flusso ERP.

```
Client
  |
  | X-Imap-Host: imaps.pec-provider.it:993
  | X-Imap-User: azienda@pec.it
  | X-Imap-Pass: <password>
  v
+---------------------------------------+
|           SSG Gateway                 |
|  [1] FirebaseAuthMiddleware (JWT)     |
|  [2] Proxy con OIDC Token             |
+---------------------------------------+
             |
             v
+---------------------------------------+
|      Mail Reader Service              |
|  [3] nexus.Guard() -> X-Nexus-User-ID |
|  [4] RequireImapHeaders() middleware  |
|      -> c.Set("imapCreds")            |
|  [5] Handler -> MailService           |
|  [6] client.DialTLS() -> IMAP server  |
+---------------------------------------+
             |
             v
    Server IMAP/PEC (TLS 1.2+)
```

---

## 2. Struttura del Progetto

```
ssg-mail-reader-service/
├── cmd/                         # Entrypoint (main.go)
├── internal/
│   ├── handlers/
│   │   ├── mail.go             # Handler HTTP per tutte le operazioni IMAP
│   │   ├── discovery.go        # Definizione contratto API (ServiceDefinition)
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

## 3. Autenticazione e Middleware Chain

La catena di middleware per le route private è composta da **due livelli** in serie:

### 3.1 `nexus.Guard()` — Identità Nexus

Primo middleware. Verifica che `X-Nexus-User-ID` sia presente (iniettato dal Gateway). Blocca con `401` se assente. Obbligatorio come da standard globali.

### 3.2 `RequireImapHeaders()` — Credenziali IMAP

**File:** `internal/middleware/imap_auth.go`

Secondo middleware. Verifica la presenza di tutti e tre gli header IMAP obbligatori. In caso di assenza risponde con `400 MISSING_IMAP_CREDENTIALS` e interrompe la catena.

```go
func RequireImapHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        host := c.GetHeader("X-Imap-Host")
        user := c.GetHeader("X-Imap-User")
        pass := c.GetHeader("X-Imap-Pass")

        if host == "" || user == "" || pass == "" {
            c.JSON(400, models.NewErrorResponse(
                "MISSING_IMAP_CREDENTIALS",
                "Fornire X-Imap-Host, X-Imap-User e X-Imap-Pass",
                nil,
            ))
            c.Abort()
            return
        }

        // Inietta nel contesto Gin per gli handler
        c.Set("imapCreds", ImapCredentials{Host: host, Username: user, Password: pass})
        c.Next()
    }
}
```

### 3.3 Header richiesti per ogni richiesta

| Header | Tipo | Esempio | Note |
|---|---|---|---|
| `X-Nexus-User-ID` | `string` | `uid_firebase_xyz` | Iniettato dal Gateway (obbligatorio) |
| `X-Imap-Host` | `string` | `imaps.pec.it:993` | Host + porta del server IMAP/PEC |
| `X-Imap-User` | `string` | `azienda@pec.it` | Username casella email |
| `X-Imap-Pass` | `string` | `<password>` | Password casella email |

> ⚠️ **Sicurezza**: Le credenziali IMAP viaggiano in chiaro negli header HTTP. La connessione tra client e Gateway **deve** essere HTTPS. In produzione valutare l’uso di token temporanei o vault secrets invece della password diretta.

---

## 4. Client IMAP — `MailService`

**File:** `internal/service/imap_client.go`
**Libreria:** [`github.com/emersion/go-imap`](https://github.com/emersion/go-imap)

### 4.1 Connessione TLS

Ogni operazione apre una nuova connessione TLS dedicata e la chiude con `defer c.Logout()`. Il servizio è **completamente stateless** a livello di connessioni.

```go
func (s *MailService) connect() (*client.Client, error) {
    // TLS minimo 1.2, ServerName estratto dall'host (senza porta)
    // per compatibilità con provider PEC (es. Aruba, Namirial, Legalmail)
    tlsConfig := &tls.Config{
        ServerName: hostOnly,
        MinVersion: tls.VersionTLS12,
    }
    c, err := client.DialTLS(s.Host, tlsConfig)
    // ...
    c.Login(s.Username, s.Password)
}
```

### 4.2 Operazioni disponibili

| Metodo | Protocollo IMAP | Descrizione |
|---|---|---|
| `ListFolders()` | `LIST "" *` | Recupera tutte le cartelle della casella |
| `Search(criteria)` | `UID SEARCH` + `UID FETCH ENVELOPE` | Ricerca per Subject/From/Body, ritorna preview |
| `GetMessage(folder, uid)` | `UID FETCH BODY[]` + `ENVELOPE` | Fetch completo: body text, HTML, allegati |
| `CreateFolder(name)` | `CREATE` | Crea nuova cartella IMAP |
| `MoveMessage(src, uid, dst)` | `UID MOVE` | Sposta email tra cartelle |
| `DeleteMessage(folder, uid)` | `UID STORE +FLAGS \\Deleted` | Marca per eliminazione (non rimuove fisicamente) |

> **Nota su `DeleteMessage`**: L’implementazione aggiunge il flag `\Deleted` IMAP standard. La rimozione fisica avviene solo dopo un comando `EXPUNGE`, non implementato nel servizio. Valutare se aggiungere `EXPUNGE` esplicito.

### 4.3 Identificazione Fatture Elettroniche

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

**⚠️ TODO aperto**: La logica di salvataggio su GCS e analisi XML SDI è predisposta ma non ancora implementata (commento nel codice). È il prossimo step naturale per integrare il flusso Fattura Elettronica con il Finance Service.

---

## 5. API Endpoints

Base path: `/api/v1/mail` (prefisso aggiunto automaticamente dal Gateway tramite `serviceName`)

Tutti gli endpoint richiedono `authRequired: true` (Nexus Guard + Firebase JWT).

### `GET /api/v1/mail/folders`

Recupera la lista di tutte le cartelle della casella IMAP.

**Risposta:**
```json
{
  "success": true,
  "data": {
    "folders": ["INBOX", "Sent", "Trash", "Fatture/2025", "Fatture/2026"]
  }
}
```

---

### `POST /api/v1/mail/folders`

Crea una nuova cartella IMAP.

**Body:**
```json
{ "name": "Fatture/2026" }
```

**Risposta:** `201 Created`
```json
{ "success": true, "data": { "message": "Cartella creata con successo" } }
```

---

### `POST /api/v1/mail/search`

Cerca email nel server IMAP in base a filtri combinabili.

**Body (`SearchCriteria`):**
```json
{
  "folder":       "INBOX",
  "subject":      "Fattura",
  "from":         "sdi@pec.fatturapa.it",
  "bodyContains": "partita IVA"
}
```

**Risposta (array di `EmailPreview`):**
```json
{
  "success": true,
  "data": {
    "emails": [
      { "uid": 1042, "subject": "Fattura n. 123/2026", "from": "sdi@pec.fatturapa.it", "date": "2026-04-15T10:30:00Z" }
    ]
  }
}
```

> I campi `subject`, `from` e `bodyContains` sono opzionali. La ricerca viene eseguita via `UID SEARCH` su tutti i campi forniti (AND logico).

---

### `GET /api/v1/mail/messages/:uid`

Recupera il contenuto completo di una singola email.

**Query param:** `?folder=INBOX` (default `INBOX`)

**Risposta (`EmailDetail`):**
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

---

### `PUT /api/v1/mail/messages/:uid/move`

Sposta un’email in una cartella di destinazione.

**Query param:** `?folder=INBOX` (cartella sorgente)

**Body:**
```json
{ "destinationFolder": "Fatture/2026" }
```

---

### `DELETE /api/v1/mail/messages/:uid`

Marca un’email con il flag `\Deleted`.

**Query param:** `?folder=INBOX`

---

## 6. Modelli di Dati

**File:** `internal/models/mail.go`

```go
// SearchCriteria — payload per POST /search
type SearchCriteria struct {
    Folder       string `json:"folder"`        // default "INBOX"
    Subject      string `json:"subject,omitempty"`
    From         string `json:"from,omitempty"`
    BodyContains string `json:"bodyContains,omitempty"`
}

// EmailPreview — usato nelle liste (solo metadati, senza body)
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

## 7. Service Discovery

**File:** `internal/handlers/discovery.go`

Il servizio si registra al Gateway come `mail-reader-service` con `version: 1.0.0`. Tutti e 6 gli endpoint dichiarano `authRequired: true`.

Endpoint esposto per ispezione: `GET /_discover`

---

## 8. Variabili d’Ambiente

> Riferimento: `.env.example` nel repo `ssg-mail-reader-service`

| Variabile | Descrizione | Esempio |
|---|---|---|
| `GATEWAY_URL` | URL del Gateway per la registrazione push | `https://ssg-gateway-xyz.run.app` |
| `SERVICE_URL` | URL Cloud Run di questo servizio | `https://mail-reader-xyz.run.app` |
| `INTERNAL_SECRET` | Segreto condiviso per l’handshake con il Gateway | `<valore da Secret Manager>` |
| `PORT` | Porta HTTP | `8080` |

---

## 9. Dipendenze Principali

| Package | Versione | Scopo |
|---|---|---|
| `github.com/emersion/go-imap` | v1.x | Client IMAP RFC 3501 |
| `github.com/emersion/go-message` | v0.x | Parser MIME multipart (body + allegati) |
| `github.com/gin-gonic/gin` | v1.9+ | HTTP framework |
| `github.com/DaniFX/ssg-nexus-sdk` | latest | Guard, Discovery, Response standard |

---

## 10. Issue Noti e TODO

| Priorità | Issue | Stato |
|---|---|---|
| 🔴 Alta | Credenziali IMAP in chiaro negli header HTTP — nessun layer di cifratura aggiuntivo | ⚠️ Design by choice, documentare rischio |
| 🔴 Alta | Logica salvataggio allegati XML/P7M su GCS non implementata (TODO nel codice) | ⏳ Aperto |
| 🟡 Media | `DeleteMessage` non esegue `EXPUNGE` — le email non vengono rimosse fisicamente | ⏳ Aperto |
| 🟡 Media | Ogni operazione apre e chiude una connessione TLS — costoso per operazioni batch | ⏳ Da valutare connection pooling |
| 🟢 Bassa | Nessun rate limiting sulle ricerche IMAP (query lente su caselle grandi) | ⏳ Aperto |
| 🟢 Bassa | Test di integrazione presenti ma disabilitati in CI (richiedono server IMAP reale) | ⏳ Aperto |

---

*Parte del progetto SSG Nexus — vedere [README.md](../README.md) per la panoramica dei repository.*
