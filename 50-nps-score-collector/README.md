# 50 - NPS Score Collector

Net Promoter Score collection via WhatsApp. Sends "How likely are you to recommend us?" after service interactions, classifies detractors/passives/promoters, follows up with open-ended feedback, and calculates running NPS in metadata.

## Problem

A SaaS company wants to measure customer satisfaction but email NPS surveys get a 15% response rate. Phone surveys are expensive and biased toward available customers. The product team makes decisions without reliable customer sentiment data. By the time they detect a trend, they've already lost customers.

## Solution

Deploy an NPS collector that:
- Sends a "0-10" rating question via WhatsApp after service interactions
- Classifies responses as detractor (0-6), passive (7-8), or promoter (9-10)
- Follows up with an open-ended question for qualitative feedback
- Uses Claude Haiku to summarize and categorize open-ended feedback
- Calculates running NPS per customer segment in metadata
- Provides real-time NPS dashboard via API

## Architecture

```
+-------------------+     trigger NPS          +--------------------+
|  Your App/CRM     | -----------------------> |  Your Express App  |
| (after ticket,    |                          |   (this code)      |
|  purchase, etc.)  |                          +--------------------+
+-------------------+                                   |
                                                        v
                                               +------------------+
                                               |   Opero WPP API  |
                                               | wpp-api.opero.so |
                                               +------------------+
                                                        |
                                          NPS question  |
                                                        v
                                               +------------------+
                                               |    Customer      |
                                               |   (WhatsApp)     |
                                               +------------------+
                                                        |
                                          score + feedback
                                                        v
                                               +------------------+
                                               | Claude Haiku     |
                                               | (categorize      |
                                               |  feedback)       |
                                               +------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

// -- Config --
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// -- Global NPS state (use Redis/DB in production) --
const npsGlobal = {
  total_responses: 0,
  promoters: 0,
  passives: 0,
  detractors: 0,
  scores: [],
  categories: {},
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

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
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

// -- NPS classification --
function classifyScore(score) {
  if (score >= 9) return 'promoter';
  if (score >= 7) return 'passive';
  return 'detractor';
}

function calculateNPS(promoters, passives, detractors) {
  const total = promoters + passives + detractors;
  if (total === 0) return 0;
  return Math.round(((promoters - detractors) / total) * 100);
}

// -- Feedback categorization with AI --
async function categorizeFeedback(feedback, score) {
  try {
    const response = await anthropic.messages.create({
      model: 'claude-haiku-4-20250414',
      max_tokens: 200,
      system: 'You categorize NPS feedback into one or two categories. Respond with JSON only: {"categories": ["category1"], "sentiment": "positive|negative|neutral", "summary": "one sentence summary"}. Categories: product_quality, customer_service, pricing, ease_of_use, reliability, speed, features, documentation, onboarding, other.',
      messages: [{
        role: 'user',
        content: `NPS score: ${score}/10\nFeedback: "${feedback}"`,
      }],
    });

    const text = response.content[0].text;
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    return jsonMatch ? JSON.parse(jsonMatch[0]) : { categories: ['other'], sentiment: 'neutral', summary: feedback.substring(0, 100) };
  } catch {
    return { categories: ['other'], sentiment: 'neutral', summary: feedback.substring(0, 100) };
  }
}

// -- Send NPS survey --
async function sendNPSSurvey(phone, context = {}) {
  const metadata = await getConversationMetadata(phone);

  // Don't survey if already surveyed recently (within 30 days)
  if (metadata.last_survey_sent) {
    const daysSinceLast = (Date.now() - new Date(metadata.last_survey_sent).getTime()) / (1000 * 60 * 60 * 24);
    if (daysSinceLast < 30) {
      return { skipped: true, reason: 'Surveyed within last 30 days' };
    }
  }

  // Initialize NPS data
  if (!metadata.nps_history) metadata.nps_history = [];

  metadata.nps_state = 'awaiting_score';
  metadata.nps_context = {
    trigger: context.trigger || 'manual',
    trigger_id: context.trigger_id || null,
    sent_at: new Date().toISOString(),
  };
  metadata.last_survey_sent = new Date().toISOString();

  await updateConversationMetadata(phone, metadata);

  const greeting = metadata.customer_name ? `Hi ${metadata.customer_name}! ` : '';
  const contextMsg = context.trigger === 'ticket_closed'
    ? 'Now that your support ticket has been resolved, we\'d love your feedback.'
    : context.trigger === 'purchase'
    ? 'Thank you for your recent purchase!'
    : 'We value your opinion.';

  await sendText(phone, [
    `${greeting}${contextMsg}`,
    ``,
    `On a scale of 0-10, how likely are you to recommend us to a friend or colleague?`,
    ``,
    `Just reply with a number from 0 to 10.`,
  ].join('\n'), {
    agent: 'nps-collector',
    type: 'survey-sent',
    trigger: context.trigger || 'manual',
  });

  return { sent: true, phone };
}

// -- Process score --
async function processScore(phone, score, metadata) {
  const category = classifyScore(score);
  const now = new Date().toISOString();

  metadata.nps_state = 'awaiting_feedback';
  metadata.nps_context.score = score;
  metadata.nps_context.category = category;
  metadata.nps_context.scored_at = now;

  await updateConversationMetadata(phone, metadata);

  // Update global stats
  npsGlobal.total_responses++;
  npsGlobal.scores.push(score);
  if (category === 'promoter') npsGlobal.promoters++;
  else if (category === 'passive') npsGlobal.passives++;
  else npsGlobal.detractors++;

  // Send follow-up question based on score
  let followUp;
  if (score >= 9) {
    followUp = `Thank you for the ${score}! We're glad you love us.\n\nWhat's the main reason for your score? (Reply with a short message, or *SKIP* to finish)`;
  } else if (score >= 7) {
    followUp = `Thanks for the ${score}! We appreciate your feedback.\n\nWhat could we do to earn a higher score? (Reply with a short message, or *SKIP* to finish)`;
  } else {
    followUp = `Thank you for being honest with the ${score}. We take this seriously.\n\nCould you share what we could improve? Your feedback directly reaches our team. (Reply with a short message, or *SKIP* to finish)`;
  }

  await sendText(phone, followUp, {
    agent: 'nps-collector',
    type: 'follow-up-sent',
    score,
    category,
  });
}

// -- Process feedback --
async function processFeedback(phone, feedback, metadata) {
  const now = new Date().toISOString();
  const score = metadata.nps_context.score;
  const category = metadata.nps_context.category;

  // Categorize feedback with AI
  await sendTyping(phone);
  const analysis = await categorizeFeedback(feedback, score);

  // Record in history
  const entry = {
    score,
    category,
    feedback: feedback === 'SKIP' ? null : feedback,
    analysis: feedback === 'SKIP' ? null : analysis,
    trigger: metadata.nps_context.trigger,
    trigger_id: metadata.nps_context.trigger_id,
    sent_at: metadata.nps_context.sent_at,
    scored_at: metadata.nps_context.scored_at,
    completed_at: now,
  };

  metadata.nps_history.push(entry);
  metadata.nps_state = 'complete';
  metadata.latest_nps_score = score;
  metadata.latest_nps_category = category;

  // Calculate customer's personal NPS trend
  const scores = metadata.nps_history.map(h => h.score);
  metadata.average_nps_score = Math.round(scores.reduce((a, b) => a + b, 0) / scores.length * 10) / 10;
  metadata.nps_trend = scores.length >= 2
    ? (scores[scores.length - 1] > scores[scores.length - 2] ? 'improving' : scores[scores.length - 1] < scores[scores.length - 2] ? 'declining' : 'stable')
    : 'new';

  delete metadata.nps_context;
  await updateConversationMetadata(phone, metadata);

  // Update global categories
  if (analysis?.categories) {
    for (const cat of analysis.categories) {
      if (!npsGlobal.categories[cat]) {
        npsGlobal.categories[cat] = { count: 0, avg_score: 0, scores: [] };
      }
      npsGlobal.categories[cat].count++;
      npsGlobal.categories[cat].scores.push(score);
      npsGlobal.categories[cat].avg_score =
        npsGlobal.categories[cat].scores.reduce((a, b) => a + b, 0) / npsGlobal.categories[cat].scores.length;
    }
  }

  // Thank you message
  const thankYou = score <= 6
    ? 'Thank you for your feedback. We\'re committed to improving your experience. A team member may reach out to discuss further.'
    : 'Thank you for your valuable feedback! It helps us keep improving.';

  await sendText(phone, thankYou, {
    agent: 'nps-collector',
    type: 'survey-complete',
    score,
    category,
    feedback_categories: analysis?.categories || [],
  });

  console.log(`NPS collected: ${phone} scored ${score} (${category})${feedback !== 'SKIP' ? ` - "${feedback.substring(0, 50)}..."` : ''}`);
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim();
  if (!messageText) return;

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    // Handle score input
    if (metadata.nps_state === 'awaiting_score') {
      const score = parseInt(messageText);

      if (isNaN(score) || score < 0 || score > 10) {
        await sendText(phone, 'Please reply with a number between 0 and 10.', {
          agent: 'nps-collector', type: 'invalid-score',
        });
        return;
      }

      await processScore(phone, score, metadata);
      return;
    }

    // Handle feedback input
    if (metadata.nps_state === 'awaiting_feedback') {
      const feedback = messageText.toUpperCase() === 'SKIP' ? 'SKIP' : messageText;
      await processFeedback(phone, feedback, metadata);
      return;
    }

    // No active survey
    // (ignore or respond based on your needs)
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- API: trigger NPS survey --
app.post('/api/survey', async (req, res) => {
  try {
    const { phone, trigger, trigger_id } = req.body;
    const result = await sendNPSSurvey(phone, { trigger, trigger_id });
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- API: bulk send surveys --
app.post('/api/survey/bulk', async (req, res) => {
  try {
    const { phones, trigger } = req.body;
    const results = [];
    for (const phone of phones) {
      // Stagger sends to avoid rate limits
      await new Promise(resolve => setTimeout(resolve, 1000));
      const result = await sendNPSSurvey(phone, { trigger });
      results.push({ phone, ...result });
    }
    res.json({ results });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- API: NPS dashboard data --
app.get('/api/nps', (req, res) => {
  const nps = calculateNPS(npsGlobal.promoters, npsGlobal.passives, npsGlobal.detractors);
  const avgScore = npsGlobal.scores.length > 0
    ? (npsGlobal.scores.reduce((a, b) => a + b, 0) / npsGlobal.scores.length).toFixed(1)
    : 0;

  res.json({
    nps_score: nps,
    average_score: parseFloat(avgScore),
    total_responses: npsGlobal.total_responses,
    breakdown: {
      promoters: npsGlobal.promoters,
      passives: npsGlobal.passives,
      detractors: npsGlobal.detractors,
    },
    categories: npsGlobal.categories,
  });
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    nps: calculateNPS(npsGlobal.promoters, npsGlobal.passives, npsGlobal.detractors),
    total_responses: npsGlobal.total_responses,
    uptime: process.uptime(),
  });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`NPS collector listening on port ${PORT}`);
});
```

## Metadata Schema

Each customer conversation tracks their NPS history:

```json
{
  "customer_name": "Laura Fernandez",
  "nps_state": "complete",
  "latest_nps_score": 9,
  "latest_nps_category": "promoter",
  "average_nps_score": 8.5,
  "nps_trend": "improving",
  "last_survey_sent": "2026-04-02T10:00:00.000Z",
  "nps_history": [
    {
      "score": 8,
      "category": "passive",
      "feedback": "Good product but onboarding was confusing",
      "analysis": {
        "categories": ["onboarding", "ease_of_use"],
        "sentiment": "neutral",
        "summary": "Product is good but onboarding needs improvement"
      },
      "trigger": "ticket_closed",
      "trigger_id": "TICKET-1234",
      "sent_at": "2026-03-01T10:00:00.000Z",
      "scored_at": "2026-03-01T10:05:00.000Z",
      "completed_at": "2026-03-01T10:08:00.000Z"
    },
    {
      "score": 9,
      "category": "promoter",
      "feedback": "Love the new features! Dashboard is great.",
      "analysis": {
        "categories": ["features", "ease_of_use"],
        "sentiment": "positive",
        "summary": "Customer loves new features and dashboard"
      },
      "trigger": "purchase",
      "sent_at": "2026-04-02T10:00:00.000Z",
      "scored_at": "2026-04-02T10:03:00.000Z",
      "completed_at": "2026-04-02T10:06:00.000Z"
    }
  ]
}
```

Outbound message metadata:

```json
{
  "agent": "nps-collector",
  "type": "survey-complete",
  "score": 9,
  "category": "promoter",
  "feedback_categories": ["features", "ease_of_use"]
}
```

## How to Run

```bash
# 1. Create the project
mkdir nps-collector && cd nps-collector
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

# 6. Trigger an NPS survey
curl -X POST http://localhost:3000/api/survey \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "trigger": "ticket_closed",
    "trigger_id": "TICKET-5678"
  }'

# 7. Check NPS dashboard
curl http://localhost:3000/api/nps
```
