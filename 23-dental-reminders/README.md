# 23 — Dental Appointment Reminders

## Problem

Dental practices lose revenue from no-shows. Patients forget appointments, and manual reminder calls are expensive and unreliable. When patients do receive reminders, there's no easy way to confirm, reschedule, or cancel — they have to call back during office hours.

## Solution

An automated reminder system that sends appointment reminders at 1 week, 1 day, and 1 hour before the appointment. Patients can reply to confirm, reschedule, or cancel directly in WhatsApp. Patient response history is tracked in metadata for future communication optimization.

## Architecture

```
┌──────────────────────────────────────────────┐
│   Reminder Scheduler (runs every 15 min)     │
│                                              │
│   For each upcoming appointment:             │
│   ├── 7 days before → Send week reminder     │
│   ├── 1 day before  → Send day reminder      │
│   └── 1 hour before → Send hour reminder     │
│                                              │
│   Track: which reminders sent, patient reply  │
└──────────────────────────────────────────────┘
              |                    ^
              v                    |
┌─────────────────────────────────────────────┐
│   Webhook Server                            │
│                                             │
│   Patient replies:                          │
│   "confirm" → Mark confirmed                │
│   "cancel"  → Cancel appointment            │
│   "reschedule" → Offer alternatives         │
│   Other → Forward to front desk             │
└─────────────────────────────────────────────┘
```

## Metadata Schema

```json
{
  "patient": {
    "name": "Sofia Fernandez",
    "preferred_name": "Sofi",
    "last_visit": "2025-10-15"
  },
  "appointments": [
    {
      "id": "APT-20260408-001",
      "type": "cleaning",
      "dentist": "Dr. Garcia",
      "date": "2026-04-08",
      "time": "10:00",
      "duration_min": 45,
      "status": "confirmed",
      "reminders": {
        "week": { "sent": true, "sent_at": "2026-04-01T10:00:00Z", "response": "seen" },
        "day": { "sent": true, "sent_at": "2026-04-07T10:00:00Z", "response": "confirmed" },
        "hour": { "sent": false }
      }
    }
  ],
  "response_history": [
    { "appointment_id": "APT-20251015-001", "confirmed_at_reminder": "day", "attended": true },
    { "appointment_id": "APT-20250615-001", "confirmed_at_reminder": "week", "attended": true }
  ],
  "no_show_count": 0,
  "total_appointments": 5,
  "preferred_reminder_time": "morning"
}
```

## Code

```javascript
// dental-reminders.js
const express = require("express");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const FRONT_DESK_PHONE = process.env.FRONT_DESK_PHONE;

// ── Appointment Database ───────────────────────────────────────────────
// In production, connect to your dental practice management system

const appointments = [
  {
    id: "APT-20260408-001",
    patient_phone: "5491155550001",
    patient_name: "Sofia Fernandez",
    type: "cleaning",
    dentist: "Dr. Garcia",
    date: "2026-04-08",
    time: "10:00",
    duration_min: 45,
  },
  {
    id: "APT-20260408-002",
    patient_phone: "5491155550002",
    patient_name: "Carlos Mendez",
    type: "filling",
    dentist: "Dr. Rodriguez",
    date: "2026-04-08",
    time: "11:00",
    duration_min: 60,
  },
  {
    id: "APT-20260409-001",
    patient_phone: "5491155550003",
    patient_name: "Ana Torres",
    type: "checkup",
    dentist: "Dr. Garcia",
    date: "2026-04-09",
    time: "09:00",
    duration_min: 30,
  },
  {
    id: "APT-20260410-001",
    patient_phone: "5491155550001",
    patient_name: "Sofia Fernandez",
    type: "follow-up",
    dentist: "Dr. Garcia",
    date: "2026-04-10",
    time: "14:00",
    duration_min: 30,
  },
];

const APPOINTMENT_TYPES = {
  cleaning: "Dental Cleaning",
  checkup: "Routine Checkup",
  filling: "Filling",
  "root-canal": "Root Canal",
  crown: "Crown Fitting",
  extraction: "Extraction",
  "follow-up": "Follow-up Visit",
  whitening: "Teeth Whitening",
};

// ── API Helpers ────────────────────────────────────────────────────────

async function apiRequest(method, path, body = null) {
  const options = {
    method,
    headers: { Authorization: `Bearer ${API_KEY}`, "Content-Type": "application/json" },
  };
  if (body) options.body = JSON.stringify(body);
  const res = await fetch(`${API_BASE}${path}`, options);
  return res.json();
}

async function getConversation(phone) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}`);
}

async function updateConversation(phone, metadata) {
  return apiRequest("PUT", `/api/numbers/${NUMBER_ID}/conversations/${phone}`, { metadata });
}

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, { phone, text, metadata });
}

// ── Reminder Messages ──────────────────────────────────────────────────

function weekReminderMessage(apt) {
  return [
    `Hi ${apt.patient_name}! This is a reminder from Bright Smile Dental.`,
    ``,
    `You have an appointment coming up next week:`,
    ``,
    `${APPOINTMENT_TYPES[apt.type] || apt.type}`,
    `Dentist: ${apt.dentist}`,
    `Date: ${formatDate(apt.date)}`,
    `Time: ${apt.time}`,
    `Duration: ~${apt.duration_min} minutes`,
    ``,
    `Reply:`,
    `- "confirm" to confirm`,
    `- "reschedule" to change the date/time`,
    `- "cancel" to cancel`,
  ].join("\n");
}

function dayReminderMessage(apt) {
  return [
    `Reminder: Your dental appointment is *tomorrow*!`,
    ``,
    `${APPOINTMENT_TYPES[apt.type] || apt.type} with ${apt.dentist}`,
    `Time: ${apt.time}`,
    ``,
    `Please arrive 10 minutes early.`,
    `Bring your insurance card if applicable.`,
    ``,
    `Reply "confirm" or "reschedule" if needed.`,
  ].join("\n");
}

function hourReminderMessage(apt) {
  return [
    `Your appointment with ${apt.dentist} is in 1 hour!`,
    ``,
    `${APPOINTMENT_TYPES[apt.type] || apt.type} at ${apt.time}`,
    ``,
    `We're looking forward to seeing you!`,
  ].join("\n");
}

function formatDate(dateStr) {
  const date = new Date(dateStr + "T00:00:00");
  return date.toLocaleDateString("en-US", {
    weekday: "long",
    month: "long",
    day: "numeric",
  });
}

// ── Reminder Scheduler ─────────────────────────────────────────────────

async function processReminders() {
  const now = new Date();

  for (const apt of appointments) {
    const aptDate = new Date(`${apt.date}T${apt.time}:00`);
    const hoursUntil = (aptDate - now) / (1000 * 60 * 60);

    // Load conversation metadata
    let convoRes, metadata;
    try {
      convoRes = await getConversation(apt.patient_phone);
      metadata = convoRes.data?.metadata || {};
    } catch {
      metadata = {};
    }

    // Initialize appointment tracking in metadata
    const aptMeta = findOrCreateAptMeta(metadata, apt);

    // Check if appointment is cancelled
    if (aptMeta.status === "cancelled") continue;

    // Week reminder (168 hours = 7 days, send between 168-156 hours)
    if (hoursUntil <= 168 && hoursUntil > 156 && !aptMeta.reminders.week.sent) {
      await sendText(apt.patient_phone, weekReminderMessage(apt), {
        reminder_type: "week",
        appointment_id: apt.id,
      });

      aptMeta.reminders.week.sent = true;
      aptMeta.reminders.week.sent_at = now.toISOString();
      await saveAptMeta(apt.patient_phone, metadata, aptMeta, apt);

      console.log(`[WEEK REMINDER] ${apt.patient_name} — ${apt.date} ${apt.time}`);
    }

    // Day reminder (24-20 hours before)
    if (hoursUntil <= 24 && hoursUntil > 20 && !aptMeta.reminders.day.sent) {
      await sendText(apt.patient_phone, dayReminderMessage(apt), {
        reminder_type: "day",
        appointment_id: apt.id,
      });

      aptMeta.reminders.day.sent = true;
      aptMeta.reminders.day.sent_at = now.toISOString();
      await saveAptMeta(apt.patient_phone, metadata, aptMeta, apt);

      console.log(`[DAY REMINDER] ${apt.patient_name} — ${apt.date} ${apt.time}`);
    }

    // Hour reminder (60-45 minutes before)
    if (hoursUntil <= 1 && hoursUntil > 0.75 && !aptMeta.reminders.hour.sent) {
      await sendText(apt.patient_phone, hourReminderMessage(apt), {
        reminder_type: "hour",
        appointment_id: apt.id,
      });

      aptMeta.reminders.hour.sent = true;
      aptMeta.reminders.hour.sent_at = now.toISOString();
      await saveAptMeta(apt.patient_phone, metadata, aptMeta, apt);

      console.log(`[HOUR REMINDER] ${apt.patient_name} — ${apt.date} ${apt.time}`);
    }
  }
}

function findOrCreateAptMeta(metadata, apt) {
  metadata.appointments = metadata.appointments || [];
  let aptMeta = metadata.appointments.find((a) => a.id === apt.id);

  if (!aptMeta) {
    aptMeta = {
      id: apt.id,
      type: apt.type,
      dentist: apt.dentist,
      date: apt.date,
      time: apt.time,
      duration_min: apt.duration_min,
      status: "pending",
      reminders: {
        week: { sent: false },
        day: { sent: false },
        hour: { sent: false },
      },
    };
    metadata.appointments.push(aptMeta);
  }

  return aptMeta;
}

async function saveAptMeta(phone, metadata, aptMeta, apt) {
  metadata.patient = metadata.patient || { name: apt.patient_name };
  const idx = metadata.appointments.findIndex((a) => a.id === aptMeta.id);
  if (idx >= 0) metadata.appointments[idx] = aptMeta;
  await updateConversation(phone, metadata);
}

// Run reminder check every 15 minutes
setInterval(processReminders, 15 * 60 * 1000);
// Also run on startup
setTimeout(processReminders, 5000);

// ── Webhook Handler (patient replies) ──────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received" || data.type !== "text") return;

  const phone = data.from;
  const messageText = (typeof data.content === "string" ? data.content : data.content?.text || "").toLowerCase().trim();

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const patientApts = metadata.appointments || [];

    // Find the next upcoming appointment for this patient
    const upcoming = patientApts
      .filter((a) => a.status !== "cancelled" && new Date(`${a.date}T${a.time}:00`) > new Date())
      .sort((a, b) => new Date(`${a.date}T${a.time}:00`) - new Date(`${b.date}T${b.time}:00`))[0];

    if (!upcoming) {
      await sendText(phone, "Hi! You don't have any upcoming appointments. Would you like to schedule one? Call us at (555) 234-5678 or reply with your preferred date and time.");
      return;
    }

    if (messageText.includes("confirm") || messageText === "yes" || messageText === "si") {
      upcoming.status = "confirmed";

      // Track which reminder prompted the confirmation
      const responseHistory = metadata.response_history || [];
      const lastReminder = upcoming.reminders.hour.sent
        ? "hour"
        : upcoming.reminders.day.sent
          ? "day"
          : "week";
      responseHistory.push({
        appointment_id: upcoming.id,
        confirmed_at_reminder: lastReminder,
        confirmed_at: new Date().toISOString(),
      });
      metadata.response_history = responseHistory;

      await updateConversation(phone, metadata);

      await sendText(phone, [
        `Your appointment is confirmed!`,
        ``,
        `${APPOINTMENT_TYPES[upcoming.type] || upcoming.type} with ${upcoming.dentist}`,
        `${formatDate(upcoming.date)} at ${upcoming.time}`,
        ``,
        `See you then!`,
      ].join("\n"), { action: "appointment_confirmed", appointment_id: upcoming.id });
    } else if (messageText.includes("cancel")) {
      upcoming.status = "cancelled";
      await updateConversation(phone, metadata);

      await sendText(phone, [
        `Your appointment on ${formatDate(upcoming.date)} at ${upcoming.time} has been cancelled.`,
        ``,
        `If you'd like to reschedule, just let us know or call (555) 234-5678.`,
        ``,
        `We recommend scheduling your next visit within 6 months.`,
      ].join("\n"), { action: "appointment_cancelled", appointment_id: upcoming.id });

      // Notify front desk
      if (FRONT_DESK_PHONE) {
        await sendText(FRONT_DESK_PHONE, `Cancellation: ${metadata.patient?.name || phone} cancelled ${upcoming.type} on ${upcoming.date} at ${upcoming.time} with ${upcoming.dentist}.`, {
          alert_type: "cancellation",
          appointment_id: upcoming.id,
        });
      }
    } else if (messageText.includes("reschedule") || messageText.includes("change")) {
      upcoming.status = "reschedule_requested";
      await updateConversation(phone, metadata);

      await sendText(phone, [
        `No problem! Let's find a new time for your ${APPOINTMENT_TYPES[upcoming.type] || upcoming.type}.`,
        ``,
        `${upcoming.dentist} has availability:`,
        `- Tomorrow: 9:00 AM, 2:00 PM, 3:30 PM`,
        `- Thursday: 10:00 AM, 11:30 AM, 4:00 PM`,
        `- Friday: 9:00 AM, 1:00 PM`,
        ``,
        `Which works best for you? Or suggest another date.`,
      ].join("\n"), { action: "reschedule_requested", appointment_id: upcoming.id });

      if (FRONT_DESK_PHONE) {
        await sendText(FRONT_DESK_PHONE, `Reschedule request: ${metadata.patient?.name || phone} wants to reschedule ${upcoming.type} on ${upcoming.date} at ${upcoming.time}.`, {
          alert_type: "reschedule",
          appointment_id: upcoming.id,
        });
      }
    } else {
      // Forward to front desk
      await sendText(phone, `Thank you for your message. Our front desk team will get back to you shortly. Your next appointment is on ${formatDate(upcoming.date)} at ${upcoming.time}.`);

      if (FRONT_DESK_PHONE) {
        await sendText(FRONT_DESK_PHONE, `Message from patient ${metadata.patient?.name || phone}: "${data.content}"`, {
          alert_type: "patient_message",
        });
      }
    }
  } catch (err) {
    console.error("Dental reminders error:", err);
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Dental reminders bot running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export FRONT_DESK_PHONE="5491155551234"  # Front desk phone

# 3. Start the server
node dental-reminders.js

# 4. Expose via ngrok
ngrok http 3000

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'
```

## Key Concepts

- **Three-stage reminders**: 1 week, 1 day, and 1 hour before — each with appropriate urgency and detail
- **Reply handling**: Patients confirm, cancel, or reschedule by replying to the reminder
- **No AI required**: This use case runs without any AI model — pure logic-based reminders and keyword matching
- **Response tracking**: `metadata.response_history` shows which reminders get responses, helping optimize timing
- **Front desk alerts**: Cancellations and reschedule requests are forwarded to staff instantly
- **Idempotent reminders**: Each reminder is tracked with `sent` flag to prevent duplicates
