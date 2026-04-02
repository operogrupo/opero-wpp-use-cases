# 47 - Patient Follow-Up Agent

Post-visit patient follow-up system via WhatsApp. After medical appointments, the bot checks in on symptoms, medication side effects, and recovery progress. AI flags concerning responses for doctor review. Tracks follow-up history in metadata.

## Problem

A medical clinic sees 60 patients daily. Post-visit follow-up is critical -- patients forget instructions, experience side effects, or develop complications. The nursing staff can only call 10-15 patients per day for follow-ups, and most calls go unanswered. Patients who need urgent attention slip through the cracks. 40% of readmissions could be prevented with timely follow-up.

## Solution

Deploy a patient follow-up bot that:
- Sends automated check-in messages 24h, 72h, and 7 days after appointments
- Asks about symptoms, medication side effects, and recovery status
- Uses Claude to analyze patient responses for red flags
- Flags concerning responses and escalates to the doctor immediately
- Tracks all follow-up interactions in conversation metadata
- Supports customizable follow-up protocols per visit type

## Architecture

```
+-------------------+
|  Cron Scheduler   |  (checks for scheduled follow-ups)
+-------------------+
         |
         v
+------------------+     follow-up message     +--------------------+
|   Opero WPP API  | <------------------------ |  Your Express App  |
| wpp-api.opero.so |                           |   (this code)      |
+------------------+                           +--------------------+
         |                                              ^
         |    webhook POST (patient response)           |
         +--------------------------------------------->+
                                                        |
                                                        v
                                               +------------------+
                                               |  Claude Sonnet   |
                                               | (response triage)|
                                               +------------------+
                                                        |
                                           if urgent    v
                                               +------------------+
                                               | Doctor Alert     |
                                               | (WhatsApp)       |
                                               +------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

// -- Config --
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const DOCTOR_PHONE = process.env.DOCTOR_PHONE;

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

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

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
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

// -- Follow-up protocols --
const PROTOCOLS = {
  general: {
    follow_ups: [
      { delay_hours: 24, questions: ['How are you feeling today?', 'Are you taking your medications as prescribed?', 'Any new symptoms or concerns?'] },
      { delay_hours: 72, questions: ['How is your recovery going?', 'Any side effects from your medications?', 'Have you been able to follow the care instructions?'] },
      { delay_hours: 168, questions: ['One-week check-in: How are you doing overall?', 'Any lingering symptoms?', 'Do you feel you need a follow-up appointment?'] },
    ],
  },
  surgery: {
    follow_ups: [
      { delay_hours: 12, questions: ['How is your pain level (1-10)?', 'Any bleeding or unusual discharge from the surgical site?', 'Are you able to eat and drink?'] },
      { delay_hours: 24, questions: ['Pain level today (1-10)?', 'How does the surgical site look? Any redness, swelling, or warmth?', 'Were you able to take your medications?'] },
      { delay_hours: 48, questions: ['How is the pain compared to yesterday?', 'Any fever or chills?', 'Are you able to move around as instructed?'] },
      { delay_hours: 72, questions: ['Recovery update: pain level (1-10)?', 'Any signs of infection at the site?', 'How is your appetite and energy?'] },
      { delay_hours: 168, questions: ['One-week post-surgery check: How are you feeling?', 'Is the surgical site healing well?', 'Any concerns before your follow-up appointment?'] },
    ],
  },
  medication_change: {
    follow_ups: [
      { delay_hours: 48, questions: ['How are you adjusting to the new medication?', 'Any side effects you\'ve noticed?', 'Are you taking it at the prescribed times?'] },
      { delay_hours: 168, questions: ['One-week medication check: Any side effects?', 'Do you feel the medication is working?', 'Any concerns about continuing?'] },
    ],
  },
};

// -- AI triage --
const TRIAGE_PROMPT = `You are a medical triage assistant reviewing patient follow-up responses.

Analyze the patient's response and classify it as one of:
- GREEN: Normal recovery, no concerns
- YELLOW: Minor concerns that should be noted but don't require immediate action
- RED: Concerning symptoms that require doctor notification (severe pain, infection signs, breathing difficulty, chest pain, high fever, excessive bleeding, allergic reactions, confusion, inability to keep food/water down)

Visit details will be provided for context.

Respond with EXACTLY this JSON format:
{
  "classification": "GREEN|YELLOW|RED",
  "reasoning": "Brief explanation",
  "follow_up_suggestion": "What to tell the patient",
  "doctor_summary": "Summary for the doctor if YELLOW or RED, null otherwise"
}

IMPORTANT: Err on the side of caution. When in doubt, classify as YELLOW or RED. Patient safety is the top priority.`;

async function triageResponse(patientMessage, visitContext) {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 500,
    system: TRIAGE_PROMPT,
    messages: [{
      role: 'user',
      content: `Visit type: ${visitContext.visit_type}\nDiagnosis: ${visitContext.diagnosis}\nMedications: ${visitContext.medications?.join(', ') || 'None'}\nDoctor notes: ${visitContext.doctor_notes || 'None'}\n\nPatient response:\n"${patientMessage}"`,
    }],
  });

  try {
    const text = response.content[0].text;
    // Extract JSON from response
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    return jsonMatch ? JSON.parse(jsonMatch[0]) : { classification: 'YELLOW', reasoning: 'Could not parse response', follow_up_suggestion: 'A nurse will follow up with you shortly.', doctor_summary: text };
  } catch {
    return {
      classification: 'YELLOW',
      reasoning: 'Failed to parse AI response',
      follow_up_suggestion: 'Thank you for your response. A nurse will follow up with you shortly.',
      doctor_summary: `Unparseable triage for patient message: "${patientMessage}"`,
    };
  }
}

// -- Schedule follow-ups --
function scheduleFollowUps(visit) {
  const protocol = PROTOCOLS[visit.visit_type] || PROTOCOLS.general;
  const visitTime = new Date(visit.visit_date);

  return protocol.follow_ups.map((fu, index) => ({
    step: index + 1,
    total_steps: protocol.follow_ups.length,
    scheduled_at: new Date(visitTime.getTime() + fu.delay_hours * 60 * 60 * 1000).toISOString(),
    questions: fu.questions,
    status: 'pending',
    delay_hours: fu.delay_hours,
  }));
}

// -- Cron: send scheduled follow-ups --
async function processFollowUps() {
  console.log(`[${new Date().toISOString()}] Processing follow-ups...`);
  const conversations = await listConversations();
  const now = new Date();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.follow_ups || metadata.role !== 'patient') continue;

    const phone = conv.phone || conv.jid;

    for (const fu of metadata.follow_ups) {
      if (fu.status !== 'pending') continue;

      const scheduledTime = new Date(fu.scheduled_at);
      if (scheduledTime > now) continue;

      // Send follow-up questions
      const message = [
        `Hi ${metadata.patient_name}, this is your ${fu.step}/${fu.total_steps} follow-up from ${metadata.clinic_name || 'the clinic'}.`,
        ``,
        ...fu.questions.map((q, i) => `${i + 1}. ${q}`),
        ``,
        `Please reply with your answers. Your health matters to us.`,
      ].join('\n');

      await sendText(phone, message, {
        agent: 'patient-follow-up',
        type: 'follow-up-sent',
        step: fu.step,
        visit_type: metadata.visit_type,
      });

      fu.status = 'sent';
      fu.sent_at = new Date().toISOString();
      metadata.awaiting_response = true;
      metadata.current_step = fu.step;

      await updateConversationMetadata(phone, metadata);
      console.log(`Follow-up ${fu.step}/${fu.total_steps} sent to ${phone}`);
      break; // Only send one at a time
    }
  }
}

// -- Webhook handler --
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
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    if (metadata.role !== 'patient' || !metadata.awaiting_response) {
      return; // Ignore messages from non-patients or when not expecting a response
    }

    await sendTyping(phone);

    // Triage the response
    const visitContext = {
      visit_type: metadata.visit_type,
      diagnosis: metadata.diagnosis,
      medications: metadata.medications,
      doctor_notes: metadata.doctor_notes,
    };

    const triage = await triageResponse(messageText, visitContext);

    // Record the response
    if (!metadata.response_history) metadata.response_history = [];
    metadata.response_history.push({
      step: metadata.current_step,
      response: messageText,
      triage: triage.classification,
      reasoning: triage.reasoning,
      responded_at: new Date().toISOString(),
    });

    // Update follow-up step status
    const currentFU = metadata.follow_ups.find(fu => fu.step === metadata.current_step);
    if (currentFU) {
      currentFU.status = 'responded';
      currentFU.response = messageText;
      currentFU.triage = triage.classification;
      currentFU.responded_at = new Date().toISOString();
    }

    metadata.awaiting_response = false;

    // Send appropriate response to patient
    await sendText(phone, triage.follow_up_suggestion, {
      agent: 'patient-follow-up',
      type: 'triage-response',
      classification: triage.classification,
      step: metadata.current_step,
    });

    // Alert doctor for YELLOW and RED classifications
    if (triage.classification === 'RED') {
      await sendText(DOCTOR_PHONE, [
        `*URGENT PATIENT ALERT*`,
        ``,
        `Patient: ${metadata.patient_name}`,
        `Phone: ${phone}`,
        `Visit type: ${metadata.visit_type}`,
        `Diagnosis: ${metadata.diagnosis}`,
        `Follow-up step: ${metadata.current_step}`,
        ``,
        `Patient said: "${messageText}"`,
        ``,
        `AI Assessment: ${triage.reasoning}`,
        ``,
        `Doctor summary: ${triage.doctor_summary}`,
      ].join('\n'), {
        agent: 'patient-follow-up',
        type: 'doctor-alert-urgent',
        patient_phone: phone,
        classification: 'RED',
      });
    } else if (triage.classification === 'YELLOW') {
      await sendText(DOCTOR_PHONE, [
        `*Patient Note (non-urgent)*`,
        ``,
        `Patient: ${metadata.patient_name} (${phone})`,
        `Visit: ${metadata.visit_type} - ${metadata.diagnosis}`,
        `Step ${metadata.current_step}: "${messageText}"`,
        ``,
        `Note: ${triage.doctor_summary}`,
      ].join('\n'), {
        agent: 'patient-follow-up',
        type: 'doctor-alert-note',
        patient_phone: phone,
        classification: 'YELLOW',
      });
    }

    await updateConversationMetadata(phone, metadata);
    console.log(`[${triage.classification}] Response from ${phone} (step ${metadata.current_step}): ${messageText.substring(0, 50)}...`);
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- Admin: register a patient visit --
app.post('/admin/visits', async (req, res) => {
  try {
    const { phone, patient_name, visit_type, diagnosis, medications, doctor_notes, visit_date, clinic_name } = req.body;

    const followUps = scheduleFollowUps({
      visit_type: visit_type || 'general',
      visit_date: visit_date || new Date().toISOString(),
    });

    const metadata = {
      role: 'patient',
      patient_name,
      visit_type: visit_type || 'general',
      diagnosis,
      medications: medications || [],
      doctor_notes: doctor_notes || '',
      clinic_name: clinic_name || 'the clinic',
      visit_date: visit_date || new Date().toISOString(),
      follow_ups: followUps,
      response_history: [],
      awaiting_response: false,
      registered_at: new Date().toISOString(),
    };

    await updateConversationMetadata(phone, metadata);

    // Send initial message
    await sendText(phone,
      `Hi ${patient_name}, thank you for visiting ${clinic_name || 'us'} today. We'll be checking in on your recovery over the next few days via WhatsApp. Please respond to our messages -- it helps us ensure you're recovering well.\n\nIf you experience any emergency symptoms, please go to the ER or call 911 immediately.`,
      { agent: 'patient-follow-up', type: 'registration', visit_type }
    );

    res.json({
      success: true,
      follow_up_schedule: followUps.map(fu => ({
        step: fu.step,
        scheduled_at: fu.scheduled_at,
      })),
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: trigger follow-up processing --
app.post('/admin/process-follow-ups', async (req, res) => {
  try {
    await processFollowUps();
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Cron: run every 30 minutes --
function scheduleFollowUpCron() {
  setInterval(processFollowUps, 30 * 60 * 1000);
  console.log('Follow-up check scheduled every 30 minutes');
}

// -- Health check --
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Patient follow-up bot listening on port ${PORT}`);
  scheduleFollowUpCron();
});
```

## Metadata Schema

Each patient conversation stores visit and follow-up data:

```json
{
  "role": "patient",
  "patient_name": "Carlos Mendez",
  "visit_type": "surgery",
  "diagnosis": "Appendectomy - laparoscopic",
  "medications": ["Ibuprofen 400mg every 8h", "Amoxicillin 500mg every 12h"],
  "doctor_notes": "Uncomplicated procedure. Watch for infection signs.",
  "clinic_name": "Clinica del Sol",
  "visit_date": "2026-04-01T10:00:00.000Z",
  "follow_ups": [
    {
      "step": 1,
      "total_steps": 5,
      "scheduled_at": "2026-04-01T22:00:00.000Z",
      "questions": ["How is your pain level (1-10)?", "Any bleeding or unusual discharge?"],
      "status": "responded",
      "sent_at": "2026-04-01T22:00:30.000Z",
      "response": "Pain is about 6, no bleeding but some swelling around the incision",
      "triage": "YELLOW",
      "responded_at": "2026-04-01T23:15:00.000Z"
    },
    {
      "step": 2,
      "total_steps": 5,
      "scheduled_at": "2026-04-02T10:00:00.000Z",
      "questions": ["Pain level today?", "How does the surgical site look?"],
      "status": "pending"
    }
  ],
  "response_history": [
    {
      "step": 1,
      "response": "Pain is about 6, no bleeding but some swelling around the incision",
      "triage": "YELLOW",
      "reasoning": "Moderate pain and swelling post-surgery, within expected range but worth monitoring",
      "responded_at": "2026-04-01T23:15:00.000Z"
    }
  ],
  "awaiting_response": false,
  "current_step": 1,
  "registered_at": "2026-04-01T10:30:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "patient-follow-up",
  "type": "triage-response",
  "classification": "YELLOW",
  "step": 1
}
```

## How to Run

```bash
# 1. Create the project
mkdir patient-follow-up && cd patient-follow-up
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export DOCTOR_PHONE="5491155551234"

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

# 6. Register a patient visit
curl -X POST http://localhost:3000/admin/visits \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "patient_name": "Carlos Mendez",
    "visit_type": "surgery",
    "diagnosis": "Appendectomy - laparoscopic",
    "medications": ["Ibuprofen 400mg every 8h", "Amoxicillin 500mg every 12h"],
    "doctor_notes": "Uncomplicated procedure. Watch for infection signs.",
    "clinic_name": "Clinica del Sol"
  }'

# 7. Manually trigger follow-up processing
curl -X POST http://localhost:3000/admin/process-follow-ups
```
