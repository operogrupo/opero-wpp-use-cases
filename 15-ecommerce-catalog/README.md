# 15 — E-commerce Product Catalog Browser

## Problem

Customers want to browse products via WhatsApp — they describe what they want in natural language ("I need a red dress under $50") and expect relevant results with images. Managing a shopping cart, showing product details, and confirming checkout all need to happen inside the chat.

## Solution

An AI-powered product browser using Claude Haiku. Customers describe what they want, the AI searches the catalog, and sends product images with details. Cart state is tracked entirely in conversation metadata — no external database needed for the shopping flow.

## Architecture

```
Customer: "show me running shoes under $100"
    |
    v
┌────────────────────────────────────┐
│   Webhook Server                   │
│                                    │
│   1. Receive message               │
│   2. Claude Haiku classifies       │
│      intent:                       │
│      - browse → search catalog     │
│      - add_to_cart → update cart   │
│      - view_cart → show cart       │
│      - checkout → confirm order    │
│      - question → answer about     │
│        product                     │
│   3. Execute action                │
│   4. Send product images / text    │
│   5. Update cart in metadata       │
└────────────────────────────────────┘
    |
    ├── Browse: Send product images with caption
    ├── Add to cart: Update metadata.cart
    ├── View cart: List items + total
    └── Checkout: Confirm + send payment link
```

## Metadata Schema

```json
{
  "cart": {
    "items": [
      {
        "product_id": "SKU-001",
        "name": "Nike Air Max 90",
        "price": 89.99,
        "quantity": 1,
        "size": "10",
        "color": "black"
      }
    ],
    "subtotal": 89.99,
    "updated_at": "2026-04-02T10:00:00Z"
  },
  "browsing_history": ["running shoes", "sneakers", "nike"],
  "last_viewed_products": ["SKU-001", "SKU-002", "SKU-003"],
  "checkout_status": null,
  "order_id": null
}
```

## Code

```javascript
// ecommerce-catalog.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Product Catalog ────────────────────────────────────────────────────
// In production, replace with a database or API call

const CATALOG = [
  {
    id: "SKU-001",
    name: "Nike Air Max 90",
    category: "running shoes",
    price: 89.99,
    sizes: ["8", "9", "10", "11", "12"],
    colors: ["black", "white", "red"],
    image_url: "https://example.com/images/nike-air-max-90.jpg",
    description: "Classic cushioned running shoe with visible Air unit",
    in_stock: true,
  },
  {
    id: "SKU-002",
    name: "Adidas Ultraboost 23",
    category: "running shoes",
    price: 129.99,
    sizes: ["8", "9", "10", "11"],
    colors: ["black", "grey", "blue"],
    image_url: "https://example.com/images/adidas-ultraboost.jpg",
    description: "Premium responsive running shoe with Boost midsole",
    in_stock: true,
  },
  {
    id: "SKU-003",
    name: "New Balance 574",
    category: "casual sneakers",
    price: 69.99,
    sizes: ["7", "8", "9", "10", "11", "12"],
    colors: ["grey", "navy", "green"],
    image_url: "https://example.com/images/nb-574.jpg",
    description: "Iconic retro lifestyle sneaker with suede and mesh upper",
    in_stock: true,
  },
  {
    id: "SKU-004",
    name: "Converse Chuck Taylor",
    category: "casual sneakers",
    price: 49.99,
    sizes: ["6", "7", "8", "9", "10", "11", "12"],
    colors: ["black", "white", "red", "navy"],
    image_url: "https://example.com/images/converse-chuck.jpg",
    description: "Timeless canvas sneaker, the original basketball shoe",
    in_stock: true,
  },
  {
    id: "SKU-005",
    name: "Puma RS-X",
    category: "casual sneakers",
    price: 79.99,
    sizes: ["8", "9", "10", "11"],
    colors: ["white/blue", "black/red"],
    image_url: "https://example.com/images/puma-rsx.jpg",
    description: "Bold chunky sneaker with retro-futuristic design",
    in_stock: false,
  },
];

// ── API Helpers ────────────────────────────────────────────────────────

async function apiRequest(method, path, body = null) {
  const options = {
    method,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
    },
  };
  if (body) options.body = JSON.stringify(body);

  const res = await fetch(`${API_BASE}${path}`, options);
  return res.json();
}

async function getConversation(phone) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}`);
}

async function updateConversation(phone, metadata) {
  return apiRequest("PUT", `/api/numbers/${NUMBER_ID}/conversations/${phone}`, {
    metadata,
  });
}

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, {
    phone,
    text,
    metadata,
  });
}

async function sendImage(phone, url, caption, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/image`, {
    phone,
    url,
    caption,
    metadata,
  });
}

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, {
    phone,
    type: "composing",
  });
}

// ── Intent Classification ──────────────────────────────────────────────

async function classifyIntent(messageText, cartItems) {
  const cartSummary =
    cartItems.length > 0
      ? `Current cart: ${cartItems.map((i) => `${i.name} x${i.quantity}`).join(", ")}`
      : "Cart is empty";

  const catalogSummary = CATALOG.map(
    (p) =>
      `${p.id}: ${p.name} ($${p.price}) - ${p.category} - colors: ${p.colors.join("/")} - sizes: ${p.sizes.join("/")} - ${p.in_stock ? "in stock" : "out of stock"}`
  ).join("\n");

  const response = await anthropic.messages.create({
    model: "claude-haiku-4-20250514",
    max_tokens: 512,
    system: `You are a shopping assistant. Classify the customer's intent and respond with JSON only.

Available products:
${catalogSummary}

${cartSummary}

Respond with ONLY valid JSON:
{
  "intent": "browse | add_to_cart | remove_from_cart | view_cart | checkout | question",
  "search_query": "string or null — what they're looking for (for browse intent)",
  "product_id": "string or null — SKU if they're referring to a specific product",
  "quantity": 1,
  "size": "string or null",
  "color": "string or null",
  "question_about": "string or null — what they're asking about",
  "response_text": "friendly response to the customer"
}`,
    messages: [{ role: "user", content: messageText }],
  });

  const text = response.content[0].text;
  try {
    return JSON.parse(text);
  } catch {
    const match = text.match(/\{[\s\S]*\}/);
    return match ? JSON.parse(match[0]) : { intent: "question", response_text: text };
  }
}

// ── Search Catalog ─────────────────────────────────────────────────────

function searchCatalog(query, maxResults = 3) {
  const q = query.toLowerCase();
  const terms = q.split(/\s+/);

  return CATALOG.filter((p) => {
    const searchable = `${p.name} ${p.category} ${p.description} ${p.colors.join(" ")}`.toLowerCase();
    return terms.some((term) => searchable.includes(term));
  })
    .filter((p) => p.in_stock)
    .slice(0, maxResults);
}

// ── Cart Operations ────────────────────────────────────────────────────

function addToCart(cart, productId, quantity = 1, size = null, color = null) {
  const product = CATALOG.find((p) => p.id === productId);
  if (!product) return { cart, error: "Product not found" };
  if (!product.in_stock) return { cart, error: "Product is out of stock" };

  const existingIndex = cart.items.findIndex(
    (i) => i.product_id === productId && i.size === size && i.color === color
  );

  if (existingIndex >= 0) {
    cart.items[existingIndex].quantity += quantity;
  } else {
    cart.items.push({
      product_id: productId,
      name: product.name,
      price: product.price,
      quantity,
      size: size || product.sizes[0],
      color: color || product.colors[0],
    });
  }

  cart.subtotal = cart.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  cart.subtotal = Math.round(cart.subtotal * 100) / 100;
  cart.updated_at = new Date().toISOString();

  return { cart, error: null };
}

function removeFromCart(cart, productId) {
  cart.items = cart.items.filter((i) => i.product_id !== productId);
  cart.subtotal = cart.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  cart.subtotal = Math.round(cart.subtotal * 100) / 100;
  cart.updated_at = new Date().toISOString();
  return cart;
}

function formatCart(cart) {
  if (cart.items.length === 0) return "Your cart is empty. Browse our products by telling me what you're looking for!";

  const lines = cart.items.map(
    (i, idx) =>
      `${idx + 1}. ${i.name} (${i.color}, size ${i.size}) x${i.quantity} — $${(i.price * i.quantity).toFixed(2)}`
  );

  lines.push("");
  lines.push(`Subtotal: $${cart.subtotal.toFixed(2)}`);
  lines.push(`Shipping: FREE`);
  lines.push(`Total: $${cart.subtotal.toFixed(2)}`);
  lines.push("");
  lines.push('Type "checkout" to complete your order or keep browsing!');

  return lines.join("\n");
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received") return;
  if (data.type !== "text") return;

  const phone = data.from;
  const messageText =
    typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    await sendTyping(phone);

    // 1. Load conversation state
    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    const cart = metadata.cart || { items: [], subtotal: 0 };
    const browsingHistory = metadata.browsing_history || [];
    const lastViewed = metadata.last_viewed_products || [];

    // 2. Classify intent
    const intent = await classifyIntent(messageText, cart.items);

    // 3. Execute action based on intent
    switch (intent.intent) {
      case "browse": {
        const query = intent.search_query || messageText;
        const results = searchCatalog(query);

        if (results.length === 0) {
          await sendText(phone, `Sorry, I couldn't find anything matching "${query}". Try searching for "sneakers", "running shoes", or browse all products by saying "show me everything".`);
          break;
        }

        // Track browsing
        if (!browsingHistory.includes(query)) browsingHistory.push(query);
        const viewedIds = results.map((p) => p.id);

        // Send each product as an image with caption
        for (const product of results) {
          const caption = [
            `*${product.name}*`,
            `$${product.price.toFixed(2)}`,
            `${product.description}`,
            `Sizes: ${product.sizes.join(", ")}`,
            `Colors: ${product.colors.join(", ")}`,
            ``,
            `Reply "${product.id}" to add to cart`,
          ].join("\n");

          await sendImage(phone, product.image_url, caption, {
            product_id: product.id,
          });
        }

        // Update metadata
        await updateConversation(phone, {
          ...metadata,
          cart,
          browsing_history: browsingHistory,
          last_viewed_products: viewedIds,
        });
        break;
      }

      case "add_to_cart": {
        // Try to find product by ID or from last viewed
        let productId = intent.product_id;
        if (!productId && lastViewed.length > 0) {
          productId = lastViewed[0];
        }

        if (!productId) {
          await sendText(phone, "Which product would you like to add? Send the product code (e.g., SKU-001) or browse first.");
          break;
        }

        const { cart: updatedCart, error } = addToCart(
          cart,
          productId,
          intent.quantity || 1,
          intent.size,
          intent.color
        );

        if (error) {
          await sendText(phone, error);
          break;
        }

        const addedProduct = CATALOG.find((p) => p.id === productId);
        await sendText(
          phone,
          `Added ${addedProduct.name} to your cart! ${intent.response_text || ""}\n\nYour cart now has ${updatedCart.items.length} item(s). Total: $${updatedCart.subtotal.toFixed(2)}\n\nType "cart" to view or "checkout" to buy.`,
          { action: "add_to_cart", product_id: productId }
        );

        await updateConversation(phone, {
          ...metadata,
          cart: updatedCart,
          browsing_history: browsingHistory,
          last_viewed_products: lastViewed,
        });
        break;
      }

      case "remove_from_cart": {
        const productId = intent.product_id;
        if (!productId) {
          await sendText(phone, "Which item do you want to remove? Send the product code.");
          break;
        }

        const updatedCart = removeFromCart(cart, productId);
        await sendText(phone, `Removed from cart.\n\n${formatCart(updatedCart)}`);

        await updateConversation(phone, { ...metadata, cart: updatedCart });
        break;
      }

      case "view_cart": {
        await sendText(phone, formatCart(cart));
        break;
      }

      case "checkout": {
        if (cart.items.length === 0) {
          await sendText(phone, "Your cart is empty! Tell me what you're looking for and I'll help you find it.");
          break;
        }

        const orderSummary = [
          `Order Summary:`,
          ...cart.items.map(
            (i) => `- ${i.name} (${i.color}, size ${i.size}) x${i.quantity}: $${(i.price * i.quantity).toFixed(2)}`
          ),
          ``,
          `Total: $${cart.subtotal.toFixed(2)}`,
          ``,
          `To confirm your order, reply "confirm". To keep shopping, just tell me what else you need!`,
        ].join("\n");

        await sendText(phone, orderSummary, { action: "checkout_initiated" });

        await updateConversation(phone, {
          ...metadata,
          cart,
          checkout_status: "pending_confirmation",
        });
        break;
      }

      default: {
        // General question or conversation
        await sendText(phone, intent.response_text || "I can help you find products! Just tell me what you're looking for, or type 'cart' to see your cart.");
        break;
      }
    }
  } catch (err) {
    console.error("Catalog bot error:", err);
    await sendText(phone, "Oops, something went wrong. Please try again!");
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("E-commerce catalog bot running on :3000"));
```

## How to Run

```bash
# 1. Install dependencies
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-key"

# 3. Start the server
node ecommerce-catalog.js

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
```

## Key Concepts

- **Intent classification**: Claude Haiku determines what the customer wants (browse, add, cart, checkout) from natural language
- **Product images**: Uses `POST /messages/image` to send product photos with captions directly in WhatsApp
- **Cart in metadata**: Shopping cart lives in `metadata.cart` — no external database needed
- **Natural language search**: "show me cheap sneakers" searches by price and category automatically
- **Browsing context**: `last_viewed_products` lets the AI reference recently shown items ("add the first one")
