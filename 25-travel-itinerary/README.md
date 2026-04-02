# 25 — AI Travel Itinerary Assistant

## Problem

Planning a trip is time-consuming. Travelers spend hours researching destinations, comparing hotels, finding restaurants, and building day-by-day itineraries. When plans change mid-trip, replanning is stressful. People want a knowledgeable travel advisor available instantly, not a static PDF itinerary.

## Solution

An AI travel assistant powered by Claude Sonnet that plans personalized itineraries based on preferences (budget, interests, dates, travel style). It builds day-by-day plans with activities, restaurants, and hotels, sends daily itinerary updates during the trip, and handles real-time changes and questions about destinations.

## Architecture

```
Traveler: "Plan a 5-day trip to Tokyo, budget $3000, I love food and temples"
    |
    v
┌──────────────────────────────────────┐
│   Webhook Server                     │
│                                      │
│   1. Classify intent:                │
│      plan_trip | modify | daily_plan │
│      | question | feedback           │
│   2. For planning:                   │
│      - Collect preferences           │
│      - Generate full itinerary       │
│      - Store in metadata             │
│   3. For active trips:               │
│      - Send daily morning briefing   │
│      - Handle changes                │
│      - Answer destination questions  │
│   4. Track everything in metadata    │
└──────────────────────────────────────┘
    |
    ├── Plan: Preferences → Full itinerary with daily breakdown
    ├── Daily: Morning briefing with today's activities
    ├── Modify: Change activities, restaurants, timing
    └── Question: Real-time answers about the destination
```

## Metadata Schema

```json
{
  "traveler": {
    "name": "Alex",
    "home_city": "Buenos Aires"
  },
  "trip": {
    "id": "TRIP-20260415",
    "destination": "Tokyo, Japan",
    "start_date": "2026-04-15",
    "end_date": "2026-04-19",
    "budget": {
      "total": 3000,
      "currency": "USD",
      "per_day": 600,
      "spent_estimate": 1200
    },
    "preferences": {
      "interests": ["food", "temples", "culture"],
      "pace": "moderate",
      "accommodation": "mid-range",
      "dietary": [],
      "travel_style": "mix of tourist and local"
    },
    "status": "active",
    "current_day": 3
  },
  "itinerary": [
    {
      "day": 1,
      "date": "2026-04-15",
      "title": "Arrival & Asakusa",
      "activities": [
        {
          "time": "14:00",
          "name": "Check into hotel",
          "location": "Shinjuku",
          "details": "Hotel Gracery Shinjuku",
          "cost_estimate": 120,
          "status": "completed"
        },
        {
          "time": "16:00",
          "name": "Senso-ji Temple",
          "location": "Asakusa",
          "details": "Tokyo's oldest temple, walk through Nakamise-dori for street food",
          "cost_estimate": 0,
          "status": "completed"
        },
        {
          "time": "19:00",
          "name": "Dinner at Hoppy Street",
          "location": "Asakusa",
          "details": "Yakitori and beer at the iconic izakaya alley",
          "cost_estimate": 25,
          "status": "completed"
        }
      ],
      "daily_tip": "Get a Suica card at the airport for easy transit",
      "weather": "Sunny, 18C",
      "briefing_sent": true
    }
  ],
  "modifications": [
    {
      "day": 2,
      "original": "Tsukiji Outer Market for breakfast",
      "changed_to": "Toyosu Market early morning tour",
      "reason": "Traveler wanted more authentic fish market experience"
    }
  ],
  "questions_asked": 3
}
```

## Code

```javascript
// travel-itinerary.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

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

async function sendLocation(phone, lat, lng, name, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/location`, {
    phone, latitude: lat, longitude: lng, name, metadata,
  });
}

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── AI Functions ───────────────────────────────────────────────────────

async function processMessage(messages, metadata) {
  const trip = metadata.trip || {};
  const itinerary = metadata.itinerary || [];
  const traveler = metadata.traveler || {};

  let itinerarySummary = "No itinerary planned yet.";
  if (itinerary.length > 0) {
    itinerarySummary = itinerary.map((day) => {
      const activities = day.activities.map((a) => `  ${a.time} ${a.name} (${a.location})`).join("\n");
      return `Day ${day.day} (${day.date}): ${day.title}\n${activities}`;
    }).join("\n\n");
  }

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    system: `You are an expert travel advisor and itinerary planner. You have deep knowledge of destinations worldwide — local restaurants, hidden gems, cultural tips, practical logistics.

Traveler: ${JSON.stringify(traveler)}
Trip details: ${JSON.stringify(trip)}
Current itinerary:
${itinerarySummary}

Today's date: ${new Date().toISOString().split("T")[0]}

CAPABILITIES:
1. PLAN TRIP: Create a full day-by-day itinerary with specific places, times, and cost estimates
2. MODIFY: Change activities, add/remove days, adjust based on feedback
3. DAILY BRIEFING: Generate morning briefing for the current day
4. ANSWER QUESTIONS: Provide local knowledge about the destination
5. COLLECT PREFERENCES: Gather trip details (destination, dates, budget, interests)

When creating/modifying an itinerary, include your plan in this JSON format at the end of your response:

{
  "intent": "plan_trip | modify | daily_briefing | question | collect_preferences",
  "traveler_update": { "name": null, "home_city": null },
  "trip_update": {
    "destination": null,
    "start_date": null,
    "end_date": null,
    "budget_total": null,
    "budget_currency": "USD",
    "interests": [],
    "pace": null,
    "accommodation": null,
    "travel_style": null
  },
  "itinerary": [
    {
      "day": 1,
      "date": "YYYY-MM-DD",
      "title": "Day title",
      "activities": [
        {
          "time": "HH:MM",
          "name": "Activity name",
          "location": "Area/neighborhood",
          "details": "Specific place, tips, what to expect",
          "cost_estimate": 0
        }
      ],
      "daily_tip": "Practical tip for the day"
    }
  ],
  "modification": { "day": null, "original": null, "changed_to": null, "reason": null }
}

RULES:
- Be specific: real restaurant names, actual places, accurate costs
- Consider travel time between activities
- Mix popular spots with off-the-beaten-path gems
- Respect budget constraints
- Adapt to the traveler's pace (relaxed vs packed)
- Include practical tips (transit, tipping, local customs)
- For food recommendations, consider dietary restrictions`,
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
    } catch { /* use text as-is */ }
  }

  return { text: cleanText, ...parsed };
}

// ── Format Daily Briefing ──────────────────────────────────────────────

function formatDayBriefing(day) {
  const lines = [
    `Good morning! Here's your plan for today:`,
    ``,
    `*Day ${day.day}: ${day.title}*`,
    ``,
  ];

  for (const activity of day.activities) {
    lines.push(`${activity.time} — *${activity.name}*`);
    lines.push(`  ${activity.location}`);
    if (activity.details) lines.push(`  ${activity.details}`);
    if (activity.cost_estimate > 0) lines.push(`  Est. cost: $${activity.cost_estimate}`);
    lines.push("");
  }

  const dailyCost = day.activities.reduce((sum, a) => sum + (a.cost_estimate || 0), 0);
  lines.push(`Estimated cost today: ~$${dailyCost}`);

  if (day.daily_tip) {
    lines.push("");
    lines.push(`Tip: ${day.daily_tip}`);
  }

  lines.push("");
  lines.push("Want to change anything? Just tell me!");

  return lines.join("\n");
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
    metadata.traveler = metadata.traveler || {};
    metadata.trip = metadata.trip || {};
    metadata.itinerary = metadata.itinerary || [];
    metadata.modifications = metadata.modifications || [];
    metadata.questions_asked = metadata.questions_asked || 0;

    // Build conversation history
    const messagesRes = await getMessages(phone, 25);
    const history = (messagesRes.data || [])
      .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
      .map((msg) => ({
        role: msg.direction === "inbound" ? "user" : "assistant",
        content: typeof msg.content === "string" ? msg.content : msg.content?.text || JSON.stringify(msg.content),
      }));

    // Process with AI
    const result = await processMessage(history, metadata);

    // Update traveler info
    if (result.traveler_update) {
      for (const [key, value] of Object.entries(result.traveler_update)) {
        if (value) metadata.traveler[key] = value;
      }
    }

    // Update trip details
    if (result.trip_update) {
      const tu = result.trip_update;
      if (tu.destination) metadata.trip.destination = tu.destination;
      if (tu.start_date) metadata.trip.start_date = tu.start_date;
      if (tu.end_date) metadata.trip.end_date = tu.end_date;
      if (tu.budget_total) {
        const days = metadata.trip.start_date && metadata.trip.end_date
          ? Math.ceil((new Date(metadata.trip.end_date) - new Date(metadata.trip.start_date)) / (1000 * 60 * 60 * 24)) + 1
          : 1;
        metadata.trip.budget = {
          total: tu.budget_total,
          currency: tu.budget_currency || "USD",
          per_day: Math.round(tu.budget_total / days),
          spent_estimate: 0,
        };
      }
      if (tu.interests?.length > 0) {
        metadata.trip.preferences = metadata.trip.preferences || {};
        metadata.trip.preferences.interests = tu.interests;
      }
      if (tu.pace) {
        metadata.trip.preferences = metadata.trip.preferences || {};
        metadata.trip.preferences.pace = tu.pace;
      }
      if (tu.accommodation) {
        metadata.trip.preferences = metadata.trip.preferences || {};
        metadata.trip.preferences.accommodation = tu.accommodation;
      }
      if (tu.travel_style) {
        metadata.trip.preferences = metadata.trip.preferences || {};
        metadata.trip.preferences.travel_style = tu.travel_style;
      }
    }

    // Update itinerary
    if (result.itinerary?.length > 0) {
      // Replace or merge itinerary days
      for (const newDay of result.itinerary) {
        const existingIdx = metadata.itinerary.findIndex((d) => d.day === newDay.day);
        if (existingIdx >= 0) {
          metadata.itinerary[existingIdx] = { ...newDay, briefing_sent: false };
        } else {
          metadata.itinerary.push({ ...newDay, briefing_sent: false });
        }
      }
      metadata.itinerary.sort((a, b) => a.day - b.day);

      // Set trip status
      metadata.trip.status = "planned";
      metadata.trip.id = metadata.trip.id || `TRIP-${metadata.trip.start_date?.replace(/-/g, "") || Date.now().toString(36)}`;

      // Calculate total estimated cost
      const totalEstimated = metadata.itinerary.reduce(
        (sum, day) => sum + day.activities.reduce((s, a) => s + (a.cost_estimate || 0), 0),
        0
      );
      if (metadata.trip.budget) {
        metadata.trip.budget.spent_estimate = totalEstimated;
      }
    }

    // Track modifications
    if (result.modification?.day) {
      metadata.modifications.push({
        ...result.modification,
        modified_at: new Date().toISOString(),
      });
    }

    // Track questions
    if (result.intent === "question") {
      metadata.questions_asked++;
    }

    await updateConversation(phone, metadata);

    // Send response (may be long for itineraries, so split if needed)
    const responseText = result.text;
    if (responseText.length > 4000) {
      // Split into chunks at paragraph boundaries
      const chunks = [];
      let current = "";
      for (const line of responseText.split("\n")) {
        if (current.length + line.length > 3500 && current.length > 0) {
          chunks.push(current.trim());
          current = "";
        }
        current += line + "\n";
      }
      if (current.trim()) chunks.push(current.trim());

      for (const chunk of chunks) {
        await sendText(phone, chunk, { intent: result.intent });
      }
    } else {
      await sendText(phone, responseText, { intent: result.intent });
    }
  } catch (err) {
    console.error("Travel bot error:", err);
    await sendText(phone, "Sorry, I had trouble with that. Could you try rephrasing?");
  }
});

// ── Daily Briefing Sender (run via cron) ───────────────────────────────

app.post("/send-daily-briefings", async (req, res) => {
  // In production, iterate over all active trips
  // For demo, accept a phone number
  const { phone } = req.body;

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const trip = metadata.trip || {};
    const itinerary = metadata.itinerary || [];

    if (trip.status !== "active" && trip.status !== "planned") {
      return res.json({ success: false, message: "No active trip" });
    }

    // Find today's day in the itinerary
    const today = new Date().toISOString().split("T")[0];
    const todayPlan = itinerary.find((d) => d.date === today);

    if (!todayPlan) {
      return res.json({ success: false, message: "No plan for today" });
    }

    if (todayPlan.briefing_sent) {
      return res.json({ success: false, message: "Briefing already sent" });
    }

    // Send the briefing
    await sendText(phone, formatDayBriefing(todayPlan), {
      action: "daily_briefing",
      day: todayPlan.day,
    });

    // Mark as sent
    todayPlan.briefing_sent = true;
    metadata.trip.current_day = todayPlan.day;
    metadata.trip.status = "active";
    await updateConversation(phone, metadata);

    res.json({ success: true, day: todayPlan.day });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Mark trip complete ─────────────────────────────────────────────────

app.post("/trip/complete", async (req, res) => {
  const { phone } = req.body;

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};

    metadata.trip.status = "completed";
    await updateConversation(phone, metadata);

    await sendText(phone, [
      `Your trip to ${metadata.trip.destination} is complete!`,
      ``,
      `Hope you had an amazing time. If you'd like to plan another trip, just tell me where you want to go next!`,
      ``,
      `Safe travels home! `,
    ].join("\n"), { action: "trip_completed" });

    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Travel itinerary assistant running on :3000"));
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
node travel-itinerary.js

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

# 6. Send daily briefing (trigger via cron or manually)
curl -X POST http://localhost:3000/send-daily-briefings \
  -H "Content-Type: application/json" \
  -d '{ "phone": "5491155550001" }'
```

## Key Concepts

- **Preference-driven planning**: Budget, interests, pace, accommodation level, and travel style shape the itinerary
- **Day-by-day structure**: Each day has timed activities with locations, details, and cost estimates
- **Daily briefings**: Morning messages with the day's plan, sent via cron or API trigger
- **Real-time modifications**: "Change dinner tonight to sushi" updates the itinerary mid-trip
- **Budget tracking**: Running cost estimate compared against total budget
- **Destination knowledge**: Claude Sonnet provides specific, real restaurant names, hidden gems, and local tips
- **Long message handling**: Itineraries are automatically split into readable chunks for WhatsApp
