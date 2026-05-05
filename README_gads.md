# Documentazione Google Ads MCP Server (Remote)

Questo progetto è un server **Model Context Protocol (MCP)** remoto progettato per interfacciarsi con le API di Google Ads (v21). L'infrastruttura si basa su **Cloudflare Workers** ed espone vari *tools* pronti all'uso per i modelli AI. Consente l'analisi delle campagne, le performance degli account, recuperare metriche e perfino scrivere query arbitrarie in GAQL (Google Ads Query Language).

## 🏢 Architettura e Funzionamento

1. **Piattaforma**: Cloudflare Workers (implementazione Node.js TypeScript esposta tramite il file `src/index.ts`). Il routing gestisce sia il protocollo MCP, sia degli endpoint HTTP specifici per l'autenticazione.
2. **Archiviazione dello Stato**: Utilizza **Cloudflare KV** (`OAUTH_TOKENS`) per persistere token OAuth, refresh token e credenziali, garantendo un'operatività continua senza dover richiedere autorizzazioni multiple.
3. **Database Clienti**: All'interno del file `index.ts` è presente un array statico `CUSTOMER_DATABASE` precompilato con decine di clienti (es. *Smeg Italia*, *Mapei*, *Moca Interactive*, ecc.) e i rispettivi *Customer ID*. Questo permette all'AI di risolvere il nome di un cliente in un ID account automaticamente.

## 🔐 Configurazione e Autenticazione (OAuth)

Il server MCP possiede una pagina web integrata per la gestione completa dei token OAuth tramite Google:
*   `GET /auth`: Endpoint per inizializzare il flusso OAuth (richiede `client_id` e `client_secret` di Google Cloud).
*   `GET /auth/callback`: Callback per ricevere il token code, che viene poi scambiato per *access_token* e *refresh_token*.
*   `GET /auth/status`: Verifica in tempo reale la presenza e la validità dei token (utile per debug visibile dal browser).
*   `GET /auth/refresh`: Forza il refresh dell'access token a partire dal refresh token archiviato in KV.

Tutti gli endpoint di autenticazione possono essere protetti mediante un `AUTH_SECRET` configurabile come variabile d'ambiente sul Worker (`env.AUTH_SECRET`). Le chiavi `GOOGLE_ADS_CREDENTIALS` e `GOOGLE_ADS_DEVELOPER_TOKEN` sono anch'esse passate come variabili sicure (secrets) al Worker CLI di Cloudflare (Wrangler).

## 🛠 Strumenti MCP Disponibili (Tools)

Il server espone i seguenti *tool* che l'assistente AI può richiamare. Ciascun tool risponde per via indiretta usando formattazioni compatte e ottimizzate per risparmiare risorse e limitare il contesto del prompt.

### 1. `resolve_customer`
*   **A cosa serve**: Cerca un account Google Ads per *nome* all'interno del database integrato.
*   **Input**: `name` (stringa - Es. "Randstad" o "Smeg").
*   **Funzionamento**: Va utilizzato per *primo* se l'utente menziona un brand per ottenere il Customer ID a 10 cifre corrispondente, che sarà richiesto da tutti gli altri strumenti.

### 2. `list_customers`
*   **A cosa serve**: Mostra l'intera lista dei clienti presenti nel database integrato (`CUSTOMER_DATABASE`) col rispettivo ID account. Non richiede parametri di input.

### 3. `list_accounts`
*   **A cosa serve**: Mostra tutti i customer ID accessibili direttamente o tramite il Manager Account (MCC) autorizzato dai token di Google.

### 4. `get_account_currency`
*   **A cosa serve**: Restituisce il codice valuta *default* di uno specifico account.
*   **Input**: `customer_id` (ID a 10 cifre dell'account).

### 5. `get_campaign_performance`
*   **A cosa serve**: Recupera le metriche di performance delle campagne per uno specifico periodo.
*   **Input**: `customer_id` (stringa), `days` (intero, default: 30).
*   **Comportamento speciale**: Per efficienza, processa **solo** campagne con stato `ENABLED` ed erogazione negli ultimi giorni specificati (`impressions > 0`), ordinate per maggiore spesa (`cost_micros DESC`), con un limite di 20 record. Include impression, click, CTR, spesa e conversioni.

### 6. `get_ad_performance`
*   **A cosa serve**: Ottiene le performance per i singoli **Annunci** (Ads).
*   **Input**: `customer_id` (stringa), `days` (intero, default: 30).
*   **Comportamento speciale**: Cerca ad attivi (`ENABLED`), in gruppi di annunci `ENABLED`, in campagne `ENABLED` limitato ai primi 15 ordinati per maggiori impression.

### 7. `get_ad_creatives`
*   **A cosa serve**: Recupera i dettagli testuali veri e propri degli *Responsive Search Ads* (RSA), come titoli (headlines), descrizioni e i relativi link (Final URLs).
*   **Input**: `customer_id` (stringa).
*   **Comportamento speciale**: Filtra solo messaggi `RESPONSIVE_SEARCH_AD` attivi, massimo 10 record, per comprendere il tipo di copy utilizzato lato web.

### 8. `run_gaql`
*   **A cosa serve**: Permette all'AI di formulare ed eseguire query complesse su misura nel linguaggio ufficiale di Google Ads (GAQL), garantendo l'estrazione di metriche avanzate o report personalizzati non coperti dai tool standard.
*   **Input**: `customer_id` (stringa), `query` (stringa GAQL valida), `format` ("table", "json" o "csv" - il default è "table" compatto).
*   **Comportamento speciale**: Nel formato testuale/tabella limita drasticamente l'output a 30 righe per preservare token (se superato segnala "...more rows").
