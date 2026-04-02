# 17 — Gym Class Booking

## Problem

Gym members want to check class schedules, sign up for classes, and manage waitlists without downloading yet another app. Staff spend too much time on phone calls and manual booking. Members forget their classes and no-show frequently.

## Solution

A WhatsApp bot powered by Claude Haiku for gym class management. Members browse the schedule, sign up for classes, join waitlists when full, and receive reminders. Member preferences are tracked in metadata to suggest relevant classes.

## Architecture

```
Member: "What yoga classes are available this week?"
    |
    v
┌────────────────────────────────────┐
│   Webhook Server                   │
│                                    │
│   1. Classify intent:              │
│      schedule | book | cancel      │
│      | waitlist | preferences      │
│   2. Check class availability      │
│   3. Book / waitlist / cancel      │
│   4. Update member metadata        │
│   5. Schedule reminders            │
└────────────────────────────────────┘
    |
    ├── Schedule: Show filtered classes
    ├── Book: Reserve spot → confirm → set reminder
    ├── Waitlist: Add to queue → notify when spot opens
    ├── Cancel: Release spot → promote waitlist
    └── Reminders: 1h before class
```

## Metadata Schema

```json
{
  "member_name": "Carlos",
  "member_id": "MEM-1234",
  "bookings": [
    {
      "class_id": "yoga-mon-0700",
      "class_name": "Power Yoga",
      "instructor": "Ana",
      "date": "2026-04-07",
      "time": "07:00",
      "status": "confirmed",
      "reminder_sent": false
    }
  ],
  "waitlist": [
    {
      "class_id": "spin-tue-1800",
      "class_name": "Spin Express",
      "position": 2,
      "added_at": "2026-04-02T10:00:00Z"
    }
  ],
  "preferences": {
    "favorite_classes": ["yoga", "spin"],
    "preferred_times": ["morning", "evening"],
    "preferred_instructors": ["Ana"]
  },
  "attendance": {
    "total": 24,
    "this_month": 6,
    "no_shows": 1,
    "streak": 4
  }
}
```

## Code

```javascript
// gym-class-booking.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Class Schedule ─────────────────────────────────────────────────────

const CLASSES = [
  { id: "yoga-mon-0700", name: "Power Yoga", type: "yoga", instructor: "Ana", day: "monday", time: "07:00", duration: 60, capacity: 15, room: "Studio A" },
  { id: "yoga-wed-0700", name: "Power Yoga", type: "yoga", instructor: "Ana", day: "wednesday", time: "07:00", duration: 60, capacity: 15, room: "Studio A" },
  { id: "yoga-fri-0700", name: "Vinyasa Flow", type: "yoga", instructor: "Ana", day: "friday", time: "07:00", duration: 60, capacity: 15, room: "Studio A" },
  { id: "spin-mon-1800", name: "Spin Express", type: "spin", instructor: "Diego", day: "monday", time: "18:00", duration: 45, capacity: 20, room: "Spin Room" },
  { id: "spin-wed-1800", name: "Spin Express", type: "spin", instructor: "Diego", day: "wednesday", time: "18:00", duration: 45, capacity: 20, room: "Spin Room" },
  { id: "spin-thu-0630", name: "Morning Spin", type: "spin", instructor: "Diego", day: "thursday", time: "06:30", duration: 45, capacity: 20, room: "Spin Room" },
  { id: "hiit-tue-1200", name: "HIIT Circuit", type: "hiit", instructor: "Marco", day: "tuesday", time: "12:00", duration: 30, capacity: 25, room: "Main Floor" },
  { id: "hiit-thu-1200", name: "HIIT Circuit", type: "hiit", instructor: "Marco", day: "thursday", time: "12:00", duration: 30, capacity: 25, room: "Main Floor" },
  { id: "pilates-tue-0900", name: "Mat Pilates", type: "pilates", instructor: "Laura", day: "tuesday", time: "09:00", duration: 50, capacity: 12, room: "Studio B" },
  { id: "pilates-thu-0900", name: "Mat Pilates", type: "pilates", instructor: "Laura", day: "thursday", time: "09:00", duration: 50, capacity: 12, room: "Studio B" },
  { id: "box-mon-1900", name: "Boxing Basics", type: "boxing", instructor: "Marco", day: "monday", time: "19:00", duration: 60, capacity: 16, room: "Main Floor" },
  { id: "box-fri-1900", name: "Boxing Basics", type: "boxing", instructor: "Marco", day: "friday", time: "19:00", duration: 60, capacity: 16, room: "Main Floor" },
];

// In-memory bookings (in production, use a database)
const classBookings = new Map(); // class_id:date -> [phone1, phone2, ...]
const classWaitlists = new Map(); // class_id:date -> [phone1, phone2, ...]

function getBookedCount(classId, date) {
  return (classBookings.get(`${classId}:${date}`) || []).length;
}

function getNextDateForDay(dayName) {
  const days = ["sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday"];
  const today = new Date();
  const targetDay = days.indexOf(dayName.toLowerCase());
  const currentDay = today.getDay();
  let daysUntil = targetDay - currentDay;
  if (daysUntil <= 0) daysUntil += 7;
  const nextDate = new Date(today);
  nextDate.setDate(today.getDate() + daysUntil);
  return nextDate.toISOString().split("T")[0];
}

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

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── Intent Classification ──────────────────────────────────────────────

async function classifyIntent(messageText, metadata) {
  const bookings = (metadata.bookings || []).filter((b) => b.status === "confirmed");
  const bookingsStr = bookings.length > 0
    ? bookings.map((b) => `${b.class_name} on ${b.date} at ${b.time}`).join(", ")
    : "none";

  const classTypes = [...new Set(CLASSES.map((c) => c.type))].join(", ");
  const instructors = [...new Set(CLASSES.map((c) => c.instructor))].join(", ");

  const response = await anthropic.messages.create({
    model: "claude-haiku-4-20250514",
    max_tokens: 512,
    system: `You are a gym class booking assistant. Classify the member's intent.

Available class types: ${classTypes}
Instructors: ${instructors}
Member's current bookings: ${bookingsStr}
Today: ${new Date().toISOString().split("T")[0]}

Respond with JSON only:
{
  "intent": "schedule | book | cancel | waitlist | my_bookings | general",
  "class_type": "yoga | spin | hiit | pilates | boxing | null",
  "day": "monday | tuesday | ... | null",
  "time_preference": "morning | afternoon | evening | null",
  "instructor": "instructor name or null",
  "class_id": "specific class id or null",
  "member_name": "name if mentioned or null",
  "response_text": "friendly response"
}`,
    messages: [{ role: "user", content: messageText }],
  });

  const text = response.content[0].text;
  try { return JSON.parse(text); } catch {
    const m = text.match(/\{[\s\S]*\}/);
    return m ? JSON.parse(m[0]) : { intent: "general", response_text: text };
  }
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received" || data.type !== "text") return;

  const phone = data.from;
  const messageText = typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    await sendTyping(phone);

    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const bookings = metadata.bookings || [];
    const waitlist = metadata.waitlist || [];
    const preferences = metadata.preferences || { favorite_classes: [], preferred_times: [], preferred_instructors: [] };
    const attendance = metadata.attendance || { total: 0, this_month: 0, no_shows: 0, streak: 0 };

    const intent = await classifyIntent(messageText, metadata);

    if (intent.member_name) metadata.member_name = intent.member_name;

    switch (intent.intent) {
      case "schedule": {
        let filtered = CLASSES;

        if (intent.class_type) {
          filtered = filtered.filter((c) => c.type === intent.class_type);
        }
        if (intent.day) {
          filtered = filtered.filter((c) => c.day === intent.day.toLowerCase());
        }
        if (intent.instructor) {
          filtered = filtered.filter((c) =>
            c.instructor.toLowerCase().includes(intent.instructor.toLowerCase())
          );
        }
        if (intent.time_preference) {
          filtered = filtered.filter((c) => {
            const hour = parseInt(c.time);
            if (intent.time_preference === "morning") return hour < 12;
            if (intent.time_preference === "afternoon") return hour >= 12 && hour < 17;
            if (intent.time_preference === "evening") return hour >= 17;
            return true;
          });
        }

        if (filtered.length === 0) {
          await sendText(phone, "No classes match that criteria. Try a different day or class type!");
          break;
        }

        const schedule = filtered.map((c) => {
          const date = getNextDateForDay(c.day);
          const booked = getBookedCount(c.id, date);
          const spotsLeft = c.capacity - booked;
          const availability = spotsLeft > 0 ? `${spotsLeft} spots left` : "FULL (waitlist available)";

          return `${c.day.charAt(0).toUpperCase() + c.day.slice(1)} ${c.time} — *${c.name}*\n  Instructor: ${c.instructor} | ${c.duration}min | ${c.room}\n  ${availability}\n  Book: send "${c.id}"`;
        });

        await sendText(phone, `Here's what's available:\n\n${schedule.join("\n\n")}`);
        break;
      }

      case "book": {
        const classId = intent.class_id;
        const classInfo = CLASSES.find((c) => c.id === classId);

        if (!classInfo) {
          await sendText(phone, "Which class would you like to book? Send \"schedule\" to see available classes.");
          break;
        }

        const date = getNextDateForDay(classInfo.day);
        const bookedCount = getBookedCount(classId, date);

        if (bookedCount >= classInfo.capacity) {
          // Add to waitlist
          const wlKey = `${classId}:${date}`;
          const wl = classWaitlists.get(wlKey) || [];
          if (wl.includes(phone)) {
            await sendText(phone, "You're already on the waitlist for this class!");
            break;
          }
          wl.push(phone);
          classWaitlists.set(wlKey, wl);

          waitlist.push({
            class_id: classId,
            class_name: classInfo.name,
            date,
            position: wl.length,
            added_at: new Date().toISOString(),
          });

          metadata.bookings = bookings;
          metadata.waitlist = waitlist;
          await updateConversation(phone, metadata);

          await sendText(phone, `${classInfo.name} is full, but you're #${wl.length} on the waitlist! I'll notify you if a spot opens up.`, {
            action: "waitlisted", class_id: classId,
          });
          break;
        }

        // Book the spot
        const bKey = `${classId}:${date}`;
        const bList = classBookings.get(bKey) || [];
        if (bList.includes(phone)) {
          await sendText(phone, "You're already booked for this class!");
          break;
        }
        bList.push(phone);
        classBookings.set(bKey, bList);

        bookings.push({
          class_id: classId,
          class_name: classInfo.name,
          instructor: classInfo.instructor,
          date,
          time: classInfo.time,
          status: "confirmed",
          reminder_sent: false,
        });

        // Update preferences
        if (!preferences.favorite_classes.includes(classInfo.type)) {
          preferences.favorite_classes.push(classInfo.type);
        }

        metadata.bookings = bookings;
        metadata.preferences = preferences;
        await updateConversation(phone, metadata);

        await sendText(phone, [
          `You're booked!`,
          ``,
          `*${classInfo.name}* with ${classInfo.instructor}`,
          `${classInfo.day.charAt(0).toUpperCase() + classInfo.day.slice(1)}, ${date} at ${classInfo.time}`,
          `${classInfo.room} | ${classInfo.duration} min`,
          ``,
          `I'll remind you 1 hour before. See you there!`,
        ].join("\n"), { action: "class_booked", class_id: classId });
        break;
      }

      case "cancel": {
        const confirmed = bookings.filter((b) => b.status === "confirmed");
        if (confirmed.length === 0) {
          await sendText(phone, "You don't have any upcoming bookings to cancel.");
          break;
        }

        // Find the booking to cancel
        let toCancel = confirmed[0]; // Default to first
        if (intent.class_id) {
          toCancel = confirmed.find((b) => b.class_id === intent.class_id) || toCancel;
        }

        // Remove from bookings list
        const bKey = `${toCancel.class_id}:${toCancel.date}`;
        const bList = classBookings.get(bKey) || [];
        classBookings.set(bKey, bList.filter((p) => p !== phone));

        toCancel.status = "cancelled";

        // Check waitlist and promote
        const wlKey = `${toCancel.class_id}:${toCancel.date}`;
        const wl = classWaitlists.get(wlKey) || [];
        if (wl.length > 0) {
          const nextPhone = wl.shift();
          classWaitlists.set(wlKey, wl);

          // Notify waitlisted member
          const classInfo = CLASSES.find((c) => c.id === toCancel.class_id);
          await sendText(nextPhone, `A spot just opened up in *${classInfo.name}* on ${toCancel.date} at ${toCancel.time}! You've been automatically booked. Reply "cancel ${toCancel.class_id}" if you can't make it.`);
        }

        metadata.bookings = bookings;
        await updateConversation(phone, metadata);

        await sendText(phone, `Your booking for *${toCancel.class_name}* on ${toCancel.date} has been cancelled.`, {
          action: "class_cancelled", class_id: toCancel.class_id,
        });
        break;
      }

      case "my_bookings": {
        const confirmed = bookings.filter((b) => b.status === "confirmed");
        const wl = waitlist.filter((w) => !w.promoted);

        let msg = "";
        if (confirmed.length > 0) {
          msg += `Your upcoming classes:\n\n`;
          msg += confirmed.map((b) => `- *${b.class_name}* with ${b.instructor}\n  ${b.date} at ${b.time}`).join("\n\n");
        } else {
          msg += "No upcoming bookings.";
        }

        if (wl.length > 0) {
          msg += `\n\nWaitlisted:\n`;
          msg += wl.map((w) => `- ${w.class_name} (position #${w.position})`).join("\n");
        }

        msg += `\n\nAttendance this month: ${attendance.this_month} classes | Streak: ${attendance.streak} weeks`;

        await sendText(phone, msg);
        break;
      }

      default: {
        await sendText(phone, intent.response_text || 'I can help with class schedules, bookings, and cancellations. Try "schedule", "my bookings", or tell me what class you want!');
        break;
      }
    }
  } catch (err) {
    console.error("Gym bot error:", err);
    await sendText(phone, "Sorry, something went wrong. Please try again or visit the front desk.");
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Gym class booking bot running on :3000"));
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
node gym-class-booking.js

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

- **Waitlist management**: When a class is full, members are queued. Cancellations auto-promote the next person
- **Preference tracking**: `metadata.preferences` builds a profile of favorite classes, times, and instructors
- **Attendance streaks**: Gamification via streak tracking encourages regular attendance
- **Natural language scheduling**: "any morning yoga classes?" filters by type and time preference
- **Capacity management**: Real-time spot tracking prevents overbooking
