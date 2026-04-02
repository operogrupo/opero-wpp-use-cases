# 22 — Property Management Bot

## Problem

Property managers deal with a constant stream of tenant requests — maintenance issues, rent inquiries, building announcements — across phone calls, emails, and texts. Urgent maintenance requests get lost. Tenants don't know who to contact for what. Tracking request status is manual and error-prone.

## Solution

A WhatsApp bot powered by Claude Haiku that handles tenant requests. Maintenance issues are tagged by urgency and type, then dispatched to the right team. Tenants can check rent status, receive building announcements, and track their maintenance request history — all through conversation metadata.

## Architecture

```
Tenant: "The kitchen sink is leaking pretty badly"
    |
    v
┌──────────────────────────────────────┐
│   Webhook Server                     │
│                                      │
│   1. Classify request type:          │
│      maintenance | rent | announce   │
│      | general                       │
│   2. For maintenance:                │
│      a. Classify urgency             │
│         (emergency/high/medium/low)  │
│      b. Classify type                │
│         (plumbing/electrical/etc)    │
│      c. Create work order            │
│      d. Dispatch to team             │
│   3. For rent: Show status           │
│   4. Update conversation metadata    │
└──────────────────────────────────────┘
    |
    ├── Emergency → Immediate dispatch + call maintenance
    ├── High → Same-day dispatch
    ├── Medium → Next business day
    ├── Low → Scheduled maintenance window
    └── Rent inquiry → Show balance + history
```

## Metadata Schema

```json
{
  "tenant": {
    "name": "Maria Lopez",
    "unit": "4B",
    "building": "Sunset Apartments",
    "lease_start": "2025-06-01",
    "lease_end": "2026-05-31"
  },
  "maintenance_requests": [
    {
      "id": "MR-001",
      "type": "plumbing",
      "urgency": "high",
      "description": "Kitchen sink leaking, water pooling on floor",
      "status": "dispatched",
      "assigned_to": "Mario (Plumbing)",
      "created_at": "2026-04-02T10:00:00Z",
      "eta": "Today before 5 PM",
      "resolved_at": null,
      "tenant_rating": null
    }
  ],
  "rent": {
    "monthly_amount": 1500,
    "due_date": 1,
    "status": "current",
    "last_payment": "2026-04-01",
    "balance": 0
  },
  "announcements_received": ["2026-03-water-shutoff", "2026-04-pest-control"],
  "request_count": 3,
  "avg_resolution_hours": 18
}
```

## Code

```javascript
// property-management.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const MAINTENANCE_PHONE = process.env.MAINTENANCE_PHONE;
const MANAGER_PHONE = process.env.MANAGER_PHONE;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Tenant Directory ───────────────────────────────────────────────────
// In production, connect to your property management system

const TENANTS = new Map([
  ["5491155550001", { name: "Maria Lopez", unit: "4B", building: "Sunset Apartments", rent: 1500, lease_end: "2026-05-31" }],
  ["5491155550002", { name: "Carlos Ruiz", unit: "2A", building: "Sunset Apartments", rent: 1200, lease_end: "2026-08-31" }],
  ["5491155550003", { name: "Ana Martinez", unit: "7C", building: "Sunset Apartments", rent: 1800, lease_end: "2027-01-31" }],
]);

// Rent ledger (simplified)
const RENT_LEDGER = new Map([
  ["5491155550001", { status: "current", balance: 0, last_payment: "2026-04-01", payments: [{ date: "2026-04-01", amount: 1500 }, { date: "2026-03-01", amount: 1500 }] }],
  ["5491155550002", { status: "overdue", balance: 1200, last_payment: "2026-03-01", payments: [{ date: "2026-03-01", amount: 1200 }] }],
  ["5491155550003", { status: "current", balance: 0, last_payment: "2026-04-01", payments: [{ date: "2026-04-01", amount: 1800 }] }],
]);

// Maintenance team
const MAINTENANCE_TEAMS = {
  plumbing: { name: "Mario", phone: "5491155559001", response: "Same day" },
  electrical: { name: "Jorge", phone: "5491155559002", response: "Same day" },
  hvac: { name: "Roberto", phone: "5491155559003", response: "Next business day" },
  general: { name: "Luis", phone: "5491155559004", response: "1-2 business days" },
  pest_control: { name: "External (PestCo)", phone: "5491155559005", response: "Scheduled" },
  locksmith: { name: "24h Locksmith", phone: "5491155559006", response: "Within 1 hour" },
};

const URGENCY_ETA = {
  emergency: "Within 1 hour",
  high: "Today",
  medium: "Next business day",
  low: "Scheduled maintenance window (weekly)",
};

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

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, { phone, text, metadata });
}

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── Request Classification ─────────────────────────────────────────────

async function classifyRequest(messageText, tenantInfo) {
  const response = await anthropic.messages.create({
    model: "claude-haiku-4-20250514",
    max_tokens: 512,
    system: `You are a property management assistant. Classify the tenant's request.

Tenant: ${tenantInfo.name}, Unit ${tenantInfo.unit}, ${tenantInfo.building}

Respond with JSON only:
{
  "type": "maintenance | rent | lease | announcement | general",
  "maintenance_type": "plumbing | electrical | hvac | general | pest_control | locksmith | null",
  "urgency": "emergency | high | medium | low | null",
  "description": "brief description of the issue",
  "response_text": "friendly response to tenant"
}

Urgency guide:
- emergency: Flooding, gas leak, fire damage, broken locks (security), no heat in winter
- high: Significant water leak, no hot water, broken appliance (fridge/oven), pest infestation
- medium: Minor leak, appliance issue (dishwasher/washer), broken fixture
- low: Cosmetic issues, squeaky door, light bulb replacement, minor wear`,
    messages: [{ role: "user", content: messageText }],
  });

  const text = response.content[0].text;
  try { return JSON.parse(text); } catch {
    const m = text.match(/\{[\s\S]*\}/);
    return m ? JSON.parse(m[0]) : { type: "general", response_text: text };
  }
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received" || data.type !== "text") return;

  const phone = data.from;
  const messageText = typeof data.content === "string" ? data.content : data.content?.text || "";

  // Look up tenant
  const tenantInfo = TENANTS.get(phone);
  if (!tenantInfo) {
    await sendText(phone, "Welcome! This is the Sunset Apartments management bot. It seems your number isn't registered in our system. Please contact the front office at (555) 123-4567 to get set up.");
    return;
  }

  try {
    await sendTyping(phone);

    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    metadata.tenant = metadata.tenant || tenantInfo;
    metadata.maintenance_requests = metadata.maintenance_requests || [];

    const classification = await classifyRequest(messageText, tenantInfo);

    switch (classification.type) {
      case "maintenance": {
        const requestId = `MR-${Date.now().toString(36).toUpperCase()}`;
        const team = MAINTENANCE_TEAMS[classification.maintenance_type] || MAINTENANCE_TEAMS.general;
        const eta = classification.urgency === "emergency"
          ? URGENCY_ETA.emergency
          : URGENCY_ETA[classification.urgency] || URGENCY_ETA.medium;

        const request = {
          id: requestId,
          type: classification.maintenance_type || "general",
          urgency: classification.urgency || "medium",
          description: classification.description || messageText.substring(0, 200),
          status: "dispatched",
          assigned_to: `${team.name} (${classification.maintenance_type || "general"})`,
          created_at: new Date().toISOString(),
          eta,
          resolved_at: null,
          tenant_rating: null,
        };

        metadata.maintenance_requests.push(request);
        metadata.request_count = (metadata.request_count || 0) + 1;
        await updateConversation(phone, metadata);

        // Respond to tenant
        const urgencyEmoji = { emergency: "!", high: "", medium: "", low: "" };
        await sendText(phone, [
          `Maintenance request submitted${classification.urgency === "emergency" ? " — EMERGENCY priority" : ""}.`,
          ``,
          `Request ID: ${requestId}`,
          `Issue: ${classification.description}`,
          `Priority: ${classification.urgency?.toUpperCase()}`,
          `Assigned to: ${team.name}`,
          `ETA: ${eta}`,
          ``,
          classification.urgency === "emergency"
            ? `Our emergency team has been notified and will contact you shortly. If this is a life-threatening emergency, please call 911.`
            : `We'll keep you updated on the status. Reply "status" anytime to check.`,
        ].join("\n"), {
          action: "maintenance_request",
          request_id: requestId,
          urgency: classification.urgency,
        });

        // Dispatch to maintenance team
        const dispatchPhone = classification.urgency === "emergency"
          ? MANAGER_PHONE || MAINTENANCE_PHONE
          : MAINTENANCE_PHONE;

        if (dispatchPhone) {
          await sendText(dispatchPhone, [
            `${classification.urgency === "emergency" ? "EMERGENCY " : ""}MAINTENANCE REQUEST`,
            ``,
            `ID: ${requestId}`,
            `Tenant: ${tenantInfo.name} — Unit ${tenantInfo.unit}`,
            `Type: ${classification.maintenance_type}`,
            `Priority: ${classification.urgency?.toUpperCase()}`,
            `Issue: ${classification.description}`,
            `ETA: ${eta}`,
            ``,
            `Assigned to: ${team.name} (${team.phone})`,
          ].join("\n"), {
            alert_type: "maintenance_dispatch",
            request_id: requestId,
            urgency: classification.urgency,
          });
        }
        break;
      }

      case "rent": {
        const ledger = RENT_LEDGER.get(phone) || { status: "unknown", balance: 0 };
        metadata.rent = {
          monthly_amount: tenantInfo.rent,
          due_date: 1,
          status: ledger.status,
          last_payment: ledger.last_payment,
          balance: ledger.balance,
        };

        await updateConversation(phone, metadata);

        if (ledger.status === "current") {
          await sendText(phone, [
            `Your rent account is current. Here's a summary:`,
            ``,
            `Monthly rent: $${tenantInfo.rent.toLocaleString()}`,
            `Last payment: ${ledger.last_payment}`,
            `Balance: $0`,
            `Next due: ${new Date().getMonth() === 11 ? "January" : new Date(2026, new Date().getMonth() + 1, 1).toLocaleDateString("en-US", { month: "long" })} 1st`,
            ``,
            `Need anything else?`,
          ].join("\n"));
        } else {
          await sendText(phone, [
            `Your rent account shows an outstanding balance:`,
            ``,
            `Monthly rent: $${tenantInfo.rent.toLocaleString()}`,
            `Outstanding balance: $${ledger.balance.toLocaleString()}`,
            `Last payment: ${ledger.last_payment}`,
            ``,
            `Please make your payment as soon as possible. If you're experiencing financial difficulties, contact our office at (555) 123-4567 to discuss payment arrangements.`,
          ].join("\n"));
        }
        break;
      }

      case "lease": {
        await sendText(phone, [
          `Your lease details:`,
          ``,
          `Unit: ${tenantInfo.unit}`,
          `Building: ${tenantInfo.building}`,
          `Lease end: ${tenantInfo.lease_end}`,
          `Monthly rent: $${tenantInfo.rent.toLocaleString()}`,
          ``,
          `For lease renewal or questions, please contact the management office at (555) 123-4567.`,
        ].join("\n"));
        break;
      }

      default: {
        // Check if asking about request status
        if (messageText.toLowerCase().includes("status")) {
          const active = metadata.maintenance_requests.filter((r) => r.status !== "resolved");
          if (active.length === 0) {
            await sendText(phone, "You have no active maintenance requests. Need to submit one?");
          } else {
            const statusList = active.map((r) =>
              `- ${r.id}: ${r.description}\n  Status: ${r.status} | Priority: ${r.urgency} | ETA: ${r.eta}`
            ).join("\n\n");
            await sendText(phone, `Your active requests:\n\n${statusList}`);
          }
        } else {
          await sendText(phone, classification.response_text || [
            `Hi ${tenantInfo.name}! I can help with:`,
            ``,
            `- *Maintenance*: Report any issues in your unit`,
            `- *Rent*: Check your payment status`,
            `- *Status*: Track your maintenance requests`,
            ``,
            `Just tell me what you need!`,
          ].join("\n"));
        }
        break;
      }
    }
  } catch (err) {
    console.error("Property management error:", err);
    await sendText(phone, "Sorry, I'm having trouble processing your request. Please call the office at (555) 123-4567.");
  }
});

// ── Resolve maintenance request (called by staff) ──────────────────────

app.post("/resolve/:requestId", async (req, res) => {
  const { requestId } = req.params;
  const { phone, notes } = req.body;

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const request = (metadata.maintenance_requests || []).find((r) => r.id === requestId);

    if (!request) return res.status(404).json({ error: "Request not found" });

    request.status = "resolved";
    request.resolved_at = new Date().toISOString();
    if (notes) request.resolution_notes = notes;

    await updateConversation(phone, metadata);

    await sendText(phone, [
      `Your maintenance request ${requestId} has been resolved!`,
      ``,
      `Issue: ${request.description}`,
      notes ? `Resolution: ${notes}` : "",
      ``,
      `If the issue persists or wasn't fully resolved, just let me know.`,
    ].filter(Boolean).join("\n"), { action: "request_resolved", request_id: requestId });

    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Broadcast announcement ─────────────────────────────────────────────

app.post("/announce", async (req, res) => {
  const { message, announcement_id } = req.body;

  try {
    const results = [];
    for (const [phone, tenant] of TENANTS) {
      const result = await sendText(phone, [
        `BUILDING ANNOUNCEMENT`,
        ``,
        message,
        ``,
        `— Sunset Apartments Management`,
      ].join("\n"), { announcement_id });

      // Track in metadata
      const convoRes = await getConversation(phone);
      const metadata = convoRes.data?.metadata || {};
      metadata.announcements_received = metadata.announcements_received || [];
      metadata.announcements_received.push(announcement_id);
      await updateConversation(phone, metadata);

      results.push({ phone, tenant: tenant.name, sent: result.success });
    }

    res.json({ success: true, results });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Property management bot running on :3000"));
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
export MAINTENANCE_PHONE="5491155559000"  # Maintenance dispatch
export MANAGER_PHONE="5491155559999"      # Property manager (emergencies)

# 3. Start the server
node property-management.js

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

# 6. Send a building announcement (optional)
curl -X POST http://localhost:3000/announce \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Water will be shut off on April 5th from 9 AM to 12 PM for pipe maintenance.",
    "announcement_id": "2026-04-water-shutoff"
  }'
```

## Key Concepts

- **Urgency-based dispatch**: AI classifies urgency — emergencies get immediate dispatch, routine issues are scheduled
- **Maintenance type routing**: Requests go to the right specialist (plumber, electrician, HVAC, etc.)
- **Tenant lookup**: Phone number maps to tenant/unit info — no login needed
- **Rent integration**: Tenants can check their payment status instantly
- **Broadcast announcements**: `POST /announce` sends building-wide messages and tracks delivery
- **Request lifecycle**: Created -> dispatched -> resolved, with tenant notifications at each step
