# 32 - Freelancer Client Updates

Freelancer project update bot. Sends weekly progress updates, handles feedback, shares deliverables, and tracks project metadata including milestones, hours, and budget.

## Problem

A freelance developer manages 5-8 concurrent clients. Each expects regular updates, but composing individual emails takes 30+ minutes per client per week. Clients reply at random times asking "how's the project going?" and the freelancer loses productive hours context-switching. Milestone tracking lives in spreadsheets that clients never check, and budget discussions happen in scattered chat threads.

## Solution

Deploy a WhatsApp project update bot that:
- Sends automated weekly progress updates to each client
- Tracks milestones, hours logged, and budget usage per project
- Handles client feedback requests and deliverable sharing
- Lets clients query project status anytime with a keyword
- Stores full project history in conversation metadata

## Architecture

```
+-------------------+     cron (weekly)       +--------------------+
|  Update Scheduler | ----------------------> |  Your Express App  |
|  + API endpoints  |                         |   (this code)      |
+-------------------+                         +--------------------+
                                                       |
         +---------------------------------------------+
         |                                             |
         v                                             v
+------------------+                          +------------------+
|   Opero WPP API  |                          | Client responds  |
|  wpp-api.opero.so|  <---  webhook  -------- | with feedback    |
+------------------+                          +------------------+
         |
         v
  PUT /conversations/:phone
  (project metadata: milestones, hours, budget)
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
const projects = new Map();
const clientPhoneIndex = new Map(); // phone -> project_id

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

// ── Project management ────────────────────────────────────────────────
function createProject(data) {
  const id = `PROJ-${Date.now().toString(36).toUpperCase()}`;
  const project = {
    id,
    client_name: data.client_name,
    client_phone: data.client_phone,
    project_name: data.project_name,
    description: data.description || '',
    budget_total: data.budget_total || 0,
    budget_spent: 0,
    hourly_rate: data.hourly_rate || 0,
    hours_logged: 0,
    start_date: data.start_date || new Date().toISOString().split('T')[0],
    estimated_end: data.estimated_end || null,
    status: 'active', // active, paused, completed, cancelled
    milestones: (data.milestones || []).map((m, i) => ({
      id: `MS-${i + 1}`,
      name: m.name,
      description: m.description || '',
      due_date: m.due_date || null,
      status: 'pending', // pending, in_progress, completed, blocked
      completed_at: null,
    })),
    updates: [],
    feedback: [],
    deliverables: [],
    created_at: new Date().toISOString(),
  };

  projects.set(id, project);
  clientPhoneIndex.set(data.client_phone, id);
  return project;
}

function logHours(projectId, hours, description) {
  const project = projects.get(projectId);
  if (!project) throw new Error('Project not found');

  project.hours_logged += hours;
  if (project.hourly_rate) {
    project.budget_spent = project.hours_logged * project.hourly_rate;
  }

  const entry = {
    hours,
    description,
    logged_at: new Date().toISOString(),
    total_hours: project.hours_logged,
    total_spent: project.budget_spent,
  };

  return entry;
}

function updateMilestone(projectId, milestoneId, status) {
  const project = projects.get(projectId);
  if (!project) throw new Error('Project not found');

  const milestone = project.milestones.find(m => m.id === milestoneId);
  if (!milestone) throw new Error('Milestone not found');

  milestone.status = status;
  if (status === 'completed') {
    milestone.completed_at = new Date().toISOString();
  }

  return milestone;
}

// ── Generate weekly update ────────────────────────────────────────────
function generateUpdateMessage(project) {
  const completedMilestones = project.milestones.filter(m => m.status === 'completed').length;
  const totalMilestones = project.milestones.length;
  const progressPct = totalMilestones > 0
    ? Math.round((completedMilestones / totalMilestones) * 100)
    : 0;

  const budgetPct = project.budget_total > 0
    ? Math.round((project.budget_spent / project.budget_total) * 100)
    : 0;

  const currentMilestone = project.milestones.find(m => m.status === 'in_progress')
    || project.milestones.find(m => m.status === 'pending');

  const milestoneStatus = project.milestones.map(m => {
    const icon = m.status === 'completed' ? '✅'
      : m.status === 'in_progress' ? '🔄'
      : m.status === 'blocked' ? '🚫'
      : '⏳';
    return `${icon} ${m.name}${m.due_date ? ` (${m.due_date})` : ''}`;
  }).join('\n');

  return (
    `📊 *Actualización semanal — ${project.project_name}*\n\n` +
    `Progreso general: *${progressPct}%* (${completedMilestones}/${totalMilestones} hitos)\n` +
    `Horas trabajadas: *${project.hours_logged}h*\n` +
    `${project.budget_total > 0 ? `Presupuesto: $${project.budget_spent.toLocaleString()} / $${project.budget_total.toLocaleString()} (${budgetPct}%)\n` : ''}` +
    `\n*Hitos:*\n${milestoneStatus}\n` +
    `${currentMilestone ? `\n🎯 Trabajando en: *${currentMilestone.name}*` : ''}` +
    `\n\nResponda:\n` +
    `*DETALLE* - Ver más información\n` +
    `*FEEDBACK* - Enviar comentarios\n` +
    `*PAUSA* - Solicitar pausa del proyecto`
  );
}

// ── Send weekly updates ───────────────────────────────────────────────
async function sendWeeklyUpdates() {
  for (const [id, project] of projects) {
    if (project.status !== 'active') continue;

    const message = generateUpdateMessage(project);
    const update = {
      type: 'weekly',
      message,
      sent_at: new Date().toISOString(),
    };

    project.updates.push(update);

    try {
      await sendText(project.client_phone, message, {
        agent: 'freelancer-updates',
        project_id: id,
        update_type: 'weekly',
        progress_pct: Math.round(
          (project.milestones.filter(m => m.status === 'completed').length / Math.max(project.milestones.length, 1)) * 100
        ),
        hours_logged: project.hours_logged,
        budget_spent: project.budget_spent,
      });

      await syncMetadata(project);
      console.log(`[${new Date().toISOString()}] Weekly update sent for ${project.project_name}`);
    } catch (err) {
      console.error(`Failed to send update for ${id}:`, err.message);
    }
  }
}

// Run weekly (every Monday at 9:00 — in production, use a proper scheduler)
setInterval(sendWeeklyUpdates, 7 * 24 * 60 * 60 * 1000);

// ── Sync metadata ─────────────────────────────────────────────────────
async function syncMetadata(project) {
  await updateConversation(project.client_phone, {
    agent: 'freelancer-updates',
    project: {
      id: project.id,
      name: project.project_name,
      status: project.status,
      progress_pct: Math.round(
        (project.milestones.filter(m => m.status === 'completed').length / Math.max(project.milestones.length, 1)) * 100
      ),
      hours_logged: project.hours_logged,
      budget_total: project.budget_total,
      budget_spent: project.budget_spent,
      milestones: project.milestones,
      deliverables_count: project.deliverables.length,
      last_update: project.updates[project.updates.length - 1]?.sent_at,
    },
  });
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

  const projectId = clientPhoneIndex.get(phone);
  if (!projectId) return;

  const project = projects.get(projectId);
  if (!project) return;

  console.log(`[${new Date().toISOString()}] Client message (${project.client_name}): ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const upper = messageText.toUpperCase();

    switch (upper) {
      case 'STATUS':
      case 'ESTADO': {
        const message = generateUpdateMessage(project);
        await sendText(phone, message, {
          agent: 'freelancer-updates',
          project_id: project.id,
          action: 'status_requested',
        });
        break;
      }

      case 'DETALLE': {
        const details = project.milestones.map(m => {
          const statusLabel = {
            pending: 'Pendiente',
            in_progress: 'En progreso',
            completed: 'Completado',
            blocked: 'Bloqueado',
          }[m.status];
          return `*${m.name}*\n  Estado: ${statusLabel}\n  ${m.description || ''}${m.due_date ? `\n  Fecha: ${m.due_date}` : ''}`;
        }).join('\n\n');

        await sendText(phone, `📋 *Detalle de hitos — ${project.project_name}*\n\n${details}`, {
          agent: 'freelancer-updates',
          project_id: project.id,
          action: 'detail_requested',
        });
        break;
      }

      case 'FEEDBACK': {
        await sendText(phone, 'Escriba su feedback o comentarios a continuación. Los recibiré de inmediato.', {
          agent: 'freelancer-updates',
          project_id: project.id,
          action: 'feedback_prompt',
        });
        // Next message will be treated as feedback
        project._awaiting_feedback = true;
        break;
      }

      case 'PAUSA': {
        project.status = 'paused';
        await sendText(phone, `⏸️ Proyecto *${project.project_name}* pausado. Para reanudar, escriba *REANUDAR*.`, {
          agent: 'freelancer-updates',
          project_id: project.id,
          action: 'project_paused',
        });
        await syncMetadata(project);
        break;
      }

      case 'REANUDAR': {
        project.status = 'active';
        await sendText(phone, `▶️ Proyecto *${project.project_name}* reanudado. Le enviaré la próxima actualización el lunes.`, {
          agent: 'freelancer-updates',
          project_id: project.id,
          action: 'project_resumed',
        });
        await syncMetadata(project);
        break;
      }

      case 'PRESUPUESTO': {
        const budgetPct = project.budget_total > 0
          ? Math.round((project.budget_spent / project.budget_total) * 100)
          : 0;
        await sendText(phone,
          `💰 *Presupuesto — ${project.project_name}*\n\n` +
          `Total: $${project.budget_total.toLocaleString()}\n` +
          `Utilizado: $${project.budget_spent.toLocaleString()} (${budgetPct}%)\n` +
          `Restante: $${(project.budget_total - project.budget_spent).toLocaleString()}\n` +
          `Horas: ${project.hours_logged}h${project.hourly_rate ? ` @ $${project.hourly_rate}/h` : ''}`,
          {
            agent: 'freelancer-updates',
            project_id: project.id,
            action: 'budget_requested',
          }
        );
        break;
      }

      case 'ENTREGAS': {
        if (project.deliverables.length === 0) {
          await sendText(phone, 'No hay entregas registradas aún.', {
            agent: 'freelancer-updates', action: 'no_deliverables',
          });
        } else {
          const list = project.deliverables.map(d =>
            `- *${d.name}*: ${d.url || d.description} (${d.delivered_at})`
          ).join('\n');
          await sendText(phone, `📦 *Entregas — ${project.project_name}*\n\n${list}`, {
            agent: 'freelancer-updates', action: 'deliverables_listed',
          });
        }
        break;
      }

      default: {
        // Check if awaiting feedback
        if (project._awaiting_feedback) {
          delete project._awaiting_feedback;
          project.feedback.push({
            message: messageText,
            received_at: new Date().toISOString(),
          });

          await sendText(phone, '✅ Feedback recibido. Gracias por sus comentarios, los tendré en cuenta.', {
            agent: 'freelancer-updates',
            project_id: project.id,
            action: 'feedback_received',
          });

          // Notify freelancer
          const FREELANCER_PHONE = process.env.FREELANCER_PHONE;
          if (FREELANCER_PHONE) {
            await sendText(FREELANCER_PHONE,
              `📩 Feedback de ${project.client_name} (${project.project_name}):\n\n${messageText}`,
              { agent: 'freelancer-updates', action: 'feedback_notification' }
            );
          }
          await syncMetadata(project);
        } else {
          await sendText(phone,
            `Comandos disponibles:\n\n` +
            `*STATUS* - Ver estado del proyecto\n` +
            `*DETALLE* - Ver hitos en detalle\n` +
            `*PRESUPUESTO* - Ver presupuesto\n` +
            `*ENTREGAS* - Ver entregas\n` +
            `*FEEDBACK* - Enviar comentarios\n` +
            `*PAUSA* / *REANUDAR* - Pausar/reanudar proyecto`,
            { agent: 'freelancer-updates', action: 'help_shown' }
          );
        }
        break;
      }
    }
  } catch (err) {
    console.error(`Error handling client message from ${phone}:`, err.message);
  }
});

// ── API: Create project ───────────────────────────────────────────────
app.post('/api/projects', async (req, res) => {
  const project = createProject(req.body);
  try {
    await syncMetadata(project);
    await sendText(project.client_phone,
      `🚀 *Proyecto iniciado: ${project.project_name}*\n\n` +
      `Hola ${project.client_name}, le confirmo que el proyecto está en marcha.\n\n` +
      `Recibirá actualizaciones semanales por este medio. También puede escribir:\n` +
      `*STATUS* — estado actual\n` +
      `*PRESUPUESTO* — uso del presupuesto\n` +
      `*FEEDBACK* — enviar comentarios`,
      { agent: 'freelancer-updates', action: 'project_started', project_id: project.id }
    );
  } catch (err) {
    console.error('Error notifying client:', err.message);
  }
  res.json(project);
});

// ── API: Log hours ────────────────────────────────────────────────────
app.post('/api/projects/:id/hours', async (req, res) => {
  try {
    const entry = logHours(req.params.id, req.body.hours, req.body.description);
    const project = projects.get(req.params.id);
    await syncMetadata(project);
    res.json(entry);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Update milestone ─────────────────────────────────────────────
app.put('/api/projects/:id/milestones/:msId', async (req, res) => {
  try {
    const milestone = updateMilestone(req.params.id, req.params.msId, req.body.status);
    const project = projects.get(req.params.id);

    if (req.body.status === 'completed') {
      await sendText(project.client_phone,
        `✅ *Hito completado: ${milestone.name}*\n\n` +
        `Proyecto: ${project.project_name}\n` +
        `${milestone.description || ''}\n\n` +
        `Escriba *STATUS* para ver el progreso general.`,
        {
          agent: 'freelancer-updates',
          project_id: project.id,
          action: 'milestone_completed',
          milestone_id: milestone.id,
        }
      );
    }

    await syncMetadata(project);
    res.json(milestone);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Add deliverable ──────────────────────────────────────────────
app.post('/api/projects/:id/deliverables', async (req, res) => {
  const project = projects.get(req.params.id);
  if (!project) return res.status(404).json({ error: 'Project not found' });

  const deliverable = {
    name: req.body.name,
    description: req.body.description || '',
    url: req.body.url || '',
    delivered_at: new Date().toISOString(),
  };

  project.deliverables.push(deliverable);

  try {
    await sendText(project.client_phone,
      `📦 *Nueva entrega: ${deliverable.name}*\n\n` +
      `${deliverable.description}\n` +
      `${deliverable.url ? `\n🔗 ${deliverable.url}` : ''}\n\n` +
      `Por favor revise y envíe su *FEEDBACK* cuando pueda.`,
      {
        agent: 'freelancer-updates',
        project_id: project.id,
        action: 'deliverable_shared',
        deliverable_name: deliverable.name,
      }
    );
    await syncMetadata(project);
  } catch (err) {
    console.error('Error sharing deliverable:', err.message);
  }

  res.json(deliverable);
});

// ── API: Trigger weekly updates ───────────────────────────────────────
app.post('/api/updates/send', async (req, res) => {
  await sendWeeklyUpdates();
  res.json({ status: 'sent' });
});

// ── API: List projects ────────────────────────────────────────────────
app.get('/api/projects', (req, res) => {
  res.json({ projects: [...projects.values()], total: projects.size });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    active_projects: [...projects.values()].filter(p => p.status === 'active').length,
    total_projects: projects.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Freelancer client updates bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "freelancer-updates",
  "project": {
    "id": "PROJ-M1A2B3",
    "name": "Rediseño sitio web",
    "status": "active",
    "progress_pct": 60,
    "hours_logged": 42,
    "budget_total": 150000,
    "budget_spent": 84000,
    "milestones": [
      { "id": "MS-1", "name": "Diseño UI", "status": "completed", "completed_at": "2026-03-15T10:00:00.000Z" },
      { "id": "MS-2", "name": "Desarrollo frontend", "status": "in_progress", "due_date": "2026-04-15" },
      { "id": "MS-3", "name": "Testing y deploy", "status": "pending", "due_date": "2026-04-30" }
    ],
    "deliverables_count": 2,
    "last_update": "2026-04-01T09:00:00.000Z"
  }
}
```

Per-message metadata:

```json
{
  "agent": "freelancer-updates",
  "project_id": "PROJ-M1A2B3",
  "update_type": "weekly",
  "progress_pct": 60,
  "hours_logged": 42,
  "budget_spent": 84000
}
```

## How to Run

```bash
# 1. Create the project
mkdir freelancer-updates && cd freelancer-updates
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export FREELANCER_PHONE="your-personal-phone"

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

# 6. Create a project
curl -X POST http://localhost:3000/api/projects \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "María González",
    "client_phone": "5491155551001",
    "project_name": "Rediseño sitio web",
    "budget_total": 150000,
    "hourly_rate": 2000,
    "estimated_end": "2026-04-30",
    "milestones": [
      { "name": "Diseño UI", "due_date": "2026-03-15" },
      { "name": "Desarrollo frontend", "due_date": "2026-04-15" },
      { "name": "Testing y deploy", "due_date": "2026-04-30" }
    ]
  }'

# 7. Log hours
curl -X POST http://localhost:3000/api/projects/PROJ-xxx/hours \
  -H "Content-Type: application/json" \
  -d '{ "hours": 8, "description": "Implementación de componentes React" }'

# 8. Send weekly updates manually
curl -X POST http://localhost:3000/api/updates/send
```
