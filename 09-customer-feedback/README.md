# 09 - Customer Feedback

Automatically collect post-purchase feedback via WhatsApp. Gather ratings and comments through a conversational flow, store responses as metadata, and generate weekly summary reports.

## Problem

An auto repair shop in Mendoza services 25 cars per day. They have no systematic way to collect feedback. Google review requests via email get 3% response rates. Bad experiences go undetected until a 1-star review appears publicly. The owner has no data on which mechanics get the best feedback or which services cause the most complaints. They're flying blind on customer satisfaction.

## Solution

After each service is completed, automatically:
- Send a friendly feedback request via WhatsApp (where customers already are)
- Collect a 1-5 rating through conversational flow (not a cold survey link)
- Ask a follow-up question based on the rating (what went well / what could improve)
- Store all feedback as conversation metadata
- Generate weekly reports with averages, trends, and flagged issues
- Immediately alert the manager if a rating is 2 or below

## Architecture

```
Service completed -> trigger via API
         |
         v
+--------------------+    POST /send-feedback    +--------------------+
|  Your Backend /    | -----------------------> |  Your Express App  |
|  POS system        |                          |   (this code)      |
+--------------------+                          +--------------------+
                                                         |
                                                         v
                                                +------------------+
                                                |   Opero WPP API  |
                                                |  wpp-api.opero.so|
                                                +------------------+
                                                         |
                                                   webhook POST
                                                   (customer replies)
                                                         |
                                                         v
                                                +--------------------+
                                                |  Claude Haiku      |
                                                |  (parse response)  |
                                                +--------------------+
                                                         |
                                                    Store in metadata
                                                    + alert if bad
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
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const MANAGER_PHONE = process.env.MANAGER_PHONE; // Alert bad ratings here
const SLACK_WEBHOOK_URL = process.env.SLACK_WEBHOOK_URL; // Optional

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// ── Feedback state tracking ──────────────────────────────────────────
// Stages: idle -> awaiting_rating -> awaiting_comment -> completed
const feedbackSessions = new Map(); // phone -> session state

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
  if (!res.ok) throw new Error(`Opero API error ${res.status}: ${await res.text()}`);
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

async function listConversations() {
  const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=200`);
  return res.data || [];
}

// ── Parse rating from message ─────────────────────────────────────────
function parseRating(text) {
  const cleaned = text.trim();

  // Direct number
  const numMatch = cleaned.match(/^(\d)$/);
  if (numMatch) {
    const num = parseInt(numMatch[1]);
    if (num >= 1 && num <= 5) return num;
  }

  // Star emojis
  const stars = (cleaned.match(/\u2B50|\u2605|\u2606/g) || []).length;
  if (stars >= 1 && stars <= 5) return stars;

  // Word-based ratings
  const wordMap = {
    1: [/pesim[oa]/i, /horrible/i, /muy mal/i, /1/],
    2: [/mal[oa]?$/i, /regular/i, /no me gusto/i, /2/],
    3: [/bien$/i, /normal/i, /ok$/i, /aceptable/i, /3/],
    4: [/muy bien/i, /bueno/i, /buen servicio/i, /4/],
    5: [/excelente/i, /perfecto/i, /increible/i, /espectacular/i, /genial/i, /5/],
  };

  for (const [rating, patterns] of Object.entries(wordMap)) {
    for (const pattern of patterns) {
      if (pattern.test(cleaned)) return parseInt(rating);
    }
  }

  return null;
}

// ── Claude for comment analysis ───────────────────────────────────────
async function analyzeComment(comment, rating, service) {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 256,
    system: `Analyze this customer feedback for an auto repair shop. Return a JSON object:
{
  "sentiment": "positive" | "neutral" | "negative",
  "themes": ["wait_time", "quality", "pricing", "staff", "cleanliness", "communication"],
  "actionable": true/false,
  "summary": "one line summary",
  "requires_followup": true/false
}`,
    messages: [{
      role: 'user',
      content: `Service: ${service}\nRating: ${rating}/5\nComment: "${comment}"`,
    }],
  });

  const text = response.content[0].text;
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  return jsonMatch ? JSON.parse(jsonMatch[0]) : { sentiment: 'neutral', themes: [], actionable: false, summary: comment };
}

// ── Alert for bad ratings ─────────────────────────────────────────────
async function alertBadRating(phone, rating, comment, service, customerName) {
  const message = `ALERTA: Rating bajo recibido\n\n` +
    `Cliente: ${customerName || phone}\n` +
    `Servicio: ${service}\n` +
    `Rating: ${'1'.repeat(rating)}/5\n` +
    `Comentario: ${comment || 'Sin comentario'}\n\n` +
    `Requiere atencion inmediata.`;

  // Alert via WhatsApp to manager
  if (MANAGER_PHONE) {
    await sendText(MANAGER_PHONE, message, {
      agent: 'customer-feedback',
      type: 'bad_rating_alert',
      customer_phone: phone,
      rating,
    });
  }

  // Alert via Slack
  if (SLACK_WEBHOOK_URL) {
    await fetch(SLACK_WEBHOOK_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: message }),
    });
  }
}

// ── Trigger feedback request ──────────────────────────────────────────
app.post('/send-feedback', async (req, res) => {
  const { phone, customer_name, service, mechanic, invoice_number } = req.body;

  if (!phone || !service) {
    return res.status(400).json({ error: 'Required: phone, service' });
  }

  try {
    // Set up feedback session
    const session = {
      stage: 'awaiting_rating',
      service,
      mechanic: mechanic || null,
      customer_name: customer_name || null,
      invoice_number: invoice_number || null,
      requested_at: new Date().toISOString(),
      rating: null,
      comment: null,
      analysis: null,
    };

    feedbackSessions.set(phone, session);

    // Send feedback request
    const greeting = customer_name ? `Hola ${customer_name}!` : 'Hola!';

    await sendText(phone,
      `${greeting} Gracias por confiar en Taller Mendoza para tu ${service}.\n\n` +
      `Nos gustaria saber como fue tu experiencia. Del 1 al 5, como calificarias nuestro servicio?\n\n` +
      `1 - Muy malo\n` +
      `2 - Malo\n` +
      `3 - Regular\n` +
      `4 - Bueno\n` +
      `5 - Excelente\n\n` +
      `Responde con un numero.`,
      {
        agent: 'customer-feedback',
        type: 'feedback_request',
        service,
        mechanic,
        invoice: invoice_number,
      }
    );

    // Update conversation metadata
    await updateConversation(phone, {
      feedback: {
        pending: true,
        service,
        mechanic,
        invoice_number,
        requested_at: session.requested_at,
      },
    });

    res.json({ success: true, message: 'Feedback request sent' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Webhook: handle customer responses ────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';

  if (!messageText.trim()) return;

  const session = feedbackSessions.get(phone);
  if (!session) return; // Not in a feedback flow

  try {
    await sendTyping(phone);

    if (session.stage === 'awaiting_rating') {
      const rating = parseRating(messageText);

      if (!rating) {
        await new Promise(r => setTimeout(r, 1000));
        await sendText(phone,
          'No entendi tu respuesta. Por favor, responde con un numero del 1 al 5.',
          { agent: 'customer-feedback', type: 'clarification' }
        );
        return;
      }

      session.rating = rating;
      session.stage = 'awaiting_comment';

      await new Promise(r => setTimeout(r, 1500));

      if (rating >= 4) {
        await sendText(phone,
          `Nos alegra saber eso! Que fue lo que mas te gusto del servicio? (o escribi "nada mas" si no queres agregar comentarios)`,
          { agent: 'customer-feedback', type: 'follow_up_positive', rating }
        );
      } else if (rating === 3) {
        await sendText(phone,
          `Gracias por tu honestidad. Que podriamos mejorar para que la proxima vez sea un 5? (o escribi "nada mas" si preferis no comentar)`,
          { agent: 'customer-feedback', type: 'follow_up_neutral', rating }
        );
      } else {
        await sendText(phone,
          `Lamentamos que tu experiencia no haya sido buena. Nos gustaria saber que paso para poder mejorar.`,
          { agent: 'customer-feedback', type: 'follow_up_negative', rating }
        );
      }
    } else if (session.stage === 'awaiting_comment') {
      const skipPhrases = /^(nada|no|nada mas|listo|ok|todo bien|eso es todo)$/i;
      const hasComment = !skipPhrases.test(messageText.trim());

      session.comment = hasComment ? messageText : null;
      session.stage = 'completed';
      session.completed_at = new Date().toISOString();

      // Analyze comment with AI if provided
      if (hasComment) {
        session.analysis = await analyzeComment(messageText, session.rating, session.service);
      }

      // Thank the customer
      await new Promise(r => setTimeout(r, 1500));

      if (session.rating >= 4) {
        await sendText(phone,
          `Muchas gracias por tu feedback! Si en el futuro necesitas algo, no dudes en escribirnos. Buen dia!`,
          {
            agent: 'customer-feedback',
            type: 'thank_you',
            rating: session.rating,
            has_comment: hasComment,
          }
        );
      } else {
        await sendText(phone,
          `Muchas gracias por contarnos. Vamos a trabajar para mejorar. Un encargado va a revisar tu comentario personalmente.`,
          {
            agent: 'customer-feedback',
            type: 'thank_you',
            rating: session.rating,
            has_comment: hasComment,
          }
        );
      }

      // Store complete feedback in conversation metadata
      const conversation = await getConversation(phone);
      const existingMeta = conversation?.metadata || {};
      const feedbackHistory = existingMeta.feedback_history || [];

      feedbackHistory.push({
        rating: session.rating,
        comment: session.comment,
        analysis: session.analysis,
        service: session.service,
        mechanic: session.mechanic,
        invoice_number: session.invoice_number,
        requested_at: session.requested_at,
        completed_at: session.completed_at,
      });

      await updateConversation(phone, {
        ...existingMeta,
        feedback: {
          pending: false,
          latest_rating: session.rating,
          latest_service: session.service,
          average_rating: feedbackHistory.reduce((sum, f) => sum + f.rating, 0) / feedbackHistory.length,
          total_feedbacks: feedbackHistory.length,
        },
        feedback_history: feedbackHistory.slice(-20),
      });

      // Alert on bad ratings
      if (session.rating <= 2) {
        await alertBadRating(phone, session.rating, session.comment, session.service, session.customer_name);
      }

      // Clean up session
      feedbackSessions.delete(phone);

      console.log(`[Feedback] ${phone}: ${session.rating}/5 for ${session.service}${hasComment ? ` - "${session.comment}"` : ''}`);
    }
  } catch (err) {
    console.error(`Error processing feedback from ${phone}:`, err.message);
  }
});

// ── Weekly report endpoint ────────────────────────────────────────────
app.get('/report', async (req, res) => {
  const { period } = req.query; // "week" or "month", defaults to week
  const daysBack = period === 'month' ? 30 : 7;
  const cutoff = new Date(Date.now() - daysBack * 86400000).toISOString();

  try {
    const conversations = await listConversations();

    const allFeedback = [];
    for (const conv of conversations) {
      const history = conv.metadata?.feedback_history || [];
      for (const fb of history) {
        if (fb.completed_at && fb.completed_at >= cutoff) {
          allFeedback.push({
            ...fb,
            phone: conv.contact_phone,
            name: conv.contact_name,
          });
        }
      }
    }

    if (allFeedback.length === 0) {
      return res.json({ message: 'No feedback collected in this period', period: daysBack + ' days' });
    }

    // Aggregate stats
    const avgRating = allFeedback.reduce((sum, f) => sum + f.rating, 0) / allFeedback.length;
    const ratingDistribution = { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 };
    allFeedback.forEach(f => ratingDistribution[f.rating]++);

    // By service
    const byService = {};
    for (const fb of allFeedback) {
      if (!byService[fb.service]) {
        byService[fb.service] = { total: 0, sum: 0, comments: [] };
      }
      byService[fb.service].total++;
      byService[fb.service].sum += fb.rating;
      if (fb.comment) byService[fb.service].comments.push(fb.comment);
    }

    for (const [service, data] of Object.entries(byService)) {
      byService[service].average = Math.round((data.sum / data.total) * 10) / 10;
    }

    // By mechanic
    const byMechanic = {};
    for (const fb of allFeedback) {
      if (!fb.mechanic) continue;
      if (!byMechanic[fb.mechanic]) {
        byMechanic[fb.mechanic] = { total: 0, sum: 0 };
      }
      byMechanic[fb.mechanic].total++;
      byMechanic[fb.mechanic].sum += fb.rating;
    }

    for (const [mechanic, data] of Object.entries(byMechanic)) {
      byMechanic[mechanic].average = Math.round((data.sum / data.total) * 10) / 10;
    }

    // Common themes (from AI analysis)
    const themes = {};
    for (const fb of allFeedback) {
      for (const theme of (fb.analysis?.themes || [])) {
        themes[theme] = (themes[theme] || 0) + 1;
      }
    }

    // Flagged issues
    const flagged = allFeedback
      .filter(f => f.rating <= 2 || f.analysis?.requires_followup)
      .map(f => ({
        phone: f.phone,
        name: f.name,
        rating: f.rating,
        comment: f.comment,
        service: f.service,
        date: f.completed_at,
      }));

    res.json({
      period: `Last ${daysBack} days`,
      total_responses: allFeedback.length,
      average_rating: Math.round(avgRating * 10) / 10,
      nps_estimate: Math.round(((ratingDistribution[5] + ratingDistribution[4]) / allFeedback.length - ratingDistribution[1] / allFeedback.length) * 100),
      rating_distribution: ratingDistribution,
      by_service: byService,
      by_mechanic: byMechanic,
      common_themes: Object.entries(themes).sort((a, b) => b[1] - a[1]),
      flagged_issues: flagged,
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Batch send feedback requests ──────────────────────────────────────
app.post('/send-feedback/batch', async (req, res) => {
  const { customers } = req.body; // Array of { phone, customer_name, service, mechanic, invoice_number }

  if (!customers?.length) {
    return res.status(400).json({ error: 'Required: customers[]' });
  }

  const results = [];
  for (const customer of customers) {
    try {
      const response = await fetch(`http://localhost:${PORT}/send-feedback`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(customer),
      });
      const result = await response.json();
      results.push({ phone: customer.phone, ...result });
    } catch (err) {
      results.push({ phone: customer.phone, error: err.message });
    }
    // Delay between sends
    await new Promise(r => setTimeout(r, 3000));
  }

  res.json({ results });
});

app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    active_feedback_sessions: feedbackSessions.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3008;
app.listen(PORT, () => {
  console.log(`Customer feedback bot listening on port ${PORT}`);
});
```

## Metadata Schema

### Conversation-level metadata

```json
{
  "feedback": {
    "pending": false,
    "latest_rating": 4,
    "latest_service": "cambio de aceite y filtro",
    "average_rating": 4.3,
    "total_feedbacks": 3
  },
  "feedback_history": [
    {
      "rating": 5,
      "comment": "Excelente atencion, muy rapido y me explicaron todo",
      "analysis": {
        "sentiment": "positive",
        "themes": ["staff", "communication", "wait_time"],
        "actionable": false,
        "summary": "Very satisfied with speed and communication",
        "requires_followup": false
      },
      "service": "revision completa",
      "mechanic": "Jorge",
      "invoice_number": "INV-2026-0341",
      "requested_at": "2026-03-15T17:00:00.000Z",
      "completed_at": "2026-03-15T17:05:00.000Z"
    },
    {
      "rating": 2,
      "comment": "Esperamos 3 horas cuando me dijeron que tardaba 1",
      "analysis": {
        "sentiment": "negative",
        "themes": ["wait_time", "communication"],
        "actionable": true,
        "summary": "Long wait time, poor time estimate communication",
        "requires_followup": true
      },
      "service": "cambio de frenos",
      "mechanic": "Raul",
      "invoice_number": "INV-2026-0412",
      "requested_at": "2026-04-01T18:00:00.000Z",
      "completed_at": "2026-04-01T18:08:00.000Z"
    }
  ]
}
```

### Message-level metadata

```json
{
  "agent": "customer-feedback",
  "type": "feedback_request",
  "service": "cambio de aceite y filtro",
  "mechanic": "Jorge",
  "invoice": "INV-2026-0341"
}
```

## How to Run

```bash
# 1. Create the project
mkdir customer-feedback && cd customer-feedback
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export MANAGER_PHONE="5492615551234"  # Manager's WhatsApp for bad rating alerts

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3008

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. Trigger a feedback request (call from your POS/backend)
curl -X POST http://localhost:3008/send-feedback \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5492615551234",
    "customer_name": "Juan",
    "service": "cambio de aceite y filtro",
    "mechanic": "Jorge",
    "invoice_number": "INV-2026-0501"
  }'

# 7. View weekly report
curl "http://localhost:3008/report?period=week"
```
