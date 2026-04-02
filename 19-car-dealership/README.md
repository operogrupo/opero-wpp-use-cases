# 19 — Car Dealership Sales Agent

## Problem

Car dealerships lose leads because salespeople can't respond fast enough. Buyers research online and reach out via WhatsApp expecting instant, knowledgeable answers about inventory, pricing, financing, and test drives. Manually qualifying every lead, showing matching vehicles, and scheduling test drives doesn't scale.

## Solution

An AI sales agent powered by Claude Sonnet that qualifies buyers conversationally, searches inventory based on their criteria, sends vehicle photos with details, schedules test drives, and handles trade-in inquiries. The full sales pipeline is tracked in conversation metadata.

## Architecture

```
Buyer: "Looking for a used SUV under 30k"
    |
    v
┌──────────────────────────────────────┐
│   Webhook Server                     │
│                                      │
│   1. Classify stage/intent:          │
│      qualify | search | details      │
│      | test_drive | trade_in         │
│      | financing | general           │
│   2. Execute action:                 │
│      - Search inventory              │
│      - Send vehicle images           │
│      - Schedule test drive           │
│      - Estimate trade-in             │
│   3. Update pipeline metadata        │
│   4. Notify sales team for           │
│      hot leads                       │
└──────────────────────────────────────┘
    |
    ├── Qualify: Budget, type, new/used, timeline
    ├── Search: Match inventory → send images
    ├── Test Drive: Schedule with calendar
    ├── Trade-in: Collect vehicle info → estimate
    └── Hot Lead Alert: Notify sales team
```

## Metadata Schema

```json
{
  "buyer": {
    "name": "James Wilson",
    "email": "james@email.com",
    "phone": "5551234567"
  },
  "pipeline": {
    "stage": "searching",
    "score": 8,
    "source": "whatsapp_organic"
  },
  "criteria": {
    "type": "SUV",
    "condition": "used",
    "budget_min": 20000,
    "budget_max": 30000,
    "brand_preference": ["Toyota", "Honda"],
    "must_have": ["AWD", "backup camera"],
    "nice_to_have": ["leather seats", "sunroof"],
    "color_preference": "dark colors"
  },
  "vehicles_shown": ["VIN-001", "VIN-002", "VIN-003"],
  "favorited": ["VIN-002"],
  "test_drives": [
    {
      "vehicle_id": "VIN-002",
      "vehicle_name": "2023 Toyota RAV4 XLE",
      "date": "2026-04-05",
      "time": "14:00",
      "status": "scheduled",
      "salesperson": "Mike"
    }
  ],
  "trade_in": {
    "year": 2019,
    "make": "Honda",
    "model": "Civic",
    "mileage": 65000,
    "condition": "good",
    "estimated_value": 14500
  },
  "financing_interested": true,
  "last_activity": "2026-04-02T10:00:00Z"
}
```

## Code

```javascript
// car-dealership.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const SALES_PHONE = process.env.SALES_PHONE;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Inventory ──────────────────────────────────────────────────────────

const INVENTORY = [
  {
    id: "VIN-001", year: 2024, make: "Toyota", model: "Camry", trim: "SE",
    condition: "new", price: 28500, mileage: 12, color: "Pearl White",
    features: ["Apple CarPlay", "Lane Departure", "Backup Camera", "Bluetooth"],
    image_url: "https://example.com/images/camry-se-2024.jpg",
    type: "sedan",
  },
  {
    id: "VIN-002", year: 2023, make: "Toyota", model: "RAV4", trim: "XLE",
    condition: "used", price: 27900, mileage: 18500, color: "Midnight Black",
    features: ["AWD", "Sunroof", "Backup Camera", "Heated Seats", "Apple CarPlay"],
    image_url: "https://example.com/images/rav4-xle-2023.jpg",
    type: "SUV",
  },
  {
    id: "VIN-003", year: 2022, make: "Honda", model: "CR-V", trim: "EX-L",
    condition: "used", price: 25800, mileage: 32000, color: "Steel Gray",
    features: ["AWD", "Leather Seats", "Sunroof", "Backup Camera", "Honda Sensing"],
    image_url: "https://example.com/images/crv-exl-2022.jpg",
    type: "SUV",
  },
  {
    id: "VIN-004", year: 2024, make: "Honda", model: "Accord", trim: "Sport",
    condition: "new", price: 31200, mileage: 8, color: "San Marino Red",
    features: ["Turbo", "Apple CarPlay", "Lane Keep", "Backup Camera", "Sport Seats"],
    image_url: "https://example.com/images/accord-sport-2024.jpg",
    type: "sedan",
  },
  {
    id: "VIN-005", year: 2023, make: "Ford", model: "Escape", trim: "SE",
    condition: "used", price: 22500, mileage: 25000, color: "Oxford White",
    features: ["AWD", "Backup Camera", "Bluetooth", "SYNC 3"],
    image_url: "https://example.com/images/escape-se-2023.jpg",
    type: "SUV",
  },
  {
    id: "VIN-006", year: 2024, make: "Hyundai", model: "Tucson", trim: "SEL",
    condition: "new", price: 29800, mileage: 5, color: "Phantom Black",
    features: ["AWD", "Panoramic Sunroof", "Backup Camera", "Wireless Charging", "Heated Seats"],
    image_url: "https://example.com/images/tucson-sel-2024.jpg",
    type: "SUV",
  },
  {
    id: "VIN-007", year: 2021, make: "Chevrolet", model: "Equinox", trim: "LT",
    condition: "used", price: 19500, mileage: 42000, color: "Silver Ice",
    features: ["AWD", "Backup Camera", "Apple CarPlay", "Remote Start"],
    image_url: "https://example.com/images/equinox-lt-2021.jpg",
    type: "SUV",
  },
];

// ── API Helpers ────────────────────────────────────────────────────────

async function apiRequest(method, path, body = null) {
  const options = {
    method,
    headers: { Authorization: `Bearer ${API_KEY}`, "Content-Type": "application/json" },
  };
  if (body) options.body = JSON.stringify(body);
  const res = await fetch(`${API_BASE}${path}`, options);
  return res.json();
}

async function getConversation(phone) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}`);
}

async function updateConversation(phone, metadata) {
  return apiRequest("PUT", `/api/numbers/${NUMBER_ID}/conversations/${phone}`, { metadata });
}

async function getMessages(phone, limit = 20) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`);
}

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, { phone, text, metadata });
}

async function sendImage(phone, url, caption, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/image`, { phone, url, caption, metadata });
}

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── Search Inventory ───────────────────────────────────────────────────

function searchInventory(criteria, maxResults = 3) {
  let results = [...INVENTORY];

  if (criteria.type) {
    results = results.filter((v) => v.type.toLowerCase() === criteria.type.toLowerCase());
  }
  if (criteria.condition) {
    results = results.filter((v) => v.condition === criteria.condition);
  }
  if (criteria.budget_max) {
    results = results.filter((v) => v.price <= criteria.budget_max);
  }
  if (criteria.budget_min) {
    results = results.filter((v) => v.price >= criteria.budget_min);
  }
  if (criteria.brand_preference?.length > 0) {
    const preferred = results.filter((v) =>
      criteria.brand_preference.some((b) => b.toLowerCase() === v.make.toLowerCase())
    );
    if (preferred.length > 0) results = preferred;
  }
  if (criteria.must_have?.length > 0) {
    results = results.filter((v) =>
      criteria.must_have.every((feat) =>
        v.features.some((f) => f.toLowerCase().includes(feat.toLowerCase()))
      )
    );
  }

  return results.slice(0, maxResults);
}

// ── AI Agent ───────────────────────────────────────────────────────────

async function processMessage(messages, metadata) {
  const criteria = metadata.criteria || {};
  const pipeline = metadata.pipeline || {};
  const vehiclesShown = metadata.vehicles_shown || [];

  const inventorySummary = INVENTORY.map(
    (v) => `${v.id}: ${v.year} ${v.make} ${v.model} ${v.trim} (${v.condition}, $${v.price.toLocaleString()}, ${v.mileage.toLocaleString()}mi, ${v.type})`
  ).join("\n");

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: `You are a professional car sales consultant at a multi-brand dealership. Be knowledgeable, helpful, and never pushy.

Current Inventory:
${inventorySummary}

Buyer's criteria so far: ${JSON.stringify(criteria)}
Pipeline stage: ${pipeline.stage || "new"}
Vehicles already shown: ${vehiclesShown.join(", ") || "none"}

Your response MUST end with a JSON block:
{
  "intent": "qualify | search | details | test_drive | trade_in | financing | general",
  "criteria_update": { "type": null, "condition": null, "budget_min": null, "budget_max": null, "brand_preference": [], "must_have": [], "color_preference": null },
  "vehicle_ids_to_show": [],
  "buyer_name": null,
  "buyer_email": null,
  "test_drive_date": null,
  "test_drive_time": null,
  "test_drive_vehicle": null,
  "trade_in_info": null,
  "lead_score": 0,
  "stage": "new | qualifying | searching | test_drive | negotiating | closed"
}

Rules:
- Qualify naturally — don't fire questions, have a conversation
- When showing vehicles, include the VIN ID so images can be sent
- Only suggest test drives after showing inventory
- Be honest about pricing — no hidden fees
- If trade-in mentioned, collect year, make, model, mileage, condition`,
    messages,
  });

  const text = response.content[0].text;
  const jsonMatch = text.match(/\{[\s\S]*\}$/);

  let parsed = {};
  let cleanText = text;

  if (jsonMatch) {
    try {
      parsed = JSON.parse(jsonMatch[0]);
      cleanText = text.replace(jsonMatch[0], "").trim();
    } catch {
      // Use text as-is
    }
  }

  return { text: cleanText, ...parsed };
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received" || data.type !== "text") return;

  const phone = data.from;
  const messageText = typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    await sendTyping(phone);

    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    metadata.buyer = metadata.buyer || {};
    metadata.pipeline = metadata.pipeline || { stage: "new", score: 0 };
    metadata.criteria = metadata.criteria || {};
    metadata.vehicles_shown = metadata.vehicles_shown || [];
    metadata.favorited = metadata.favorited || [];
    metadata.test_drives = metadata.test_drives || [];

    // Build history
    const messagesRes = await getMessages(phone, 25);
    const history = (messagesRes.data || [])
      .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
      .map((msg) => ({
        role: msg.direction === "inbound" ? "user" : "assistant",
        content: typeof msg.content === "string" ? msg.content : msg.content?.text || JSON.stringify(msg.content),
      }));

    // Process with AI
    const result = await processMessage(history, metadata);

    // Update criteria
    if (result.criteria_update) {
      for (const [key, value] of Object.entries(result.criteria_update)) {
        if (value !== null && value !== undefined && !(Array.isArray(value) && value.length === 0)) {
          if (Array.isArray(value) && Array.isArray(metadata.criteria[key])) {
            metadata.criteria[key] = [...new Set([...metadata.criteria[key], ...value])];
          } else {
            metadata.criteria[key] = value;
          }
        }
      }
    }

    // Update buyer info
    if (result.buyer_name) metadata.buyer.name = result.buyer_name;
    if (result.buyer_email) metadata.buyer.email = result.buyer_email;

    // Update pipeline
    if (result.stage) metadata.pipeline.stage = result.stage;
    if (result.lead_score) metadata.pipeline.score = Math.max(metadata.pipeline.score, result.lead_score);

    // Update trade-in info
    if (result.trade_in_info) {
      metadata.trade_in = { ...metadata.trade_in, ...result.trade_in_info };
    }

    // Schedule test drive
    if (result.test_drive_date && result.test_drive_vehicle) {
      const vehicle = INVENTORY.find((v) => v.id === result.test_drive_vehicle);
      if (vehicle) {
        metadata.test_drives.push({
          vehicle_id: vehicle.id,
          vehicle_name: `${vehicle.year} ${vehicle.make} ${vehicle.model} ${vehicle.trim}`,
          date: result.test_drive_date,
          time: result.test_drive_time || "14:00",
          status: "scheduled",
        });
        metadata.pipeline.stage = "test_drive";
      }
    }

    // Send text response first
    await sendText(phone, result.text, {
      intent: result.intent,
      stage: metadata.pipeline.stage,
    });

    // Send vehicle images if specified
    if (result.vehicle_ids_to_show?.length > 0) {
      for (const vid of result.vehicle_ids_to_show) {
        const vehicle = INVENTORY.find((v) => v.id === vid);
        if (!vehicle) continue;

        if (!metadata.vehicles_shown.includes(vid)) {
          metadata.vehicles_shown.push(vid);
        }

        const caption = [
          `*${vehicle.year} ${vehicle.make} ${vehicle.model} ${vehicle.trim}*`,
          `${vehicle.condition === "new" ? "NEW" : `${vehicle.mileage.toLocaleString()} miles`} | ${vehicle.color}`,
          `*$${vehicle.price.toLocaleString()}*`,
          ``,
          `Features: ${vehicle.features.join(", ")}`,
          ``,
          `Reply "${vid}" for more details or to schedule a test drive`,
        ].join("\n");

        await sendImage(phone, vehicle.image_url, caption, { vehicle_id: vid });
      }
    }

    // Alert sales team for hot leads (score >= 7)
    if (metadata.pipeline.score >= 7 && SALES_PHONE && !metadata.pipeline.alert_sent) {
      await sendText(SALES_PHONE, [
        `HOT LEAD ALERT`,
        ``,
        `Buyer: ${metadata.buyer.name || "Unknown"} (${phone})`,
        `Score: ${metadata.pipeline.score}/10`,
        `Looking for: ${metadata.criteria.type || "?"} (${metadata.criteria.condition || "?"})`,
        `Budget: $${metadata.criteria.budget_min?.toLocaleString() || "?"} - $${metadata.criteria.budget_max?.toLocaleString() || "?"}`,
        `Stage: ${metadata.pipeline.stage}`,
        `Vehicles interested in: ${metadata.vehicles_shown.join(", ")}`,
      ].join("\n"), { alert_type: "hot_lead", customer_phone: phone });

      metadata.pipeline.alert_sent = true;
    }

    metadata.last_activity = new Date().toISOString();
    await updateConversation(phone, metadata);
  } catch (err) {
    console.error("Dealership bot error:", err);
    await sendText(phone, "Let me check on that for you. In the meantime, you can reach our team at (555) 123-4567.");
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Car dealership bot running on :3000"));
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
export SALES_PHONE="5491155551234"  # Sales manager phone

# 3. Start the server
node car-dealership.js

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

- **Full pipeline tracking**: Every buyer's journey from first message to test drive is tracked in `metadata.pipeline`
- **Vehicle images**: Uses `POST /messages/image` to send car photos with specs directly in WhatsApp
- **Lead scoring**: AI assigns a 1-10 score based on buying signals (budget clarity, timeline, specific preferences)
- **Hot lead alerts**: Sales team gets instant WhatsApp alerts when a lead scores 7 or higher
- **Trade-in handling**: Collects vehicle details and provides preliminary trade-in estimates
- **Criteria refinement**: `metadata.criteria` builds up as the conversation progresses — the search gets more targeted
