# 05 - Real Estate Agent

A full-featured real estate sales bot that qualifies buyers, matches properties from a database, sends property images with details, and schedules viewings -- all via WhatsApp.

## Problem

A real estate agency in Palermo, Buenos Aires lists 120+ properties. Agents spend their mornings answering the same questions: "What do you have in Belgrano under $200k?", "Do you have 3-bedroom apartments near the subte?" Each agent handles 30-40 daily inquiries but can only show 5-6 properties per day. Unqualified leads waste viewing slots while serious buyers get frustrated by slow responses. The agency estimates 20% of deals are lost to competitors who respond faster.

## Solution

An AI real estate agent that:
- Qualifies buyers through natural conversation (budget, bedrooms, location, timeline)
- Searches a property database with matching criteria
- Sends property photos with formatted details via WhatsApp images
- Tracks buyer profile and funnel stage in conversation metadata
- Schedules viewings when a buyer shows strong interest
- Hands off to a human agent when the buyer is ready to negotiate

## Architecture

```
Buyer: "Busco depto 2 amb en Belgrano hasta USD 180k"
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                   +------+------+
         |                                   v             v
         |                          +-------------+  +-----------+
         |                          | Claude Haiku|  | Property  |
         |                          | (qualifier) |  | Database  |
         |                          +-------------+  +-----------+
         |                                   |
         |    POST text + images             |
         |    PUT conversation metadata      |
         +-----------------------------------+
                                             |
                                    (if ready to visit)
                                             v
                                    +------------------+
                                    | Calendar / Agent |
                                    | notification     |
                                    +------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// ── Property database (replace with your real DB/API) ─────────────────
const PROPERTIES = [
  {
    id: 'prop_001',
    type: 'departamento',
    operation: 'venta',
    neighborhood: 'Belgrano',
    city: 'Buenos Aires',
    bedrooms: 2,
    bathrooms: 1,
    area_m2: 65,
    price_usd: 165000,
    expenses_ars: 45000,
    floor: 8,
    amenities: ['balcon', 'cochera', 'lavadero'],
    description: 'Luminoso 2 ambientes en Belgrano R, a 2 cuadras del subte D. Piso alto con vista abierta. Cocina integrada, balcon corrido.',
    address: 'Echeverria 2400',
    images: [
      'https://images.example.com/prop001_living.jpg',
      'https://images.example.com/prop001_bedroom.jpg',
      'https://images.example.com/prop001_kitchen.jpg',
    ],
    available: true,
  },
  {
    id: 'prop_002',
    type: 'departamento',
    operation: 'venta',
    neighborhood: 'Palermo',
    city: 'Buenos Aires',
    bedrooms: 3,
    bathrooms: 2,
    area_m2: 95,
    price_usd: 230000,
    expenses_ars: 65000,
    floor: 5,
    amenities: ['balcon', 'pileta', 'gym', 'sum', 'cochera'],
    description: 'Moderno 4 ambientes en Palermo Hollywood. Edificio con amenities completos. Suite con vestidor. 2 balcones terraza.',
    address: 'Honduras 5800',
    images: [
      'https://images.example.com/prop002_living.jpg',
      'https://images.example.com/prop002_suite.jpg',
      'https://images.example.com/prop002_terrace.jpg',
    ],
    available: true,
  },
  {
    id: 'prop_003',
    type: 'departamento',
    operation: 'venta',
    neighborhood: 'Belgrano',
    city: 'Buenos Aires',
    bedrooms: 3,
    bathrooms: 2,
    area_m2: 110,
    price_usd: 195000,
    expenses_ars: 55000,
    floor: 3,
    amenities: ['balcon', 'dependencia', 'cochera'],
    description: 'Amplio 4 ambientes en Belgrano C. Living-comedor con salida a balcon. Dependencia de servicio. Cochera cubierta incluida.',
    address: 'Juramento 2100',
    images: [
      'https://images.example.com/prop003_living.jpg',
      'https://images.example.com/prop003_bedroom.jpg',
    ],
    available: true,
  },
  {
    id: 'prop_004',
    type: 'casa',
    operation: 'venta',
    neighborhood: 'Martinez',
    city: 'GBA Norte',
    bedrooms: 4,
    bathrooms: 3,
    area_m2: 220,
    price_usd: 350000,
    expenses_ars: 0,
    floor: null,
    amenities: ['jardin', 'pileta', 'quincho', 'cochera_doble'],
    description: 'Casa en barrio cerrado en Martinez. 4 dormitorios, pileta climatizada, quincho con parrilla. Lote de 600m2. Seguridad 24hs.',
    address: 'Barrio Los Robles, Martinez',
    images: [
      'https://images.example.com/prop004_front.jpg',
      'https://images.example.com/prop004_pool.jpg',
      'https://images.example.com/prop004_garden.jpg',
    ],
    available: true,
  },
  {
    id: 'prop_005',
    type: 'departamento',
    operation: 'venta',
    neighborhood: 'Nunez',
    city: 'Buenos Aires',
    bedrooms: 1,
    bathrooms: 1,
    area_m2: 42,
    price_usd: 98000,
    expenses_ars: 28000,
    floor: 12,
    amenities: ['balcon', 'pileta', 'gym', 'laundry'],
    description: 'Monoambiente divisible en torre premium en Nunez. Vista al rio. Amenities completos. Ideal inversion o primera vivienda.',
    address: 'Del Libertador 7200',
    images: [
      'https://images.example.com/prop005_studio.jpg',
      'https://images.example.com/prop005_view.jpg',
    ],
    available: true,
  },
];

// ── Property search engine ────────────────────────────────────────────
function searchProperties(criteria) {
  return PROPERTIES.filter(p => {
    if (!p.available) return false;
    if (criteria.max_price && p.price_usd > criteria.max_price) return false;
    if (criteria.min_price && p.price_usd < criteria.min_price) return false;
    if (criteria.bedrooms && p.bedrooms < criteria.bedrooms) return false;
    if (criteria.neighborhood) {
      const target = criteria.neighborhood.toLowerCase();
      if (!p.neighborhood.toLowerCase().includes(target) &&
          !p.city.toLowerCase().includes(target)) return false;
    }
    if (criteria.type && !p.type.toLowerCase().includes(criteria.type.toLowerCase())) return false;
    if (criteria.min_area && p.area_m2 < criteria.min_area) return false;
    return true;
  }).sort((a, b) => {
    // Score relevance
    let scoreA = 0, scoreB = 0;
    if (criteria.max_price) {
      scoreA += (criteria.max_price - a.price_usd) / criteria.max_price;
      scoreB += (criteria.max_price - b.price_usd) / criteria.max_price;
    }
    return scoreB - scoreA;
  });
}

function formatPropertyCard(property) {
  const price = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }).format(property.price_usd);
  const expenses = property.expenses_ars > 0
    ? new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS', maximumFractionDigits: 0 }).format(property.expenses_ars)
    : 'Sin expensas';

  return `🏠 *${property.type === 'casa' ? 'Casa' : 'Departamento'} en ${property.neighborhood}*\n` +
    `📍 ${property.address}\n` +
    `💰 ${price}\n` +
    `📐 ${property.area_m2}m² | ${property.bedrooms} amb | ${property.bathrooms} baño(s)\n` +
    (property.floor ? `🏢 Piso ${property.floor}\n` : '') +
    `💵 Expensas: ${expenses}\n` +
    `✨ ${property.amenities.join(', ')}\n\n` +
    `${property.description}`;
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

async function sendText(phone, text, metadata = {}) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/text`, {
    method: 'POST',
    body: JSON.stringify({ phone, text, metadata }),
  });
}

async function sendImage(phone, url, caption, metadata = {}) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/image`, {
    method: 'POST',
    body: JSON.stringify({ phone, url, caption, metadata }),
  });
}

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
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

// ── Claude for conversation understanding ─────────────────────────────
async function parseConversation(message, buyerProfile, conversationHistory) {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 1024,
    system: `You are a real estate assistant for Inmobiliaria Patagonia in Buenos Aires, Argentina. You help buyers find their ideal property.

Current buyer profile: ${JSON.stringify(buyerProfile)}

Your tasks:
1. Extract or update buyer preferences from their message
2. Determine what action to take
3. Craft a natural response in Argentine Spanish (use "vos")

Available neighborhoods: Belgrano, Palermo, Nunez, Martinez, Recoleta, Caballito, Villa Urquiza

Return ONLY a JSON object:
{
  "action": "search" | "qualify" | "schedule_viewing" | "more_info" | "handoff_to_agent" | "greeting",
  "updated_profile": {
    "budget_max_usd": number or null,
    "budget_min_usd": number or null,
    "bedrooms": number or null,
    "neighborhood": "string" or null,
    "property_type": "departamento" | "casa" or null,
    "min_area_m2": number or null,
    "timeline": "string" or null,
    "must_have": ["amenity1", "amenity2"],
    "buyer_name": "string" or null
  },
  "property_id_interested": "prop_xxx" or null,
  "viewing_date_requested": "YYYY-MM-DD" or null,
  "viewing_time_requested": "HH:mm" or null,
  "message_to_user": "Your friendly response in Spanish",
  "qualification_complete": true/false,
  "ready_for_human": true/false
}

Rules:
- When budget is in pesos, assume they mean USD for property purchase
- "2 ambientes" = 1 bedroom, "3 ambientes" = 2 bedrooms, etc.
- If someone says "hasta 200" they likely mean USD 200,000
- Always confirm understanding before searching
- If profile is mostly complete (budget + location or bedrooms), set qualification_complete: true`,
    messages: [
      ...(conversationHistory || []).slice(-6),
      { role: 'user', content: message },
    ],
  });

  const text = response.content[0].text;
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (!jsonMatch) throw new Error('Failed to parse AI response');
  return JSON.parse(jsonMatch[0]);
}

// ── Funnel stages ─────────────────────────────────────────────────────
const STAGES = {
  NEW: 'new',
  QUALIFYING: 'qualifying',
  SEARCHING: 'searching',
  VIEWING_SCHEDULED: 'viewing_scheduled',
  NEGOTIATING: 'negotiating',
  HANDED_OFF: 'handed_off',
};

// ── Message handler ───────────────────────────────────────────────────
async function handleBuyerMessage(phone, messageText) {
  const conversation = await getConversation(phone);
  const meta = conversation?.metadata || {};

  let buyerProfile = meta.buyer_profile || {
    budget_max_usd: null,
    budget_min_usd: null,
    bedrooms: null,
    neighborhood: null,
    property_type: null,
    min_area_m2: null,
    timeline: null,
    must_have: [],
    buyer_name: null,
  };

  const stage = meta.stage || STAGES.NEW;
  const conversationHistory = meta.conversation_history || [];

  // Add user message to history
  conversationHistory.push({ role: 'user', content: messageText });

  // Parse with Claude
  const parsed = await parseConversation(messageText, buyerProfile, conversationHistory);

  // Merge updated profile
  const updatedProfile = { ...buyerProfile };
  for (const [key, value] of Object.entries(parsed.updated_profile || {})) {
    if (value !== null && value !== undefined) {
      updatedProfile[key] = value;
    }
  }

  let reply = parsed.message_to_user;
  let newStage = stage;
  const propertiesSent = meta.properties_sent || [];
  const viewingsScheduled = meta.viewings_scheduled || [];

  // Handle actions
  if (parsed.action === 'search' && parsed.qualification_complete) {
    const results = searchProperties({
      max_price: updatedProfile.budget_max_usd,
      min_price: updatedProfile.budget_min_usd,
      bedrooms: updatedProfile.bedrooms,
      neighborhood: updatedProfile.neighborhood,
      type: updatedProfile.property_type,
      min_area: updatedProfile.min_area_m2,
    });

    // Filter out already-sent properties
    const newResults = results.filter(p => !propertiesSent.includes(p.id));

    if (newResults.length === 0 && results.length === 0) {
      reply += '\n\nNo encontre propiedades que coincidan exactamente. Queres que amplie la busqueda?';
    } else if (newResults.length === 0) {
      reply += '\n\nYa te mostre todas las propiedades que tenemos con esos criterios. Queres que busque con otros parametros?';
    } else {
      const toShow = newResults.slice(0, 3);
      reply += `\n\nEncontre ${results.length} propiedad(es). Te muestro las mejores:`;

      // Send property cards with images (done after the main reply)
      for (const property of toShow) {
        propertiesSent.push(property.id);
      }

      // Store properties to send after main message
      parsed._propertiesToSend = toShow;
    }

    newStage = STAGES.SEARCHING;
  }

  if (parsed.action === 'schedule_viewing' && parsed.property_id_interested) {
    const property = PROPERTIES.find(p => p.id === parsed.property_id_interested);
    if (property) {
      viewingsScheduled.push({
        property_id: property.id,
        property_address: property.address,
        neighborhood: property.neighborhood,
        requested_date: parsed.viewing_date_requested,
        requested_time: parsed.viewing_time_requested,
        scheduled_at: new Date().toISOString(),
        status: 'pending_confirmation',
      });
      newStage = STAGES.VIEWING_SCHEDULED;
    }
  }

  if (parsed.action === 'handoff_to_agent' || parsed.ready_for_human) {
    newStage = STAGES.HANDED_OFF;
  }

  if (parsed.action === 'qualify') {
    newStage = STAGES.QUALIFYING;
  }

  // Add assistant reply to history
  conversationHistory.push({ role: 'assistant', content: reply });

  // Build updated metadata
  const updatedMeta = {
    buyer_profile: updatedProfile,
    stage: newStage,
    properties_sent: propertiesSent,
    properties_viewed: meta.properties_viewed || [],
    viewings_scheduled: viewingsScheduled,
    conversation_history: conversationHistory.slice(-20),
    qualification_complete: parsed.qualification_complete || meta.qualification_complete,
    total_messages: (meta.total_messages || 0) + 1,
    last_interaction: new Date().toISOString(),
    first_contact: meta.first_contact || new Date().toISOString(),
  };

  return {
    reply,
    metadata: updatedMeta,
    propertiesToSend: parsed._propertiesToSend || [],
  };
}

// ── Webhook handler ───────────────────────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';

  if (!messageText.trim()) return;

  console.log(`[${new Date().toISOString()}] Buyer ${phone}: ${messageText}`);

  try {
    await sendTyping(phone);

    const { reply, metadata, propertiesToSend } = await handleBuyerMessage(phone, messageText);

    // Send main text reply
    await new Promise(resolve => setTimeout(resolve, 1500));
    await sendText(phone, reply, {
      agent: 'real-estate-agent',
      stage: metadata.stage,
    });

    // Send property images with details
    for (const property of propertiesToSend) {
      await new Promise(resolve => setTimeout(resolve, 2000));
      await sendTyping(phone);

      const card = formatPropertyCard(property);

      if (property.images && property.images.length > 0) {
        // Send first image with full details as caption
        await sendImage(phone, property.images[0], card, {
          agent: 'real-estate-agent',
          property_id: property.id,
          action: 'property_shown',
        });

        // Send additional images without caption
        for (let i = 1; i < Math.min(property.images.length, 3); i++) {
          await new Promise(resolve => setTimeout(resolve, 1000));
          await sendImage(phone, property.images[i], '', {
            agent: 'real-estate-agent',
            property_id: property.id,
          });
        }
      } else {
        await sendText(phone, card, {
          agent: 'real-estate-agent',
          property_id: property.id,
          action: 'property_shown',
        });
      }
    }

    if (propertiesToSend.length > 0) {
      await new Promise(resolve => setTimeout(resolve, 1500));
      await sendText(phone, 'Te interesa alguna? Puedo darte mas detalles o agendar una visita.', {
        agent: 'real-estate-agent',
        action: 'follow_up_prompt',
      });
    }

    // Update conversation metadata
    await updateConversation(phone, metadata);

    console.log(`[${new Date().toISOString()}] Stage: ${metadata.stage}, Properties sent: ${propertiesToSend.length}`);
  } catch (err) {
    console.error(`Error with buyer ${phone}:`, err.message);
    await sendText(phone, 'Disculpa, tuve un problema tecnico. Un asesor se va a comunicar con vos en breve.', {
      agent: 'real-estate-agent',
      error: true,
    });
  }
});

// ── Dashboard: view buyer pipeline ────────────────────────────────────
app.get('/pipeline', async (req, res) => {
  try {
    const result = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=200`);
    const conversations = result.data || [];

    const pipeline = {
      new: [],
      qualifying: [],
      searching: [],
      viewing_scheduled: [],
      negotiating: [],
      handed_off: [],
    };

    for (const conv of conversations) {
      const meta = conv.metadata;
      if (!meta?.stage) continue;

      pipeline[meta.stage]?.push({
        phone: conv.contact_phone,
        name: meta.buyer_profile?.buyer_name || conv.contact_name,
        budget: meta.buyer_profile?.budget_max_usd,
        bedrooms: meta.buyer_profile?.bedrooms,
        neighborhood: meta.buyer_profile?.neighborhood,
        properties_seen: meta.properties_sent?.length || 0,
        viewings: meta.viewings_scheduled?.length || 0,
        last_interaction: meta.last_interaction,
      });
    }

    res.json(pipeline);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', properties_listed: PROPERTIES.filter(p => p.available).length });
});

const PORT = process.env.PORT || 3004;
app.listen(PORT, () => {
  console.log(`Real estate agent listening on port ${PORT}`);
});
```

## Metadata Schema

### Conversation-level metadata

```json
{
  "buyer_profile": {
    "budget_max_usd": 200000,
    "budget_min_usd": null,
    "bedrooms": 2,
    "neighborhood": "Belgrano",
    "property_type": "departamento",
    "min_area_m2": 60,
    "timeline": "en los proximos 3 meses",
    "must_have": ["balcon", "cochera"],
    "buyer_name": "Carlos Mendez"
  },
  "stage": "searching",
  "properties_sent": ["prop_001", "prop_003"],
  "properties_viewed": [],
  "viewings_scheduled": [
    {
      "property_id": "prop_001",
      "property_address": "Echeverria 2400",
      "neighborhood": "Belgrano",
      "requested_date": "2026-04-05",
      "requested_time": "10:00",
      "scheduled_at": "2026-04-02T15:00:00.000Z",
      "status": "pending_confirmation"
    }
  ],
  "qualification_complete": true,
  "total_messages": 8,
  "first_contact": "2026-04-02T14:00:00.000Z",
  "last_interaction": "2026-04-02T15:30:00.000Z"
}
```

### Message-level metadata

```json
{
  "agent": "real-estate-agent",
  "stage": "searching",
  "property_id": "prop_001",
  "action": "property_shown"
}
```

## How to Run

```bash
# 1. Create the project
mkdir real-estate-agent && cd real-estate-agent
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3004

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. View buyer pipeline
curl http://localhost:3004/pipeline
```
