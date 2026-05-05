# Google Analytics 4 MCP Server v2

Server MCP (Model Context Protocol) per Google Analytics 4 che consente a Claude Desktop di interrogare dati analytics tramite un'interfaccia unificata.

## Panoramica

Questo server MCP permette di:
- Interrogare dati GA4 da Claude Desktop usando linguaggio naturale
- Gestire molteplici proprietà GA4 con risoluzione automatica nome-cliente → ID
- Ottenere metriche in tempo reale e storiche
- Creare report personalizzati

## Architettura

```
Claude Desktop
    │
    ▼
MCP Client (JSON-RPC)
    │
    ▼
┌─────────────────────────────────────────┐
│  Cloudflare Workers                     │
│  ├── Hono (Web Framework)               │
│  ├── AuthManager (OAuth2 Google)        │
│  ├── GA4Manager (Google Analytics API)  │
│  └── PropertyMapper (Risoluzione nomi)  │
└─────────────────────────────────────────┘
    │
    ▼
Google Analytics Data API
```

## Prerequisiti

1. **Node.js** v18 o superiore
2. **Account Cloudflare** con Workers abilitato
3. **Progetto Google Cloud Console** con:
   - Google Analytics Data API abilitata
   - Credenziali OAuth 2.0 configurate

## Installazione

### 1. Clonare e installare dipendenze

```bash
cd mcp-ga4-remoto-v2
npm install
```

### 2. Configurare le credenziali Google

Creare un progetto su [Google Cloud Console](https://console.cloud.google.com/):

1. Vai su **API & Services** → **Library**
2. Cerca e abilita **Google Analytics Data API**
3. Vai su **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**
4. Tipo: **Web Application**
5. Redirect URI autorizzato: `https://tuo-worker.workers.dev/callback`
6. Copia **Client ID** e **Client Secret**

### 3. Configurare Wrangler

Creare il namespace KV per i token:

```bash
npx wrangler kv:namespace create GA4_TOKENS
```

Annotare l'ID restituito e aggiornare `wrangler.toml`:

```toml
[[kv_namespaces]]
binding = "GA4_TOKENS"
id = "IL_TUO_ID_QUI"
preview_id = "IL_TUO_ID_QUI"
```

### 4. Configurare i secrets

```bash
npx wrangler secret put GA4_CLIENT_ID
# Inserire il Client ID

npx wrangler secret put GA4_CLIENT_SECRET
# Inserire il Client Secret

npx wrangler secret put GA4_REDIRECT_URI
# Inserire: https://tuo-worker.workers.dev/callback

npx wrangler secret put TOKEN_ENCRYPTION_KEY
# Inserire una chiave di almeno 32 caratteri per cifrare i token
# Esempio: openssl rand -base64 32
```

### 5. Deploy

```bash
npm run deploy
```

## Uso

### Autenticazione OAuth

1. Visita `https://tuo-worker.workers.dev/auth`
2. Accedi con il tuo account Google
3. Concedi i permessi richiesti
4. Il token viene salvato automaticamente

### Configurazione Claude Desktop

Aggiungi al file `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "ga4": {
      "command": "curl",
      "args": ["-X", "POST", "https://tuo-worker.workers.dev/mcp"]
    }
  }
}
```

**Nota**: Per la configurazione effettiva di Claude Desktop con server MCP remoti, consulta la documentazione ufficiale di Anthropic.

### Comandi disponibili (Tools MCP)

| Tool | Descrizione |
|------|-------------|
| `resolve_property` | Risolve nome cliente in ID GA4 |
| `list_clients` | Elenca tutti i clienti disponibili |
| `get_basic_metrics` | Metriche base (utenti, sessioni, bounce rate) |
| `get_traffic_sources` | Sorgenti di traffico (source, medium, campagne) |
| `get_page_analytics` | Pagine più visitate |
| `get_realtime_data` | Dati in tempo reale |
| `get_custom_report` | Report personalizzati |

### Esempi di utilizzo con Claude

```
"Mostrami le metriche base di Venezianico degli ultimi 7 giorni"

"Quali sono le sorgenti di traffico principali per Eurospin questo mese?"

"Quanti utenti sono attivi adesso su Randstad?"

"Crea un report con sessioni e visualizzazioni pagina per paese"
```

## Endpoint HTTP

| Endpoint | Metodo | Descrizione |
|----------|--------|-------------|
| `/` | GET | Info sul server |
| `/auth` | GET | Avvia flusso OAuth |
| `/callback` | GET | Callback OAuth |
| `/mcp` | POST | Endpoint MCP principale |
| `/refresh` | POST | Refresh manuale token |

## Sviluppo locale

```bash
# Avvia server di sviluppo
npm run dev

# Visualizza logs in tempo reale
npm run tail
```

Il server sarà disponibile su `http://localhost:8787`

---

# Analisi di Sicurezza

## Avviso Importante

Questo progetto presenta diverse vulnerabilità di sicurezza che devono essere considerate prima dell'uso in produzione.

## Rischi generali dei server MCP

Come indicato nella domanda originale:

> - MCP lets AI agents control tools and access data on your machine
> - Claude for Desktop runs untrusted MCP servers with full user privileges and no sandbox
> - Most MCP plugins expect secrets via plaintext config files
> - Any Claude for Desktop install in your org is a potential exfiltration point

### Implicazioni per questo progetto

1. **Accesso dati senza sandbox**: Questo server può accedere a TUTTE le proprietà GA4 configurate (1300+) una volta autenticato
2. **Privilegi elevati**: Opera con credenziali OAuth complete senza restrizioni
3. **Potenziale esfiltrazione**: I dati analytics possono essere estratti attraverso le risposte MCP

---

## Vulnerabilità Identificate

### CRITICHE

#### 1. Token senza scadenza
**File**: `src/auth-manager.js` (righe 174-179)

```javascript
await this.env.GA4_TOKENS.put(
  `tokens_${state}`,
  JSON.stringify(tokenData)
  // NESSUN expirationTtl - il token persiste indefinitamente
);
```

**Rischio**: Se il KV storage viene compromesso, l'accesso a GA4 rimane valido per sempre.

**Mitigazione consigliata**:
```javascript
await this.env.GA4_TOKENS.put(
  `tokens_${state}`,
  JSON.stringify(tokenData),
  { expirationTtl: 86400 * 30 } // 30 giorni
);
```

#### 2. Logging di dati sensibili
**File**: `src/auth-manager.js` (righe 95-99, 374-378)

```javascript
console.log('Token status:', {
  has_refresh_token: !!tokens.refresh_token,
  expired: tokens.expiry_date ? Date.now() >= tokens.expiry_date : 'unknown',
  age_hours: ...
});
```

**Rischio**: Metadati dei token esposti nei log di Cloudflare Workers.

**Mitigazione**: Rimuovere o condizionare i log in produzione.

#### 3. CORS completamente aperto
**File**: `src/index.js` (riga 10)

```javascript
app.use('/*', cors());
```

**Rischio**: Qualsiasi sito web può fare richieste all'API.

**Mitigazione consigliata**:
```javascript
app.use('/*', cors({
  origin: ['https://claude.ai', 'https://tuodominio.com'],
  credentials: true
}));
```

### ALTE

#### 4. Endpoint di debug in produzione
~~**File**: `src/index.js` (righe 419-499)~~

~~Gli endpoint `/test-mapper`, `/debug-kv`, `/debug-tokens` sono accessibili senza autenticazione.~~

**RISOLTO**: Tutti gli endpoint di debug sono stati rimossi.

#### 5. Lista clienti pubblica
**File**: `src/index.js` (riga 426)

```javascript
app.get('/clients', (c) => {
  let clients = propertyMapper.getClientList();
  ...
});
```

**Rischio**: Chiunque può enumerare tutti i 1300+ clienti con i loro ID GA4.

#### 6. Token KV non cifrati
~~I token sono salvati in chiaro nel KV storage. Se un attaccante ottiene accesso al KV, ha accesso completo.~~

**RISOLTO**: I token vengono ora cifrati con AES-256-GCM prima di essere salvati nel KV storage. Vedi sezione "Cifratura Token".

### MEDIE

#### 7. Nessun rate limiting
Tutti gli endpoint sono privi di throttling.

**Rischio**: Enumerazione, DoS, abuso delle risorse.

#### 8. Messaggi di errore dettagliati
```javascript
message: `Errore nell'esecuzione del tool: ${toolError.message}`
```

**Rischio**: Stack trace e dettagli interni esposti al client.

#### 9. State parameter non validato
**File**: `src/auth-manager.js` (riga 14)

```javascript
const state = c.req.query('state') || 'default';
```

**Rischio**: Potenziale CSRF se lo state non viene validato correttamente.

---

## Dati sensibili esposti

### Endpoint accessibili senza autenticazione:

~~Gli endpoint di debug sono stati rimossi.~~

I tool MCP `list_clients` e `resolve_property` permettono ancora l'enumerazione dei clienti attraverso l'endpoint `/mcp`.

### File con dati sensibili:

- `mapping-proprietà-ga4.csv` - 1348 mappature cliente-ID
- `src/property-mapper.js` - Array hardcoded con tutti i clienti

---

## Raccomandazioni per la produzione

### Priorità 1 - Critiche

1. **Aggiungere scadenza ai token KV**
2. **Rimuovere console.log sensibili**
3. **Configurare CORS restrittivo**
4. **Rimuovere endpoint di debug**

### Priorità 2 - Alte

5. **Proteggere `/clients` con autenticazione**
6. **Cifrare token nel KV storage**
7. **Implementare rate limiting**

### Priorità 3 - Medie

8. **Validare state parameter OAuth**
9. **Sanitizzare messaggi di errore**
10. **Aggiungere logging di audit**

---

## Considerazioni MCP-specifiche

### Rischi intrinseci del protocollo MCP

1. **Nessun sandboxing**: Claude Desktop esegue server MCP con privilegi utente completi
2. **Trust implicito**: Una volta autenticato, il server ha accesso a tutti i dati
3. **Esfiltrazione dati**: Le risposte MCP possono contenere dati sensibili

### Per questo progetto specifico

- **Scope elevato**: Accesso a 1300+ proprietà GA4
- **Persistenza**: Token senza scadenza = accesso perpetuo
- **Dati business**: Metriche analytics di clienti enterprise

### Mitigazioni consigliate

1. ~~**Limitare scope OAuth** a `analytics.readonly`~~ **FATTO**
2. **Implementare audit log** di tutte le query
3. **Aggiungere approvazione per property** specifiche
4. **Considerare token a breve scadenza** con re-autenticazione

---

## Cifratura Token

I token OAuth vengono cifrati prima di essere salvati nel KV storage usando **AES-256-GCM**.

### Come funziona

1. **Derivazione chiave**: La chiave AES-256 viene derivata dalla `TOKEN_ENCRYPTION_KEY` usando PBKDF2 con 100.000 iterazioni
2. **Cifratura**: I token vengono cifrati con AES-GCM (authenticated encryption)
3. **Storage**: Nel KV viene salvato: `base64(salt + iv + ciphertext)`
4. **Migrazione automatica**: I token non cifrati (legacy) vengono automaticamente migrati al formato cifrato

### Configurazione

```bash
# Genera una chiave sicura
openssl rand -base64 32

# Configura come secret in Cloudflare
npx wrangler secret put TOKEN_ENCRYPTION_KEY
```

### Sicurezza

- **Salt unico** per ogni token (16 bytes)
- **IV casuale** per ogni operazione di cifratura (12 bytes)
- **PBKDF2** con SHA-256 e 100.000 iterazioni
- **AES-GCM** fornisce sia confidenzialità che integrità

### Nota importante

Se `TOKEN_ENCRYPTION_KEY` non è configurata, i token verranno salvati in chiaro (comportamento legacy) con un warning nei log.

---

## Changelog sicurezza

| Data | Versione | Note |
|------|----------|------|
| 2025-01 | 2.1.0 | Aggiunta cifratura AES-256-GCM per token, rimossi endpoint debug, scope ridotto a readonly |
| 2024 | 2.0.0 | Versione iniziale - vulnerabilità documentate |

---

## Licenza

ISC

## Supporto

Per problemi di sicurezza, contattare immediatamente il maintainer del progetto.
