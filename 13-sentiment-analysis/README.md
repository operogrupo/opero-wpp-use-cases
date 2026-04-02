# 13 — Real-Time Sentiment Analysis

## Problem

You receive hundreds of WhatsApp messages daily but have no visibility into customer mood. A frustrated customer might churn before anyone notices. You need automated sentiment tracking to catch negative trends early and route unhappy customers to human agents.

## Solution

Classify every inbound message with GPT-4o-mini for sentiment (positive/neutral/negative) and emotion (happy/angry/confused/urgent). Tag each message with the result. Track streaks of negative sentiment and alert your team when a customer needs attention.

## Architecture

```
Inbound WhatsApp Message
    |
    v
┌──────────────────────────────┐
│   Webhook Server             │
│                              │
│   1. Receive message         │
│   2. Classify with           │
│      GPT-4o-mini             │
│      → sentiment + emotion   │
│   3. Tag message metadata    │
│   4. Update conversation     │
│      sentiment history       │
│   5. Check for negative      │
│      streak (3+ in a row)    │
│   6. Alert team if needed    │
└──────────────────────────────┘
    |
    ├── Tag message: { sentiment: "negative", emotion: "angry" }
    |
    ├── Update conversation: { sentiment_history: [...], streak: -3 }
    |
    └── If streak <= -3: Alert via WhatsApp to support number
```

## Metadata Schema

**Message metadata** (per-message tagging):
```json
{
  "sentiment": "positive | neutral | negative",
  "emotion": "happy | satisfied | neutral | confused | frustrated | angry | urgent",
  "confidence": 0.92,
  "analyzed_at": "2026-04-02T10:00:00Z"
}
```

**Conversation metadata** (aggregate tracking):
```json
{
  "sentiment": {
    "current": "negative",
    "current_emotion": "frustrated",
    "streak": -3,
    "history": [
      { "sentiment": "neutral", "emotion": "neutral", "at": "2026-04-02T09:50:00Z" },
      { "sentiment": "negative", "emotion": "confused", "at": "2026-04-02T09:55:00Z" },
      { "sentiment": "negative", "emotion": "frustrated", "at": "2026-04-02T10:00:00Z" },
      { "sentiment": "negative", "emotion": "angry", "at": "2026-04-02T10:05:00Z" }
    ],
    "positive_count": 5,
    "neutral_count": 8,
    "negative_count": 4,
    "avg_score": 0.15,
    "alert_sent": true,
    "alert_sent_at": "2026-04-02T10:05:00Z"
  }
}
```

## Code

```javascript
// sentiment-analysis.js
const express = require("express");
const OpenAI = require("openai");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const ALERT_PHONE = process.env.ALERT_PHONE; // Support team phone number

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const NEGATIVE_STREAK_THRESHOLD = 3; // Alert after 3 consecutive negative messages
const MAX_HISTORY_LENGTH = 50; // Keep last 50 sentiment entries

// ── API Helpers ────────────────────────────────────────────────────────

async function apiRequest(method, path, body = null) {
  const options = {
    method,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
    },
  };
  if (body) options.body = JSON.stringify(body);

  const res = await fetch(`${API_BASE}${path}`, options);
  return res.json();
}

async function getConversation(phone) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}`);
}

async function updateConversation(phone, metadata) {
  return apiRequest("PUT", `/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    metadata,
  });
}

async function tagMessage(phone, messageId, metadata) {
  return apiRequest(
    "PATCH",
    `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages/${messageId}`,
    { metadata }
  );
}

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, {
    phone,
    text,
    metadata,
  });
}

// ── Sentiment Classification ───────────────────────────────────────────

async function classifySentiment(messageText) {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    max_tokens: 150,
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `Classify the sentiment and emotion of this WhatsApp message. Return JSON only:

{
  "sentiment": "positive" | "neutral" | "negative",
  "emotion": "happy" | "satisfied" | "neutral" | "confused" | "frustrated" | "angry" | "urgent",
  "confidence": 0.0 to 1.0,
  "reasoning": "brief explanation"
}

Rules:
- "urgent" is an emotion, not a sentiment (urgent messages can be neutral/negative)
- Consider context: short replies like "ok" are neutral, not positive
- Sarcasm should be classified by intent, not surface meaning
- Messages with questions are usually neutral unless clearly frustrated`,
      },
      {
        role: "user",
        content: messageText,
      },
    ],
  });

  return JSON.parse(response.choices[0].message.content);
}

// ── Streak Calculation ─────────────────────────────────────────────────

function sentimentToScore(sentiment) {
  const scores = { positive: 1, neutral: 0, negative: -1 };
  return scores[sentiment] ?? 0;
}

function calculateStreak(history) {
  if (history.length === 0) return 0;

  let streak = 0;
  const lastSentiment = history[history.length - 1].sentiment;

  for (let i = history.length - 1; i >= 0; i--) {
    if (history[i].sentiment === lastSentiment) {
      streak++;
    } else {
      break;
    }
  }

  // Return negative streak as negative number
  return lastSentiment === "negative" ? -streak : streak;
}

function calculateAvgScore(history) {
  if (history.length === 0) return 0;
  const total = history.reduce((sum, h) => sum + sentimentToScore(h.sentiment), 0);
  return Math.round((total / history.length) * 100) / 100;
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received") return;
  if (data.type !== "text") return;

  const phone = data.from;
  const messageId = data.id;
  const messageText =
    typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    // 1. Classify sentiment
    const classification = await classifySentiment(messageText);

    // 2. Tag the message
    await tagMessage(phone, messageId, {
      sentiment: classification.sentiment,
      emotion: classification.emotion,
      confidence: classification.confidence,
      analyzed_at: new Date().toISOString(),
    });

    // 3. Update conversation sentiment history
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const sentimentData = metadata.sentiment || {
      history: [],
      positive_count: 0,
      neutral_count: 0,
      negative_count: 0,
      alert_sent: false,
    };

    // Add to history
    sentimentData.history.push({
      sentiment: classification.sentiment,
      emotion: classification.emotion,
      at: new Date().toISOString(),
    });

    // Trim history if too long
    if (sentimentData.history.length > MAX_HISTORY_LENGTH) {
      sentimentData.history = sentimentData.history.slice(-MAX_HISTORY_LENGTH);
    }

    // Update counters
    sentimentData[`${classification.sentiment}_count`] =
      (sentimentData[`${classification.sentiment}_count`] || 0) + 1;

    // Update current state
    sentimentData.current = classification.sentiment;
    sentimentData.current_emotion = classification.emotion;
    sentimentData.streak = calculateStreak(sentimentData.history);
    sentimentData.avg_score = calculateAvgScore(sentimentData.history);

    // 4. Check for negative streak alert
    if (
      sentimentData.streak <= -NEGATIVE_STREAK_THRESHOLD &&
      !sentimentData.alert_sent
    ) {
      // Send alert to support team
      if (ALERT_PHONE) {
        const alertMsg = [
          `ALERT: Negative sentiment streak detected`,
          ``,
          `Customer: ${phone}`,
          `Streak: ${Math.abs(sentimentData.streak)} negative messages in a row`,
          `Current emotion: ${classification.emotion}`,
          `Overall score: ${sentimentData.avg_score}`,
          ``,
          `Please review this conversation and consider reaching out.`,
        ].join("\n");

        await sendText(ALERT_PHONE, alertMsg, {
          alert_type: "negative_sentiment_streak",
          customer_phone: phone,
          streak: sentimentData.streak,
        });
      }

      sentimentData.alert_sent = true;
      sentimentData.alert_sent_at = new Date().toISOString();

      console.log(
        `ALERT: Negative streak of ${Math.abs(sentimentData.streak)} for ${phone}`
      );
    }

    // Reset alert flag if sentiment improves
    if (sentimentData.streak > 0 && sentimentData.alert_sent) {
      sentimentData.alert_sent = false;
    }

    // 5. Save updated sentiment data
    await updateConversation(phone, {
      ...metadata,
      sentiment: sentimentData,
    });

    console.log(
      `[${phone}] sentiment=${classification.sentiment} emotion=${classification.emotion} streak=${sentimentData.streak}`
    );
  } catch (err) {
    console.error("Sentiment analysis error:", err);
  }
});

// ── Dashboard endpoint (optional) ─────────────────────────────────────

app.get("/sentiment/:phone", async (req, res) => {
  try {
    const convoRes = await getConversation(req.params.phone);
    const sentiment = convoRes.data?.metadata?.sentiment || {};
    res.json({ phone: req.params.phone, sentiment });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Sentiment analysis running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express openai

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export OPENAI_API_KEY="your-openai-key"
export ALERT_PHONE="5491155551234"  # Support team WhatsApp number

# 3. Start the server
node sentiment-analysis.js

# 4. Expose via ngrok
ngrok http 3000

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'
```

## Key Concepts

- **Per-message tagging**: Every message gets its own sentiment/emotion metadata via `PATCH /messages/{msgId}`
- **Conversation-level aggregation**: Running counts, averages, and streaks stored in conversation metadata
- **Streak detection**: Consecutive negative messages trigger alerts — catches escalating frustration
- **Alert reset**: Alerts clear when sentiment turns positive, so repeat alerts only fire for new incidents
- **Low cost**: GPT-4o-mini is cheap enough to classify every single message in real time
