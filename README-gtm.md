# MCP GTM Remote (MOCA)

Server MCP remoto per **Google Tag Manager API v2**, hostato su Cloudflare Workers.
Pattern identico a `mcp-ga4-remoto-v2` / `mcp-gsc-remoto`: un solo consent OAuth iniziale, refresh token cifrato in KV, accesso condiviso a tutti i container GTM della tua organizzazione.

## Cosa copre (~70 tool)

Coverage completa API v2:

- **Accounts**: list, get, update, user permissions (CRUD)
- **Containers**: list, get, create, update, delete, lookup, combine, move_tag_id, snippet
- **Workspaces**: list, get, create, update, delete, status, sync, resolve_conflict, quick_preview
- **Tags / Triggers / Variables / Folders / Templates / Clients / Transformations / Zones / GtagConfig**: CRUD completo + **revert**
- **Built-in Variables**: list, enable, disable, revert
- **Versions**: list_version_headers, latest, get, live, create, update, delete, publish, set_latest, undelete
- **Environments**: CRUD + reauthorize
- **Destinations**: list, get, link
- **Utility**: `ping`, `auth_status`

Tutte le operazioni distruttive richiedono `confirm: true`.

## Setup

### 1. Google Cloud
1. Abilita **Tag Manager API** in [Google Cloud Console](https://console.cloud.google.com/apis/library/tagmanager.googleapis.com)
2. Crea credenziali OAuth 2.0 (tipo: Web application)
3. Authorized redirect URI: `https://mcp-gtm-remote-moca.<tuo-subdomain>.workers.dev/callback`
4. Salva `CLIENT_ID` e `CLIENT_SECRET`

### 2. Cloudflare
```bash
cd "/Users/danielepisciottano/Desktop/Varie/Server MCP/server-mcp-remoto/mcp-gtm-remoto"
npm install

# Crea KV namespace e sostituisci l'ID in wrangler.toml
wrangler kv namespace create GTM_TOKENS

# Imposta i secrets
wrangler secret put GTM_CLIENT_ID
wrangler secret put GTM_CLIENT_SECRET
wrangler secret put GTM_REDIRECT_URI   # https://mcp-gtm-remote-moca.<subdomain>.workers.dev/callback
wrangler secret put TOKEN_ENCRYPTION_KEY  # stringa random ≥32 caratteri

# Deploy
wrangler deploy
```

### 3. Autorizzazione iniziale
Apri nel browser:
```
https://mcp-gtm-remote-moca.<subdomain>.workers.dev/auth
```
Accedi con l'account Google che ha i permessi sui container GTM. Il refresh token viene salvato in KV cifrato (AES-256-GCM) e non scade mai.

### 4. Configurazione in Claude

**Claude Desktop** (`claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "gtm": {
      "url": "https://mcp-gtm-remote-moca.<subdomain>.workers.dev/sse"
    }
  }
}
```

**Claude Code**:
```bash
claude mcp add -t sse gtm https://mcp-gtm-remote-moca.<subdomain>.workers.dev/sse
```

## Come funziona

```
Claude ──SSE──▶ Worker (Hono) ──▶ Durable Object (MCPSessionDO)
                     │                      │
                     │                      └─▶ GTMManager ──▶ googleapis.tagmanager('v2')
                     │                                              ▲
                     └──▶ AuthManager ──▶ KV (token cifrato) ───────┘
```

- Un consent OAuth → un refresh token in KV
- Ogni tool call → `getValidTokens()` decifra, verifica scadenza, se serve fa refresh, salva di nuovo
- Chi riceve il link dell'MCP server lo infila nel proprio `claude_desktop_config.json` e usa i **tuoi** token per accedere ai **tuoi** container GTM

## Scopes richiesti

```
tagmanager.readonly
tagmanager.edit.containers
tagmanager.edit.containerversions
tagmanager.delete.containers
tagmanager.publish
tagmanager.manage.users
tagmanager.manage.accounts
```

## Note

- `autoEventFilter` (su trigger click/form): bug API Google nota, viene silenziosamente scartato — configurare da UI GTM
- Nessuna mappa statica dei container: elencazione dinamica via `list_accounts` → `list_containers` → `list_workspaces`
- Versione 1.0.0
