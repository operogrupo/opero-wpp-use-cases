# 28 - School Notifications

Parent notification system for schools. Broadcast homework, events, closures, and grade updates. Two-way communication lets parents ask questions.

## Problem

A school with 500 students sends daily announcements via printed notes and a rarely-checked email system. Parents miss field trip deadlines, don't know about closures until they arrive, and teachers spend 3+ hours per week fielding repetitive questions. The school needs a direct communication channel that parents actually check.

## Solution

Deploy a WhatsApp notification system that:
- Broadcasts homework assignments, events, closures, and grade updates to parents
- Supports group broadcasts by grade level, section, or entire school
- Enables two-way communication so parents can reply with questions
- Routes parent questions to the appropriate teacher or administrator
- Tracks message delivery and read receipts in metadata

## Architecture

```
+--------------------+     POST /api/broadcast    +--------------------+
|  School Admin      | --------------------------> |  Your Express App  |
|  Dashboard / API   |                             |   (this code)      |
+--------------------+                             +--------------------+
                                                            |
         +--------------------------------------------------+
         |                                                  |
         v                                                  v
+------------------+                               +------------------+
|   Opero WPP API  |  --- sends to each parent --> | Parent receives  |
|  wpp-api.opero.so|                               | WhatsApp message |
+------------------+                               +------------------+
         ^                                                  |
         |              webhook (parent replies)            |
         +--------------------------------------------------+
         |
         v
  PUT /conversations/:phone (track Q&A)
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

// ── Student/parent database (use a real DB in production) ─────────────
const students = new Map();
const parentPhones = new Map(); // phone -> { student_ids, name }
const broadcastHistory = [];

// Seed example data
function seedData() {
  students.set('S001', {
    id: 'S001',
    name: 'Lucía Fernández',
    grade: '5A',
    parent_phone: '5491155551001',
    parent_name: 'Carolina Fernández',
  });
  students.set('S002', {
    id: 'S002',
    name: 'Mateo García',
    grade: '5A',
    parent_phone: '5491155551002',
    parent_name: 'Roberto García',
  });
  students.set('S003', {
    id: 'S003',
    name: 'Valentina López',
    grade: '3B',
    parent_phone: '5491155551003',
    parent_name: 'Silvia López',
  });

  for (const [id, student] of students) {
    const existing = parentPhones.get(student.parent_phone) || {
      name: student.parent_name,
      student_ids: [],
    };
    existing.student_ids.push(id);
    parentPhones.set(student.parent_phone, existing);
  }
}
seedData();

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

// ── Notification types ────────────────────────────────────────────────
const NOTIFICATION_FORMATTERS = {
  homework: (data) =>
    `📚 *Tarea - ${data.subject}*\nGrado: ${data.grade}\n\n${data.description}\n\nFecha de entrega: *${data.due_date}*\n${data.notes ? `\nNota: ${data.notes}` : ''}`,

  event: (data) =>
    `📅 *Evento: ${data.title}*\n\nFecha: *${data.date}*\nHora: ${data.time}\nLugar: ${data.location}\n\n${data.description}\n${data.requires_permission ? '\n⚠️ Se requiere autorización firmada. Responda "AUTORIZO" para confirmar.' : ''}`,

  closure: (data) =>
    `🏫 *Aviso: Sin clases*\n\nFecha: *${data.date}*\nMotivo: ${data.reason}\n\n${data.details || 'Las clases se reanudan normalmente al día siguiente.'}`,

  grade_update: (data) =>
    `📝 *Calificaciones - ${data.subject}*\n\nAlumno/a: ${data.student_name}\nEvaluación: ${data.evaluation}\nNota: *${data.grade}*\n${data.comments ? `\nComentarios: ${data.comments}` : ''}\n\nPara consultas, responda "CONSULTA ${data.subject}".`,

  general: (data) =>
    `📢 *${data.title}*\n\n${data.message}`,
};

// ── Broadcast engine ──────────────────────────────────────────────────
async function broadcast(type, data, targetGrades = null) {
  const formatter = NOTIFICATION_FORMATTERS[type];
  if (!formatter) throw new Error(`Unknown notification type: ${type}`);

  const message = formatter(data);
  const targets = [];

  for (const [id, student] of students) {
    if (targetGrades && !targetGrades.includes(student.grade)) continue;
    targets.push({
      phone: student.parent_phone,
      student_id: id,
      student_name: student.name,
      parent_name: student.parent_name,
      grade: student.grade,
    });
  }

  // Deduplicate by phone (parents with multiple children)
  const uniquePhones = new Map();
  for (const target of targets) {
    if (!uniquePhones.has(target.phone)) {
      uniquePhones.set(target.phone, target);
    }
  }

  const broadcastId = `broadcast-${Date.now()}`;
  const results = { sent: 0, failed: 0, errors: [] };

  for (const [phone, target] of uniquePhones) {
    try {
      // Personalize for grade updates
      let personalMessage = message;
      if (type === 'grade_update') {
        personalMessage = formatter({ ...data, student_name: target.student_name });
      }

      await sendText(phone, personalMessage, {
        agent: 'school-notifications',
        broadcast_id: broadcastId,
        notification_type: type,
        grade: target.grade,
        student_id: target.student_id,
        sent_at: new Date().toISOString(),
      });

      results.sent++;

      // Small delay to avoid rate limiting
      await new Promise(resolve => setTimeout(resolve, 200));
    } catch (err) {
      results.failed++;
      results.errors.push({ phone, error: err.message });
    }
  }

  const record = {
    id: broadcastId,
    type,
    data,
    target_grades: targetGrades,
    results,
    created_at: new Date().toISOString(),
  };
  broadcastHistory.push(record);

  console.log(`[${new Date().toISOString()}] Broadcast ${broadcastId}: ${results.sent} sent, ${results.failed} failed`);
  return record;
}

// ── Webhook handler (parent replies) ──────────────────────────────────
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

  const parent = parentPhones.get(phone);
  const parentName = parent?.name || 'Padre/Madre';

  console.log(`[${new Date().toISOString()}] Reply from ${parentName} (${phone}): ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const upperMessage = messageText.toUpperCase();

    if (upperMessage === 'AUTORIZO') {
      await sendText(phone, `✅ Autorización registrada para ${parentName}. Gracias.`, {
        agent: 'school-notifications',
        action: 'permission_granted',
        parent_name: parentName,
      });

      await updateConversation(phone, {
        agent: 'school-notifications',
        last_action: 'permission_granted',
        authorized_at: new Date().toISOString(),
      });
    } else if (upperMessage.startsWith('CONSULTA ')) {
      const subject = messageText.substring(9).trim();
      await sendText(phone, `Recibimos su consulta sobre *${subject}*. El/la docente correspondiente se comunicará con usted dentro de las 24 horas.`, {
        agent: 'school-notifications',
        action: 'teacher_query',
        subject,
        parent_name: parentName,
      });

      // Notify admin/teacher
      const ADMIN_PHONE = process.env.ADMIN_PHONE || '5491155550000';
      await sendText(ADMIN_PHONE, `📩 Consulta de ${parentName} (${phone}) sobre ${subject}. Por favor responda.`, {
        agent: 'school-notifications',
        action: 'admin_notification',
        parent_phone: phone,
        subject,
      });
    } else if (upperMessage === 'MENU' || upperMessage === 'AYUDA') {
      await sendText(phone, `*Opciones disponibles:*\n\n- Responda *AUTORIZO* para dar permiso a una actividad\n- Responda *CONSULTA [materia]* para contactar a un docente\n- Responda *HORARIOS* para ver el horario de clases\n- Responda *CONTACTO* para datos de la escuela`, {
        agent: 'school-notifications',
        action: 'menu_shown',
      });
    } else if (upperMessage === 'HORARIOS') {
      const studentIds = parent?.student_ids || [];
      const studentNames = studentIds.map(id => students.get(id)?.name).filter(Boolean).join(', ');
      await sendText(phone, `Horarios para: ${studentNames || 'sus hijos/as'}\n\nLunes a Viernes: 7:45 - 12:30\nTurno tarde (si aplica): 13:30 - 17:30\n\nPara horarios específicos por materia, consulte el cuaderno de comunicaciones.`, {
        agent: 'school-notifications',
        action: 'schedule_sent',
      });
    } else if (upperMessage === 'CONTACTO') {
      await sendText(phone, `*Escuela San Martín*\n\nDirección: Av. Rivadavia 1234, CABA\nTeléfono: +54 11 4444-5678\nEmail: info@escuelasanmartin.edu.ar\nHorario administrativo: 8:00 - 16:00\n\nDirectora: Prof. Elena Martínez\nSecretaría: Sra. Laura Gómez`, {
        agent: 'school-notifications',
        action: 'contact_sent',
      });
    } else {
      // Free-text question
      await updateConversation(phone, {
        agent: 'school-notifications',
        last_question: messageText,
        asked_at: new Date().toISOString(),
        parent_name: parentName,
      });

      await sendText(phone, `Gracias por su mensaje, ${parentName}. Lo derivaremos a quien corresponda. Recibirá respuesta dentro de las 24 horas.\n\nEscriba *MENU* para ver las opciones disponibles.`, {
        agent: 'school-notifications',
        action: 'question_forwarded',
      });
    }
  } catch (err) {
    console.error(`Error handling parent reply from ${phone}:`, err.message);
  }
});

// ── API: Send broadcast ──────────────────────────────────────────────
app.post('/api/broadcast', async (req, res) => {
  const { type, data, target_grades } = req.body;

  if (!type || !data) {
    return res.status(400).json({ error: 'type and data are required' });
  }

  try {
    const result = await broadcast(type, data, target_grades);
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── API: Send to individual parent ───────────────────────────────────
app.post('/api/notify/:studentId', async (req, res) => {
  const student = students.get(req.params.studentId);
  if (!student) {
    return res.status(404).json({ error: 'Student not found' });
  }

  const { type, data } = req.body;
  const formatter = NOTIFICATION_FORMATTERS[type];
  if (!formatter) {
    return res.status(400).json({ error: `Unknown type: ${type}` });
  }

  try {
    const message = formatter({ ...data, student_name: student.name });
    await sendText(student.parent_phone, message, {
      agent: 'school-notifications',
      notification_type: type,
      student_id: student.id,
      student_name: student.name,
    });
    res.json({ status: 'sent', phone: student.parent_phone });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── API: Broadcast history ───────────────────────────────────────────
app.get('/api/broadcasts', (req, res) => {
  res.json({ broadcasts: broadcastHistory, total: broadcastHistory.length });
});

// ── API: Register student ────────────────────────────────────────────
app.post('/api/students', (req, res) => {
  const { id, name, grade, parent_phone, parent_name } = req.body;
  if (!id || !name || !grade || !parent_phone) {
    return res.status(400).json({ error: 'id, name, grade, and parent_phone are required' });
  }

  students.set(id, { id, name, grade, parent_phone, parent_name: parent_name || '' });

  const existing = parentPhones.get(parent_phone) || { name: parent_name || '', student_ids: [] };
  if (!existing.student_ids.includes(id)) {
    existing.student_ids.push(id);
  }
  parentPhones.set(parent_phone, existing);

  res.json({ status: 'registered', student: students.get(id) });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    students_registered: students.size,
    parents_registered: parentPhones.size,
    broadcasts_sent: broadcastHistory.length,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`School notifications bot listening on port ${PORT}`);
});
```

## Metadata Schema

Per-message metadata:

```json
{
  "agent": "school-notifications",
  "broadcast_id": "broadcast-1711958400000",
  "notification_type": "homework",
  "grade": "5A",
  "student_id": "S001",
  "sent_at": "2026-04-02T14:00:00.000Z"
}
```

Conversation metadata (updated on parent interaction):

```json
{
  "agent": "school-notifications",
  "last_action": "permission_granted",
  "authorized_at": "2026-04-02T15:00:00.000Z",
  "last_question": "A qué hora es la reunión de padres?",
  "asked_at": "2026-04-02T16:00:00.000Z",
  "parent_name": "Carolina Fernández"
}
```

## How to Run

```bash
# 1. Create the project
mkdir school-notifications && cd school-notifications
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

# 6. Send a homework broadcast
curl -X POST http://localhost:3000/api/broadcast \
  -H "Content-Type: application/json" \
  -d '{
    "type": "homework",
    "data": {
      "subject": "Matemática",
      "grade": "5A",
      "description": "Resolver ejercicios 1 a 15 de la página 42",
      "due_date": "2026-04-05"
    },
    "target_grades": ["5A"]
  }'

# 7. Send a school closure notice to all parents
curl -X POST http://localhost:3000/api/broadcast \
  -H "Content-Type: application/json" \
  -d '{
    "type": "closure",
    "data": {
      "date": "2026-04-10",
      "reason": "Jornada docente"
    }
  }'
```
