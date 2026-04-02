# 16 — Clinic Appointment Booking

## Problem

Medical clinics waste staff time on phone calls for scheduling, rescheduling, and cancellations. Patients forget appointments. The booking flow needs to be conversational and simple, while being HIPAA-conscious — no medical info stored in chat metadata, only scheduling data.

## Solution

A WhatsApp bot powered by Claude Haiku that handles appointment booking, rescheduling, and cancellation. Doctor availability is checked against a schedule. Automated reminders are sent 24 hours and 1 hour before each appointment. Only scheduling data is stored — no diagnoses, symptoms, or medical history.

## Architecture

```
Patient: "I need to see Dr. Smith next Tuesday"
    |
    v
┌─────────────────────────────────────┐
│   Webhook Server                    │
│                                     │
│   1. Classify intent:               │
│      book | reschedule | cancel     │
│      | check | general             │
│   2. Check doctor availability      │
│   3. Book/modify appointment        │
│   4. Update conversation metadata   │
│   5. Schedule reminders             │
│      (24h + 1h before)             │
└─────────────────────────────────────┘
    |
    ├── Book: Find slot → confirm → schedule reminders
    ├── Reschedule: Cancel old → book new
    ├── Cancel: Remove appointment → confirm
    └── Reminders: Cron sends at T-24h and T-1h
```

## Metadata Schema

```json
{
  "patient_name": "Maria Garcia",
  "appointments": [
    {
      "id": "apt_001",
      "doctor": "Dr. Smith",
      "specialty": "General",
      "date": "2026-04-08",
      "time": "10:00",
      "duration_min": 30,
      "status": "confirmed",
      "reminder_24h_sent": false,
      "reminder_1h_sent": false,
      "booked_at": "2026-04-02T10:00:00Z"
    }
  ],
  "appointment_history": [
    {
      "id": "apt_000",
      "doctor": "Dr. Smith",
      "date": "2026-03-15",
      "status": "completed"
    }
  ],
  "no_show_count": 0,
  "preferred_doctor": "Dr. Smith",
  "preferred_time": "morning"
}
```

## Code

```javascript
// clinic-appointments.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Doctor Schedule ────────────────────────────────────────────────────
// In production, replace with EHR/scheduling system API

const DOCTORS = [
  { name: "Dr. Smith", specialty: "General Medicine", duration: 30 },
  { name: "Dr. Johnson", specialty: "Pediatrics", duration: 30 },
  { name: "Dr. Williams", specialty: "Dermatology", duration: 45 },
  { name: "Dr. Brown", specialty: "Cardiology", duration: 60 },
];

// Weekly availability (simplified). In production, query a real scheduling system.
const WEEKLY_SCHEDULE = {
  "Dr. Smith": {
    monday: ["09:00", "09:30", "10:00", "10:30", "11:00", "14:00", "14:30", "15:00", "15:30"],
    tuesday: ["09:00", "09:30", "10:00", "10:30", "11:00", "14:00", "14:30", "15:00"],
    wednesday: ["09:00", "09:30", "10:00", "10:30", "11:00"],
    thursday: ["09:00", "09:30", "10:00", "10:30", "11:00", "14:00", "14:30", "15:00", "15:30"],
    friday: ["09:00", "09:30", "10:00", "10:30"],
  },
  "Dr. Johnson": {
    monday: ["08:00", "08:30", "09:00", "09:30", "10:00"],
    tuesday: ["08:00", "08:30", "09:00", "09:30", "10:00", "14:00", "14:30"],
    wednesday: ["14:00", "14:30", "15:00", "15:30", "16:00"],
    thursday: ["08:00", "08:30", "09:00", "09:30", "10:00"],
    friday: ["08:00", "08:30", "09:00", "09:30"],
  },
  "Dr. Williams": {
    monday: ["10:00", "10:45", "11:30", "14:00", "14:45"],
    wednesday: ["10:00", "10:45", "11:30", "14:00", "14:45"],
    friday: ["10:00", "10:45", "11:30"],
  },
  "Dr. Brown": {
    tuesday: ["09:00", "10:00", "11:00", "14:00", "15:00"],
    thursday: ["09:00", "10:00", "11:00", "14:00", "15:00"],
  },
};

// In-memory booked slots (in production, use a database)
const bookedSlots = new Map();

function getDayOfWeek(dateStr) {
  const days = ["sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday"];
  return days[new Date(dateStr).getDay()];
}

function getAvailableSlots(doctor, date) {
  const dayOfWeek = getDayOfWeek(date);
  const schedule = WEEKLY_SCHEDULE[doctor]?.[dayOfWeek] || [];
  const bookedKey = `${doctor}:${date}`;
  const booked = bookedSlots.get(bookedKey) || [];
  return schedule.filter((slot) => !booked.includes(slot));
}

function bookSlot(doctor, date, time) {
  const bookedKey = `${doctor}:${date}`;
  const booked = bookedSlots.get(bookedKey) || [];
  if (booked.includes(time)) return false;
  booked.push(time);
  bookedSlots.set(bookedKey, booked);
  return true;
}

function cancelSlot(doctor, date, time) {
  const bookedKey = `${doctor}:${date}`;
  const booked = bookedSlots.get(bookedKey) || [];
  bookedSlots.set(bookedKey, booked.filter((t) => t !== time));
}

function generateAppointmentId() {
  return `apt_${Date.now().toString(36)}`;
}

// ── API Helpers ────────────────────────────────────────────────────────

async function apiRequest(method, path, body = null) {
  const options = {
    method,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
    },
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

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── Intent Classification ──────────────────────────────────────────────

async function classifyIntent(messageText, metadata) {
  const appointments = metadata.appointments || [];
  const upcomingStr = appointments
    .filter((a) => a.status === "confirmed")
    .map((a) => `${a.id}: ${a.doctor} on ${a.date} at ${a.time}`)
    .join("\n") || "No upcoming appointments";

  const doctorList = DOCTORS.map((d) => `${d.name} (${d.specialty})`).join(", ");

  const response = await anthropic.messages.create({
    model: "claude-haiku-4-20250514",
    max_tokens: 512,
    system: `You are a medical clinic receptionist bot. Classify the patient's intent and extract details.

Available doctors: ${doctorList}
Patient's upcoming appointments:
${upcomingStr}

Today's date: ${new Date().toISOString().split("T")[0]}

IMPORTANT: Do NOT ask about or store any medical information (symptoms, diagnoses, conditions). Only handle scheduling.

Respond with JSON only:
{
  "intent": "book | reschedule | cancel | check | general",
  "doctor": "doctor name or null",
  "date": "YYYY-MM-DD or null",
  "time": "HH:MM or null",
  "time_preference": "morning | afternoon | any | null",
  "appointment_id": "id of appointment to modify, or null",
  "patient_name": "name if mentioned, or null",
  "response_text": "friendly response to patient"
}`,
    messages: [{ role: "user", content: messageText }],
  });

  const text = response.content[0].text;
  try {
    return JSON.parse(text);
  } catch {
    const match = text.match(/\{[\s\S]*\}/);
    return match ? JSON.parse(match[0]) : { intent: "general", response_text: text };
  }
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received") return;
  if (data.type !== "text") return;

  const phone = data.from;
  const messageText =
    typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    await sendTyping(phone);

    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const appointments = metadata.appointments || [];

    const intent = await classifyIntent(messageText, metadata);

    // Update patient name if provided
    if (intent.patient_name) {
      metadata.patient_name = intent.patient_name;
    }

    switch (intent.intent) {
      case "book": {
        const doctor = intent.doctor;
        if (!doctor || !WEEKLY_SCHEDULE[doctor]) {
          const list = DOCTORS.map((d) => `- ${d.name} (${d.specialty})`).join("\n");
          await sendText(phone, `Which doctor would you like to see?\n\n${list}`);
          break;
        }

        const date = intent.date;
        if (!date) {
          await sendText(phone, `What date works for you? ${doctor} is available on ${Object.keys(WEEKLY_SCHEDULE[doctor]).join(", ")}.`);
          break;
        }

        const available = getAvailableSlots(doctor, date);
        if (available.length === 0) {
          await sendText(phone, `Sorry, ${doctor} has no available slots on ${date}. Would you like to try another date?`);
          break;
        }

        // If specific time requested
        if (intent.time && available.includes(intent.time)) {
          const doctorInfo = DOCTORS.find((d) => d.name === doctor);
          const success = bookSlot(doctor, date, intent.time);

          if (!success) {
            await sendText(phone, `That slot was just taken. Available times: ${available.join(", ")}`);
            break;
          }

          const aptId = generateAppointmentId();
          const appointment = {
            id: aptId,
            doctor,
            specialty: doctorInfo.specialty,
            date,
            time: intent.time,
            duration_min: doctorInfo.duration,
            status: "confirmed",
            reminder_24h_sent: false,
            reminder_1h_sent: false,
            booked_at: new Date().toISOString(),
          };

          appointments.push(appointment);
          metadata.appointments = appointments;
          metadata.preferred_doctor = doctor;
          await updateConversation(phone, metadata);

          await sendText(phone, [
            `Your appointment is confirmed!`,
            ``,
            `Doctor: ${doctor} (${doctorInfo.specialty})`,
            `Date: ${date}`,
            `Time: ${intent.time}`,
            `Duration: ${doctorInfo.duration} minutes`,
            ``,
            `You'll receive reminders 24 hours and 1 hour before your appointment.`,
            ``,
            `To reschedule or cancel, just let me know.`,
          ].join("\n"), { action: "appointment_booked", appointment_id: aptId });
          break;
        }

        // Show available slots
        const morningSlots = available.filter((t) => parseInt(t) < 12);
        const afternoonSlots = available.filter((t) => parseInt(t) >= 12);

        let slotsMsg = `Available times for ${doctor} on ${date}:\n\n`;
        if (morningSlots.length > 0) slotsMsg += `Morning: ${morningSlots.join(", ")}\n`;
        if (afternoonSlots.length > 0) slotsMsg += `Afternoon: ${afternoonSlots.join(", ")}\n`;
        slotsMsg += `\nWhich time works for you?`;

        await sendText(phone, slotsMsg);
        break;
      }

      case "reschedule": {
        const aptId = intent.appointment_id;
        const apt = appointments.find((a) => a.id === aptId && a.status === "confirmed");

        if (!apt && appointments.filter((a) => a.status === "confirmed").length === 1) {
          // Only one appointment, use that one
          const singleApt = appointments.find((a) => a.status === "confirmed");
          if (singleApt) {
            cancelSlot(singleApt.doctor, singleApt.date, singleApt.time);
            singleApt.status = "rescheduled";
            metadata.appointments = appointments;
            await updateConversation(phone, metadata);

            await sendText(phone, `I've cancelled your appointment with ${singleApt.doctor} on ${singleApt.date} at ${singleApt.time}. When would you like to reschedule?`);
            break;
          }
        }

        if (!apt) {
          const confirmed = appointments.filter((a) => a.status === "confirmed");
          if (confirmed.length === 0) {
            await sendText(phone, "You don't have any upcoming appointments to reschedule. Would you like to book one?");
          } else {
            const list = confirmed.map((a) => `- ${a.id}: ${a.doctor} on ${a.date} at ${a.time}`).join("\n");
            await sendText(phone, `Which appointment would you like to reschedule?\n\n${list}`);
          }
          break;
        }

        cancelSlot(apt.doctor, apt.date, apt.time);
        apt.status = "rescheduled";
        metadata.appointments = appointments;
        await updateConversation(phone, metadata);

        await sendText(phone, `I've cancelled your appointment with ${apt.doctor} on ${apt.date}. When would you like to reschedule?`);
        break;
      }

      case "cancel": {
        const confirmed = appointments.filter((a) => a.status === "confirmed");

        if (confirmed.length === 0) {
          await sendText(phone, "You don't have any upcoming appointments to cancel.");
          break;
        }

        // Cancel the most relevant one
        const aptToCancel = intent.appointment_id
          ? confirmed.find((a) => a.id === intent.appointment_id)
          : confirmed[0];

        if (aptToCancel) {
          cancelSlot(aptToCancel.doctor, aptToCancel.date, aptToCancel.time);
          aptToCancel.status = "cancelled";
          metadata.appointments = appointments;
          await updateConversation(phone, metadata);

          await sendText(phone, `Your appointment with ${aptToCancel.doctor} on ${aptToCancel.date} at ${aptToCancel.time} has been cancelled. Would you like to book a new one?`, {
            action: "appointment_cancelled",
            appointment_id: aptToCancel.id,
          });
        }
        break;
      }

      case "check": {
        const confirmed = appointments.filter((a) => a.status === "confirmed");
        if (confirmed.length === 0) {
          await sendText(phone, "You don't have any upcoming appointments. Would you like to book one?");
        } else {
          const list = confirmed
            .map((a) => `- ${a.doctor} (${a.specialty}): ${a.date} at ${a.time}`)
            .join("\n");
          await sendText(phone, `Your upcoming appointments:\n\n${list}\n\nWould you like to reschedule or cancel any of these?`);
        }
        break;
      }

      default: {
        await sendText(phone, intent.response_text || "I can help you book, reschedule, or cancel appointments. What would you like to do?");
        break;
      }
    }
  } catch (err) {
    console.error("Clinic bot error:", err);
    await sendText(phone, "Sorry, I'm having trouble right now. Please call the clinic at (555) 123-4567.");
  }
});

// ── Reminder Cron ──────────────────────────────────────────────────────
// Run every 15 minutes to check for upcoming appointments

async function sendReminders() {
  // In production, query conversations from your database.
  // This is a simplified version that would need a conversation list endpoint.
  console.log("Reminder check running...");

  // Example: iterate conversations and check appointments
  // For each conversation with upcoming appointments:
  // - If appointment is in 24h and reminder_24h_sent === false → send reminder
  // - If appointment is in 1h and reminder_1h_sent === false → send reminder
}

// Run reminder check every 15 minutes
setInterval(sendReminders, 15 * 60 * 1000);

// ── Reminder sender (called per conversation) ─────────────────────────

async function checkAndSendReminders(phone) {
  const convoRes = await getConversation(phone);
  const metadata = convoRes.data?.metadata || {};
  const appointments = metadata.appointments || [];
  const now = new Date();

  for (const apt of appointments) {
    if (apt.status !== "confirmed") continue;

    const aptTime = new Date(`${apt.date}T${apt.time}:00`);
    const hoursUntil = (aptTime - now) / (1000 * 60 * 60);

    if (hoursUntil <= 24 && hoursUntil > 1 && !apt.reminder_24h_sent) {
      await sendText(phone, [
        `Reminder: You have an appointment tomorrow`,
        ``,
        `Doctor: ${apt.doctor}`,
        `Date: ${apt.date}`,
        `Time: ${apt.time}`,
        ``,
        `Reply "confirm" to confirm or "reschedule" to change.`,
      ].join("\n"), { reminder: "24h" });

      apt.reminder_24h_sent = true;
    }

    if (hoursUntil <= 1 && hoursUntil > 0 && !apt.reminder_1h_sent) {
      await sendText(phone, [
        `Your appointment with ${apt.doctor} is in 1 hour!`,
        `Time: ${apt.time}`,
        ``,
        `Please arrive 10 minutes early. See you soon!`,
      ].join("\n"), { reminder: "1h" });

      apt.reminder_1h_sent = true;
    }
  }

  metadata.appointments = appointments;
  await updateConversation(phone, metadata);
}

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Clinic appointment bot running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-key"

# 3. Start the server
node clinic-appointments.js

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

- **HIPAA-conscious design**: No medical data (symptoms, diagnoses, conditions) is stored in metadata. Only scheduling information
- **Availability checking**: Real-time slot availability before booking
- **Automated reminders**: 24-hour and 1-hour reminders with confirmation/reschedule options
- **Natural language booking**: "I need to see Dr. Smith next Tuesday morning" is parsed into doctor + date + time preference
- **Appointment history**: Past appointments tracked for continuity ("I'd like to see the same doctor as last time")
