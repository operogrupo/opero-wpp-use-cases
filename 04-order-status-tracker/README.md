# 04 - Order Status Tracker

Customers text their order number via WhatsApp and get instant status updates. No AI needed -- just pattern matching and API lookup.

## Problem

An online furniture store in Rosario ships 80 orders per day. Their customer service team spends 40% of their time answering "where is my order?" messages on WhatsApp -- the same question, over and over. Each lookup takes 2-3 minutes: open the CRM, search the order, copy the status, type it back. That's 4+ hours of repetitive work daily for a team of three.

## Solution

A lightweight bot that:
- Detects order numbers in messages using regex (format: `ORD-XXXXX` or just 5-digit numbers)
- Looks up order status from your existing order management system
- Replies with a formatted status update including tracking link
- Stores which orders each customer has queried in conversation metadata
- No AI model needed -- pure pattern matching keeps costs at zero

## Architecture

```
Customer sends "Hola, mi pedido es ORD-48291"
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                   +------+------+
         |                                   | Regex parse |
         |                                   | order number|
         |                                   +------+------+
         |                                          |
         |                                          v
         |                                 +------------------+
         |                                 | Order API / DB   |
         |                                 | (your backend)   |
         |                                 +------------------+
         |                                          |
         |      POST send formatted status          |
         |      PUT conversation metadata           |
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
const ORDER_API_URL = process.env.ORDER_API_URL || 'http://localhost:4000'; // Your order system

// ── Order number patterns ─────────────────────────────────────────────
const ORDER_PATTERNS = [
  /\bORD-(\d{4,6})\b/i,           // ORD-48291
  /\bpedido\s*#?\s*(\d{4,6})\b/i, // pedido #48291, pedido 48291
  /\border\s*#?\s*(\d{4,6})\b/i,  // order #48291
  /\b(\d{5})\b/,                   // bare 5-digit number (last resort)
];

function extractOrderNumber(text) {
  for (const pattern of ORDER_PATTERNS) {
    const match = text.match(pattern);
    if (match) {
      const num = match[1];
      return `ORD-${num}`;
    }
  }
  return null;
}

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

async function getConversation(phone) {
  try {
    const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations/${phone}`);
    return res.data;
  } catch {
    return null;
  }
}

// ── Order lookup ──────────────────────────────────────────────────────
// Replace this with your actual order system integration
async function lookupOrder(orderNumber) {
  try {
    const res = await fetch(`${ORDER_API_URL}/api/orders/${orderNumber}`);
    if (!res.ok) return null;
    return res.json();
  } catch {
    // Fallback: simulated order data for demo purposes
    return simulateOrder(orderNumber);
  }
}

function simulateOrder(orderNumber) {
  // Demo data -- remove this and use your real API
  const statuses = ['preparando', 'enviado', 'en_camino', 'entregado'];
  const hash = orderNumber.split('').reduce((a, c) => a + c.charCodeAt(0), 0);
  const statusIndex = hash % statuses.length;

  return {
    order_number: orderNumber,
    status: statuses[statusIndex],
    items: [
      { name: 'Mesa de comedor Escandinava', qty: 1, price: 185000 },
      { name: 'Sillas Nordicas x4', qty: 1, price: 120000 },
    ],
    total: 305000,
    shipping: {
      carrier: 'Andreani',
      tracking_number: `AND${orderNumber.replace('ORD-', '')}AR`,
      tracking_url: `https://www.andreani.com/#!/tracking/${orderNumber.replace('ORD-', '')}`,
      estimated_delivery: '2026-04-05',
    },
    created_at: '2026-03-28T10:00:00Z',
    updated_at: '2026-04-01T16:30:00Z',
  };
}

// ── Format order status message ───────────────────────────────────────
const STATUS_LABELS = {
  preparando: { label: 'En preparacion', emoji: '📦' },
  enviado: { label: 'Enviado', emoji: '🚚' },
  en_camino: { label: 'En camino', emoji: '🛣️' },
  entregado: { label: 'Entregado', emoji: '✅' },
  cancelado: { label: 'Cancelado', emoji: '❌' },
};

function formatOrderStatus(order) {
  const status = STATUS_LABELS[order.status] || { label: order.status, emoji: '' };
  const items = order.items.map(i => `  - ${i.name} x${i.qty}`).join('\n');
  const total = new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' }).format(order.total);

  let message = `${status.emoji} *Pedido ${order.order_number}*\n`;
  message += `Estado: ${status.label}\n\n`;
  message += `Productos:\n${items}\n`;
  message += `Total: ${total}\n`;

  if (order.shipping && order.status !== 'preparando') {
    message += `\nEnvio: ${order.shipping.carrier}\n`;
    message += `Seguimiento: ${order.shipping.tracking_number}\n`;

    if (order.shipping.tracking_url) {
      message += `Rastreo: ${order.shipping.tracking_url}\n`;
    }

    if (order.shipping.estimated_delivery) {
      const deliveryDate = new Date(order.shipping.estimated_delivery + 'T12:00:00');
      const formatted = deliveryDate.toLocaleDateString('es-AR', {
        weekday: 'long',
        day: 'numeric',
        month: 'long',
      });
      message += `Entrega estimada: ${formatted}\n`;
    }
  }

  if (order.status === 'preparando') {
    message += `\nTu pedido esta siendo preparado en nuestro deposito. Te notificaremos cuando se despache.`;
  }

  return message;
}

// ── Webhook handler ───────────────────────────────────────────────────
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
    const orderNumber = extractOrderNumber(messageText);

    if (!orderNumber) {
      // No order number found -- send help message
      await sendText(phone,
        `Hola! Soy el asistente de seguimiento de pedidos de Muebles del Sur.\n\n` +
        `Para consultar tu pedido, envame tu numero de orden. Lo encontras en el email de confirmacion.\n\n` +
        `Formato: ORD-48291 o simplemente el numero 48291`,
        { agent: 'order-tracker', action: 'help' }
      );
      return;
    }

    console.log(`[${new Date().toISOString()}] Looking up order: ${orderNumber}`);

    // Look up the order
    const order = await lookupOrder(orderNumber);

    if (!order) {
      await sendText(phone,
        `No encontramos el pedido ${orderNumber}. Verifica el numero y volve a intentar.\n\n` +
        `Si el problema persiste, contacta a soporte: (341) 555-6789`,
        { agent: 'order-tracker', action: 'not_found', order_number: orderNumber }
      );
      return;
    }

    // Send formatted status
    const statusMessage = formatOrderStatus(order);
    await sendText(phone, statusMessage, {
      agent: 'order-tracker',
      action: 'status_sent',
      order_number: orderNumber,
      order_status: order.status,
    });

    // Track this query in conversation metadata
    const conversation = await getConversation(phone);
    const existingMeta = conversation?.metadata || {};
    const queries = existingMeta.order_queries || [];

    queries.push({
      order_number: orderNumber,
      status_at_query: order.status,
      queried_at: new Date().toISOString(),
    });

    await updateConversation(phone, {
      ...existingMeta,
      order_queries: queries.slice(-50), // Keep last 50 queries
      last_order_queried: orderNumber,
      total_queries: (existingMeta.total_queries || 0) + 1,
      customer_type: 'order_tracker',
    });

    console.log(`[${new Date().toISOString()}] Status sent for ${orderNumber} to ${phone}`);
  } catch (err) {
    console.error(`Error handling order query from ${phone}:`, err.message);
    await sendText(phone,
      'Disculpa, no pudimos consultar tu pedido en este momento. Intenta de nuevo en unos minutos.',
      { agent: 'order-tracker', error: true }
    );
  }
});

// ── Proactive status update endpoint ──────────────────────────────────
// Call this from your order system when a status changes
app.post('/notify-status-change', async (req, res) => {
  const { order_number, phone, old_status, new_status } = req.body;

  if (!order_number || !phone || !new_status) {
    return res.status(400).json({ error: 'Missing required fields: order_number, phone, new_status' });
  }

  try {
    const order = await lookupOrder(order_number);
    if (!order) {
      return res.status(404).json({ error: 'Order not found' });
    }

    const statusLabel = STATUS_LABELS[new_status]?.label || new_status;
    const message = `Actualizacion de tu pedido ${order_number}:\n\n` +
      `Nuevo estado: ${statusLabel}\n\n` +
      formatOrderStatus(order);

    await sendText(phone, message, {
      agent: 'order-tracker',
      action: 'proactive_update',
      order_number,
      old_status,
      new_status,
    });

    res.json({ success: true, message: 'Notification sent' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

const PORT = process.env.PORT || 3003;
app.listen(PORT, () => {
  console.log(`Order status tracker listening on port ${PORT}`);
});
```

## Metadata Schema

### Message-level metadata

```json
{
  "agent": "order-tracker",
  "action": "status_sent",
  "order_number": "ORD-48291",
  "order_status": "en_camino"
}
```

### Conversation-level metadata

```json
{
  "customer_type": "order_tracker",
  "last_order_queried": "ORD-48291",
  "total_queries": 3,
  "order_queries": [
    {
      "order_number": "ORD-48291",
      "status_at_query": "preparando",
      "queried_at": "2026-03-29T10:00:00.000Z"
    },
    {
      "order_number": "ORD-48291",
      "status_at_query": "enviado",
      "queried_at": "2026-04-01T14:00:00.000Z"
    },
    {
      "order_number": "ORD-48291",
      "status_at_query": "en_camino",
      "queried_at": "2026-04-02T09:00:00.000Z"
    }
  ]
}
```

## How to Run

```bash
# 1. Create the project
mkdir order-status-tracker && cd order-status-tracker
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ORDER_API_URL="http://your-order-system.com"  # Optional, uses demo data if not set

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3003

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. Trigger a proactive status update (from your order system)
curl -X POST http://localhost:3003/notify-status-change \
  -H "Content-Type: application/json" \
  -d '{
    "order_number": "ORD-48291",
    "phone": "5493415551234",
    "old_status": "enviado",
    "new_status": "en_camino"
  }'
```
