# 12 — Structured Data Extraction

## Problem

Customers send unstructured messages like "I'm John, my email is john@email.com, budget around 5k, looking for something near downtown." Manually parsing these into structured CRM fields is slow and error-prone. You need to extract names, emails, addresses, preferences, and budgets from free-form text automatically.

## Solution

Use Claude Haiku to extract structured data from every inbound message. Extracted fields are merged into the conversation metadata incrementally — each message adds to or updates the profile. When enough data is collected, trigger follow-up actions (e.g., send a confirmation, notify sales).

## Architecture

```
WhatsApp Message (free-form text)
    |
    v
┌──────────────────────────────┐
│   Webhook Server             │
│                              │
│   1. Receive message         │
│   2. Load existing metadata  │
│   3. Send to Claude Haiku    │
│      with extraction prompt  │
│   4. Parse JSON response     │
│   5. Deep-merge with         │
│      existing metadata       │
│   6. Check completeness      │
│   7. Trigger actions if      │
│      data is sufficient      │
│   8. Reply with confirmation │
└──────────────────────────────┘
    |
    v
Conversation Metadata (accumulated)
{
  "name": "John Smith",
  "email": "john@email.com",
  "phone_alt": null,
  "address": "123 Main St, Downtown",
  "budget": { "min": 4000, "max": 6000, "currency": "USD" },
  "preferences": ["near downtown", "2 bedrooms"],
  "extraction_completeness": 0.75,
  "fields_collected": ["name", "email", "budget", "preferences"]
}
```

## Metadata Schema

```json
{
  "extracted": {
    "name": "John Smith",
    "email": "john@email.com",
    "phone_alt": "+1234567890",
    "company": "Acme Corp",
    "address": {
      "street": "123 Main St",
      "city": "Austin",
      "state": "TX",
      "zip": "78701"
    },
    "budget": {
      "min": 4000,
      "max": 6000,
      "currency": "USD"
    },
    "preferences": ["near downtown", "modern design", "2 bedrooms"],
    "timeline": "next 2 months",
    "notes": "Prefers email communication for documents"
  },
  "extraction_log": [
    {
      "message_id": "msg_abc123",
      "fields_found": ["name", "email"],
      "timestamp": "2026-04-02T10:00:00Z"
    }
  ],
  "completeness": 0.75,
  "fields_collected": ["name", "email", "budget", "preferences"],
  "fields_missing": ["phone_alt", "company", "address"],
  "actions_triggered": ["confirmation_sent"]
}
```

## Code

```javascript
// structured-data-extraction.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Fields we want to extract ──────────────────────────────────────────

const TARGET_FIELDS = [
  "name",
  "email",
  "phone_alt",
  "company",
  "address",
  "budget",
  "preferences",
  "timeline",
  "notes",
];

const REQUIRED_FIELDS = ["name", "email", "budget"];

const EXTRACTION_PROMPT = `You are a data extraction assistant. Extract structured information from the user's WhatsApp message.

Return ONLY a valid JSON object with these fields (use null for fields not mentioned):

{
  "name": "string or null — full name",
  "email": "string or null — email address",
  "phone_alt": "string or null — alternative phone number",
  "company": "string or null — company/organization name",
  "address": {
    "street": "string or null",
    "city": "string or null",
    "state": "string or null",
    "zip": "string or null"
  },
  "budget": {
    "min": "number or null",
    "max": "number or null",
    "currency": "string, default USD"
  },
  "preferences": ["array of preference strings, empty if none"],
  "timeline": "string or null — when they need the service",
  "notes": "string or null — any other relevant info"
}

Rules:
- Extract ONLY information explicitly stated in the message
- Do NOT infer or guess values
- Parse budget ranges ("5-10k" → min: 5000, max: 10000)
- Normalize phone numbers to international format
- Return valid JSON only, no markdown, no explanation`;

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

// ── Extraction Logic ───────────────────────────────────────────────────

async function extractFromMessage(messageText) {
  const response = await anthropic.messages.create({
    model: "claude-haiku-4-20250514",
    max_tokens: 1024,
    messages: [{ role: "user", content: messageText }],
    system: EXTRACTION_PROMPT,
  });

  const text = response.content[0].text;

  try {
    return JSON.parse(text);
  } catch {
    // Try to extract JSON from markdown code blocks
    const jsonMatch = text.match(/```(?:json)?\s*([\s\S]*?)```/);
    if (jsonMatch) {
      return JSON.parse(jsonMatch[1].trim());
    }
    console.error("Failed to parse extraction result:", text);
    return null;
  }
}

function deepMerge(existing, incoming) {
  const merged = { ...existing };

  for (const [key, value] of Object.entries(incoming)) {
    if (value === null || value === undefined) continue;

    if (Array.isArray(value)) {
      // Merge arrays, deduplicate
      const existingArr = Array.isArray(merged[key]) ? merged[key] : [];
      merged[key] = [...new Set([...existingArr, ...value])];
    } else if (typeof value === "object" && !Array.isArray(value)) {
      // Recursively merge objects
      merged[key] = deepMerge(merged[key] || {}, value);
    } else {
      // Overwrite scalar values
      merged[key] = value;
    }
  }

  return merged;
}

function calculateCompleteness(extracted) {
  const collected = [];
  const missing = [];

  for (const field of TARGET_FIELDS) {
    const value = extracted[field];
    const hasValue =
      value !== null &&
      value !== undefined &&
      value !== "" &&
      !(typeof value === "object" && Object.values(value).every((v) => v === null)) &&
      !(Array.isArray(value) && value.length === 0);

    if (hasValue) {
      collected.push(field);
    } else {
      missing.push(field);
    }
  }

  return {
    completeness: collected.length / TARGET_FIELDS.length,
    fields_collected: collected,
    fields_missing: missing,
  };
}

function checkRequiredFieldsComplete(extracted) {
  return REQUIRED_FIELDS.every((field) => {
    const value = extracted[field];
    return (
      value !== null &&
      value !== undefined &&
      value !== "" &&
      !(typeof value === "object" && Object.values(value).every((v) => v === null))
    );
  });
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
    await sendTyping(phone);

    // 1. Extract data from message
    const newData = await extractFromMessage(messageText);
    if (!newData) {
      await sendText(phone, "Got your message! Could you share a bit more details?");
      return;
    }

    // 2. Get existing metadata
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const existingExtracted = metadata.extracted || {};
    const existingLog = metadata.extraction_log || [];
    const existingActions = metadata.actions_triggered || [];

    // 3. Merge new extraction with existing
    const mergedExtracted = deepMerge(existingExtracted, newData);

    // 4. Calculate completeness
    const { completeness, fields_collected, fields_missing } =
      calculateCompleteness(mergedExtracted);

    // 5. Log this extraction
    const newFieldsFound = Object.entries(newData)
      .filter(([, v]) => v !== null && v !== undefined)
      .map(([k]) => k);

    existingLog.push({
      message_id: data.id,
      fields_found: newFieldsFound,
      timestamp: new Date().toISOString(),
    });

    // 6. Update conversation metadata
    const updatedMetadata = {
      ...metadata,
      extracted: mergedExtracted,
      extraction_log: existingLog,
      completeness,
      fields_collected,
      fields_missing,
      actions_triggered: existingActions,
    };

    await updateConversation(phone, updatedMetadata);

    // 7. Check if we should trigger actions
    const allRequiredCollected = checkRequiredFieldsComplete(mergedExtracted);

    if (allRequiredCollected && !existingActions.includes("confirmation_sent")) {
      // Send confirmation with collected data
      const summary = [
        `Here's what I have so far:`,
        mergedExtracted.name ? `- Name: ${mergedExtracted.name}` : null,
        mergedExtracted.email ? `- Email: ${mergedExtracted.email}` : null,
        mergedExtracted.company ? `- Company: ${mergedExtracted.company}` : null,
        mergedExtracted.budget?.min
          ? `- Budget: $${mergedExtracted.budget.min.toLocaleString()}${mergedExtracted.budget.max ? ` - $${mergedExtracted.budget.max.toLocaleString()}` : ""}`
          : null,
        mergedExtracted.timeline ? `- Timeline: ${mergedExtracted.timeline}` : null,
        mergedExtracted.preferences?.length
          ? `- Preferences: ${mergedExtracted.preferences.join(", ")}`
          : null,
        ``,
        `Does everything look correct? Anything you'd like to add or change?`,
      ]
        .filter(Boolean)
        .join("\n");

      await sendText(phone, summary, { action: "confirmation_sent" });

      updatedMetadata.actions_triggered.push("confirmation_sent");
      await updateConversation(phone, updatedMetadata);
    } else if (newFieldsFound.length > 0) {
      // Acknowledge the new info and ask for missing fields
      const ack = `Thanks! I've noted your ${newFieldsFound.join(", ")}.`;
      const missingRequired = REQUIRED_FIELDS.filter(
        (f) => !fields_collected.includes(f)
      );

      let followUp = "";
      if (missingRequired.length > 0) {
        const prompts = {
          name: "What's your name?",
          email: "What's your email address?",
          budget: "What's your budget range?",
        };
        followUp = ` ${prompts[missingRequired[0]] || `Could you also share your ${missingRequired[0]}?`}`;
      }

      await sendText(phone, ack + followUp, {
        fields_extracted: newFieldsFound,
        completeness,
      });
    } else {
      await sendText(phone, "Thanks for the message! Could you share some details like your name, email, and budget range?");
    }
  } catch (err) {
    console.error("Extraction error:", err);
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Data extraction bot running on :3000"));
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

# 3. Start the server
node structured-data-extraction.js

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

- **Incremental extraction**: Each message adds to the profile, not replaces it
- **Deep merge**: Arrays are deduplicated, objects are recursively merged, scalars are overwritten
- **Completeness tracking**: `metadata.completeness` shows data collection progress (0.0 to 1.0)
- **Action triggers**: Automatic actions fire when enough required data is collected
- **Extraction log**: Every extraction is logged with the message ID and fields found for auditability
