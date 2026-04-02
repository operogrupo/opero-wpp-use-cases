# 18 — Hotel Concierge Bot

## Problem

Hotel guests expect instant service — room service orders, spa bookings, restaurant recommendations, late checkout requests — but the front desk is overwhelmed. Different requests need to go to different departments, and tracking everything across phone calls and in-person visits is chaotic.

## Solution

An AI concierge powered by Claude Sonnet that handles all guest requests via WhatsApp. Requests are classified by department and routed accordingly. Guest preferences and request history are tracked in conversation metadata for personalized service throughout their stay.

## Architecture

```
Guest: "Can I order a club sandwich to room 412?"
    |
    v
┌──────────────────────────────────────┐
│   Webhook Server                     │
│                                      │
│   1. Classify request by department: │
│      room_service | spa | restaurant │
│      | front_desk | activities       │
│      | general                       │
│   2. Process based on department     │
│   3. Update request log in metadata  │
│   4. Send confirmation to guest      │
│   5. Notify department staff         │
└──────────────────────────────────────┘
    |
    ├── Room Service → Menu + order + ETA
    ├── Spa → Availability + booking
    ├── Restaurant → Recommendations + reservations
    ├── Front Desk → Checkout, keys, amenities
    └── Activities → Local suggestions + booking
```

## Metadata Schema

```json
{
  "guest": {
    "name": "Mr. Anderson",
    "room": "412",
    "check_in": "2026-04-01",
    "check_out": "2026-04-05",
    "loyalty_tier": "gold",
    "preferences": {
      "dietary": ["no nuts"],
      "pillow_type": "firm",
      "newspaper": "none",
      "temperature": "cool"
    }
  },
  "requests": [
    {
      "id": "req_001",
      "department": "room_service",
      "description": "Club sandwich, no fries, diet coke",
      "status": "in_progress",
      "eta_minutes": 25,
      "created_at": "2026-04-02T10:00:00Z",
      "completed_at": null
    }
  ],
  "spa_bookings": [],
  "restaurant_reservations": [],
  "late_checkout_requested": false,
  "satisfaction_score": null
}
```

## Code

```javascript
// hotel-concierge.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// Staff notification phones (per department)
const DEPT_PHONES = {
  room_service: process.env.ROOM_SERVICE_PHONE,
  spa: process.env.SPA_PHONE,
  front_desk: process.env.FRONT_DESK_PHONE,
};

// ── Hotel Data ─────────────────────────────────────────────────────────

const ROOM_SERVICE_MENU = [
  { name: "Club Sandwich", price: 18, category: "sandwiches", prep_time: 20 },
  { name: "Caesar Salad", price: 15, category: "salads", prep_time: 15 },
  { name: "Margherita Pizza", price: 22, category: "mains", prep_time: 25 },
  { name: "Grilled Salmon", price: 32, category: "mains", prep_time: 30 },
  { name: "Wagyu Burger", price: 28, category: "mains", prep_time: 25 },
  { name: "Chocolate Lava Cake", price: 14, category: "desserts", prep_time: 20 },
  { name: "Fruit Platter", price: 16, category: "desserts", prep_time: 10 },
  { name: "Coffee", price: 5, category: "beverages", prep_time: 5 },
  { name: "Fresh Juice", price: 8, category: "beverages", prep_time: 5 },
  { name: "Bottle of Wine (House)", price: 35, category: "beverages", prep_time: 5 },
];

const SPA_SERVICES = [
  { name: "Swedish Massage", duration: 60, price: 120 },
  { name: "Deep Tissue Massage", duration: 90, price: 160 },
  { name: "Facial Treatment", duration: 45, price: 90 },
  { name: "Aromatherapy", duration: 60, price: 130 },
  { name: "Hot Stone Massage", duration: 75, price: 150 },
];

const SPA_HOURS = { open: "09:00", close: "21:00" };

const RESTAURANTS = [
  { name: "The Terrace", cuisine: "Mediterranean", dress_code: "smart casual", rating: 4.5 },
  { name: "Sakura", cuisine: "Japanese", dress_code: "casual", rating: 4.7 },
  { name: "Steakhouse 1890", cuisine: "Steakhouse", dress_code: "formal", rating: 4.3 },
  { name: "Poolside Grill", cuisine: "BBQ & Light bites", dress_code: "casual", rating: 4.1 },
];

const LOCAL_ACTIVITIES = [
  { name: "City Walking Tour", duration: "3 hours", price: 45, time: "10:00 AM daily" },
  { name: "Sunset Sailing", duration: "2 hours", price: 85, time: "5:00 PM daily" },
  { name: "Wine Tasting", duration: "2 hours", price: 65, time: "3:00 PM, Wed-Sun" },
  { name: "Museum Pass", duration: "full day", price: 25, time: "9:00 AM - 6:00 PM" },
  { name: "Bike Rental", duration: "per day", price: 30, time: "7:00 AM - 8:00 PM" },
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

async function getMessages(phone, limit = 15) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`);
}

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, { phone, text, metadata });
}

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── Intent Classification ──────────────────────────────────────────────

async function classifyRequest(messages, guestInfo) {
  const menuStr = ROOM_SERVICE_MENU.map((m) => `${m.name} ($${m.price})`).join(", ");
  const spaStr = SPA_SERVICES.map((s) => `${s.name} (${s.duration}min, $${s.price})`).join(", ");
  const restStr = RESTAURANTS.map((r) => `${r.name} (${r.cuisine})`).join(", ");
  const actStr = LOCAL_ACTIVITIES.map((a) => `${a.name} ($${a.price})`).join(", ");

  const guestCtx = guestInfo.name
    ? `Guest: ${guestInfo.name}, Room ${guestInfo.room}, Check-out: ${guestInfo.check_out}, Loyalty: ${guestInfo.loyalty_tier || "standard"}`
    : "Guest info not yet collected";

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: `You are the concierge at a luxury hotel. Be warm, professional, and proactive.

${guestCtx}
${guestInfo.preferences?.dietary ? `Dietary restrictions: ${guestInfo.preferences.dietary.join(", ")}` : ""}

Room Service Menu: ${menuStr}
Spa Services: ${spaStr} (Hours: ${SPA_HOURS.open}-${SPA_HOURS.close})
Restaurants: ${restStr}
Local Activities: ${actStr}

Classify the request and respond. Return your response in this format:

DEPARTMENT: room_service | spa | restaurant | front_desk | activities | general
ITEMS: [list of items/services requested, or empty]
ROOM: room number if mentioned
GUEST_NAME: name if mentioned
---
Your friendly response to the guest here. Include prices, ETAs, and recommendations.
Be specific and helpful. Anticipate needs.`,
    messages,
  });

  const text = response.content[0].text;

  // Parse structured header
  const deptMatch = text.match(/DEPARTMENT:\s*(\w+)/);
  const itemsMatch = text.match(/ITEMS:\s*\[([^\]]*)\]/);
  const roomMatch = text.match(/ROOM:\s*(\S+)/);
  const nameMatch = text.match(/GUEST_NAME:\s*(.+)/);
  const responseSplit = text.split("---");

  return {
    department: deptMatch?.[1] || "general",
    items: itemsMatch?.[1]?.split(",").map((s) => s.trim()).filter(Boolean) || [],
    room: roomMatch?.[1] || guestInfo.room || null,
    guest_name: nameMatch?.[1]?.trim() || null,
    response_text: (responseSplit[1] || text).trim(),
  };
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

    // Load guest data
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const guest = metadata.guest || {};
    const requests = metadata.requests || [];

    // Build conversation context
    const messagesRes = await getMessages(phone, 15);
    const history = (messagesRes.data || [])
      .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
      .map((msg) => ({
        role: msg.direction === "inbound" ? "user" : "assistant",
        content: typeof msg.content === "string" ? msg.content : msg.content?.text || JSON.stringify(msg.content),
      }));

    // Classify and respond
    const result = await classifyRequest(history, guest);

    // Update guest info if new data
    if (result.guest_name && !guest.name) guest.name = result.guest_name;
    if (result.room && !guest.room) guest.room = result.room;

    // Log the request
    const requestId = `req_${Date.now().toString(36)}`;
    if (result.department !== "general") {
      const newRequest = {
        id: requestId,
        department: result.department,
        description: result.items.length > 0 ? result.items.join(", ") : messageText.substring(0, 100),
        status: "received",
        created_at: new Date().toISOString(),
        completed_at: null,
      };

      // Add ETA for room service
      if (result.department === "room_service" && result.items.length > 0) {
        const maxPrepTime = Math.max(
          ...result.items.map((item) => {
            const menuItem = ROOM_SERVICE_MENU.find((m) =>
              m.name.toLowerCase().includes(item.toLowerCase())
            );
            return menuItem?.prep_time || 20;
          })
        );
        newRequest.eta_minutes = maxPrepTime + 10; // prep + delivery
        newRequest.status = "in_progress";
      }

      requests.push(newRequest);

      // Notify department staff
      const deptPhone = DEPT_PHONES[result.department];
      if (deptPhone) {
        await sendText(deptPhone, [
          `New ${result.department.replace("_", " ")} request`,
          `Guest: ${guest.name || "Unknown"} | Room: ${guest.room || "?"}`,
          `Request: ${newRequest.description}`,
          `ID: ${requestId}`,
        ].join("\n"), {
          alert_type: "department_request",
          request_id: requestId,
          department: result.department,
        });
      }
    }

    // Handle special cases
    if (result.department === "front_desk" && messageText.toLowerCase().includes("late checkout")) {
      metadata.late_checkout_requested = true;
    }

    // Save metadata
    metadata.guest = guest;
    metadata.requests = requests;
    await updateConversation(phone, metadata);

    // Send response to guest
    await sendText(phone, result.response_text, {
      department: result.department,
      request_id: result.department !== "general" ? requestId : null,
    });
  } catch (err) {
    console.error("Concierge error:", err);
    await sendText(phone, "I apologize for the inconvenience. Let me connect you with the front desk. You can also dial 0 from your room phone.");
  }
});

// ── Request status update (called by staff) ────────────────────────────

app.post("/request/:requestId/complete", async (req, res) => {
  const { requestId } = req.params;
  const { phone } = req.body;

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const requests = metadata.requests || [];

    const request = requests.find((r) => r.id === requestId);
    if (!request) return res.status(404).json({ error: "Request not found" });

    request.status = "completed";
    request.completed_at = new Date().toISOString();
    metadata.requests = requests;
    await updateConversation(phone, metadata);

    // Notify guest
    await sendText(phone, `Your ${request.department.replace("_", " ")} request has been completed. Is there anything else I can help with?`, {
      request_id: requestId,
      action: "request_completed",
    });

    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Hotel concierge bot running on :3000"));
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
export ROOM_SERVICE_PHONE="5491155550001"  # Room service staff
export SPA_PHONE="5491155550002"            # Spa staff
export FRONT_DESK_PHONE="5491155550003"     # Front desk

# 3. Start the server
node hotel-concierge.js

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

- **Multi-department routing**: Requests automatically go to the right department via `metadata.department`
- **Staff notifications**: Department staff receive WhatsApp alerts for new requests
- **Guest profile building**: Name, room, preferences accumulate in metadata across the stay
- **Request tracking**: Every request gets an ID and status (received/in_progress/completed)
- **Rich context**: Claude Sonnet uses conversation history for natural follow-ups ("same as yesterday" works)
- **Proactive service**: The AI anticipates needs (dietary restrictions, loyalty perks) using guest metadata
