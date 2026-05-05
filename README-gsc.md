# MCP GSC Remote - Google Search Console MCP Server (MOCA)

Server **MCP (Model Context Protocol) remoto** per **Google Search Console**, deployato su **Cloudflare Workers**. Espone gli endpoint dell'API GSC come tool richiamabili da un client MCP (es. Claude Desktop, Claude Code, qualsiasi client compatibile MCP via HTTP/JSON-RPC), gestisce l'OAuth2 di Google e include un mapping interno **cliente → proprietà GSC** per oltre 700 proprietà su ~400 clienti MOCA.

- **Worker name**: `mcp-gsc-remote-moca`
- **Entry point**: [src/index.js](src/index.js)
- **Runtime**: Cloudflare Workers (`nodejs_compat`)
- **Storage**: Cloudflare KV (`AUTH_STORE`) — token OAuth persistenti
- **Protocollo**: JSON-RPC 2.0 (MCP `2025-06-18`)
- **Versione server**: `1.0.0`

---

## Indice

- [Architettura](#architettura)
- [Endpoint HTTP](#endpoint-http)
- [Flusso di autenticazione](#flusso-di-autenticazione)
- [Tools MCP](#tools-mcp)
- [Prompts MCP](#prompts-mcp)
- [Mapping clienti → proprietà](#mapping-clienti--proprietà)
- [Setup e deployment](#setup-e-deployment)
- [Sviluppo locale](#sviluppo-locale)
- [Configurazione client MCP](#configurazione-client-mcp)
- [Note sui dati GSC](#note-sui-dati-gsc)
- [Struttura del progetto](#struttura-del-progetto)

---

## Architettura

```
                  ┌─────────────────────┐
                  │   Client MCP        │
                  │ (Claude / altro)    │
                  └──────────┬──────────┘
                             │  POST /mcp  (JSON-RPC 2.0)
                             ▼
              ┌───────────────────────────────┐
              │  Cloudflare Worker            │
              │  mcp-gsc-remote-moca          │
              │                               │
              │  • /          health check    │
              │  • /auth      OAuth start     │
              │  • /callback  OAuth callback  │
              │  • /auth/status               │
              │  • /mcp       MCP endpoint    │
              └───────┬───────────────┬───────┘
                      │               │
                      ▼               ▼
              ┌──────────────┐  ┌────────────────────┐
              │ KV AUTH_STORE│  │ Google APIs        │
              │ (tokens)     │  │ • OAuth2           │
              └──────────────┘  │ • Search Console v3│
                                │ • URL Inspection v1│
                                └────────────────────┘
```

Il Worker tiene una sola istanza condivisa di token OAuth nello slot `tokens` della KV: una sola autenticazione abilita tutti i client che chiamano `/mcp`.

---

## Endpoint HTTP

| Metodo | Path           | Descrizione                                                                 |
|--------|----------------|-----------------------------------------------------------------------------|
| `GET`  | `/`            | Health check + elenco endpoint (JSON).                                      |
| `GET`  | `/auth`        | Avvia il flusso OAuth2 Google (redirect a `accounts.google.com`).           |
| `GET`  | `/callback`    | Callback OAuth: scambia il `code` per `access_token` + `refresh_token` e li salva su KV. Risponde con una pagina HTML di conferma. |
| `GET`  | `/auth/status` | Indica se ci sono token salvati (`{ authenticated: bool }`).                |
| `POST` | `/mcp`         | Endpoint MCP JSON-RPC. Supporta `initialize`, `tools/list`, `tools/call`, `prompts/list`, `prompts/get`. |
| `OPTIONS` | *           | Preflight CORS (`*` come origin).                                           |

CORS è completamente aperto (`Access-Control-Allow-Origin: *`) e gli header `Authorization`, `X-Request-Id`, `Mcp-Session-Id` sono accettati ed esposti.

---

## Flusso di autenticazione

1. L'admin apre `https://<worker>.workers.dev/auth` nel browser.
2. Google chiede consenso per lo scope `https://www.googleapis.com/auth/webmasters.readonly`.
3. Google reindirizza su `/callback?code=...`.
4. Il Worker scambia il `code` per `access_token` + `refresh_token` e li scrive in KV alla chiave `tokens` con il campo `expires_at` (timestamp ms).
5. Ad ogni chiamata API, [`getValidAccessToken`](src/index.js:1080) controlla la scadenza e, se necessario, esegue un refresh automatico via `refresh_token`, riscrivendo i token in KV.

> Tutti i token sono condivisi a livello di Worker: non c'è multi-tenancy, l'autenticazione è "globale" per l'istanza deployata.

---

## Tools MCP

Sono esposti **7 tools** via `tools/list`. Tutti rispondono con `content[].type = "text"`.

### 1. `get_properties`

Restituisce le proprietà GSC verificate (filtrate, esclude `siteUnverifiedUser`).

| Parametro | Tipo | Required | Default | Descrizione |
|-----------|------|----------|---------|-------------|
| —         | —    | —        | —       | Nessun parametro |

### 2. `resolve_client`

Cerca le proprietà GSC associate a un cliente per nome (match esatto, parziale, o multipli) usando il mapping CSV embedded. Da chiamare **prima** di altri tool quando l'utente specifica un nome cliente invece di un URL.

| Parametro       | Tipo    | Required | Default | Descrizione |
|-----------------|---------|----------|---------|-------------|
| `clientName`    | string  | ✅       | —       | Nome cliente (es: `mapei`, `aism`, `smeg`). Case-insensitive. |
| `autoSelectMain`| boolean | ❌       | `false` | Se `true` e ci sono più proprietà, seleziona automaticamente quella principale. |

**Tipi di match**:
- `exact` — nome cliente esatto.
- `partial` — un solo cliente contiene/è contenuto nel nome cercato.
- `multiple` — più clienti compatibili → restituisce suggerimenti.
- `not_found` — nessuna corrispondenza.

La selezione della "proprietà principale" segue la priorità: `sc-domain:` > `https://www.` > `https://` > `http://`, con preferenza per gli URL più corti (root domain).

### 3. `list_clients`

Mostra l'elenco dei clienti configurati nel mapping interno.

| Parametro   | Tipo    | Required | Default | Descrizione |
|-------------|---------|----------|---------|-------------|
| `filter`    | string  | ❌       | —       | Filtro per ricerca parziale sul nome cliente. |
| `showCount` | boolean | ❌       | `false` | Se `true`, mostra solo `nome (N proprietà)` senza dettaglio URL. |

Output limitato a 50 clienti per chiamata: usa `filter` per cercare casi specifici.

### 4. `get_property_stats`

**Totali aggregati REALI** (clic, impressioni, CTR, posizione media) per la proprietà nel periodo. Corrispondono ai numeri visibili in Search Console (non sono la somma delle query).

| Parametro          | Tipo    | Required | Default | Descrizione |
|--------------------|---------|----------|---------|-------------|
| `siteUrl`          | string  | ✅       | —       | URL proprietà (es: `https://example.com/` o `sc-domain:example.com`). |
| `startDate`        | string  | ✅       | —       | `YYYY-MM-DD`. |
| `endDate`          | string  | ✅       | —       | `YYYY-MM-DD`. |
| `includeFreshData` | boolean | ❌       | `true`  | Se `true` → `dataState: "all"` (include fresh data); altrimenti `"final"`. |

Quando `includeFreshData=true` il response segnala `first_incomplete_date` (data da cui i dati sono ancora soggetti a variazioni).

### 5. `get_search_queries`

Top query con dimensione `query` e metriche per riga.

| Parametro          | Tipo    | Required | Default | Descrizione |
|--------------------|---------|----------|---------|-------------|
| `siteUrl`          | string  | ✅       | —       | URL proprietà. |
| `startDate`        | string  | ✅       | —       | `YYYY-MM-DD`. |
| `endDate`          | string  | ✅       | —       | `YYYY-MM-DD`. |
| `limit`            | number  | ❌       | `100`   | `rowLimit` GSC (max consentito 25.000 lato API; il tool non lo cappa). |
| `includeFreshData` | boolean | ❌       | `true`  | Idem `get_property_stats`. |

⚠️ I totali sommati delle query **NON** corrispondono ai totali reali a causa delle query anonimizzate da Google: per i totali reali usare `get_property_stats`.

### 6. `get_top_pages`

Top pagine con dimensione `page`.

| Parametro          | Tipo    | Required | Default | Descrizione |
|--------------------|---------|----------|---------|-------------|
| `siteUrl`          | string  | ✅       | —       | URL proprietà. |
| `startDate`        | string  | ✅       | —       | `YYYY-MM-DD`. |
| `endDate`          | string  | ✅       | —       | `YYYY-MM-DD`. |
| `limit`            | number  | ❌       | `50`    | Numero max pagine. |
| `includeFreshData` | boolean | ❌       | `true`  | Idem sopra. |

### 7. `inspect_url`

Wrapper su `urlInspection/index:inspect` (URL Inspection API v1). Stato di indicizzazione, copertura, ultimo crawl, fetch, robots.txt.

| Parametro    | Tipo   | Required | Default | Descrizione |
|--------------|--------|----------|---------|-------------|
| `siteUrl`    | string | ✅       | —       | URL della proprietà GSC. |
| `inspectUrl` | string | ✅       | —       | URL specifico da ispezionare (deve appartenere alla proprietà). |

Output: `coverageState`, `indexingState`, `lastCrawlTime`, e warning su `pageFetchState` / `robotsTxtState`.

---

## Prompts MCP

Sono esposti **8 prompt** via `prompts/list`, generati dinamicamente con date relative al momento della chiamata. Tornano un `messages[]` con il prompt da iniettare nel client.

| Nome prompt                     | Argomenti (✅ = required)                                                                        | Cosa fa |
|---------------------------------|--------------------------------------------------------------------------------------------------|---------|
| `analizza_traffico`             | `siteUrl`✅, `periodo1_inizio`✅, `periodo1_fine`✅, `periodo2_inizio`, `periodo2_fine`           | Confronto traffico organico tra due periodi (con auto-fallback al mese precedente se il periodo 2 è omesso). |
| `trova_cannibalizzazioni`       | `siteUrl`✅, `giorni` (default 30), `soglia_minima` (default 100)                                | Identifica query con più pagine in competizione. |
| `analisi_pagine_top`            | `siteUrl`✅, `numero_pagine` (default 20), `periodo_giorni` (default 30)                         | Analisi performance delle top pagine con quota sul totale sito. |
| `brand_vs_non_brand`            | `siteUrl`✅, `brand_terms`✅ (CSV), `giorni` (default 30)                                        | Suddivisione brand vs non-brand con metriche comparative. |
| `opportunita_featured_snippets` | `siteUrl`✅, `giorni` (default 30)                                                               | Trova query in pos. 1–5 candidate a featured snippet. |
| `analisi_ctr_anomalo`           | `siteUrl`✅, `giorni` (default 30)                                                               | CTR vs benchmark per posizione, identifica anomalie. |
| `query_in_crescita`             | `siteUrl`✅, `giorni_recenti` (default 7), `giorni_precedenti` (default 7)                       | Query in crescita confrontando due finestre temporali contigue. |
| `analisi_stagionalita`          | `siteUrl`✅, `data_inizio`✅, `data_fine`✅                                                       | Confronto YoY (stesso periodo dell'anno precedente). |

Tutti i prompt istruiscono il modello a usare `get_property_stats` per i totali reali e a NON sommare le righe di `get_search_queries` / `get_top_pages`.

---

## Mapping clienti → proprietà

Il Worker include direttamente nel sorgente ([src/index.js:4](src/index.js:4)) un CSV embedded `ASSOCIAZIONI_CSV` con ~787 righe che mappano `Proprietà,Nome` (lo stesso contenuto del file [associazioni-gsc.csv](associazioni-gsc.csv) versionato a fianco). Il file CSV non è letto a runtime: la fonte di verità è il template literal in `index.js`.

Funzioni interne:
- [`parseAssociazioni()`](src/index.js:794) — costruisce la `Map<nome, proprietà[]>`.
- [`cercaProprietaPerCliente(nome)`](src/index.js:822) — risolve un nome con match esatto, parziale, o multiplo.
- [`getProprietaPrincipale(proprieta[])`](src/index.js:868) — euristica per scegliere la proprietà "canonica" tra più varianti dello stesso cliente.

> **Per aggiornare il mapping**: modificare il template literal in `src/index.js` (sostituire l'intero blocco `ASSOCIAZIONI_CSV`) e ridepoyare. Il CSV esterno `associazioni-gsc.csv` serve solo come copia editabile/comoda.

---

## Setup e deployment

### Prerequisiti

- Account Cloudflare con accesso a Workers.
- `wrangler` ≥ 4 (installato come devDependency).
- Credenziali OAuth2 di Google Cloud per Search Console:
  - Crea un progetto su [console.cloud.google.com](https://console.cloud.google.com).
  - Abilita **Google Search Console API**.
  - Crea una credenziale **OAuth Client ID** di tipo *Web application*.
  - Aggiungi l'URI `https://<worker>.workers.dev/callback` come *Authorized redirect URI*.

### Installazione

```bash
npm install
```

### Configurazione KV

Il `wrangler.toml` punta già a un namespace KV con id `a28a4f74a9314d6a878132922faf0b2e` (binding `AUTH_STORE`). Per un nuovo deploy creane uno tuo:

```bash
npx wrangler kv namespace create AUTH_STORE
# copia l'id risultante in wrangler.toml
```

### Secrets

Imposta i secret OAuth (NON committarli):

```bash
npx wrangler secret put GSC_CLIENT_ID
npx wrangler secret put GSC_CLIENT_SECRET
# opzionale, usato solo da auth-manager.js (legacy):
npx wrangler secret put GSC_REDIRECT_URI
```

> Il flusso OAuth in `index.js` calcola autonomamente il `redirect_uri` come `${url.origin}/callback`, quindi `GSC_REDIRECT_URI` non è strettamente richiesto dal Worker principale.

### Deploy

```bash
npm run deploy
```

### Prima autenticazione

1. Apri `https://mcp-gsc-remote-moca.<account>.workers.dev/auth` nel browser.
2. Completa il consenso Google (deve essere un account con accesso alle proprietà GSC).
3. Verifica con `GET /auth/status` che la risposta sia `{ "authenticated": true }`.

---

## Sviluppo locale

```bash
npm run dev      # wrangler dev — esegue il Worker su localhost
npm run tail     # wrangler tail — log live in produzione
```

In dev mode, l'OAuth callback va su `http://localhost:8787/callback`: aggiungi questo URI ai redirect autorizzati su Google Cloud Console se vuoi testare il flusso completo in locale.

---

## Configurazione client MCP

### Claude Desktop / Claude Code (HTTP transport)

```json
{
  "mcpServers": {
    "gsc-moca": {
      "transport": {
        "type": "http",
        "url": "https://mcp-gsc-remote-moca.<account>.workers.dev/mcp"
      }
    }
  }
}
```

### Esempio chiamata cURL

```bash
curl -X POST https://mcp-gsc-remote-moca.<account>.workers.dev/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "get_property_stats",
      "arguments": {
        "siteUrl": "sc-domain:example.com",
        "startDate": "2026-04-01",
        "endDate": "2026-04-30"
      }
    }
  }'
```

---

## Note sui dati GSC

- **`dataState: "all"` (fresh data)**: include i dati degli ultimi giorni ancora in elaborazione. Il response include `metadata.first_incomplete_date` per indicare la soglia.
- **`dataState: "final"`**: solo dati consolidati, esclude le ultime ~48–72h.
- **Totali aggregati ≠ somma delle query**: a causa delle query anonimizzate per privacy, la somma delle righe di `get_search_queries` è sempre inferiore al totale reale di `get_property_stats`. Stessa cosa per `get_top_pages` rispetto al totale sito.
- **Permessi**: lo scope richiesto è `webmasters.readonly`. L'utente che autentica deve avere accesso (anche `siteRestrictedUser`) alle proprietà che si vogliono interrogare; le proprietà con `siteUnverifiedUser` vengono filtrate da `get_properties`.
- **Quota API**: GSC ha limiti di QPS / quota giornaliera per progetto Google Cloud — vedi la [documentazione ufficiale](https://developers.google.com/webmaster-tools/limits).

---

## Struttura del progetto

```
mcp-gsc-remoto/
├── src/
│   ├── index.js                  ← entry point Worker (TUTTA la logica MCP + OAuth + CSV)
│   ├── mcp-server.js             ← scaffolding MCP standalone (non usato in produzione)
│   ├── auth-manager.js           ← AuthManager basato su google-auth-library + Hono (legacy)
│   ├── search-console-manager.js ← (legacy)
│   └── search-console-api.js     ← (legacy)
├── associazioni-gsc.csv          ← copia editabile del mapping cliente → proprietà
├── wrangler.toml                 ← config Worker, KV binding, vars
├── package.json
└── README.md
```

> I file `mcp-server.js`, `auth-manager.js`, `search-console-manager.js`, `search-console-api.js` derivano da una prima iterazione del progetto e **non sono importati da `src/index.js`**: il Worker live usa esclusivamente `src/index.js`. Possono essere rimossi se non servono come riferimento.

---

## Tecnologie

- [Cloudflare Workers](https://developers.cloudflare.com/workers/) — runtime serverless edge.
- [Wrangler](https://developers.cloudflare.com/workers/wrangler/) — CLI di deploy.
- [Cloudflare KV](https://developers.cloudflare.com/kv/) — storage token OAuth.
- [Google Search Console API v3](https://developers.google.com/webmaster-tools/v1/api_reference_index) — `searchAnalytics/query`, `sites`.
- [Google URL Inspection API v1](https://developers.google.com/webmaster-tools/v1/urlInspection.index/inspect).
- [Model Context Protocol](https://modelcontextprotocol.io/) — protocollo di comunicazione (`@modelcontextprotocol/sdk` ^1.18.2 in dependencies, ma il Worker implementa JSON-RPC manualmente senza usare la SDK).

---

## Licenza

ISC (vedi `package.json`).
