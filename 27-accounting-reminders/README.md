# 27 - Accounting Reminders

Automated tax deadline and document request reminders for accounting firms, with escalating follow-ups and client response tracking.

## Problem

An accounting firm manages 200+ clients with overlapping tax deadlines, quarterly payments, and document requests. Staff spend 15+ hours per month manually calling clients who miss deadlines. Late filings result in penalties that damage client relationships. Documents arrive at the last minute, causing overtime during tax season.

## Solution

Deploy a WhatsApp reminder bot that:
- Sends proactive reminders for tax filing deadlines, quarterly payments, and document requests
- Escalates reminders with increasing urgency (friendly -> firm -> final warning)
- Tracks client acknowledgment and document submission status
- Stores all interactions in conversation metadata for compliance records
- Supports scheduling reminders via a simple API

## Architecture

```
+-------------------+     cron / API call     +--------------------+
|  Reminder Engine  | ----------------------> |  Your Express App  |
|  (scheduled jobs) |                         |   (this code)      |
+-------------------+                         +--------------------+
                                                       |
         +---------------------------------------------+
         |                                             |
         v                                             v
+------------------+                          +------------------+
|   Opero WPP API  |                          | Client responds  |
|  wpp-api.opero.so|  <---  webhook  -------- | via WhatsApp     |
+------------------+                          +------------------+
         |
         v
  PUT /conversations/:phone
  (update metadata with response tracking)
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

// ── Reminder database (use a real DB in production) ───────────────────
const reminders = new Map();
const clientProfiles = new Map();

// ── Escalation levels ─────────────────────────────────────────────────
const ESCALATION_TEMPLATES = {
  tax_deadline: [
    {
      level: 1,
      days_before: 30,
      template: (client, deadline) =>
        `Hola ${client.name}, le recordamos que la fecha límite para ${deadline.description} es el *${deadline.due_date}*.\n\nSi necesita ayuda con la documentación, responda "AYUDA". Si ya lo completó, responda "LISTO".`,
    },
    {
      level: 2,
      days_before: 14,
      template: (client, deadline) =>
        `${client.name}, quedan *14 días* para ${deadline.description} (vence ${deadline.due_date}).\n\nDocumentos pendientes:\n${(deadline.pending_docs || []).map(d => `- ${d}`).join('\n')}\n\nResponda "LISTO" cuando haya enviado todo.`,
    },
    {
      level: 3,
      days_before: 3,
      template: (client, deadline) =>
        `⚠️ *URGENTE* - ${client.name}, ${deadline.description} vence en *3 días* (${deadline.due_date}).\n\nSin la documentación completa no podremos realizar la presentación a tiempo, lo que podría generar multas.\n\nPor favor contáctenos de inmediato o responda "LLAMAR" para que lo llamemos.`,
    },
    {
      level: 4,
      days_before: 0,
      template: (client, deadline) =>
        `🚨 ${client.name}, *HOY* vence ${deadline.description}. Si no recibimos su documentación en las próximas horas, no podremos cumplir con el plazo.\n\nLlámenos al +54 11 4444-3333 o responda "LLAMAR".`,
    },
  ],
  quarterly_payment: [
    {
      level: 1,
      days_before: 7,
      template: (client, deadline) =>
        `Hola ${client.name}, le recordamos que el pago trimestral de ${deadline.description} vence el *${deadline.due_date}*.\n\nMonto estimado: *$${deadline.amount}*\n\nResponda "PAGADO" cuando lo haya abonado.`,
    },
    {
      level: 2,
      days_before: 1,
      template: (client, deadline) =>
        `${client.name}, mañana vence el pago de ${deadline.description} por *$${deadline.amount}*.\n\n¿Necesita los datos de pago? Responda "DATOS".`,
    },
  ],
  document_request: [
    {
      level: 1,
      days_before: 14,
      template: (client, deadline) =>
        `Hola ${client.name}, necesitamos los siguientes documentos para ${deadline.description}:\n\n${deadline.documents.map(d => `- ${d}`).join('\n')}\n\nPor favor envíelos antes del *${deadline.due_date}*. Puede enviar fotos o PDFs por este chat.`,
    },
    {
      level: 2,
      days_before: 7,
      template: (client, deadline) =>
        `${client.name}, aún no recibimos los documentos para ${deadline.description}.\n\nPendientes:\n${deadline.pending_docs.map(d => `- ${d}`).join('\n')}\n\nVencimiento: *${deadline.due_date}*`,
    },
  ],
};

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

// ── Reminder scheduling ──────────────────────────────────────────────
function scheduleReminder(clientPhone, deadline) {
  const id = `${clientPhone}-${deadline.type}-${Date.now()}`;
  const reminder = {
    id,
    client_phone: clientPhone,
    deadline,
    escalation_level: 0,
    created_at: new Date().toISOString(),
    responses: [],
    status: 'active', // active, acknowledged, completed, overdue
  };
  reminders.set(id, reminder);
  return reminder;
}

async function sendReminder(reminderId) {
  const reminder = reminders.get(reminderId);
  if (!reminder || reminder.status === 'completed') return;

  const templates = ESCALATION_TEMPLATES[reminder.deadline.type];
  if (!templates) return;

  const client = clientProfiles.get(reminder.client_phone) || { name: 'Cliente' };
  const template = templates[reminder.escalation_level];
  if (!template) {
    reminder.status = 'overdue';
    return;
  }

  const message = template.template(client, reminder.deadline);

  await sendText(reminder.client_phone, message, {
    agent: 'accounting-reminders',
    reminder_id: reminder.id,
    deadline_type: reminder.deadline.type,
    escalation_level: reminder.escalation_level + 1,
    due_date: reminder.deadline.due_date,
  });

  reminder.escalation_level++;
  reminder.last_sent_at = new Date().toISOString();

  // Update conversation metadata
  await updateConversation(reminder.client_phone, {
    agent: 'accounting-reminders',
    active_reminders: [...reminders.values()]
      .filter(r => r.client_phone === reminder.client_phone && r.status === 'active'),
  });

  console.log(`[${new Date().toISOString()}] Reminder sent: ${reminderId} (level ${reminder.escalation_level})`);
}

// ── Check and send due reminders ──────────────────────────────────────
async function processReminders() {
  const now = new Date();

  for (const [id, reminder] of reminders) {
    if (reminder.status !== 'active') continue;

    const dueDate = new Date(reminder.deadline.due_date);
    const daysUntilDue = Math.ceil((dueDate - now) / (1000 * 60 * 60 * 24));

    const templates = ESCALATION_TEMPLATES[reminder.deadline.type];
    if (!templates || reminder.escalation_level >= templates.length) continue;

    const nextTemplate = templates[reminder.escalation_level];
    if (daysUntilDue <= nextTemplate.days_before) {
      try {
        await sendReminder(id);
      } catch (err) {
        console.error(`Error sending reminder ${id}:`, err.message);
      }
    }
  }
}

// Run reminder check every hour
setInterval(processReminders, 60 * 60 * 1000);

// ── Webhook handler (client responses) ────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received') return;
  if (data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim().toUpperCase();

  if (!messageText) return;

  console.log(`[${new Date().toISOString()}] Response from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    // Find active reminders for this client
    const activeReminders = [...reminders.values()]
      .filter(r => r.client_phone === phone && r.status === 'active');

    if (activeReminders.length === 0) {
      await sendText(phone, 'Gracias por su mensaje. No tiene recordatorios activos en este momento. Si necesita algo, comuníquese con nuestras oficinas.', {
        agent: 'accounting-reminders',
        action: 'no_active_reminders',
      });
      return;
    }

    const latestReminder = activeReminders[activeReminders.length - 1];

    // Handle responses
    switch (messageText) {
      case 'LISTO':
      case 'PAGADO':
        latestReminder.status = 'completed';
        latestReminder.responses.push({
          text: messageText,
          at: new Date().toISOString(),
          action: 'completed',
        });

        await sendText(phone, '✅ Perfecto, registramos su confirmación. Gracias por cumplir a tiempo.', {
          agent: 'accounting-reminders',
          action: 'client_confirmed',
          reminder_id: latestReminder.id,
        });

        await updateConversation(phone, {
          agent: 'accounting-reminders',
          last_response: messageText,
          reminder_completed: latestReminder.id,
          completed_at: new Date().toISOString(),
        });
        break;

      case 'AYUDA':
        latestReminder.responses.push({
          text: messageText,
          at: new Date().toISOString(),
          action: 'help_requested',
        });

        await sendText(phone, 'Un contador se comunicará con usted dentro de las próximas 2 horas para asistirlo. Mientras tanto, ¿tiene alguna consulta específica?', {
          agent: 'accounting-reminders',
          action: 'help_requested',
          reminder_id: latestReminder.id,
        });
        break;

      case 'LLAMAR':
        latestReminder.responses.push({
          text: messageText,
          at: new Date().toISOString(),
          action: 'callback_requested',
        });

        await sendText(phone, 'Anotado. Lo llamaremos a la brevedad.', {
          agent: 'accounting-reminders',
          action: 'callback_requested',
          reminder_id: latestReminder.id,
        });

        // Notify staff
        const STAFF_PHONE = process.env.STAFF_PHONE || '5491155550000';
        await sendText(STAFF_PHONE, `📞 Solicitud de llamada de ${clientProfiles.get(phone)?.name || phone} - Asunto: ${latestReminder.deadline.description}`, {
          agent: 'accounting-reminders',
          action: 'callback_notification',
          client_phone: phone,
        });
        break;

      case 'DATOS':
        await sendText(phone, `Datos para el pago:\n\nBanco: Banco Nación\nCBU: 0110099930009900112233\nAlias: ESTUDIO-CONTABLE\nConcepto: ${latestReminder.deadline.description}\nMonto: $${latestReminder.deadline.amount || 'consultar'}`, {
          agent: 'accounting-reminders',
          action: 'payment_info_sent',
          reminder_id: latestReminder.id,
        });
        break;

      default:
        latestReminder.responses.push({
          text: messageText,
          at: new Date().toISOString(),
          action: 'free_text',
        });

        await sendText(phone, 'Gracias por su mensaje. Lo hemos registrado. Puede responder:\n\n- *LISTO* o *PAGADO* para confirmar\n- *AYUDA* para asistencia\n- *LLAMAR* para que lo contactemos\n- *DATOS* para datos de pago', {
          agent: 'accounting-reminders',
          action: 'unrecognized_response',
        });
        break;
    }
  } catch (err) {
    console.error(`Error handling response from ${phone}:`, err.message);
  }
});

// ── API: Create reminder ──────────────────────────────────────────────
app.post('/api/reminders', async (req, res) => {
  const { client_phone, client_name, deadline } = req.body;

  if (!client_phone || !deadline) {
    return res.status(400).json({ error: 'client_phone and deadline are required' });
  }

  // Store client profile
  if (client_name) {
    clientProfiles.set(client_phone, { name: client_name });
  }

  const reminder = scheduleReminder(client_phone, deadline);

  // Send first reminder immediately if within the first escalation window
  try {
    await sendReminder(reminder.id);
  } catch (err) {
    console.error('Error sending initial reminder:', err.message);
  }

  res.json({ reminder });
});

// ── API: List reminders ───────────────────────────────────────────────
app.get('/api/reminders', (req, res) => {
  const { phone, status } = req.query;
  let results = [...reminders.values()];

  if (phone) results = results.filter(r => r.client_phone === phone);
  if (status) results = results.filter(r => r.status === status);

  res.json({ reminders: results, total: results.length });
});

// ── API: Trigger reminder check ───────────────────────────────────────
app.post('/api/reminders/process', async (req, res) => {
  await processReminders();
  res.json({ status: 'processed' });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...reminders.values()];
  res.json({
    status: 'ok',
    total_reminders: all.length,
    active: all.filter(r => r.status === 'active').length,
    completed: all.filter(r => r.status === 'completed').length,
    overdue: all.filter(r => r.status === 'overdue').length,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Accounting reminders bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "accounting-reminders",
  "active_reminders": [
    {
      "id": "5491155559999-tax_deadline-1711958400000",
      "deadline": {
        "type": "tax_deadline",
        "description": "Declaración Jurada de Ganancias",
        "due_date": "2026-06-30"
      },
      "escalation_level": 2,
      "status": "active",
      "responses": [
        { "text": "AYUDA", "at": "2026-06-01T14:00:00.000Z", "action": "help_requested" }
      ]
    }
  ]
}
```

Per-message metadata:

```json
{
  "agent": "accounting-reminders",
  "reminder_id": "5491155559999-tax_deadline-1711958400000",
  "deadline_type": "tax_deadline",
  "escalation_level": 2,
  "due_date": "2026-06-30"
}
```

## How to Run

```bash
# 1. Create the project
mkdir accounting-reminders && cd accounting-reminders
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

# 6. Create a reminder via the API
curl -X POST http://localhost:3000/api/reminders \
  -H "Content-Type: application/json" \
  -d '{
    "client_phone": "5491155559999",
    "client_name": "María González",
    "deadline": {
      "type": "tax_deadline",
      "description": "Declaración Jurada de Ganancias",
      "due_date": "2026-06-30",
      "pending_docs": ["Certificado de ingresos", "Formulario 649"]
    }
  }'
```
