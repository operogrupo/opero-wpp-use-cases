# 01 - Auto-Reply Bot

Acknowledge every incoming WhatsApp message and respond contextually using Claude Haiku, with typing indicators and conversation history.

## Problem

A small law firm receives 50+ WhatsApp messages daily outside business hours. Clients expect instant responses, but unanswered messages lead to lost retainers. Hiring someone for 24/7 coverage costs $3,000/month. Meanwhile, clients who wait more than 10 minutes move on to another firm.

## Solution

Deploy an auto-reply bot powered by Claude Haiku that:
- Acknowledges every message within seconds with a typing indicator
- Maintains conversation history so replies stay contextual
- Knows the firm's services, hours, and common Q&A
- Hands off to a human when the conversation gets complex
- Tags each message with AI metadata for later review

## Architecture

```
Client sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |  Claude Haiku    |
         |                                 |  (Anthropic API) |
         |                                 +------------------+
         |                                          |
         |      POST /api/numbers/:id/presence      |
         |      POST /api/numbers/:id/messages/text  |
         +------------------------------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID; // Your registered number ID

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// In-memory conversation history (use Redis in production)
const conversations = new Map();
const MAX_HISTORY = 20;

// ── System prompt with real business context ──────────────────────────
const SYSTEM_PROMPT = `You are the virtual assistant for Martinez & Associates, a family law firm in Buenos Aires.

Office hours: Monday-Friday 9:00-18:00 (Argentina time, UTC-3).
Services: divorce proceedings, custody arrangements, prenuptial agreements, estate planning.
Consultation fee: $5,000 ARS for the first meeting (30 min).

Rules:
- Be warm but professional. Use "usted" form in Spanish.
- If someone asks about pricing beyond the consultation fee, say you'll connect them with an attorney.
- If the matter sounds urgent (domestic violence, emergency custody), provide the 24h hotline: +54 11 4444-5555.
- Never give legal advice. You can explain processes in general terms.
- Keep responses under 200 words.
- If you don't know something, say so honestly and offer to have an attorney follow up.`;

// ── Opero API helpers ─────────────────────────────────────────────────
async function operoFetch(path, options = {}) {
  const res = await fetch(`${OPERO_BASE_URL}${path}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${OPERO_API_KEY}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`Opero API error ${res.status}: ${text}`);
  }

  return res.json();
}

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
  });
}

async function sendText(phone, text, metadata = {}) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/text`, {
    method: 'POST',
    body: JSON.stringify({ phone, text, metadata }),
  });
}

async function markAsRead(phone, messageIds) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/read`, {
    method: 'POST',
    body: JSON.stringify({ phone, message_ids: messageIds }),
  });
}

async function fetchConversationHistory(phone) {
  try {
    const res = await operoFetch(
      `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=10`
    );
    return res.data || [];
  } catch {
    return [];
  }
}

// ── Build conversation context ────────────────────────────────────────
function getConversationHistory(phone) {
  if (!conversations.has(phone)) {
    conversations.set(phone, []);
  }
  return conversations.get(phone);
}

function addToHistory(phone, role, content) {
  const history = getConversationHistory(phone);
  history.push({ role, content });

  // Keep history bounded
  if (history.length > MAX_HISTORY) {
    // Always keep the first message for context
    const first = history[0];
    conversations.set(phone, [first, ...history.slice(-MAX_HISTORY + 1)]);
  }
}

// ── AI response generation ────────────────────────────────────────────
async function generateReply(phone, incomingMessage) {
  addToHistory(phone, 'user', incomingMessage);

  const history = getConversationHistory(phone);

  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 512,
    system: SYSTEM_PROMPT,
    messages: history,
  });

  const reply = response.content[0].text;
  addToHistory(phone, 'assistant', reply);

  return {
    text: reply,
    usage: {
      input_tokens: response.usage.input_tokens,
      output_tokens: response.usage.output_tokens,
    },
  };
}

// ── Webhook handler ───────────────────────────────────────────────────
app.post('/webhook', async (req, res) => {
  // Respond immediately so Opero doesn't retry
  res.status(200).json({ received: true });

  const { event, data } = req.body;

  // Only handle incoming text messages
  if (event !== 'message.received') return;
  if (data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';

  if (!messageText.trim()) return;

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    // 1. Mark message as read
    await markAsRead(phone, [data.wa_message_id || data.id]);

    // 2. Show typing indicator
    await sendTyping(phone);

    // 3. Generate AI reply
    const { text: reply, usage } = await generateReply(phone, messageText);

    // 4. Simulate natural typing delay (40ms per character, max 3s)
    const typingDelay = Math.min(reply.length * 40, 3000);
    await new Promise(resolve => setTimeout(resolve, typingDelay));

    // 5. Send reply with metadata
    await sendText(phone, reply, {
      agent: 'auto-reply-bot',
      model: 'claude-haiku-4-20250414',
      tokens_used: usage.input_tokens + usage.output_tokens,
      conversation_length: getConversationHistory(phone).length,
      generated_at: new Date().toISOString(),
    });

    console.log(`[${new Date().toISOString()}] Reply sent to ${phone} (${usage.input_tokens + usage.output_tokens} tokens)`);
  } catch (err) {
    console.error(`[${new Date().toISOString()}] Error handling message from ${phone}:`, err.message);

    // Send a fallback message so the client isn't left hanging
    try {
      await sendText(phone, 'Disculpe, estamos experimentando dificultades. Un abogado se comunicará con usted a la brevedad.', {
        agent: 'auto-reply-bot',
        error: true,
        error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback message also failed:', fallbackErr.message);
    }
  }
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    conversations_active: conversations.size,
    uptime: process.uptime(),
  });
});

// ── Start server ──────────────────────────────────────────────────────
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Auto-reply bot listening on port ${PORT}`);
  console.log(`Webhook URL: POST http://localhost:${PORT}/webhook`);
});
```

## Metadata Schema

Each outbound message is tagged with:

```json
{
  "agent": "auto-reply-bot",
  "model": "claude-haiku-4-20250414",
  "tokens_used": 342,
  "conversation_length": 5,
  "generated_at": "2026-04-02T14:30:00.000Z"
}
```

On error:

```json
{
  "agent": "auto-reply-bot",
  "error": true,
  "error_message": "Anthropic API rate limit exceeded"
}
```

## How to Run

```bash
# 1. Create the project
mkdir auto-reply-bot && cd auto-reply-bot
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"

# 3. Start the server
node server.js

# 4. Expose locally with ngrok (for development)
ngrok http 3000

# 5. Register your webhook with Opero
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'
```
