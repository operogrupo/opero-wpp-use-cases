# 07 - CRM Sync

Bidirectional sync between WhatsApp conversations and a CRM. Every message is tagged with deal info, every conversation tracks pipeline stage. Metadata replaces the need for a separate integration database.

## Problem

A B2B software company uses HubSpot for deal tracking but their sales team lives in WhatsApp. Conversations happen on WhatsApp, but updates land in HubSpot hours later -- if at all. When a rep is sick, nobody knows what was discussed with a client because the context lives in the rep's phone. Deal stages in the CRM lag behind reality by days. The VP of Sales has no real-time pipeline visibility, and handoffs between reps lose weeks of context.

## Solution

A middleware that:
- Tags every WhatsApp message with the CRM deal_id, contact_id, and pipeline stage
- Updates CRM deal stage when conversation signals progress (meeting booked, proposal sent, etc.)
- Syncs contact details between WhatsApp and CRM
- Stores pipeline context in Opero conversation metadata -- so any rep can pick up where another left off
- Provides a webhook both ways: WhatsApp changes update CRM, CRM changes update WhatsApp metadata
- No separate database needed -- Opero metadata is the integration layer

## Architecture

```
                     +------------------+
                     |     HubSpot      |
                     |   (or any CRM)   |
                     +--------+---------+
                              |
          CRM webhook         |        API calls
          (deal updated)      |        (create/update deals)
                              |
+------------------+     +----+---------------+     +------------------+
|   Opero WPP API  |<--->|  Your Express App  |<--->|   Your CRM API   |
|  wpp-api.opero.so|     |    (this code)     |     |   (HubSpot SDK)  |
+------------------+     +--------------------+     +------------------+
         |                        |
    webhook POST                  |
    (message.received)            |
         |                        |
         v                        v
  Tag message with           Update deal stage
  deal_id + stage            in CRM automatically
```

## Code

```javascript
// server.js
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const HUBSPOT_API_KEY = process.env.HUBSPOT_API_KEY;
const HUBSPOT_BASE_URL = 'https://api.hubapi.com';

// ── Pipeline stages (match your CRM) ─────────────────────────────────
const PIPELINE_STAGES = {
  new_lead: { label: 'Nuevo Lead', order: 1 },
  contacted: { label: 'Contactado', order: 2 },
  qualified: { label: 'Calificado', order: 3 },
  meeting_scheduled: { label: 'Reunion Agendada', order: 4 },
  proposal_sent: { label: 'Propuesta Enviada', order: 5 },
  negotiation: { label: 'Negociacion', order: 6 },
  closed_won: { label: 'Ganado', order: 7 },
  closed_lost: { label: 'Perdido', order: 8 },
};

// Signal words that indicate pipeline progression
const STAGE_SIGNALS = [
  { patterns: [/reuni[oó]n/i, /meeting/i, /call/i, /agenda/i, /nos juntamos/i], stage: 'meeting_scheduled' },
  { patterns: [/propuesta/i, /presupuesto/i, /cotizaci[oó]n/i, /te env[ií]o/i, /adjunto/i], stage: 'proposal_sent' },
  { patterns: [/descuento/i, /negoci/i, /precio final/i, /condiciones/i], stage: 'negotiation' },
  { patterns: [/cerramos/i, /dale.*adelante/i, /aceptamos/i, /firmamos/i, /de acuerdo/i], stage: 'closed_won' },
  { patterns: [/no.*interesa/i, /no.*momento/i, /no.*presupuesto/i, /cancelar/i], stage: 'closed_lost' },
];

function detectStageProgression(message, currentStage) {
  const currentOrder = PIPELINE_STAGES[currentStage]?.order || 0;

  for (const signal of STAGE_SIGNALS) {
    for (const pattern of signal.patterns) {
      if (pattern.test(message)) {
        const newOrder = PIPELINE_STAGES[signal.stage]?.order || 0;
        // Only progress forward, never backward
        if (newOrder > currentOrder) {
          return signal.stage;
        }
      }
    }
  }

  return null;
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

async function tagMessage(conversationPhone, messageId, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${conversationPhone}/messages/${messageId}`, {
    method: 'PATCH',
    body: JSON.stringify({ metadata }),
  });
}

async function updateConversation(phone, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    method: 'PUT',
    body: JSON.stringify({ metadata }),
  });
}

async function getConversation(phone) {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`);
    return res.data;
  } catch {
    return null;
  }
}

async function listConversations() {
  const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=200`);
  return res.data || [];
}

// ── HubSpot CRM helpers ──────────────────────────────────────────────
async function hubspotFetch(path, options = {}) {
  const res = await fetch(`${HUBSPOT_BASE_URL}${path}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${HUBSPOT_API_KEY}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });
  if (!res.ok) {
    const text = await res.text();
    console.error(`HubSpot API error: ${res.status} ${text}`);
    return null;
  }
  return res.json();
}

async function findOrCreateContact(phone, name) {
  // Search for existing contact by phone
  const searchResult = await hubspotFetch('/crm/v3/objects/contacts/search', {
    method: 'POST',
    body: JSON.stringify({
      filterGroups: [{
        filters: [{
          propertyName: 'phone',
          operator: 'CONTAINS_TOKEN',
          value: phone.slice(-10), // Last 10 digits
        }],
      }],
    }),
  });

  if (searchResult?.results?.length > 0) {
    return searchResult.results[0];
  }

  // Create new contact
  const newContact = await hubspotFetch('/crm/v3/objects/contacts', {
    method: 'POST',
    body: JSON.stringify({
      properties: {
        phone: phone,
        firstname: name || 'WhatsApp Lead',
        lifecyclestage: 'lead',
        hs_lead_status: 'NEW',
        whatsapp_synced: 'true',
      },
    }),
  });

  return newContact;
}

async function findOrCreateDeal(contactId, phone, name) {
  // Search for existing open deal
  const searchResult = await hubspotFetch('/crm/v3/objects/deals/search', {
    method: 'POST',
    body: JSON.stringify({
      filterGroups: [{
        filters: [{
          propertyName: 'whatsapp_phone',
          operator: 'EQ',
          value: phone,
        }],
      }],
      sorts: [{ propertyName: 'createdate', direction: 'DESCENDING' }],
    }),
  });

  if (searchResult?.results?.length > 0) {
    const deal = searchResult.results[0];
    // Only return if deal is not closed
    const stage = deal.properties.dealstage;
    if (stage !== 'closedwon' && stage !== 'closedlost') {
      return deal;
    }
  }

  // Create new deal
  const deal = await hubspotFetch('/crm/v3/objects/deals', {
    method: 'POST',
    body: JSON.stringify({
      properties: {
        dealname: `WhatsApp - ${name || phone}`,
        pipeline: 'default',
        dealstage: 'appointmentscheduled', // HubSpot default stage
        whatsapp_phone: phone,
        whatsapp_synced: 'true',
      },
    }),
  });

  // Associate deal with contact
  if (deal && contactId) {
    await hubspotFetch(`/crm/v3/objects/deals/${deal.id}/associations/contacts/${contactId}/3`, {
      method: 'PUT',
    });
  }

  return deal;
}

async function updateDealStage(dealId, stage, note) {
  // Map our stages to HubSpot stages
  const hubspotStageMap = {
    new_lead: 'appointmentscheduled',
    contacted: 'appointmentscheduled',
    qualified: 'qualifiedtobuy',
    meeting_scheduled: 'presentationscheduled',
    proposal_sent: 'decisionmakerboughtin',
    negotiation: 'contractsent',
    closed_won: 'closedwon',
    closed_lost: 'closedlost',
  };

  const hubspotStage = hubspotStageMap[stage];
  if (!hubspotStage) return;

  await hubspotFetch(`/crm/v3/objects/deals/${dealId}`, {
    method: 'PATCH',
    body: JSON.stringify({
      properties: {
        dealstage: hubspotStage,
        notes_last_updated: new Date().toISOString(),
        description: note || `Stage updated via WhatsApp sync`,
      },
    }),
  });
}

async function addDealNote(dealId, content) {
  await hubspotFetch('/crm/v3/objects/notes', {
    method: 'POST',
    body: JSON.stringify({
      properties: {
        hs_note_body: content,
        hs_timestamp: new Date().toISOString(),
      },
      associations: [{
        to: { id: dealId },
        types: [{ associationCategory: 'HUBSPOT_DEFINED', associationTypeId: 214 }],
      }],
    }),
  });
}

// ── Inbound: WhatsApp message -> CRM sync ─────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' && event !== 'message.sent') return;

  const phone = event === 'message.received' ? data.from : data.to;
  const direction = event === 'message.received' ? 'inbound' : 'outbound';
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '[non-text message]';

  console.log(`[${new Date().toISOString()}] ${direction} message ${phone}: ${messageText.slice(0, 100)}`);

  try {
    // 1. Get or create CRM records
    const conversation = await getConversation(phone);
    const meta = conversation?.metadata || {};
    let crmState = meta.crm || {};

    let contactId = crmState.contact_id;
    let dealId = crmState.deal_id;
    let currentStage = crmState.pipeline_stage || 'new_lead';

    // Sync with CRM if not yet linked
    if (!contactId || !dealId) {
      const contact = await findOrCreateContact(phone, conversation?.contact_name);
      if (contact) {
        contactId = contact.id;
        const deal = await findOrCreateDeal(contactId, phone, conversation?.contact_name);
        if (deal) {
          dealId = deal.id;
        }
      }
    }

    // 2. Detect stage progression from message content
    const newStage = detectStageProgression(messageText, currentStage);
    if (newStage) {
      console.log(`[CRM] Stage change detected: ${currentStage} -> ${newStage}`);
      currentStage = newStage;

      // Update CRM deal stage
      if (dealId) {
        await updateDealStage(dealId, newStage, `Auto-detected from WhatsApp: "${messageText.slice(0, 200)}"`);
      }
    }

    // 3. Log message as CRM note
    if (dealId) {
      const noteBody = `[WhatsApp ${direction}] ${messageText}`;
      await addDealNote(dealId, noteBody);
    }

    // 4. Tag the message with CRM context
    if (data.id) {
      await tagMessage(phone, data.id, {
        crm: {
          contact_id: contactId,
          deal_id: dealId,
          pipeline_stage: currentStage,
          direction,
          synced_at: new Date().toISOString(),
        },
      });
    }

    // 5. Update conversation metadata
    const messageLog = (crmState.message_log || []).slice(-50);
    messageLog.push({
      direction,
      preview: messageText.slice(0, 100),
      stage: currentStage,
      timestamp: new Date().toISOString(),
    });

    const updatedCrm = {
      contact_id: contactId,
      deal_id: dealId,
      pipeline_stage: currentStage,
      stage_label: PIPELINE_STAGES[currentStage]?.label || currentStage,
      total_messages: (crmState.total_messages || 0) + 1,
      inbound_count: (crmState.inbound_count || 0) + (direction === 'inbound' ? 1 : 0),
      outbound_count: (crmState.outbound_count || 0) + (direction === 'outbound' ? 1 : 0),
      first_contact: crmState.first_contact || new Date().toISOString(),
      last_activity: new Date().toISOString(),
      message_log: messageLog,
      stage_history: [
        ...(crmState.stage_history || []),
        ...(newStage ? [{ from: crmState.pipeline_stage, to: newStage, at: new Date().toISOString(), trigger: messageText.slice(0, 100) }] : []),
      ].slice(-20),
    };

    await updateConversation(phone, { ...meta, crm: updatedCrm });

    console.log(`[${new Date().toISOString()}] Synced to CRM: deal=${dealId}, stage=${currentStage}`);
  } catch (err) {
    console.error(`CRM sync error for ${phone}:`, err.message);
  }
});

// ── Inbound: CRM webhook -> WhatsApp metadata update ─────────────────
app.post('/crm-webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const events = Array.isArray(req.body) ? req.body : [req.body];

  for (const event of events) {
    if (event.subscriptionType !== 'deal.propertyChange') continue;
    if (event.propertyName !== 'dealstage') continue;

    const dealId = String(event.objectId);
    const newHubspotStage = event.propertyValue;

    console.log(`[CRM Webhook] Deal ${dealId} stage changed to ${newHubspotStage}`);

    // Find the WhatsApp conversation linked to this deal
    try {
      const conversations = await listConversations();
      const linked = conversations.find(c => c.metadata?.crm?.deal_id === dealId);

      if (linked) {
        // Reverse-map HubSpot stage to our stages
        const reverseMap = {
          appointmentscheduled: 'contacted',
          qualifiedtobuy: 'qualified',
          presentationscheduled: 'meeting_scheduled',
          decisionmakerboughtin: 'proposal_sent',
          contractsent: 'negotiation',
          closedwon: 'closed_won',
          closedlost: 'closed_lost',
        };

        const ourStage = reverseMap[newHubspotStage] || linked.metadata.crm.pipeline_stage;

        await updateConversation(linked.contact_phone, {
          ...linked.metadata,
          crm: {
            ...linked.metadata.crm,
            pipeline_stage: ourStage,
            stage_label: PIPELINE_STAGES[ourStage]?.label || ourStage,
            last_crm_sync: new Date().toISOString(),
            stage_history: [
              ...(linked.metadata.crm.stage_history || []),
              { from: linked.metadata.crm.pipeline_stage, to: ourStage, at: new Date().toISOString(), trigger: 'CRM update' },
            ].slice(-20),
          },
        });

        console.log(`[CRM Webhook] Updated WhatsApp metadata for ${linked.contact_phone} -> ${ourStage}`);
      }
    } catch (err) {
      console.error(`CRM webhook processing error:`, err.message);
    }
  }
});

// ── Pipeline dashboard ────────────────────────────────────────────────
app.get('/pipeline', async (req, res) => {
  try {
    const conversations = await listConversations();

    const pipeline = {};
    for (const stage of Object.keys(PIPELINE_STAGES)) {
      pipeline[stage] = { label: PIPELINE_STAGES[stage].label, deals: [] };
    }

    for (const conv of conversations) {
      const crm = conv.metadata?.crm;
      if (!crm?.pipeline_stage) continue;

      const bucket = pipeline[crm.pipeline_stage];
      if (bucket) {
        bucket.deals.push({
          phone: conv.contact_phone,
          name: conv.contact_name,
          deal_id: crm.deal_id,
          total_messages: crm.total_messages,
          last_activity: crm.last_activity,
          days_in_stage: Math.floor(
            (Date.now() - new Date(crm.stage_history?.at(-1)?.at || crm.first_contact).getTime()) / 86400000
          ),
        });
      }
    }

    res.json(pipeline);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

const PORT = process.env.PORT || 3006;
app.listen(PORT, () => {
  console.log(`CRM sync listening on port ${PORT}`);
});
```

## Metadata Schema

### Message-level metadata

```json
{
  "crm": {
    "contact_id": "12345",
    "deal_id": "67890",
    "pipeline_stage": "proposal_sent",
    "direction": "outbound",
    "synced_at": "2026-04-02T15:00:00.000Z"
  }
}
```

### Conversation-level metadata

```json
{
  "crm": {
    "contact_id": "12345",
    "deal_id": "67890",
    "pipeline_stage": "proposal_sent",
    "stage_label": "Propuesta Enviada",
    "total_messages": 24,
    "inbound_count": 12,
    "outbound_count": 12,
    "first_contact": "2026-03-20T10:00:00.000Z",
    "last_activity": "2026-04-02T15:00:00.000Z",
    "last_crm_sync": "2026-04-02T15:00:00.000Z",
    "stage_history": [
      { "from": "new_lead", "to": "contacted", "at": "2026-03-20T10:00:00.000Z", "trigger": "First outbound message" },
      { "from": "contacted", "to": "qualified", "at": "2026-03-22T14:00:00.000Z", "trigger": "Budget discussed" },
      { "from": "qualified", "to": "meeting_scheduled", "at": "2026-03-25T09:00:00.000Z", "trigger": "nos juntamos el jueves?" },
      { "from": "meeting_scheduled", "to": "proposal_sent", "at": "2026-04-01T11:00:00.000Z", "trigger": "Te envio la propuesta adjunta" }
    ],
    "message_log": [
      { "direction": "outbound", "preview": "Hola, soy Martin de TechCorp...", "stage": "contacted", "timestamp": "2026-03-20T10:00:00.000Z" },
      { "direction": "inbound", "preview": "Hola Martin, si estamos interesados...", "stage": "contacted", "timestamp": "2026-03-20T10:15:00.000Z" }
    ]
  }
}
```

## How to Run

```bash
# 1. Create the project
mkdir crm-sync && cd crm-sync
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export HUBSPOT_API_KEY="your-hubspot-private-app-token"

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3006

# 5. Register webhook with Opero (both received and sent)
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received", "message.sent"]
  }'

# 6. Register CRM webhook in HubSpot
# In HubSpot: Settings -> Integrations -> Private Apps -> Webhooks
# Add subscription for "deal.propertyChange" on property "dealstage"
# URL: https://your-ngrok-url.ngrok.io/crm-webhook

# 7. View pipeline
curl http://localhost:3006/pipeline
```
