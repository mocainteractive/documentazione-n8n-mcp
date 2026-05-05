# Meta Ads MCP Server

Server MCP (Model Context Protocol) per interfacciare Claude con Meta Ads API (Facebook/Instagram).

**Ottimizzato per:**
- Ridurre il volume di dati ed evitare blocchi per eccesso di caratteri
- Fornire analisi e suggerimenti automatici sulle performance
- Estrarre URL scaricabili per immagini e video delle creatività

---

## Indice

- [Quick Start](#-quick-start)
- [Tool Disponibili](#-tool-disponibili)
- [Esempi di Utilizzo](#-esempi-di-utilizzo)
- [Metriche e KPI](#-metriche-e-kpi)
- [Deploy e Configurazione](#-deploy-e-configurazione)
- [Troubleshooting](#-troubleshooting)

---

## Setup Rapido

### 1. Deploy su Cloudflare Workers

```bash
# Login
wrangler login

# Configura secret
wrangler secret put META_ACCESS_TOKEN
# Incolla il tuo token

# Deploy
npm run deploy
```

### 2. Configura Claude Desktop

**File:** `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "meta-ads": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://meta-mcp-server.YOUR-SUBDOMAIN.workers.dev/mcp",
        "--transport",
        "http-only"
      ]
    }
  }
}
```

### 3. Riavvia Claude Desktop e Testa

```
"Fammi una panoramica dell'account act_123456789"
```

---

## Tool Disponibili

### Tool Principali (Inizia da Qui)

| Tool | Descrizione | Quando Usarlo |
|------|-------------|---------------|
| `get_account_overview` | Panoramica KPI + ranking campagne + suggerimenti | **Prima cosa da chiamare** per capire l'andamento |
| `get_targeting_analysis` | Analisi target per adset con performance | Per capire quali audience performano meglio |
| `get_reach_breakdown` | Copertura per età, genere, piattaforma, placement | Per capire dove viene distribuito il budget |

### Tool Lista (Per Drill-Down)

| Tool | Descrizione | Quando Usarlo |
|------|-------------|---------------|
| `list_ad_accounts` | Lista account accessibili | Per scegliere su quale account lavorare |
| `list_active_campaigns` | Lista campagne (nome, obiettivo, status) | Per ottenere gli ID campagna |
| `list_adsets` | Lista ad sets base | Per ottenere gli ID adset |
| `list_ads` | Lista ads con ID creative | Per ottenere gli ID ad/creative |

### Tool Dettaglio (Per Elementi Singoli)

| Tool | Descrizione | Quando Usarlo |
|------|-------------|---------------|
| `get_ad_creative` | Creatività con **URL scaricabili** | Per vedere/scaricare immagini e video |
| `get_video_details` | Dettagli video (URL download, thumbnail) | Per scaricare un video specifico |
| `get_campaign_insights` | Dettagli singola campagna | Per analisi specifica |
| `get_account_insights` | Insights grezzi con breakdown custom | Per analisi personalizzate |
| `get_account_info` | Info base account | Per valuta, timezone, spesa storica |

---

## Dettaglio Tool

### `get_account_overview`

**Panoramica completa con analisi automatica.**

```
"Fammi vedere l'andamento dell'account act_123456789 negli ultimi 7 giorni"
```

**Parametri:**
- `ad_account_id`: ID account (act_123456). Opzionale se configurato default
- `date_preset`: `last_7d`, `last_30d`, `last_90d`, `this_month`, `last_month`
- `include_campaigns`: Includi ranking campagne (default: true)

**Restituisce:**
```json
{
  "account": {
    "id": "act_123456789",
    "nome": "Nome Account",
    "valuta": "EUR"
  },
  "analisi": {
    "periodo": "last_30d",
    "kpi_principali": {
      "spesa_totale": "1500.00",
      "impressions": 250000,
      "link_clicks": 3500,
      "reach": 80000,
      "ctr_link": "1.4",
      "cpc_link": "0.43",
      "cpm": "6.00",
      "frequency": "3.1"
    },
    "conversioni": {
      "acquisti": 45,
      "valore_acquisti": "4500.00",
      "costo_per_acquisto": "33.33",
      "roas": "3.0"
    }
  },
  "campagne_per_spesa": [
    {"nome": "Campagna TOF", "spesa": "800.00", "ctr_link": "1.2", "conversioni": "20"},
    {"nome": "Campagna Retargeting", "spesa": "700.00", "ctr_link": "2.1", "conversioni": "25"}
  ],
  "suggerimenti": [
    "Frequenza alta (>3): il pubblico potrebbe essere saturo, considera di espandere il target",
    "ROAS buono (3x): le campagne sono profittevoli"
  ]
}
```

---

### `get_targeting_analysis`

**Analisi targeting per adset con performance.**

```
"Analizza i target delle campagne dell'account act_123456789"
```

**Parametri:**
- `ad_account_id`: ID account
- `campaign_id`: Filtra per campagna specifica (opzionale)
- `date_preset`: Periodo

**Restituisce:**
```json
{
  "periodo": "last_30d",
  "totale_adset_analizzati": 5,
  "targeting_per_spesa": [
    {
      "adset_name": "Donne 25-44 Interessate Moda",
      "optimization_goal": "CONVERSIONS",
      "eta": "25-44",
      "genere": "Donne",
      "localita": "IT",
      "interessi": "Fashion, Shopping online, Luxury brands",
      "custom_audiences": 2,
      "ha_lookalike": true,
      "performance": {
        "spesa": "500.00",
        "link_clicks": "1200",
        "ctr_link": "1.8",
        "cpc_link": "0.42",
        "conversioni": "15"
      }
    }
  ],
  "insight_targeting": [
    "Miglior CTR Link: \"Donne 25-44 Interessate Moda\" con 1.8%",
    "3/5 adset usano custom audiences",
    "2/5 adset usano lookalike audiences"
  ]
}
```

---

### `get_reach_breakdown`

**Copertura per segmento demografico o piattaforma.**

```
"Come è distribuita la copertura per età?"
"Su quali piattaforme stiamo spendendo di più?"
```

**Parametri:**
- `ad_account_id`: ID account
- `breakdown_type`: `age`, `gender`, `platform`, `placement`
- `date_preset`: Periodo

**Restituisce:**
```json
{
  "breakdown_type": "age",
  "periodo": "last_30d",
  "segmenti": [
    {"segmento": "25-34", "reach": "25000", "impressions": "75000", "link_clicks": "1200", "spesa": "450.00", "ctr_link": "1.6"},
    {"segmento": "35-44", "reach": "18000", "impressions": "50000", "link_clicks": "800", "spesa": "320.00", "ctr_link": "1.6"}
  ],
  "totale_reach": 80000
}
```

---

### `get_ad_creative`

**Dettagli creatività con URL per download.**

```
"Mostrami la creatività dell'ad 123456789"
```

**Parametri:**
- `ad_id`: ID dell'ad (richiesto)

**Restituisce:**
```json
{
  "id": "123456789",
  "name": "Ad Nome",
  "creative": {
    "id": "987654321",
    "title": "Titolo Annuncio",
    "body": "Testo del copy...",
    "image_url": "https://scontent...",
    "link_url": "https://www.sito.com/landing",
    "call_to_action_type": "SHOP_NOW"
  },
  "video_details": {
    "video_url": "https://video.xx.fbcdn.net/...",
    "thumbnail_url": "https://scontent...",
    "duration_seconds": 30,
    "title": "Nome Video"
  }
}
```

> **Nota:** `video_url` è l'URL diretto per scaricare il video. `image_url` è l'URL dell'immagine.

---

### `get_video_details`

**URL download per un video specifico.**

```
"Dammi l'URL per scaricare il video 123456789"
```

**Parametri:**
- `video_id`: ID del video Meta

**Restituisce:**
```json
{
  "video_id": "123456789",
  "download_url": "https://video.xx.fbcdn.net/v/...",
  "thumbnail_url": "https://scontent...",
  "duration_seconds": 30,
  "title": "Nome Video",
  "description": "Descrizione...",
  "created_time": "2024-01-15T10:30:00+0000"
}
```

---

### `list_ad_accounts`

**Lista account accessibili dal token.**

```
"Quali account pubblicitari posso gestire?"
```

**Parametri:**
- `limit`: Max account (default: 25)

**Restituisce:**
```json
[
  {"id": "act_123456789", "account_id": "123456789", "name": "Account Ecommerce", "account_status": 1, "currency": "EUR"},
  {"id": "act_987654321", "account_id": "987654321", "name": "Account Lead Gen", "account_status": 1, "currency": "EUR"}
]
```

---

### `get_account_insights`

**Insights con breakdown personalizzati.**

```
"Dammi gli insights per età e genere"
```

**Parametri:**
- `ad_account_id`: ID account
- `date_preset`: Periodo
- `time_range`: Range custom `{since: "2024-01-01", until: "2024-01-31"}`
- `level`: `account`, `campaign`, `adset`, `ad`
- `breakdowns`: Array es. `["age", "gender"]`, `["publisher_platform"]`
- `limit`: Max risultati (default: 30)

---

## Esempi di Utilizzo

### Panoramica Account
```
"Come sta andando l'account act_123456789?"
→ Usa get_account_overview

"Confronta le performance degli ultimi 7 giorni vs ultimi 30 giorni"
→ Usa get_account_overview con date_preset diversi
```

### Analisi Target
```
"Quali target stanno performando meglio?"
→ Usa get_targeting_analysis

"Analizza i target della campagna 123456"
→ Usa get_targeting_analysis con campaign_id
```

### Analisi Copertura
```
"Come si distribuisce il budget per fascia d'età?"
→ Usa get_reach_breakdown con breakdown_type: "age"

"Su quali piattaforme stiamo spendendo?"
→ Usa get_reach_breakdown con breakdown_type: "platform"
```

### Creatività
```
"Mostrami le creatività della campagna X"
→ Prima list_ads con campaign_id, poi get_ad_creative per ogni ad

"Voglio scaricare i video delle inserzioni attive"
→ Prima list_ads, poi get_ad_creative, usa i video_url restituiti
```

### Multi-Account
```
"Lista tutti i miei account"
→ Usa list_ad_accounts

"Panoramica per ogni account"
→ Usa get_account_overview per ciascun account
```

---

## Metriche e KPI

### Metriche Link Clicks (Default)

| Metrica | Descrizione |
|---------|-------------|
| `link_clicks` | Click sul link (più accurato per advertising) |
| `ctr_link` | Click-Through Rate sui link (%) |
| `cpc_link` | Cost Per Link Click |

### Metriche Base

| Metrica | Descrizione |
|---------|-------------|
| `impressions` | Impression totali |
| `reach` | Utenti unici raggiunti |
| `frequency` | Media impression per utente |
| `spend` | Spesa totale |
| `cpm` | Cost Per Mille impression |

### Metriche Conversioni

| Metrica | Descrizione |
|---------|-------------|
| `acquisti` | Numero conversioni purchase |
| `valore_acquisti` | Valore totale acquisti |
| `costo_per_acquisto` | CPA (Cost Per Acquisition) |
| `roas` | Return On Ad Spend |

### Breakdown Disponibili

| Tipo | Valori |
|------|--------|
| `age` | 18-24, 25-34, 35-44, 45-54, 55-64, 65+ |
| `gender` | male, female, unknown |
| `platform` | facebook, instagram, messenger, audience_network |
| `placement` | feed, stories, reels, right_column, etc. |

---

## Deploy e Configurazione

### Requisiti

- Node.js >= 18
- Account Cloudflare (gratuito)
- Meta Access Token con permessi: `ads_read`, `read_insights`, `business_management`

### Token Consigliato

**System User Token** (Business Manager):
- Non scade mai
- Perfetto per produzione
- Business Manager → Impostazioni → Utenti di sistema → Genera token

### Variabili Ambiente

| Variabile | Obbligatoria | Descrizione |
|-----------|--------------|-------------|
| `META_ACCESS_TOKEN` | Sì | Token Meta con permessi ads |
| `AD_ACCOUNT_ID` | No | Account default (formato: act_123456) |

### Deploy

```bash
# 1. Install
npm install

# 2. Configura secrets
wrangler secret put META_ACCESS_TOKEN

# 3. Deploy
npm run deploy
```

### Sviluppo Locale

```bash
# Crea .dev.vars
cp .dev.vars.example .dev.vars
# Aggiungi META_ACCESS_TOKEN

# Run
npm run dev
# Server su http://localhost:8787
```

---

## Limiti e Ottimizzazioni

### Limiti Default (Ottimizzati)

| Tool | Limite | Note |
|------|--------|------|
| `list_ad_accounts` | 25 | Ridotto da 500 |
| `list_active_campaigns` | 25 | Ridotto da 100 |
| `list_adsets` | 20 | Ridotto da 100 |
| `list_ads` | 15 | Ridotto da 100 |
| `get_account_insights` | 30 | Ridotto da 100 |

### Troncamento Automatico

Se una risposta supera 45.000 caratteri, viene automaticamente troncata con messaggio:
```json
{
  "_truncated": true,
  "_message": "Risposta troncata: mostrati 10/50 elementi. Usa filtri più specifici."
}
```

**Soluzione:** Usa `campaign_id`, `adset_id` o `limit` inferiore.

---

## Troubleshooting

### "Invalid OAuth access token"
- Verifica token su [Graph API Explorer](https://developers.facebook.com/tools/explorer/)
- Controlla permessi: `ads_read`, `read_insights`, `business_management`
- Se System User: verifica accesso all'account in Business Manager

### "Rate limit exceeded"
- Attendi qualche secondo
- Riduci frequenza chiamate
- Usa `date_preset` invece di `time_range`

### "Risposta troncata"
- Usa filtri più specifici (`campaign_id`, `adset_id`)
- Riduci il `limit`
- Usa tool più mirati (es. `get_account_overview` invece di `get_account_insights`)

### Token Scaduto (User Access Token)
- I User Token scadono dopo 60 giorni
- Rigenera su Graph API Explorer
- **Consigliato:** usa System User Token (non scade)

---

## Risorse

- [Meta Marketing API Docs](https://developers.facebook.com/docs/marketing-api)
- [Graph API Explorer](https://developers.facebook.com/tools/explorer/)
- [MCP Protocol Spec](https://modelcontextprotocol.io)
- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)

---

## License

MIT
