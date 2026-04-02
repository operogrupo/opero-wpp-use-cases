# 55 - Churn Prevention Agent

AI-powered churn prevention system via WhatsApp. Monitors conversation patterns for decreasing engagement, negative sentiment, and complaints. Proactively reaches out to at-risk users with personalized retention offers. Tracks churn risk score in metadata.

## Problem

A SaaS company with 2,000 active users has a 5% monthly churn rate -- losing 100 users per month at $80/month average, that's $96,000 in annual recurring revenue lost every month. By the time the support team notices a user is disengaging, it's too late. 60% of churned users showed warning signs (fewer logins, support complaints, negative feedback) weeks before canceling, but no one was monitoring for them.

## Solution

Deploy a churn prevention agent that:
- Calculates a churn risk score (0-100) based on engagement signals
- Monitors conversation sentiment, complaint frequency, and interaction patterns
- Proactively reaches out to at-risk users before they churn
- Uses Claude Sonnet to craft personalized retention messages based on the user's history
- Offers tailored retention incentives (discounts, feature unlocks, personal onboarding)
- Tracks all intervention attempts and outcomes in metadata
- Provides a real-time churn risk dashboard via API

## Architecture

```
+-------------------+     engagement data      +--------------------+
|  Your App Backend | -----------------------> |  Your Express App  |
| (events, logins,  |                         |   (this code)      |
|  usage metrics)   |                         +--------------------+
+-------------------+                                   |
                                                        v
                                               +------------------+
                                               |  Risk Scoring    |
                                               |  Engine          |
                                               +------------------+
                                                        |
                                         if at-risk     v
                                               +------------------+
                                               |  Claude Sonnet   |
                                               | (personalized    |
                                               |  retention msg)  |
                                               +------------------+
                                                        |
                                                        v
                                               +------------------+
                                               |   Opero WPP API  |
                                               | wpp-api.opero.so |
                                               +------------------+
                                                        |
                                    retention message   |
                                                        v
                                               +------------------+
                                               |   At-Risk User   |
                                               |   (WhatsApp)     |
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
const CSM_PHONE = process.env.CSM_PHONE; // Customer success manager

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// -- Churn risk thresholds --
const RISK_THRESHOLDS = {
  low: 30,       // 0-29: healthy
  medium: 60,    // 30-59: monitor
  high: 80,      // 60-79: at risk
  critical: 100, // 80-100: likely to churn
};

// -- Retention offers --
const RETENTION_OFFERS = {
  low_engagement: {
    type: 'feature_highlight',
    message: 'personalized feature recommendations based on their use case',
  },
  price_sensitive: {
    type: 'discount',
    discount_pct: 25,
    duration: '3 months',
    message: '25% discount for 3 months',
  },
  frustrated: {
    type: 'personal_support',
    message: 'dedicated personal onboarding session',
  },
  competitor: {
    type: 'comparison',
    message: 'feature comparison showing our advantages',
  },
  downgrade: {
    type: 'plan_adjustment',
    message: 'suggest a more suitable plan instead of canceling',
  },
};

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

async function getMessages(phone, limit = 30) {
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

// -- Churn risk calculation --
function calculateChurnRisk(metadata) {
  let risk = 0;
  const now = Date.now();

  // Factor 1: Days since last login (0-25 points)
  if (metadata.last_login) {
    const daysSinceLogin = (now - new Date(metadata.last_login).getTime()) / (1000 * 60 * 60 * 24);
    if (daysSinceLogin > 30) risk += 25;
    else if (daysSinceLogin > 14) risk += 15;
    else if (daysSinceLogin > 7) risk += 8;
  }

  // Factor 2: Login frequency decline (0-20 points)
  if (metadata.login_history?.length >= 2) {
    const recentMonth = metadata.login_history.filter(
      l => new Date(l).getTime() > now - 30 * 24 * 60 * 60 * 1000
    ).length;
    const prevMonth = metadata.login_history.filter(
      l => {
        const t = new Date(l).getTime();
        return t > now - 60 * 24 * 60 * 60 * 1000 && t <= now - 30 * 24 * 60 * 60 * 1000;
      }
    ).length;

    if (prevMonth > 0) {
      const decline = (prevMonth - recentMonth) / prevMonth;
      if (decline > 0.7) risk += 20;
      else if (decline > 0.5) risk += 15;
      else if (decline > 0.3) risk += 8;
    }
  }

  // Factor 3: Support tickets / complaints (0-20 points)
  const recentComplaints = (metadata.complaints || []).filter(
    c => new Date(c.date).getTime() > now - 30 * 24 * 60 * 60 * 1000
  );
  if (recentComplaints.length >= 3) risk += 20;
  else if (recentComplaints.length >= 2) risk += 12;
  else if (recentComplaints.length >= 1) risk += 5;

  // Factor 4: Negative sentiment in messages (0-15 points)
  if (metadata.sentiment_scores?.length > 0) {
    const avgSentiment = metadata.sentiment_scores.reduce((a, b) => a + b, 0) / metadata.sentiment_scores.length;
    if (avgSentiment < -0.5) risk += 15;
    else if (avgSentiment < -0.2) risk += 8;
    else if (avgSentiment < 0) risk += 3;
  }

  // Factor 5: Feature usage decline (0-10 points)
  if (metadata.feature_usage) {
    const recent = metadata.feature_usage.recent || 0;
    const previous = metadata.feature_usage.previous || 0;
    if (previous > 0) {
      const decline = (previous - recent) / previous;
      if (decline > 0.6) risk += 10;
      else if (decline > 0.3) risk += 5;
    }
  }

  // Factor 6: NPS score (0-10 points)
  if (metadata.latest_nps_score !== undefined) {
    if (metadata.latest_nps_score <= 4) risk += 10;
    else if (metadata.latest_nps_score <= 6) risk += 5;
  }

  // Negative factor: recent positive interactions reduce risk
  if (metadata.last_positive_interaction) {
    const daysSince = (now - new Date(metadata.last_positive_interaction).getTime()) / (1000 * 60 * 60 * 24);
    if (daysSince < 7) risk -= 10;
  }

  return Math.max(0, Math.min(100, risk));
}

function getRiskLevel(score) {
  if (score >= RISK_THRESHOLDS.high) return 'critical';
  if (score >= RISK_THRESHOLDS.medium) return 'high';
  if (score >= RISK_THRESHOLDS.low) return 'medium';
  return 'low';
}

function identifyChurnReason(metadata) {
  const reasons = [];

  if (metadata.complaints?.length > 0) {
    const lastComplaint = metadata.complaints[metadata.complaints.length - 1];
    reasons.push({ type: 'frustrated', detail: lastComplaint.topic || 'general complaint' });
  }

  if (metadata.last_login) {
    const daysSince = (Date.now() - new Date(metadata.last_login).getTime()) / (1000 * 60 * 60 * 24);
    if (daysSince > 14) reasons.push({ type: 'low_engagement', detail: `${Math.round(daysSince)} days since last login` });
  }

  if (metadata.competitors_mentioned?.length > 0) {
    reasons.push({ type: 'competitor', detail: metadata.competitors_mentioned.join(', ') });
  }

  if (metadata.price_objections > 0) {
    reasons.push({ type: 'price_sensitive', detail: `${metadata.price_objections} price-related objections` });
  }

  if (metadata.downgrade_requested) {
    reasons.push({ type: 'downgrade', detail: 'Requested plan downgrade' });
  }

  return reasons.length > 0 ? reasons : [{ type: 'low_engagement', detail: 'General engagement decline' }];
}

// -- AI retention message generation --
async function generateRetentionMessage(metadata) {
  const reasons = identifyChurnReason(metadata);
  const primaryReason = reasons[0];
  const offer = RETENTION_OFFERS[primaryReason.type] || RETENTION_OFFERS.low_engagement;

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 400,
    system: `You are a customer success agent reaching out to an at-risk user via WhatsApp. Your goal is to re-engage them and prevent churn.

User profile:
- Name: ${metadata.user_name || 'there'}
- Company: ${metadata.company || 'their business'}
- Plan: ${metadata.plan || 'Standard'}
- Member since: ${metadata.joined_at || 'unknown'}
- Primary use case: ${metadata.primary_use_case || 'unknown'}
- Churn risk: ${metadata.churn_risk_score}/100 (${getRiskLevel(metadata.churn_risk_score)})
- Likely reason: ${primaryReason.type} (${primaryReason.detail})

Available retention offer: ${offer.message}

Rules:
- Be genuine and empathetic, not salesy
- Acknowledge the specific issue (don't pretend everything is fine)
- Offer a concrete solution or benefit
- Keep it under 120 words
- Don't mention "churn" or "risk score" -- the user shouldn't know they're flagged
- Use a warm, personal tone
- End with a clear call to action or question`,
    messages: [{
      role: 'user',
      content: `Write a WhatsApp retention message for this user. Reason: ${primaryReason.type} - ${primaryReason.detail}`,
    }],
  });

  return {
    message: response.content[0].text,
    reason: primaryReason,
    offer: offer,
  };
}

// -- Sentiment analysis for incoming messages --
async function analyzeSentiment(text) {
  try {
    const response = await anthropic.messages.create({
      model: 'claude-haiku-4-20250414',
      max_tokens: 50,
      messages: [{
        role: 'user',
        content: `Rate the sentiment of this message from -1.0 (very negative) to 1.0 (very positive). Respond with only the number.\n\n"${text}"`,
      }],
    });
    const score = parseFloat(response.content[0].text.trim());
    return isNaN(score) ? 0 : Math.max(-1, Math.min(1, score));
  } catch {
    return 0;
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

    if (metadata.role !== 'user') return;

    // Analyze sentiment
    const sentiment = await analyzeSentiment(messageText);
    if (!metadata.sentiment_scores) metadata.sentiment_scores = [];
    metadata.sentiment_scores.push(sentiment);
    // Keep last 20 sentiment scores
    if (metadata.sentiment_scores.length > 20) {
      metadata.sentiment_scores = metadata.sentiment_scores.slice(-20);
    }

    // Detect complaints
    if (sentiment < -0.3) {
      if (!metadata.complaints) metadata.complaints = [];
      metadata.complaints.push({
        text: messageText.substring(0, 200),
        sentiment,
        date: new Date().toISOString(),
      });
    }

    // Detect competitor mentions
    const competitorKeywords = ['competitor', 'alternative', 'switch to', 'moving to', 'trying out'];
    const mentionsCompetitor = competitorKeywords.some(kw =>
      messageText.toLowerCase().includes(kw)
    );
    if (mentionsCompetitor) {
      if (!metadata.competitors_mentioned) metadata.competitors_mentioned = [];
      metadata.competitors_mentioned.push(messageText.substring(0, 100));
    }

    // Detect price objections
    const priceKeywords = ['expensive', 'too much', 'cheaper', 'cost', 'price', 'can\'t afford', 'budget'];
    if (priceKeywords.some(kw => messageText.toLowerCase().includes(kw)) && sentiment < 0) {
      metadata.price_objections = (metadata.price_objections || 0) + 1;
    }

    // Detect cancel intent
    const cancelKeywords = ['cancel', 'unsubscribe', 'close my account', 'delete my account', 'stop'];
    const wantsToCancel = cancelKeywords.some(kw => messageText.toLowerCase().includes(kw));

    if (wantsToCancel) {
      metadata.cancel_intent_detected = true;
      metadata.cancel_intent_at = new Date().toISOString();
    }

    // Recalculate churn risk
    metadata.churn_risk_score = calculateChurnRisk(metadata);
    metadata.churn_risk_level = getRiskLevel(metadata.churn_risk_score);
    metadata.last_interaction = new Date().toISOString();

    await updateConversationMetadata(phone, metadata);

    // If cancel intent detected, respond immediately with retention
    if (wantsToCancel) {
      await sendTyping(phone);
      const retention = await generateRetentionMessage(metadata);

      if (!metadata.retention_attempts) metadata.retention_attempts = [];
      metadata.retention_attempts.push({
        reason: retention.reason.type,
        offer: retention.offer.type,
        sent_at: new Date().toISOString(),
        triggered_by: 'cancel_intent',
      });

      await updateConversationMetadata(phone, metadata);

      const typingDelay = Math.min(retention.message.length * 30, 3000);
      await new Promise(resolve => setTimeout(resolve, typingDelay));

      await sendText(phone, retention.message, {
        agent: 'churn-prevention',
        type: 'retention-message',
        risk_score: metadata.churn_risk_score,
        reason: retention.reason.type,
        offer: retention.offer.type,
        triggered_by: 'cancel_intent',
      });

      // Alert CSM
      await sendText(CSM_PHONE, [
        `*CANCEL INTENT DETECTED*`,
        ``,
        `User: ${metadata.user_name} (${phone})`,
        `Company: ${metadata.company || 'N/A'}`,
        `Plan: ${metadata.plan || 'N/A'}`,
        `Risk score: ${metadata.churn_risk_score}/100`,
        `Reason: ${retention.reason.detail}`,
        ``,
        `Message: "${messageText.substring(0, 200)}"`,
        ``,
        `Retention offer sent: ${retention.offer.message}`,
        `Please follow up personally.`,
      ].join('\n'), {
        agent: 'churn-prevention', type: 'csm-alert', user_phone: phone,
      });
    }

    console.log(`[${new Date().toISOString()}] ${phone} - sentiment: ${sentiment.toFixed(2)}, risk: ${metadata.churn_risk_score}`);
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- Proactive outreach cron (run daily) --
async function proactiveOutreach() {
  console.log(`[${new Date().toISOString()}] Running proactive churn prevention...`);
  const conversations = await listConversations();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.role || metadata.role !== 'user') continue;

    const phone = conv.phone || conv.jid;

    // Recalculate risk
    metadata.churn_risk_score = calculateChurnRisk(metadata);
    metadata.churn_risk_level = getRiskLevel(metadata.churn_risk_score);

    // Only proactively reach out to high/critical risk users
    if (metadata.churn_risk_score < RISK_THRESHOLDS.medium) {
      await updateConversationMetadata(phone, metadata);
      continue;
    }

    // Don't reach out more than once per week
    const lastAttempt = metadata.retention_attempts?.slice(-1)[0];
    if (lastAttempt) {
      const daysSinceAttempt = (Date.now() - new Date(lastAttempt.sent_at).getTime()) / (1000 * 60 * 60 * 24);
      if (daysSinceAttempt < 7) {
        await updateConversationMetadata(phone, metadata);
        continue;
      }
    }

    // Max 3 retention attempts
    if ((metadata.retention_attempts?.length || 0) >= 3) {
      await updateConversationMetadata(phone, metadata);
      continue;
    }

    // Generate and send retention message
    const retention = await generateRetentionMessage(metadata);

    if (!metadata.retention_attempts) metadata.retention_attempts = [];
    metadata.retention_attempts.push({
      reason: retention.reason.type,
      offer: retention.offer.type,
      sent_at: new Date().toISOString(),
      triggered_by: 'proactive',
      risk_score_at_time: metadata.churn_risk_score,
    });

    await updateConversationMetadata(phone, metadata);

    await sendText(phone, retention.message, {
      agent: 'churn-prevention',
      type: 'proactive-retention',
      risk_score: metadata.churn_risk_score,
      reason: retention.reason.type,
      offer: retention.offer.type,
    });

    console.log(`Proactive outreach to ${phone} (risk: ${metadata.churn_risk_score}, reason: ${retention.reason.type})`);

    // Stagger messages
    await new Promise(resolve => setTimeout(resolve, 2000));
  }
}

// -- API: ingest engagement events from your app --
app.post('/api/events', async (req, res) => {
  try {
    const { phone, event_type, data } = req.body;
    const metadata = await getConversationMetadata(phone);

    switch (event_type) {
      case 'login':
        metadata.last_login = new Date().toISOString();
        if (!metadata.login_history) metadata.login_history = [];
        metadata.login_history.push(new Date().toISOString());
        if (metadata.login_history.length > 60) {
          metadata.login_history = metadata.login_history.slice(-60);
        }
        break;

      case 'feature_usage':
        if (!metadata.feature_usage) metadata.feature_usage = { recent: 0, previous: 0 };
        metadata.feature_usage.recent = data?.count || 0;
        break;

      case 'nps':
        metadata.latest_nps_score = data?.score;
        break;

      case 'support_ticket':
        if (!metadata.complaints) metadata.complaints = [];
        metadata.complaints.push({
          topic: data?.topic || 'support ticket',
          date: new Date().toISOString(),
          sentiment: -0.3,
        });
        break;

      case 'positive_interaction':
        metadata.last_positive_interaction = new Date().toISOString();
        break;

      case 'downgrade_request':
        metadata.downgrade_requested = true;
        break;
    }

    // Recalculate risk
    metadata.churn_risk_score = calculateChurnRisk(metadata);
    metadata.churn_risk_level = getRiskLevel(metadata.churn_risk_score);

    await updateConversationMetadata(phone, metadata);
    res.json({ risk_score: metadata.churn_risk_score, risk_level: metadata.churn_risk_level });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- API: churn risk dashboard --
app.get('/api/churn-dashboard', async (req, res) => {
  try {
    const conversations = await listConversations();
    const users = (conversations.data || []).filter(c => c.metadata?.role === 'user');

    const dashboard = {
      total_users: users.length,
      risk_distribution: {
        low: users.filter(u => (u.metadata.churn_risk_score || 0) < RISK_THRESHOLDS.low).length,
        medium: users.filter(u => {
          const s = u.metadata.churn_risk_score || 0;
          return s >= RISK_THRESHOLDS.low && s < RISK_THRESHOLDS.medium;
        }).length,
        high: users.filter(u => {
          const s = u.metadata.churn_risk_score || 0;
          return s >= RISK_THRESHOLDS.medium && s < RISK_THRESHOLDS.high;
        }).length,
        critical: users.filter(u => (u.metadata.churn_risk_score || 0) >= RISK_THRESHOLDS.high).length,
      },
      at_risk_users: users
        .filter(u => (u.metadata.churn_risk_score || 0) >= RISK_THRESHOLDS.medium)
        .map(u => ({
          phone: u.phone || u.jid,
          name: u.metadata.user_name,
          company: u.metadata.company,
          risk_score: u.metadata.churn_risk_score,
          risk_level: u.metadata.churn_risk_level,
          retention_attempts: u.metadata.retention_attempts?.length || 0,
          last_interaction: u.metadata.last_interaction,
        }))
        .sort((a, b) => b.risk_score - a.risk_score),
      avg_risk_score: users.length > 0
        ? Math.round(users.reduce((sum, u) => sum + (u.metadata.churn_risk_score || 0), 0) / users.length)
        : 0,
    };

    res.json(dashboard);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: register a user for churn monitoring --
app.post('/admin/users', async (req, res) => {
  try {
    const { phone, name, company, plan, joined_at, primary_use_case } = req.body;
    const metadata = {
      role: 'user',
      user_name: name,
      company,
      plan: plan || 'standard',
      primary_use_case,
      joined_at: joined_at || new Date().toISOString(),
      churn_risk_score: 0,
      churn_risk_level: 'low',
      sentiment_scores: [],
      complaints: [],
      login_history: [new Date().toISOString()],
      last_login: new Date().toISOString(),
      retention_attempts: [],
      registered_at: new Date().toISOString(),
    };

    await updateConversationMetadata(phone, metadata);
    res.json({ success: true, metadata });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: trigger proactive outreach --
app.post('/admin/proactive-outreach', async (req, res) => {
  try {
    await proactiveOutreach();
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
  console.log(`Churn prevention agent listening on port ${PORT}`);
});
```

## Metadata Schema

Each user conversation stores churn signals and retention data:

```json
{
  "role": "user",
  "user_name": "Carolina Vega",
  "company": "Vega Consultores",
  "plan": "professional",
  "primary_use_case": "customer support automation",
  "joined_at": "2025-08-15T10:00:00.000Z",
  "churn_risk_score": 68,
  "churn_risk_level": "high",
  "last_login": "2026-03-20T14:00:00.000Z",
  "login_history": ["2026-03-20T14:00:00.000Z", "2026-03-15T09:00:00.000Z"],
  "feature_usage": { "recent": 12, "previous": 45 },
  "sentiment_scores": [-0.3, -0.5, 0.1, -0.4, -0.2],
  "latest_nps_score": 5,
  "complaints": [
    {
      "text": "The response times have been terrible lately",
      "sentiment": -0.6,
      "date": "2026-03-28T10:00:00.000Z"
    },
    {
      "topic": "support ticket",
      "date": "2026-04-01T11:00:00.000Z",
      "sentiment": -0.3
    }
  ],
  "competitors_mentioned": ["saw a demo of alternativeX"],
  "price_objections": 1,
  "cancel_intent_detected": false,
  "retention_attempts": [
    {
      "reason": "frustrated",
      "offer": "personal_support",
      "sent_at": "2026-04-01T09:00:00.000Z",
      "triggered_by": "proactive",
      "risk_score_at_time": 65
    }
  ],
  "last_interaction": "2026-04-02T14:30:00.000Z",
  "registered_at": "2025-08-15T10:00:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "churn-prevention",
  "type": "proactive-retention",
  "risk_score": 68,
  "reason": "frustrated",
  "offer": "personal_support"
}
```

## How to Run

```bash
# 1. Create the project
mkdir churn-prevention && cd churn-prevention
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export CSM_PHONE="5491155551234"

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

# 6. Register a user for churn monitoring
curl -X POST http://localhost:3000/admin/users \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "name": "Carolina Vega",
    "company": "Vega Consultores",
    "plan": "professional",
    "primary_use_case": "customer support automation"
  }'

# 7. Send engagement events from your app
curl -X POST http://localhost:3000/api/events \
  -H "Content-Type: application/json" \
  -d '{ "phone": "5491155559999", "event_type": "login" }'

# 8. Check churn dashboard
curl http://localhost:3000/api/churn-dashboard

# 9. Trigger proactive outreach
curl -X POST http://localhost:3000/admin/proactive-outreach
```
