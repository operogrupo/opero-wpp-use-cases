# 42 - Pharmacy Refill Reminders

Automated pharmacy refill reminder system via WhatsApp. Tracks medications per patient, sends timely reminders, handles confirmations, and alerts pharmacists for fulfillment.

## Problem

A neighborhood pharmacy manages 200+ patients on recurring medications. Staff manually tracks refill dates on spreadsheets, calls patients one by one, and frequently misses reminders. 30% of patients miss their refill windows, leading to medication gaps and lost revenue. Each manual call takes 3-5 minutes, consuming 10+ hours per week.

## Solution

Deploy a WhatsApp refill reminder system that:
- Stores each patient's medication list and refill schedules in conversation metadata
- Automatically sends reminders when refills are due (3 days before, day of)
- Handles "yes refill" or "skip this time" responses
- Alerts the pharmacist when a refill is confirmed so they can prepare it
- Tracks refill history for compliance reporting
- No AI needed -- uses simple keyword matching for responses

## Architecture

```
+-------------------+
|  Cron Scheduler   |  (checks daily for upcoming refills)
+-------------------+
         |
         v
+------------------+     proactive message      +--------------------+
|   Opero WPP API  | <------------------------- |  Your Express App  |
| wpp-api.opero.so |                            |   (this code)      |
+------------------+                            +--------------------+
         |                                               ^
         |  webhook POST (patient replies)               |
         +---------------------------------------------->+
                                                         |
                                              +--------------------+
                                              | Pharmacist Alert   |
                                              | (WhatsApp message) |
                                              +--------------------+
```

## Code

```javascript
// server.js
import express from 'express';

const app = express();
app.use(express.json());

// -- Config --
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const PHARMACIST_PHONE = process.env.PHARMACIST_PHONE; // Pharmacist's WhatsApp number

// -- Opero API helpers --
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

async function markAsRead(phone, messageIds) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/read`, {
    method: 'POST',
    body: JSON.stringify({ phone, message_ids: messageIds }),
  });
}

async function getConversationMetadata(phone) {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`);
    return res.data?.metadata || {};
  } catch {
    return {};
  }
}

async function updateConversationMetadata(phone, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    method: 'PUT',
    body: JSON.stringify({ metadata }),
  });
}

async function listConversations() {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=500`);
    return res.data || [];
  } catch {
    return [];
  }
}

// -- Patient registration (call this to add patients) --
async function registerPatient(phone, patientName, medications) {
  const metadata = await getConversationMetadata(phone);

  metadata.patient_name = patientName;
  metadata.medications = medications.map(med => ({
    name: med.name,
    dosage: med.dosage,
    frequency: med.frequency,
    last_refill: med.last_refill || new Date().toISOString().split('T')[0],
    refill_interval_days: med.refill_interval_days || 30,
    next_refill: med.next_refill || calculateNextRefill(
      med.last_refill || new Date().toISOString().split('T')[0],
      med.refill_interval_days || 30
    ),
    status: 'active',
  }));
  metadata.refill_history = metadata.refill_history || [];
  metadata.registered_at = new Date().toISOString();

  await updateConversationMetadata(phone, metadata);
  return metadata;
}

function calculateNextRefill(lastRefill, intervalDays) {
  const date = new Date(lastRefill);
  date.setDate(date.getDate() + intervalDays);
  return date.toISOString().split('T')[0];
}

function daysUntil(dateStr) {
  const target = new Date(dateStr);
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  target.setHours(0, 0, 0, 0);
  return Math.ceil((target - today) / (1000 * 60 * 60 * 24));
}

// -- Refill check (run daily via cron) --
async function checkRefills() {
  console.log(`[${new Date().toISOString()}] Running daily refill check...`);
  const conversations = await listConversations();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.medications) continue;

    const phone = conv.phone || conv.jid;
    const patientName = metadata.patient_name || 'Patient';

    for (const med of metadata.medications) {
      if (med.status !== 'active') continue;

      const days = daysUntil(med.next_refill);

      // Send reminder at 3 days and on the day
      if (days === 3) {
        await sendText(phone,
          `Hello ${patientName}! Your refill for *${med.name} (${med.dosage})* is due in 3 days (${med.next_refill}).\n\nReply:\n- *YES* to confirm refill\n- *SKIP* to skip this time`,
          {
            agent: 'pharmacy-refills',
            type: 'reminder',
            medication: med.name,
            days_until_refill: days,
          }
        );
        console.log(`Sent 3-day reminder to ${phone} for ${med.name}`);
      } else if (days === 0) {
        await sendText(phone,
          `${patientName}, today is your refill day for *${med.name} (${med.dosage})*.\n\nReply *YES* to confirm or *SKIP* to postpone.`,
          {
            agent: 'pharmacy-refills',
            type: 'reminder-urgent',
            medication: med.name,
            days_until_refill: 0,
          }
        );
        console.log(`Sent day-of reminder to ${phone} for ${med.name}`);
      } else if (days < 0 && days >= -3 && !med.reminder_sent_overdue) {
        await sendText(phone,
          `${patientName}, your refill for *${med.name}* is ${Math.abs(days)} day(s) overdue. Please reply *YES* to refill or contact us if you have questions.`,
          {
            agent: 'pharmacy-refills',
            type: 'reminder-overdue',
            medication: med.name,
            days_overdue: Math.abs(days),
          }
        );
        med.reminder_sent_overdue = true;
        await updateConversationMetadata(phone, metadata);
      }
    }
  }
}

// -- Identify which medication a response is about --
function findPendingMedication(medications) {
  const now = new Date();
  now.setHours(0, 0, 0, 0);

  return medications
    .filter(m => m.status === 'active')
    .sort((a, b) => new Date(a.next_refill) - new Date(b.next_refill))
    .find(m => {
      const days = daysUntil(m.next_refill);
      return days <= 3;
    });
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim().toUpperCase();

  if (!messageText) return;

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    if (!metadata.medications || metadata.medications.length === 0) {
      await sendText(phone, 'Welcome! You are not yet registered in our refill system. Please visit the pharmacy to set up your medication reminders.', {
        agent: 'pharmacy-refills',
        type: 'unregistered',
      });
      return;
    }

    const pendingMed = findPendingMedication(metadata.medications);

    if (messageText === 'YES' || messageText === 'SI' || messageText === 'REFILL') {
      if (!pendingMed) {
        await sendText(phone, 'You have no pending refills at the moment. We\'ll remind you when it\'s time!', {
          agent: 'pharmacy-refills',
          type: 'no-pending',
        });
        return;
      }

      // Update medication refill data
      const oldNextRefill = pendingMed.next_refill;
      pendingMed.last_refill = new Date().toISOString().split('T')[0];
      pendingMed.next_refill = calculateNextRefill(pendingMed.last_refill, pendingMed.refill_interval_days);
      pendingMed.reminder_sent_overdue = false;

      // Record in history
      if (!metadata.refill_history) metadata.refill_history = [];
      metadata.refill_history.push({
        medication: pendingMed.name,
        dosage: pendingMed.dosage,
        requested_at: new Date().toISOString(),
        original_due_date: oldNextRefill,
        action: 'confirmed',
      });

      await updateConversationMetadata(phone, metadata);

      // Confirm to patient
      await sendText(phone,
        `Your refill for *${pendingMed.name} (${pendingMed.dosage})* has been confirmed. The pharmacy will prepare it.\n\nNext refill: ${pendingMed.next_refill}`,
        {
          agent: 'pharmacy-refills',
          type: 'refill-confirmed',
          medication: pendingMed.name,
          next_refill: pendingMed.next_refill,
        }
      );

      // Alert pharmacist
      await sendText(PHARMACIST_PHONE,
        `REFILL REQUEST\nPatient: ${metadata.patient_name}\nPhone: ${phone}\nMedication: ${pendingMed.name} (${pendingMed.dosage})\nFrequency: ${pendingMed.frequency}\nRequested at: ${new Date().toISOString()}`,
        {
          agent: 'pharmacy-refills',
          type: 'pharmacist-alert',
          patient_phone: phone,
          medication: pendingMed.name,
        }
      );

      console.log(`Refill confirmed for ${phone}: ${pendingMed.name}`);
    } else if (messageText === 'SKIP' || messageText === 'NO') {
      if (!pendingMed) {
        await sendText(phone, 'No pending refills to skip. We\'ll remind you when it\'s time!', {
          agent: 'pharmacy-refills',
          type: 'no-pending',
        });
        return;
      }

      // Push the next refill date out by 7 days
      pendingMed.next_refill = calculateNextRefill(pendingMed.next_refill, 7);
      pendingMed.reminder_sent_overdue = false;

      if (!metadata.refill_history) metadata.refill_history = [];
      metadata.refill_history.push({
        medication: pendingMed.name,
        dosage: pendingMed.dosage,
        requested_at: new Date().toISOString(),
        action: 'skipped',
      });

      await updateConversationMetadata(phone, metadata);

      await sendText(phone,
        `Got it, skipping the refill for *${pendingMed.name}* this time. We'll remind you again on ${pendingMed.next_refill}.`,
        {
          agent: 'pharmacy-refills',
          type: 'refill-skipped',
          medication: pendingMed.name,
          next_reminder: pendingMed.next_refill,
        }
      );

      console.log(`Refill skipped for ${phone}: ${pendingMed.name}`);
    } else if (messageText === 'LIST' || messageText === 'MEDS') {
      const medList = metadata.medications
        .filter(m => m.status === 'active')
        .map(m => `- *${m.name}* (${m.dosage}) - ${m.frequency}\n  Next refill: ${m.next_refill}`)
        .join('\n\n');

      await sendText(phone,
        `Your active medications:\n\n${medList}\n\nReply *YES* to confirm a pending refill or *SKIP* to postpone.`,
        {
          agent: 'pharmacy-refills',
          type: 'medication-list',
        }
      );
    } else {
      await sendText(phone,
        `Hi ${metadata.patient_name}! I can help with your medication refills.\n\nCommands:\n- *YES* - Confirm a pending refill\n- *SKIP* - Skip/postpone a refill\n- *LIST* - View your medications\n\nOr visit the pharmacy for any questions.`,
        {
          agent: 'pharmacy-refills',
          type: 'help',
        }
      );
    }
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- Admin endpoint: register a patient --
app.post('/admin/patients', async (req, res) => {
  try {
    const { phone, name, medications } = req.body;
    const metadata = await registerPatient(phone, name, medications);

    await sendText(phone,
      `Hello ${name}! You've been registered in the pharmacy refill reminder system. We'll send you reminders when your medications are due for refill.\n\nReply *LIST* to see your medications.`,
      { agent: 'pharmacy-refills', type: 'registration' }
    );

    res.json({ success: true, metadata });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin endpoint: trigger daily check --
app.post('/admin/check-refills', async (req, res) => {
  try {
    await checkRefills();
    res.json({ success: true, checked_at: new Date().toISOString() });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Daily cron (runs every day at 9 AM) --
function scheduleDailyCheck() {
  const now = new Date();
  const next9am = new Date();
  next9am.setHours(9, 0, 0, 0);
  if (now >= next9am) next9am.setDate(next9am.getDate() + 1);

  const delay = next9am - now;
  setTimeout(() => {
    checkRefills();
    setInterval(checkRefills, 24 * 60 * 60 * 1000);
  }, delay);

  console.log(`Daily refill check scheduled for ${next9am.toISOString()}`);
}

// -- Health check --
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Pharmacy refill bot listening on port ${PORT}`);
  scheduleDailyCheck();
});
```

## Metadata Schema

Each patient conversation stores medication and refill data:

```json
{
  "patient_name": "Maria Garcia",
  "medications": [
    {
      "name": "Metformin",
      "dosage": "500mg",
      "frequency": "twice daily",
      "last_refill": "2026-03-15",
      "refill_interval_days": 30,
      "next_refill": "2026-04-14",
      "status": "active",
      "reminder_sent_overdue": false
    },
    {
      "name": "Lisinopril",
      "dosage": "10mg",
      "frequency": "once daily",
      "last_refill": "2026-03-20",
      "refill_interval_days": 30,
      "next_refill": "2026-04-19",
      "status": "active",
      "reminder_sent_overdue": false
    }
  ],
  "refill_history": [
    {
      "medication": "Metformin",
      "dosage": "500mg",
      "requested_at": "2026-03-15T10:30:00.000Z",
      "original_due_date": "2026-03-15",
      "action": "confirmed"
    }
  ],
  "registered_at": "2026-01-10T09:00:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "pharmacy-refills",
  "type": "refill-confirmed",
  "medication": "Metformin",
  "next_refill": "2026-04-14"
}
```

## How to Run

```bash
# 1. Create the project
mkdir pharmacy-refills && cd pharmacy-refills
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export PHARMACIST_PHONE="5491155551234"

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

# 6. Register a patient
curl -X POST http://localhost:3000/admin/patients \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "name": "Maria Garcia",
    "medications": [
      {
        "name": "Metformin",
        "dosage": "500mg",
        "frequency": "twice daily",
        "refill_interval_days": 30
      }
    ]
  }'

# 7. Manually trigger a refill check (or wait for 9 AM)
curl -X POST http://localhost:3000/admin/check-refills
```
