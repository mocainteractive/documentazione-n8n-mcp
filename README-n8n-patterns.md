# N8N WORKFLOW PATTERNS — GUIDA AI PATTERN CORRETTI MOCA

Questo documento definisce i pattern architetturali corretti da usare nella generazione di workflow n8n, basati su workflow reali e funzionanti. Va usato come riferimento obbligatorio prima di generare qualsiasi workflow con loop multi-cliente.

---

## 1. PATTERN LISTA CLIENTI — NODO CODE (non Set)

La lista clienti NON va messa in un nodo `Set` con un array JSON inline. Va messa in un **nodo `Code`** che restituisce direttamente un array di items, uno per cliente. Questo è il pattern corretto e funzionante:

```javascript
// Nodo: "Client Names" — type: n8n-nodes-base.code, parametro: jsCode
const clients = [
  {
    client_name: "Cliente A",
    property_url: "sc-domain:clientea.it",
    brand_regex: "clientea|cliente\\s?a"
  },
  {
    client_name: "Cliente B",
    property_url: "https://www.clienteb.it/",
    brand_regex: "clienteb"
  }
];

return clients.map(client => ({ json: client }));
```

Il nodo Code usa `jsCode` (non `code`) come chiave del parametro. Ogni oggetto dell'array diventa un item separato in n8n. Il `splitInBatches` successivo itera su questi items automaticamente, uno alla volta.

**ANTI-PATTERN DA NON USARE MAI:**
```
// SBAGLIATO: nodo Set con array inline + nodo Code separato per esplodere
// Questo approccio è più fragile e macchinoso.
Set "Config" { clienti: "[{...}]" } → Code "Espandi Clienti" { return config.clienti.map(...) }
```

---

## 2. LOGICA DEL LOOP CON `splitInBatches`

### COME FUNZIONA

Il nodo `splitInBatches` riceve N items (uno per cliente) e li processa uno alla volta. Ha **due output**:

| Output | Index | Quando si attiva | Cosa collegare |
|--------|-------|-----------------|----------------|
| `done` | 0 | Quando **tutti i batch sono esauriti** | Il nodo di aggregazione finale (Build Mail, Genera Report) |
| `loop` | 1 | Ad ogni iterazione, finché ci sono items | Il **primo nodo** della catena di elaborazione del singolo cliente |

### SCHEMA CORRETTO (dal workflow reale "Report SEO Clienti Daniele")

```
[Schedule Trigger]
       ↓
[Client Names]  ← Code che ritorna N items (uno per cliente)
       ↓
[Loop Over Items]  ← splitInBatches, batchSize default
    │           │
  done(0)     loop(1)
    │           │
[Build GSC Mail] [Prep GSC Data]  ← primo nodo della catena, Code che prepara il contesto
    │                  ↓
[Send Email]    [GSC Stats]
                       ↓
                [GSC Stats Previous]
                       ↓
                [GSC Queries]
                       ↓
                [GSC Pages]
                       ↓
                [Prep DataForSEO]
                       ↓
                [DFS Keywords]
                       ↓
                [OpenAI]
                       ↓
                [Format Client HTML]  ← produce html_card per questo cliente
                       ↓
                [Wait 3s]
                       ↓
              torna a [Loop Over Items] input 0
```

### CONNECTIONS CORRETTE NEL JSON

```json
"Loop Over Items": {
  "main": [
    [{ "node": "Build GSC Mail", "type": "main", "index": 0 }],
    [{ "node": "Prep GSC Data", "type": "main", "index": 0 }]
  ]
},
"Wait": {
  "main": [
    [{ "node": "Loop Over Items", "type": "main", "index": 0 }]
  ]
}
```

**CRITICO:** `main[0]` = done = nodo finale. `main[1]` = loop = primo nodo della catena. Il Wait alla fine della catena torna SEMPRE all'input 0 di `Loop Over Items`.

---

## 3. PRIMO NODO DEL LOOP — "Prep Data" CENTRALIZZATO

Subito dopo l'output loop di `splitInBatches`, il primo nodo è sempre un **nodo Code** che:
1. Legge i dati del cliente dal nodo lista via `$('Client Names').item.json`
2. Calcola tutte le date necessarie (periodo corrente e precedente)
3. Prepara il contesto completo che tutti i nodi a valle leggeranno via `$('Prep GSC Data').item.json`

```javascript
// Nodo: "Prep GSC Data" — jsCode
const originalClientData = $('Client Names').item.json;

const today = new Date();
const endDate = new Date();
endDate.setDate(today.getDate() - 3);
const startDate = new Date();
startDate.setDate(endDate.getDate() - 27); // 28 giorni
const prevEnd = new Date(startDate);
prevEnd.setDate(prevEnd.getDate() - 1);
const prevStart = new Date(prevEnd);
prevStart.setDate(prevEnd.getDate() - 27);

const formatDate = (d) => d.toISOString().split('T')[0];

return {
    json: {
        client_name: originalClientData.client_name,
        property_url: originalClientData.property_url,
        brand_regex: originalClientData.brand_regex,
        site_url: originalClientData.property_url,
        startDate: formatDate(startDate),
        endDate: formatDate(endDate),
        prevStartDate: formatDate(prevStart),
        prevEndDate: formatDate(prevEnd),
        period_string: `${formatDate(startDate)} - ${formatDate(endDate)}`
    }
};
```

**IMPORTANTE:** ritorna `{ json: {...} }` (singolo oggetto, non array), perché stiamo lavorando su un singolo item del loop.

---

## 4. PASSAGGIO DEL CONTESTO — `$('NomeNodo').item.json`

In n8n, i nodi a valle di una catena possono leggere l'output di QUALSIASI nodo upstream tramite:

```javascript
$('Prep GSC Data').item.json.site_url
$('GSC Stats').item.json.result.content[0].text
$('OpenAI').item.json.output[0].content[0].text
```

Questo funziona perché n8n mantiene il contesto dell'esecuzione corrente del loop. **NON è necessario ri-propagare i campi in ogni nodo Set intermedio.**

**ANTI-PATTERN DA NON USARE MAI:**
```json
// SBAGLIATO: ri-propagare esplicitamente decine di campi in ogni nodo Set
{
  "name": "Parse Stats P1",
  "parameters": {
    "assignments": [
      { "name": "statsP1", "value": "={{ JSON.parse($json.result.content[0].text) }}" },
      { "name": "nome", "value": "={{ $('Loop Clienti').item.json.nome }}" },
      { "name": "siteUrl", "value": "={{ $('Loop Clienti').item.json.siteUrl }}" },
      { "name": "startP1", "value": "={{ $('Loop Clienti').item.json.startP1 }}" }
      // ...e altri 10 campi ripetuti
    ]
  }
}
// Questo è ridondante, ingombrante e fragile.
```

---

## 5. ACCUMULO RISULTATI — PATTERN `$('NomeNodo').all()`

Per aggregare i risultati di tutte le iterazioni nel nodo finale (collegato a `done`), il pattern corretto usa `$('NomeNodo').all()` che recupera tutti gli output prodotti dal nodo specificato durante tutte le iterazioni del loop corrente:

```javascript
// Nodo: "Build GSC Mail" — collegato all'output done del loop
const items = [];
let runIndex = 0;
let hasMore = true;

while (hasMore) {
    try {
        const batch = $('Format Client HTML').all(0, runIndex);
        if (batch.length > 0) {
            items.push(batch[0].json);
            runIndex++;
        } else {
            hasMore = false;
        }
    } catch (e) { hasMore = false; }
}

// ora items contiene tutti gli html_card di tutti i clienti
const DATE = new Date().toLocaleDateString('it-IT');
const fullHtml = `<!DOCTYPE html><html><body>
  ${items.map(i => i.html_card).join('')}
</body></html>`;

return { json: { html_body: fullHtml, subject: `Report - ${DATE}` } };
```

**Perché NON usare `getWorkflowStaticData`:**
- `getWorkflowStaticData` persiste tra run diversi e contamina i dati se il workflow fallisce a metà
- `$('NomeNodo').all()` legge solo i dati del run corrente, è più affidabile e non richiede reset manuale

---

## 6. CONFIGURAZIONE NODI HTTP REQUEST

Per i nodi HTTP Request che chiamano i MCP server MOCA o API esterne:

```json
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.3,
  "onError": "continueRegularOutput"
}
```

- `typeVersion`: usare **4.3** (non 4.2)
- `onError`: usare **`"continueRegularOutput"`** (non il vecchio `continueOnFail: true`)
- I parametri nel `jsonBody` si referenziano con `{{ $('Prep GSC Data').item.json.site_url }}`

---

## 7. NODO WAIT — POSIZIONE E CONFIGURAZIONE

Il nodo Wait va posizionato come **ultimo nodo della catena di elaborazione**, subito prima del ritorno al loop. Il `webhookId` deve essere un UUID reale:

```json
{
  "parameters": { "amount": 3 },
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1,
  "webhookId": "ae855709-4553-4061-9b6c-266729415875"
}
```

Valori consigliati: 2-3 secondi per API Google e Meta.

---

## 8. NODO CODE — PARAMETRO `jsCode`

Il nodo Code usa **`jsCode`** come chiave del parametro (non `code`):

```json
{
  "parameters": {
    "jsCode": "return items.map(item => ({ json: item.json }));"
  },
  "type": "n8n-nodes-base.code",
  "typeVersion": 2
}
```

---

## 9. CHECKLIST PRIMA DI GENERARE UN WORKFLOW CON LOOP

- [ ] I clienti sono in un nodo **Code** (`jsCode`) che ritorna `clients.map(c => ({ json: c }))`
- [ ] Nessun nodo `Set` + `Code` separato per la lista clienti
- [ ] Il nodo Code clienti è connesso direttamente a `splitInBatches`
- [ ] `connections["Loop Over Items"].main[0]` → nodo **done** (Build Mail)
- [ ] `connections["Loop Over Items"].main[1]` → nodo **loop** (Prep Data)
- [ ] Il primo nodo della catena loop è un **Code** che legge da `$('Client Names').item.json` e calcola date + contesto
- [ ] I nodi a valle usano `$('Prep GSC Data').item.json.campo` direttamente (NO ri-propagazione)
- [ ] Il nodo finale della catena produce un campo aggregabile (es. `html_card`)
- [ ] Il nodo **Wait** è l'ultimo della catena prima del ritorno al loop
- [ ] Il nodo **Wait** torna a `Loop Over Items` input 0
- [ ] Il nodo **done** usa `$('NomeNodo').all()` per aggregare tutti i risultati
- [ ] I nodi HTTP Request hanno `onError: "continueRegularOutput"` e `typeVersion: 4.3`
- [ ] **NON** ci sono nodi `IF` per controllare `noItemsLeft`
- [ ] **NON** si usa `getWorkflowStaticData` per accumulare tra iterazioni
- [ ] **NON** si ri-propagano i campi di contesto in ogni nodo Set intermedio

---

## 10. CONNECTIONS COMPLETE — ESEMPIO MINIMO DI RIFERIMENTO

```json
"connections": {
  "Schedule Trigger": {
    "main": [[{ "node": "Client Names", "type": "main", "index": 0 }]]
  },
  "Client Names": {
    "main": [[{ "node": "Loop Over Items", "type": "main", "index": 0 }]]
  },
  "Loop Over Items": {
    "main": [
      [{ "node": "Build GSC Mail", "type": "main", "index": 0 }],
      [{ "node": "Prep GSC Data", "type": "main", "index": 0 }]
    ]
  },
  "Prep GSC Data": {
    "main": [[{ "node": "GSC Stats", "type": "main", "index": 0 }]]
  },
  "GSC Stats": {
    "main": [[{ "node": "GSC Stats Previous", "type": "main", "index": 0 }]]
  },
  "GSC Stats Previous": {
    "main": [[{ "node": "GSC Queries", "type": "main", "index": 0 }]]
  },
  "GSC Queries": {
    "main": [[{ "node": "Format Client HTML", "type": "main", "index": 0 }]]
  },
  "Format Client HTML": {
    "main": [[{ "node": "Wait", "type": "main", "index": 0 }]]
  },
  "Wait": {
    "main": [[{ "node": "Loop Over Items", "type": "main", "index": 0 }]]
  },
  "Build GSC Mail": {
    "main": [[{ "node": "Send Email", "type": "main", "index": 0 }]]
  }
}
```