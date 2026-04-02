# 48 - Membership Renewal Reminders

Membership renewal reminder system for clubs, gyms, and associations. Sends reminders at 30/7/1 days before expiry, handles renewal confirmations, offers upgrade options, and tracks renewal history in metadata.

## Problem

A fitness club with 500 members has a 25% churn rate at renewal time. Members forget their expiry date, and the front desk only catches lapses after the fact. Renewal calls are awkward and time-consuming. 40% of lapsed members say they would have renewed if reminded. The club loses $15,000/month in preventable churn.

## Solution

Deploy a membership renewal system that:
- Sends automated reminders at 30, 7, and 1 day(s) before expiry
- Handles "RENEW" confirmations via WhatsApp
- Offers plan upgrades during renewal
- Tracks renewal history and lifetime value per member
- Sends a final "we miss you" message after expiry with a comeback offer
- No AI needed -- uses structured messaging and keyword responses

## Architecture

```
+-------------------+
|  Cron Scheduler   |  (daily check for upcoming expirations)
+-------------------+
         |
         v
+------------------+     renewal reminder      +--------------------+
|   Opero WPP API  | <------------------------ |  Your Express App  |
| wpp-api.opero.so |                           |   (this code)      |
+------------------+                           +--------------------+
         |                                              ^
         |    webhook POST (member replies)             |
         +--------------------------------------------->+
                                                        |
                                               +------------------+
                                               | Membership DB    |
                                               | (conversation    |
                                               |  metadata)       |
                                               +------------------+
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
const ADMIN_PHONE = process.env.ADMIN_PHONE;

// -- Membership plans --
const PLANS = {
  basic: { name: 'Basic', monthly: 29.99, annual: 299.99, features: ['Gym access (Mon-Fri 6am-10pm)', 'Locker room'] },
  premium: { name: 'Premium', monthly: 49.99, annual: 499.99, features: ['24/7 gym access', 'Pool & sauna', 'Locker room', '1 group class/week'] },
  elite: { name: 'Elite', monthly: 79.99, annual: 799.99, features: ['24/7 gym access', 'Pool & sauna', 'All group classes', 'Personal training (2x/month)', 'Towel service'] },
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

// -- Date helpers --
function daysUntil(dateStr) {
  const target = new Date(dateStr);
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  target.setHours(0, 0, 0, 0);
  return Math.ceil((target - today) / (1000 * 60 * 60 * 24));
}

function addDays(dateStr, days) {
  const d = new Date(dateStr);
  d.setDate(d.getDate() + days);
  return d.toISOString().split('T')[0];
}

function addMonths(dateStr, months) {
  const d = new Date(dateStr);
  d.setMonth(d.getMonth() + months);
  return d.toISOString().split('T')[0];
}

function addYear(dateStr) {
  const d = new Date(dateStr);
  d.setFullYear(d.getFullYear() + 1);
  return d.toISOString().split('T')[0];
}

// -- Renewal check (daily cron) --
async function checkRenewals() {
  console.log(`[${new Date().toISOString()}] Checking membership renewals...`);
  const conversations = await listConversations();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.role || metadata.role !== 'member') continue;
    if (metadata.status === 'cancelled') continue;

    const phone = conv.phone || conv.jid;
    const days = daysUntil(metadata.expiry_date);
    const memberName = metadata.member_name;
    const plan = PLANS[metadata.plan] || PLANS.basic;

    // Already sent reminders tracking
    if (!metadata.reminders_sent) metadata.reminders_sent = {};

    // 30-day reminder
    if (days === 30 && !metadata.reminders_sent['30day']) {
      await sendText(phone, [
        `Hi ${memberName}! Your *${plan.name}* membership expires on *${metadata.expiry_date}*.`,
        ``,
        `That's 30 days away -- a great time to plan your renewal!`,
        ``,
        `Reply *RENEW* to renew your current plan ($${metadata.billing_cycle === 'annual' ? plan.annual + '/year' : plan.monthly + '/month'})`,
        `Reply *PLANS* to see all available plans`,
        `Reply *UPGRADE* to explore upgrade options`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'reminder-30day',
      });
      metadata.reminders_sent['30day'] = new Date().toISOString();
      await updateConversationMetadata(phone, metadata);
    }

    // 7-day reminder
    else if (days === 7 && !metadata.reminders_sent['7day']) {
      await sendText(phone, [
        `${memberName}, your *${plan.name}* membership expires in *7 days* (${metadata.expiry_date}).`,
        ``,
        `Don't lose access to: ${plan.features.slice(0, 3).join(', ')}.`,
        ``,
        `Reply *RENEW* to renew now`,
        `Reply *UPGRADE* for upgrade options`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'reminder-7day',
      });
      metadata.reminders_sent['7day'] = new Date().toISOString();
      await updateConversationMetadata(phone, metadata);
    }

    // 1-day reminder
    else if (days === 1 && !metadata.reminders_sent['1day']) {
      await sendText(phone, [
        `${memberName}, your membership expires *TOMORROW*.`,
        ``,
        `Reply *RENEW* now to avoid any interruption to your gym access.`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'reminder-1day',
      });
      metadata.reminders_sent['1day'] = new Date().toISOString();
      await updateConversationMetadata(phone, metadata);
    }

    // Expiry day
    else if (days === 0 && !metadata.reminders_sent['expired']) {
      metadata.status = 'expired';
      await sendText(phone, [
        `${memberName}, your *${plan.name}* membership has expired today.`,
        ``,
        `You can still renew to reactivate immediately.`,
        `Reply *RENEW* to reactivate.`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'expiry-notice',
      });
      metadata.reminders_sent['expired'] = new Date().toISOString();
      await updateConversationMetadata(phone, metadata);
    }

    // 7 days after expiry -- win-back offer
    else if (days === -7 && !metadata.reminders_sent['winback']) {
      await sendText(phone, [
        `We miss you, ${memberName}!`,
        ``,
        `As a returning member, we'd like to offer you *10% off* your renewal.`,
        ``,
        `Reply *COMEBACK* to renew with your discount.`,
        `This offer expires in 7 days.`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'winback-offer',
      });
      metadata.reminders_sent['winback'] = new Date().toISOString();
      await updateConversationMetadata(phone, metadata);
    }
  }
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

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    if (metadata.role !== 'member') {
      await sendText(phone, 'Welcome! Please visit the front desk to register your membership.', {
        agent: 'membership-renewal', type: 'unregistered',
      });
      return;
    }

    // RENEW command
    if (messageText === 'RENEW' || messageText === 'RENOVAR') {
      const plan = PLANS[metadata.plan] || PLANS.basic;
      const price = metadata.billing_cycle === 'annual' ? plan.annual : plan.monthly;
      const newExpiry = metadata.billing_cycle === 'annual'
        ? addYear(metadata.expiry_date > new Date().toISOString().split('T')[0] ? metadata.expiry_date : new Date().toISOString().split('T')[0])
        : addMonths(metadata.expiry_date > new Date().toISOString().split('T')[0] ? metadata.expiry_date : new Date().toISOString().split('T')[0], 1);

      // Record renewal
      if (!metadata.renewal_history) metadata.renewal_history = [];
      metadata.renewal_history.push({
        plan: metadata.plan,
        billing_cycle: metadata.billing_cycle,
        amount: price,
        renewed_at: new Date().toISOString(),
        previous_expiry: metadata.expiry_date,
        new_expiry: newExpiry,
      });

      metadata.expiry_date = newExpiry;
      metadata.status = 'active';
      metadata.reminders_sent = {};
      metadata.total_spent = (metadata.total_spent || 0) + price;
      metadata.renewal_count = (metadata.renewal_count || 0) + 1;

      await updateConversationMetadata(phone, metadata);

      await sendText(phone, [
        `Membership renewed! Here's your confirmation:`,
        ``,
        `Plan: *${plan.name}*`,
        `Amount: *$${price}*`,
        `Valid until: *${newExpiry}*`,
        ``,
        `Thank you for staying with us, ${metadata.member_name}!`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'renewal-confirmed', plan: metadata.plan, amount: price,
      });

      // Notify admin
      await sendText(ADMIN_PHONE, `Membership renewed: ${metadata.member_name} (${phone}) - ${plan.name} $${price} until ${newExpiry}`, {
        agent: 'membership-renewal', type: 'admin-notification',
      });
    }

    // COMEBACK (winback offer)
    else if (messageText === 'COMEBACK') {
      const plan = PLANS[metadata.plan] || PLANS.basic;
      const basePrice = metadata.billing_cycle === 'annual' ? plan.annual : plan.monthly;
      const discountedPrice = Math.round(basePrice * 0.9 * 100) / 100;
      const newExpiry = metadata.billing_cycle === 'annual'
        ? addYear(new Date().toISOString().split('T')[0])
        : addMonths(new Date().toISOString().split('T')[0], 1);

      if (!metadata.renewal_history) metadata.renewal_history = [];
      metadata.renewal_history.push({
        plan: metadata.plan,
        billing_cycle: metadata.billing_cycle,
        amount: discountedPrice,
        discount: '10%',
        renewed_at: new Date().toISOString(),
        previous_expiry: metadata.expiry_date,
        new_expiry: newExpiry,
        type: 'winback',
      });

      metadata.expiry_date = newExpiry;
      metadata.status = 'active';
      metadata.reminders_sent = {};
      metadata.total_spent = (metadata.total_spent || 0) + discountedPrice;
      metadata.renewal_count = (metadata.renewal_count || 0) + 1;

      await updateConversationMetadata(phone, metadata);

      await sendText(phone, [
        `Welcome back, ${metadata.member_name}!`,
        ``,
        `Plan: *${plan.name}* (10% comeback discount)`,
        `Amount: *$${discountedPrice}* (was $${basePrice})`,
        `Valid until: *${newExpiry}*`,
        ``,
        `Great to have you back!`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'winback-confirmed', plan: metadata.plan, amount: discountedPrice,
      });
    }

    // PLANS command
    else if (messageText === 'PLANS') {
      const planList = Object.entries(PLANS).map(([key, plan]) => [
        `*${plan.name}*${key === metadata.plan ? ' (current)' : ''}`,
        `  Monthly: $${plan.monthly} | Annual: $${plan.annual}`,
        `  ${plan.features.join(', ')}`,
      ].join('\n')).join('\n\n');

      await sendText(phone, `Available plans:\n\n${planList}\n\nReply *UPGRADE PLANNAME* to switch (e.g., UPGRADE PREMIUM)`, {
        agent: 'membership-renewal', type: 'plans-list',
      });
    }

    // UPGRADE command
    else if (messageText.startsWith('UPGRADE')) {
      const targetPlanKey = messageText.replace('UPGRADE', '').trim().toLowerCase();
      const targetPlan = PLANS[targetPlanKey];
      const currentPlan = PLANS[metadata.plan];

      if (!targetPlan) {
        await sendText(phone, `Available plans to upgrade to: ${Object.keys(PLANS).filter(k => k !== metadata.plan).join(', ').toUpperCase()}`, {
          agent: 'membership-renewal', type: 'upgrade-options',
        });
        return;
      }

      if (targetPlanKey === metadata.plan) {
        await sendText(phone, `You're already on the ${targetPlan.name} plan!`, {
          agent: 'membership-renewal', type: 'already-on-plan',
        });
        return;
      }

      // Apply upgrade
      const price = metadata.billing_cycle === 'annual' ? targetPlan.annual : targetPlan.monthly;
      metadata.plan = targetPlanKey;

      if (!metadata.renewal_history) metadata.renewal_history = [];
      metadata.renewal_history.push({
        plan: targetPlanKey,
        billing_cycle: metadata.billing_cycle,
        amount: price,
        type: 'upgrade',
        from_plan: metadata.plan,
        upgraded_at: new Date().toISOString(),
      });

      metadata.total_spent = (metadata.total_spent || 0) + price;
      await updateConversationMetadata(phone, metadata);

      await sendText(phone, [
        `Plan upgraded to *${targetPlan.name}*!`,
        ``,
        `New features: ${targetPlan.features.join(', ')}`,
        `Amount: $${price}/${metadata.billing_cycle === 'annual' ? 'year' : 'month'}`,
        `Valid until: ${metadata.expiry_date}`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'upgrade-confirmed', plan: targetPlanKey,
      });
    }

    // STATUS command
    else if (messageText === 'STATUS' || messageText === 'MI PLAN') {
      const plan = PLANS[metadata.plan] || PLANS.basic;
      const days = daysUntil(metadata.expiry_date);

      await sendText(phone, [
        `*Your Membership*`,
        ``,
        `Member: ${metadata.member_name}`,
        `Plan: ${plan.name}`,
        `Status: ${metadata.status}`,
        `Expires: ${metadata.expiry_date} (${days > 0 ? `${days} days left` : 'expired'})`,
        `Billing: ${metadata.billing_cycle}`,
        `Renewals: ${metadata.renewal_count || 0}`,
        `Total spent: $${metadata.total_spent || 0}`,
        `Member since: ${metadata.joined_at?.split('T')[0] || 'N/A'}`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'status-check',
      });
    }

    // Help
    else {
      await sendText(phone, [
        `Hi ${metadata.member_name}! Commands:`,
        ``,
        `*RENEW* - Renew your current plan`,
        `*PLANS* - View available plans`,
        `*UPGRADE PLAN* - Switch to a different plan`,
        `*STATUS* - Check your membership`,
      ].join('\n'), {
        agent: 'membership-renewal', type: 'help',
      });
    }
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- Admin: register a member --
app.post('/admin/members', async (req, res) => {
  try {
    const { phone, name, plan, billing_cycle, expiry_date } = req.body;
    const selectedPlan = PLANS[plan] || PLANS.basic;

    const metadata = {
      role: 'member',
      member_name: name,
      plan: plan || 'basic',
      billing_cycle: billing_cycle || 'monthly',
      expiry_date: expiry_date || addMonths(new Date().toISOString().split('T')[0], billing_cycle === 'annual' ? 12 : 1),
      status: 'active',
      total_spent: billing_cycle === 'annual' ? selectedPlan.annual : selectedPlan.monthly,
      renewal_count: 0,
      renewal_history: [],
      reminders_sent: {},
      joined_at: new Date().toISOString(),
    };

    await updateConversationMetadata(phone, metadata);
    await sendText(phone,
      `Welcome to the club, ${name}! Your *${selectedPlan.name}* membership is active until ${metadata.expiry_date}.\n\nReply *STATUS* anytime to check your membership.`,
      { agent: 'membership-renewal', type: 'registration' }
    );

    res.json({ success: true, metadata });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: trigger renewal check --
app.post('/admin/check-renewals', async (req, res) => {
  try {
    await checkRenewals();
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Daily cron --
function scheduleDailyCheck() {
  const now = new Date();
  const next9am = new Date();
  next9am.setHours(9, 0, 0, 0);
  if (now >= next9am) next9am.setDate(next9am.getDate() + 1);
  setTimeout(() => {
    checkRenewals();
    setInterval(checkRenewals, 24 * 60 * 60 * 1000);
  }, next9am - now);
  console.log(`Renewal check scheduled for ${next9am.toISOString()}`);
}

// -- Health check --
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Membership renewal bot listening on port ${PORT}`);
  scheduleDailyCheck();
});
```

## Metadata Schema

Each member's conversation stores membership and renewal data:

```json
{
  "role": "member",
  "member_name": "Sofia Martinez",
  "plan": "premium",
  "billing_cycle": "monthly",
  "expiry_date": "2026-05-01",
  "status": "active",
  "total_spent": 349.93,
  "renewal_count": 7,
  "renewal_history": [
    {
      "plan": "premium",
      "billing_cycle": "monthly",
      "amount": 49.99,
      "renewed_at": "2026-04-01T10:30:00.000Z",
      "previous_expiry": "2026-04-01",
      "new_expiry": "2026-05-01"
    }
  ],
  "reminders_sent": {
    "30day": "2026-03-02T09:00:00.000Z"
  },
  "joined_at": "2025-10-01T09:00:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "membership-renewal",
  "type": "renewal-confirmed",
  "plan": "premium",
  "amount": 49.99
}
```

## How to Run

```bash
# 1. Create the project
mkdir membership-renewal && cd membership-renewal
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ADMIN_PHONE="5491155551234"

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

# 6. Register a member
curl -X POST http://localhost:3000/admin/members \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "name": "Sofia Martinez",
    "plan": "premium",
    "billing_cycle": "monthly"
  }'

# 7. Trigger renewal check
curl -X POST http://localhost:3000/admin/check-renewals
```
