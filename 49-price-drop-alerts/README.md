# 49 - Price Drop Alerts

E-commerce price drop notification system via WhatsApp. Customers subscribe to products they're watching. When prices drop, they get an instant alert with the new price and purchase link. All watched products tracked in conversation metadata. No AI needed.

## Problem

An e-commerce store has 2,000+ products with prices that change weekly due to promotions, clearances, and competitor price matching. Customers who wanted a product at a lower price have no way to know when prices drop. Email alerts have a 12% open rate and 2% click rate. The store estimates $8,000/month in missed sales from customers who would buy at lower prices but never find out.

## Solution

Deploy a price drop alert system that:
- Lets customers subscribe to products they're watching via WhatsApp
- Tracks watched products and target prices per customer in metadata
- Sends instant alerts when product prices drop
- Includes current price, original price, discount percentage, and direct purchase link
- Supports price target thresholds (notify only if price drops below X)
- Automatically removes products from watchlist after purchase
- Provides on-demand price checks

## Architecture

```
+---------------------+     price update      +--------------------+
|  E-commerce Backend | --------------------> |  Your Express App  |
|  (webhook on price  |                       |   (this code)      |
|   change)           |                       +--------------------+
+---------------------+                               |
                                        check watchlists|
                                                       v
                                              +------------------+
                                              |   Opero WPP API  |
                                              | wpp-api.opero.so |
                                              +------------------+
                                                       |
                                         price alert   |
                                                       v
                                              +------------------+
                                              |    Customer      |
                                              |   (WhatsApp)     |
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
const STORE_URL = process.env.STORE_URL || 'https://store.example.com';

// -- Product catalog (in production, query your e-commerce DB) --
const products = new Map([
  ['SKU-001', { name: 'Sony WH-1000XM5 Headphones', price: 349.99, original_price: 399.99, category: 'Electronics', url: '/products/sony-wh1000xm5', image: 'https://...' }],
  ['SKU-002', { name: 'Nike Air Max 90', price: 129.99, original_price: 129.99, category: 'Shoes', url: '/products/nike-air-max-90', image: 'https://...' }],
  ['SKU-003', { name: 'Kindle Paperwhite', price: 139.99, original_price: 149.99, category: 'Electronics', url: '/products/kindle-paperwhite', image: 'https://...' }],
  ['SKU-004', { name: 'Dyson V15 Vacuum', price: 749.99, original_price: 749.99, category: 'Home', url: '/products/dyson-v15', image: 'https://...' }],
]);

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

// -- Price change handler (called by e-commerce backend) --
async function handlePriceChange(sku, oldPrice, newPrice) {
  if (newPrice >= oldPrice) return { notified: 0 }; // Only alert on drops

  const product = products.get(sku);
  if (!product) return { error: 'Product not found' };

  // Update local catalog
  product.price = newPrice;

  const conversations = await listConversations();
  let notified = 0;

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.watchlist) continue;

    const watched = metadata.watchlist.find(w => w.sku === sku);
    if (!watched) continue;

    // Check if price meets customer's target
    if (watched.target_price && newPrice > watched.target_price) continue;

    const phone = conv.phone || conv.jid;
    const discount = Math.round((1 - newPrice / product.original_price) * 100);
    const savedAmount = (oldPrice - newPrice).toFixed(2);

    await sendText(phone, [
      `*PRICE DROP ALERT*`,
      ``,
      `${product.name}`,
      ``,
      `~$${oldPrice.toFixed(2)}~ -> *$${newPrice.toFixed(2)}*`,
      `You save: $${savedAmount} (${discount}% off)`,
      watched.target_price ? `\nBelow your target of $${watched.target_price}!` : '',
      ``,
      `Buy now: ${STORE_URL}${product.url}`,
      ``,
      `Reply *BUY ${sku}* to remove from watchlist after purchase`,
      `Reply *REMOVE ${sku}* to stop watching this product`,
    ].filter(Boolean).join('\n'), {
      agent: 'price-drop-alerts',
      type: 'price-alert',
      sku,
      old_price: oldPrice,
      new_price: newPrice,
      discount_pct: discount,
    });

    // Record alert in watchlist
    watched.last_alerted = new Date().toISOString();
    watched.last_alerted_price = newPrice;
    watched.alert_count = (watched.alert_count || 0) + 1;
    await updateConversationMetadata(phone, metadata);

    notified++;
  }

  console.log(`Price drop: ${product.name} $${oldPrice} -> $${newPrice}, notified ${notified} watchers`);
  return { sku, product: product.name, old_price: oldPrice, new_price: newPrice, notified };
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

    if (!metadata.watchlist) metadata.watchlist = [];
    if (!metadata.alert_history) metadata.alert_history = [];

    // WATCH command: add product to watchlist
    const watchMatch = messageText.match(/^WATCH\s+(\S+)(?:\s+(\d+(?:\.\d+)?))?$/);
    if (watchMatch) {
      const sku = watchMatch[1];
      const targetPrice = watchMatch[2] ? parseFloat(watchMatch[2]) : null;
      const product = products.get(sku);

      if (!product) {
        await sendText(phone, `Product ${sku} not found. Reply *CATALOG* to browse available products.`, {
          agent: 'price-drop-alerts', type: 'product-not-found',
        });
        return;
      }

      const existing = metadata.watchlist.find(w => w.sku === sku);
      if (existing) {
        if (targetPrice) {
          existing.target_price = targetPrice;
          await updateConversationMetadata(phone, metadata);
          await sendText(phone, `Updated target price for *${product.name}* to $${targetPrice}.`, {
            agent: 'price-drop-alerts', type: 'target-updated', sku,
          });
        } else {
          await sendText(phone, `You're already watching *${product.name}*. Current price: $${product.price}`, {
            agent: 'price-drop-alerts', type: 'already-watching', sku,
          });
        }
        return;
      }

      metadata.watchlist.push({
        sku,
        product_name: product.name,
        price_when_added: product.price,
        target_price: targetPrice,
        added_at: new Date().toISOString(),
        alert_count: 0,
      });

      await updateConversationMetadata(phone, metadata);
      await sendText(phone, [
        `Added *${product.name}* to your watchlist!`,
        ``,
        `Current price: $${product.price}`,
        targetPrice ? `Target price: $${targetPrice}` : 'You\'ll be notified on any price drop.',
        ``,
        `Watching ${metadata.watchlist.length} product(s) total.`,
      ].join('\n'), {
        agent: 'price-drop-alerts', type: 'product-watched', sku,
      });
      return;
    }

    // REMOVE command
    const removeMatch = messageText.match(/^REMOVE\s+(\S+)$/);
    if (removeMatch) {
      const sku = removeMatch[1];
      const idx = metadata.watchlist.findIndex(w => w.sku === sku);

      if (idx === -1) {
        await sendText(phone, `${sku} is not on your watchlist.`, {
          agent: 'price-drop-alerts', type: 'not-on-list',
        });
        return;
      }

      const removed = metadata.watchlist.splice(idx, 1)[0];
      await updateConversationMetadata(phone, metadata);
      await sendText(phone, `Removed *${removed.product_name}* from your watchlist.`, {
        agent: 'price-drop-alerts', type: 'product-removed', sku,
      });
      return;
    }

    // BUY command (remove after purchase)
    const buyMatch = messageText.match(/^BUY\s+(\S+)$/);
    if (buyMatch) {
      const sku = buyMatch[1];
      const idx = metadata.watchlist.findIndex(w => w.sku === sku);

      if (idx >= 0) {
        const item = metadata.watchlist.splice(idx, 1)[0];
        metadata.alert_history.push({
          ...item,
          action: 'purchased',
          purchased_at: new Date().toISOString(),
          final_price: products.get(sku)?.price || 0,
        });
        await updateConversationMetadata(phone, metadata);
      }

      await sendText(phone, `Great purchase! Removed from your watchlist. Enjoy your new ${products.get(sku)?.name || 'item'}!`, {
        agent: 'price-drop-alerts', type: 'purchase-confirmed', sku,
      });
      return;
    }

    // CHECK command: check current price
    const checkMatch = messageText.match(/^CHECK\s+(\S+)$/);
    if (checkMatch) {
      const sku = checkMatch[1];
      const product = products.get(sku);

      if (!product) {
        await sendText(phone, `Product ${sku} not found.`, {
          agent: 'price-drop-alerts', type: 'product-not-found',
        });
        return;
      }

      const discount = product.price < product.original_price
        ? ` (${Math.round((1 - product.price / product.original_price) * 100)}% off)`
        : '';

      const watched = metadata.watchlist.find(w => w.sku === sku);

      await sendText(phone, [
        `*${product.name}* (${sku})`,
        `Current price: $${product.price}${discount}`,
        product.price < product.original_price ? `Original: $${product.original_price}` : '',
        `Category: ${product.category}`,
        watched ? `\nOn your watchlist${watched.target_price ? ` (target: $${watched.target_price})` : ''}` : '\nNot on your watchlist -- reply *WATCH ' + sku + '* to track it',
        ``,
        `${STORE_URL}${product.url}`,
      ].filter(Boolean).join('\n'), {
        agent: 'price-drop-alerts', type: 'price-check', sku, price: product.price,
      });
      return;
    }

    // LIST command: show watchlist
    if (messageText === 'LIST' || messageText === 'WATCHLIST') {
      if (metadata.watchlist.length === 0) {
        await sendText(phone, 'Your watchlist is empty. Reply *WATCH SKU* to start tracking a product.', {
          agent: 'price-drop-alerts', type: 'empty-watchlist',
        });
        return;
      }

      const lines = metadata.watchlist.map(w => {
        const product = products.get(w.sku);
        const currentPrice = product?.price || 'N/A';
        const change = product && w.price_when_added !== product.price
          ? ` (was $${w.price_when_added} when added)`
          : '';
        return `- *${w.product_name}* (${w.sku}): $${currentPrice}${change}${w.target_price ? ` [target: $${w.target_price}]` : ''}`;
      });

      await sendText(phone, `*Your Watchlist* (${metadata.watchlist.length} items)\n\n${lines.join('\n')}\n\nReply *CHECK SKU* to see details.`, {
        agent: 'price-drop-alerts', type: 'watchlist',
      });
      return;
    }

    // CATALOG command
    if (messageText === 'CATALOG' || messageText === 'PRODUCTS') {
      const productList = Array.from(products.entries()).map(([sku, p]) => {
        const discount = p.price < p.original_price
          ? ` (-${Math.round((1 - p.price / p.original_price) * 100)}%)`
          : '';
        return `- ${p.name} (${sku}): $${p.price}${discount}`;
      });

      await sendText(phone, `*Available Products:*\n\n${productList.join('\n')}\n\nReply *WATCH SKU* to track a product.\nReply *WATCH SKU PRICE* to set a target price.`, {
        agent: 'price-drop-alerts', type: 'catalog',
      });
      return;
    }

    // Help
    await sendText(phone, [
      `*Price Drop Alert Commands:*`,
      ``,
      `*WATCH SKU* - Watch a product for price drops`,
      `*WATCH SKU 99.99* - Watch with a target price`,
      `*REMOVE SKU* - Stop watching a product`,
      `*CHECK SKU* - Check current price`,
      `*LIST* - View your watchlist`,
      `*CATALOG* - Browse products`,
      `*BUY SKU* - Mark as purchased & remove`,
    ].join('\n'), {
      agent: 'price-drop-alerts', type: 'help',
    });
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- API: receive price change from e-commerce backend --
app.post('/api/price-change', async (req, res) => {
  try {
    const { sku, old_price, new_price } = req.body;
    const result = await handlePriceChange(sku, old_price, new_price);
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- API: bulk price update --
app.post('/api/price-change/bulk', async (req, res) => {
  try {
    const { changes } = req.body; // [{ sku, old_price, new_price }]
    const results = [];
    for (const change of changes) {
      const result = await handlePriceChange(change.sku, change.old_price, change.new_price);
      results.push(result);
    }
    res.json({ results });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    products_tracked: products.size,
    uptime: process.uptime(),
  });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Price drop alert system listening on port ${PORT}`);
});
```

## Metadata Schema

Each customer conversation stores their watchlist:

```json
{
  "watchlist": [
    {
      "sku": "SKU-001",
      "product_name": "Sony WH-1000XM5 Headphones",
      "price_when_added": 349.99,
      "target_price": 299.99,
      "added_at": "2026-03-20T10:00:00.000Z",
      "alert_count": 2,
      "last_alerted": "2026-04-01T14:00:00.000Z",
      "last_alerted_price": 319.99
    }
  ],
  "alert_history": [
    {
      "sku": "SKU-003",
      "product_name": "Kindle Paperwhite",
      "action": "purchased",
      "purchased_at": "2026-03-25T16:00:00.000Z",
      "final_price": 119.99
    }
  ]
}
```

Outbound message metadata:

```json
{
  "agent": "price-drop-alerts",
  "type": "price-alert",
  "sku": "SKU-001",
  "old_price": 349.99,
  "new_price": 299.99,
  "discount_pct": 25
}
```

## How to Run

```bash
# 1. Create the project
mkdir price-drop-alerts && cd price-drop-alerts
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export STORE_URL="https://store.example.com"

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

# 6. Simulate a price drop (from your e-commerce backend)
curl -X POST http://localhost:3000/api/price-change \
  -H "Content-Type: application/json" \
  -d '{
    "sku": "SKU-001",
    "old_price": 349.99,
    "new_price": 299.99
  }'
```
