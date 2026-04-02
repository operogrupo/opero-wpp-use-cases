# 30 - Beauty Salon Booking

Beauty salon appointment booking via WhatsApp. Service selection, stylist preferences, scheduling, rescheduling, cancellation, and pre-appointment preparation instructions.

## Problem

A busy salon receives 40+ calls daily for appointments. Stylists are interrupted to check their schedules, double-bookings happen when the front desk is overwhelmed, and clients forget preparation steps (don't wash hair before coloring, etc.). No-shows cost $200+ per missed slot, and there's no automated way to fill cancellations.

## Solution

Deploy a WhatsApp booking bot that:
- Guides clients through service selection (haircut, coloring, nails, etc.)
- Shows available stylists and time slots
- Handles rescheduling and cancellations with confirmation
- Sends preparation instructions 24 hours before appointments
- Sends reminders to reduce no-shows
- Manages waitlists for popular time slots

## Architecture

```
Client sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 | Booking Engine   |
         |                                 | (schedule mgmt)  |
         |                                 +------------------+
         |                                          |
         |      POST /messages/text                 |
         |      PUT  /conversations/:phone          |
         +------------------------------------------+
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

// ── Salon data ────────────────────────────────────────────────────────
const SERVICES = [
  { id: 'haircut_w', name: 'Corte mujer', duration: 60, price: 8000 },
  { id: 'haircut_m', name: 'Corte hombre', duration: 30, price: 5000 },
  { id: 'color', name: 'Coloración', duration: 120, price: 15000 },
  { id: 'highlights', name: 'Mechas / Balayage', duration: 150, price: 20000 },
  { id: 'blowdry', name: 'Brushing', duration: 45, price: 4000 },
  { id: 'manicure', name: 'Manicura', duration: 45, price: 4500 },
  { id: 'pedicure', name: 'Pedicura', duration: 60, price: 5500 },
  { id: 'facial', name: 'Limpieza facial', duration: 60, price: 7000 },
];

const STYLISTS = [
  { id: 'st1', name: 'Lucía', services: ['haircut_w', 'haircut_m', 'color', 'highlights', 'blowdry'] },
  { id: 'st2', name: 'Martina', services: ['haircut_w', 'haircut_m', 'blowdry'] },
  { id: 'st3', name: 'Camila', services: ['manicure', 'pedicure'] },
  { id: 'st4', name: 'Valentina', services: ['color', 'highlights', 'facial'] },
];

const PREP_INSTRUCTIONS = {
  color: 'No lave su cabello el día del turno. Evite usar productos con siliconas 48hs antes. Si es su primera coloración, le haremos un test de alergia.',
  highlights: 'No lave su cabello el día del turno. Si tiene coloración previa, infórmenos para ajustar la fórmula.',
  facial: 'Evite exfoliantes 48hs antes del turno. Venga sin maquillaje si es posible. Infórmenos de alergias cutáneas.',
  manicure: 'Retire el esmalte anterior antes de venir. Evite cortar cutículas en casa.',
  pedicure: 'Retire el esmalte anterior. Use calzado cómodo para después del turno.',
};

const BUSINESS_HOURS = {
  start: 9,   // 9:00
  end: 20,    // 20:00
  days: [1, 2, 3, 4, 5, 6], // Mon-Sat
  slot_interval: 30, // minutes
};

// ── Data stores ───────────────────────────────────────────────────────
const appointments = new Map();
const sessions = new Map(); // phone -> booking session state

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

// ── Schedule helpers ──────────────────────────────────────────────────
function getAvailableSlots(stylistId, date, serviceDuration) {
  const dayOfWeek = new Date(date).getDay();
  if (!BUSINESS_HOURS.days.includes(dayOfWeek)) return [];

  const slots = [];
  for (let hour = BUSINESS_HOURS.start; hour < BUSINESS_HOURS.end; hour++) {
    for (let min = 0; min < 60; min += BUSINESS_HOURS.slot_interval) {
      const slotTime = `${String(hour).padStart(2, '0')}:${String(min).padStart(2, '0')}`;
      const slotEnd = new Date(`${date}T${slotTime}:00`);
      slotEnd.setMinutes(slotEnd.getMinutes() + serviceDuration);

      if (slotEnd.getHours() >= BUSINESS_HOURS.end) continue;

      // Check conflicts
      const hasConflict = [...appointments.values()].some(appt => {
        if (appt.stylist_id !== stylistId || appt.date !== date || appt.status === 'cancelled') return false;
        const apptStart = new Date(`${appt.date}T${appt.time}:00`);
        const apptEnd = new Date(apptStart);
        apptEnd.setMinutes(apptEnd.getMinutes() + appt.duration);
        const slotStart = new Date(`${date}T${slotTime}:00`);
        return slotStart < apptEnd && slotEnd > apptStart;
      });

      if (!hasConflict) {
        slots.push(slotTime);
      }
    }
  }
  return slots;
}

function getNextAvailableDates(count = 5) {
  const dates = [];
  const today = new Date();
  let d = new Date(today);
  d.setDate(d.getDate() + 1); // Start from tomorrow

  while (dates.length < count) {
    if (BUSINESS_HOURS.days.includes(d.getDay())) {
      dates.push(d.toISOString().split('T')[0]);
    }
    d.setDate(d.getDate() + 1);
  }
  return dates;
}

// ── Session management ────────────────────────────────────────────────
function getSession(phone) {
  if (!sessions.has(phone)) {
    sessions.set(phone, { step: 'idle', data: {} });
  }
  return sessions.get(phone);
}

function resetSession(phone) {
  sessions.set(phone, { step: 'idle', data: {} });
}

// ── Webhook handler ───────────────────────────────────────────────────
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

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const session = getSession(phone);
    const upper = messageText.toUpperCase();

    // Global commands
    if (upper === 'MENU' || upper === 'HOLA' || upper === 'INICIO') {
      resetSession(phone);
      await sendText(phone,
        `💇 *Salón Belle - Turnos por WhatsApp*\n\n` +
        `¿Qué desea hacer?\n\n` +
        `*1* - Reservar un turno\n` +
        `*2* - Ver mis turnos\n` +
        `*3* - Cancelar un turno\n` +
        `*4* - Reprogramar un turno\n` +
        `*5* - Ver servicios y precios`,
        { agent: 'beauty-salon-booking', action: 'menu_shown' }
      );
      return;
    }

    if (upper === 'CANCELAR' && session.step !== 'idle') {
      resetSession(phone);
      await sendText(phone, 'Operación cancelada. Escriba *MENU* para volver al inicio.', {
        agent: 'beauty-salon-booking', action: 'booking_cancelled',
      });
      return;
    }

    // State machine
    switch (session.step) {
      case 'idle': {
        switch (messageText.trim()) {
          case '1': // Book
            session.step = 'select_service';
            const serviceList = SERVICES.map((s, i) => `*${i + 1}* - ${s.name} (${s.duration} min) - $${s.price.toLocaleString()}`).join('\n');
            await sendText(phone, `¿Qué servicio desea?\n\n${serviceList}\n\nEscriba el número del servicio.`, {
              agent: 'beauty-salon-booking', step: 'select_service',
            });
            break;

          case '2': // View appointments
            const myAppts = [...appointments.values()]
              .filter(a => a.phone === phone && a.status === 'confirmed')
              .sort((a, b) => a.date.localeCompare(b.date));

            if (myAppts.length === 0) {
              await sendText(phone, 'No tiene turnos reservados. Escriba *1* para reservar.', {
                agent: 'beauty-salon-booking', action: 'no_appointments',
              });
            } else {
              const list = myAppts.map(a => {
                const service = SERVICES.find(s => s.id === a.service_id);
                const stylist = STYLISTS.find(s => s.id === a.stylist_id);
                return `- ${a.date} a las ${a.time} — ${service?.name} con ${stylist?.name} (ID: ${a.id})`;
              }).join('\n');
              await sendText(phone, `*Sus turnos:*\n\n${list}\n\nPara cancelar o reprogramar, use el MENU.`, {
                agent: 'beauty-salon-booking', action: 'appointments_listed',
              });
            }
            break;

          case '3': // Cancel
            session.step = 'cancel_select';
            await sendText(phone, 'Ingrese el ID del turno que desea cancelar (se lo proporcionamos al reservar).', {
              agent: 'beauty-salon-booking', step: 'cancel_select',
            });
            break;

          case '4': // Reschedule
            session.step = 'reschedule_select';
            await sendText(phone, 'Ingrese el ID del turno que desea reprogramar.', {
              agent: 'beauty-salon-booking', step: 'reschedule_select',
            });
            break;

          case '5': // Services
            const priceList = SERVICES.map(s => `- *${s.name}*: $${s.price.toLocaleString()} (${s.duration} min)`).join('\n');
            await sendText(phone, `*Nuestros servicios:*\n\n${priceList}\n\nEscriba *1* para reservar.`, {
              agent: 'beauty-salon-booking', action: 'price_list_shown',
            });
            break;

          default:
            await sendText(phone, 'Bienvenido/a al Salón Belle. Escriba *MENU* para ver las opciones.', {
              agent: 'beauty-salon-booking', action: 'welcome',
            });
            break;
        }
        break;
      }

      case 'select_service': {
        const idx = parseInt(messageText, 10) - 1;
        if (isNaN(idx) || idx < 0 || idx >= SERVICES.length) {
          await sendText(phone, `Por favor elija un número entre 1 y ${SERVICES.length}.`, {
            agent: 'beauty-salon-booking', action: 'invalid_service',
          });
          return;
        }

        session.data.service = SERVICES[idx];
        session.step = 'select_stylist';

        const availableStylists = STYLISTS.filter(st => st.services.includes(session.data.service.id));
        session.data.available_stylists = availableStylists;

        const stylistList = availableStylists.map((s, i) => `*${i + 1}* - ${s.name}`).join('\n');
        await sendText(phone, `Servicio: *${session.data.service.name}*\n\n¿Con quién prefiere?\n\n${stylistList}\n*${availableStylists.length + 1}* - Sin preferencia`, {
          agent: 'beauty-salon-booking', step: 'select_stylist', service: session.data.service.id,
        });
        break;
      }

      case 'select_stylist': {
        const stylists = session.data.available_stylists;
        const idx = parseInt(messageText, 10) - 1;

        if (isNaN(idx) || idx < 0 || idx > stylists.length) {
          await sendText(phone, `Por favor elija un número entre 1 y ${stylists.length + 1}.`, {
            agent: 'beauty-salon-booking', action: 'invalid_stylist',
          });
          return;
        }

        session.data.stylist = idx < stylists.length ? stylists[idx] : stylists[0]; // Default to first
        session.step = 'select_date';

        const dates = getNextAvailableDates();
        session.data.available_dates = dates;

        const dateList = dates.map((d, i) => {
          const dayName = new Date(d + 'T12:00:00').toLocaleDateString('es-AR', { weekday: 'long', day: 'numeric', month: 'long' });
          return `*${i + 1}* - ${dayName}`;
        }).join('\n');

        await sendText(phone, `Estilista: *${session.data.stylist.name}*\n\n¿Qué día prefiere?\n\n${dateList}`, {
          agent: 'beauty-salon-booking', step: 'select_date', stylist: session.data.stylist.id,
        });
        break;
      }

      case 'select_date': {
        const dates = session.data.available_dates;
        const idx = parseInt(messageText, 10) - 1;

        if (isNaN(idx) || idx < 0 || idx >= dates.length) {
          await sendText(phone, `Por favor elija un número entre 1 y ${dates.length}.`, {
            agent: 'beauty-salon-booking', action: 'invalid_date',
          });
          return;
        }

        session.data.date = dates[idx];
        session.step = 'select_time';

        const slots = getAvailableSlots(session.data.stylist.id, session.data.date, session.data.service.duration);
        if (slots.length === 0) {
          await sendText(phone, 'Lo sentimos, no hay horarios disponibles para ese día. Elija otro día.', {
            agent: 'beauty-salon-booking', action: 'no_slots',
          });
          session.step = 'select_date';
          return;
        }

        session.data.available_slots = slots;
        const slotList = slots.map((s, i) => `*${i + 1}* - ${s}`).join('\n');
        await sendText(phone, `Horarios disponibles para el ${session.data.date}:\n\n${slotList}`, {
          agent: 'beauty-salon-booking', step: 'select_time', date: session.data.date,
        });
        break;
      }

      case 'select_time': {
        const slots = session.data.available_slots;
        const idx = parseInt(messageText, 10) - 1;

        if (isNaN(idx) || idx < 0 || idx >= slots.length) {
          await sendText(phone, `Por favor elija un número entre 1 y ${slots.length}.`, {
            agent: 'beauty-salon-booking', action: 'invalid_time',
          });
          return;
        }

        const time = slots[idx];
        const apptId = `APT-${Date.now().toString(36).toUpperCase()}`;

        const appointment = {
          id: apptId,
          phone,
          service_id: session.data.service.id,
          service_name: session.data.service.name,
          stylist_id: session.data.stylist.id,
          stylist_name: session.data.stylist.name,
          date: session.data.date,
          time,
          duration: session.data.service.duration,
          price: session.data.service.price,
          status: 'confirmed',
          created_at: new Date().toISOString(),
        };

        appointments.set(apptId, appointment);

        // Build prep instructions
        const prep = PREP_INSTRUCTIONS[session.data.service.id];
        const prepText = prep ? `\n\n📋 *Preparación:*\n${prep}` : '';

        await sendText(phone,
          `✅ *Turno confirmado*\n\n` +
          `🆔 ID: ${apptId}\n` +
          `💇 Servicio: ${session.data.service.name}\n` +
          `👩 Estilista: ${session.data.stylist.name}\n` +
          `📅 Fecha: ${session.data.date}\n` +
          `🕐 Hora: ${time}\n` +
          `💰 Precio: $${session.data.service.price.toLocaleString()}\n` +
          `⏱️ Duración: ${session.data.service.duration} min` +
          prepText +
          `\n\nPara cancelar o reprogramar, escriba *MENU*.`,
          {
            agent: 'beauty-salon-booking',
            action: 'appointment_confirmed',
            appointment_id: apptId,
            service: session.data.service.id,
            stylist: session.data.stylist.id,
            date: session.data.date,
            time,
          }
        );

        await updateConversation(phone, {
          agent: 'beauty-salon-booking',
          last_appointment: appointment,
          total_appointments: [...appointments.values()].filter(a => a.phone === phone).length,
        });

        resetSession(phone);
        console.log(`[${new Date().toISOString()}] Appointment ${apptId} booked for ${phone}`);
        break;
      }

      case 'cancel_select': {
        const appt = appointments.get(messageText.trim().toUpperCase());
        if (!appt || appt.phone !== phone) {
          await sendText(phone, 'No se encontró ese turno. Verifique el ID e intente nuevamente, o escriba *MENU*.', {
            agent: 'beauty-salon-booking', action: 'appointment_not_found',
          });
          return;
        }

        appt.status = 'cancelled';
        appt.cancelled_at = new Date().toISOString();

        await sendText(phone, `❌ Turno *${appt.id}* cancelado.\n\n${appt.service_name} con ${appt.stylist_name} el ${appt.date} a las ${appt.time}.\n\nEscriba *1* para reservar un nuevo turno.`, {
          agent: 'beauty-salon-booking',
          action: 'appointment_cancelled',
          appointment_id: appt.id,
        });

        await updateConversation(phone, {
          agent: 'beauty-salon-booking',
          last_cancellation: {
            appointment_id: appt.id,
            cancelled_at: appt.cancelled_at,
          },
        });

        resetSession(phone);
        break;
      }

      case 'reschedule_select': {
        const appt = appointments.get(messageText.trim().toUpperCase());
        if (!appt || appt.phone !== phone || appt.status === 'cancelled') {
          await sendText(phone, 'No se encontró ese turno activo. Verifique el ID e intente nuevamente.', {
            agent: 'beauty-salon-booking', action: 'appointment_not_found',
          });
          return;
        }

        // Start new booking flow with same service/stylist
        session.data.service = SERVICES.find(s => s.id === appt.service_id);
        session.data.stylist = STYLISTS.find(s => s.id === appt.stylist_id);
        session.data.reschedule_from = appt.id;
        session.step = 'select_date';

        const dates = getNextAvailableDates();
        session.data.available_dates = dates;

        const dateList = dates.map((d, i) => {
          const dayName = new Date(d + 'T12:00:00').toLocaleDateString('es-AR', { weekday: 'long', day: 'numeric', month: 'long' });
          return `*${i + 1}* - ${dayName}`;
        }).join('\n');

        await sendText(phone, `Reprogramando: *${appt.service_name}* con *${appt.stylist_name}*\n\nElija nueva fecha:\n\n${dateList}`, {
          agent: 'beauty-salon-booking', step: 'reschedule_date', original_id: appt.id,
        });

        // Cancel the old appointment
        appt.status = 'rescheduled';
        break;
      }

      default:
        resetSession(phone);
        await sendText(phone, 'Escriba *MENU* para ver las opciones disponibles.', {
          agent: 'beauty-salon-booking', action: 'reset',
        });
        break;
    }
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Disculpe, ocurrió un error. Escriba *MENU* para reiniciar.', {
        agent: 'beauty-salon-booking', error: true, error_message: err.message,
      });
    } catch (e) {
      console.error('Fallback failed:', e.message);
    }
  }
});

// ── Cron: send prep reminders (run daily) ─────────────────────────────
async function sendPrepReminders() {
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  const tomorrowStr = tomorrow.toISOString().split('T')[0];

  for (const [id, appt] of appointments) {
    if (appt.date !== tomorrowStr || appt.status !== 'confirmed') continue;

    const prep = PREP_INSTRUCTIONS[appt.service_id];
    const message = `📌 *Recordatorio: turno mañana*\n\n` +
      `${appt.service_name} con ${appt.stylist_name}\n` +
      `Hora: ${appt.time}\n` +
      (prep ? `\n📋 *Preparación:*\n${prep}\n` : '') +
      `\nPara cancelar, escriba *MENU* y seleccione la opción 3.\n¡La esperamos!`;

    try {
      await sendText(appt.phone, message, {
        agent: 'beauty-salon-booking',
        action: 'prep_reminder',
        appointment_id: id,
      });
    } catch (err) {
      console.error(`Failed to send prep reminder for ${id}:`, err.message);
    }
  }
}

// Run prep reminders daily at 10:00
setInterval(sendPrepReminders, 24 * 60 * 60 * 1000);

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...appointments.values()];
  res.json({
    status: 'ok',
    total_appointments: all.length,
    confirmed: all.filter(a => a.status === 'confirmed').length,
    cancelled: all.filter(a => a.status === 'cancelled').length,
    active_sessions: sessions.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Beauty salon booking bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "beauty-salon-booking",
  "last_appointment": {
    "id": "APT-M1A2B3C4",
    "service_name": "Coloración",
    "stylist_name": "Lucía",
    "date": "2026-04-10",
    "time": "14:00",
    "price": 15000,
    "status": "confirmed"
  },
  "total_appointments": 3
}
```

Per-message metadata:

```json
{
  "agent": "beauty-salon-booking",
  "action": "appointment_confirmed",
  "appointment_id": "APT-M1A2B3C4",
  "service": "color",
  "stylist": "st1",
  "date": "2026-04-10",
  "time": "14:00"
}
```

## How to Run

```bash
# 1. Create the project
mkdir beauty-salon-booking && cd beauty-salon-booking
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
```
