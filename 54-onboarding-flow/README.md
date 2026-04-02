# 54 - User Onboarding Flow

Multi-step guided onboarding via WhatsApp for new users. Collects business info, preferences, and configures accounts through progressive disclosure -- asking one thing at a time. Tracks onboarding progress in metadata with a welcome message sequence.

## Problem

A SaaS platform signs up 50 new users per month, but only 30% complete their account setup. The onboarding form has 15 fields across 4 pages, and users abandon when they see the wall of questions. Users who don't complete onboarding within 48h have a 70% chance of never coming back. Email onboarding sequences have a 20% open rate.

## Solution

Deploy a WhatsApp onboarding flow that:
- Guides new users through setup one question at a time (progressive disclosure)
- Collects business info, preferences, and configuration in a conversational format
- Tracks onboarding progress with completion percentage in metadata
- Sends welcome message sequences at timed intervals
- Uses AI to validate and enrich responses (e.g., infer industry from business name)
- Allows users to pause and resume anytime
- Celebrates milestones and provides value at each step

## Architecture

```
New user signs up (website/API)
         |
         v
+--------------------+     trigger onboarding    +--------------------+
|  Your App Backend  | -------------------------> |  Your Express App  |
|                    |                            |   (this code)      |
+--------------------+                            +--------------------+
                                                           |
                                                           v
                                                  +------------------+
                                                  |   Opero WPP API  |
                                                  | wpp-api.opero.so |
                                                  +------------------+
                                                           |
                                            step-by-step   |
                                            questions      v
                                                  +------------------+
                                                  |    New User      |
                                                  |   (WhatsApp)     |
                                                  +------------------+
                                                           |
                                              responses    |
                                                           v
                                                  +------------------+
                                                  |  Claude Sonnet   |
                                                  | (validate/enrich)|
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

// -- Onboarding steps definition --
const ONBOARDING_STEPS = [
  {
    id: 'welcome',
    type: 'message', // Just sends a message, no input expected
    message: (name) => [
      `Welcome to Opero, ${name}!`,
      ``,
      `I'll help you set up your account in just a few minutes. I'll ask you one question at a time -- no rush.`,
      ``,
      `You can pause anytime and come back later. Just reply *CONTINUE* to pick up where you left off.`,
      ``,
      `Let's get started!`,
    ].join('\n'),
    delay_ms: 0,
  },
  {
    id: 'business_name',
    type: 'text',
    question: 'What\'s the name of your business or company?',
    field: 'business_name',
    validation: (val) => val.length >= 2,
    error_message: 'Please enter your business name (at least 2 characters).',
  },
  {
    id: 'industry',
    type: 'choice',
    question: [
      'What industry are you in?',
      '',
      '1. Retail / E-commerce',
      '2. Healthcare',
      '3. Real Estate',
      '4. Professional Services',
      '5. Hospitality / Food',
      '6. Education',
      '7. Technology / SaaS',
      '8. Other',
      '',
      'Reply with a number (1-8).',
    ].join('\n'),
    field: 'industry',
    options: {
      '1': 'Retail / E-commerce',
      '2': 'Healthcare',
      '3': 'Real Estate',
      '4': 'Professional Services',
      '5': 'Hospitality / Food',
      '6': 'Education',
      '7': 'Technology / SaaS',
      '8': 'Other',
    },
    validation: (val) => ['1', '2', '3', '4', '5', '6', '7', '8'].includes(val.trim()),
    error_message: 'Please reply with a number from 1 to 8.',
  },
  {
    id: 'team_size',
    type: 'choice',
    question: [
      'How many people are on your team?',
      '',
      '1. Just me',
      '2. 2-5 people',
      '3. 6-20 people',
      '4. 21-50 people',
      '5. 50+ people',
      '',
      'Reply with a number (1-5).',
    ].join('\n'),
    field: 'team_size',
    options: {
      '1': 'Solo',
      '2': '2-5',
      '3': '6-20',
      '4': '21-50',
      '5': '50+',
    },
    validation: (val) => ['1', '2', '3', '4', '5'].includes(val.trim()),
    error_message: 'Please reply with a number from 1 to 5.',
  },
  {
    id: 'primary_goal',
    type: 'choice',
    question: [
      'What\'s your primary goal with Opero?',
      '',
      '1. Automate customer support',
      '2. Generate and qualify leads',
      '3. Send notifications & reminders',
      '4. Run outreach campaigns',
      '5. All of the above',
      '',
      'Reply with a number (1-5).',
    ].join('\n'),
    field: 'primary_goal',
    options: {
      '1': 'Customer support',
      '2': 'Lead generation',
      '3': 'Notifications',
      '4': 'Outreach campaigns',
      '5': 'All of the above',
    },
    validation: (val) => ['1', '2', '3', '4', '5'].includes(val.trim()),
    error_message: 'Please reply with a number from 1 to 5.',
  },
  {
    id: 'monthly_messages',
    type: 'choice',
    question: [
      'How many WhatsApp messages do you estimate sending per month?',
      '',
      '1. Less than 500',
      '2. 500 - 2,000',
      '3. 2,000 - 10,000',
      '4. 10,000+',
      '',
      'Reply with a number (1-4).',
    ].join('\n'),
    field: 'monthly_messages',
    options: {
      '1': 'Under 500',
      '2': '500-2K',
      '3': '2K-10K',
      '4': '10K+',
    },
    validation: (val) => ['1', '2', '3', '4'].includes(val.trim()),
    error_message: 'Please reply with a number from 1 to 4.',
  },
  {
    id: 'timezone',
    type: 'text',
    question: 'What timezone are you in? (e.g., "Buenos Aires", "New York", "London", "UTC-3")',
    field: 'timezone',
    validation: (val) => val.length >= 2,
    error_message: 'Please enter your timezone or city.',
  },
  {
    id: 'ai_preference',
    type: 'choice',
    question: [
      'Which AI features interest you most?',
      '',
      '1. Auto-replies to customer messages',
      '2. Lead qualification & scoring',
      '3. Sentiment analysis',
      '4. Content generation (messages, campaigns)',
      '5. Not sure yet -- show me everything!',
      '',
      'Reply with a number (1-5).',
    ].join('\n'),
    field: 'ai_preference',
    options: {
      '1': 'Auto-replies',
      '2': 'Lead qualification',
      '3': 'Sentiment analysis',
      '4': 'Content generation',
      '5': 'Show me everything',
    },
    validation: (val) => ['1', '2', '3', '4', '5'].includes(val.trim()),
    error_message: 'Please reply with a number from 1 to 5.',
  },
  {
    id: 'complete',
    type: 'message',
    message: (name, profile) => [
      `You're all set, ${name}!`,
      ``,
      `Here's your profile:`,
      `- Business: ${profile.business_name}`,
      `- Industry: ${profile.industry}`,
      `- Team: ${profile.team_size}`,
      `- Goal: ${profile.primary_goal}`,
      `- Volume: ${profile.monthly_messages} msgs/month`,
      `- Timezone: ${profile.timezone}`,
      `- AI interest: ${profile.ai_preference}`,
      ``,
      `Your account is now configured! Here's what you can do next:`,
      ``,
      `1. Connect your WhatsApp number in the dashboard`,
      `2. Set up your first automation`,
      `3. Import your contacts`,
      ``,
      `Need help? Reply *HELP* anytime. Welcome aboard!`,
    ].join('\n'),
    delay_ms: 0,
  },
];

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

// -- AI enrichment (optional) --
async function enrichBusinessInfo(businessName, industry) {
  try {
    const response = await anthropic.messages.create({
      model: 'claude-haiku-4-20250414',
      max_tokens: 150,
      messages: [{
        role: 'user',
        content: `Business: "${businessName}", Industry: "${industry}". Suggest: 1) A one-line business description, 2) Top use case for WhatsApp automation. Respond as JSON: {"description": "...", "top_use_case": "..."}`,
      }],
    });
    const text = response.content[0].text;
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    return jsonMatch ? JSON.parse(jsonMatch[0]) : null;
  } catch {
    return null;
  }
}

// -- Step processing --
async function processStep(phone, metadata) {
  const stepIndex = metadata.current_step;
  const step = ONBOARDING_STEPS[stepIndex];

  if (!step) {
    // Onboarding complete
    metadata.onboarding_status = 'complete';
    metadata.completed_at = new Date().toISOString();
    await updateConversationMetadata(phone, metadata);
    return;
  }

  if (step.type === 'message') {
    // Send message and advance to next step
    const message = typeof step.message === 'function'
      ? step.message(metadata.user_name, metadata.profile || {})
      : step.message;

    if (step.delay_ms > 0) {
      await new Promise(resolve => setTimeout(resolve, step.delay_ms));
    }

    await sendText(phone, message, {
      agent: 'onboarding-flow',
      type: 'step-message',
      step: step.id,
      step_index: stepIndex,
    });

    // Auto-advance to next step
    metadata.current_step = stepIndex + 1;
    await updateConversationMetadata(phone, metadata);

    // If next step is also a message, process it too (with delay)
    const nextStep = ONBOARDING_STEPS[metadata.current_step];
    if (nextStep && nextStep.type === 'message') {
      await new Promise(resolve => setTimeout(resolve, 1500));
      await processStep(phone, metadata);
    } else if (nextStep) {
      // Send the question for the next input step
      await new Promise(resolve => setTimeout(resolve, 1000));
      await sendText(phone, nextStep.question, {
        agent: 'onboarding-flow',
        type: 'step-question',
        step: nextStep.id,
        step_index: metadata.current_step,
      });
    }
  }
}

// -- Handle user response --
async function handleResponse(phone, response, metadata) {
  const stepIndex = metadata.current_step;
  const step = ONBOARDING_STEPS[stepIndex];

  if (!step || step.type === 'message') return;

  // Validate response
  if (step.validation && !step.validation(response)) {
    await sendText(phone, step.error_message, {
      agent: 'onboarding-flow',
      type: 'validation-error',
      step: step.id,
    });
    return;
  }

  // Store the response
  if (!metadata.profile) metadata.profile = {};

  if (step.type === 'choice' && step.options) {
    const key = response.trim();
    metadata.profile[step.field] = step.options[key] || response;
  } else {
    metadata.profile[step.field] = response.trim();
  }

  metadata.steps_completed = (metadata.steps_completed || 0) + 1;

  // Calculate progress
  const inputSteps = ONBOARDING_STEPS.filter(s => s.type !== 'message');
  metadata.progress_pct = Math.round((metadata.steps_completed / inputSteps.length) * 100);

  // Advance to next step
  metadata.current_step = stepIndex + 1;
  metadata.last_interaction = new Date().toISOString();
  await updateConversationMetadata(phone, metadata);

  // AI enrichment after business name + industry are collected
  if (step.id === 'industry' && metadata.profile.business_name) {
    const enrichment = await enrichBusinessInfo(metadata.profile.business_name, metadata.profile.industry);
    if (enrichment) {
      metadata.profile.ai_description = enrichment.description;
      metadata.profile.ai_top_use_case = enrichment.top_use_case;
      await updateConversationMetadata(phone, metadata);
    }
  }

  // Send milestone celebration at 50%
  if (metadata.progress_pct >= 50 && metadata.progress_pct - (100 / inputSteps.length) < 50) {
    await sendText(phone, `Halfway there! You're doing great.`, {
      agent: 'onboarding-flow', type: 'milestone', progress_pct: metadata.progress_pct,
    });
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  // Process next step
  await processStep(phone, metadata);
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim();
  if (!messageText) return;

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    // Not in onboarding
    if (metadata.onboarding_status !== 'in_progress') {
      if (metadata.onboarding_status === 'complete') {
        await sendText(phone, 'Your onboarding is complete! Head to the dashboard to explore features, or reply *HELP* for assistance.', {
          agent: 'onboarding-flow', type: 'already-complete',
        });
      }
      return;
    }

    const upper = messageText.toUpperCase();

    // CONTINUE command
    if (upper === 'CONTINUE') {
      const step = ONBOARDING_STEPS[metadata.current_step];
      if (step && step.type !== 'message') {
        await sendText(phone, `Let's continue! Here's where we left off:\n\n${step.question}`, {
          agent: 'onboarding-flow', type: 'resumed', step: step.id,
        });
      } else {
        await processStep(phone, metadata);
      }
      return;
    }

    // STATUS command
    if (upper === 'STATUS') {
      const inputSteps = ONBOARDING_STEPS.filter(s => s.type !== 'message');
      const completed = metadata.steps_completed || 0;

      await sendText(phone, [
        `*Onboarding Progress: ${metadata.progress_pct || 0}%*`,
        `Steps completed: ${completed}/${inputSteps.length}`,
        ``,
        ...inputSteps.map((s, i) => {
          const done = i < completed;
          return `${done ? '[x]' : '[ ]'} ${s.field?.replace(/_/g, ' ') || s.id}`;
        }),
      ].join('\n'), {
        agent: 'onboarding-flow', type: 'status', progress_pct: metadata.progress_pct || 0,
      });
      return;
    }

    // SKIP command (skip optional steps)
    if (upper === 'SKIP') {
      const step = ONBOARDING_STEPS[metadata.current_step];
      if (step && step.type !== 'message') {
        metadata.current_step++;
        metadata.last_interaction = new Date().toISOString();
        await updateConversationMetadata(phone, metadata);
        await processStep(phone, metadata);
      }
      return;
    }

    // Process the response for the current step
    await sendTyping(phone);
    await handleResponse(phone, messageText, metadata);
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Oops, something went wrong. Reply *CONTINUE* to pick up where you left off.', {
        agent: 'onboarding-flow', error: true, error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback failed:', fallbackErr.message);
    }
  }
});

// -- Admin: start onboarding for a new user --
app.post('/admin/onboard', async (req, res) => {
  try {
    const { phone, name, email, user_id } = req.body;

    const metadata = {
      role: 'user',
      user_id: user_id || null,
      user_name: name,
      email: email || null,
      onboarding_status: 'in_progress',
      current_step: 0,
      steps_completed: 0,
      progress_pct: 0,
      profile: {},
      started_at: new Date().toISOString(),
    };

    await updateConversationMetadata(phone, metadata);

    // Start the onboarding flow
    await processStep(phone, metadata);

    res.json({ success: true, phone });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: get onboarding stats --
app.get('/admin/onboarding-stats', async (req, res) => {
  try {
    const conversations = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=500`);
    const users = (conversations.data || []).filter(c => c.metadata?.role === 'user');

    const stats = {
      total: users.length,
      in_progress: users.filter(u => u.metadata.onboarding_status === 'in_progress').length,
      complete: users.filter(u => u.metadata.onboarding_status === 'complete').length,
      avg_progress: users.length > 0
        ? Math.round(users.reduce((sum, u) => sum + (u.metadata.progress_pct || 0), 0) / users.length)
        : 0,
      drop_off_steps: {},
    };

    // Calculate drop-off per step
    for (const user of users) {
      if (user.metadata.onboarding_status === 'in_progress') {
        const step = ONBOARDING_STEPS[user.metadata.current_step];
        const stepName = step?.id || 'unknown';
        stats.drop_off_steps[stepName] = (stats.drop_off_steps[stepName] || 0) + 1;
      }
    }

    res.json(stats);
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
  console.log(`Onboarding flow bot listening on port ${PORT}`);
});
```

## Metadata Schema

Each user conversation stores onboarding progress:

```json
{
  "role": "user",
  "user_id": "usr_abc123",
  "user_name": "Diego Ramirez",
  "email": "diego@techco.com",
  "onboarding_status": "in_progress",
  "current_step": 4,
  "steps_completed": 3,
  "progress_pct": 43,
  "profile": {
    "business_name": "TechCo Argentina",
    "industry": "Technology / SaaS",
    "team_size": "6-20",
    "ai_description": "Argentine SaaS company providing tech solutions",
    "ai_top_use_case": "Automated lead qualification and follow-up"
  },
  "started_at": "2026-04-02T10:00:00.000Z",
  "last_interaction": "2026-04-02T10:05:00.000Z"
}
```

Completed onboarding profile:

```json
{
  "onboarding_status": "complete",
  "progress_pct": 100,
  "steps_completed": 7,
  "profile": {
    "business_name": "TechCo Argentina",
    "industry": "Technology / SaaS",
    "team_size": "6-20",
    "primary_goal": "All of the above",
    "monthly_messages": "2K-10K",
    "timezone": "Buenos Aires (UTC-3)",
    "ai_preference": "Lead qualification",
    "ai_description": "Argentine SaaS company providing tech solutions",
    "ai_top_use_case": "Automated lead qualification and follow-up"
  },
  "started_at": "2026-04-02T10:00:00.000Z",
  "completed_at": "2026-04-02T10:12:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir onboarding-flow && cd onboarding-flow
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

# 6. Start onboarding for a new user
curl -X POST http://localhost:3000/admin/onboard \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "name": "Diego Ramirez",
    "email": "diego@techco.com",
    "user_id": "usr_abc123"
  }'

# 7. Check onboarding stats
curl http://localhost:3000/admin/onboarding-stats
```
