# 35 - Subscription Management

Subscription management bot via WhatsApp. Check plan status, upgrade/downgrade, cancel, update payment info, and view billing history. All subscription state tracked in metadata.

## Problem

A SaaS company with 2,000+ subscribers handles 50+ support tickets daily about billing: "What plan am I on?", "How do I cancel?", "When is my next payment?". Each ticket takes 10 minutes to resolve. Churn increases when cancellation is hard, but making it too easy loses revenue. The company needs self-service subscription management that feels personal.

## Solution

Deploy a WhatsApp subscription management bot that:
- Lets subscribers check their plan status, usage, and billing history
- Handles upgrades, downgrades, and cancellations with confirmation flows
- Sends payment reminders and failed payment notifications
- Offers retention deals when users try to cancel
- Tracks all subscription state in conversation metadata

## Architecture

```
Subscriber sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 | Subscription DB  |
         |                                 | (in-memory/Redis)|
         |                                 +------------------+
         |                                          |
         |      POST /messages/text                 |
         |      PUT  /conversations/:phone          |
         +------------------------------------------+
```

## Code

```javascript
// server.js
import express from 'express';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

// ── Plans ─────────────────────────────────────────────────────────────
const PLANS = {
  free: { name: 'Free', price: 0, features: ['5 proyectos', '1 GB almacenamiento', 'Soporte por email'] },
  starter: { name: 'Starter', price: 9900, features: ['25 proyectos', '10 GB almacenamiento', 'Soporte por chat', 'Integraciones básicas'] },
  pro: { name: 'Pro', price: 24900, features: ['Proyectos ilimitados', '100 GB almacenamiento', 'Soporte prioritario', 'Todas las integraciones', 'API access'] },
  enterprise: { name: 'Enterprise', price: 79900, features: ['Todo en Pro', 'Almacenamiento ilimitado', 'Soporte dedicado', 'SLA 99.9%', 'SSO', 'Auditoría'] },
};

const PLAN_ORDER = ['free', 'starter', 'pro', 'enterprise'];

// ── Data stores ───────────────────────────────────────────────────────
const subscriptions = new Map(); // phone -> subscription
const sessions = new Map();      // phone -> session state

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

// ── Subscription helpers ──────────────────────────────────────────────
function getSubscription(phone) {
  return subscriptions.get(phone);
}

function createSubscription(phone, planId, name) {
  const plan = PLANS[planId];
  if (!plan) throw new Error('Invalid plan');

  const sub = {
    phone,
    customer_name: name,
    plan_id: planId,
    plan_name: plan.name,
    price: plan.price,
    status: 'active', // active, past_due, cancelled, paused
    billing_cycle: 'monthly',
    next_billing_date: getNextMonth(),
    created_at: new Date().toISOString(),
    billing_history: [{
      date: new Date().toISOString().split('T')[0],
      amount: plan.price,
      status: 'paid',
      description: `Suscripción ${plan.name} - primer mes`,
    }],
    cancellation_date: null,
    cancel_reason: null,
  };

  subscriptions.set(phone, sub);
  return sub;
}

function getNextMonth() {
  const d = new Date();
  d.setMonth(d.getMonth() + 1);
  return d.toISOString().split('T')[0];
}

function formatPrice(cents) {
  return `$${(cents / 100).toLocaleString('es-AR', { minimumFractionDigits: 0 })}`;
}

async function syncMetadata(phone) {
  const sub = subscriptions.get(phone);
  if (!sub) return;

  await updateConversation(phone, {
    agent: 'subscription-management',
    subscription: {
      plan: sub.plan_id,
      plan_name: sub.plan_name,
      status: sub.status,
      price: sub.price,
      next_billing: sub.next_billing_date,
      created_at: sub.created_at,
      billing_history_count: sub.billing_history.length,
    },
  });
}

// ── Webhook handler ───────────────────────────────────────────────────
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

  console.log(`[${new Date().toISOString()}] Sub message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const sub = getSubscription(phone);
    const session = sessions.get(phone) || { step: 'idle' };
    const upper = messageText.toUpperCase();

    // No subscription found
    if (!sub && upper !== 'MENU' && session.step === 'idle') {
      await sendText(phone, 'No encontramos una suscripción asociada a este número. Si cree que es un error, contacte soporte en support@example.com.', {
        agent: 'subscription-management', action: 'no_subscription',
      });
      return;
    }

    // Global menu
    if (upper === 'MENU' || upper === 'HOLA' || upper === 'INICIO') {
      sessions.delete(phone);
      if (!sub) {
        await sendText(phone, 'No tiene suscripción activa. Visite nuestro sitio web para crear una cuenta.', {
          agent: 'subscription-management', action: 'no_subscription',
        });
        return;
      }

      await sendText(phone,
        `📋 *Gestión de Suscripción*\n\n` +
        `Plan actual: *${sub.plan_name}* (${formatPrice(sub.price)}/mes)\n` +
        `Estado: ${sub.status === 'active' ? '✅ Activo' : sub.status === 'past_due' ? '⚠️ Pago pendiente' : sub.status === 'paused' ? '⏸️ Pausado' : '❌ Cancelado'}\n\n` +
        `¿Qué desea hacer?\n\n` +
        `*1* - Ver detalle de mi plan\n` +
        `*2* - Cambiar de plan\n` +
        `*3* - Ver historial de pagos\n` +
        `*4* - Información de pago\n` +
        `*5* - Cancelar suscripción\n` +
        `*6* - Hablar con soporte`,
        { agent: 'subscription-management', action: 'menu_shown' }
      );
      return;
    }

    // Handle session-based flows
    if (session.step === 'confirm_change') {
      if (upper === 'SI' || upper === 'SÍ' || messageText === '1') {
        const newPlanId = session.data.new_plan;
        const newPlan = PLANS[newPlanId];
        const oldPlan = sub.plan_name;

        sub.plan_id = newPlanId;
        sub.plan_name = newPlan.name;
        sub.price = newPlan.price;

        sub.billing_history.push({
          date: new Date().toISOString().split('T')[0],
          amount: newPlan.price,
          status: 'paid',
          description: `Cambio de plan: ${oldPlan} → ${newPlan.name}`,
        });

        sessions.delete(phone);

        await sendText(phone,
          `✅ *Plan actualizado*\n\n` +
          `Nuevo plan: *${newPlan.name}* (${formatPrice(newPlan.price)}/mes)\n` +
          `Efectivo desde hoy.\n\n` +
          `Características:\n${newPlan.features.map(f => `- ${f}`).join('\n')}`,
          {
            agent: 'subscription-management',
            action: 'plan_changed',
            old_plan: oldPlan,
            new_plan: newPlanId,
          }
        );

        await syncMetadata(phone);
      } else {
        sessions.delete(phone);
        await sendText(phone, 'Cambio cancelado. Su plan no fue modificado. Escriba *MENU* para volver.', {
          agent: 'subscription-management', action: 'change_cancelled',
        });
      }
      return;
    }

    if (session.step === 'confirm_cancel') {
      if (upper === 'CONFIRMAR') {
        sub.status = 'cancelled';
        sub.cancellation_date = new Date().toISOString();
        sub.cancel_reason = session.data.reason || 'No especificado';

        sessions.delete(phone);

        await sendText(phone,
          `Su suscripción *${sub.plan_name}* ha sido cancelada.\n\n` +
          `Tendrá acceso hasta el *${sub.next_billing_date}*.\n` +
          `No se realizarán más cobros.\n\n` +
          `Si cambia de opinión, escriba *REACTIVAR* antes de esa fecha.`,
          {
            agent: 'subscription-management',
            action: 'subscription_cancelled',
            plan: sub.plan_id,
            reason: sub.cancel_reason,
          }
        );

        await syncMetadata(phone);
      } else {
        sessions.delete(phone);
        await sendText(phone, '🎉 ¡Nos alegra que se quede! Su suscripción sigue activa. Escriba *MENU* para volver.', {
          agent: 'subscription-management', action: 'cancel_aborted',
        });
      }
      return;
    }

    if (session.step === 'cancel_reason') {
      session.data.reason = messageText;
      session.step = 'confirm_cancel';
      sessions.set(phone, session);

      // Retention offer based on plan
      const currentIdx = PLAN_ORDER.indexOf(sub.plan_id);
      let retentionOffer = '';
      if (currentIdx > 0) {
        const lowerPlan = PLANS[PLAN_ORDER[currentIdx - 1]];
        retentionOffer = `\n\n💡 *Oferta especial:* ¿Preferiría cambiar al plan *${lowerPlan.name}* por ${formatPrice(lowerPlan.price)}/mes en lugar de cancelar?`;
      }

      await sendText(phone,
        `Entendemos. Antes de cancelar, queremos asegurarnos de que sea lo que necesita.${retentionOffer}\n\n` +
        `Para confirmar la cancelación, escriba *CONFIRMAR*.\n` +
        `Para volver al menú, escriba *MENU*.`,
        {
          agent: 'subscription-management',
          action: 'retention_attempt',
          cancel_reason: messageText,
        }
      );
      return;
    }

    // Main menu options
    switch (messageText.trim()) {
      case '1': // Plan details
        await sendText(phone,
          `📋 *Detalle de su plan: ${sub.plan_name}*\n\n` +
          `Precio: ${formatPrice(sub.price)}/mes\n` +
          `Estado: ${sub.status}\n` +
          `Próximo cobro: ${sub.next_billing_date}\n` +
          `Miembro desde: ${sub.created_at.split('T')[0]}\n\n` +
          `*Características incluidas:*\n${PLANS[sub.plan_id].features.map(f => `✓ ${f}`).join('\n')}`,
          { agent: 'subscription-management', action: 'plan_details' }
        );
        break;

      case '2': { // Change plan
        const currentIdx = PLAN_ORDER.indexOf(sub.plan_id);
        const availablePlans = PLAN_ORDER
          .filter(p => p !== sub.plan_id && p !== 'free')
          .map((p, i) => {
            const plan = PLANS[p];
            const direction = PLAN_ORDER.indexOf(p) > currentIdx ? '⬆️' : '⬇️';
            return `*${i + 1}* - ${direction} ${plan.name} (${formatPrice(plan.price)}/mes)`;
          });

        sessions.set(phone, { step: 'select_plan', data: { available: PLAN_ORDER.filter(p => p !== sub.plan_id && p !== 'free') } });

        await sendText(phone,
          `*Cambiar plan* (actual: ${sub.plan_name})\n\n${availablePlans.join('\n')}\n\n` +
          `Elija el número del plan deseado o escriba *MENU* para volver.`,
          { agent: 'subscription-management', action: 'change_plan_menu' }
        );
        break;
      }

      case '3': { // Billing history
        const history = sub.billing_history.slice(-5).reverse();
        const list = history.map(h =>
          `${h.date} — ${formatPrice(h.amount)} — ${h.status === 'paid' ? '✅' : '❌'} ${h.description}`
        ).join('\n');

        await sendText(phone,
          `💳 *Historial de pagos* (últimos 5)\n\n${list}\n\n` +
          `Total pagado: ${formatPrice(sub.billing_history.filter(h => h.status === 'paid').reduce((sum, h) => sum + h.amount, 0))}`,
          { agent: 'subscription-management', action: 'billing_history' }
        );
        break;
      }

      case '4': // Payment info
        await sendText(phone,
          `💳 *Información de pago*\n\n` +
          `Método: Tarjeta terminada en ****4242\n` +
          `Próximo cobro: ${sub.next_billing_date}\n` +
          `Monto: ${formatPrice(sub.price)}\n\n` +
          `Para actualizar su método de pago, visite:\nhttps://app.example.com/billing\n\n` +
          `O responda *SOPORTE* para asistencia.`,
          { agent: 'subscription-management', action: 'payment_info' }
        );
        break;

      case '5': // Cancel
        if (sub.status === 'cancelled') {
          await sendText(phone, 'Su suscripción ya está cancelada. Escriba *REACTIVAR* para volver a activarla.', {
            agent: 'subscription-management', action: 'already_cancelled',
          });
        } else {
          sessions.set(phone, { step: 'cancel_reason', data: {} });
          await sendText(phone,
            `Lamentamos que quiera cancelar. Para ayudarnos a mejorar, ¿podría indicarnos el motivo?\n\n` +
            `1. Precio muy alto\n` +
            `2. No uso lo suficiente\n` +
            `3. Encontré otra solución\n` +
            `4. Falta de funcionalidades\n` +
            `5. Otro`,
            { agent: 'subscription-management', action: 'cancel_started' }
          );
        }
        break;

      case '6': // Support
        await sendText(phone,
          `📞 *Soporte*\n\n` +
          `Email: support@example.com\n` +
          `Chat: https://app.example.com/support\n` +
          `Horario: Lunes a Viernes 9:00-18:00\n\n` +
          `O describa su consulta aquí y le responderemos.`,
          { agent: 'subscription-management', action: 'support_info' }
        );
        break;

      default: {
        // Handle plan selection flow
        if (session.step === 'select_plan') {
          const idx = parseInt(messageText, 10) - 1;
          const available = session.data.available;
          if (isNaN(idx) || idx < 0 || idx >= available.length) {
            await sendText(phone, `Por favor elija un número entre 1 y ${available.length}.`, {
              agent: 'subscription-management', action: 'invalid_plan_selection',
            });
            return;
          }

          const newPlanId = available[idx];
          const newPlan = PLANS[newPlanId];
          const isUpgrade = PLAN_ORDER.indexOf(newPlanId) > PLAN_ORDER.indexOf(sub.plan_id);

          sessions.set(phone, {
            step: 'confirm_change',
            data: { new_plan: newPlanId },
          });

          await sendText(phone,
            `*Confirmar cambio de plan*\n\n` +
            `${isUpgrade ? '⬆️ Upgrade' : '⬇️ Downgrade'}\n` +
            `De: ${sub.plan_name} (${formatPrice(sub.price)}/mes)\n` +
            `A: *${newPlan.name}* (${formatPrice(newPlan.price)}/mes)\n\n` +
            `Diferencia: ${formatPrice(newPlan.price - sub.price)}/mes\n\n` +
            `¿Confirmar? Responda *SI* o *NO*.`,
            {
              agent: 'subscription-management',
              action: 'confirm_change',
              new_plan: newPlanId,
              direction: isUpgrade ? 'upgrade' : 'downgrade',
            }
          );
          return;
        }

        // Handle reactivation
        if (upper === 'REACTIVAR' && sub.status === 'cancelled') {
          sub.status = 'active';
          sub.cancellation_date = null;
          sub.cancel_reason = null;
          sub.next_billing_date = getNextMonth();

          await sendText(phone,
            `✅ *Suscripción reactivada*\n\n` +
            `Plan: ${sub.plan_name}\n` +
            `Próximo cobro: ${sub.next_billing_date}\n\n` +
            `¡Bienvenido/a de vuelta!`,
            { agent: 'subscription-management', action: 'reactivated' }
          );
          await syncMetadata(phone);
          return;
        }

        await sendText(phone, 'No entendí su mensaje. Escriba *MENU* para ver las opciones.', {
          agent: 'subscription-management', action: 'unrecognized',
        });
        break;
      }
    }
  } catch (err) {
    console.error(`Error from ${phone}:`, err.message);
  }
});

// ── API: Create subscription ──────────────────────────────────────────
app.post('/api/subscriptions', async (req, res) => {
  const { phone, plan_id, customer_name } = req.body;
  try {
    const sub = createSubscription(phone, plan_id, customer_name);
    await syncMetadata(phone);
    res.json(sub);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Get subscription ─────────────────────────────────────────────
app.get('/api/subscriptions/:phone', (req, res) => {
  const sub = subscriptions.get(req.params.phone);
  if (!sub) return res.status(404).json({ error: 'Not found' });
  res.json(sub);
});

// ── API: Send payment reminder ────────────────────────────────────────
app.post('/api/subscriptions/:phone/remind', async (req, res) => {
  const sub = subscriptions.get(req.params.phone);
  if (!sub) return res.status(404).json({ error: 'Not found' });

  try {
    await sendText(sub.phone,
      `💳 *Recordatorio de pago*\n\n` +
      `Plan: ${sub.plan_name}\n` +
      `Monto: ${formatPrice(sub.price)}\n` +
      `Fecha de cobro: ${sub.next_billing_date}\n\n` +
      `Si necesita actualizar su método de pago, escriba *MENU* y seleccione la opción 4.`,
      {
        agent: 'subscription-management',
        action: 'payment_reminder',
        amount: sub.price,
      }
    );
    res.json({ status: 'sent' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...subscriptions.values()];
  res.json({
    status: 'ok',
    total_subscriptions: all.length,
    active: all.filter(s => s.status === 'active').length,
    cancelled: all.filter(s => s.status === 'cancelled').length,
    mrr: all.filter(s => s.status === 'active').reduce((sum, s) => sum + s.price, 0),
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Subscription management bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "subscription-management",
  "subscription": {
    "plan": "pro",
    "plan_name": "Pro",
    "status": "active",
    "price": 24900,
    "next_billing": "2026-05-02",
    "created_at": "2026-01-15T10:00:00.000Z",
    "billing_history_count": 3
  }
}
```

Per-message metadata:

```json
{
  "agent": "subscription-management",
  "action": "plan_changed",
  "old_plan": "Starter",
  "new_plan": "pro"
}
```

## How to Run

```bash
# 1. Create the project
mkdir subscription-management && cd subscription-management
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"

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

# 6. Create a subscription via API
curl -X POST http://localhost:3000/api/subscriptions \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "plan_id": "pro",
    "customer_name": "María González"
  }'
```
