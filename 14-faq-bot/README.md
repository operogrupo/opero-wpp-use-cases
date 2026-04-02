# 14 — FAQ Bot with Context

## Problem

Customers ask the same questions repeatedly. A static FAQ page doesn't work because people ask in different ways, follow up on answers, and need conversational context. When the bot can't answer, there's no smooth handoff to a human.

## Solution

A context-aware FAQ bot powered by Claude Haiku. The knowledge base is embedded in the system prompt so there's no vector database needed. The bot uses conversation history to handle follow-up questions naturally, tags each answer with the matched FAQ topic, and escalates to a human when confidence is low.

## Architecture

```
WhatsApp Message
    |
    v
┌──────────────────────────────────┐
│   Webhook Server                 │
│                                  │
│   1. Receive message             │
│   2. Check if escalated to human │
│      (metadata.escalated = true) │
│      → skip AI, notify agent     │
│   3. Load conversation history   │
│   4. Send to Claude Haiku with   │
│      FAQ knowledge base          │
│   5. Parse response:             │
│      - matched_topic             │
│      - confidence (0-1)          │
│      - answer                    │
│   6. If confidence < 0.5         │
│      → escalate to human         │
│   7. Tag message with topic      │
│   8. Send reply                  │
└──────────────────────────────────┘
```

## Metadata Schema

**Message metadata**:
```json
{
  "faq_topic": "pricing",
  "faq_confidence": 0.95,
  "escalated": false
}
```

**Conversation metadata**:
```json
{
  "topics_asked": ["pricing", "refunds", "shipping"],
  "message_count": 12,
  "escalated": false,
  "escalated_at": null,
  "escalation_reason": "low_confidence",
  "assigned_agent": null,
  "resolved": false
}
```

## Code

```javascript
// faq-bot.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const SUPPORT_PHONE = process.env.SUPPORT_PHONE; // Human agent phone

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const CONFIDENCE_THRESHOLD = 0.5;

// ── Knowledge Base ─────────────────────────────────────────────────────
// Edit this to match your business. No external DB needed.

const KNOWLEDGE_BASE = `
## Company: TechCo SaaS Platform

### Pricing
- Starter: $29/month — 3 users, 10GB storage, email support
- Pro: $79/month — 15 users, 100GB storage, priority support, API access
- Enterprise: $249/month — unlimited users, 1TB storage, dedicated account manager, SLA
- All plans include a 14-day free trial, no credit card required
- Annual billing gives 20% discount
- Students get 50% off any plan with valid .edu email

### Refunds
- Full refund within 30 days of purchase, no questions asked
- Refunds processed within 5-7 business days
- Annual plans can be refunded pro-rata after 30 days
- Contact billing@techco.com for refund requests

### Shipping / Delivery
- This is a digital product, no physical shipping
- Account access is instant after signup
- Data exports are available within 24 hours of request

### Account Management
- Change plan anytime from Settings > Billing
- Downgrade takes effect at next billing cycle
- Upgrade is immediate with pro-rated charges
- Cancel anytime, keep access until end of billing period
- Data is retained for 90 days after cancellation

### Technical Support
- Email: support@techco.com (24h response time)
- Chat: available Mon-Fri 9am-6pm EST
- Phone: Pro and Enterprise plans only
- Status page: status.techco.com

### Integrations
- Native integrations: Slack, Google Workspace, Salesforce, HubSpot
- API available on Pro and Enterprise plans
- Zapier integration for 1000+ apps
- Webhook support for custom integrations

### Security
- SOC 2 Type II certified
- Data encrypted at rest and in transit
- 2FA available for all accounts
- SSO available on Enterprise plan
- GDPR compliant, data stored in US/EU (your choice)

### Getting Started
- Sign up at app.techco.com
- Follow the onboarding wizard (5 minutes)
- Import data from CSV, or connect existing tools
- Join our weekly webinar for a live demo (Tuesdays 2pm EST)
`;

const SYSTEM_PROMPT = `You are the FAQ assistant for TechCo. Answer customer questions using ONLY the knowledge base below. Be friendly, concise, and helpful.

${KNOWLEDGE_BASE}

IMPORTANT RULES:
1. ONLY answer based on the knowledge base above
2. If the question is not covered or you're unsure, say so honestly
3. Keep answers short — 2-4 sentences max unless the question needs more detail
4. For follow-up questions, use conversation context to give relevant answers
5. Never make up information not in the knowledge base

You MUST end every response with a JSON block on its own line:
{"topic": "the_matched_faq_topic", "confidence": 0.0_to_1.0}

Topic must be one of: pricing, refunds, shipping, account, support, integrations, security, getting_started, general, unknown
Confidence: 1.0 = perfect match, 0.5 = partial, 0.0 = no match at all`;

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

async function getMessages(phone, limit = 20) {
  return apiRequest(
    "GET",
    `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`
  );
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

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, {
    phone,
    type: "composing",
  });
}

// ── Response Parsing ───────────────────────────────────────────────────

function parseResponse(text) {
  // Extract JSON metadata from end of response
  const jsonMatch = text.match(/\{[\s]*"topic"[\s]*:[\s]*"[^"]*"[\s]*,[\s]*"confidence"[\s]*:[\s]*[\d.]+[\s]*\}/);

  if (jsonMatch) {
    try {
      const meta = JSON.parse(jsonMatch[0]);
      const cleanText = text.replace(jsonMatch[0], "").trim();
      return { answer: cleanText, topic: meta.topic, confidence: meta.confidence };
    } catch {
      // Fall through
    }
  }

  return { answer: text.trim(), topic: "unknown", confidence: 0.3 };
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received") return;
  if (data.type !== "text") return;

  const phone = data.from;
  const messageText =
    typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    // 1. Check if conversation is escalated to human
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};

    if (metadata.escalated) {
      // Notify the human agent about the new message
      if (SUPPORT_PHONE) {
        await sendText(
          SUPPORT_PHONE,
          `[Escalated customer ${phone}]: ${messageText}`,
          { alert_type: "escalated_message", customer_phone: phone }
        );
      }
      console.log(`[${phone}] Escalated — forwarding to human agent`);
      return;
    }

    await sendTyping(phone);

    // 2. Build conversation history
    const messagesRes = await getMessages(phone, 20);
    const messages = (messagesRes.data || [])
      .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
      .map((msg) => ({
        role: msg.direction === "inbound" ? "user" : "assistant",
        content:
          typeof msg.content === "string"
            ? msg.content
            : msg.content?.text || JSON.stringify(msg.content),
      }));

    // 3. Call Claude Haiku
    const response = await anthropic.messages.create({
      model: "claude-haiku-4-20250514",
      max_tokens: 512,
      system: SYSTEM_PROMPT,
      messages,
    });

    const rawResponse = response.content[0].text;
    const { answer, topic, confidence } = parseResponse(rawResponse);

    // 4. Tag the inbound message
    await tagMessage(phone, data.id, {
      faq_topic: topic,
      faq_confidence: confidence,
      escalated: false,
    });

    // 5. Update conversation metadata
    const topicsAsked = metadata.topics_asked || [];
    if (topic !== "unknown" && !topicsAsked.includes(topic)) {
      topicsAsked.push(topic);
    }

    const updatedMetadata = {
      ...metadata,
      topics_asked: topicsAsked,
      message_count: (metadata.message_count || 0) + 1,
    };

    // 6. Check confidence — escalate if too low
    if (confidence < CONFIDENCE_THRESHOLD) {
      updatedMetadata.escalated = true;
      updatedMetadata.escalated_at = new Date().toISOString();
      updatedMetadata.escalation_reason = "low_confidence";

      await updateConversation(phone, updatedMetadata);

      const escalationMsg =
        "I'm not fully confident I can answer that correctly. Let me connect you with a team member who can help. They'll be with you shortly!";

      await sendText(phone, escalationMsg, {
        faq_topic: topic,
        faq_confidence: confidence,
        escalated: true,
      });

      // Notify support
      if (SUPPORT_PHONE) {
        await sendText(
          SUPPORT_PHONE,
          [
            `New escalation from ${phone}`,
            `Question: ${messageText}`,
            `Closest topic: ${topic} (confidence: ${confidence})`,
            `Topics discussed: ${topicsAsked.join(", ") || "none"}`,
          ].join("\n"),
          { alert_type: "escalation", customer_phone: phone }
        );
      }

      console.log(`[${phone}] Escalated — confidence ${confidence} below threshold`);
      return;
    }

    // 7. Send the answer
    await updateConversation(phone, updatedMetadata);
    await sendText(phone, answer, {
      faq_topic: topic,
      faq_confidence: confidence,
    });

    console.log(`[${phone}] topic=${topic} confidence=${confidence}`);
  } catch (err) {
    console.error("FAQ bot error:", err);
  }
});

// ── De-escalation endpoint ─────────────────────────────────────────────
// Call this when a human agent resolves the issue

app.post("/resolve/:phone", async (req, res) => {
  try {
    const convoRes = await getConversation(req.params.phone);
    const metadata = convoRes.data?.metadata || {};

    metadata.escalated = false;
    metadata.resolved = true;
    metadata.resolved_at = new Date().toISOString();

    await updateConversation(req.params.phone, metadata);
    res.json({ success: true, message: "Conversation de-escalated" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("FAQ bot running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-key"
export SUPPORT_PHONE="5491155551234"  # Human agent phone

# 3. Start the server
node faq-bot.js

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

- **Knowledge base in system prompt**: No vector DB needed. Edit the `KNOWLEDGE_BASE` constant to update answers
- **Follow-up handling**: Full conversation history is sent to Claude, so "what about the pro plan?" works after asking about pricing
- **Confidence-based escalation**: The model self-reports confidence. Below 0.5 = handoff to human
- **Topic tracking**: Each message is tagged with the matched FAQ topic for analytics
- **Human-in-the-loop**: Escalated conversations forward new messages to the support agent. Call `/resolve/:phone` to return to bot mode
