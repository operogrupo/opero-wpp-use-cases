# 26 - Legal Intake

Automate law firm client intake via WhatsApp. Collect case details, classify case type with AI, and route to the appropriate attorney.

## Problem

A mid-size law firm receives 30+ intake inquiries per day across phone, email, and walk-ins. Paralegals spend 2+ hours daily on initial screening calls that often lead nowhere. Critical details get lost in handwritten notes, and urgent cases (restraining orders, bail hearings) sometimes wait hours before reaching the right attorney. The firm estimates $8,000/month in lost billable hours from inefficient intake.

## Solution

Deploy a WhatsApp intake bot that:
- Guides clients through a structured intake form (case type, incident details, timeline, urgency)
- Uses AI to classify the case type and assess urgency
- Routes the intake to the appropriate attorney based on specialization
- Stores the complete intake form in conversation metadata for CRM integration
- Flags emergency situations for immediate human escalation

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
         |                                 |  Claude Haiku    |
         |                                 |  (classify case) |
         |                                 +------------------+
         |                                          |
         |      POST /messages/text                 |
         |      PUT  /conversations/:phone          |
         +------------------------------------------+
                                                    |
                                                    v
                                           +------------------+
                                           | Attorney Routing |
                                           | (notify via WPP) |
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

// ── Intake state machine ─────────────────────────────────────────────
const INTAKE_STEPS = [
  {
    key: 'full_name',
    question: 'Bienvenido/a al Estudio Jurídico. Para asistirlo, necesitamos algunos datos.\n\n¿Cuál es su nombre completo?',
  },
  {
    key: 'case_description',
    question: 'Describa brevemente su situación legal. Incluya qué sucedió, cuándo ocurrió y quiénes están involucrados.',
  },
  {
    key: 'timeline',
    question: '¿Cuándo ocurrieron los hechos? ¿Hay alguna fecha límite o audiencia programada?',
  },
  {
    key: 'urgency',
    question: '¿Qué tan urgente considera su caso?\n\n1. Emergencia (riesgo inmediato)\n2. Urgente (necesito respuesta esta semana)\n3. Normal (puedo esperar)\n4. Consulta general',
  },
  {
    key: 'contact_preference',
    question: '¿Cuál es su número de teléfono alternativo y horario preferido de contacto?',
  },
];

// Attorney routing configuration
const ATTORNEYS = {
  family: { name: 'Dra. María López', phone: '5491155551001', specialization: 'Derecho de Familia' },
  criminal: { name: 'Dr. Carlos Ruiz', phone: '5491155551002', specialization: 'Derecho Penal' },
  labor: { name: 'Dra. Ana García', phone: '5491155551003', specialization: 'Derecho Laboral' },
  civil: { name: 'Dr. Martín Pérez', phone: '5491155551004', specialization: 'Derecho Civil' },
  commercial: { name: 'Dra. Laura Fernández', phone: '5491155551005', specialization: 'Derecho Comercial' },
  immigration: { name: 'Dr. Diego Torres', phone: '5491155551006', specialization: 'Migraciones' },
};

// In-memory intake sessions (use Redis in production)
const sessions = new Map();

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

async function markAsRead(phone, messageIds) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/read`, {
    method: 'POST',
    body: JSON.stringify({ phone, message_ids: messageIds }),
  });
}

// ── AI case classification ────────────────────────────────────────────
async function classifyCase(intakeData) {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 512,
    system: `You are a legal intake classifier. Based on the client's description, classify the case and respond in JSON only.

Categories: family, criminal, labor, civil, commercial, immigration

Respond with exactly this JSON structure:
{
  "case_type": "one of the categories above",
  "case_type_label": "human-readable label in Spanish",
  "summary": "2-3 sentence summary of the case in Spanish",
  "urgency_assessment": "emergency|urgent|normal|consultation",
  "key_issues": ["issue1", "issue2"],
  "recommended_documents": ["doc1", "doc2"],
  "conflict_check_names": ["name1", "name2"]
}`,
    messages: [
      {
        role: 'user',
        content: `Client intake information:
Name: ${intakeData.full_name}
Description: ${intakeData.case_description}
Timeline: ${intakeData.timeline}
Self-reported urgency: ${intakeData.urgency}
Contact preference: ${intakeData.contact_preference}`,
      },
    ],
  });

  try {
    return JSON.parse(response.content[0].text);
  } catch {
    return {
      case_type: 'civil',
      case_type_label: 'Derecho Civil (clasificación por defecto)',
      summary: intakeData.case_description,
      urgency_assessment: 'normal',
      key_issues: [],
      recommended_documents: [],
      conflict_check_names: [intakeData.full_name],
    };
  }
}

// ── Intake session management ─────────────────────────────────────────
function getSession(phone) {
  if (!sessions.has(phone)) {
    sessions.set(phone, {
      step: -1, // -1 means not started
      data: {},
      started_at: new Date().toISOString(),
      completed: false,
    });
  }
  return sessions.get(phone);
}

// ── Webhook handler ───────────────────────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received') return;
  if (data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';

  if (!messageText.trim()) return;

  console.log(`[${new Date().toISOString()}] Intake message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    await sendTyping(phone);

    const session = getSession(phone);

    // Allow restart
    if (messageText.toLowerCase() === 'reiniciar') {
      sessions.delete(phone);
      const freshSession = getSession(phone);
      freshSession.step = 0;
      await sendText(phone, INTAKE_STEPS[0].question, {
        agent: 'legal-intake',
        step: 'full_name',
        action: 'restart',
      });
      return;
    }

    // If intake is already completed
    if (session.completed) {
      await sendText(phone, 'Su consulta ya fue registrada y derivada al abogado correspondiente. Si necesita realizar una nueva consulta, escriba "reiniciar".', {
        agent: 'legal-intake',
        action: 'already_completed',
      });
      return;
    }

    // First message — start the intake
    if (session.step === -1) {
      session.step = 0;
      await sendText(phone, INTAKE_STEPS[0].question, {
        agent: 'legal-intake',
        step: 'welcome',
        action: 'start_intake',
      });
      return;
    }

    // Store the answer for the current step
    const currentStep = INTAKE_STEPS[session.step];
    session.data[currentStep.key] = messageText;

    // Validate urgency input
    if (currentStep.key === 'urgency') {
      const urgencyMap = { '1': 'Emergencia', '2': 'Urgente', '3': 'Normal', '4': 'Consulta general' };
      session.data.urgency = urgencyMap[messageText.trim()] || messageText;

      // Emergency escalation
      if (messageText.trim() === '1') {
        await sendText(phone, '⚠️ Entendemos que su situación es una emergencia. Estamos contactando a un abogado de guardia de inmediato. Por favor, si está en peligro, llame al 911.', {
          agent: 'legal-intake',
          step: 'emergency_escalation',
          urgency: 'emergency',
        });
      }
    }

    // Move to next step
    session.step++;

    if (session.step < INTAKE_STEPS.length) {
      // Ask next question
      const nextStep = INTAKE_STEPS[session.step];
      await sendText(phone, nextStep.question, {
        agent: 'legal-intake',
        step: nextStep.key,
        progress: `${session.step + 1}/${INTAKE_STEPS.length}`,
      });
    } else {
      // All steps completed — classify and route
      session.completed = true;

      await sendText(phone, 'Gracias por proporcionar toda la información. Estamos analizando su caso...', {
        agent: 'legal-intake',
        action: 'classifying',
      });

      const classification = await classifyCase(session.data);

      // Build full intake record
      const intakeRecord = {
        ...session.data,
        classification,
        phone,
        started_at: session.started_at,
        completed_at: new Date().toISOString(),
        status: 'pending_review',
      };

      // Store in conversation metadata
      await updateConversation(phone, {
        agent: 'legal-intake',
        intake: intakeRecord,
      });

      // Determine attorney
      const attorney = ATTORNEYS[classification.case_type] || ATTORNEYS.civil;

      // Notify the client
      await sendText(phone,
        `Su caso ha sido clasificado como: *${classification.case_type_label}*\n\n` +
        `Resumen: ${classification.summary}\n\n` +
        `Ha sido asignado a *${attorney.name}* (${attorney.specialization}), quien se comunicará con usted a la brevedad.\n\n` +
        `Documentos recomendados para su consulta:\n${classification.recommended_documents.map(d => `- ${d}`).join('\n') || '- Ninguno por el momento'}`,
        {
          agent: 'legal-intake',
          action: 'intake_complete',
          case_type: classification.case_type,
          assigned_attorney: attorney.name,
        }
      );

      // Notify the attorney
      await sendText(attorney.phone,
        `📋 *Nueva consulta asignada*\n\n` +
        `Cliente: ${session.data.full_name}\n` +
        `Teléfono: ${phone}\n` +
        `Tipo: ${classification.case_type_label}\n` +
        `Urgencia: ${classification.urgency_assessment}\n\n` +
        `Resumen: ${classification.summary}\n\n` +
        `Temas clave: ${classification.key_issues.join(', ')}\n` +
        `Verificar conflictos: ${classification.conflict_check_names.join(', ')}`,
        {
          agent: 'legal-intake',
          action: 'attorney_notification',
          client_phone: phone,
          case_type: classification.case_type,
        }
      );

      console.log(`[${new Date().toISOString()}] Intake completed for ${phone}, routed to ${attorney.name}`);
    }
  } catch (err) {
    console.error(`[${new Date().toISOString()}] Error in intake for ${phone}:`, err.message);
    try {
      await sendText(phone, 'Disculpe, ocurrió un error. Por favor intente nuevamente o llame a nuestro número principal: +54 11 4444-0000.', {
        agent: 'legal-intake',
        error: true,
        error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback message failed:', fallbackErr.message);
    }
  }
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    active_intakes: sessions.size,
    completed_intakes: [...sessions.values()].filter(s => s.completed).length,
    uptime: process.uptime(),
  });
});

// ── Start server ──────────────────────────────────────────────────────
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Legal intake bot listening on port ${PORT}`);
  console.log(`Webhook URL: POST http://localhost:${PORT}/webhook`);
});
```

## Metadata Schema

Conversation metadata after intake completion:

```json
{
  "agent": "legal-intake",
  "intake": {
    "full_name": "Juan Carlos Rodríguez",
    "case_description": "Mi ex esposa no me deja ver a mis hijos desde hace 3 meses...",
    "timeline": "Desde enero 2026, no hay audiencias programadas",
    "urgency": "Urgente",
    "contact_preference": "+5491155550123, preferentemente por la tarde",
    "classification": {
      "case_type": "family",
      "case_type_label": "Derecho de Familia",
      "summary": "Caso de régimen de visitas. El padre reporta impedimento de contacto...",
      "urgency_assessment": "urgent",
      "key_issues": ["impedimento de contacto", "régimen de visitas"],
      "recommended_documents": ["DNI", "partida de nacimiento de los menores", "sentencia de divorcio"],
      "conflict_check_names": ["Juan Carlos Rodríguez"]
    },
    "phone": "5491155559999",
    "started_at": "2026-04-02T10:00:00.000Z",
    "completed_at": "2026-04-02T10:05:00.000Z",
    "status": "pending_review"
  }
}
```

Per-message metadata:

```json
{
  "agent": "legal-intake",
  "step": "case_description",
  "progress": "2/5"
}
```

## How to Run

```bash
# 1. Create the project
mkdir legal-intake && cd legal-intake
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
```
