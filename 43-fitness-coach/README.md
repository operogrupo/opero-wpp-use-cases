# 43 - AI Fitness Coach

Personalized AI fitness coach via WhatsApp. Delivers daily workout plans, tracks completed workouts, adjusts difficulty based on feedback, and sends weekly progress reports.

## Problem

A personal trainer manages 80 clients but can only meet each one twice a week. Between sessions, clients lose motivation, skip workouts, and don't know what to do on their own. The trainer charges $50/session but loses clients after 2-3 months because they don't see consistent progress. Hiring more trainers costs $4,000/month per hire.

## Solution

Deploy an AI fitness coach that:
- Generates personalized daily workout plans based on fitness level, goals, and available equipment
- Tracks completed workouts and rest days in conversation metadata
- Adjusts workout difficulty based on user feedback (too easy/hard)
- Sends weekly progress summaries with stats
- Motivates with streak tracking and milestone celebrations
- Escalates to the human trainer for injury concerns or plateaus

## Architecture

```
Client sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
| wpp-api.opero.so |                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |  Claude Sonnet   |
         |                                 | (Anthropic API)  |
         |                                 +------------------+
         |                                          |
         |     GET/PUT conversation metadata        |
         |     POST /messages/text                  |
         +------------------------------------------+
         |
         v
+------------------+
|  Cron: Weekly     |
|  Progress Reports |
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

async function getMessages(phone, limit = 20) {
  try {
    const res = await operoFetch(
      `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`
    );
    return res.data || [];
  } catch {
    return [];
  }
}

async function listConversations() {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=500`);
    return res.data || [];
  } catch {
    return [];
  }
}

// -- Fitness logic --
function initializeProfile(metadata) {
  return {
    ...metadata,
    fitness_level: metadata.fitness_level || 'beginner', // beginner, intermediate, advanced
    goals: metadata.goals || [],
    equipment: metadata.equipment || ['bodyweight'],
    workout_history: metadata.workout_history || [],
    current_streak: metadata.current_streak || 0,
    longest_streak: metadata.longest_streak || 0,
    total_workouts: metadata.total_workouts || 0,
    difficulty_modifier: metadata.difficulty_modifier || 0, // -2 to +2
    weekly_target: metadata.weekly_target || 4,
    rest_days: metadata.rest_days || ['Sunday'],
    onboarded: metadata.onboarded || false,
    joined_at: metadata.joined_at || new Date().toISOString(),
  };
}

function getToday() {
  return new Date().toISOString().split('T')[0];
}

function getWeekStart() {
  const now = new Date();
  const day = now.getDay();
  const diff = now.getDate() - day + (day === 0 ? -6 : 1);
  return new Date(now.setDate(diff)).toISOString().split('T')[0];
}

function calculateStreak(history) {
  if (history.length === 0) return 0;

  const sorted = [...history]
    .sort((a, b) => new Date(b.date) - new Date(a.date));

  let streak = 0;
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  for (let i = 0; i < sorted.length; i++) {
    const workoutDate = new Date(sorted[i].date);
    workoutDate.setHours(0, 0, 0, 0);

    const expectedDate = new Date(today);
    expectedDate.setDate(expectedDate.getDate() - i);

    // Allow 1 day gap (rest days)
    const diffDays = Math.abs((expectedDate - workoutDate) / (1000 * 60 * 60 * 24));
    if (diffDays <= 1) {
      streak++;
    } else {
      break;
    }
  }
  return streak;
}

function getWeeklyStats(history) {
  const weekStart = getWeekStart();
  const thisWeek = history.filter(w => w.date >= weekStart);

  return {
    workouts_this_week: thisWeek.length,
    total_duration_min: thisWeek.reduce((sum, w) => sum + (w.duration_min || 0), 0),
    exercises_completed: thisWeek.reduce((sum, w) => sum + (w.exercises_completed || 0), 0),
    avg_difficulty_rating: thisWeek.length > 0
      ? (thisWeek.reduce((sum, w) => sum + (w.difficulty_rating || 3), 0) / thisWeek.length).toFixed(1)
      : 'N/A',
  };
}

// -- AI integration --
function buildSystemPrompt(metadata) {
  const profile = initializeProfile(metadata);
  const stats = getWeeklyStats(profile.workout_history);

  return `You are Coach Fit, a motivating and knowledgeable AI fitness coach on WhatsApp.

Client profile:
- Fitness level: ${profile.fitness_level} (modifier: ${profile.difficulty_modifier > 0 ? '+' : ''}${profile.difficulty_modifier})
- Goals: ${profile.goals.length > 0 ? profile.goals.join(', ') : 'Not set yet'}
- Available equipment: ${profile.equipment.join(', ')}
- Weekly target: ${profile.weekly_target} workouts
- Rest days: ${profile.rest_days.join(', ')}
- Current streak: ${profile.current_streak} days
- Longest streak: ${profile.longest_streak} days
- Total workouts: ${profile.total_workouts}

This week's stats:
- Workouts: ${stats.workouts_this_week}/${profile.weekly_target}
- Total training time: ${stats.total_duration_min} min
- Exercises completed: ${stats.exercises_completed}

${!profile.onboarded ? `This is a new client. Ask about their:
1. Fitness level (beginner/intermediate/advanced)
2. Goals (weight loss, muscle gain, flexibility, endurance, general fitness)
3. Available equipment (bodyweight, dumbbells, resistance bands, full gym)
4. How many days per week they want to train
Tag your response with [ONBOARD] so we know to collect their answers.` : ''}

When generating a workout, format it clearly with:
- Warm-up (5 min)
- Main workout (exercises with sets x reps)
- Cool-down (5 min)
- Estimated duration

Use these action tags in your responses:
- [WORKOUT_GENERATED] - when you create a new workout plan
- [WORKOUT_LOGGED:DURATION_MIN:EXERCISES_COUNT] - when client reports completing a workout (e.g., [WORKOUT_LOGGED:45:8])
- [DIFFICULTY_ADJUST:UP] or [DIFFICULTY_ADJUST:DOWN] - when client says it was too easy/hard
- [PROFILE_UPDATE:FIELD:VALUE] - to update profile (e.g., [PROFILE_UPDATE:fitness_level:intermediate])
- [INJURY_ALERT] - if client mentions pain or injury (escalate to human trainer)

Be encouraging but realistic. Use emojis sparingly. Keep messages under 300 words unless delivering a full workout plan.`;
}

async function generateReply(phone, message, metadata) {
  const history = await getMessages(phone, 10);
  const messages = history
    .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    .map(m => ({
      role: m.from_me ? 'assistant' : 'user',
      content: m.text || m.content?.text || '',
    }))
    .filter(m => m.content);

  messages.push({ role: 'user', content: message });

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1500,
    system: buildSystemPrompt(metadata),
    messages: messages.slice(-20),
  });

  return response.content[0].text;
}

// -- Action processor --
async function processActions(phone, aiResponse, metadata) {
  let finalResponse = aiResponse;
  const profile = initializeProfile(metadata);

  // Handle workout logging
  const logMatch = aiResponse.match(/\[WORKOUT_LOGGED:(\d+):(\d+)\]/);
  if (logMatch) {
    const durationMin = parseInt(logMatch[1]);
    const exercisesCount = parseInt(logMatch[2]);

    profile.workout_history.push({
      date: getToday(),
      duration_min: durationMin,
      exercises_completed: exercisesCount,
      logged_at: new Date().toISOString(),
    });

    profile.total_workouts++;
    profile.current_streak = calculateStreak(profile.workout_history);
    profile.longest_streak = Math.max(profile.longest_streak, profile.current_streak);

    finalResponse = finalResponse.replace(logMatch[0], '');

    // Milestone celebrations
    if (profile.total_workouts === 10) {
      finalResponse += '\n\n10 workouts completed! You\'re building a real habit!';
    } else if (profile.total_workouts === 50) {
      finalResponse += '\n\n50 workouts! You\'re a machine! Your consistency is impressive.';
    } else if (profile.total_workouts === 100) {
      finalResponse += '\n\n100 WORKOUTS! This is a massive achievement. You should be incredibly proud.';
    }

    if (profile.current_streak >= 7 && profile.current_streak % 7 === 0) {
      finalResponse += `\n\n${profile.current_streak}-day streak! Keep it going!`;
    }
  }

  // Handle difficulty adjustment
  const diffMatch = aiResponse.match(/\[DIFFICULTY_ADJUST:(UP|DOWN)\]/);
  if (diffMatch) {
    if (diffMatch[1] === 'UP' && profile.difficulty_modifier < 2) {
      profile.difficulty_modifier++;
    } else if (diffMatch[1] === 'DOWN' && profile.difficulty_modifier > -2) {
      profile.difficulty_modifier--;
    }
    finalResponse = finalResponse.replace(diffMatch[0], '');
  }

  // Handle profile updates
  const profileMatches = aiResponse.matchAll(/\[PROFILE_UPDATE:(\w+):([^\]]+)\]/g);
  for (const match of profileMatches) {
    const field = match[1];
    const value = match[2];

    if (['fitness_level', 'weekly_target'].includes(field)) {
      profile[field] = field === 'weekly_target' ? parseInt(value) : value;
    } else if (field === 'goals') {
      profile.goals = value.split(',').map(g => g.trim());
    } else if (field === 'equipment') {
      profile.equipment = value.split(',').map(e => e.trim());
    } else if (field === 'rest_days') {
      profile.rest_days = value.split(',').map(d => d.trim());
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Handle onboarding flag
  if (aiResponse.includes('[ONBOARD]')) {
    finalResponse = finalResponse.replace('[ONBOARD]', '');
    // Check if we have enough info to mark as onboarded
    if (profile.goals.length > 0 && profile.fitness_level !== 'beginner') {
      profile.onboarded = true;
    }
  }

  // Handle workout generated tag
  if (aiResponse.includes('[WORKOUT_GENERATED]')) {
    finalResponse = finalResponse.replace('[WORKOUT_GENERATED]', '');
    profile.last_workout_sent = new Date().toISOString();
  }

  // Handle injury alert
  if (aiResponse.includes('[INJURY_ALERT]')) {
    finalResponse = finalResponse.replace('[INJURY_ALERT]', '');
    finalResponse += '\n\nI\'ve flagged this for your trainer. They\'ll reach out to you directly.';
  }

  // Save updated metadata
  profile.last_interaction = new Date().toISOString();
  await updateConversationMetadata(phone, profile);

  // Clean remaining tags
  finalResponse = finalResponse.replace(/\[\w+(?::\w+)*\]/g, '').trim();

  return finalResponse;
}

// -- Weekly progress reports (run via cron every Monday at 8 AM) --
async function sendWeeklyReports() {
  console.log(`[${new Date().toISOString()}] Sending weekly progress reports...`);
  const conversations = await listConversations();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.onboarded) continue;

    const phone = conv.phone || conv.jid;
    const profile = initializeProfile(metadata);
    const stats = getWeeklyStats(profile.workout_history);

    const targetMet = stats.workouts_this_week >= profile.weekly_target;
    const message = [
      `*Weekly Progress Report*`,
      ``,
      `Workouts: ${stats.workouts_this_week}/${profile.weekly_target} ${targetMet ? '(target met!)' : ''}`,
      `Training time: ${stats.total_duration_min} minutes`,
      `Exercises completed: ${stats.exercises_completed}`,
      `Current streak: ${profile.current_streak} days`,
      `Total workouts: ${profile.total_workouts}`,
      ``,
      targetMet
        ? `Great week! You hit your target. Keep this momentum going!`
        : `You were ${profile.weekly_target - stats.workouts_this_week} workout(s) short of your target. Every workout counts -- let's crush it this week!`,
      ``,
      `Reply "workout" for today's plan.`,
    ].join('\n');

    await sendText(phone, message, {
      agent: 'fitness-coach',
      type: 'weekly-report',
      workouts_this_week: stats.workouts_this_week,
      target_met: targetMet,
    });
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

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    await sendTyping(phone);

    const metadata = await getConversationMetadata(phone);
    const aiResponse = await generateReply(phone, messageText, metadata);
    const finalResponse = await processActions(phone, aiResponse, metadata);

    const typingDelay = Math.min(finalResponse.length * 25, 3000);
    await new Promise(resolve => setTimeout(resolve, typingDelay));

    await sendText(phone, finalResponse, {
      agent: 'fitness-coach',
      model: 'claude-sonnet-4-20250514',
      total_workouts: metadata.total_workouts || 0,
      current_streak: metadata.current_streak || 0,
      generated_at: new Date().toISOString(),
    });
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Sorry, something went wrong. Try again in a moment!', {
        agent: 'fitness-coach',
        error: true,
        error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback message also failed:', fallbackErr.message);
    }
  }
});

// -- Admin: trigger weekly report --
app.post('/admin/weekly-report', async (req, res) => {
  try {
    await sendWeeklyReports();
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Fitness coach bot listening on port ${PORT}`);
});
```

## Metadata Schema

Each client conversation stores their fitness profile and history:

```json
{
  "fitness_level": "intermediate",
  "goals": ["muscle gain", "endurance"],
  "equipment": ["dumbbells", "resistance bands", "pull-up bar"],
  "workout_history": [
    {
      "date": "2026-04-01",
      "duration_min": 45,
      "exercises_completed": 8,
      "logged_at": "2026-04-01T18:30:00.000Z"
    }
  ],
  "current_streak": 5,
  "longest_streak": 12,
  "total_workouts": 34,
  "difficulty_modifier": 1,
  "weekly_target": 4,
  "rest_days": ["Sunday"],
  "onboarded": true,
  "joined_at": "2026-02-15T10:00:00.000Z",
  "last_workout_sent": "2026-04-02T08:00:00.000Z",
  "last_interaction": "2026-04-02T18:30:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "fitness-coach",
  "model": "claude-sonnet-4-20250514",
  "total_workouts": 34,
  "current_streak": 5,
  "generated_at": "2026-04-02T18:30:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir fitness-coach && cd fitness-coach
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

# 6. Trigger weekly report manually
curl -X POST http://localhost:3000/admin/weekly-report
```
