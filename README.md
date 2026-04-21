# TikTok → Telegram + AI Reply (gg4lab / "Giulia")

Workflow n8n che:

1. Riceve i nuovi **DM** TikTok tramite il community node [`n8n-nodes-social-tiktok`](https://www.npmjs.com/package/n8n-nodes-social-tiktok) e i nuovi **commenti** tramite **webhook** alimentato da un servizio esterno.
2. Li inoltra su **Telegram** (canale o chat privata) formattati in modo leggibile.
3. Genera con un **LLM** una risposta nello stile di **Giulia** — una ragazza italiana di 27 anni, semplice, genuina, che prova e sponsorizza i prodotti **gg4lab**.
4. Invia la risposta suggerita su Telegram (per approvazione manuale).
5. Opzionalmente la pubblica in automatico su TikTok:
   - **DM** → tramite il nodo nativo `TikTok Send` del community node.
   - **Commenti** → tramite bridge HTTP opzionale (il community node non supporta ancora i commenti).

File del workflow: [`workflows/tiktok-telegram-gg4lab.json`](workflows/tiktok-telegram-gg4lab.json)

---

## 1. Import in n8n

1. Apri n8n → **Workflows** → **Import from File** → seleziona `workflows/tiktok-telegram-gg4lab.json`.
2. In alto a destra premi **Save**.
3. Configura le **credenziali** e le **environment variables** (vedi sotto).
4. Attiva il workflow (toggle **Active** in alto).

---

## 2. Credenziali richieste

| Credenziale n8n | Tipo | Dove la usi |
|---|---|---|
| `Telegram Bot (gg4lab)` | Telegram API | Nodi `Telegram: incoming`, `Telegram: AI reply`, `Telegram: error alert` |
| `OpenAI (gg4lab)` | OpenAI API | Nodo `AI: Giulia reply` |
| `TikTok Credential (gg4lab)` | `tiktokApi` (community) | Nodi `TikTok Trigger (DM)` e `TikTok Send (DM autoreply)` |

### Telegram
1. Crea un bot con [@BotFather](https://t.me/BotFather) → ottieni l'**access token**.
2. In n8n → **Credentials** → **New** → *Telegram API* → nome: `Telegram Bot (gg4lab)` → incolla il token.
3. Recupera il **chat_id** destinazione (tua chat o canale):
   - Manda un messaggio al bot, poi apri `https://api.telegram.org/bot<TOKEN>/getUpdates` e prendi `chat.id`.
   - Se è un canale, aggiungi il bot come admin e usa l'ID del canale (inizia con `-100`).

### OpenAI
1. Prendi una chiave su https://platform.openai.com/api-keys.
2. In n8n → **Credentials** → **New** → *OpenAI API* → nome: `OpenAI (gg4lab)`.
3. Il modello di default nel workflow è `gpt-4o-mini` (veloce e molto economico). Puoi cambiarlo nel nodo `AI: Giulia reply`.

> Vuoi usare Anthropic (Claude) al posto di OpenAI? Sostituisci il nodo `AI: Giulia reply` con un nodo *Anthropic Chat Model* o un *HTTP Request* verso `https://api.anthropic.com/v1/messages` — il system prompt è già scritto in italiano e funziona identico.

### TikTok (community node)

1. In n8n apri **Settings → Community Nodes → Install** e inserisci `n8n-nodes-social-tiktok`, conferma il warning "I understand the risks".
2. Dopo l'installazione, apri **Credentials → New → TikTok API** (tipo `tiktokApi`) e chiamala esattamente **`TikTok Credential (gg4lab)`**, così matcha il placeholder già presente nei nodi `TikTok Trigger (DM)` e `TikTok Send (DM autoreply)`. Se la chiami diversamente, dopo l'import riapri i due nodi e riseleziona la credenziale dal dropdown.
3. Segui la procedura del community node per loggare il tuo account TikTok (tipicamente via sessione/cookie o QR, non OAuth ufficiale).

> **Limite noto:** il pacchetto copre solo i **DM** (conversazioni). Per i **commenti pubblici** il workflow mantiene il webhook `POST /webhook/tiktok/comment`, alimentato da un servizio esterno (TikAPI.io, Apify, scraper custom).

---

## 3. Environment variables

Impostale in n8n (Settings → Variables, oppure in `.env` del container):

| Variabile | Obbligatoria | Descrizione |
|---|---|---|
| `TELEGRAM_CHAT_ID` | ✅ | ID chat/canale Telegram dove ricevere notifiche e risposte AI |
| `TIKTOK_AUTOREPLY_ENABLED` | ❌ | `true` per pubblicare in automatico la risposta su TikTok (DM nativo + commenti via bridge), `false` (default) per limitarsi al suggerimento su Telegram |
| `TIKTOK_BRIDGE_URL` | ❌ | URL del bridge esterno usato **solo per l'autoreply sui commenti** (i DM passano dal nodo nativo) |
| `TIKTOK_BRIDGE_TOKEN` | ❌ | Bearer token del bridge (solo commenti) |

---

## 4. Come alimentare il workflow

### DM → nessuna configurazione di endpoint

Il nodo **`TikTok Trigger (DM)`** del community node si mette in ascolto automaticamente sulla sessione TikTok configurata nella credenziale `tiktokApi`. Non servono webhook esterni per i DM: quando arriva un nuovo messaggio, il workflow parte.

Schema dati emesso dal trigger (usato da `Normalize DM`):

```json
{
  "data": {
    "message_type": "text",              // "text" | "image" | "sticker" (solo "text" arriva all'LLM)
    "text": "ciao hai un codice sconto?",
    "conversation_id": "...",
    "conversation_short_id": "...",
    "from_user_id": "...",
    "from_user_name": "Marco"
  }
}
```

Messaggi con `message_type` diverso da `text` vengono scartati da `Filter DM: text only` (puoi modificarlo se vuoi gestire anche sticker/immagini).

### Commenti → webhook alimentato da servizio esterno

TikTok **non offre un'API pubblica ufficiale** per leggere i commenti di profili creator e il community node non espone (ancora) un trigger per i commenti. Devi alimentare tu l'endpoint con uno di:

- **TikAPI.io** / **Apify TikTok actor** (polling schedulato → POST al webhook di n8n).
- Un piccolo scraper su Playwright che gira su un VPS.
- TikTok Business API (se hai account Business con accesso ai comment events).

Endpoint esposto dal workflow:

- `POST https://<TUO_N8N>/webhook/tiktok/comment`

### Payload atteso — commenti

```json
{
  "profile": "gg4lab_giulia",
  "author": "@marco.rossi",
  "author_name": "Marco Rossi",
  "text": "ma dove lo compro??",
  "video_id": "7348912345678901234",
  "comment_id": "6945000000000000001",
  "timestamp": "2026-04-21T10:12:03Z"
}
```

I campi **extra** sono ignorati; i campi **mancanti** vengono sostituiti con default sicuri.

Test veloce con `curl` (commento):

```bash
curl -X POST https://<TUO_N8N>/webhook/tiktok/comment \
  -H 'Content-Type: application/json' \
  -d '{"profile":"gg4lab_giulia","author":"@test","author_name":"Test","text":"bellissimo prodotto!!","video_id":"123","comment_id":"456"}'
```

Per testare i **DM**, manda un messaggio privato all'account TikTok loggato nel community node: il workflow parte da solo.

Dovresti vedere arrivare su Telegram (1) il messaggio originale e (2) la risposta generata da Giulia.

---

## 5. Persona "Giulia" — prompt

Il system prompt è già scritto nel nodo `AI: Giulia reply`. In sintesi:

- Ragazza italiana, 27 anni, solare e genuina.
- Italiano informale da chat: minuscolo, frasi corte, qualche `cmq`/`nn` ma non esagerato.
- 0-2 emoji per risposta, solo comuni (😊 🫶 💕 ✨ 🥰 🙈).
- Mai venditrice aggressiva, mai tutto maiuscolo, mai hashtag spam.
- Non inventa specifiche dei prodotti gg4lab che non conosce.
- Non promette codici sconto non autorizzati.
- Niente politica / religione / argomenti sensibili.
- Non dichiara mai di essere un'AI.

**Formato output:**
- Commento pubblico → risposta max 1-2 frasi, ~120 caratteri.
- DM → risposta 2-4 frasi, più personale.

Puoi personalizzare la persona (nome, età, tono, prodotti specifici) modificando direttamente il campo `system` del nodo `AI: Giulia reply`.

---

## 6. Autoreply su TikTok (opzionale)

Se metti `TIKTOK_AUTOREPLY_ENABLED=true`, lo `Switch` finale indirizza la risposta AI su due rami:

- **DM** → nodo nativo **`TikTok Send (DM autoreply)`** del community node, che usa `conversation_id`/`conversation_short_id` del trigger per rispondere nella stessa chat.
- **Commenti** → nodo **`TikTok bridge (HTTP, comments only)`** che chiama il tuo servizio in `POST {TIKTOK_BRIDGE_URL}/reply` con questo body:

  ```json
  {
    "type": "comment",
    "profile": "gg4lab_giulia",
    "target_author": "@marco.rossi",
    "item_id": "6945000000000000001",
    "context_id": "7348912345678901234",
    "reply": "aww grazie 🥰 lo trovi sul sito gg4lab!"
  }
  ```

  Il bridge è responsabile di chiamare TikTok (via TikAPI, scraper, ecc.) con le credenziali del profilo giusto.

> **Consiglio**: lascia `TIKTOK_AUTOREPLY_ENABLED=false` all'inizio. Guarda per qualche giorno le risposte suggerite su Telegram e, quando ti fidi, attiva l'autoreply.

---

## 7. Gestione errori

Il nodo **Error Trigger** → `Telegram: error alert` ti manda un alert su Telegram se qualche nodo del workflow fallisce, con workflow/nodo/messaggio e link diretto all'esecuzione.

---

## 8. Struttura del flusso

```
TikTok Trigger (DM) ── Normalize DM ── Filter(message_type=='text') ─┐
                                                                    │
Webhook Comment ── ACK 200 ── Normalize Comment ────────────────────┼── Merge ── Filter(empty) ── TG(in) ── LLM(Giulia) ── Collect ── TG(reply) ── Switch(autoreply)
                                                                    │                                                                              ├── type=dm   → TikTok Send (DM)
                                                                    │                                                                              └── type=comm → TikTok bridge (HTTP, opz.)
Error Trigger ── Telegram: error alert
```

L'ACK 200 sul webhook commenti risponde subito al mittente, così il servizio esterno che pusha i commenti non va in timeout anche se la generazione LLM impiega qualche secondo.
