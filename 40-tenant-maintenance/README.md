# 40 - Tenant Maintenance

Tenant maintenance request system via WhatsApp. Report issues with photos, AI classifies urgency, dispatches to correct maintenance team, and sends status updates throughout the lifecycle.

## Problem

A property management company manages 300+ rental units. Tenants call, email, and text maintenance requests on personal phones -- there's no single system of record. Emergency plumbing issues sit in an email inbox for hours. Staff can't tell if a request is a dripping faucet or a burst pipe without calling back. Tenants have no visibility into when their issue will be fixed, generating repeated follow-up calls.

## Solution

Deploy a WhatsApp maintenance request bot that:
- Accepts issue reports with photos and descriptions
- Uses AI to classify urgency (emergency, high, medium, low)
- Routes to the correct maintenance team (plumbing, electrical, HVAC, general)
- Dispatches the nearest available technician
- Sends status updates through the full request lifecycle
- Stores the complete request history in conversation metadata

## Architecture

```
Tenant sends WhatsApp message
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
         |                                 |  (classify issue)|
         |                                 +------------------+
         |                                          |
         |      POST /messages/text                 |
         |      PUT  /conversations/:phone          |
         +------------------------------------------+
                    |                       |
                    v                       v
           +------------------+    +------------------+
           | Maintenance Team |    | Property Manager |
           | (notified)       |    | (dashboard)      |
           +------------------+    +------------------+
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
const MANAGER_PHONE = process.env.MANAGER_PHONE || '5491155550000';

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// ── Maintenance teams ─────────────────────────────────────────────────
const TEAMS = {
  plumbing: {
    name: 'Plomería',
    phone: '5491155551001',
    tech_name: 'Roberto Sánchez',
    response_time: { emergency: '1 hora', high: '4 horas', medium: '24 horas', low: '48 horas' },
  },
  electrical: {
    name: 'Electricidad',
    phone: '5491155551002',
    tech_name: 'Miguel Fernández',
    response_time: { emergency: '1 hora', high: '4 horas', medium: '24 horas', low: '48 horas' },
  },
  hvac: {
    name: 'Climatización',
    phone: '5491155551003',
    tech_name: 'Carlos López',
    response_time: { emergency: '2 horas', high: '6 horas', medium: '48 horas', low: '72 horas' },
  },
  general: {
    name: 'Mantenimiento General',
    phone: '5491155551004',
    tech_name: 'Juan Pérez',
    response_time: { emergency: '2 horas', high: '8 horas', medium: '48 horas', low: '1 semana' },
  },
  pest_control: {
    name: 'Control de Plagas',
    phone: '5491155551005',
    tech_name: 'Pedro Gómez',
    response_time: { emergency: '4 horas', high: '24 horas', medium: '48 horas', low: '1 semana' },
  },
};

// ── Data stores ───────────────────────────────────────────────────────
const tenants = new Map();   // phone -> tenant info
const requests = new Map();  // requestId -> maintenance request
const sessions = new Map();  // phone -> session state

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

// ── AI issue classification ───────────────────────────────────────────
async function classifyIssue(description, location) {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 512,
    system: `You are a property maintenance issue classifier. Based on the tenant's description, classify the issue and respond in JSON only.

Categories: plumbing, electrical, hvac, general, pest_control

Urgency levels:
- emergency: Immediate safety risk or major property damage (gas leak, electrical fire, flooding, no heat in freezing weather)
- high: Significant but not immediately dangerous (no hot water, toilet not working, broken lock, AC out in extreme heat)
- medium: Inconvenient but livable (leaky faucet, running toilet, broken appliance, minor electrical issue)
- low: Cosmetic or non-urgent (paint peeling, squeaky door, lightbulb replacement, minor damage)

Respond with exactly this JSON:
{
  "category": "one of the categories above",
  "urgency": "emergency|high|medium|low",
  "summary_es": "2-sentence summary in Spanish",
  "safety_warning": "safety warning in Spanish if applicable, or null",
  "recommended_action": "what the tenant should do immediately, in Spanish",
  "estimated_scope": "quick_fix|repair|replacement|inspection"
}`,
    messages: [{
      role: 'user',
      content: `Tenant maintenance request:
Location: ${location || 'not specified'}
Description: ${description}`,
    }],
  });

  try {
    return JSON.parse(response.content[0].text);
  } catch {
    return {
      category: 'general',
      urgency: 'medium',
      summary_es: description,
      safety_warning: null,
      recommended_action: 'Espere la visita del técnico.',
      estimated_scope: 'inspection',
    };
  }
}

// ── Tenant registration ───────────────────────────────────────────────
function getTenant(phone) {
  return tenants.get(phone);
}

function registerTenant(phone, data) {
  tenants.set(phone, {
    phone,
    name: data.name,
    unit: data.unit,
    building: data.building || '',
    registered_at: new Date().toISOString(),
  });
}

// ── Request management ────────────────────────────────────────────────
function createRequest(tenantPhone, data) {
  const id = `MR-${Date.now().toString(36).toUpperCase()}`;
  const tenant = getTenant(tenantPhone);

  const request = {
    id,
    tenant_phone: tenantPhone,
    tenant_name: tenant?.name || '',
    unit: tenant?.unit || data.unit || '',
    building: tenant?.building || '',
    description: data.description,
    location_in_unit: data.location || '',
    has_photos: data.has_photos || false,
    photo_ids: data.photo_ids || [],
    classification: data.classification || null,
    assigned_team: null,
    assigned_tech: null,
    status: 'submitted', // submitted, classified, assigned, scheduled, in_progress, completed, cancelled
    urgency: null,
    scheduled_date: null,
    scheduled_time: null,
    resolution_notes: null,
    timeline: [{
      status: 'submitted',
      timestamp: new Date().toISOString(),
      note: 'Solicitud recibida',
    }],
    created_at: new Date().toISOString(),
    completed_at: null,
  };

  requests.set(id, request);
  return request;
}

async function syncMetadata(phone) {
  const tenant = getTenant(phone);
  const tenantRequests = [...requests.values()]
    .filter(r => r.tenant_phone === phone)
    .map(r => ({
      id: r.id,
      description: r.classification?.summary_es || r.description.substring(0, 80),
      status: r.status,
      urgency: r.urgency,
      assigned_team: r.assigned_team,
      created_at: r.created_at,
    }));

  await updateConversation(phone, {
    agent: 'tenant-maintenance',
    tenant: tenant ? { name: tenant.name, unit: tenant.unit, building: tenant.building } : null,
    requests: tenantRequests,
    active_requests: tenantRequests.filter(r => !['completed', 'cancelled'].includes(r.status)).length,
    total_requests: tenantRequests.length,
  });
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

  const phone = data.from;
  const isText = data.type === 'text';
  const isImage = data.type === 'image';

  const messageText = isText
    ? (typeof data.content === 'string' ? data.content : data.content?.text || '').trim()
    : '';

  console.log(`[${new Date().toISOString()}] Maintenance message from ${phone}: ${isImage ? '[PHOTO]' : messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const tenant = getTenant(phone);
    const session = getSession(phone);
    const upper = messageText.toUpperCase();

    // Check if tenant is registered
    if (!tenant && session.step !== 'register_name' && session.step !== 'register_unit') {
      if (upper === 'MENU' || upper === 'HOLA' || session.step === 'idle') {
        sessions.set(phone, { step: 'register_name', data: {} });
        await sendText(phone,
          `🏠 *Sistema de Mantenimiento*\n\n` +
          `Bienvenido/a. Para registrar su solicitud, necesitamos algunos datos.\n\n` +
          `¿Cuál es su nombre completo?`,
          { agent: 'tenant-maintenance', step: 'register_name' }
        );
        return;
      }
    }

    // Registration flow
    if (session.step === 'register_name') {
      session.data.name = messageText;
      session.step = 'register_unit';
      sessions.set(phone, session);
      await sendText(phone, `¿En qué unidad/departamento vive? (ej: 3B, PB A, Depto 12)`, {
        agent: 'tenant-maintenance', step: 'register_unit',
      });
      return;
    }

    if (session.step === 'register_unit') {
      registerTenant(phone, { name: session.data.name, unit: messageText });
      resetSession(phone);
      await sendText(phone,
        `✅ Registrado: *${session.data.name}* — Unidad *${messageText}*\n\n` +
        `Ahora puede reportar problemas de mantenimiento.\n\n` +
        `¿Qué desea hacer?\n\n` +
        `*1* - Reportar un problema\n` +
        `*2* - Ver mis solicitudes\n` +
        `*3* - Emergencia`,
        { agent: 'tenant-maintenance', action: 'registered' }
      );
      return;
    }

    // Global commands
    if (upper === 'MENU' || upper === 'HOLA' || upper === 'INICIO') {
      resetSession(phone);
      const activeCount = [...requests.values()]
        .filter(r => r.tenant_phone === phone && !['completed', 'cancelled'].includes(r.status)).length;

      await sendText(phone,
        `🏠 *Mantenimiento — ${tenant.name} (${tenant.unit})*\n\n` +
        `${activeCount > 0 ? `Solicitudes activas: ${activeCount}\n\n` : ''}` +
        `¿Qué desea hacer?\n\n` +
        `*1* - Reportar un problema\n` +
        `*2* - Ver mis solicitudes\n` +
        `*3* - Emergencia\n` +
        `*4* - Contactar administración`,
        { agent: 'tenant-maintenance', action: 'menu_shown' }
      );
      return;
    }

    // Emergency shortcut
    if (upper === '3' && session.step === 'idle' || upper === 'EMERGENCIA') {
      await sendText(phone,
        `🚨 *EMERGENCIA*\n\n` +
        `Si hay riesgo inmediato para su seguridad, llame al *911*.\n\n` +
        `Para emergencias de mantenimiento:\n` +
        `🔥 Incendio/gas: llame al *100*\n` +
        `💧 Inundación: cierre la llave de paso\n` +
        `⚡ Problema eléctrico: desconecte el interruptor\n\n` +
        `Describa la emergencia y enviaremos un técnico de inmediato.`,
        { agent: 'tenant-maintenance', action: 'emergency_info', urgency: 'emergency' }
      );
      sessions.set(phone, { step: 'describe_issue', data: { is_emergency: true } });
      return;
    }

    // State machine
    switch (session.step) {
      case 'idle': {
        switch (messageText.trim()) {
          case '1': // Report issue
            sessions.set(phone, { step: 'describe_issue', data: {} });
            await sendText(phone, 'Describa el problema con el mayor detalle posible. ¿Qué ocurre y dónde exactamente dentro de su unidad?', {
              agent: 'tenant-maintenance', step: 'describe_issue',
            });
            break;

          case '2': // View requests
            const myRequests = [...requests.values()]
              .filter(r => r.tenant_phone === phone)
              .sort((a, b) => new Date(b.created_at) - new Date(a.created_at))
              .slice(0, 5);

            if (myRequests.length === 0) {
              await sendText(phone, 'No tiene solicitudes registradas. Escriba *1* para reportar un problema.', {
                agent: 'tenant-maintenance', action: 'no_requests',
              });
            } else {
              const urgencyIcons = { emergency: '🔴', high: '🟠', medium: '🟡', low: '🟢' };
              const statusLabels = {
                submitted: 'Recibida',
                classified: 'Clasificada',
                assigned: 'Asignada',
                scheduled: 'Programada',
                in_progress: 'En ejecución',
                completed: 'Completada',
                cancelled: 'Cancelada',
              };

              const list = myRequests.map(r => {
                const icon = urgencyIcons[r.urgency] || '⚪';
                const status = statusLabels[r.status] || r.status;
                return `${icon} *${r.id}*\n  ${r.classification?.summary_es || r.description.substring(0, 60)}\n  Estado: ${status}${r.scheduled_date ? ` | Fecha: ${r.scheduled_date}` : ''}`;
              }).join('\n\n');

              await sendText(phone, `*Sus solicitudes (últimas 5):*\n\n${list}\n\nEscriba el ID de una solicitud para ver el detalle.`, {
                agent: 'tenant-maintenance', action: 'requests_listed',
              });
            }
            break;

          case '4': // Contact admin
            await sendText(phone, `📞 *Administración*\n\nTeléfono: +54 11 4444-8888\nEmail: admin@edificio.com\nHorario: Lunes a Viernes 9:00-17:00\n\nO escriba su consulta aquí.`, {
              agent: 'tenant-maintenance', action: 'admin_contact',
            });
            break;

          default: {
            // Check if it's a request ID
            const req = requests.get(messageText.toUpperCase());
            if (req && req.tenant_phone === phone) {
              const urgencyLabels = { emergency: 'Emergencia', high: 'Alta', medium: 'Media', low: 'Baja' };
              const statusLabels = {
                submitted: 'Recibida', classified: 'Clasificada', assigned: 'Asignada',
                scheduled: 'Programada', in_progress: 'En ejecución', completed: 'Completada',
              };

              const history = req.timeline.map(t =>
                `${new Date(t.timestamp).toLocaleString('es-AR')} — ${t.note}`
              ).join('\n');

              await sendText(phone,
                `📋 *Solicitud ${req.id}*\n\n` +
                `Estado: *${statusLabels[req.status] || req.status}*\n` +
                `Urgencia: ${urgencyLabels[req.urgency] || req.urgency}\n` +
                `Área: ${TEAMS[req.assigned_team]?.name || 'Por asignar'}\n` +
                `${req.assigned_tech ? `Técnico: ${req.assigned_tech}\n` : ''}` +
                `${req.scheduled_date ? `Fecha programada: ${req.scheduled_date} ${req.scheduled_time || ''}\n` : ''}` +
                `\n*Descripción:*\n${req.description}\n` +
                `\n*Historial:*\n${history}` +
                `${req.resolution_notes ? `\n\n*Resolución:*\n${req.resolution_notes}` : ''}`,
                {
                  agent: 'tenant-maintenance',
                  request_id: req.id,
                  action: 'request_detail',
                }
              );
            } else {
              await sendText(phone, 'Escriba *MENU* para ver las opciones disponibles.', {
                agent: 'tenant-maintenance', action: 'unrecognized',
              });
            }
            break;
          }
        }
        break;
      }

      case 'describe_issue': {
        if (isImage) {
          // Photo received
          if (!session.data.photo_ids) session.data.photo_ids = [];
          session.data.photo_ids.push(data.wa_message_id || data.id);
          session.data.has_photos = true;

          await sendText(phone, '📸 Foto recibida. Puede enviar más fotos o escribir *LISTO* para continuar.', {
            agent: 'tenant-maintenance', action: 'photo_received',
          });
          return;
        }

        if (upper === 'LISTO' && session.data.has_photos) {
          // Photos sent, need description still
          if (!session.data.description) {
            await sendText(phone, 'Ahora describa el problema con palabras.', {
              agent: 'tenant-maintenance', step: 'describe_issue',
            });
            return;
          }
        } else if (upper !== 'LISTO') {
          session.data.description = messageText;
        }

        if (!session.data.description) return;

        session.step = 'location_in_unit';
        sessions.set(phone, session);

        await sendText(phone, '¿En qué parte de su unidad está el problema?\n\n1. Cocina\n2. Baño\n3. Dormitorio\n4. Living/Comedor\n5. Balcón/Terraza\n6. Pasillo/Entrada\n7. Otro', {
          agent: 'tenant-maintenance', step: 'location_in_unit',
        });
        break;
      }

      case 'location_in_unit': {
        const locationMap = {
          '1': 'Cocina', '2': 'Baño', '3': 'Dormitorio',
          '4': 'Living/Comedor', '5': 'Balcón/Terraza', '6': 'Pasillo/Entrada', '7': 'Otro',
        };
        session.data.location = locationMap[messageText.trim()] || messageText;

        // Classify with AI
        await sendText(phone, '⏳ Analizando su solicitud...', {
          agent: 'tenant-maintenance', action: 'classifying',
        });

        const classification = await classifyIssue(session.data.description, session.data.location);

        // Create the request
        const request = createRequest(phone, {
          description: session.data.description,
          location: session.data.location,
          has_photos: session.data.has_photos || false,
          photo_ids: session.data.photo_ids || [],
          classification,
        });

        request.classification = classification;
        request.urgency = classification.urgency;
        request.assigned_team = classification.category;

        const team = TEAMS[classification.category] || TEAMS.general;
        request.assigned_tech = team.tech_name;

        // Update timeline
        request.timeline.push({
          status: 'classified',
          timestamp: new Date().toISOString(),
          note: `Clasificado: ${team.name} — Urgencia: ${classification.urgency}`,
        });
        request.timeline.push({
          status: 'assigned',
          timestamp: new Date().toISOString(),
          note: `Asignado a ${team.tech_name} (${team.name})`,
        });
        request.status = 'assigned';

        const urgencyLabels = { emergency: 'Emergencia', high: 'Alta', medium: 'Media', low: 'Baja' };
        const urgencyIcons = { emergency: '🔴', high: '🟠', medium: '🟡', low: '🟢' };

        // Notify tenant
        await sendText(phone,
          `✅ *Solicitud registrada: ${request.id}*\n\n` +
          `${urgencyIcons[classification.urgency]} Urgencia: *${urgencyLabels[classification.urgency]}*\n` +
          `🔧 Área: *${team.name}*\n` +
          `👷 Técnico: ${team.tech_name}\n` +
          `⏱️ Tiempo de respuesta: ${team.response_time[classification.urgency]}\n\n` +
          `${classification.summary_es}\n` +
          `${classification.safety_warning ? `\n⚠️ *${classification.safety_warning}*\n` : ''}` +
          `${classification.recommended_action ? `\n💡 ${classification.recommended_action}` : ''}\n\n` +
          `Le avisaremos cuando el técnico confirme la visita.`,
          {
            agent: 'tenant-maintenance',
            request_id: request.id,
            action: 'request_created',
            category: classification.category,
            urgency: classification.urgency,
          }
        );

        // Notify maintenance team
        await sendText(team.phone,
          `🔧 *Nueva solicitud: ${request.id}*\n\n` +
          `${urgencyIcons[classification.urgency]} Urgencia: *${urgencyLabels[classification.urgency]}*\n\n` +
          `Inquilino: ${tenant.name}\n` +
          `Unidad: ${tenant.unit} ${tenant.building}\n` +
          `Ubicación: ${session.data.location}\n` +
          `Teléfono: ${phone}\n\n` +
          `Problema: ${session.data.description}\n` +
          `Fotos: ${session.data.has_photos ? 'Sí' : 'No'}\n\n` +
          `Evaluación: ${classification.summary_es}\n` +
          `Alcance: ${classification.estimated_scope}\n\n` +
          `Responda *ACEPTO ${request.id}* para confirmar.`,
          {
            agent: 'tenant-maintenance',
            request_id: request.id,
            action: 'team_notification',
            urgency: classification.urgency,
          }
        );

        // Notify manager for emergencies
        if (classification.urgency === 'emergency') {
          await sendText(MANAGER_PHONE,
            `🚨 *EMERGENCIA: ${request.id}*\n\n` +
            `Inquilino: ${tenant.name} (${tenant.unit})\n` +
            `${classification.summary_es}\n` +
            `Asignado a: ${team.tech_name}`,
            {
              agent: 'tenant-maintenance',
              request_id: request.id,
              action: 'manager_alert',
              urgency: 'emergency',
            }
          );
        }

        await syncMetadata(phone);
        resetSession(phone);

        console.log(`[${new Date().toISOString()}] Request ${request.id} created: ${classification.category} / ${classification.urgency}`);
        break;
      }

      default:
        resetSession(phone);
        await sendText(phone, 'Escriba *MENU* para ver las opciones.', {
          agent: 'tenant-maintenance', action: 'reset',
        });
        break;
    }
  } catch (err) {
    console.error(`Error from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Disculpe, ocurrió un error. Escriba *MENU* para reintentar o llame al +54 11 4444-8888.', {
        agent: 'tenant-maintenance', error: true, error_message: err.message,
      });
    } catch (e) {
      console.error('Fallback failed:', e.message);
    }
  }
});

// ── API: Register tenant ──────────────────────────────────────────────
app.post('/api/tenants', (req, res) => {
  const { phone, name, unit, building } = req.body;
  if (!phone || !name || !unit) {
    return res.status(400).json({ error: 'phone, name, and unit are required' });
  }
  registerTenant(phone, { name, unit, building });
  res.json({ status: 'registered', tenant: getTenant(phone) });
});

// ── API: Update request status ────────────────────────────────────────
app.put('/api/requests/:id/status', async (req, res) => {
  const request = requests.get(req.params.id);
  if (!request) return res.status(404).json({ error: 'Request not found' });

  const { status, note, scheduled_date, scheduled_time, resolution_notes } = req.body;
  const oldStatus = request.status;
  request.status = status;

  if (scheduled_date) request.scheduled_date = scheduled_date;
  if (scheduled_time) request.scheduled_time = scheduled_time;
  if (resolution_notes) request.resolution_notes = resolution_notes;
  if (status === 'completed') request.completed_at = new Date().toISOString();

  request.timeline.push({
    status,
    timestamp: new Date().toISOString(),
    note: note || `Estado actualizado a ${status}`,
  });

  // Notify tenant
  const statusMessages = {
    scheduled: `📅 Su solicitud *${request.id}* fue programada para el *${scheduled_date}${scheduled_time ? ` a las ${scheduled_time}` : ''}*.\n\nTécnico: ${request.assigned_tech}\n\nPor favor asegúrese de estar disponible.`,
    in_progress: `🔧 El técnico está trabajando en su solicitud *${request.id}*.`,
    completed: `✅ Su solicitud *${request.id}* fue completada.\n\n${resolution_notes ? `Resolución: ${resolution_notes}\n\n` : ''}¿Quedó conforme? Responda:\n*1* - Sí, todo bien\n*2* - No, el problema persiste`,
  };

  const message = statusMessages[status];
  if (message) {
    try {
      await sendText(request.tenant_phone, message, {
        agent: 'tenant-maintenance',
        request_id: request.id,
        action: 'status_update',
        old_status: oldStatus,
        new_status: status,
      });
    } catch (err) {
      console.error('Failed to notify tenant:', err.message);
    }
  }

  await syncMetadata(request.tenant_phone);
  res.json(request);
});

// ── API: List requests ────────────────────────────────────────────────
app.get('/api/requests', (req, res) => {
  const { status, urgency, team, phone } = req.query;
  let results = [...requests.values()];
  if (status) results = results.filter(r => r.status === status);
  if (urgency) results = results.filter(r => r.urgency === urgency);
  if (team) results = results.filter(r => r.assigned_team === team);
  if (phone) results = results.filter(r => r.tenant_phone === phone);
  res.json({ requests: results, total: results.length });
});

// ── API: Dashboard stats ──────────────────────────────────────────────
app.get('/api/dashboard', (req, res) => {
  const all = [...requests.values()];
  res.json({
    total: all.length,
    by_status: {
      submitted: all.filter(r => r.status === 'submitted').length,
      assigned: all.filter(r => r.status === 'assigned').length,
      scheduled: all.filter(r => r.status === 'scheduled').length,
      in_progress: all.filter(r => r.status === 'in_progress').length,
      completed: all.filter(r => r.status === 'completed').length,
    },
    by_urgency: {
      emergency: all.filter(r => r.urgency === 'emergency').length,
      high: all.filter(r => r.urgency === 'high').length,
      medium: all.filter(r => r.urgency === 'medium').length,
      low: all.filter(r => r.urgency === 'low').length,
    },
    by_team: Object.fromEntries(
      Object.keys(TEAMS).map(t => [t, all.filter(r => r.assigned_team === t).length])
    ),
    avg_resolution_hours: (() => {
      const completed = all.filter(r => r.completed_at);
      if (completed.length === 0) return 0;
      const totalHours = completed.reduce((sum, r) => {
        return sum + (new Date(r.completed_at) - new Date(r.created_at)) / (1000 * 60 * 60);
      }, 0);
      return Math.round(totalHours / completed.length * 10) / 10;
    })(),
  });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...requests.values()];
  res.json({
    status: 'ok',
    tenants: tenants.size,
    total_requests: all.length,
    active_requests: all.filter(r => !['completed', 'cancelled'].includes(r.status)).length,
    emergencies: all.filter(r => r.urgency === 'emergency' && r.status !== 'completed').length,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Tenant maintenance bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "tenant-maintenance",
  "tenant": {
    "name": "María González",
    "unit": "4B",
    "building": "Torre Norte"
  },
  "requests": [
    {
      "id": "MR-M1A2B3",
      "description": "Pérdida de agua en el baño debajo del lavatorio...",
      "status": "assigned",
      "urgency": "high",
      "assigned_team": "plumbing",
      "created_at": "2026-04-02T10:00:00.000Z"
    }
  ],
  "active_requests": 1,
  "total_requests": 3
}
```

Per-message metadata:

```json
{
  "agent": "tenant-maintenance",
  "request_id": "MR-M1A2B3",
  "action": "request_created",
  "category": "plumbing",
  "urgency": "high"
}
```

## How to Run

```bash
# 1. Create the project
mkdir tenant-maintenance && cd tenant-maintenance
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export MANAGER_PHONE="5491155550000"

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

# 6. Pre-register tenants via API
curl -X POST http://localhost:3000/api/tenants \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "name": "María González",
    "unit": "4B",
    "building": "Torre Norte"
  }'

# 7. Update request status (from maintenance team)
curl -X PUT http://localhost:3000/api/requests/MR-xxx/status \
  -H "Content-Type: application/json" \
  -d '{
    "status": "scheduled",
    "scheduled_date": "2026-04-04",
    "scheduled_time": "10:00",
    "note": "Técnico confirmó visita"
  }'

# 8. View dashboard
curl http://localhost:3000/api/dashboard
```
