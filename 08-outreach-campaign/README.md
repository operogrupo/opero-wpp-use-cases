# 08 - Outreach Campaign

Send personalized WhatsApp messages to a contact list. Verify numbers, generate personalized messages with AI, send with natural delays, track delivery/read status, and auto-follow-up unread messages after 24 hours.

## Problem

A digital marketing agency signs up new restaurant clients and needs to send them onboarding messages. Currently, a team member manually types and sends 40-50 personalized messages per day. Each message takes 3-4 minutes to personalize. The work is tedious, messages go out at inconsistent times, and there's no systematic follow-up for people who don't respond. The agency estimates they miss 30% of potential conversions because follow-ups fall through the cracks.

## Solution

An outreach campaign system that:
- Imports a contact list with personalization fields (name, business, details)
- Checks which phone numbers are actually on WhatsApp before sending
- Generates personalized messages using Claude Sonnet based on each contact's context
- Sends messages with randomized natural delays (not all at once)
- Tracks delivery and read receipts via webhooks
- Automatically follows up with contacts who haven't read the message after 24 hours
- Provides campaign analytics (sent, delivered, read, replied rates)

## Architecture

```
+-------------------+
|   Contact List    |
|   (CSV/JSON)      |
+--------+----------+
         |
         v
+--------------------+    check phones     +------------------+
|  Your Express App  | -----------------> |   Opero WPP API  |
|   (this code)      |                     |  wpp-api.opero.so|
+--------------------+                     +------------------+
         |                                          |
         v                                          |
+--------------------+                              |
|  Claude Sonnet     |                              |
|  (personalize msg) |                              |
+--------------------+                              |
         |                                          |
         |  POST messages with typing delay         |
         +----------------------------------------->|
                                                    |
         +<----- webhook: delivery/read status -----+
         |
         v
+--------------------+
| Follow-up Scheduler|
| (24h check)        |
+--------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';
import { readFileSync } from 'fs';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// ── Campaign state ───────────────────────────────────────────────────
const campaigns = new Map();

class Campaign {
  constructor(id, name, contacts, template, config = {}) {
    this.id = id;
    this.name = name;
    this.template = template;
    this.config = {
      min_delay_seconds: config.min_delay_seconds || 30,
      max_delay_seconds: config.max_delay_seconds || 120,
      follow_up_after_hours: config.follow_up_after_hours || 24,
      follow_up_template: config.follow_up_template || null,
      send_start_hour: config.send_start_hour || 9,  // Don't send before 9am
      send_end_hour: config.send_end_hour || 20,     // Don't send after 8pm
      use_ai_personalization: config.use_ai_personalization !== false,
    };
    this.contacts = contacts.map(c => ({
      ...c,
      status: 'pending',       // pending | checking | invalid | queued | sending | sent | delivered | read | replied | failed
      wa_registered: null,
      message_sent: null,
      message_id: null,
      sent_at: null,
      delivered_at: null,
      read_at: null,
      replied_at: null,
      follow_up_sent: false,
      follow_up_at: null,
      error: null,
    }));
    this.status = 'created';   // created | checking | ready | running | paused | completed
    this.created_at = new Date().toISOString();
    this.started_at = null;
    this.completed_at = null;
  }

  get stats() {
    const counts = {};
    for (const c of this.contacts) {
      counts[c.status] = (counts[c.status] || 0) + 1;
    }
    return {
      total: this.contacts.length,
      ...counts,
      delivery_rate: counts.delivered ? counts.delivered / (counts.sent || 1) : 0,
      read_rate: counts.read ? counts.read / (counts.delivered || 1) : 0,
      reply_rate: counts.replied ? counts.replied / (counts.read || 1) : 0,
    };
  }
}

// ── Opero API helpers ─────────────────────────────────────────────────
async function operoFetch(path, options = {}) {
  const res = await fetch(`${OPERO_BASE_URL}${path}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${OPERO_API_KEY}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });
  if (!res.ok) throw new Error(`Opero API error ${res.status}: ${await res.text()}`);
  return res.json();
}

async function checkPhone(phone) {
  const res = await operoFetch(`/api/numbers/${NUMBER_ID}/contacts/${phone}/check`);
  return res.data;
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

async function updateConversation(phone, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    method: 'PUT',
    body: JSON.stringify({ metadata }),
  });
}

// ── AI personalization ────────────────────────────────────────────────
async function personalizeMessage(template, contact) {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 512,
    system: `You are a professional outreach copywriter. Take the template message and personalize it for the specific contact.

Rules:
- Keep the core message intact but make it feel personal, not mass-sent
- Use Argentine Spanish, informal "vos" form
- Keep it under 300 characters for WhatsApp readability
- Don't add emojis unless the template has them
- Don't be overly familiar -- this is a professional first contact
- Output ONLY the personalized message, nothing else`,
    messages: [{
      role: 'user',
      content: `Template: "${template}"

Contact info:
- Name: ${contact.name}
- Business: ${contact.business || 'N/A'}
- Industry: ${contact.industry || 'N/A'}
- Location: ${contact.location || 'N/A'}
- Notes: ${contact.notes || 'N/A'}`,
    }],
  });

  return response.content[0].text;
}

// ── Campaign execution ────────────────────────────────────────────────
function randomDelay(min, max) {
  return Math.floor(Math.random() * (max - min + 1) + min) * 1000;
}

function isWithinSendingHours(config) {
  const now = new Date();
  const hour = now.getHours();
  return hour >= config.send_start_hour && hour < config.send_end_hour;
}

async function checkPhones(campaign) {
  campaign.status = 'checking';
  console.log(`[Campaign ${campaign.id}] Checking ${campaign.contacts.length} phone numbers...`);

  for (const contact of campaign.contacts) {
    contact.status = 'checking';
    try {
      const result = await checkPhone(contact.phone);
      contact.wa_registered = result.registered;
      contact.status = result.registered ? 'queued' : 'invalid';

      if (!result.registered) {
        console.log(`[Campaign ${campaign.id}] ${contact.phone} not on WhatsApp`);
      }
    } catch (err) {
      console.error(`[Campaign ${campaign.id}] Error checking ${contact.phone}:`, err.message);
      contact.wa_registered = null;
      contact.status = 'queued'; // Try sending anyway
    }

    // Small delay between checks to avoid rate limiting
    await new Promise(r => setTimeout(r, 500));
  }

  campaign.status = 'ready';
  const valid = campaign.contacts.filter(c => c.status === 'queued').length;
  console.log(`[Campaign ${campaign.id}] Ready: ${valid}/${campaign.contacts.length} valid numbers`);
}

async function runCampaign(campaign) {
  campaign.status = 'running';
  campaign.started_at = new Date().toISOString();

  const queued = campaign.contacts.filter(c => c.status === 'queued');
  console.log(`[Campaign ${campaign.id}] Sending to ${queued.length} contacts...`);

  for (const contact of queued) {
    // Pause if outside sending hours
    while (!isWithinSendingHours(campaign.config)) {
      console.log(`[Campaign ${campaign.id}] Outside sending hours, pausing...`);
      await new Promise(r => setTimeout(r, 60000)); // Check every minute
    }

    if (campaign.status === 'paused') {
      console.log(`[Campaign ${campaign.id}] Paused by user`);
      return;
    }

    contact.status = 'sending';

    try {
      // Generate personalized message
      let message;
      if (campaign.config.use_ai_personalization) {
        message = await personalizeMessage(campaign.template, contact);
      } else {
        message = campaign.template
          .replace('{{name}}', contact.name || '')
          .replace('{{business}}', contact.business || '')
          .replace('{{industry}}', contact.industry || '')
          .replace('{{location}}', contact.location || '');
      }

      contact.message_sent = message;

      // Show typing indicator
      await sendTyping(contact.phone);

      // Natural typing delay based on message length
      const typingMs = Math.min(message.length * 50, 4000);
      await new Promise(r => setTimeout(r, typingMs));

      // Send message
      const result = await sendText(contact.phone, message, {
        agent: 'outreach-campaign',
        campaign_id: campaign.id,
        campaign_name: campaign.name,
        contact_name: contact.name,
        personalized: campaign.config.use_ai_personalization,
      });

      contact.status = 'sent';
      contact.message_id = result.data?.id;
      contact.sent_at = new Date().toISOString();

      // Update conversation metadata
      await updateConversation(contact.phone, {
        outreach: {
          campaign_id: campaign.id,
          campaign_name: campaign.name,
          sent_at: contact.sent_at,
          status: 'sent',
          contact_name: contact.name,
          contact_business: contact.business,
        },
      });

      console.log(`[Campaign ${campaign.id}] Sent to ${contact.name} (${contact.phone})`);
    } catch (err) {
      contact.status = 'failed';
      contact.error = err.message;
      console.error(`[Campaign ${campaign.id}] Failed for ${contact.phone}:`, err.message);
    }

    // Random delay between messages
    const delay = randomDelay(campaign.config.min_delay_seconds, campaign.config.max_delay_seconds);
    console.log(`[Campaign ${campaign.id}] Waiting ${delay / 1000}s before next message...`);
    await new Promise(r => setTimeout(r, delay));
  }

  campaign.status = 'completed';
  campaign.completed_at = new Date().toISOString();
  console.log(`[Campaign ${campaign.id}] Completed!`, campaign.stats);
}

// ── Follow-up scheduler ───────────────────────────────────────────────
async function processFollowUps() {
  const now = Date.now();

  for (const campaign of campaigns.values()) {
    if (campaign.status !== 'completed') continue;
    if (!campaign.config.follow_up_template) continue;

    for (const contact of campaign.contacts) {
      if (contact.status !== 'sent' && contact.status !== 'delivered') continue;
      if (contact.follow_up_sent) continue;

      const sentTime = new Date(contact.sent_at).getTime();
      const hoursSinceSent = (now - sentTime) / 3600000;

      if (hoursSinceSent >= campaign.config.follow_up_after_hours) {
        console.log(`[Follow-up] ${contact.name} hasn't read after ${Math.round(hoursSinceSent)}h`);

        try {
          let followUpMsg;
          if (campaign.config.use_ai_personalization) {
            followUpMsg = await personalizeMessage(campaign.config.follow_up_template, contact);
          } else {
            followUpMsg = campaign.config.follow_up_template
              .replace('{{name}}', contact.name || '');
          }

          await sendTyping(contact.phone);
          await new Promise(r => setTimeout(r, 2000));

          await sendText(contact.phone, followUpMsg, {
            agent: 'outreach-campaign',
            campaign_id: campaign.id,
            type: 'follow_up',
            hours_since_first: Math.round(hoursSinceSent),
          });

          contact.follow_up_sent = true;
          contact.follow_up_at = new Date().toISOString();

          console.log(`[Follow-up] Sent to ${contact.name} (${contact.phone})`);
        } catch (err) {
          console.error(`[Follow-up] Failed for ${contact.phone}:`, err.message);
        }

        // Delay between follow-ups
        await new Promise(r => setTimeout(r, randomDelay(30, 60)));
      }
    }
  }
}

// Check follow-ups every 30 minutes
setInterval(processFollowUps, 30 * 60 * 1000);

// ── Webhook: track delivery and read status ───────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;

  // Track read receipts and replies
  for (const campaign of campaigns.values()) {
    for (const contact of campaign.contacts) {
      if (event === 'message.status' && data.id === contact.message_id) {
        if (data.status === 'delivered') {
          contact.status = 'delivered';
          contact.delivered_at = new Date().toISOString();
        } else if (data.status === 'read') {
          contact.status = 'read';
          contact.read_at = new Date().toISOString();
        }
      }

      if (event === 'message.received' && data.from === contact.phone) {
        contact.status = 'replied';
        contact.replied_at = new Date().toISOString();
      }
    }
  }
});

// ── API: Campaign management ──────────────────────────────────────────

// Create a campaign
app.post('/campaigns', async (req, res) => {
  const { name, contacts, template, follow_up_template, config } = req.body;

  if (!name || !contacts?.length || !template) {
    return res.status(400).json({ error: 'Required: name, contacts[], template' });
  }

  const id = `camp_${Date.now().toString(36)}`;
  const campaign = new Campaign(id, name, contacts, template, {
    ...config,
    follow_up_template,
  });

  campaigns.set(id, campaign);

  res.json({
    campaign_id: id,
    total_contacts: contacts.length,
    status: campaign.status,
    message: 'Campaign created. POST /campaigns/:id/check to verify phone numbers, then POST /campaigns/:id/start to begin sending.',
  });
});

// Check phone numbers
app.post('/campaigns/:id/check', async (req, res) => {
  const campaign = campaigns.get(req.params.id);
  if (!campaign) return res.status(404).json({ error: 'Campaign not found' });

  res.json({ message: 'Phone check started', campaign_id: campaign.id });

  // Run in background
  checkPhones(campaign).catch(err => {
    console.error(`Phone check failed for ${campaign.id}:`, err.message);
  });
});

// Start sending
app.post('/campaigns/:id/start', async (req, res) => {
  const campaign = campaigns.get(req.params.id);
  if (!campaign) return res.status(404).json({ error: 'Campaign not found' });

  if (campaign.status !== 'ready' && campaign.status !== 'paused') {
    return res.status(400).json({
      error: `Campaign is ${campaign.status}. Must be 'ready' or 'paused' to start.`,
    });
  }

  res.json({ message: 'Campaign started', campaign_id: campaign.id });

  runCampaign(campaign).catch(err => {
    console.error(`Campaign ${campaign.id} failed:`, err.message);
    campaign.status = 'paused';
  });
});

// Pause campaign
app.post('/campaigns/:id/pause', (req, res) => {
  const campaign = campaigns.get(req.params.id);
  if (!campaign) return res.status(404).json({ error: 'Campaign not found' });

  campaign.status = 'paused';
  res.json({ message: 'Campaign paused', campaign_id: campaign.id });
});

// Get campaign status
app.get('/campaigns/:id', (req, res) => {
  const campaign = campaigns.get(req.params.id);
  if (!campaign) return res.status(404).json({ error: 'Campaign not found' });

  res.json({
    id: campaign.id,
    name: campaign.name,
    status: campaign.status,
    stats: campaign.stats,
    created_at: campaign.created_at,
    started_at: campaign.started_at,
    completed_at: campaign.completed_at,
    contacts: campaign.contacts.map(c => ({
      phone: c.phone,
      name: c.name,
      status: c.status,
      sent_at: c.sent_at,
      delivered_at: c.delivered_at,
      read_at: c.read_at,
      replied_at: c.replied_at,
      follow_up_sent: c.follow_up_sent,
      error: c.error,
    })),
  });
});

// List all campaigns
app.get('/campaigns', (req, res) => {
  const list = [...campaigns.values()].map(c => ({
    id: c.id,
    name: c.name,
    status: c.status,
    stats: c.stats,
    created_at: c.created_at,
  }));
  res.json({ campaigns: list });
});

app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    active_campaigns: [...campaigns.values()].filter(c => c.status === 'running').length,
    total_campaigns: campaigns.size,
  });
});

const PORT = process.env.PORT || 3007;
app.listen(PORT, () => {
  console.log(`Outreach campaign manager listening on port ${PORT}`);
});
```

## Metadata Schema

### Conversation-level metadata (set on each outreach target)

```json
{
  "outreach": {
    "campaign_id": "camp_m1a2b3c",
    "campaign_name": "Restaurant Onboarding Q2",
    "sent_at": "2026-04-02T10:30:00.000Z",
    "status": "sent",
    "contact_name": "Carlos",
    "contact_business": "La Parrilla de Carlos"
  }
}
```

### Message-level metadata

```json
{
  "agent": "outreach-campaign",
  "campaign_id": "camp_m1a2b3c",
  "campaign_name": "Restaurant Onboarding Q2",
  "contact_name": "Carlos",
  "personalized": true,
  "type": "follow_up",
  "hours_since_first": 26
}
```

## How to Run

```bash
# 1. Create the project
mkdir outreach-campaign && cd outreach-campaign
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3007

# 5. Register webhook (for delivery/read tracking)
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received", "message.status"]
  }'

# 6. Create a campaign
curl -X POST http://localhost:3007/campaigns \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Restaurant Onboarding Q2",
    "template": "Hola {{name}}, vi que tenes {{business}} y queria contarte como podemos ayudarte a recibir mas reservas por WhatsApp. Te interesa saber mas?",
    "follow_up_template": "Hola {{name}}, te escribi ayer sobre una herramienta para gestionar reservas. Tenes 5 minutos para que te cuente?",
    "contacts": [
      { "phone": "5491155551234", "name": "Carlos", "business": "La Parrilla de Carlos", "industry": "gastronomia", "location": "San Telmo" },
      { "phone": "5491155555678", "name": "Maria", "business": "Cafe Tortoni", "industry": "gastronomia", "location": "Monserrat" },
      { "phone": "5491155559012", "name": "Pedro", "business": "Sushi Club Norte", "industry": "gastronomia", "location": "Belgrano" }
    ],
    "config": {
      "min_delay_seconds": 45,
      "max_delay_seconds": 90,
      "follow_up_after_hours": 24,
      "use_ai_personalization": true
    }
  }'

# 7. Check phone numbers
curl -X POST http://localhost:3007/campaigns/camp_XXXXX/check

# 8. Start sending
curl -X POST http://localhost:3007/campaigns/camp_XXXXX/start

# 9. Monitor progress
curl http://localhost:3007/campaigns/camp_XXXXX
```
