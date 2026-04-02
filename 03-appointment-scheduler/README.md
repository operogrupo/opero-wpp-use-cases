# 03 - Appointment Scheduler

A WhatsApp bot that handles appointment scheduling with natural language understanding, availability checking, booking confirmation, and day-before reminders.

## Problem

A dental clinic in Cordoba has two receptionists juggling phone calls, WhatsApp messages, and walk-ins. Double bookings happen weekly. Patients who message at night get no response until morning -- by then, 30% book elsewhere. The clinic loses an estimated 15 appointments per week to slow response times and scheduling errors.

## Solution

A Claude Haiku-powered scheduling bot that:
- Understands natural language dates ("manana a las 3", "el lunes que viene a la tarde")
- Checks real-time availability against the clinic's schedule
- Confirms bookings instantly with a summary
- Sends reminders 24 hours before the appointment
- Stores all booking data as conversation metadata -- no external database required for the scheduling logic

## Architecture

```
Patient sends "Quiero un turno para el martes a las 10"
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                   +------+------+
         |                                   |             |
         |                                   v             v
         |                          +-------------+  +-----------+
         |                          | Claude Haiku|  | Schedule  |
         |                          | (NLU parse) |  | Manager   |
         |                          +-------------+  +-----------+
         |                                   |
         |    POST send confirmation         |
         |    PUT conversation metadata      |
         +-----------------------------------+
         |
         |    (cron: every hour)
         v
+--------------------+
| Reminder Scheduler |
| checks next-day    |
| appointments       |
+--------------------+
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

// ── Schedule configuration ────────────────────────────────────────────
// In production, load from a database or Google Calendar API
const BUSINESS_HOURS = {
  1: { open: '09:00', close: '18:00' }, // Monday
  2: { open: '09:00', close: '18:00' }, // Tuesday
  3: { open: '09:00', close: '18:00' }, // Wednesday
  4: { open: '09:00', close: '20:00' }, // Thursday (extended)
  5: { open: '09:00', close: '18:00' }, // Friday
  6: { open: '09:00', close: '13:00' }, // Saturday (half day)
  // Sunday: closed
};

const SLOT_DURATION_MINUTES = 30;
const TIMEZONE = 'America/Argentina/Cordoba';

// In-memory bookings store (use a database in production)
const bookings = new Map(); // key: "YYYY-MM-DD_HH:mm" -> { phone, name, service, confirmed }

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

// ── Schedule helpers ──────────────────────────────────────────────────
function getAvailableSlots(dateStr) {
  const date = new Date(dateStr + 'T00:00:00');
  const dayOfWeek = date.getDay(); // 0=Sunday
  const hours = BUSINESS_HOURS[dayOfWeek];

  if (!hours) return []; // Closed

  const slots = [];
  const [openH, openM] = hours.open.split(':').map(Number);
  const [closeH, closeM] = hours.close.split(':').map(Number);

  let currentMinutes = openH * 60 + openM;
  const endMinutes = closeH * 60 + closeM;

  while (currentMinutes + SLOT_DURATION_MINUTES <= endMinutes) {
    const h = String(Math.floor(currentMinutes / 60)).padStart(2, '0');
    const m = String(currentMinutes % 60).padStart(2, '0');
    const slotKey = `${dateStr}_${h}:${m}`;

    if (!bookings.has(slotKey)) {
      slots.push(`${h}:${m}`);
    }

    currentMinutes += SLOT_DURATION_MINUTES;
  }

  return slots;
}

function bookSlot(dateStr, time, phone, name, service) {
  const slotKey = `${dateStr}_${time}`;
  if (bookings.has(slotKey)) return false;

  bookings.set(slotKey, {
    phone,
    name,
    service,
    confirmed: true,
    booked_at: new Date().toISOString(),
  });

  return true;
}

function cancelSlot(dateStr, time) {
  const slotKey = `${dateStr}_${time}`;
  return bookings.delete(slotKey);
}

function formatDate(dateStr) {
  const date = new Date(dateStr + 'T12:00:00');
  return date.toLocaleDateString('es-AR', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    timeZone: TIMEZONE,
  });
}

// ── Claude Haiku for natural language understanding ───────────────────
async function parseSchedulingIntent(message, conversationState) {
  const today = new Date().toLocaleDateString('en-CA', { timeZone: TIMEZONE }); // YYYY-MM-DD
  const currentTime = new Date().toLocaleTimeString('en-US', { hour12: false, timeZone: TIMEZONE });

  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 1024,
    system: `You are a scheduling assistant for Clinica Dental Sonrisa in Cordoba, Argentina. Today is ${today}, current time is ${currentTime}.

Your job is to parse the user's message and return a JSON object with the scheduling intent.

Services offered:
- limpieza (cleaning): 30 min, $15,000 ARS
- revision (checkup): 30 min, $12,000 ARS
- blanqueamiento (whitening): 60 min, $45,000 ARS
- ortodoncia_consulta (orthodontics consultation): 30 min, $8,000 ARS
- extraccion (extraction): 30 min, $25,000 ARS

Current conversation state: ${JSON.stringify(conversationState)}

Return ONLY a JSON object with these fields:
{
  "action": "book" | "cancel" | "reschedule" | "check_availability" | "list_services" | "confirm" | "unclear",
  "date": "YYYY-MM-DD" or null,
  "time": "HH:mm" or null,
  "service": "service_key" or null,
  "name": "patient name if mentioned" or null,
  "message_to_user": "Your response in Spanish to the patient"
}

Rules:
- "manana" = tomorrow, "pasado manana" = day after tomorrow
- "lunes que viene" = next Monday from today
- "a la tarde" without specific time = suggest available afternoon slots
- If date/time is ambiguous, ask for clarification in message_to_user
- If they want to book but haven't specified a service, ask which service
- Be warm and professional, use "vos" (Argentine Spanish)`,
    messages: [{ role: 'user', content: message }],
  });

  const text = response.content[0].text;
  // Extract JSON from response (handles markdown code blocks)
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (!jsonMatch) throw new Error('Failed to parse AI response');
  return JSON.parse(jsonMatch[0]);
}

// ── Conversation state machine ────────────────────────────────────────
// States: idle -> collecting_info -> confirming -> booked
async function handleMessage(phone, messageText) {
  const conversation = await getConversation(phone);
  const state = conversation?.metadata?.scheduling_state || {
    stage: 'idle',
    pending_date: null,
    pending_time: null,
    pending_service: null,
    pending_name: null,
    appointments: [],
  };

  const parsed = await parseSchedulingIntent(messageText, state);

  let reply = parsed.message_to_user;
  let newStage = state.stage;

  switch (parsed.action) {
    case 'book':
    case 'check_availability': {
      if (parsed.date && parsed.time && parsed.service) {
        // We have all info -- try to book
        const available = getAvailableSlots(parsed.date);
        if (available.includes(parsed.time)) {
          const name = parsed.name || state.pending_name || 'Paciente';
          const success = bookSlot(parsed.date, parsed.time, phone, name, parsed.service);

          if (success) {
            const dateFormatted = formatDate(parsed.date);
            reply = `Perfecto! Tu turno quedo confirmado:\n\n` +
              `Fecha: ${dateFormatted}\n` +
              `Hora: ${parsed.time}\n` +
              `Servicio: ${parsed.service.replace('_', ' ')}\n` +
              `Paciente: ${name}\n\n` +
              `Te vamos a enviar un recordatorio el dia anterior. Si necesitas cancelar o reprogramar, escribinos por aca.`;
            newStage = 'booked';

            state.appointments = [...(state.appointments || []), {
              date: parsed.date,
              time: parsed.time,
              service: parsed.service,
              name,
              booked_at: new Date().toISOString(),
              status: 'confirmed',
            }];
          } else {
            reply = `Disculpa, ese horario acaba de ser reservado. Los horarios disponibles para ${formatDate(parsed.date)} son:\n\n` +
              available.filter(s => s !== parsed.time).slice(0, 6).map(s => `- ${s}`).join('\n') +
              `\n\nCual te queda mejor?`;
            newStage = 'collecting_info';
          }
        } else {
          const available2 = getAvailableSlots(parsed.date);
          if (available2.length === 0) {
            reply = `Disculpa, no tenemos disponibilidad para el ${formatDate(parsed.date)}. Queres que busque otro dia?`;
          } else {
            reply = `Ese horario no esta disponible. Para el ${formatDate(parsed.date)} tenemos:\n\n` +
              available2.slice(0, 8).map(s => `- ${s}`).join('\n') +
              `\n\nCual preferis?`;
          }
          newStage = 'collecting_info';
        }
      } else if (parsed.date) {
        // Have date, show availability
        const available = getAvailableSlots(parsed.date);
        if (available.length === 0) {
          reply = `No tenemos turnos disponibles para el ${formatDate(parsed.date)}. Queres que busque otro dia?`;
        } else {
          reply = `Turnos disponibles para el ${formatDate(parsed.date)}:\n\n` +
            available.slice(0, 10).map(s => `- ${s}`).join('\n') +
            (available.length > 10 ? `\n... y ${available.length - 10} mas` : '') +
            `\n\nQue horario te queda bien?` +
            (!parsed.service ? ' Y para que servicio seria?' : '');
        }
        newStage = 'collecting_info';
        state.pending_date = parsed.date;
        state.pending_service = parsed.service || state.pending_service;
      } else {
        newStage = 'collecting_info';
      }
      break;
    }

    case 'cancel': {
      const activeAppointments = (state.appointments || []).filter(a => a.status === 'confirmed');
      if (activeAppointments.length > 0 && parsed.date) {
        const apt = activeAppointments.find(a => a.date === parsed.date);
        if (apt) {
          cancelSlot(apt.date, apt.time);
          apt.status = 'cancelled';
          reply = `Tu turno del ${formatDate(apt.date)} a las ${apt.time} fue cancelado. Queres agendar otro?`;
          newStage = 'idle';
        }
      } else if (activeAppointments.length === 1) {
        const apt = activeAppointments[0];
        cancelSlot(apt.date, apt.time);
        apt.status = 'cancelled';
        reply = `Tu turno del ${formatDate(apt.date)} a las ${apt.time} fue cancelado. Queres agendar otro?`;
        newStage = 'idle';
      }
      break;
    }

    case 'list_services': {
      reply = `Nuestros servicios:\n\n` +
        `- Limpieza dental: 30 min - $15,000\n` +
        `- Revision general: 30 min - $12,000\n` +
        `- Blanqueamiento: 60 min - $45,000\n` +
        `- Consulta ortodoncia: 30 min - $8,000\n` +
        `- Extraccion: 30 min - $25,000\n\n` +
        `Para cual te gustaria sacar turno?`;
      newStage = 'collecting_info';
      break;
    }

    default:
      break;
  }

  // Update state
  state.stage = newStage;
  state.pending_name = parsed.name || state.pending_name;
  state.pending_date = parsed.date || state.pending_date;
  state.pending_time = parsed.time || state.pending_time;
  state.pending_service = parsed.service || state.pending_service;
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

  console.log(`[${new Date().toISOString()}] Scheduling request from ${phone}: ${messageText}`);

  try {
    await sendTyping(phone);

    const { reply, state } = await handleMessage(phone, messageText);

    await new Promise(resolve => setTimeout(resolve, 1500));

    await sendText(phone, reply, {
      agent: 'appointment-scheduler',
      model: 'claude-haiku-4-20250414',
      scheduling_action: state.stage,
    });

    await updateConversation(phone, { scheduling_state: state });

    console.log(`[${new Date().toISOString()}] Reply sent to ${phone}, stage: ${state.stage}`);
  } catch (err) {
    console.error(`Error handling scheduling from ${phone}:`, err.message);
    await sendText(phone, 'Disculpa, tuve un problema procesando tu pedido. Podes intentar de nuevo o llamarnos al (351) 555-1234.', {
      agent: 'appointment-scheduler',
      error: true,
    });
  }
});

// ── Reminder cron (run every hour) ────────────────────────────────────
async function sendReminders() {
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  const tomorrowStr = tomorrow.toLocaleDateString('en-CA', { timeZone: TIMEZONE });

  console.log(`[Reminders] Checking appointments for ${tomorrowStr}`);

  for (const [slotKey, booking] of bookings.entries()) {
    if (!slotKey.startsWith(tomorrowStr)) continue;
    if (booking.reminder_sent) continue;

    const time = slotKey.split('_')[1];
    try {
      await sendText(booking.phone,
        `Hola ${booking.name}! Te recordamos que manana tenes turno a las ${time} en Clinica Dental Sonrisa.\n\n` +
        `Servicio: ${booking.service.replace('_', ' ')}\n` +
        `Direccion: Av. Colon 1234, Cordoba\n\n` +
        `Si necesitas cancelar o reprogramar, respondeme a este mensaje.`,
        {
          agent: 'appointment-scheduler',
          type: 'reminder',
          appointment_date: tomorrowStr,
          appointment_time: time,
        }
      );
      booking.reminder_sent = true;
      console.log(`[Reminders] Sent reminder to ${booking.phone} for ${slotKey}`);
    } catch (err) {
      console.error(`[Reminders] Failed to send reminder to ${booking.phone}:`, err.message);
    }
  }
}

// Run reminder check every hour
setInterval(sendReminders, 60 * 60 * 1000);

// ── View bookings ─────────────────────────────────────────────────────
app.get('/bookings', (req, res) => {
  const { date } = req.query;
  const entries = [...bookings.entries()]
    .filter(([key]) => !date || key.startsWith(date))
    .map(([key, booking]) => {
      const [d, t] = key.split('_');
      return { date: d, time: t, ...booking };
    })
    .sort((a, b) => `${a.date}_${a.time}`.localeCompare(`${b.date}_${b.time}`));

  res.json({ bookings: entries, total: entries.length });
});

app.get('/availability', (req, res) => {
  const { date } = req.query;
  if (!date) return res.status(400).json({ error: 'date query parameter required (YYYY-MM-DD)' });
  const slots = getAvailableSlots(date);
  res.json({ date, available_slots: slots, total: slots.length });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', total_bookings: bookings.size, uptime: process.uptime() });
});

const PORT = process.env.PORT || 3002;
app.listen(PORT, () => {
  console.log(`Appointment scheduler listening on port ${PORT}`);
  sendReminders(); // Check on startup
});
```

## Metadata Schema

### Conversation-level metadata

```json
{
  "scheduling_state": {
    "stage": "booked",
    "pending_date": null,
    "pending_time": null,
    "pending_service": null,
    "pending_name": "Maria Lopez",
    "last_interaction": "2026-04-02T15:00:00.000Z",
    "appointments": [
      {
        "date": "2026-04-05",
        "time": "10:00",
        "service": "limpieza",
        "name": "Maria Lopez",
        "booked_at": "2026-04-02T15:00:00.000Z",
        "status": "confirmed"
      }
    ]
  }
}
```

### Message-level metadata

```json
{
  "agent": "appointment-scheduler",
  "model": "claude-haiku-4-20250414",
  "scheduling_action": "booked",
  "type": "reminder",
  "appointment_date": "2026-04-05",
  "appointment_time": "10:00"
}
```

## How to Run

```bash
# 1. Create the project
mkdir appointment-scheduler && cd appointment-scheduler
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3002

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. Check availability
curl "http://localhost:3002/availability?date=2026-04-05"

# 7. View bookings
curl "http://localhost:3002/bookings?date=2026-04-05"
```
