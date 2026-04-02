# 37 - Abandoned Cart Recovery

E-commerce abandoned cart recovery via WhatsApp. Detect abandoned carts via webhook, send personalized reminders with AI-generated copy, offer discounts on follow-up, and track conversion.

## Problem

An e-commerce store has a 72% cart abandonment rate. Email recovery campaigns have 15% open rates and 2% conversion. Cart values average $45, and 500+ carts are abandoned daily. The store loses an estimated $300,000/month in recoverable revenue. SMS feels impersonal and has character limits.

## Solution

Deploy a WhatsApp cart recovery bot that:
- Receives abandoned cart webhooks from the e-commerce platform
- Sends personalized recovery messages with product details
- Uses AI to generate compelling, personalized copy
- Offers progressive discounts (first reminder: no discount, second: 10%, third: 15%)
- Tracks conversion in metadata (recovered vs. lost)
- Handles customer responses and questions about products

## Architecture

```
+-------------------+     POST /api/cart-abandoned   +--------------------+
|  E-commerce       | -----------------------------> |  Your Express App  |
|  Platform webhook |                                |   (this code)      |
+-------------------+                                +--------------------+
                                                              |
         +----------------------------------------------------+
         |                    |                               |
         v                    v                               v
+------------------+  Claude Haiku                    Schedule
|   Opero WPP API  |  (personalized copy)            follow-ups
|  wpp-api.opero.so|                                       |
+------------------+                                       v
         ^                                          +-----------+
         |            webhook                       | Timer:    |
         +---- customer responds  -------+          | 1h, 24h, |
                                         |          | 72h       |
                                         v          +-----------+
                                  Track conversion
                                  in metadata
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const STORE_URL = process.env.STORE_URL || 'https://tienda.example.com';

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// ── Recovery schedule ─────────────────────────────────────────────────
const RECOVERY_SCHEDULE = [
  { delay_ms: 60 * 60 * 1000, discount: 0, level: 1 },          // 1 hour, no discount
  { delay_ms: 24 * 60 * 60 * 1000, discount: 10, level: 2 },    // 24 hours, 10% off
  { delay_ms: 72 * 60 * 60 * 1000, discount: 15, level: 3 },    // 72 hours, 15% off
];

// ── Data stores ───────────────────────────────────────────────────────
const carts = new Map();      // cartId -> cart data
const timers = new Map();     // cartId -> timeout references
const conversions = new Map(); // phone -> conversion tracking

// ── Opero API helpers ─────────────────────────────────────────────────
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

async function updateConversation(phone, metadata) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    method: 'PUT',
    body: JSON.stringify({ metadata }),
  });
}

async function markAsRead(phone, messageIds) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/read`, {
    method: 'POST',
    body: JSON.stringify({ phone, message_ids: messageIds }),
  });
}

// ── AI copy generation ────────────────────────────────────────────────
async function generateRecoveryCopy(cart, level, discount) {
  const itemList = cart.items.map(i => `${i.name} ($${i.price})`).join(', ');

  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 256,
    system: `You write WhatsApp cart recovery messages in Spanish for an Argentine e-commerce store.
Rules:
- Keep it under 100 words
- Be friendly and conversational, not pushy
- Use the customer's first name
- Mention specific products they left behind
- If there's a discount, make it the hook
- Include a clear call to action
- Level 1: gentle reminder, no discount
- Level 2: mention the discount prominently
- Level 3: urgency + best discount + items may sell out
- Do NOT use markdown headers or bullet points
- Use WhatsApp formatting: *bold* for emphasis`,
    messages: [{
      role: 'user',
      content: `Customer: ${cart.customer_name}
Items in cart: ${itemList}
Cart total: $${cart.total}
Level: ${level}
Discount: ${discount}%
Cart link: ${cart.checkout_url}`,
    }],
  });

  return response.content[0].text;
}

// ── Recovery message sender ───────────────────────────────────────────
async function sendRecoveryMessage(cartId, scheduleIndex) {
  const cart = carts.get(cartId);
  if (!cart) return;

  // Check if already recovered
  if (cart.status === 'recovered' || cart.status === 'dismissed') {
    console.log(`Cart ${cartId} already ${cart.status}, skipping.`);
    return;
  }

  const schedule = RECOVERY_SCHEDULE[scheduleIndex];
  if (!schedule) return;

  try {
    // Generate personalized copy
    const copy = await generateRecoveryCopy(cart, schedule.level, schedule.discount);

    // Build message
    let message = copy;
    if (schedule.discount > 0) {
      const discountCode = `VOLVER${schedule.discount}`;
      cart.active_discount_code = discountCode;
      cart.active_discount_pct = schedule.discount;
      message += `\n\n🏷️ Código de descuento: *${discountCode}*`;
    }
    message += `\n\n🛒 ${cart.checkout_url}`;

    await sendText(cart.phone, message, {
      agent: 'abandoned-cart',
      cart_id: cartId,
      recovery_level: schedule.level,
      discount_pct: schedule.discount,
      cart_total: cart.total,
      items_count: cart.items.length,
    });

    cart.recovery_attempts++;
    cart.last_recovery_at = new Date().toISOString();

    // Update conversation metadata
    await updateConversation(cart.phone, {
      agent: 'abandoned-cart',
      cart: {
        id: cartId,
        total: cart.total,
        items: cart.items.map(i => i.name),
        status: cart.status,
        recovery_attempts: cart.recovery_attempts,
        active_discount: schedule.discount > 0 ? `${schedule.discount}%` : null,
      },
    });

    console.log(`[${new Date().toISOString()}] Recovery L${schedule.level} sent for cart ${cartId} (${cart.phone})`);

    // Schedule next follow-up
    if (scheduleIndex + 1 < RECOVERY_SCHEDULE.length) {
      const nextSchedule = RECOVERY_SCHEDULE[scheduleIndex + 1];
      const nextDelay = nextSchedule.delay_ms - schedule.delay_ms;
      const timer = setTimeout(() => sendRecoveryMessage(cartId, scheduleIndex + 1), nextDelay);
      timers.set(`${cartId}-${scheduleIndex + 1}`, timer);
    }
  } catch (err) {
    console.error(`Error sending recovery for cart ${cartId}:`, err.message);
  }
}

// ── Cancel recovery timers ────────────────────────────────────────────
function cancelTimers(cartId) {
  for (let i = 0; i < RECOVERY_SCHEDULE.length; i++) {
    const key = `${cartId}-${i}`;
    const timer = timers.get(key);
    if (timer) {
      clearTimeout(timer);
      timers.delete(key);
    }
  }
}

// ── Webhook: Cart abandoned ───────────────────────────────────────────
app.post('/api/cart-abandoned', async (req, res) => {
  const { cart_id, phone, customer_name, items, total, checkout_url } = req.body;

  if (!cart_id || !phone || !items) {
    return res.status(400).json({ error: 'cart_id, phone, and items are required' });
  }

  const cart = {
    id: cart_id,
    phone,
    customer_name: customer_name || 'Cliente',
    items, // [{ name, price, quantity, image_url }]
    total: total || items.reduce((sum, i) => sum + i.price * (i.quantity || 1), 0),
    checkout_url: checkout_url || `${STORE_URL}/checkout/${cart_id}`,
    status: 'abandoned', // abandoned, recovered, dismissed, expired
    recovery_attempts: 0,
    active_discount_code: null,
    active_discount_pct: 0,
    abandoned_at: new Date().toISOString(),
    recovered_at: null,
    last_recovery_at: null,
  };

  carts.set(cart_id, cart);

  // Schedule first recovery message
  const firstDelay = RECOVERY_SCHEDULE[0].delay_ms;
  const timer = setTimeout(() => sendRecoveryMessage(cart_id, 0), firstDelay);
  timers.set(`${cart_id}-0`, timer);

  console.log(`[${new Date().toISOString()}] Cart ${cart_id} abandoned by ${phone}, first recovery in ${firstDelay / 1000}s`);

  res.json({ status: 'scheduled', cart_id, first_recovery_in: `${firstDelay / 1000}s` });
});

// ── Webhook: Cart recovered (purchase completed) ──────────────────────
app.post('/api/cart-recovered', async (req, res) => {
  const { cart_id, phone } = req.body;
  const cart = carts.get(cart_id);

  if (cart) {
    cart.status = 'recovered';
    cart.recovered_at = new Date().toISOString();
    cancelTimers(cart_id);

    // Track conversion
    const conversion = {
      cart_id,
      cart_total: cart.total,
      discount_used: cart.active_discount_code,
      discount_pct: cart.active_discount_pct,
      recovery_attempts_before_conversion: cart.recovery_attempts,
      abandoned_at: cart.abandoned_at,
      recovered_at: cart.recovered_at,
      time_to_recover_hours: Math.round((new Date(cart.recovered_at) - new Date(cart.abandoned_at)) / (1000 * 60 * 60) * 10) / 10,
    };

    conversions.set(cart_id, conversion);

    try {
      await sendText(cart.phone,
        `🎉 ¡Gracias por tu compra, ${cart.customer_name}! Tu pedido está siendo procesado.\n\n` +
        `Si necesitás algo, estamos aquí para ayudarte.`,
        {
          agent: 'abandoned-cart',
          cart_id,
          action: 'purchase_confirmed',
          recovered: true,
          recovery_attempts: cart.recovery_attempts,
        }
      );

      await updateConversation(cart.phone, {
        agent: 'abandoned-cart',
        cart: { id: cart_id, status: 'recovered', total: cart.total },
        conversion,
      });
    } catch (err) {
      console.error('Error confirming purchase:', err.message);
    }
  }

  res.json({ status: 'ok' });
});

// ── Webhook handler (customer responses) ──────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received') return;
  if (data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim();

  if (!messageText) return;

  // Find active cart for this phone
  const activeCart = [...carts.values()].find(c => c.phone === phone && c.status === 'abandoned');
  if (!activeCart) return;

  console.log(`[${new Date().toISOString()}] Cart response from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const upper = messageText.toUpperCase();

    if (upper === 'NO' || upper === 'NO GRACIAS' || upper === 'BASTA') {
      activeCart.status = 'dismissed';
      cancelTimers(activeCart.id);

      await sendText(phone, 'Entendido, no le enviaremos más recordatorios sobre este carrito. ¡Gracias!', {
        agent: 'abandoned-cart',
        cart_id: activeCart.id,
        action: 'dismissed',
      });
    } else if (upper === 'DESCUENTO' || upper === 'CUPON' || upper === 'CÓDIGO') {
      if (activeCart.active_discount_code) {
        await sendText(phone,
          `🏷️ Su código de descuento: *${activeCart.active_discount_code}* (${activeCart.active_discount_pct}% off)\n\n` +
          `Aplíquelo en: ${activeCart.checkout_url}`,
          {
            agent: 'abandoned-cart',
            cart_id: activeCart.id,
            action: 'discount_reshared',
          }
        );
      } else {
        await sendText(phone,
          `Su carrito está esperándolo:\n${activeCart.checkout_url}\n\n` +
          `¡Pronto podría haber un descuento especial!`,
          {
            agent: 'abandoned-cart',
            cart_id: activeCart.id,
            action: 'no_discount_yet',
          }
        );
      }
    } else {
      // Any other message — treat as a question, respond with cart info
      const itemList = activeCart.items.map(i => `- ${i.name} ($${i.price})`).join('\n');
      await sendText(phone,
        `Su carrito:\n\n${itemList}\n\nTotal: *$${activeCart.total}*` +
        `${activeCart.active_discount_code ? `\n\nCódigo descuento: *${activeCart.active_discount_code}* (${activeCart.active_discount_pct}% off)` : ''}` +
        `\n\n🛒 ${activeCart.checkout_url}\n\n` +
        `Responda *NO* si no desea más recordatorios.`,
        {
          agent: 'abandoned-cart',
          cart_id: activeCart.id,
          action: 'cart_info_resent',
        }
      );
    }
  } catch (err) {
    console.error(`Error handling response from ${phone}:`, err.message);
  }
});

// ── API: Analytics ────────────────────────────────────────────────────
app.get('/api/analytics', (req, res) => {
  const allCarts = [...carts.values()];
  const allConversions = [...conversions.values()];

  const totalAbandoned = allCarts.length;
  const totalRecovered = allCarts.filter(c => c.status === 'recovered').length;
  const totalDismissed = allCarts.filter(c => c.status === 'dismissed').length;
  const totalPending = allCarts.filter(c => c.status === 'abandoned').length;

  const recoveredRevenue = allConversions.reduce((sum, c) => {
    const discountedTotal = c.cart_total * (1 - (c.discount_pct || 0) / 100);
    return sum + discountedTotal;
  }, 0);

  const avgTimeToRecover = allConversions.length > 0
    ? allConversions.reduce((sum, c) => sum + c.time_to_recover_hours, 0) / allConversions.length
    : 0;

  const byLevel = {};
  for (const c of allConversions) {
    const level = c.recovery_attempts_before_conversion;
    byLevel[level] = (byLevel[level] || 0) + 1;
  }

  res.json({
    total_abandoned: totalAbandoned,
    recovered: totalRecovered,
    dismissed: totalDismissed,
    pending: totalPending,
    recovery_rate: totalAbandoned > 0 ? `${((totalRecovered / totalAbandoned) * 100).toFixed(1)}%` : '0%',
    recovered_revenue: Math.round(recoveredRevenue),
    avg_time_to_recover_hours: Math.round(avgTimeToRecover * 10) / 10,
    conversions_by_attempt: byLevel,
  });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    active_carts: [...carts.values()].filter(c => c.status === 'abandoned').length,
    total_processed: carts.size,
    recovered: [...carts.values()].filter(c => c.status === 'recovered').length,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Abandoned cart recovery bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "abandoned-cart",
  "cart": {
    "id": "cart-12345",
    "total": 8500,
    "items": ["Remera algodón azul", "Pantalón cargo verde"],
    "status": "recovered"
  },
  "conversion": {
    "cart_total": 8500,
    "discount_used": "VOLVER10",
    "discount_pct": 10,
    "recovery_attempts_before_conversion": 2,
    "abandoned_at": "2026-04-02T10:00:00.000Z",
    "recovered_at": "2026-04-03T14:30:00.000Z",
    "time_to_recover_hours": 28.5
  }
}
```

Per-message metadata:

```json
{
  "agent": "abandoned-cart",
  "cart_id": "cart-12345",
  "recovery_level": 2,
  "discount_pct": 10,
  "cart_total": 8500,
  "items_count": 2
}
```

## How to Run

```bash
# 1. Create the project
mkdir abandoned-cart-recovery && cd abandoned-cart-recovery
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export STORE_URL="https://tienda.example.com"

# 3. Start the server
node server.js

# 4. Expose locally with ngrok
ngrok http 3000

# 5. Register your webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. Simulate an abandoned cart (from your e-commerce platform)
curl -X POST http://localhost:3000/api/cart-abandoned \
  -H "Content-Type: application/json" \
  -d '{
    "cart_id": "cart-12345",
    "phone": "5491155559999",
    "customer_name": "María",
    "items": [
      { "name": "Remera algodón azul", "price": 4500, "quantity": 1 },
      { "name": "Pantalón cargo verde", "price": 4000, "quantity": 1 }
    ],
    "total": 8500
  }'

# 7. Simulate a purchase (cart recovered)
curl -X POST http://localhost:3000/api/cart-recovered \
  -H "Content-Type: application/json" \
  -d '{ "cart_id": "cart-12345", "phone": "5491155559999" }'

# 8. Check analytics
curl http://localhost:3000/api/analytics
```
