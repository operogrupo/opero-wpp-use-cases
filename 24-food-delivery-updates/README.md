# 24 — Food Delivery Status Updates

## Problem

Customers order food and have no visibility into their delivery status. They call the restaurant repeatedly asking "where's my food?" The restaurant staff is busy cooking and can't handle status calls. Customers need real-time updates and an easy way to check their ETA.

## Solution

Automated delivery status updates via WhatsApp. The system pushes notifications at each stage: order confirmed, preparing, out for delivery, and delivered. Customers can ask for ETA at any point. All delivery metadata (driver, ETA, address) is tracked in conversation metadata.

## Architecture

```
Order placed → System sends confirmation
    |
    v
┌─────────────────────────────────────┐
│   Status Update Pipeline            │
│                                     │
│   1. ORDER_CONFIRMED → WhatsApp     │
│      "Your order #123 is confirmed" │
│                                     │
│   2. PREPARING → WhatsApp           │
│      "Chef is preparing your order" │
│                                     │
│   3. OUT_FOR_DELIVERY → WhatsApp    │
│      "Driver Juan is on the way!"   │
│      + ETA                          │
│                                     │
│   4. DELIVERED → WhatsApp           │
│      "Enjoy your meal!"             │
│      + Rating request               │
└─────────────────────────────────────┘
         ^
         |
Customer can message anytime:
  "Where's my order?" → Current status + ETA
```

## Metadata Schema

```json
{
  "customer": {
    "name": "Laura Perez",
    "address": "Av. Corrientes 1234, CABA"
  },
  "current_order": {
    "id": "ORD-20260402-001",
    "status": "out_for_delivery",
    "items": [
      { "name": "Margherita Pizza", "qty": 1, "price": 15.99 },
      { "name": "Caesar Salad", "qty": 1, "price": 8.99 },
      { "name": "Tiramisu", "qty": 1, "price": 7.50 }
    ],
    "subtotal": 32.48,
    "delivery_fee": 3.99,
    "total": 36.47,
    "payment_method": "card",
    "placed_at": "2026-04-02T19:00:00Z",
    "confirmed_at": "2026-04-02T19:01:00Z",
    "preparing_at": "2026-04-02T19:05:00Z",
    "out_for_delivery_at": "2026-04-02T19:30:00Z",
    "delivered_at": null,
    "delivery": {
      "driver_name": "Juan",
      "driver_phone": "5491155559001",
      "eta_minutes": 15,
      "eta_updated_at": "2026-04-02T19:35:00Z",
      "address": "Av. Corrientes 1234, CABA"
    }
  },
  "order_history": [
    { "id": "ORD-20260325-001", "total": 28.99, "rating": 5, "date": "2026-03-25" }
  ],
  "total_orders": 4,
  "avg_rating_given": 4.5
}
```

## Code

```javascript
// food-delivery-updates.js
const express = require("express");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

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

async function sendLocation(phone, lat, lng, name, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/location`, {
    phone,
    latitude: lat,
    longitude: lng,
    name,
    metadata,
  });
}

// ── Status Messages ────────────────────────────────────────────────────

function formatOrderItems(items) {
  return items.map((i) => `  ${i.qty}x ${i.name} — $${i.price.toFixed(2)}`).join("\n");
}

function orderConfirmedMessage(order, customerName) {
  return [
    `Hi ${customerName}! Your order has been confirmed.`,
    ``,
    `Order #${order.id}`,
    formatOrderItems(order.items),
    ``,
    `  Subtotal: $${order.subtotal.toFixed(2)}`,
    `  Delivery: $${order.delivery_fee.toFixed(2)}`,
    `  *Total: $${order.total.toFixed(2)}*`,
    ``,
    `Estimated delivery: 35-45 minutes`,
    `We'll notify you at every step!`,
  ].join("\n");
}

function preparingMessage(order) {
  return [
    `Your order #${order.id} is being prepared!`,
    ``,
    `Our chef is working on your food now.`,
    `Estimated time until pickup: ~20 minutes.`,
  ].join("\n");
}

function outForDeliveryMessage(order) {
  const driver = order.delivery;
  return [
    `Your order is on its way!`,
    ``,
    `Driver: ${driver.driver_name}`,
    `ETA: ~${driver.eta_minutes} minutes`,
    `Delivering to: ${driver.address}`,
    ``,
    `You can ask "ETA" anytime for an update.`,
  ].join("\n");
}

function deliveredMessage(order) {
  return [
    `Your order #${order.id} has been delivered!`,
    ``,
    `We hope you enjoy your meal!`,
    ``,
    `How was your experience? Rate us 1-5:`,
    `1 - Poor`,
    `2 - Below average`,
    `3 - Average`,
    `4 - Good`,
    `5 - Excellent`,
  ].join("\n");
}

function etaMessage(order) {
  if (!order || order.status === "delivered") {
    return "Your last order has been delivered. Enjoy!";
  }

  const statusMessages = {
    confirmed: `Your order #${order.id} is confirmed and will be sent to the kitchen shortly. Estimated delivery: 35-45 minutes.`,
    preparing: `Your order #${order.id} is being prepared by our chef. Estimated delivery: 20-30 minutes.`,
    out_for_delivery: `Your order #${order.id} is on its way! ${order.delivery?.driver_name} is delivering. ETA: ~${order.delivery?.eta_minutes || "15"} minutes.`,
  };

  return statusMessages[order.status] || `Order #${order.id} status: ${order.status}`;
}

// ── Status Update Endpoint (called by your order system) ───────────────

app.post("/order/status", async (req, res) => {
  const { phone, order_id, status, driver_name, driver_phone, eta_minutes } = req.body;

  if (!phone || !order_id || !status) {
    return res.status(400).json({ error: "phone, order_id, and status are required" });
  }

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const order = metadata.current_order;

    if (!order || order.id !== order_id) {
      return res.status(404).json({ error: "Order not found in conversation" });
    }

    const now = new Date().toISOString();

    switch (status) {
      case "confirmed":
        order.status = "confirmed";
        order.confirmed_at = now;
        await updateConversation(phone, metadata);
        await sendText(phone, orderConfirmedMessage(order, metadata.customer?.name || ""), {
          order_status: "confirmed", order_id,
        });
        break;

      case "preparing":
        order.status = "preparing";
        order.preparing_at = now;
        await updateConversation(phone, metadata);
        await sendText(phone, preparingMessage(order), {
          order_status: "preparing", order_id,
        });
        break;

      case "out_for_delivery":
        order.status = "out_for_delivery";
        order.out_for_delivery_at = now;
        order.delivery = {
          ...order.delivery,
          driver_name: driver_name || order.delivery?.driver_name || "Your driver",
          driver_phone: driver_phone || order.delivery?.driver_phone,
          eta_minutes: eta_minutes || 20,
          eta_updated_at: now,
        };
        await updateConversation(phone, metadata);
        await sendText(phone, outForDeliveryMessage(order), {
          order_status: "out_for_delivery", order_id,
        });
        break;

      case "delivered":
        order.status = "delivered";
        order.delivered_at = now;
        // Move to history
        metadata.order_history = metadata.order_history || [];
        metadata.order_history.push({
          id: order.id,
          total: order.total,
          date: now.split("T")[0],
          items_count: order.items.length,
          rating: null,
        });
        metadata.total_orders = (metadata.total_orders || 0) + 1;
        await updateConversation(phone, metadata);
        await sendText(phone, deliveredMessage(order), {
          order_status: "delivered", order_id,
        });
        break;

      default:
        return res.status(400).json({ error: `Unknown status: ${status}` });
    }

    res.json({ success: true, status: order.status });
  } catch (err) {
    console.error("Status update error:", err);
    res.status(500).json({ error: err.message });
  }
});

// ── Update ETA ─────────────────────────────────────────────────────────

app.post("/order/eta", async (req, res) => {
  const { phone, order_id, eta_minutes } = req.body;

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const order = metadata.current_order;

    if (!order || order.id !== order_id) {
      return res.status(404).json({ error: "Order not found" });
    }

    order.delivery = order.delivery || {};
    order.delivery.eta_minutes = eta_minutes;
    order.delivery.eta_updated_at = new Date().toISOString();

    await updateConversation(phone, metadata);

    // Only notify if ETA changed significantly (> 5 min difference)
    await sendText(phone, `Update: Your estimated delivery time is now ~${eta_minutes} minutes. Thanks for your patience!`, {
      action: "eta_update", eta_minutes,
    });

    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Create New Order ───────────────────────────────────────────────────

app.post("/order/create", async (req, res) => {
  const { phone, customer_name, items, address, delivery_fee = 3.99 } = req.body;

  if (!phone || !items || items.length === 0) {
    return res.status(400).json({ error: "phone and items are required" });
  }

  try {
    const subtotal = items.reduce((sum, i) => sum + i.price * i.qty, 0);
    const total = Math.round((subtotal + delivery_fee) * 100) / 100;
    const orderId = `ORD-${new Date().toISOString().split("T")[0].replace(/-/g, "")}-${Date.now().toString(36).toUpperCase()}`;

    const order = {
      id: orderId,
      status: "confirmed",
      items,
      subtotal: Math.round(subtotal * 100) / 100,
      delivery_fee,
      total,
      payment_method: "card",
      placed_at: new Date().toISOString(),
      confirmed_at: new Date().toISOString(),
      delivery: { address: address || "Address on file" },
    };

    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    metadata.customer = metadata.customer || { name: customer_name || "Valued Customer" };
    metadata.customer.address = address || metadata.customer.address;
    metadata.current_order = order;

    await updateConversation(phone, metadata);

    await sendText(phone, orderConfirmedMessage(order, metadata.customer.name), {
      order_status: "confirmed", order_id: orderId,
    });

    res.json({ success: true, order_id: orderId });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Webhook Handler (customer messages) ────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received" || data.type !== "text") return;

  const phone = data.from;
  const messageText = (typeof data.content === "string" ? data.content : data.content?.text || "").toLowerCase().trim();

  try {
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const order = metadata.current_order;

    // Check for rating (1-5)
    if (order?.status === "delivered" && /^[1-5]$/.test(messageText)) {
      const rating = parseInt(messageText);

      // Update the order history entry
      const historyEntry = (metadata.order_history || []).find((h) => h.id === order.id);
      if (historyEntry) historyEntry.rating = rating;

      // Calculate average rating
      const rated = (metadata.order_history || []).filter((h) => h.rating);
      metadata.avg_rating_given = rated.length > 0
        ? Math.round((rated.reduce((sum, h) => sum + h.rating, 0) / rated.length) * 10) / 10
        : null;

      metadata.current_order = null; // Clear current order
      await updateConversation(phone, metadata);

      const thankYou = rating >= 4
        ? `Thank you for the ${rating}-star rating! We're glad you enjoyed your meal. See you next time!`
        : `Thank you for your feedback. We're sorry it wasn't a perfect experience. We'll work on improving. Your feedback has been shared with our team.`;

      await sendText(phone, thankYou, { action: "order_rated", rating });
      return;
    }

    // ETA check
    if (messageText.includes("eta") || messageText.includes("where") || messageText.includes("status") || messageText.includes("how long") || messageText.includes("when")) {
      await sendText(phone, etaMessage(order), {
        action: "eta_check",
        order_status: order?.status,
      });
      return;
    }

    // Help / general
    if (order && order.status !== "delivered") {
      await sendText(phone, [
        `Your order #${order.id} is currently: *${order.status.replace(/_/g, " ")}*`,
        ``,
        `You can:`,
        `- Ask "ETA" for delivery estimate`,
        `- Ask "status" for current status`,
        ``,
        `For other questions, call us at (555) 345-6789.`,
      ].join("\n"));
    } else {
      await sendText(phone, "Hi! You don't have an active order right now. Place a new order through our app or website!");
    }
  } catch (err) {
    console.error("Delivery webhook error:", err);
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Food delivery updates running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"

# 3. Start the server
node food-delivery-updates.js

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

# 6. Create an order
curl -X POST http://localhost:3000/order/create \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155550001",
    "customer_name": "Laura Perez",
    "address": "Av. Corrientes 1234, CABA",
    "items": [
      { "name": "Margherita Pizza", "qty": 1, "price": 15.99 },
      { "name": "Caesar Salad", "qty": 1, "price": 8.99 }
    ]
  }'

# 7. Update order status
curl -X POST http://localhost:3000/order/status \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155550001",
    "order_id": "ORD-...",
    "status": "preparing"
  }'
```

## Key Concepts

- **Push-based updates**: Status changes are pushed to customers — no polling needed
- **REST API for integration**: Your order system calls `POST /order/status` to trigger WhatsApp notifications
- **ETA on demand**: Customers can ask "where's my food?" at any time
- **No AI required**: Pure event-driven notifications with keyword-based reply handling
- **Rating collection**: After delivery, customers rate 1-5 and the average is tracked
- **Order history**: Past orders are stored in `metadata.order_history` for customer relationship context
