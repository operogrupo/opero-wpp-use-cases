# 52 - Multi-Language Customer Support

Multi-language customer support via WhatsApp. Auto-detects the customer's language from their message, responds in the same language, and stores language preference in metadata. Supports 10+ languages via GPT-4o.

## Problem

A global e-commerce platform serves customers across Latin America, Brazil, Europe, and the US. The support team only speaks Spanish and English, but 30% of inquiries come in Portuguese, French, German, and other languages. These get routed to expensive third-party translation services that add 24h delays. Customers who can't communicate in their language have a 3x higher churn rate.

## Solution

Deploy a multi-language support bot that:
- Auto-detects the customer's language from their first message
- Responds fluently in the same language using GPT-4o
- Stores language preference in metadata for future interactions
- Handles common support tasks (order status, returns, FAQs) in any language
- Translates conversation summaries to the agent's language for review
- Supports 10+ languages without separate bot instances

## Architecture

```
Customer sends message (any language)
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
| wpp-api.opero.so |                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |    GPT-4o        |
         |                                 |   (OpenAI API)   |
         |                                 +------------------+
         |                                          |
         |       detect language, respond           |
         |       in same language                   |
         |                                          |
         |    GET/PUT metadata, POST /messages/text |
         +------------------------------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import OpenAI from 'openai';

const app = express();
app.use(express.json());

// -- Config --
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const SUPPORT_TEAM_LANGUAGE = process.env.SUPPORT_TEAM_LANGUAGE || 'Spanish';

const openai = new OpenAI({ apiKey: OPENAI_API_KEY });

// -- Supported languages --
const SUPPORTED_LANGUAGES = {
  en: 'English',
  es: 'Spanish',
  pt: 'Portuguese',
  fr: 'French',
  de: 'German',
  it: 'Italian',
  nl: 'Dutch',
  ja: 'Japanese',
  ko: 'Korean',
  zh: 'Chinese',
  ar: 'Arabic',
  hi: 'Hindi',
  ru: 'Russian',
  tr: 'Turkish',
};

// -- Knowledge base (in production, connect to your help center) --
const KNOWLEDGE_BASE = {
  shipping: {
    content: 'Standard shipping: 5-7 business days. Express: 2-3 business days ($9.99). Free shipping on orders over $50. International shipping available to 40+ countries (7-14 business days).',
  },
  returns: {
    content: 'Returns accepted within 30 days of delivery. Items must be unused and in original packaging. Free return shipping for defective items. Refund processed within 5-7 business days after we receive the item.',
  },
  payment: {
    content: 'We accept Visa, Mastercard, American Express, PayPal, and bank transfer. Payment is processed immediately. All transactions are secured with SSL encryption.',
  },
  order_status: {
    content: 'To check your order status, provide your order number (format: ORD-XXXXXX). You\'ll receive tracking information via WhatsApp once your order ships.',
  },
  warranty: {
    content: '1-year warranty on all products. Coverage includes manufacturing defects. Not covered: normal wear, accidental damage, unauthorized modifications.',
  },
  contact: {
    content: 'Email: support@example.com. Phone: +1-800-555-0123 (Mon-Fri 9AM-6PM EST). Live chat available on our website.',
  },
};

// -- Opero API helpers --
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

async function getConversationMetadata(phone) {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`);
    return res.data?.metadata || {};
  } catch {
    return {};
  }
}

async function updateConversationMetadata(phone, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    method: 'PUT',
    body: JSON.stringify({ metadata }),
  });
}

async function getMessages(phone, limit = 10) {
  try {
    const res = await operoFetch(
      `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`
    );
    return res.data || [];
  } catch {
    return [];
  }
}

// -- Language detection --
async function detectLanguage(text) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    max_tokens: 20,
    messages: [
      {
        role: 'system',
        content: 'Detect the language of the user\'s message. Respond with ONLY the ISO 639-1 language code (e.g., "en", "es", "pt", "fr", "de", "it", "ja", "ko", "zh", "ar", "hi", "ru", "tr", "nl"). If unsure, respond with "en".',
      },
      { role: 'user', content: text },
    ],
  });

  const code = response.choices[0].message.content.trim().toLowerCase().replace(/[^a-z]/g, '');
  return SUPPORTED_LANGUAGES[code] ? code : 'en';
}

// -- Generate support response --
async function generateResponse(phone, message, metadata) {
  const language = SUPPORTED_LANGUAGES[metadata.language] || 'English';

  const history = await getMessages(phone, 10);
  const messages = history
    .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    .map(m => ({
      role: m.from_me ? 'assistant' : 'user',
      content: m.text || m.content?.text || '',
    }))
    .filter(m => m.content);

  messages.push({ role: 'user', content: message });

  const knowledgeContext = Object.entries(KNOWLEDGE_BASE)
    .map(([topic, data]) => `[${topic}]: ${data.content}`)
    .join('\n\n');

  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    max_tokens: 600,
    messages: [
      {
        role: 'system',
        content: `You are a friendly and helpful customer support agent. You MUST respond in ${language}. The customer's detected language is ${language} -- always use this language regardless of what language the knowledge base is in.

Knowledge base (translate as needed):
${knowledgeContext}

Customer profile:
- Name: ${metadata.customer_name || 'Unknown'}
- Order history: ${metadata.orders?.length || 0} orders
- Previous issues: ${metadata.issue_history?.length || 0}

Rules:
- Always respond in ${language}
- Be empathetic and solution-oriented
- If you can answer from the knowledge base, do so
- If the issue requires human intervention, say so and use [ESCALATE]
- If the customer provides an order number, use [ORDER:NUMBER]
- Keep responses under 150 words
- Be culturally appropriate for the language/region`,
      },
      ...messages.slice(-20),
    ],
  });

  return {
    text: response.choices[0].message.content,
    usage: response.usage,
  };
}

// -- Generate conversation summary for support team --
async function generateSummary(phone, metadata) {
  const history = await getMessages(phone, 20);
  const conversation = history
    .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    .map(m => {
      const role = m.from_me ? 'Agent' : 'Customer';
      return `${role}: ${m.text || m.content?.text || ''}`;
    })
    .filter(m => m)
    .join('\n');

  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    max_tokens: 300,
    messages: [
      {
        role: 'system',
        content: `Summarize this customer support conversation in ${SUPPORT_TEAM_LANGUAGE}. Include: customer's issue, language they speak, what was discussed, current status, and any action items. Keep it under 100 words.`,
      },
      { role: 'user', content: `Customer language: ${SUPPORTED_LANGUAGES[metadata.language] || 'Unknown'}\n\n${conversation}` },
    ],
  });

  return response.choices[0].message.content;
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';
  if (!messageText.trim()) return;

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    await sendTyping(phone);

    const metadata = await getConversationMetadata(phone);

    // Detect language on first message or if not set
    if (!metadata.language) {
      const langCode = await detectLanguage(messageText);
      metadata.language = langCode;
      metadata.language_name = SUPPORTED_LANGUAGES[langCode];
      metadata.language_detected_at = new Date().toISOString();
      console.log(`Language detected for ${phone}: ${SUPPORTED_LANGUAGES[langCode]}`);
    }

    // Initialize support data
    if (!metadata.issue_history) metadata.issue_history = [];
    if (!metadata.interaction_count) metadata.interaction_count = 0;
    metadata.interaction_count++;

    // Generate response in customer's language
    const { text: reply, usage } = await generateResponse(phone, messageText, metadata);

    // Process action tags
    let finalResponse = reply;

    // Check for escalation
    if (reply.includes('[ESCALATE]')) {
      finalResponse = finalResponse.replace(/\[ESCALATE\]/g, '');
      metadata.needs_human = true;
      metadata.escalated_at = new Date().toISOString();

      // Generate summary for support team
      const summary = await generateSummary(phone, metadata);
      metadata.last_summary = summary;
    }

    // Check for order lookup
    const orderMatch = reply.match(/\[ORDER:([^\]]+)\]/);
    if (orderMatch) {
      finalResponse = finalResponse.replace(orderMatch[0], '');
      metadata.last_order_queried = orderMatch[1].trim();
    }

    metadata.last_interaction = new Date().toISOString();
    await updateConversationMetadata(phone, metadata);

    // Natural delay
    const typingDelay = Math.min(finalResponse.length * 30, 3000);
    await new Promise(resolve => setTimeout(resolve, typingDelay));

    await sendText(phone, finalResponse, {
      agent: 'multi-language-support',
      model: 'gpt-4o',
      language: metadata.language,
      language_name: metadata.language_name,
      tokens_used: (usage?.prompt_tokens || 0) + (usage?.completion_tokens || 0),
      generated_at: new Date().toISOString(),
    });

    console.log(`[${new Date().toISOString()}] Reply sent to ${phone} in ${metadata.language_name}`);
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      // Fallback in English
      await sendText(phone, 'Sorry, we are experiencing technical difficulties. Please try again in a moment.', {
        agent: 'multi-language-support', error: true, error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback failed:', fallbackErr.message);
    }
  }
});

// -- Admin: get conversation summary --
app.get('/admin/summary/:phone', async (req, res) => {
  try {
    const { phone } = req.params;
    const metadata = await getConversationMetadata(phone);
    const summary = await generateSummary(phone, metadata);
    res.json({
      phone,
      language: metadata.language_name,
      summary,
      interaction_count: metadata.interaction_count,
      needs_human: metadata.needs_human || false,
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: override language --
app.post('/admin/language/:phone', async (req, res) => {
  try {
    const { phone } = req.params;
    const { language } = req.body;
    const metadata = await getConversationMetadata(phone);

    if (!SUPPORTED_LANGUAGES[language]) {
      return res.status(400).json({
        error: 'Unsupported language',
        supported: Object.entries(SUPPORTED_LANGUAGES).map(([code, name]) => ({ code, name })),
      });
    }

    metadata.language = language;
    metadata.language_name = SUPPORTED_LANGUAGES[language];
    metadata.language_override = true;
    await updateConversationMetadata(phone, metadata);

    res.json({ success: true, language: SUPPORTED_LANGUAGES[language] });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    supported_languages: Object.keys(SUPPORTED_LANGUAGES).length,
    uptime: process.uptime(),
  });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Multi-language support bot listening on port ${PORT}`);
  console.log(`Supported languages: ${Object.values(SUPPORTED_LANGUAGES).join(', ')}`);
});
```

## Metadata Schema

Each customer conversation stores language and support data:

```json
{
  "language": "pt",
  "language_name": "Portuguese",
  "language_detected_at": "2026-04-02T10:00:00.000Z",
  "language_override": false,
  "customer_name": "Joao Silva",
  "interaction_count": 5,
  "issue_history": [
    {
      "topic": "shipping",
      "resolved": true,
      "resolved_at": "2026-03-28T15:00:00.000Z"
    }
  ],
  "orders": ["ORD-123456"],
  "last_order_queried": "ORD-123456",
  "needs_human": false,
  "last_summary": "Cliente perguntou sobre status do pedido ORD-123456...",
  "last_interaction": "2026-04-02T14:30:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "multi-language-support",
  "model": "gpt-4o",
  "language": "pt",
  "language_name": "Portuguese",
  "tokens_used": 520,
  "generated_at": "2026-04-02T14:30:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir multi-language-support && cd multi-language-support
npm init -y
npm install express openai

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export OPENAI_API_KEY="your-openai-api-key"
export SUPPORT_TEAM_LANGUAGE="Spanish"

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

# 6. Get a conversation summary (translated to team language)
curl http://localhost:3000/admin/summary/5491155559999

# 7. Override a customer's language
curl -X POST http://localhost:3000/admin/language/5491155559999 \
  -H "Content-Type: application/json" \
  -d '{ "language": "fr" }'
```
