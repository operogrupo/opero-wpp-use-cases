# 06 - Restaurant Reservations

Handle table reservations via WhatsApp -- party size, date/time, dietary requirements, booking confirmation, and day-before reminders.

## Problem

A popular parrilla in San Telmo gets 60+ reservation requests per day on WhatsApp. The host manages these between seating guests, taking phone calls, and coordinating the floor. Double bookings happen 2-3 times per week, resulting in angry customers and bad Google reviews. Weekend reservations fill up fast, and the restaurant has no way to offer alternatives when a time slot is taken. They lose an estimated 10 covers per night to competitors who confirm faster.

## Solution

A WhatsApp bot that:
- Understands natural language reservations ("mesa para 4 el sabado a las 21")
- Checks table availability based on capacity and existing bookings
- Captures dietary requirements and special requests (celiaco, vegetariano, cumpleanos)
- Confirms with a formatted summary
- Sends reminders 24 hours before
- Stores full reservation details as conversation metadata

## Architecture

```
Customer: "Quiero reservar para 6 personas el viernes a la noche"
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
         |                          | Claude Haiku|  |  Table    |
         |                          | (NLU parse) |  |  Manager  |
         |                          +-------------+  +-----------+
         |                                   |
         |    POST confirmation + reminder   |
         |    PUT conversation metadata      |
         +-----------------------------------+
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

// ── Restaurant configuration ─────────────────────────────────────────
const RESTAURANT = {
  name: 'La Estancia de San Telmo',
  address: 'Defensa 1250, San Telmo, CABA',
  phone: '(011) 4361-5555',
  timezone: 'America/Argentina/Buenos_Aires',
};

const SERVICE_HOURS = {
  // day of week (0=Sunday) -> service slots
  0: { lunch: null, dinner: { open: '19:30', close: '23:00' } }, // Sunday dinner only
  1: null, // Monday closed
  2: { lunch: { open: '12:00', close: '15:00' }, dinner: { open: '19:30', close: '23:30' } },
  3: { lunch: { open: '12:00', close: '15:00' }, dinner: { open: '19:30', close: '23:30' } },
  4: { lunch: { open: '12:00', close: '15:00' }, dinner: { open: '19:30', close: '00:00' } },
  5: { lunch: { open: '12:00', close: '15:30' }, dinner: { open: '19:30', close: '00:30' } }, // Friday
  6: { lunch: { open: '12:00', close: '15:30' }, dinner: { open: '19:30', close: '01:00' } }, // Saturday
};

// Table configuration
const TABLES = [
  { id: 't1', seats: 2, section: 'terraza' },
  { id: 't2', seats: 2, section: 'terraza' },
  { id: 't3', seats: 4, section: 'terraza' },
  { id: 't4', seats: 4, section: 'salon' },
  { id: 't5', seats: 4, section: 'salon' },
  { id: 't6', seats: 6, section: 'salon' },
  { id: 't7', seats: 6, section: 'salon' },
  { id: 't8', seats: 8, section: 'privado' },
  { id: 't9', seats: 10, section: 'privado' },
];

const SLOT_DURATION_MINUTES = 120; // 2 hours per reservation

// In-memory bookings (use a database in production)
const reservations = new Map(); // key: "YYYY-MM-DD_HH:mm_tableId" -> reservation

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

// ── Availability engine ──────────────────────────────────────────────
function getServiceForTime(date, time) {
  const day = new Date(date + 'T12:00:00').getDay();
  const hours = SERVICE_HOURS[day];
  if (!hours) return null;

  const [h] = time.split(':').map(Number);
  if (hours.lunch && h >= 12 && h < 16) return 'lunch';
  if (hours.dinner && h >= 19) return 'dinner';
  return null;
}

function findAvailableTable(date, time, partySize) {
  // Find tables that can seat the party
  const suitableTables = TABLES
    .filter(t => t.seats >= partySize && t.seats <= partySize + 2) // Don't waste a 10-top on 2 people
    .sort((a, b) => a.seats - b.seats); // Prefer smallest suitable table

  for (const table of suitableTables) {
    const key = `${date}_${time}_${table.id}`;
    if (!reservations.has(key)) {
      // Check for overlapping reservations (2-hour slots)
      const [h, m] = time.split(':').map(Number);
      const startMin = h * 60 + m;
      let conflict = false;

      for (const [resKey] of reservations) {
        if (!resKey.includes(table.id)) continue;
        const [resDate, resTime] = resKey.split('_');
        if (resDate !== date) continue;
        const [rh, rm] = resTime.split(':').map(Number);
        const resStart = rh * 60 + rm;
        if (Math.abs(startMin - resStart) < SLOT_DURATION_MINUTES) {
          conflict = true;
          break;
        }
      }

      if (!conflict) return table;
    }
  }

  return null;
}

function getAvailableTimeSlots(date, partySize) {
  const day = new Date(date + 'T12:00:00').getDay();
  const hours = SERVICE_HOURS[day];
  if (!hours) return [];

  const slots = [];

  for (const service of ['lunch', 'dinner']) {
    if (!hours[service]) continue;
    const [openH, openM] = hours[service].open.split(':').map(Number);
    const [closeH, closeM] = hours[service].close.split(':').map(Number);

    let current = openH * 60 + openM;
    const end = (closeH < openH ? closeH + 24 : closeH) * 60 + closeM - SLOT_DURATION_MINUTES;

    while (current <= end) {
      const h = String(Math.floor(current / 60) % 24).padStart(2, '0');
      const m = String(current % 60).padStart(2, '0');
      const time = `${h}:${m}`;

      const table = findAvailableTable(date, time, partySize);
      if (table) {
        slots.push({ time, service, section: table.section });
      }

      current += 30; // 30-minute increments
    }
  }

  return slots;
}

function makeReservation(date, time, partySize, phone, details) {
  const table = findAvailableTable(date, time, partySize);
  if (!table) return null;

  const reservation = {
    id: `RES-${Date.now().toString(36).toUpperCase()}`,
    date,
    time,
    party_size: partySize,
    table_id: table.id,
    section: table.section,
    phone,
    name: details.name || null,
    dietary: details.dietary || [],
    special_requests: details.special_requests || null,
    status: 'confirmed',
    created_at: new Date().toISOString(),
    reminder_sent: false,
  };

  const key = `${date}_${time}_${table.id}`;
  reservations.set(key, reservation);

  return reservation;
}

function cancelReservation(reservationId) {
  for (const [key, res] of reservations) {
    if (res.id === reservationId) {
      res.status = 'cancelled';
      reservations.delete(key);
      return res;
    }
  }
  return null;
}

function formatDate(dateStr) {
  const date = new Date(dateStr + 'T12:00:00');
  return date.toLocaleDateString('es-AR', {
    weekday: 'long',
    day: 'numeric',
    month: 'long',
    timeZone: RESTAURANT.timezone,
  });
}

// ── Claude for NLU ────────────────────────────────────────────────────
async function parseReservationIntent(message, currentState) {
  const today = new Date().toLocaleDateString('en-CA', { timeZone: RESTAURANT.timezone });
  const now = new Date().toLocaleTimeString('en-US', { hour12: false, timeZone: RESTAURANT.timezone });

  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 1024,
    system: `You are the reservation assistant for ${RESTAURANT.name}, a traditional Argentine parrilla in San Telmo.

Today: ${today}, current time: ${now}.
Closed Mondays. Lunch: 12:00-15:00, Dinner: 19:30-late.
Maximum party size: 10.

Current reservation state: ${JSON.stringify(currentState)}

Return ONLY a JSON object:
{
  "action": "reserve" | "cancel" | "modify" | "check_availability" | "greeting" | "unclear" | "confirm",
  "party_size": number or null,
  "date": "YYYY-MM-DD" or null,
  "time": "HH:mm" or null,
  "name": "guest name" or null,
  "dietary": ["celiaco", "vegetariano", "vegano", "kosher"] or [],
  "special_requests": "string" or null,
  "message_to_user": "Friendly response in Argentine Spanish (vos)",
  "needs_info": ["party_size", "date", "time", "name"] // what's still missing
}

Rules:
- "a la noche" without time = 20:30 default
- "almuerzo" without time = 12:30 default
- Argentine dining: dinner starts 20:00+, lunch 12:00-14:00
- "el finde" = this coming Saturday
- Always ask for name to confirm`,
    messages: [{ role: 'user', content: message }],
  });

  const text = response.content[0].text;
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (!jsonMatch) throw new Error('Failed to parse AI response');
  return JSON.parse(jsonMatch[0]);
}

// ── Message handler ───────────────────────────────────────────────────
async function handleMessage(phone, messageText) {
  const conversation = await getConversation(phone);
  const meta = conversation?.metadata || {};
  const state = meta.reservation_state || {
    pending_party_size: null,
    pending_date: null,
    pending_time: null,
    pending_name: null,
    pending_dietary: [],
    pending_special: null,
    reservations: [],
  };

  const parsed = await parseReservationIntent(messageText, state);
  let reply = parsed.message_to_user;

  // Merge parsed data into pending state
  if (parsed.party_size) state.pending_party_size = parsed.party_size;
  if (parsed.date) state.pending_date = parsed.date;
  if (parsed.time) state.pending_time = parsed.time;
  if (parsed.name) state.pending_name = parsed.name;
  if (parsed.dietary?.length) state.pending_dietary = parsed.dietary;
  if (parsed.special_requests) state.pending_special = parsed.special_requests;

  if (parsed.action === 'reserve' || parsed.action === 'confirm') {
    const { pending_party_size: size, pending_date: date, pending_time: time, pending_name: name } = state;

    if (size && date && time) {
      // Check if restaurant is open
      const day = new Date(date + 'T12:00:00').getDay();
      if (!SERVICE_HOURS[day]) {
        reply = `Disculpa, los ${formatDate(date).split(',')[0]} estamos cerrados. Queres reservar para otro dia?`;
      } else {
        const reservation = makeReservation(date, time, size, phone, {
          name: name,
          dietary: state.pending_dietary,
          special_requests: state.pending_special,
        });

        if (reservation) {
          const dateFormatted = formatDate(date);
          reply = `Reserva confirmada!\n\n` +
            `Codigo: ${reservation.id}\n` +
            `Fecha: ${dateFormatted}\n` +
            `Hora: ${time}\n` +
            `Personas: ${size}\n` +
            `Mesa: ${reservation.section}\n` +
            (name ? `A nombre de: ${name}\n` : '') +
            (state.pending_dietary.length ? `Dieta: ${state.pending_dietary.join(', ')}\n` : '') +
            (state.pending_special ? `Nota: ${state.pending_special}\n` : '') +
            `\nDireccion: ${RESTAURANT.address}\n` +
            `\nTe enviaremos un recordatorio manana. Para cancelar, escribi "cancelar ${reservation.id}".`;

          state.reservations.push(reservation);
          // Reset pending
          state.pending_party_size = null;
          state.pending_date = null;
          state.pending_time = null;
          state.pending_name = null;
          state.pending_dietary = [];
          state.pending_special = null;
        } else {
          // No table available -- suggest alternatives
          const alternatives = getAvailableTimeSlots(date, size);
          if (alternatives.length > 0) {
            const altList = alternatives.slice(0, 6).map(s =>
              `  - ${s.time} (${s.service === 'lunch' ? 'almuerzo' : 'cena'}, ${s.section})`
            ).join('\n');
            reply = `Esa hora no esta disponible para ${size} personas el ${formatDate(date)}. Horarios libres:\n\n${altList}\n\nCual te queda mejor?`;
          } else {
            reply = `No tenemos disponibilidad para ${size} personas el ${formatDate(date)}. Queres probar otro dia?`;
          }
        }
      }
    }
  }

  if (parsed.action === 'cancel') {
    const activeReservations = state.reservations.filter(r => r.status === 'confirmed');
    // Try to find the reservation to cancel
    const resIdMatch = messageText.match(/RES-[A-Z0-9]+/i);
    if (resIdMatch) {
      const cancelled = cancelReservation(resIdMatch[0]);
      if (cancelled) {
        const res = state.reservations.find(r => r.id === resIdMatch[0]);
        if (res) res.status = 'cancelled';
        reply = `Tu reserva ${resIdMatch[0]} para el ${formatDate(cancelled.date)} a las ${cancelled.time} fue cancelada. Queres hacer una nueva reserva?`;
      }
    } else if (activeReservations.length === 1) {
      const res = activeReservations[0];
      cancelReservation(res.id);
      res.status = 'cancelled';
      reply = `Tu reserva para el ${formatDate(res.date)} a las ${res.time} fue cancelada. Queres hacer una nueva reserva?`;
    } else if (activeReservations.length > 1) {
      const list = activeReservations.map(r =>
        `  - ${r.id}: ${formatDate(r.date)} ${r.time} (${r.party_size} pers.)`
      ).join('\n');
      reply = `Tenes varias reservas activas:\n\n${list}\n\nCual queres cancelar? Envame el codigo.`;
    }
  }

  if (parsed.action === 'check_availability' && state.pending_date) {
    const slots = getAvailableTimeSlots(
      state.pending_date,
      state.pending_party_size || 2
    );
    if (slots.length === 0) {
      reply = `No hay disponibilidad para el ${formatDate(state.pending_date)}. Queres que busque otro dia?`;
    } else {
      const grouped = { lunch: [], dinner: [] };
      slots.forEach(s => grouped[s.service]?.push(s));

      let availMsg = `Disponibilidad para el ${formatDate(state.pending_date)} (${state.pending_party_size || 2} personas):\n\n`;
      if (grouped.lunch.length) {
        availMsg += `Almuerzo:\n${grouped.lunch.map(s => `  - ${s.time}`).join('\n')}\n\n`;
      }
      if (grouped.dinner.length) {
        availMsg += `Cena:\n${grouped.dinner.map(s => `  - ${s.time}`).join('\n')}\n`;
      }
      availMsg += `\nQue horario te gusta?`;
      reply = availMsg;
    }
  }

  state.last_interaction = new Date().toISOString();

  return { reply, state };
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

  try {
    await sendTyping(phone);

    const { reply, state } = await handleMessage(phone, messageText);

    await new Promise(resolve => setTimeout(resolve, 1500));
    await sendText(phone, reply, {
      agent: 'restaurant-reservations',
      model: 'claude-haiku-4-20250414',
    });

    await updateConversation(phone, { reservation_state: state });
  } catch (err) {
    console.error(`Error from ${phone}:`, err.message);
    await sendText(phone, `Disculpa, tuve un problema. Podes llamarnos al ${RESTAURANT.phone} para hacer tu reserva.`, {
      agent: 'restaurant-reservations',
      error: true,
    });
  }
});

// ── Reminder cron ─────────────────────────────────────────────────────
async function sendReminders() {
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  const tomorrowStr = tomorrow.toLocaleDateString('en-CA', { timeZone: RESTAURANT.timezone });

  for (const [, reservation] of reservations) {
    if (reservation.date !== tomorrowStr) continue;
    if (reservation.status !== 'confirmed') continue;
    if (reservation.reminder_sent) continue;

    try {
      await sendText(reservation.phone,
        `Hola${reservation.name ? ' ' + reservation.name : ''}! Te recordamos tu reserva en ${RESTAURANT.name} para manana:\n\n` +
        `Hora: ${reservation.time}\n` +
        `Personas: ${reservation.party_size}\n` +
        `Mesa: ${reservation.section}\n` +
        `Direccion: ${RESTAURANT.address}\n\n` +
        `Para cancelar, responde "cancelar". Te esperamos!`,
        {
          agent: 'restaurant-reservations',
          type: 'reminder',
          reservation_id: reservation.id,
        }
      );
      reservation.reminder_sent = true;
    } catch (err) {
      console.error(`Reminder failed for ${reservation.id}:`, err.message);
    }
  }
}

setInterval(sendReminders, 60 * 60 * 1000);

// ── View reservations ─────────────────────────────────────────────────
app.get('/reservations', (req, res) => {
  const { date } = req.query;
  const list = [...reservations.values()]
    .filter(r => !date || r.date === date)
    .filter(r => r.status === 'confirmed')
    .sort((a, b) => `${a.date}_${a.time}`.localeCompare(`${b.date}_${b.time}`));

  res.json({ reservations: list, total: list.length });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', active_reservations: [...reservations.values()].filter(r => r.status === 'confirmed').length });
});

const PORT = process.env.PORT || 3005;
app.listen(PORT, () => {
  console.log(`Restaurant reservations bot listening on port ${PORT}`);
  sendReminders();
});
```

## Metadata Schema

### Conversation-level metadata

```json
{
  "reservation_state": {
    "pending_party_size": null,
    "pending_date": null,
    "pending_time": null,
    "pending_name": "Lucia Fernandez",
    "pending_dietary": [],
    "pending_special": null,
    "last_interaction": "2026-04-02T18:00:00.000Z",
    "reservations": [
      {
        "id": "RES-M1A2B3C",
        "date": "2026-04-05",
        "time": "21:00",
        "party_size": 6,
        "table_id": "t6",
        "section": "salon",
        "phone": "5491155551234",
        "name": "Lucia Fernandez",
        "dietary": ["celiaco"],
        "special_requests": "cumpleanos, si pueden preparar algo",
        "status": "confirmed",
        "created_at": "2026-04-02T18:00:00.000Z",
        "reminder_sent": false
      }
    ]
  }
}
```

### Message-level metadata

```json
{
  "agent": "restaurant-reservations",
  "model": "claude-haiku-4-20250414",
  "type": "reminder",
  "reservation_id": "RES-M1A2B3C"
}
```

## How to Run

```bash
# 1. Create the project
mkdir restaurant-reservations && cd restaurant-reservations
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3005

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. View today's reservations
curl "http://localhost:3005/reservations?date=2026-04-05"
```
