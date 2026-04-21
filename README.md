# TikTok → Telegram + AI Reply (gg4lab / "Giulia")

Workflow n8n che:

1. Riceve via **webhook** i nuovi **commenti** e **DM** dai tuoi profili TikTok.
2. Li inoltra su **Telegram** (canale o chat privata) formattati in modo leggibile.
3. Genera con un **LLM** una risposta nello stile di **Giulia** — una ragazza italiana di 27 anni, semplice, genuina, che prova e sponsorizza i prodotti **gg4lab**.
4. Invia la risposta suggerita su Telegram (per approvazione manuale).
5. Opzionalmente la pubblica in automatico su TikTok tramite un bridge HTTP.

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

---

## 3. Environment variables

Impostale in n8n (Settings → Variables, oppure in `.env` del container):

| Variabile | Obbligatoria | Descrizione |
|---|---|---|
| `TELEGRAM_CHAT_ID` | ✅ | ID chat/canale Telegram dove ricevere notifiche e risposte AI |
| `TIKTOK_AUTOREPLY_ENABLED` | ❌ | `true` per pubblicare in automatico la risposta su TikTok, `false` (default) per limitarsi al suggerimento su Telegram |
| `TIKTOK_BRIDGE_URL` | ❌ | URL del tuo servizio bridge che sa parlare con TikTok (es. TikAPI, un tuo scraper) |
| `TIKTOK_BRIDGE_TOKEN` | ❌ | Bearer token del bridge |

---

## 4. Webhook: come alimentare il workflow

TikTok **non offre un'API pubblica ufficiale** per leggere commenti/DM di profili creator, quindi il workflow è progettato per ricevere eventi da un servizio esterno. Opzioni comuni:

- **TikAPI.io** / **Apify TikTok actor** (polling schedulato → POST al webhook di n8n).
- Un piccolo scraper su Playwright che tu fai girare su un VPS e chiama gli endpoint del workflow.
- Integrazione custom della TikTok Business API (se hai account Business con accesso ai comment events).

Una volta attivato il workflow, n8n espone due endpoint:

- `POST https://<TUO_N8N>/webhook/tiktok/comment`
- `POST https://<TUO_N8N>/webhook/tiktok/dm`

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

### Payload atteso — DM

```json
{
  "profile": "gg4lab_giulia",
  "author": "@marco.rossi",
  "author_name": "Marco",
  "text": "ciao, hai un codice sconto?",
  "conversation_id": "conv_abc123",
  "message_id": "msg_xyz789",
  "timestamp": "2026-04-21T10:15:40Z"
}
```

I campi **extra** sono ignorati; i campi **mancanti** vengono sostituiti con default sicuri.

Test veloce con `curl`:

```bash
curl -X POST https://<TUO_N8N>/webhook/tiktok/comment \
  -H 'Content-Type: application/json' \
  -d '{"profile":"gg4lab_giulia","author":"@test","author_name":"Test","text":"bellissimo prodotto!!","video_id":"123","comment_id":"456"}'
```

Dovresti vedere arrivare su Telegram (1) il commento originale e (2) la risposta generata da Giulia.

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

Se metti `TIKTOK_AUTOREPLY_ENABLED=true`, il nodo `TikTok bridge (HTTP)` chiama il tuo servizio in `POST {TIKTOK_BRIDGE_URL}/reply` con questo body:

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

Il bridge sarà responsabile di chiamare TikTok (via TikAPI, MS, scraper, ecc.) con le credenziali del profilo giusto.

> **Consiglio**: lascia `TIKTOK_AUTOREPLY_ENABLED=false` all'inizio. Guarda per qualche giorno le risposte suggerite su Telegram e, quando ti fidi, attiva l'autoreply.

---

## 7. Gestione errori

Il nodo **Error Trigger** → `Telegram: error alert` ti manda un alert su Telegram se qualche nodo del workflow fallisce, con workflow/nodo/messaggio e link diretto all'esecuzione.

---

## 8. Struttura del flusso

```
Webhook Comment ─┐                                                                                   ┌─ (if autoreply ON) → TikTok bridge
                 ├─ Normalize ─ Merge ─ Filter empty ─ Telegram(in) ─ LLM(Giulia) ─ Collect ─ Telegram(reply) ─┤
Webhook DM ──────┘                                                                                   └─ (else) stop

Error Trigger ── Telegram: error alert
```

Gli ACK 200 sui due webhook rispondono subito al mittente, così il servizio che pusha gli eventi non va in timeout anche se la generazione LLM impiega qualche secondo.
