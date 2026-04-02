# 51 - AI Sales Closer

Advanced AI sales closer via WhatsApp. Handles objections, negotiates pricing, creates urgency, and closes deals. Uses conversation history and lead metadata to personalize the approach with multiple closing techniques. Powered by Claude Opus for highest quality.

## Problem

A B2B SaaS company generates 200 leads per month but only closes 15%. The sales team spends 70% of their time on repetitive back-and-forth messaging -- handling the same objections, explaining pricing, following up with unresponsive leads. Top closers outperform average reps by 3x because they know which technique to use and when. But top closers are expensive ($120K/year) and can only handle 40 leads at a time.

## Solution

Deploy an AI sales closer that:
- Engages leads with personalized messaging based on their profile and behavior
- Handles common objections (price, timing, competition, authority) with proven techniques
- Negotiates pricing within configurable bounds
- Creates genuine urgency without being pushy
- Uses multiple closing techniques (assumptive, summary, urgency, trial, alternative)
- Tracks deal stage, objections, and engagement score in metadata
- Escalates to a human closer when the deal is complex or the lead requests it
- Uses Claude Opus for the highest quality, most natural conversations

## Architecture

```
Lead sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
| wpp-api.opero.so |                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |   Claude Opus    |
         |                                 | (Anthropic API)  |
         |                                 +------------------+
         |                                          |
         |    GET/PUT metadata, POST /messages/text |
         +------------------------------------------+
         |
         v
+------------------+       escalation       +------------------+
| Human Sales Rep  | <--------------------- | Escalation Logic |
+------------------+                        +------------------+
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
const SALES_MANAGER_PHONE = process.env.SALES_MANAGER_PHONE;

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// -- Product/pricing config --
const PRODUCT = {
  name: 'Opero Platform',
  tagline: 'AI-powered business automation',
  plans: {
    starter: { name: 'Starter', monthly: 49, annual: 470, features: ['5 automations', '1,000 messages/mo', 'Email support'] },
    professional: { name: 'Professional', monthly: 99, annual: 950, features: ['Unlimited automations', '10,000 messages/mo', 'Priority support', 'API access', 'Analytics'] },
    enterprise: { name: 'Enterprise', monthly: 249, annual: 2390, features: ['Everything in Pro', 'Unlimited messages', 'Dedicated CSM', 'Custom integrations', 'SLA guarantee', 'SSO'] },
  },
  max_discount_pct: 20,
  trial_days: 14,
  competitor_comparisons: {
    'competitor_a': 'We offer 3x more automations at the same price, plus our AI is more advanced.',
    'competitor_b': 'Unlike Competitor B, we include API access in all plans and have a 99.9% uptime SLA.',
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

// -- Lead state management --
function initializeLead(metadata) {
  return {
    ...metadata,
    role: metadata.role || 'lead',
    lead_name: metadata.lead_name || '',
    company: metadata.company || '',
    industry: metadata.industry || '',
    company_size: metadata.company_size || '',
    deal_stage: metadata.deal_stage || 'new', // new, qualifying, demo_scheduled, proposal, negotiation, closed_won, closed_lost
    recommended_plan: metadata.recommended_plan || 'professional',
    objections_raised: metadata.objections_raised || [],
    objections_handled: metadata.objections_handled || [],
    engagement_score: metadata.engagement_score || 0,
    messages_sent: metadata.messages_sent || 0,
    messages_received: metadata.messages_received || 0,
    discount_offered: metadata.discount_offered || 0,
    closing_attempts: metadata.closing_attempts || 0,
    pain_points: metadata.pain_points || [],
    budget_mentioned: metadata.budget_mentioned || null,
    decision_timeline: metadata.decision_timeline || null,
    competitors_mentioned: metadata.competitors_mentioned || [],
    human_requested: metadata.human_requested || false,
    created_at: metadata.created_at || new Date().toISOString(),
  };
}

function updateEngagementScore(lead) {
  let score = 0;
  score += Math.min(lead.messages_received * 5, 30); // Up to 30 for message count
  score += lead.pain_points.length * 10; // 10 per pain point identified
  score += lead.objections_handled.length * 5; // 5 per objection overcome
  if (lead.budget_mentioned) score += 15;
  if (lead.decision_timeline) score += 10;
  if (lead.deal_stage === 'proposal') score += 20;
  if (lead.deal_stage === 'negotiation') score += 30;
  score -= lead.objections_raised.length * 3; // Slight penalty for many objections

  return Math.max(0, Math.min(100, score));
}

// -- AI Sales Agent --
function buildSystemPrompt(lead) {
  const plan = PRODUCT.plans[lead.recommended_plan] || PRODUCT.plans.professional;

  return `You are an elite AI sales closer for ${PRODUCT.name} (${PRODUCT.tagline}). You are warm, consultative, and results-driven. You never lie or make promises you can't keep.

LEAD PROFILE:
- Name: ${lead.lead_name || 'Unknown'}
- Company: ${lead.company || 'Unknown'}
- Industry: ${lead.industry || 'Unknown'}
- Company size: ${lead.company_size || 'Unknown'}
- Deal stage: ${lead.deal_stage}
- Engagement score: ${lead.engagement_score}/100
- Pain points identified: ${lead.pain_points.length > 0 ? lead.pain_points.join(', ') : 'None yet'}
- Objections raised: ${lead.objections_raised.length > 0 ? lead.objections_raised.join(', ') : 'None yet'}
- Budget: ${lead.budget_mentioned || 'Not discussed'}
- Timeline: ${lead.decision_timeline || 'Not discussed'}
- Competitors mentioned: ${lead.competitors_mentioned.length > 0 ? lead.competitors_mentioned.join(', ') : 'None'}
- Closing attempts: ${lead.closing_attempts}
- Discount offered: ${lead.discount_offered}%

RECOMMENDED PLAN: ${plan.name} ($${plan.monthly}/mo or $${plan.annual}/yr)
Features: ${plan.features.join(', ')}

ALL PLANS:
${Object.entries(PRODUCT.plans).map(([k, p]) => `- ${p.name}: $${p.monthly}/mo | $${p.annual}/yr`).join('\n')}

PRICING RULES:
- Maximum discount: ${PRODUCT.max_discount_pct}% (only offer if the lead pushes back on price)
- Start with no discount, offer 10% if pressed, ${PRODUCT.max_discount_pct}% only as a last resort
- Annual billing already saves ~20% over monthly
- Free ${PRODUCT.trial_days}-day trial available

COMPETITOR INTEL:
${Object.entries(PRODUCT.competitor_comparisons).map(([k, v]) => `- ${k}: ${v}`).join('\n')}

SALES METHODOLOGY:
1. DISCOVER: Ask about their challenges, goals, current tools
2. QUALIFY: Understand budget, authority, need, timeline (BANT)
3. PRESENT: Match their pain points to specific features
4. HANDLE OBJECTIONS: Address concerns with empathy and evidence
5. CLOSE: Use appropriate closing technique based on context

CLOSING TECHNIQUES (use naturally, don't force):
- Assumptive close: "Should I set up your Professional account today?"
- Summary close: List all benefits they've expressed interest in, then ask
- Urgency close: "Our Q2 pricing locks in at this rate for 12 months"
- Trial close: "Why not try the 14-day free trial? No credit card needed"
- Alternative close: "Would the monthly or annual plan work better for you?"

ACTION TAGS (include naturally in your response):
- [STAGE:stage_name] - Update deal stage (qualifying, demo_scheduled, proposal, negotiation, closed_won, closed_lost)
- [PAIN:description] - New pain point identified
- [OBJECTION:type] - Objection raised (price, timing, competition, authority, need, technical)
- [OBJECTION_HANDLED:type] - Objection successfully addressed
- [DISCOUNT:percent] - Discount offered (only if necessary, max ${PRODUCT.max_discount_pct})
- [BUDGET:amount] - Budget mentioned by lead
- [TIMELINE:description] - Decision timeline mentioned
- [COMPETITOR:name] - Competitor mentioned
- [CLOSE_ATTEMPT] - You're making a closing attempt
- [ESCALATE] - Lead wants to talk to a human

RULES:
- Be conversational, not robotic. Use WhatsApp-appropriate language
- Don't send walls of text. Keep messages under 150 words
- Ask one question at a time
- Reference their specific pain points and situation
- If they say "no" firmly, respect it and don't push
- If they ask for a human, immediately tag [ESCALATE]
- Never reveal you are AI unless directly asked. If asked, be honest
- Focus on value, not features
- Use their name occasionally for personalization`;
}

async function generateReply(phone, message, lead) {
  const history = await getMessages(phone, 20);
  const messages = history
    .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    .map(m => ({
      role: m.from_me ? 'assistant' : 'user',
      content: m.text || m.content?.text || '',
    }))
    .filter(m => m.content);

  messages.push({ role: 'user', content: message });

  const response = await anthropic.messages.create({
    model: 'claude-opus-4-20250514',
    max_tokens: 800,
    system: buildSystemPrompt(lead),
    messages: messages.slice(-30),
  });

  return {
    text: response.content[0].text,
    usage: response.usage,
  };
}

// -- Action processor --
async function processActions(phone, aiResponse, lead) {
  let finalResponse = aiResponse;

  // Stage update
  const stageMatch = aiResponse.match(/\[STAGE:(\w+)\]/);
  if (stageMatch) {
    lead.deal_stage = stageMatch[1];
    finalResponse = finalResponse.replace(stageMatch[0], '');
  }

  // Pain points
  const painMatches = aiResponse.matchAll(/\[PAIN:([^\]]+)\]/g);
  for (const match of painMatches) {
    const pain = match[1].trim();
    if (!lead.pain_points.includes(pain)) {
      lead.pain_points.push(pain);
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Objections raised
  const objMatches = aiResponse.matchAll(/\[OBJECTION:(\w+)\]/g);
  for (const match of objMatches) {
    const obj = match[1].trim();
    if (!lead.objections_raised.includes(obj)) {
      lead.objections_raised.push(obj);
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Objections handled
  const handledMatches = aiResponse.matchAll(/\[OBJECTION_HANDLED:(\w+)\]/g);
  for (const match of handledMatches) {
    const obj = match[1].trim();
    if (!lead.objections_handled.includes(obj)) {
      lead.objections_handled.push(obj);
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Discount
  const discountMatch = aiResponse.match(/\[DISCOUNT:(\d+)\]/);
  if (discountMatch) {
    const discount = Math.min(parseInt(discountMatch[1]), PRODUCT.max_discount_pct);
    lead.discount_offered = Math.max(lead.discount_offered, discount);
    finalResponse = finalResponse.replace(discountMatch[0], '');
  }

  // Budget
  const budgetMatch = aiResponse.match(/\[BUDGET:([^\]]+)\]/);
  if (budgetMatch) {
    lead.budget_mentioned = budgetMatch[1].trim();
    finalResponse = finalResponse.replace(budgetMatch[0], '');
  }

  // Timeline
  const timelineMatch = aiResponse.match(/\[TIMELINE:([^\]]+)\]/);
  if (timelineMatch) {
    lead.decision_timeline = timelineMatch[1].trim();
    finalResponse = finalResponse.replace(timelineMatch[0], '');
  }

  // Competitor
  const compMatches = aiResponse.matchAll(/\[COMPETITOR:([^\]]+)\]/g);
  for (const match of compMatches) {
    const comp = match[1].trim().toLowerCase();
    if (!lead.competitors_mentioned.includes(comp)) {
      lead.competitors_mentioned.push(comp);
    }
    finalResponse = finalResponse.replace(match[0], '');
  }

  // Close attempt
  if (aiResponse.includes('[CLOSE_ATTEMPT]')) {
    lead.closing_attempts++;
    finalResponse = finalResponse.replace(/\[CLOSE_ATTEMPT\]/g, '');
  }

  // Escalation
  if (aiResponse.includes('[ESCALATE]')) {
    lead.human_requested = true;
    lead.escalated_at = new Date().toISOString();
    finalResponse = finalResponse.replace(/\[ESCALATE\]/g, '');

    // Notify sales manager
    await sendText(SALES_MANAGER_PHONE, [
      `*LEAD ESCALATION*`,
      ``,
      `Lead: ${lead.lead_name || 'Unknown'} (${phone})`,
      `Company: ${lead.company || 'Unknown'}`,
      `Stage: ${lead.deal_stage}`,
      `Engagement: ${lead.engagement_score}/100`,
      `Pain points: ${lead.pain_points.join(', ') || 'None'}`,
      `Objections: ${lead.objections_raised.join(', ') || 'None'}`,
      `Budget: ${lead.budget_mentioned || 'N/A'}`,
      `Discount offered: ${lead.discount_offered}%`,
      `Messages exchanged: ${lead.messages_received + lead.messages_sent}`,
      ``,
      `Please take over this conversation.`,
    ].join('\n'), {
      agent: 'ai-sales-closer',
      type: 'escalation',
      lead_phone: phone,
    });
  }

  // Update engagement score
  lead.messages_received++;
  lead.engagement_score = updateEngagementScore(lead);
  lead.last_interaction = new Date().toISOString();

  // Save
  await updateConversationMetadata(phone, lead);

  // Clean remaining tags
  finalResponse = finalResponse.replace(/\[\w+(?::[^\]]+)*\]/g, '').trim();

  return finalResponse;
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

  console.log(`[${new Date().toISOString()}] Lead ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    await sendTyping(phone);

    const metadata = await getConversationMetadata(phone);
    const lead = initializeLead(metadata);

    // Don't respond if escalated to human
    if (lead.human_requested) {
      console.log(`Skipping AI response for ${phone} (escalated to human)`);
      return;
    }

    // Don't respond if deal is closed
    if (lead.deal_stage === 'closed_won' || lead.deal_stage === 'closed_lost') {
      return;
    }

    const { text: aiResponse, usage } = await generateReply(phone, messageText, lead);
    const finalResponse = await processActions(phone, aiResponse, lead);

    // Natural typing delay
    const typingDelay = Math.min(finalResponse.length * 35, 4000);
    await new Promise(resolve => setTimeout(resolve, typingDelay));

    lead.messages_sent++;
    await updateConversationMetadata(phone, lead);

    await sendText(phone, finalResponse, {
      agent: 'ai-sales-closer',
      model: 'claude-opus-4-20250514',
      deal_stage: lead.deal_stage,
      engagement_score: lead.engagement_score,
      closing_attempts: lead.closing_attempts,
      tokens_used: usage.input_tokens + usage.output_tokens,
      generated_at: new Date().toISOString(),
    });

    console.log(`[${new Date().toISOString()}] Reply sent to ${phone} (stage: ${lead.deal_stage}, engagement: ${lead.engagement_score})`);
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Apologies, let me get back to you in a moment.', {
        agent: 'ai-sales-closer', error: true, error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback failed:', fallbackErr.message);
    }
  }
});

// -- Admin: register a lead --
app.post('/admin/leads', async (req, res) => {
  try {
    const { phone, name, company, industry, company_size, plan, source } = req.body;
    const lead = initializeLead({
      role: 'lead',
      lead_name: name,
      company,
      industry,
      company_size,
      recommended_plan: plan || 'professional',
      source: source || 'manual',
      created_at: new Date().toISOString(),
    });

    await updateConversationMetadata(phone, lead);

    // Send initial outreach
    await sendText(phone,
      `Hi ${name}! I'm reaching out from ${PRODUCT.name}. I noticed ${company} might benefit from our ${PRODUCT.tagline} platform. Would you have a couple minutes to chat about how we could help?`,
      { agent: 'ai-sales-closer', type: 'outreach', source }
    );

    lead.messages_sent++;
    await updateConversationMetadata(phone, lead);
    res.json({ success: true, lead });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: view pipeline --
app.get('/admin/pipeline', async (req, res) => {
  try {
    const conversations = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=500`);
    const leads = (conversations.data || [])
      .filter(c => c.metadata?.role === 'lead')
      .map(c => ({
        phone: c.phone || c.jid,
        name: c.metadata.lead_name,
        company: c.metadata.company,
        stage: c.metadata.deal_stage,
        engagement: c.metadata.engagement_score,
        objections: c.metadata.objections_raised,
        discount: c.metadata.discount_offered,
        closing_attempts: c.metadata.closing_attempts,
        last_interaction: c.metadata.last_interaction,
      }));

    const pipeline = {
      new: leads.filter(l => l.stage === 'new').length,
      qualifying: leads.filter(l => l.stage === 'qualifying').length,
      demo_scheduled: leads.filter(l => l.stage === 'demo_scheduled').length,
      proposal: leads.filter(l => l.stage === 'proposal').length,
      negotiation: leads.filter(l => l.stage === 'negotiation').length,
      closed_won: leads.filter(l => l.stage === 'closed_won').length,
      closed_lost: leads.filter(l => l.stage === 'closed_lost').length,
    };

    res.json({ pipeline, leads, total: leads.length });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: resume AI after human handoff --
app.post('/admin/leads/:phone/resume', async (req, res) => {
  try {
    const { phone } = req.params;
    const metadata = await getConversationMetadata(phone);
    metadata.human_requested = false;
    await updateConversationMetadata(phone, metadata);
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
  console.log(`AI sales closer listening on port ${PORT}`);
});
```

## Metadata Schema

Each lead conversation stores full deal intelligence:

```json
{
  "role": "lead",
  "lead_name": "Martin Rodriguez",
  "company": "TechCo Argentina",
  "industry": "SaaS",
  "company_size": "50-200",
  "deal_stage": "negotiation",
  "recommended_plan": "professional",
  "objections_raised": ["price", "timing"],
  "objections_handled": ["price"],
  "engagement_score": 72,
  "messages_sent": 12,
  "messages_received": 10,
  "discount_offered": 10,
  "closing_attempts": 2,
  "pain_points": ["manual processes", "no API integration", "slow customer support"],
  "budget_mentioned": "$100-150/month",
  "decision_timeline": "End of Q2",
  "competitors_mentioned": ["competitor_a"],
  "human_requested": false,
  "source": "linkedin",
  "created_at": "2026-03-20T10:00:00.000Z",
  "last_interaction": "2026-04-02T14:30:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "ai-sales-closer",
  "model": "claude-opus-4-20250514",
  "deal_stage": "negotiation",
  "engagement_score": 72,
  "closing_attempts": 2,
  "tokens_used": 1850,
  "generated_at": "2026-04-02T14:30:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir ai-sales-closer && cd ai-sales-closer
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export SALES_MANAGER_PHONE="5491155551234"

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

# 6. Register a lead and send initial outreach
curl -X POST http://localhost:3000/admin/leads \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "name": "Martin Rodriguez",
    "company": "TechCo Argentina",
    "industry": "SaaS",
    "company_size": "50-200",
    "plan": "professional",
    "source": "linkedin"
  }'

# 7. View sales pipeline
curl http://localhost:3000/admin/pipeline
```
