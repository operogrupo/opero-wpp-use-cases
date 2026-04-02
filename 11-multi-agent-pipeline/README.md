# 11 — Multi-Agent Sales Pipeline

## Problem

A single AI model cannot handle every stage of a sales conversation equally well. Qualification needs speed, presentation needs depth, negotiation needs reasoning, and closing needs persuasion. Using one model means overpaying for simple tasks or underperforming on complex ones.

## Solution

Route conversations through 4 specialized AI agents based on `metadata.stage`. Each agent is powered by the optimal model for its job:

| Stage | Agent | Model | Job |
|-------|-------|-------|-----|
| `qualify` | Qualifier | Claude Haiku | Fast screening — budget, timeline, fit |
| `present` | Presenter | Claude Sonnet | Product presentation with rich detail |
| `negotiate` | Negotiator | GPT-4 | Handle objections, pricing discussions |
| `close` | Closer | Claude Opus | Final push, contract, commitment |

Each agent updates the conversation metadata when transitioning to the next stage.

## Architecture

```
WhatsApp Message
    |
    v
┌──────────────────────┐
│   Webhook Server     │
│   (Express.js)       │
│                      │
│   1. Receive message │
│   2. Fetch convo     │
│      metadata.stage  │
│   3. Route to agent  │
└──────┬───────────────┘
       |
       ├── stage=qualify ──> Qualifier (Haiku)
       │                      • Asks budget, timeline, needs
       │                      • Scores lead 1-10
       │                      • If score >= 7 → stage=present
       │
       ├── stage=present ──> Presenter (Sonnet)
       │                      • Tailored product pitch
       │                      • Handles feature questions
       │                      • When interest confirmed → stage=negotiate
       │
       ├── stage=negotiate -> Negotiator (GPT-4)
       │                      • Handles objections
       │                      • Discusses pricing/terms
       │                      • When terms agreed → stage=close
       │
       └── stage=close ────> Closer (Opus)
                              • Confirms commitment
                              • Sends contract/payment link
                              • Marks deal won/lost
```

## Metadata Schema

```json
{
  "stage": "qualify | present | negotiate | close",
  "lead_score": 8,
  "qualified_at": "2026-04-02T10:00:00Z",
  "budget": "$5,000-$10,000",
  "timeline": "Q2 2026",
  "needs": ["automation", "reporting"],
  "product_interest": "Pro Plan",
  "objections": ["price too high", "need more integrations"],
  "proposed_price": 7500,
  "deal_status": "open | won | lost",
  "agent_history": [
    { "agent": "qualifier", "entered_at": "2026-04-02T09:55:00Z" },
    { "agent": "presenter", "entered_at": "2026-04-02T10:00:00Z" }
  ]
}
```

## Code

```javascript
// multi-agent-pipeline.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");
const OpenAI = require("openai");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// ── Agent Definitions ──────────────────────────────────────────────────

const AGENTS = {
  qualify: {
    model: "claude-haiku-4-20250514",
    provider: "anthropic",
    system: `You are a lead qualification agent for a SaaS company. Your job is to quickly assess if a prospect is a good fit.

Ask about (one at a time, conversationally):
1. What problem they're trying to solve
2. Their budget range
3. Their timeline for implementation
4. Team size / decision-making process

After gathering enough info, score the lead 1-10 internally.

IMPORTANT: When you've qualified the lead (have budget, timeline, and needs), end your message with exactly:
[TRANSITION:present]

Keep responses short and friendly. Max 2-3 sentences per reply.`,
  },

  present: {
    model: "claude-sonnet-4-20250514",
    provider: "anthropic",
    system: `You are a product presentation specialist. The lead has been qualified and you have their needs in the conversation context.

Your job:
- Present the product tailored to their specific needs
- Highlight relevant features and benefits
- Share pricing tiers when asked
- Answer technical questions with depth

Available plans:
- Starter: $49/mo — 5 users, basic features
- Pro: $149/mo — 25 users, advanced features, API access
- Enterprise: $499/mo — unlimited users, custom integrations, dedicated support

When the prospect shows clear interest and wants to discuss terms/pricing details, end your message with:
[TRANSITION:negotiate]

Be enthusiastic but not pushy. Use concrete examples.`,
  },

  negotiate: {
    model: "gpt-4",
    provider: "openai",
    system: `You are an expert sales negotiator. The prospect is interested in the product and wants to discuss terms.

Your techniques:
- Acknowledge objections genuinely before addressing them
- Use anchoring (start with higher-value packages)
- Offer limited-time incentives if needed (max 15% discount)
- Focus on ROI and value, not just price
- Create urgency without being aggressive

Available discount authority:
- Up to 10% for annual commitment
- Up to 15% for 2-year commitment
- Free onboarding (normally $500) for deals over $5,000/yr

When terms are agreed upon, end your message with:
[TRANSITION:close]

Never lie. Never make promises you can't keep.`,
  },

  close: {
    model: "claude-opus-4-20250514",
    provider: "anthropic",
    system: `You are a senior sales closer. The prospect has agreed to terms and you need to finalize the deal.

Your job:
- Confirm the agreed terms clearly
- Address any last-minute concerns with confidence
- Guide them through the next steps (contract signing, payment)
- Send a payment/contract link when ready

Next steps to communicate:
1. You'll receive a contract via email within the hour
2. Payment can be made via the secure link
3. Onboarding call will be scheduled within 48 hours

When the deal is confirmed, end your message with:
[DEAL:won]

If the prospect backs out, end with:
[DEAL:lost]

Be warm, professional, and reassuring. This is the final step.`,
  },
};

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

// ── AI Routing ─────────────────────────────────────────────────────────

async function callAgent(agentKey, messages) {
  const agent = AGENTS[agentKey];

  if (agent.provider === "anthropic") {
    const response = await anthropic.messages.create({
      model: agent.model,
      max_tokens: 1024,
      system: agent.system,
      messages,
    });
    return response.content[0].text;
  }

  if (agent.provider === "openai") {
    const response = await openai.chat.completions.create({
      model: agent.model,
      max_tokens: 1024,
      messages: [{ role: "system", content: agent.system }, ...messages],
    });
    return response.choices[0].message.content;
  }
}

function buildMessageHistory(rawMessages) {
  return rawMessages
    .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
    .map((msg) => ({
      role: msg.direction === "inbound" ? "user" : "assistant",
      content: typeof msg.content === "string" ? msg.content : JSON.stringify(msg.content),
    }));
}

function parseTransition(responseText) {
  const transitionMatch = responseText.match(/\[TRANSITION:(\w+)\]/);
  if (transitionMatch) {
    return {
      nextStage: transitionMatch[1],
      cleanText: responseText.replace(/\[TRANSITION:\w+\]/, "").trim(),
    };
  }

  const dealMatch = responseText.match(/\[DEAL:(\w+)\]/);
  if (dealMatch) {
    return {
      dealResult: dealMatch[1],
      cleanText: responseText.replace(/\[DEAL:\w+\]/, "").trim(),
    };
  }

  return { cleanText: responseText };
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received") return;

  const phone = data.from;
  const messageText =
    typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    // 1. Get or initialize conversation metadata
    const convoRes = await getConversation(phone);
    let metadata = convoRes.data?.metadata || {};

    if (!metadata.stage) {
      metadata = {
        stage: "qualify",
        agent_history: [
          { agent: "qualifier", entered_at: new Date().toISOString() },
        ],
      };
      await updateConversation(phone, metadata);
    }

    const currentStage = metadata.stage;

    // 2. Fetch conversation history
    const messagesRes = await getMessages(phone, 30);
    const history = buildMessageHistory(messagesRes.data || []);

    // 3. Show typing indicator
    await sendTyping(phone);

    // 4. Call the appropriate agent
    const rawResponse = await callAgent(currentStage, history);

    // 5. Parse for transitions
    const { nextStage, dealResult, cleanText } = parseTransition(rawResponse);

    // 6. Handle stage transition
    if (nextStage && AGENTS[nextStage]) {
      metadata.stage = nextStage;
      metadata[`${currentStage}_completed_at`] = new Date().toISOString();
      metadata.agent_history = metadata.agent_history || [];
      metadata.agent_history.push({
        agent: nextStage,
        entered_at: new Date().toISOString(),
      });
      await updateConversation(phone, metadata);
    }

    // 7. Handle deal result
    if (dealResult) {
      metadata.deal_status = dealResult;
      metadata.closed_at = new Date().toISOString();
      await updateConversation(phone, metadata);
    }

    // 8. Send response
    await sendText(phone, cleanText, {
      agent: currentStage,
      transitioned_to: nextStage || null,
      deal_result: dealResult || null,
    });
  } catch (err) {
    console.error("Pipeline error:", err);
    await sendText(phone, "Sorry, I'm having a moment. Let me get back to you shortly.");
  }
});

// ── Health check ───────────────────────────────────────────────────────

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Multi-agent pipeline running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express @anthropic-ai/sdk openai

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-key"
export OPENAI_API_KEY="your-openai-key"

# 3. Start the server
node multi-agent-pipeline.js

# 4. Expose via ngrok (for local development)
ngrok http 3000

# 5. Register your webhook in Opero WPP
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'
```

## Key Concepts

- **Stage-based routing**: `metadata.stage` determines which agent handles the message
- **Transition markers**: Agents signal transitions via `[TRANSITION:stage]` in their output, which the router strips before sending
- **Agent history**: Every stage change is logged in `metadata.agent_history` for analytics
- **Model optimization**: Cheap/fast models for simple tasks, powerful models for complex ones
- **Stateless agents**: Each agent gets the full conversation history and uses conversation metadata for context — no external database required
