# 20 — Insurance Quote Generator

## Problem

Insurance agents spend hours on the phone collecting applicant information for quotes. Customers drop off when the process feels too long or invasive. Getting from initial contact to a preliminary quote takes days instead of minutes.

## Solution

A conversational quote generator powered by Claude Sonnet. It collects applicant information naturally (age, coverage type, details) through a WhatsApp conversation, calculates a preliminary quote based on underwriting rules, and routes qualified applicants to a human agent for the final quote. All collected data is structured in metadata.

## Architecture

```
Applicant: "I need car insurance for my 2022 Honda Civic"
    |
    v
┌──────────────────────────────────────┐
│   Webhook Server                     │
│                                      │
│   1. Identify coverage type:         │
│      auto | home | life | health     │
│   2. Collect required fields         │
│      conversationally                │
│   3. When all fields collected:      │
│      → Calculate preliminary quote   │
│   4. Present quote options           │
│   5. If interested → route to        │
│      human agent for final quote     │
└──────────────────────────────────────┘
    |
    ├── Data Collection: Natural Q&A, one topic at a time
    ├── Quote Calculation: Rules-based preliminary pricing
    ├── Quote Presentation: 3 tiers (basic/standard/premium)
    └── Agent Handoff: Qualified lead → human agent
```

## Metadata Schema

```json
{
  "applicant": {
    "name": "Sarah Chen",
    "age": 34,
    "email": "sarah@email.com",
    "zip_code": "90210"
  },
  "coverage_type": "auto",
  "collection_stage": "quoting",
  "auto_details": {
    "vehicle_year": 2022,
    "vehicle_make": "Honda",
    "vehicle_model": "Civic",
    "vehicle_trim": "EX",
    "ownership": "financed",
    "annual_mileage": 12000,
    "primary_use": "commute",
    "driving_record": {
      "years_licensed": 16,
      "accidents_3yr": 0,
      "violations_3yr": 1
    },
    "current_coverage": "basic",
    "desired_deductible": 500
  },
  "quote": {
    "generated_at": "2026-04-02T10:00:00Z",
    "options": [
      { "tier": "basic", "monthly": 89, "deductible": 1000, "coverage": "liability only" },
      { "tier": "standard", "monthly": 134, "deductible": 500, "coverage": "full coverage" },
      { "tier": "premium", "monthly": 178, "deductible": 250, "coverage": "full + extras" }
    ],
    "selected_tier": "standard",
    "valid_until": "2026-04-09T10:00:00Z"
  },
  "fields_collected": ["name", "age", "vehicle", "driving_record"],
  "fields_missing": ["zip_code", "annual_mileage"],
  "agent_assigned": null,
  "status": "quote_presented"
}
```

## Code

```javascript
// insurance-quotes.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const AGENT_PHONE = process.env.AGENT_PHONE;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Required Fields per Coverage Type ──────────────────────────────────

const REQUIRED_FIELDS = {
  auto: {
    applicant: ["name", "age", "zip_code"],
    details: ["vehicle_year", "vehicle_make", "vehicle_model", "annual_mileage", "primary_use", "years_licensed", "accidents_3yr"],
  },
  home: {
    applicant: ["name", "age", "zip_code"],
    details: ["home_value", "year_built", "square_feet", "construction_type", "security_system", "claims_5yr"],
  },
  life: {
    applicant: ["name", "age", "zip_code"],
    details: ["coverage_amount", "term_years", "smoker", "health_conditions", "occupation"],
  },
};

// ── Quote Calculation ──────────────────────────────────────────────────

function calculateAutoQuote(applicant, details) {
  let baseRate = 100; // monthly base

  // Age factor
  if (applicant.age < 25) baseRate *= 1.6;
  else if (applicant.age < 30) baseRate *= 1.2;
  else if (applicant.age > 65) baseRate *= 1.3;

  // Vehicle age factor
  const vehicleAge = 2026 - (details.vehicle_year || 2020);
  if (vehicleAge <= 2) baseRate *= 1.15;
  else if (vehicleAge >= 10) baseRate *= 0.85;

  // Mileage factor
  if ((details.annual_mileage || 12000) > 20000) baseRate *= 1.15;
  else if ((details.annual_mileage || 12000) < 8000) baseRate *= 0.9;

  // Driving record
  const accidents = details.accidents_3yr || 0;
  const violations = details.violations_3yr || 0;
  baseRate *= 1 + accidents * 0.25 + violations * 0.1;

  return [
    { tier: "basic", monthly: Math.round(baseRate * 0.65), deductible: 1000, coverage: "Liability only (state minimum)" },
    { tier: "standard", monthly: Math.round(baseRate), deductible: 500, coverage: "Full coverage (liability + collision + comprehensive)" },
    { tier: "premium", monthly: Math.round(baseRate * 1.35), deductible: 250, coverage: "Full coverage + rental, roadside, gap, new car replacement" },
  ];
}

function calculateHomeQuote(applicant, details) {
  const homeValue = details.home_value || 300000;
  let baseRate = homeValue * 0.003 / 12; // ~0.3% annually

  const homeAge = 2026 - (details.year_built || 2000);
  if (homeAge > 30) baseRate *= 1.3;
  else if (homeAge < 10) baseRate *= 0.85;

  if (details.security_system) baseRate *= 0.9;
  baseRate *= 1 + (details.claims_5yr || 0) * 0.2;

  return [
    { tier: "basic", monthly: Math.round(baseRate * 0.7), deductible: 2500, coverage: "Dwelling + liability (basic limits)" },
    { tier: "standard", monthly: Math.round(baseRate), deductible: 1000, coverage: "Dwelling + liability + personal property" },
    { tier: "premium", monthly: Math.round(baseRate * 1.4), deductible: 500, coverage: "Replacement cost + scheduled items + water backup" },
  ];
}

function calculateLifeQuote(applicant, details) {
  const amount = details.coverage_amount || 500000;
  const term = details.term_years || 20;
  let baseRate = (amount / 100000) * 12; // per $100k per month

  if (applicant.age > 50) baseRate *= 2.5;
  else if (applicant.age > 40) baseRate *= 1.5;

  if (details.smoker) baseRate *= 2;
  if (details.health_conditions?.length > 0) baseRate *= 1.3;

  baseRate *= term / 20; // adjust for term length

  return [
    { tier: "basic", monthly: Math.round(baseRate * 0.8), deductible: 0, coverage: `$${(amount * 0.5).toLocaleString()} / ${term}-year term` },
    { tier: "standard", monthly: Math.round(baseRate), deductible: 0, coverage: `$${amount.toLocaleString()} / ${term}-year term` },
    { tier: "premium", monthly: Math.round(baseRate * 1.5), deductible: 0, coverage: `$${(amount * 1.5).toLocaleString()} / ${term}-year term + living benefits` },
  ];
}

function calculateQuote(coverageType, applicant, details) {
  switch (coverageType) {
    case "auto": return calculateAutoQuote(applicant, details);
    case "home": return calculateHomeQuote(applicant, details);
    case "life": return calculateLifeQuote(applicant, details);
    default: return null;
  }
}

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

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── AI Data Collection ─────────────────────────────────────────────────

async function collectData(messages, metadata) {
  const coverageType = metadata.coverage_type || "unknown";
  const applicant = metadata.applicant || {};
  const details = metadata[`${coverageType}_details`] || {};
  const fieldsCollected = metadata.fields_collected || [];

  const required = REQUIRED_FIELDS[coverageType];
  const allRequired = required
    ? [...required.applicant, ...required.details]
    : [];
  const missing = allRequired.filter((f) => !fieldsCollected.includes(f));

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: `You are a friendly insurance advisor collecting information for a ${coverageType} insurance quote. You work for SafeGuard Insurance.

Current applicant data: ${JSON.stringify(applicant)}
Current ${coverageType} details: ${JSON.stringify(details)}
Fields still needed: ${missing.join(", ") || "ALL COLLECTED"}

IMPORTANT RULES:
- Collect information conversationally, 1-2 fields at a time
- Be warm and explain WHY you need each piece of info
- If all fields are collected, say [READY_FOR_QUOTE]
- Never ask for SSN, bank details, or payment info
- If user asks about pricing before all fields collected, say you need a few more details first

If the user hasn't specified a coverage type yet, ask what type of insurance they need (auto, home, or life).

End your response with JSON:
{
  "coverage_type": "auto | home | life | null",
  "extracted": {
    "name": null, "age": null, "zip_code": null,
    "vehicle_year": null, "vehicle_make": null, "vehicle_model": null,
    "annual_mileage": null, "primary_use": null,
    "years_licensed": null, "accidents_3yr": null, "violations_3yr": null,
    "home_value": null, "year_built": null, "square_feet": null,
    "construction_type": null, "security_system": null, "claims_5yr": null,
    "coverage_amount": null, "term_years": null, "smoker": null,
    "health_conditions": null, "occupation": null
  },
  "ready_for_quote": false,
  "selected_tier": null
}`,
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
    metadata.applicant = metadata.applicant || {};
    metadata.fields_collected = metadata.fields_collected || [];
    metadata.collection_stage = metadata.collection_stage || "collecting";

    // Check if quote was already presented and user is selecting a tier
    if (metadata.collection_stage === "quote_presented") {
      const tierMatch = messageText.toLowerCase().match(/\b(basic|standard|premium)\b/);
      if (tierMatch) {
        metadata.quote.selected_tier = tierMatch[1];
        metadata.status = "tier_selected";
        metadata.collection_stage = "agent_handoff";
        await updateConversation(phone, metadata);

        const selectedOption = metadata.quote.options.find((o) => o.tier === tierMatch[1]);

        await sendText(phone, [
          `Great choice! You selected the *${tierMatch[1].toUpperCase()}* plan at $${selectedOption.monthly}/month.`,
          ``,
          `I'm connecting you with one of our licensed agents who will finalize your quote and walk you through the policy details. They'll reach out shortly!`,
          ``,
          `Your quote is valid for 7 days (until ${metadata.quote.valid_until?.split("T")[0]}).`,
        ].join("\n"), { action: "tier_selected", tier: tierMatch[1] });

        // Notify agent
        if (AGENT_PHONE) {
          const applicant = metadata.applicant;
          await sendText(AGENT_PHONE, [
            `NEW QUALIFIED LEAD — ${metadata.coverage_type?.toUpperCase()} INSURANCE`,
            ``,
            `Applicant: ${applicant.name || "Unknown"} (age ${applicant.age || "?"})`,
            `Phone: ${phone}`,
            `Zip: ${applicant.zip_code || "?"}`,
            `Selected: ${tierMatch[1]} ($${selectedOption.monthly}/mo)`,
            ``,
            `Full details in conversation metadata.`,
          ].join("\n"), { alert_type: "qualified_lead", customer_phone: phone });
        }
        return;
      }
    }

    // Build conversation history
    const messagesRes = await getMessages(phone, 25);
    const history = (messagesRes.data || [])
      .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
      .map((msg) => ({
        role: msg.direction === "inbound" ? "user" : "assistant",
        content: typeof msg.content === "string" ? msg.content : msg.content?.text || JSON.stringify(msg.content),
      }));

    // Collect data via AI
    const result = await collectData(history, metadata);

    // Update coverage type
    if (result.coverage_type && result.coverage_type !== "null") {
      metadata.coverage_type = result.coverage_type;
    }

    // Merge extracted data
    if (result.extracted) {
      const detailsKey = `${metadata.coverage_type}_details`;
      metadata[detailsKey] = metadata[detailsKey] || {};

      for (const [key, value] of Object.entries(result.extracted)) {
        if (value === null || value === undefined) continue;

        // Determine if it's an applicant field or detail field
        if (["name", "age", "zip_code", "email"].includes(key)) {
          metadata.applicant[key] = value;
        } else {
          metadata[detailsKey][key] = value;
        }

        if (!metadata.fields_collected.includes(key)) {
          metadata.fields_collected.push(key);
        }
      }
    }

    // Check if ready for quote
    if (result.ready_for_quote && metadata.coverage_type) {
      const detailsKey = `${metadata.coverage_type}_details`;
      const quoteOptions = calculateQuote(
        metadata.coverage_type,
        metadata.applicant,
        metadata[detailsKey] || {}
      );

      if (quoteOptions) {
        const validUntil = new Date();
        validUntil.setDate(validUntil.getDate() + 7);

        metadata.quote = {
          generated_at: new Date().toISOString(),
          options: quoteOptions,
          selected_tier: null,
          valid_until: validUntil.toISOString(),
        };
        metadata.collection_stage = "quote_presented";
        metadata.status = "quote_presented";

        await updateConversation(phone, metadata);

        const quoteMsg = [
          `Here are your preliminary quote options for ${metadata.coverage_type} insurance:`,
          ``,
          ...quoteOptions.map(
            (o) => `*${o.tier.toUpperCase()}* — $${o.monthly}/month\n${o.coverage}${o.deductible ? `\nDeductible: $${o.deductible.toLocaleString()}` : ""}`
          ),
          ``,
          `These are preliminary rates. Your final quote may vary based on a full underwriting review.`,
          ``,
          `Reply with *basic*, *standard*, or *premium* to select a plan, or ask any questions!`,
        ].join("\n\n");

        await sendText(phone, quoteMsg, { action: "quote_presented" });
        return;
      }
    }

    // Save and respond
    await updateConversation(phone, metadata);
    await sendText(phone, result.text, {
      collection_stage: metadata.collection_stage,
      fields_count: metadata.fields_collected.length,
    });
  } catch (err) {
    console.error("Insurance bot error:", err);
    await sendText(phone, "I'm having a technical issue. Please try again or call us at (555) 123-4567 for immediate assistance.");
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Insurance quote bot running on :3000"));
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
export AGENT_PHONE="5491155551234"  # Licensed agent phone

# 3. Start the server
node insurance-quotes.js

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

- **Conversational data collection**: AI asks 1-2 questions at a time naturally, not a form
- **Multi-product support**: Auto, home, and life insurance each have their own field requirements and quote calculations
- **Preliminary quoting**: Rules-based pricing gives instant preliminary quotes while making clear final rates may differ
- **Tiered presentation**: Three options (basic/standard/premium) with anchoring toward the middle tier
- **Agent handoff**: Qualified leads with selected tiers are routed to licensed agents with full context
- **No sensitive data**: No SSN, bank details, or payment info is collected in the bot
