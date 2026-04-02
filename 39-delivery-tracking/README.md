# 39 - Delivery Tracking

Real-time delivery tracking updates via WhatsApp. Integration with delivery API, automatic status updates at each stage, and customer self-service "where's my order?" queries.

## Problem

A delivery company handles 500+ shipments daily. Customers call the support line 200+ times per day asking "where's my package?". Each call costs $3 in agent time. The tracking page on the website gets low traffic because customers don't bookmark it. SMS updates are limited to 160 characters and feel robotic.

## Solution

Deploy a WhatsApp delivery tracking bot that:
- Sends proactive status updates at each delivery stage (picked up, in transit, out for delivery, delivered)
- Lets customers ask "where's my order?" anytime and get instant status
- Provides estimated delivery times and driver contact
- Handles delivery issues (delay, failed attempt, wrong address)
- Stores full tracking history in conversation metadata

## Architecture

```
+-------------------+     POST /api/shipments    +--------------------+
|  Delivery System  | --------------------------> |  Your Express App  |
|  (status updates) |                            |   (this code)      |
+-------------------+                            +--------------------+
                                                          |
         +------------------------------------------------+
         |                    |                           |
         v                    v                           v
+------------------+   Send updates             Track status
|   Opero WPP API  |   to customer              in metadata
|  wpp-api.opero.so|                                    |
+------------------+                                    v
         ^                                      +-----------+
         |            webhook                   | Customer  |
         +---- customer asks status  ---------> | queries   |
                                                +-----------+
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

// ── Delivery stages ───────────────────────────────────────────────────
const STAGES = {
  order_confirmed: {
    label: 'Pedido confirmado',
    icon: '📦',
    message: (s) => `Su pedido *${s.order_id}* ha sido confirmado.\n\nProductos: ${s.items.join(', ')}\n\nLe avisaremos cuando sea despachado.`,
  },
  picked_up: {
    label: 'Retirado',
    icon: '🏪',
    message: (s) => `Su pedido *${s.order_id}* fue retirado del depósito y está en camino.\n\nEntrega estimada: *${s.estimated_delivery}*`,
  },
  in_transit: {
    label: 'En tránsito',
    icon: '🚚',
    message: (s) => `Su pedido *${s.order_id}* está en tránsito.\n\nUbicación actual: ${s.current_location || 'En ruta'}\nEntrega estimada: *${s.estimated_delivery}*`,
  },
  out_for_delivery: {
    label: 'En reparto',
    icon: '📍',
    message: (s) => `Su pedido *${s.order_id}* está *en reparto*.\n\nRepartidor: ${s.driver_name || 'Asignado'}\n${s.driver_phone ? `📞 Contacto: ${s.driver_phone}\n` : ''}Dirección: ${s.delivery_address}\n\nLlegará en las próximas horas. Por favor asegúrese de estar disponible.`,
  },
  delivered: {
    label: 'Entregado',
    icon: '✅',
    message: (s) => `¡Su pedido *${s.order_id}* fue entregado!\n\n${s.delivery_proof ? `Recibido por: ${s.delivery_proof}\n` : ''}Hora: ${new Date().toLocaleTimeString('es-AR')}\n\n¿Todo en orden? Responda:\n*1* - Todo perfecto\n*2* - Tengo un problema`,
  },
  failed_attempt: {
    label: 'Intento fallido',
    icon: '⚠️',
    message: (s) => `No pudimos entregar su pedido *${s.order_id}*.\n\nMotivo: ${s.failure_reason || 'No había nadie en el domicilio'}\n\nSe intentará nuevamente el próximo día hábil.\n\nSi desea reprogramar, responda con la fecha deseada (DD/MM).`,
  },
  delayed: {
    label: 'Demorado',
    icon: '⏰',
    message: (s) => `Lamentamos informarle que su pedido *${s.order_id}* sufrió una demora.\n\nMotivo: ${s.delay_reason || 'Condiciones climáticas'}\nNueva fecha estimada: *${s.new_estimated_delivery || 'por confirmar'}*\n\nDisculpe las molestias.`,
  },
  returned: {
    label: 'Devuelto',
    icon: '↩️',
    message: (s) => `Su pedido *${s.order_id}* fue devuelto al remitente.\n\nMotivo: ${s.return_reason || 'No se pudo entregar'}\n\nPara coordinar un nuevo envío, responda *REENVIAR* o contacte soporte.`,
  },
};

// ── Data stores ───────────────────────────────────────────────────────
const shipments = new Map();    // shipmentId -> shipment
const orderIndex = new Map();   // orderId -> shipmentId
const phoneIndex = new Map();   // phone -> [shipmentIds]

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

// ── Shipment management ───────────────────────────────────────────────
function createShipment(data) {
  const id = data.shipment_id || `SHIP-${Date.now().toString(36).toUpperCase()}`;
  const shipment = {
    id,
    order_id: data.order_id,
    customer_phone: data.customer_phone,
    customer_name: data.customer_name || '',
    items: data.items || [],
    delivery_address: data.delivery_address || '',
    estimated_delivery: data.estimated_delivery || '',
    driver_name: data.driver_name || null,
    driver_phone: data.driver_phone || null,
    current_location: null,
    status: 'order_confirmed',
    tracking_history: [{
      status: 'order_confirmed',
      timestamp: new Date().toISOString(),
      details: 'Pedido confirmado',
    }],
    delivery_proof: null,
    created_at: new Date().toISOString(),
  };

  shipments.set(id, shipment);
  orderIndex.set(data.order_id, id);

  if (!phoneIndex.has(data.customer_phone)) {
    phoneIndex.set(data.customer_phone, []);
  }
  phoneIndex.get(data.customer_phone).push(id);

  return shipment;
}

async function updateShipmentStatus(shipmentId, status, extras = {}) {
  const shipment = shipments.get(shipmentId);
  if (!shipment) throw new Error('Shipment not found');

  // Update shipment fields
  shipment.status = status;
  Object.assign(shipment, extras);

  // Add to tracking history
  shipment.tracking_history.push({
    status,
    timestamp: new Date().toISOString(),
    details: STAGES[status]?.label || status,
    ...extras,
  });

  // Send notification
  const stage = STAGES[status];
  if (stage) {
    const message = `${stage.icon} *${stage.label}*\n\n${stage.message(shipment)}`;

    await sendText(shipment.customer_phone, message, {
      agent: 'delivery-tracking',
      shipment_id: shipmentId,
      order_id: shipment.order_id,
      status,
      stage_label: stage.label,
    });
  }

  // Update conversation metadata
  await syncMetadata(shipment.customer_phone);

  console.log(`[${new Date().toISOString()}] Shipment ${shipmentId} → ${status}`);
  return shipment;
}

async function syncMetadata(phone) {
  const shipmentIds = phoneIndex.get(phone) || [];
  const activeShipments = shipmentIds
    .map(id => shipments.get(id))
    .filter(s => s && !['delivered', 'returned'].includes(s.status));

  const recentDelivered = shipmentIds
    .map(id => shipments.get(id))
    .filter(s => s && s.status === 'delivered')
    .slice(-3);

  await updateConversation(phone, {
    agent: 'delivery-tracking',
    active_shipments: activeShipments.map(s => ({
      id: s.id,
      order_id: s.order_id,
      status: s.status,
      estimated_delivery: s.estimated_delivery,
      items: s.items,
    })),
    recent_delivered: recentDelivered.map(s => ({
      id: s.id,
      order_id: s.order_id,
      delivered_at: s.tracking_history.find(h => h.status === 'delivered')?.timestamp,
    })),
    total_shipments: shipmentIds.length,
  });
}

// ── Webhook handler (customer queries) ────────────────────────────────
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

  console.log(`[${new Date().toISOString()}] Tracking query from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const upper = messageText.toUpperCase();
    const shipmentIds = phoneIndex.get(phone) || [];

    if (shipmentIds.length === 0) {
      await sendText(phone, 'No encontramos envíos asociados a este número. Si tiene un número de pedido, escríbalo para rastrearlo.', {
        agent: 'delivery-tracking', action: 'no_shipments',
      });
      return;
    }

    // Find active shipments
    const activeShipments = shipmentIds
      .map(id => shipments.get(id))
      .filter(s => s && !['delivered', 'returned'].includes(s.status));

    // Check for order ID lookup
    const orderLookup = orderIndex.get(messageText.toUpperCase());
    if (orderLookup) {
      const shipment = shipments.get(orderLookup);
      if (shipment) {
        const stage = STAGES[shipment.status];
        const history = shipment.tracking_history
          .slice(-5)
          .map(h => `${new Date(h.timestamp).toLocaleString('es-AR')} — ${h.details}`)
          .join('\n');

        await sendText(phone,
          `📦 *Pedido ${shipment.order_id}*\n\n` +
          `Estado: ${stage?.icon || '?'} *${stage?.label || shipment.status}*\n` +
          `Productos: ${shipment.items.join(', ')}\n` +
          `Entrega estimada: ${shipment.estimated_delivery}\n` +
          `${shipment.driver_name ? `Repartidor: ${shipment.driver_name}\n` : ''}` +
          `\n*Historial:*\n${history}`,
          {
            agent: 'delivery-tracking',
            shipment_id: shipment.id,
            order_id: shipment.order_id,
            action: 'status_query',
          }
        );
        return;
      }
    }

    // "Where's my order?" type queries
    if (upper.includes('PEDIDO') || upper.includes('ENVIO') || upper.includes('ENVÍO') ||
        upper.includes('DONDE') || upper.includes('DÓNDE') || upper.includes('ESTADO') ||
        upper.includes('RASTREAR') || upper === 'MENU' || upper === 'HOLA') {

      if (activeShipments.length === 0) {
        const lastDelivered = shipmentIds
          .map(id => shipments.get(id))
          .filter(s => s && s.status === 'delivered')
          .pop();

        if (lastDelivered) {
          await sendText(phone, `No tiene envíos activos. Su último pedido *${lastDelivered.order_id}* fue entregado.\n\n¿Tiene un nuevo número de pedido? Escríbalo para rastrearlo.`, {
            agent: 'delivery-tracking', action: 'no_active_shipments',
          });
        } else {
          await sendText(phone, 'No tiene envíos activos. Si tiene un número de pedido, escríbalo.', {
            agent: 'delivery-tracking', action: 'no_active_shipments',
          });
        }
      } else if (activeShipments.length === 1) {
        const s = activeShipments[0];
        const stage = STAGES[s.status];

        await sendText(phone,
          `📦 *Pedido ${s.order_id}*\n\n` +
          `${stage?.icon || '?'} Estado: *${stage?.label || s.status}*\n` +
          `Productos: ${s.items.join(', ')}\n` +
          `Entrega estimada: *${s.estimated_delivery}*\n` +
          `${s.current_location ? `Ubicación: ${s.current_location}\n` : ''}` +
          `${s.driver_name ? `Repartidor: ${s.driver_name}\n` : ''}` +
          `${s.driver_phone ? `📞 Contacto repartidor: ${s.driver_phone}\n` : ''}`,
          {
            agent: 'delivery-tracking',
            shipment_id: s.id,
            order_id: s.order_id,
            action: 'status_query',
            status: s.status,
          }
        );
      } else {
        const list = activeShipments.map(s => {
          const stage = STAGES[s.status];
          return `${stage?.icon || '?'} *${s.order_id}* — ${stage?.label || s.status} (ETA: ${s.estimated_delivery})`;
        }).join('\n');

        await sendText(phone, `Tiene ${activeShipments.length} envíos activos:\n\n${list}\n\nEscriba el número de pedido para ver el detalle.`, {
          agent: 'delivery-tracking', action: 'multiple_shipments',
        });
      }
    } else if (messageText === '1') {
      // Delivery feedback: all good
      const lastDelivered = shipmentIds
        .map(id => shipments.get(id))
        .filter(s => s && s.status === 'delivered')
        .pop();

      if (lastDelivered) {
        await sendText(phone, '¡Excelente! Nos alegra que todo haya llegado bien. ¡Gracias por su compra!', {
          agent: 'delivery-tracking',
          order_id: lastDelivered.order_id,
          action: 'positive_feedback',
        });
      }
    } else if (messageText === '2') {
      // Delivery problem
      await sendText(phone, 'Lamentamos el inconveniente. Por favor describa el problema y un agente lo contactará.\n\nTambién puede llamar al +54 11 4444-3456.', {
        agent: 'delivery-tracking', action: 'problem_reported',
      });
    } else if (upper === 'REENVIAR') {
      await sendText(phone, 'Para coordinar un reenvío, contáctenos al +54 11 4444-3456 o responda con la nueva dirección de entrega.', {
        agent: 'delivery-tracking', action: 'reship_requested',
      });
    } else {
      await sendText(phone,
        `No entendí su mensaje. Puede:\n\n` +
        `- Escribir su *número de pedido* para rastrearlo\n` +
        `- Escribir *ESTADO* para ver sus envíos activos\n` +
        `- Llamar al +54 11 4444-3456 para soporte`,
        { agent: 'delivery-tracking', action: 'unrecognized' }
      );
    }
  } catch (err) {
    console.error(`Error handling query from ${phone}:`, err.message);
  }
});

// ── API: Create shipment ──────────────────────────────────────────────
app.post('/api/shipments', async (req, res) => {
  try {
    const shipment = createShipment(req.body);

    // Send initial notification
    const stage = STAGES.order_confirmed;
    await sendText(shipment.customer_phone,
      `${stage.icon} *${stage.label}*\n\n${stage.message(shipment)}`,
      {
        agent: 'delivery-tracking',
        shipment_id: shipment.id,
        order_id: shipment.order_id,
        status: 'order_confirmed',
      }
    );

    await syncMetadata(shipment.customer_phone);
    res.json(shipment);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Update shipment status ───────────────────────────────────────
app.put('/api/shipments/:id/status', async (req, res) => {
  try {
    const { status, ...extras } = req.body;
    const shipment = await updateShipmentStatus(req.params.id, status, extras);
    res.json(shipment);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Get shipment ─────────────────────────────────────────────────
app.get('/api/shipments/:id', (req, res) => {
  const shipment = shipments.get(req.params.id);
  if (!shipment) return res.status(404).json({ error: 'Shipment not found' });
  res.json(shipment);
});

// ── API: List shipments ───────────────────────────────────────────────
app.get('/api/shipments', (req, res) => {
  const { status, phone } = req.query;
  let results = [...shipments.values()];
  if (status) results = results.filter(s => s.status === status);
  if (phone) results = results.filter(s => s.customer_phone === phone);
  res.json({ shipments: results, total: results.length });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...shipments.values()];
  res.json({
    status: 'ok',
    total_shipments: all.length,
    in_transit: all.filter(s => ['picked_up', 'in_transit', 'out_for_delivery'].includes(s.status)).length,
    delivered: all.filter(s => s.status === 'delivered').length,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Delivery tracking bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "delivery-tracking",
  "active_shipments": [
    {
      "id": "SHIP-M1A2B3",
      "order_id": "ORD-5678",
      "status": "in_transit",
      "estimated_delivery": "2026-04-03",
      "items": ["Zapatillas Nike Air", "Remera Adidas"]
    }
  ],
  "recent_delivered": [
    {
      "id": "SHIP-X9Y8Z7",
      "order_id": "ORD-4321",
      "delivered_at": "2026-04-01T14:30:00.000Z"
    }
  ],
  "total_shipments": 5
}
```

Per-message metadata:

```json
{
  "agent": "delivery-tracking",
  "shipment_id": "SHIP-M1A2B3",
  "order_id": "ORD-5678",
  "status": "out_for_delivery",
  "stage_label": "En reparto"
}
```

## How to Run

```bash
# 1. Create the project
mkdir delivery-tracking && cd delivery-tracking
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

# 6. Create a shipment
curl -X POST http://localhost:3000/api/shipments \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "ORD-5678",
    "customer_phone": "5491155559999",
    "customer_name": "María González",
    "items": ["Zapatillas Nike Air", "Remera Adidas"],
    "delivery_address": "Av. Corrientes 1234, CABA",
    "estimated_delivery": "2026-04-03"
  }'

# 7. Update status (simulate delivery flow)
curl -X PUT http://localhost:3000/api/shipments/SHIP-xxx/status \
  -H "Content-Type: application/json" \
  -d '{ "status": "picked_up" }'

curl -X PUT http://localhost:3000/api/shipments/SHIP-xxx/status \
  -H "Content-Type: application/json" \
  -d '{ "status": "in_transit", "current_location": "Centro de distribución Avellaneda" }'

curl -X PUT http://localhost:3000/api/shipments/SHIP-xxx/status \
  -H "Content-Type: application/json" \
  -d '{ "status": "out_for_delivery", "driver_name": "Carlos", "driver_phone": "+5491155550123" }'

curl -X PUT http://localhost:3000/api/shipments/SHIP-xxx/status \
  -H "Content-Type: application/json" \
  -d '{ "status": "delivered", "delivery_proof": "María González" }'
```
