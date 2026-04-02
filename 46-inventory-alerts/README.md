# 46 - Inventory Alerts

Inventory monitoring and alert system via WhatsApp. Tracks stock levels, sends low-stock alerts, handles reorder confirmations, and maintains inventory state in conversation metadata.

## Problem

A retail store manages 500+ SKUs across 3 locations. Staff discovers items are out of stock only when a customer asks. Reordering happens 2-3 days late because the manager checks inventory weekly. Emergency reorders cost 20% more due to rush shipping. The store loses $3,000/month in missed sales from out-of-stock popular items.

## Solution

Deploy an inventory alert system that:
- Monitors stock levels and sends WhatsApp alerts when items fall below reorder thresholds
- Lets managers confirm or skip reorders via simple replies
- Tracks inventory state per manager's conversation metadata
- Supports multiple locations with different thresholds
- Provides on-demand stock checks and daily summaries
- No AI needed -- uses threshold-based alerts and keyword commands

## Architecture

```
+-----------------------+     stock update      +--------------------+
|  POS / Inventory API  | --------------------> |  Your Express App  |
|  (webhook on sale)    |                       |   (this code)      |
+-----------------------+                       +--------------------+
                                                         |
                                    check thresholds     |
                                                         v
                                                +------------------+
                                                |   Opero WPP API  |
                                                | wpp-api.opero.so |
                                                +------------------+
                                                         |
                                          low-stock alert|
                                                         v
                                                +------------------+
                                                |  Store Manager   |
                                                |  (WhatsApp)      |
                                                +------------------+
                                                         |
                                          REORDER / SKIP |
                                                         v
                                                +--------------------+
                                                |  Reorder Recorded  |
                                                |  (metadata updated)|
                                                +--------------------+
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
const MANAGER_PHONE = process.env.MANAGER_PHONE;

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

// -- Inventory state --
function initializeInventory(metadata) {
  return {
    ...metadata,
    role: 'inventory_manager',
    items: metadata.items || {},
    pending_reorders: metadata.pending_reorders || [],
    reorder_history: metadata.reorder_history || [],
    alert_settings: metadata.alert_settings || { enabled: true, quiet_hours: { start: 22, end: 7 } },
    last_daily_summary: metadata.last_daily_summary || null,
  };
}

function checkThreshold(item) {
  if (!item.reorder_threshold) return false;
  return item.quantity <= item.reorder_threshold;
}

function isCritical(item) {
  return item.quantity <= Math.floor((item.reorder_threshold || 0) / 2);
}

// -- Stock update (called by POS integration or admin API) --
async function updateStock(sku, quantityChange, source = 'manual') {
  const metadata = await getConversationMetadata(MANAGER_PHONE);
  const inventory = initializeInventory(metadata);

  const item = inventory.items[sku];
  if (!item) {
    return { error: `SKU ${sku} not found in inventory` };
  }

  const oldQuantity = item.quantity;
  item.quantity += quantityChange;
  if (item.quantity < 0) item.quantity = 0;
  item.last_updated = new Date().toISOString();
  item.last_update_source = source;

  // Check if we crossed the threshold (going down)
  const wasAbove = oldQuantity > (item.reorder_threshold || 0);
  const isBelow = checkThreshold(item);
  const critical = isCritical(item);

  await updateConversationMetadata(MANAGER_PHONE, inventory);

  // Send alert if crossed threshold
  if (isBelow && (wasAbove || critical)) {
    const alertType = critical ? 'CRITICAL' : 'LOW STOCK';
    const alreadyPending = inventory.pending_reorders.some(r => r.sku === sku);

    if (!alreadyPending && inventory.alert_settings.enabled) {
      // Check quiet hours
      const hour = new Date().getHours();
      const quiet = inventory.alert_settings.quiet_hours;
      if (hour >= quiet.start || hour < quiet.end) {
        // Queue for morning
        if (!inventory.queued_alerts) inventory.queued_alerts = [];
        inventory.queued_alerts.push({ sku, type: alertType, quantity: item.quantity });
        await updateConversationMetadata(MANAGER_PHONE, inventory);
      } else {
        inventory.pending_reorders.push({
          sku,
          item_name: item.name,
          current_quantity: item.quantity,
          reorder_quantity: item.reorder_quantity || 0,
          alerted_at: new Date().toISOString(),
          status: 'pending',
        });
        await updateConversationMetadata(MANAGER_PHONE, inventory);

        await sendText(MANAGER_PHONE, [
          `*${alertType} ALERT*`,
          ``,
          `Item: ${item.name} (${sku})`,
          `Current stock: ${item.quantity} units`,
          `Reorder point: ${item.reorder_threshold} units`,
          `Suggested reorder: ${item.reorder_quantity || 'N/A'} units`,
          `Supplier: ${item.supplier || 'N/A'}`,
          ``,
          `Reply *REORDER ${sku}* to place order`,
          `Reply *SKIP ${sku}* to dismiss`,
        ].join('\n'), {
          agent: 'inventory-alerts',
          type: 'low-stock-alert',
          sku,
          quantity: item.quantity,
          critical,
        });
      }
    }
  }

  return {
    sku,
    old_quantity: oldQuantity,
    new_quantity: item.quantity,
    below_threshold: isBelow,
    critical,
  };
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
    const inventory = initializeInventory(metadata);

    // REORDER command
    const reorderMatch = messageText.match(/^REORDER\s+(\S+)(?:\s+(\d+))?$/);
    if (reorderMatch) {
      const sku = reorderMatch[1];
      const quantity = reorderMatch[2] ? parseInt(reorderMatch[2]) : null;
      const pending = inventory.pending_reorders.find(r => r.sku === sku && r.status === 'pending');

      if (pending) {
        pending.status = 'confirmed';
        pending.confirmed_at = new Date().toISOString();
        pending.confirmed_quantity = quantity || pending.reorder_quantity;

        inventory.reorder_history.push({ ...pending });
        inventory.pending_reorders = inventory.pending_reorders.filter(r => !(r.sku === sku && r.status === 'confirmed'));

        await updateConversationMetadata(phone, inventory);
        await sendText(phone,
          `Reorder confirmed for *${pending.item_name}* (${sku}): ${pending.confirmed_quantity} units.\nSupplier: ${inventory.items[sku]?.supplier || 'N/A'}`,
          { agent: 'inventory-alerts', type: 'reorder-confirmed', sku }
        );
      } else {
        await sendText(phone, `No pending alert for SKU ${sku}.`, {
          agent: 'inventory-alerts', type: 'no-pending',
        });
      }
      return;
    }

    // SKIP command
    const skipMatch = messageText.match(/^SKIP\s+(\S+)$/);
    if (skipMatch) {
      const sku = skipMatch[1];
      inventory.pending_reorders = inventory.pending_reorders.filter(r => r.sku !== sku);
      await updateConversationMetadata(phone, inventory);
      await sendText(phone, `Alert dismissed for ${sku}.`, {
        agent: 'inventory-alerts', type: 'alert-dismissed', sku,
      });
      return;
    }

    // CHECK command - check stock for a specific item
    const checkMatch = messageText.match(/^CHECK\s+(\S+)$/);
    if (checkMatch) {
      const sku = checkMatch[1];
      const item = inventory.items[sku];
      if (item) {
        const status = checkThreshold(item) ? (isCritical(item) ? 'CRITICAL' : 'LOW') : 'OK';
        await sendText(phone, [
          `*${item.name}* (${sku})`,
          `Stock: ${item.quantity} units [${status}]`,
          `Reorder point: ${item.reorder_threshold}`,
          `Supplier: ${item.supplier || 'N/A'}`,
          `Last updated: ${item.last_updated || 'N/A'}`,
        ].join('\n'), {
          agent: 'inventory-alerts', type: 'stock-check', sku,
        });
      } else {
        await sendText(phone, `SKU ${sku} not found.`, {
          agent: 'inventory-alerts', type: 'sku-not-found',
        });
      }
      return;
    }

    // SUMMARY command
    if (messageText === 'SUMMARY' || messageText === 'STATUS') {
      const items = Object.entries(inventory.items);
      const lowStock = items.filter(([, item]) => checkThreshold(item));
      const critical = items.filter(([, item]) => isCritical(item));
      const pendingCount = inventory.pending_reorders.filter(r => r.status === 'pending').length;

      let summary = [
        `*Inventory Summary*`,
        ``,
        `Total SKUs: ${items.length}`,
        `Low stock: ${lowStock.length}`,
        `Critical: ${critical.length}`,
        `Pending reorders: ${pendingCount}`,
      ];

      if (lowStock.length > 0) {
        summary.push('', '*Low stock items:*');
        for (const [sku, item] of lowStock) {
          const flag = isCritical(item) ? '!!' : '!';
          summary.push(`${flag} ${item.name} (${sku}): ${item.quantity}/${item.reorder_threshold}`);
        }
      }

      await sendText(phone, summary.join('\n'), {
        agent: 'inventory-alerts', type: 'summary',
        low_stock_count: lowStock.length,
        critical_count: critical.length,
      });
      return;
    }

    // ALERTS ON/OFF
    if (messageText === 'ALERTS ON') {
      inventory.alert_settings.enabled = true;
      await updateConversationMetadata(phone, inventory);
      await sendText(phone, 'Inventory alerts enabled.', { agent: 'inventory-alerts', type: 'alerts-on' });
      return;
    }
    if (messageText === 'ALERTS OFF') {
      inventory.alert_settings.enabled = false;
      await updateConversationMetadata(phone, inventory);
      await sendText(phone, 'Inventory alerts disabled.', { agent: 'inventory-alerts', type: 'alerts-off' });
      return;
    }

    // Help
    await sendText(phone, [
      `*Inventory Alert Commands:*`,
      ``,
      `*REORDER SKU [qty]* - Confirm reorder`,
      `*SKIP SKU* - Dismiss alert`,
      `*CHECK SKU* - Check stock for an item`,
      `*SUMMARY* - Inventory overview`,
      `*ALERTS ON/OFF* - Toggle alerts`,
    ].join('\n'), {
      agent: 'inventory-alerts', type: 'help',
    });
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- API: receive stock updates from POS --
app.post('/api/stock-update', async (req, res) => {
  try {
    const { sku, quantity_change, source } = req.body;
    const result = await updateStock(sku, quantity_change, source || 'pos');
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- API: bulk stock update --
app.post('/api/stock-update/bulk', async (req, res) => {
  try {
    const { updates } = req.body; // [{ sku, quantity_change, source }]
    const results = [];
    for (const update of updates) {
      const result = await updateStock(update.sku, update.quantity_change, update.source || 'bulk');
      results.push(result);
    }
    res.json({ results });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- API: add an inventory item --
app.post('/api/items', async (req, res) => {
  try {
    const { sku, name, quantity, reorder_threshold, reorder_quantity, supplier, category } = req.body;
    const metadata = await getConversationMetadata(MANAGER_PHONE);
    const inventory = initializeInventory(metadata);

    inventory.items[sku] = {
      name,
      quantity,
      reorder_threshold,
      reorder_quantity: reorder_quantity || reorder_threshold * 2,
      supplier: supplier || null,
      category: category || 'general',
      last_updated: new Date().toISOString(),
      created_at: new Date().toISOString(),
    };

    await updateConversationMetadata(MANAGER_PHONE, inventory);
    res.json({ success: true, sku });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Daily summary cron --
async function sendDailySummary() {
  const metadata = await getConversationMetadata(MANAGER_PHONE);
  const inventory = initializeInventory(metadata);

  const items = Object.entries(inventory.items);
  const lowStock = items.filter(([, item]) => checkThreshold(item));

  if (lowStock.length > 0) {
    const lines = lowStock.map(([sku, item]) => {
      const flag = isCritical(item) ? 'CRITICAL' : 'LOW';
      return `[${flag}] ${item.name} (${sku}): ${item.quantity} units`;
    });

    await sendText(MANAGER_PHONE, [
      `*Daily Inventory Report*`,
      `${new Date().toLocaleDateString()}`,
      ``,
      `${lowStock.length} item(s) need attention:`,
      ``,
      ...lines,
      ``,
      `Reply *SUMMARY* for full details.`,
    ].join('\n'), {
      agent: 'inventory-alerts',
      type: 'daily-summary',
      low_stock_count: lowStock.length,
    });
  }

  // Send queued alerts
  if (inventory.queued_alerts?.length > 0) {
    for (const alert of inventory.queued_alerts) {
      const item = inventory.items[alert.sku];
      if (item && checkThreshold(item)) {
        inventory.pending_reorders.push({
          sku: alert.sku,
          item_name: item.name,
          current_quantity: item.quantity,
          reorder_quantity: item.reorder_quantity || 0,
          alerted_at: new Date().toISOString(),
          status: 'pending',
        });
      }
    }
    inventory.queued_alerts = [];
    await updateConversationMetadata(MANAGER_PHONE, inventory);
  }

  inventory.last_daily_summary = new Date().toISOString();
  await updateConversationMetadata(MANAGER_PHONE, inventory);
}

app.post('/admin/daily-summary', async (req, res) => {
  try {
    await sendDailySummary();
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
  console.log(`Inventory alert system listening on port ${PORT}`);
});
```

## Metadata Schema

Manager conversation stores inventory state:

```json
{
  "role": "inventory_manager",
  "items": {
    "MILK-001": {
      "name": "Whole Milk 1L",
      "quantity": 8,
      "reorder_threshold": 15,
      "reorder_quantity": 48,
      "supplier": "Dairy Fresh Co",
      "category": "dairy",
      "last_updated": "2026-04-02T14:00:00.000Z",
      "last_update_source": "pos",
      "created_at": "2026-01-01T09:00:00.000Z"
    }
  },
  "pending_reorders": [
    {
      "sku": "MILK-001",
      "item_name": "Whole Milk 1L",
      "current_quantity": 8,
      "reorder_quantity": 48,
      "alerted_at": "2026-04-02T14:05:00.000Z",
      "status": "pending"
    }
  ],
  "reorder_history": [
    {
      "sku": "BREAD-005",
      "item_name": "White Bread Loaf",
      "confirmed_at": "2026-04-01T09:30:00.000Z",
      "confirmed_quantity": 24,
      "status": "confirmed"
    }
  ],
  "alert_settings": {
    "enabled": true,
    "quiet_hours": { "start": 22, "end": 7 }
  },
  "last_daily_summary": "2026-04-02T08:00:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir inventory-alerts && cd inventory-alerts
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export MANAGER_PHONE="5491155551234"

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

# 6. Add inventory items
curl -X POST http://localhost:3000/api/items \
  -H "Content-Type: application/json" \
  -d '{
    "sku": "MILK-001",
    "name": "Whole Milk 1L",
    "quantity": 50,
    "reorder_threshold": 15,
    "reorder_quantity": 48,
    "supplier": "Dairy Fresh Co",
    "category": "dairy"
  }'

# 7. Simulate a sale (decrease stock)
curl -X POST http://localhost:3000/api/stock-update \
  -H "Content-Type: application/json" \
  -d '{ "sku": "MILK-001", "quantity_change": -42, "source": "pos" }'
```
