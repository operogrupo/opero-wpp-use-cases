# 29 - Event RSVP

Event invitation and RSVP management via WhatsApp. Send invitations, collect responses, manage guest counts and dietary preferences, send reminders, and coordinate day-of logistics.

## Problem

An event planning company manages corporate events and weddings with 100-500 guests each. RSVPs come in via email, phone, and text in no consistent format. Tracking dietary restrictions, plus-ones, and no-shows is a manual spreadsheet nightmare. Last-minute changes cause catering errors and seating conflicts. Staff spend 20+ hours per event chasing RSVPs.

## Solution

Deploy a WhatsApp RSVP bot that:
- Sends formatted invitations with event details
- Collects structured RSVPs (attending / not attending / maybe) with guest count and dietary needs
- Sends automatic reminders to non-responders
- Provides day-of logistics (directions, parking, schedule)
- Tracks all RSVP data in conversation metadata for real-time dashboards

## Architecture

```
+--------------------+     POST /api/events      +--------------------+
|  Event Organizer   | ----------------------->  |  Your Express App  |
|  (creates event)   |                           |   (this code)      |
+--------------------+                           +--------------------+
                                                          |
         +------------------------------------------------+
         |                    |                           |
         v                    v                           v
+------------------+  Send invitations           Track RSVPs
|   Opero WPP API  |  to guest list             in metadata
|  wpp-api.opero.so|                                     |
+------------------+                                     v
         ^                                     +-----------------+
         |            webhook                  | RSVP Dashboard  |
         +---- guest responds via WPP -------> | GET /api/events |
                                               +-----------------+
```

## Code

```javascript
// server.js
import express from 'express';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

// ── Data stores (use a real DB in production) ─────────────────────────
const events = new Map();
const guestIndex = new Map(); // phone -> { event_id, guest_id }

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
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`Opero API error ${res.status}: ${text}`);
  }
  return res.json();
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

async function markAsRead(phone, messageIds) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/read`, {
    method: 'POST',
    body: JSON.stringify({ phone, message_ids: messageIds }),
  });
}

// ── RSVP state machine ───────────────────────────────────────────────
const RSVP_STATES = {
  INVITED: 'invited',
  ATTENDING: 'attending',
  NOT_ATTENDING: 'not_attending',
  MAYBE: 'maybe',
  COLLECTING_GUESTS: 'collecting_guests',
  COLLECTING_DIETARY: 'collecting_dietary',
  CONFIRMED: 'confirmed',
};

// ── Event management ──────────────────────────────────────────────────
function createEvent(eventData) {
  const id = `evt-${Date.now()}`;
  const event = {
    id,
    ...eventData,
    guests: [],
    created_at: new Date().toISOString(),
  };
  events.set(id, event);
  return event;
}

function addGuest(eventId, guestData) {
  const event = events.get(eventId);
  if (!event) throw new Error('Event not found');

  const guestId = `guest-${Date.now()}-${Math.random().toString(36).substr(2, 4)}`;
  const guest = {
    id: guestId,
    ...guestData,
    rsvp_status: RSVP_STATES.INVITED,
    guest_count: 0,
    dietary_restrictions: [],
    invited_at: null,
    responded_at: null,
    reminders_sent: 0,
  };

  event.guests.push(guest);
  guestIndex.set(guestData.phone, { event_id: eventId, guest_id: guestId });
  return guest;
}

function getGuestByPhone(phone) {
  const index = guestIndex.get(phone);
  if (!index) return null;

  const event = events.get(index.event_id);
  if (!event) return null;

  const guest = event.guests.find(g => g.id === index.guest_id);
  return { event, guest };
}

// ── Send invitations ──────────────────────────────────────────────────
async function sendInvitation(eventId, guestId) {
  const event = events.get(eventId);
  if (!event) throw new Error('Event not found');

  const guest = event.guests.find(g => g.id === guestId);
  if (!guest) throw new Error('Guest not found');

  const invitation =
    `🎉 *${event.title}*\n\n` +
    `Estimado/a ${guest.name},\n\n` +
    `Tiene el agrado de invitarlo/a a ${event.description}\n\n` +
    `📅 Fecha: *${event.date}*\n` +
    `🕐 Hora: *${event.time}*\n` +
    `📍 Lugar: *${event.location}*\n` +
    `${event.dress_code ? `👔 Vestimenta: ${event.dress_code}\n` : ''}` +
    `\n¿Podrá asistir?\n\n` +
    `Responda:\n` +
    `*1* - Sí, confirmo asistencia\n` +
    `*2* - No podré asistir\n` +
    `*3* - Aún no estoy seguro/a`;

  await sendText(guest.phone, invitation, {
    agent: 'event-rsvp',
    event_id: eventId,
    guest_id: guestId,
    action: 'invitation_sent',
  });

  guest.invited_at = new Date().toISOString();
  guest.rsvp_status = RSVP_STATES.INVITED;
}

async function sendInvitations(eventId) {
  const event = events.get(eventId);
  if (!event) throw new Error('Event not found');

  const results = { sent: 0, failed: 0 };

  for (const guest of event.guests) {
    if (guest.invited_at) continue; // Already invited

    try {
      await sendInvitation(eventId, guest.id);
      results.sent++;
      await new Promise(resolve => setTimeout(resolve, 300));
    } catch (err) {
      results.failed++;
      console.error(`Failed to invite ${guest.name}:`, err.message);
    }
  }

  return results;
}

// ── Send reminders ────────────────────────────────────────────────────
async function sendReminders(eventId) {
  const event = events.get(eventId);
  if (!event) throw new Error('Event not found');

  const pendingGuests = event.guests.filter(g =>
    g.rsvp_status === RSVP_STATES.INVITED || g.rsvp_status === RSVP_STATES.MAYBE
  );

  const results = { sent: 0 };

  for (const guest of pendingGuests) {
    const message = guest.rsvp_status === RSVP_STATES.MAYBE
      ? `Hola ${guest.name}, le recordamos que *${event.title}* es el *${event.date}*. ¿Ya pudo confirmar su asistencia?\n\n*1* - Sí\n*2* - No\n*3* - Todavía no sé`
      : `Hola ${guest.name}, aún no recibimos su respuesta para *${event.title}* (${event.date}).\n\nResponda:\n*1* - Sí, confirmo\n*2* - No podré ir\n*3* - Aún no sé`;

    try {
      await sendText(guest.phone, message, {
        agent: 'event-rsvp',
        event_id: eventId,
        guest_id: guest.id,
        action: 'reminder_sent',
        reminder_number: guest.reminders_sent + 1,
      });
      guest.reminders_sent++;
      results.sent++;
      await new Promise(resolve => setTimeout(resolve, 300));
    } catch (err) {
      console.error(`Failed to remind ${guest.name}:`, err.message);
    }
  }

  return results;
}

// ── Day-of logistics ──────────────────────────────────────────────────
async function sendDayOfInfo(eventId) {
  const event = events.get(eventId);
  if (!event) throw new Error('Event not found');

  const confirmedGuests = event.guests.filter(g =>
    g.rsvp_status === RSVP_STATES.CONFIRMED || g.rsvp_status === RSVP_STATES.ATTENDING
  );

  for (const guest of confirmedGuests) {
    const message =
      `🎉 *¡Hoy es el día! - ${event.title}*\n\n` +
      `📍 ${event.location}\n` +
      `🕐 ${event.time}\n` +
      `${event.parking_info ? `🅿️ Estacionamiento: ${event.parking_info}\n` : ''}` +
      `${event.contact_on_site ? `📞 Contacto en el lugar: ${event.contact_on_site}\n` : ''}` +
      `\n${event.day_of_notes || '¡Los esperamos!'}`;

    try {
      await sendText(guest.phone, message, {
        agent: 'event-rsvp',
        event_id: eventId,
        guest_id: guest.id,
        action: 'day_of_info',
      });
      await new Promise(resolve => setTimeout(resolve, 300));
    } catch (err) {
      console.error(`Failed to send day-of info to ${guest.name}:`, err.message);
    }
  }
}

// ── Webhook handler (guest responses) ─────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received') return;
  if (data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim();

  if (!messageText) return;

  const lookup = getGuestByPhone(phone);
  if (!lookup) return; // Unknown number — not a guest

  const { event: evt, guest } = lookup;

  console.log(`[${new Date().toISOString()}] RSVP from ${guest.name} (${phone}): ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    // Handle based on current state
    if (guest.rsvp_status === RSVP_STATES.INVITED || guest.rsvp_status === RSVP_STATES.MAYBE) {
      switch (messageText.trim()) {
        case '1': // Attending
          guest.rsvp_status = RSVP_STATES.COLLECTING_GUESTS;
          guest.responded_at = new Date().toISOString();

          await sendText(phone, `¡Excelente, ${guest.name}! Nos alegra que pueda asistir.\n\n¿Cuántas personas asistirán en total (incluyéndolo/a a usted)?`, {
            agent: 'event-rsvp',
            event_id: evt.id,
            guest_id: guest.id,
            action: 'rsvp_attending',
          });
          break;

        case '2': // Not attending
          guest.rsvp_status = RSVP_STATES.NOT_ATTENDING;
          guest.responded_at = new Date().toISOString();

          await sendText(phone, `Lamentamos que no pueda acompañarnos, ${guest.name}. Si cambia de opinión, no dude en escribirnos. ¡Saludos!`, {
            agent: 'event-rsvp',
            event_id: evt.id,
            guest_id: guest.id,
            action: 'rsvp_declined',
          });

          await updateConversation(phone, {
            agent: 'event-rsvp',
            event_id: evt.id,
            rsvp_status: 'not_attending',
            responded_at: guest.responded_at,
          });
          break;

        case '3': // Maybe
          guest.rsvp_status = RSVP_STATES.MAYBE;
          guest.responded_at = new Date().toISOString();

          await sendText(phone, `Entendido, ${guest.name}. Le enviaremos un recordatorio más adelante. Cuando decida, responda *1* para confirmar o *2* para declinar.`, {
            agent: 'event-rsvp',
            event_id: evt.id,
            guest_id: guest.id,
            action: 'rsvp_maybe',
          });
          break;

        default:
          await sendText(phone, `Por favor responda con:\n*1* - Sí, confirmo\n*2* - No podré ir\n*3* - Aún no estoy seguro/a`, {
            agent: 'event-rsvp',
            action: 'invalid_response',
          });
          break;
      }
    } else if (guest.rsvp_status === RSVP_STATES.COLLECTING_GUESTS) {
      const count = parseInt(messageText, 10);
      if (isNaN(count) || count < 1 || count > 20) {
        await sendText(phone, 'Por favor indique un número válido de asistentes (1-20).', {
          agent: 'event-rsvp',
          action: 'invalid_guest_count',
        });
        return;
      }

      guest.guest_count = count;
      guest.rsvp_status = RSVP_STATES.COLLECTING_DIETARY;

      await sendText(phone, `Perfecto, ${count} persona(s).\n\n¿Alguno de los asistentes tiene restricciones alimentarias?\n\n*1* - No, ninguna\n*2* - Vegetariano\n*3* - Vegano\n*4* - Celíaco\n*5* - Otro (especificar)`, {
        agent: 'event-rsvp',
        event_id: evt.id,
        guest_id: guest.id,
        action: 'guest_count_collected',
        guest_count: count,
      });
    } else if (guest.rsvp_status === RSVP_STATES.COLLECTING_DIETARY) {
      const dietaryMap = {
        '1': 'Ninguna',
        '2': 'Vegetariano',
        '3': 'Vegano',
        '4': 'Celíaco',
      };

      const dietary = dietaryMap[messageText.trim()] || messageText;
      guest.dietary_restrictions.push(dietary);
      guest.rsvp_status = RSVP_STATES.CONFIRMED;

      await sendText(phone,
        `✅ *RSVP confirmado*\n\n` +
        `Evento: ${evt.title}\n` +
        `Fecha: ${evt.date}\n` +
        `Hora: ${evt.time}\n` +
        `Asistentes: ${guest.guest_count}\n` +
        `Restricciones: ${guest.dietary_restrictions.join(', ')}\n\n` +
        `Le enviaremos un recordatorio antes del evento. Si necesita cambiar algo, escriba *CAMBIAR*.`,
        {
          agent: 'event-rsvp',
          event_id: evt.id,
          guest_id: guest.id,
          action: 'rsvp_confirmed',
          guest_count: guest.guest_count,
          dietary: guest.dietary_restrictions,
        }
      );

      await updateConversation(phone, {
        agent: 'event-rsvp',
        event_id: evt.id,
        rsvp_status: 'confirmed',
        guest_count: guest.guest_count,
        dietary_restrictions: guest.dietary_restrictions,
        responded_at: guest.responded_at,
        confirmed_at: new Date().toISOString(),
      });
    } else if (guest.rsvp_status === RSVP_STATES.CONFIRMED || guest.rsvp_status === RSVP_STATES.NOT_ATTENDING) {
      if (messageText.toUpperCase() === 'CAMBIAR') {
        guest.rsvp_status = RSVP_STATES.INVITED;
        guest.guest_count = 0;
        guest.dietary_restrictions = [];

        await sendText(phone, `Entendido. ¿Podrá asistir a *${evt.title}* el *${evt.date}*?\n\n*1* - Sí\n*2* - No\n*3* - Todavía no sé`, {
          agent: 'event-rsvp',
          event_id: evt.id,
          guest_id: guest.id,
          action: 'rsvp_restart',
        });
      } else {
        await sendText(phone, `Su RSVP ya está registrado. Si necesita modificarlo, escriba *CAMBIAR*.`, {
          agent: 'event-rsvp',
          action: 'already_responded',
        });
      }
    }
  } catch (err) {
    console.error(`Error handling RSVP from ${phone}:`, err.message);
  }
});

// ── API: Create event ─────────────────────────────────────────────────
app.post('/api/events', (req, res) => {
  const event = createEvent(req.body);
  res.json(event);
});

// ── API: Add guests ───────────────────────────────────────────────────
app.post('/api/events/:eventId/guests', (req, res) => {
  const { guests } = req.body; // Array of { name, phone }
  if (!Array.isArray(guests)) {
    return res.status(400).json({ error: 'guests array is required' });
  }

  try {
    const added = guests.map(g => addGuest(req.params.eventId, g));
    res.json({ guests: added });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Send invitations ─────────────────────────────────────────────
app.post('/api/events/:eventId/invite', async (req, res) => {
  try {
    const results = await sendInvitations(req.params.eventId);
    res.json(results);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Send reminders ───────────────────────────────────────────────
app.post('/api/events/:eventId/remind', async (req, res) => {
  try {
    const results = await sendReminders(req.params.eventId);
    res.json(results);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Send day-of info ─────────────────────────────────────────────
app.post('/api/events/:eventId/day-of', async (req, res) => {
  try {
    await sendDayOfInfo(req.params.eventId);
    res.json({ status: 'sent' });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Event summary ───────────────────────────────────────────────
app.get('/api/events/:eventId', (req, res) => {
  const event = events.get(req.params.eventId);
  if (!event) return res.status(404).json({ error: 'Event not found' });

  const summary = {
    ...event,
    stats: {
      total_invited: event.guests.length,
      confirmed: event.guests.filter(g => g.rsvp_status === RSVP_STATES.CONFIRMED).length,
      attending: event.guests.filter(g => g.rsvp_status === RSVP_STATES.ATTENDING).length,
      declined: event.guests.filter(g => g.rsvp_status === RSVP_STATES.NOT_ATTENDING).length,
      maybe: event.guests.filter(g => g.rsvp_status === RSVP_STATES.MAYBE).length,
      pending: event.guests.filter(g => g.rsvp_status === RSVP_STATES.INVITED).length,
      total_guest_count: event.guests.reduce((sum, g) => sum + g.guest_count, 0),
      dietary_summary: event.guests
        .flatMap(g => g.dietary_restrictions)
        .reduce((acc, d) => { acc[d] = (acc[d] || 0) + 1; return acc; }, {}),
    },
  };

  res.json(summary);
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    events_count: events.size,
    guests_tracked: guestIndex.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Event RSVP bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata after RSVP confirmation:

```json
{
  "agent": "event-rsvp",
  "event_id": "evt-1711958400000",
  "rsvp_status": "confirmed",
  "guest_count": 3,
  "dietary_restrictions": ["Vegetariano"],
  "responded_at": "2026-04-02T10:00:00.000Z",
  "confirmed_at": "2026-04-02T10:03:00.000Z"
}
```

Per-message metadata:

```json
{
  "agent": "event-rsvp",
  "event_id": "evt-1711958400000",
  "guest_id": "guest-1711958400000-a1b2",
  "action": "rsvp_confirmed",
  "guest_count": 3,
  "dietary": ["Vegetariano"]
}
```

## How to Run

```bash
# 1. Create the project
mkdir event-rsvp && cd event-rsvp
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"

# 3. Start the server
node server.js

# 4. Expose locally with ngrok
ngrok http 3000

# 5. Register your webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. Create an event
curl -X POST http://localhost:3000/api/events \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Cena Anual de la Empresa",
    "description": "la Cena Anual 2026 de TechCorp",
    "date": "2026-05-15",
    "time": "20:00",
    "location": "Hotel Alvear, Av. Alvear 1891, CABA",
    "dress_code": "Formal",
    "parking_info": "Valet parking disponible en la entrada",
    "contact_on_site": "+54 11 4444-5678"
  }'

# 7. Add guests (use the event ID from the response)
curl -X POST http://localhost:3000/api/events/evt-xxx/guests \
  -H "Content-Type: application/json" \
  -d '{
    "guests": [
      { "name": "María González", "phone": "5491155551001" },
      { "name": "Carlos López", "phone": "5491155551002" }
    ]
  }'

# 8. Send invitations
curl -X POST http://localhost:3000/api/events/evt-xxx/invite
```
