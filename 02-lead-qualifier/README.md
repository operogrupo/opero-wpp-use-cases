# 02 - Lead Qualifier

Classify incoming WhatsApp messages by intent, sentiment, urgency, and lead score using GPT-4o-mini, then route high-value leads to a human sales rep.

## Problem

A solar panel installation company gets 200+ WhatsApp inquiries daily. Sales reps waste 60% of their time talking to people who are "just asking" or won't qualify for financing. Meanwhile, hot leads -- homeowners ready to sign -- wait in a queue behind tire-kickers. The company estimates they lose $15,000/month in deals that went cold because response time to qualified leads averaged 4 hours.

## Solution

Every incoming message is scored by GPT-4o-mini in real-time:
- **Intent** classified as `question`, `purchase`, `complaint`, `support`, or `spam`
- **Sentiment** detected as `positive`, `neutral`, or `negative`
- **Urgency** tagged as `low`, `medium`, or `high`
- **Lead score** from 0.0 to 1.0 based on buying signals
- Leads scoring above 0.7 are immediately tagged in metadata and the sales team is notified
- Low-score messages get an automated response; high-score messages get a human

## Architecture

```
WhatsApp message arrives
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |   GPT-4o-mini    |
         |                                 | (structured out) |
         |                                 +------------------+
         |                                          |
         |    PATCH message metadata                |
         |    PUT conversation metadata             |
         |    POST send reply (if auto)             |
         +------------------------------------------+
                                                    |
                                           (if score > 0.7)
                                                    v
                                           +------------------+
                                           | Slack/Email/CRM  |
                                           | notification     |
                                           +------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import OpenAI from 'openai';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const SLACK_WEBHOOK_URL = process.env.SLACK_WEBHOOK_URL; // Optional
const LEAD_SCORE_THRESHOLD = 0.7;

const openai = new OpenAI({ apiKey: OPENAI_API_KEY });

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

async function sendText(phone, text, metadata = {}) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/text`, {
    method: 'POST',
    body: JSON.stringify({ phone, text, metadata }),
  });
}

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
  });
}

async function tagMessage(messageId, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/_/messages/${messageId}`, {
    method: 'PATCH',
    body: JSON.stringify({ metadata }),
  });
}

async function updateConversation(phone, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    method: 'PUT',
    body: JSON.stringify({ metadata }),
  });
}

async function getConversation(phone) {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`);
    return res.data;
  } catch {
    return null;
  }
}

// ── Lead classification with GPT-4o-mini ──────────────────────────────
const CLASSIFICATION_SCHEMA = {
  type: 'object',
  properties: {
    intent: {
      type: 'string',
      enum: ['question', 'purchase', 'complaint', 'support', 'spam'],
      description: 'Primary intent of the message',
    },
    sentiment: {
      type: 'string',
      enum: ['positive', 'neutral', 'negative'],
    },
    urgency: {
      type: 'string',
      enum: ['low', 'medium', 'high'],
    },
    lead_score: {
      type: 'number',
      minimum: 0,
      maximum: 1,
      description: 'Likelihood of converting to a sale. 0 = no buying intent, 1 = ready to buy now.',
    },
    buying_signals: {
      type: 'array',
      items: { type: 'string' },
      description: 'Specific phrases or signals that indicate purchase intent',
    },
    summary: {
      type: 'string',
      description: 'One-line summary for the sales team',
    },
  },
  required: ['intent', 'sentiment', 'urgency', 'lead_score', 'buying_signals', 'summary'],
  additionalProperties: false,
};

async function classifyMessage(message, conversationHistory = '') {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    temperature: 0,
    response_format: { type: 'json_schema', json_schema: { name: 'lead_classification', schema: CLASSIFICATION_SCHEMA, strict: true } },
    messages: [
      {
        role: 'system',
        content: `You are a lead qualification AI for SolarTech Argentina, a solar panel installation company.

Context about the business:
- Residential solar panel installation: $2,000-8,000 USD
- Commercial installations: $10,000-50,000 USD
- Financing available for homeowners
- Service area: Greater Buenos Aires
- Installation timeline: 2-4 weeks

High lead_score signals (0.7-1.0):
- Mentions budget, pricing, financing
- Asks about timeline or availability
- Mentions property type or roof
- Wants a quote or site visit
- Expresses urgency ("this month", "as soon as possible")

Medium lead_score signals (0.3-0.7):
- General questions about solar energy
- Asks about savings or ROI
- Comparing options

Low lead_score signals (0.0-0.3):
- Just browsing or curious
- Student doing research
- Clearly not in service area
- Spam or irrelevant

Conversation history (if any):
${conversationHistory}`,
      },
      { role: 'user', content: message },
    ],
  });

  return JSON.parse(response.choices[0].message.content);
}

// ── Auto-response templates ───────────────────────────────────────────
const AUTO_RESPONSES = {
  question: 'Gracias por su consulta sobre energia solar. Un asesor revisara su mensaje y le respondera a la brevedad. Mientras tanto, puede ver nuestros proyectos en solartecharg.com/proyectos',
  support: 'Recibimos su mensaje. Nuestro equipo de soporte se pondra en contacto con usted dentro de las proximas 2 horas habiles.',
  complaint: 'Lamentamos la situacion. Su caso ha sido escalado a nuestro equipo de atencion prioritaria. Le contactaremos en menos de 1 hora.',
  spam: null, // Don't respond to spam
};

// ── Notification to sales team ────────────────────────────────────────
async function notifySalesTeam(phone, classification, messageText) {
  const urgencyEmoji = { low: '', medium: '', high: '' };

  const payload = {
    text: `${urgencyEmoji[classification.urgency]} New hot lead!`,
    blocks: [
      {
        type: 'header',
        text: { type: 'plain_text', text: `Hot Lead: ${phone}` },
      },
      {
        type: 'section',
        fields: [
          { type: 'mrkdwn', text: `*Score:* ${classification.lead_score}` },
          { type: 'mrkdwn', text: `*Intent:* ${classification.intent}` },
          { type: 'mrkdwn', text: `*Urgency:* ${classification.urgency}` },
          { type: 'mrkdwn', text: `*Sentiment:* ${classification.sentiment}` },
        ],
      },
      {
        type: 'section',
        text: { type: 'mrkdwn', text: `*Message:* ${messageText}` },
      },
      {
        type: 'section',
        text: { type: 'mrkdwn', text: `*Summary:* ${classification.summary}` },
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*Buying signals:* ${classification.buying_signals.join(', ') || 'None detected'}`,
        },
      },
    ],
  };

  if (SLACK_WEBHOOK_URL) {
    await fetch(SLACK_WEBHOOK_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
  } else {
    console.log('[NOTIFICATION]', JSON.stringify(payload, null, 2));
  }
}

// ── Webhook handler ───────────────────────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;

  if (event !== 'message.received') return;
  if (data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';

  if (!messageText.trim()) return;

  console.log(`[${new Date().toISOString()}] Classifying message from ${phone}`);

  try {
    // 1. Get existing conversation context
    const conversation = await getConversation(phone);
    const existingMeta = conversation?.metadata || {};
    const previousClassifications = existingMeta.classifications || [];

    // Build history context for better classification
    const historyContext = previousClassifications
      .slice(-3)
      .map(c => `[${c.intent}/${c.sentiment}] score=${c.lead_score}`)
      .join(' -> ');

    // 2. Classify the message
    const classification = await classifyMessage(messageText, historyContext);

    console.log(`[${new Date().toISOString()}] Classification:`, JSON.stringify(classification));

    // 3. Tag the individual message with classification
    if (data.id) {
      await tagMessage(data.id, {
        classification,
        classified_at: new Date().toISOString(),
        classifier: 'gpt-4o-mini',
      });
    }

    // 4. Update conversation-level metadata
    const updatedClassifications = [...previousClassifications, {
      ...classification,
      message_id: data.id,
      timestamp: new Date().toISOString(),
    }].slice(-20); // Keep last 20 classifications

    // Calculate rolling average lead score
    const avgScore = updatedClassifications.reduce((sum, c) => sum + c.lead_score, 0)
      / updatedClassifications.length;

    const isHotLead = classification.lead_score >= LEAD_SCORE_THRESHOLD;
    const wasAlreadyHot = existingMeta.is_hot_lead === true;

    await updateConversation(phone, {
      ...existingMeta,
      classifications: updatedClassifications,
      latest_intent: classification.intent,
      latest_sentiment: classification.sentiment,
      latest_urgency: classification.urgency,
      latest_lead_score: classification.lead_score,
      average_lead_score: Math.round(avgScore * 100) / 100,
      is_hot_lead: isHotLead || wasAlreadyHot,
      total_messages: (existingMeta.total_messages || 0) + 1,
      last_classified_at: new Date().toISOString(),
    });

    // 5. Route based on score
    if (isHotLead) {
      // Notify sales team immediately
      await notifySalesTeam(phone, classification, messageText);

      // Send personalized acknowledgment
      await sendTyping(phone);
      await new Promise(resolve => setTimeout(resolve, 1500));
      await sendText(phone, `Excelente, gracias por su interes! Un asesor especializado se comunicara con usted en los proximos minutos para atender su consulta personalmente.`, {
        agent: 'lead-qualifier',
        routed_to: 'human',
        lead_score: classification.lead_score,
      });
    } else if (AUTO_RESPONSES[classification.intent]) {
      // Send auto-response for non-hot leads
      await sendTyping(phone);
      await new Promise(resolve => setTimeout(resolve, 2000));
      await sendText(phone, AUTO_RESPONSES[classification.intent], {
        agent: 'lead-qualifier',
        routed_to: 'auto',
        lead_score: classification.lead_score,
      });
    }
    // spam: no response sent
  } catch (err) {
    console.error(`[${new Date().toISOString()}] Error classifying message from ${phone}:`, err.message);
  }
});

// ── Dashboard endpoint: view all hot leads ────────────────────────────
app.get('/leads', async (req, res) => {
  try {
    const result = await operoFetch(
      `/api/numbers/${NUMBER_ID}/conversations?limit=100`
    );

    const conversations = result.data || [];
    const hotLeads = conversations
      .filter(c => c.metadata?.is_hot_lead)
      .map(c => ({
        phone: c.contact_phone,
        name: c.contact_name,
        lead_score: c.metadata.latest_lead_score,
        average_score: c.metadata.average_lead_score,
        intent: c.metadata.latest_intent,
        urgency: c.metadata.latest_urgency,
        total_messages: c.metadata.total_messages,
        last_activity: c.metadata.last_classified_at,
      }))
      .sort((a, b) => b.lead_score - a.lead_score);

    res.json({ hot_leads: hotLeads, total: hotLeads.length });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`Lead qualifier listening on port ${PORT}`);
});
```

## Metadata Schema

### Message-level metadata (tagged on each inbound message)

```json
{
  "classification": {
    "intent": "purchase",
    "sentiment": "positive",
    "urgency": "high",
    "lead_score": 0.85,
    "buying_signals": ["asked about pricing", "mentioned roof type", "wants quote this week"],
    "summary": "Homeowner with tile roof interested in 10-panel system, asking about financing options"
  },
  "classified_at": "2026-04-02T14:30:00.000Z",
  "classifier": "gpt-4o-mini"
}
```

### Conversation-level metadata (rolling state)

```json
{
  "latest_intent": "purchase",
  "latest_sentiment": "positive",
  "latest_urgency": "high",
  "latest_lead_score": 0.85,
  "average_lead_score": 0.72,
  "is_hot_lead": true,
  "total_messages": 5,
  "last_classified_at": "2026-04-02T14:30:00.000Z",
  "classifications": [
    {
      "intent": "question",
      "sentiment": "neutral",
      "urgency": "low",
      "lead_score": 0.3,
      "message_id": "msg_001",
      "timestamp": "2026-04-02T14:00:00.000Z"
    },
    {
      "intent": "purchase",
      "sentiment": "positive",
      "urgency": "high",
      "lead_score": 0.85,
      "message_id": "msg_002",
      "timestamp": "2026-04-02T14:30:00.000Z"
    }
  ]
}
```

## How to Run

```bash
# 1. Create the project
mkdir lead-qualifier && cd lead-qualifier
npm init -y
npm install express openai

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export OPENAI_API_KEY="your-openai-api-key"
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."  # Optional

# 3. Start the server
node server.js

# 4. Expose locally with ngrok
ngrok http 3001

# 5. Register webhook with Opero
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. View hot leads
curl http://localhost:3001/leads
```
