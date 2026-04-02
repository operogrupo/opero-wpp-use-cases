# 44 - Wedding Planner Assistant

AI-powered wedding planner assistant via WhatsApp. Manages guest lists, vendor contacts, timelines, budgets, task reminders, and RSVPs -- all tracked in conversation metadata.

## Problem

A wedding planning company juggles 15 active weddings simultaneously. Each couple constantly messages asking about vendor contacts, budget status, or guest counts. The planning team spends 3 hours daily on repetitive status updates and reminder follow-ups. Couples feel anxious because they can't get instant answers outside business hours. RSVPs trickle in with no centralized tracking.

## Solution

Deploy a wedding planner assistant that:
- Maintains the full wedding state in conversation metadata (guest list, vendors, budget, timeline)
- Answers questions about any aspect of the wedding instantly
- Sends task reminders as deadlines approach
- Collects and tracks RSVPs from a shareable link or direct messages
- Manages budget with line items, payments, and remaining balance
- Uses AI to provide suggestions and flag potential issues

## Architecture

```
Couple / Guest sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
| wpp-api.opero.so |                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |  Claude Sonnet   |
         |                                 | (Anthropic API)  |
         |                                 +------------------+
         |                                          |
         |    GET/PUT metadata, POST /messages/text |
         +------------------------------------------+
         |
         v
+------------------+
| Cron: Deadline    |
| Reminders         |
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

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
  });
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

async function getMessages(phone, limit = 20) {
  try {
    const res = await operoFetch(
      `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`
    );
    return res.data || [];
  } catch {
    return [];
  }
}

async function listConversations() {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=500`);
    return res.data || [];
  } catch {
    return [];
  }
}

// -- Wedding data helpers --
function initializeWedding(metadata) {
  return {
    ...metadata,
    couple_names: metadata.couple_names || '',
    wedding_date: metadata.wedding_date || '',
    venue: metadata.venue || '',
    guests: metadata.guests || [],
    vendors: metadata.vendors || [],
    budget: metadata.budget || { total: 0, items: [], payments: [] },
    timeline: metadata.timeline || [],
    tasks: metadata.tasks || [],
    rsvp_stats: metadata.rsvp_stats || { confirmed: 0, declined: 0, pending: 0 },
    created_at: metadata.created_at || new Date().toISOString(),
  };
}

function daysUntil(dateStr) {
  const target = new Date(dateStr);
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  target.setHours(0, 0, 0, 0);
  return Math.ceil((target - today) / (1000 * 60 * 60 * 24));
}

function calculateBudget(budget) {
  const totalBudgeted = budget.items.reduce((sum, i) => sum + i.estimated, 0);
  const totalPaid = budget.payments.reduce((sum, p) => sum + p.amount, 0);
  return {
    total_budget: budget.total,
    total_budgeted: totalBudgeted,
    total_paid: totalPaid,
    remaining: budget.total - totalPaid,
    unallocated: budget.total - totalBudgeted,
  };
}

function formatGuestSummary(guests) {
  const confirmed = guests.filter(g => g.rsvp === 'confirmed');
  const declined = guests.filter(g => g.rsvp === 'declined');
  const pending = guests.filter(g => !g.rsvp || g.rsvp === 'pending');

  return {
    total: guests.length,
    confirmed: confirmed.length,
    declined: declined.length,
    pending: pending.length,
    plus_ones: confirmed.reduce((sum, g) => sum + (g.plus_ones || 0), 0),
    dietary: confirmed.filter(g => g.dietary).map(g => `${g.name}: ${g.dietary}`),
  };
}

// -- AI integration --
function buildSystemPrompt(metadata) {
  const wedding = initializeWedding(metadata);
  const guestSummary = formatGuestSummary(wedding.guests);
  const budgetSummary = calculateBudget(wedding.budget);
  const daysLeft = wedding.wedding_date ? daysUntil(wedding.wedding_date) : 'Not set';

  const pendingTasks = wedding.tasks.filter(t => t.status !== 'done');
  const overdueTasks = pendingTasks.filter(t => t.due_date && daysUntil(t.due_date) < 0);

  return `You are a wedding planner assistant for ${wedding.couple_names || 'a couple'}.

Wedding date: ${wedding.wedding_date || 'Not set'} (${daysLeft} days away)
Venue: ${wedding.venue || 'Not set'}

Guest list: ${guestSummary.total} total (${guestSummary.confirmed} confirmed, ${guestSummary.declined} declined, ${guestSummary.pending} pending)
Expected attendees (with +1s): ${guestSummary.confirmed + guestSummary.plus_ones}

Budget: $${budgetSummary.total_budget} total | $${budgetSummary.total_paid} paid | $${budgetSummary.remaining} remaining | $${budgetSummary.unallocated} unallocated

Vendors: ${wedding.vendors.length > 0 ? wedding.vendors.map(v => `${v.category}: ${v.name} ($${v.price})`).join(', ') : 'None added'}

Pending tasks: ${pendingTasks.length} (${overdueTasks.length} overdue)
${pendingTasks.slice(0, 5).map(t => `- ${t.title} (due: ${t.due_date || 'no date'}) [${t.status}]`).join('\n')}

Use these action tags in your responses:
- [ADD_GUEST:NAME:PHONE:TABLE] - Add a guest (phone and table optional, use "none" if not provided)
- [RSVP:GUEST_NAME:STATUS] - Update RSVP (confirmed/declined)
- [RSVP:GUEST_NAME:STATUS:PLUS_ONES:DIETARY] - RSVP with details
- [ADD_VENDOR:CATEGORY:NAME:PHONE:PRICE] - Add vendor
- [ADD_BUDGET_ITEM:CATEGORY:DESCRIPTION:AMOUNT] - Add budget line item
- [ADD_PAYMENT:VENDOR:AMOUNT:NOTE] - Record a payment
- [ADD_TASK:TITLE:DUE_DATE:PRIORITY] - Add a task (priority: high/medium/low)
- [COMPLETE_TASK:TASK_INDEX] - Mark task as done (0-based index)
- [SET_DATE:YYYY-MM-DD] - Set wedding date
- [SET_VENUE:VENUE_NAME] - Set venue
- [SET_BUDGET:AMOUNT] - Set total budget

Be organized, proactive, and detail-oriented. Flag any concerns (budget overrun, missing RSVPs, approaching deadlines).
Keep responses concise unless asked for a detailed breakdown.`;
}

async function generateReply(phone, message, metadata) {
  const history = await getMessages(phone, 10);
  const messages = history
    .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    .map(m => ({
      role: m.from_me ? 'assistant' : 'user',
      content: m.text || m.content?.text || '',
    }))
    .filter(m => m.content);

  messages.push({ role: 'user', content: message });

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1500,
    system: buildSystemPrompt(metadata),
    messages: messages.slice(-20),
  });

  return response.content[0].text;
}

// -- Action processor --
async function processActions(phone, aiResponse, metadata) {
  let finalResponse = aiResponse;
  const wedding = initializeWedding(metadata);

  // Add guest
  const guestMatches = aiResponse.matchAll(/\[ADD_GUEST:([^:]+):([^:]*):([^\]]*)\]/g);
  for (const match of guestMatches) {
    wedding.guests.push({
      name: match[1].trim(),
      phone: match[2].trim() !== 'none' ? match[2].trim() : null,
      table: match[3].trim() !== 'none' ? match[3].trim() : null,
      rsvp: 'pending',
      added_at: new Date().toISOString(),
    });
    finalResponse = finalResponse.replace(match[0], '');
  }

  // RSVP update
  const rsvpMatches = aiResponse.matchAll(/\[RSVP:([^:]+):([^:\]]+)(?::(\d+))?(?::([^\]]*))?\]/g);
  for (const match of rsvpMatches) {
    const guestName = match[1].trim().toLowerCase();
    const status = match[2].trim().toLowerCase();
    const plusOnes = match[3] ? parseInt(match[3]) : 0;
    const dietary = match[4]?.trim() || null;

    const guest = wedding.guests.find(g => g.name.toLowerCase() === guestName);
    if (guest) {
      guest.rsvp = status;
      guest.plus_ones = plusOnes;
      if (dietary) guest.dietary = dietary;
      guest.rsvp_date = new Date().toISOString();
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Recalculate RSVP stats
  wedding.rsvp_stats = {
    confirmed: wedding.guests.filter(g => g.rsvp === 'confirmed').length,
    declined: wedding.guests.filter(g => g.rsvp === 'declined').length,
    pending: wedding.guests.filter(g => !g.rsvp || g.rsvp === 'pending').length,
  };

  // Add vendor
  const vendorMatches = aiResponse.matchAll(/\[ADD_VENDOR:([^:]+):([^:]+):([^:]*):([^\]]+)\]/g);
  for (const match of vendorMatches) {
    wedding.vendors.push({
      category: match[1].trim(),
      name: match[2].trim(),
      phone: match[3].trim() || null,
      price: parseFloat(match[4]),
      added_at: new Date().toISOString(),
    });
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Budget item
  const budgetMatches = aiResponse.matchAll(/\[ADD_BUDGET_ITEM:([^:]+):([^:]+):([^\]]+)\]/g);
  for (const match of budgetMatches) {
    wedding.budget.items.push({
      category: match[1].trim(),
      description: match[2].trim(),
      estimated: parseFloat(match[3]),
      added_at: new Date().toISOString(),
    });
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Payment
  const paymentMatches = aiResponse.matchAll(/\[ADD_PAYMENT:([^:]+):([^:]+):([^\]]*)\]/g);
  for (const match of paymentMatches) {
    wedding.budget.payments.push({
      vendor: match[1].trim(),
      amount: parseFloat(match[2]),
      note: match[3].trim() || '',
      date: new Date().toISOString(),
    });
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Task
  const taskMatches = aiResponse.matchAll(/\[ADD_TASK:([^:]+):([^:]+):([^\]]+)\]/g);
  for (const match of taskMatches) {
    wedding.tasks.push({
      title: match[1].trim(),
      due_date: match[2].trim(),
      priority: match[3].trim(),
      status: 'pending',
      created_at: new Date().toISOString(),
    });
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Complete task
  const completeMatches = aiResponse.matchAll(/\[COMPLETE_TASK:(\d+)\]/g);
  for (const match of completeMatches) {
    const idx = parseInt(match[1]);
    const pendingTasks = wedding.tasks.filter(t => t.status !== 'done');
    if (pendingTasks[idx]) {
      pendingTasks[idx].status = 'done';
      pendingTasks[idx].completed_at = new Date().toISOString();
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Set wedding date
  const dateMatch = aiResponse.match(/\[SET_DATE:(\d{4}-\d{2}-\d{2})\]/);
  if (dateMatch) {
    wedding.wedding_date = dateMatch[1];
    finalResponse = finalResponse.replace(dateMatch[0], '');
  }

  // Set venue
  const venueMatch = aiResponse.match(/\[SET_VENUE:([^\]]+)\]/);
  if (venueMatch) {
    wedding.venue = venueMatch[1].trim();
    finalResponse = finalResponse.replace(venueMatch[0], '');
  }

  // Set budget
  const budgetMatch = aiResponse.match(/\[SET_BUDGET:([^\]]+)\]/);
  if (budgetMatch) {
    wedding.budget.total = parseFloat(budgetMatch[1]);
    finalResponse = finalResponse.replace(budgetMatch[0], '');
  }

  // Save
  wedding.last_interaction = new Date().toISOString();
  await updateConversationMetadata(phone, wedding);

  // Clean remaining tags
  finalResponse = finalResponse.replace(/\[\w+(?::[^\]]+)*\]/g, '').trim();

  return finalResponse;
}

// -- Deadline reminder cron --
async function checkDeadlines() {
  console.log(`[${new Date().toISOString()}] Checking deadlines...`);
  const conversations = await listConversations();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.tasks) continue;

    const phone = conv.phone || conv.jid;
    const pendingTasks = metadata.tasks.filter(t => t.status !== 'done' && t.due_date);

    for (const task of pendingTasks) {
      const days = daysUntil(task.due_date);

      if (days === 3 || days === 1 || days === 0) {
        const urgency = days === 0 ? 'TODAY' : `in ${days} day(s)`;
        await sendText(phone,
          `Reminder: *${task.title}* is due ${urgency} (${task.due_date}). Priority: ${task.priority}.`,
          { agent: 'wedding-planner', type: 'task-reminder', task: task.title, days_until: days }
        );
      }
    }

    // Wedding countdown reminders at key milestones
    if (metadata.wedding_date) {
      const weddingDays = daysUntil(metadata.wedding_date);
      if ([90, 60, 30, 14, 7, 3, 1].includes(weddingDays)) {
        await sendText(phone,
          `${weddingDays} days until the wedding! ${metadata.couple_names ? `${metadata.couple_names}, ` : ''}here's your status:\n\nGuests confirmed: ${metadata.rsvp_stats?.confirmed || 0}\nPending RSVPs: ${metadata.rsvp_stats?.pending || 0}\nPending tasks: ${metadata.tasks.filter(t => t.status !== 'done').length}`,
          { agent: 'wedding-planner', type: 'countdown', days_until_wedding: weddingDays }
        );
      }
    }
  }
}

// -- RSVP endpoint (for sharing with guests) --
app.post('/rsvp', async (req, res) => {
  const { wedding_phone, guest_name, status, plus_ones, dietary } = req.body;

  try {
    const metadata = await getConversationMetadata(wedding_phone);
    if (!metadata.guests) {
      return res.status(404).json({ error: 'Wedding not found' });
    }

    const guest = metadata.guests.find(
      g => g.name.toLowerCase() === guest_name.toLowerCase()
    );
    if (!guest) {
      return res.status(404).json({ error: 'Guest not found on the list' });
    }

    guest.rsvp = status;
    guest.plus_ones = plus_ones || 0;
    if (dietary) guest.dietary = dietary;
    guest.rsvp_date = new Date().toISOString();

    metadata.rsvp_stats = {
      confirmed: metadata.guests.filter(g => g.rsvp === 'confirmed').length,
      declined: metadata.guests.filter(g => g.rsvp === 'declined').length,
      pending: metadata.guests.filter(g => !g.rsvp || g.rsvp === 'pending').length,
    };

    await updateConversationMetadata(wedding_phone, metadata);

    // Notify the couple
    await sendText(wedding_phone,
      `RSVP update: *${guest_name}* has ${status}${plus_ones > 0 ? ` (+${plus_ones})` : ''}.${dietary ? ` Dietary: ${dietary}` : ''}\n\nTotal confirmed: ${metadata.rsvp_stats.confirmed}/${metadata.guests.length}`,
      { agent: 'wedding-planner', type: 'rsvp-notification', guest: guest_name }
    );

    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';
  if (!messageText.trim()) return;

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    await sendTyping(phone);

    const metadata = await getConversationMetadata(phone);
    const aiResponse = await generateReply(phone, messageText, metadata);
    const finalResponse = await processActions(phone, aiResponse, metadata);

    const typingDelay = Math.min(finalResponse.length * 25, 3000);
    await new Promise(resolve => setTimeout(resolve, typingDelay));

    await sendText(phone, finalResponse, {
      agent: 'wedding-planner',
      model: 'claude-sonnet-4-20250514',
      guests_count: metadata.guests?.length || 0,
      generated_at: new Date().toISOString(),
    });
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Sorry, I ran into an issue. Please try again!', {
        agent: 'wedding-planner', error: true, error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback failed:', fallbackErr.message);
    }
  }
});

// -- Admin: trigger deadline check --
app.post('/admin/check-deadlines', async (req, res) => {
  try {
    await checkDeadlines();
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Wedding planner bot listening on port ${PORT}`);
});
```

## Metadata Schema

Each couple's conversation stores the full wedding state:

```json
{
  "couple_names": "Ana & Carlos",
  "wedding_date": "2026-11-15",
  "venue": "Estancia La Linda, Pilar",
  "guests": [
    {
      "name": "Roberto Garcia",
      "phone": "5491155551234",
      "table": "Family",
      "rsvp": "confirmed",
      "plus_ones": 1,
      "dietary": "vegetarian",
      "rsvp_date": "2026-04-01T10:00:00.000Z",
      "added_at": "2026-03-01T09:00:00.000Z"
    }
  ],
  "vendors": [
    {
      "category": "Photography",
      "name": "Studio Luz",
      "phone": "5491155559999",
      "price": 2500,
      "added_at": "2026-03-10T09:00:00.000Z"
    }
  ],
  "budget": {
    "total": 30000,
    "items": [
      { "category": "Venue", "description": "Venue rental + catering", "estimated": 12000, "added_at": "2026-03-01T09:00:00.000Z" }
    ],
    "payments": [
      { "vendor": "Estancia La Linda", "amount": 6000, "note": "50% deposit", "date": "2026-03-15T09:00:00.000Z" }
    ]
  },
  "tasks": [
    {
      "title": "Send invitations",
      "due_date": "2026-08-15",
      "priority": "high",
      "status": "pending",
      "created_at": "2026-03-01T09:00:00.000Z"
    }
  ],
  "rsvp_stats": { "confirmed": 45, "declined": 3, "pending": 72 },
  "created_at": "2026-03-01T09:00:00.000Z",
  "last_interaction": "2026-04-02T14:30:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir wedding-planner && cd wedding-planner
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

# 6. Submit an RSVP via API
curl -X POST http://localhost:3000/rsvp \
  -H "Content-Type: application/json" \
  -d '{
    "wedding_phone": "5491155550000",
    "guest_name": "Roberto Garcia",
    "status": "confirmed",
    "plus_ones": 1,
    "dietary": "vegetarian"
  }'
```
