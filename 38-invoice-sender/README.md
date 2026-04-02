# 38 - Invoice Sender

Send invoices via WhatsApp as documents, track delivery status, and send payment reminders for overdue invoices. No AI needed.

## Problem

A small business sends 100+ invoices monthly via email. 25% of invoices go to spam or are never opened. Payment is delayed an average of 15 days past due. Staff spend 10+ hours per month manually following up on overdue invoices. There's no reliable way to know if a client even received their invoice.

## Solution

Deploy a WhatsApp invoice bot that:
- Sends invoices as PDF documents via WhatsApp
- Tracks sent/viewed/paid status in conversation metadata
- Sends automatic payment reminders for overdue invoices
- Allows clients to confirm payment or ask questions
- Provides a dashboard of invoice status and aging reports
- No AI required -- pure business logic

## Architecture

```
+--------------------+    POST /api/invoices/send   +--------------------+
|  Billing System    | --------------------------->  |  Your Express App  |
|  (creates invoice) |                              |   (this code)      |
+--------------------+                              +--------------------+
                                                             |
         +---------------------------------------------------+
         |                      |                            |
         v                      v                            v
+------------------+    Send document            Schedule payment
|   Opero WPP API  |    + summary text          reminders
|  wpp-api.opero.so|                                    |
+------------------+                                    v
         ^                                      +-----------+
         |            webhook                   | Overdue   |
         +---- client responds  -------+        | checker   |
                                       |        | (cron)    |
                                       v        +-----------+
                                Track payment
                                in metadata
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

// ── Data stores ───────────────────────────────────────────────────────
const invoices = new Map();    // invoiceId -> invoice
const clientIndex = new Map(); // phone -> [invoiceIds]

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

async function sendDocument(phone, documentUrl, filename, caption, metadata = {}) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/messages/document`, {
    method: 'POST',
    body: JSON.stringify({
      phone,
      url: documentUrl,
      filename,
      caption,
      metadata,
    }),
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

// ── Invoice management ────────────────────────────────────────────────
function createInvoice(data) {
  const id = data.invoice_id || `INV-${Date.now().toString(36).toUpperCase()}`;
  const invoice = {
    id,
    client_phone: data.client_phone,
    client_name: data.client_name || '',
    items: data.items || [],
    subtotal: data.subtotal || 0,
    tax: data.tax || 0,
    total: data.total || data.subtotal || 0,
    currency: data.currency || 'ARS',
    due_date: data.due_date,
    document_url: data.document_url,   // URL to the PDF
    payment_methods: data.payment_methods || [],
    status: 'created', // created, sent, viewed, paid, overdue, cancelled
    sent_at: null,
    viewed_at: null,
    paid_at: null,
    reminders_sent: 0,
    created_at: new Date().toISOString(),
    notes: data.notes || '',
  };

  invoices.set(id, invoice);

  // Index by client phone
  if (!clientIndex.has(data.client_phone)) {
    clientIndex.set(data.client_phone, []);
  }
  clientIndex.get(data.client_phone).push(id);

  return invoice;
}

function formatCurrency(amount, currency = 'ARS') {
  return `$${amount.toLocaleString('es-AR')} ${currency}`;
}

function getDaysOverdue(dueDate) {
  const due = new Date(dueDate);
  const now = new Date();
  const diff = Math.floor((now - due) / (1000 * 60 * 60 * 24));
  return diff > 0 ? diff : 0;
}

// ── Send invoice ──────────────────────────────────────────────────────
async function sendInvoice(invoiceId) {
  const invoice = invoices.get(invoiceId);
  if (!invoice) throw new Error('Invoice not found');

  const itemList = invoice.items.map(i =>
    `- ${i.description}: ${formatCurrency(i.amount)}`
  ).join('\n');

  const paymentInfo = invoice.payment_methods.length > 0
    ? `\n\n💳 *Formas de pago:*\n${invoice.payment_methods.map(p => `- ${p}`).join('\n')}`
    : '';

  const summary =
    `📄 *Factura ${invoice.id}*\n\n` +
    `Cliente: ${invoice.client_name}\n` +
    `Fecha de emisión: ${new Date().toLocaleDateString('es-AR')}\n` +
    `Vencimiento: *${invoice.due_date}*\n\n` +
    `*Detalle:*\n${itemList}\n\n` +
    `Subtotal: ${formatCurrency(invoice.subtotal)}\n` +
    `IVA: ${formatCurrency(invoice.tax)}\n` +
    `*Total: ${formatCurrency(invoice.total)}*` +
    paymentInfo +
    `\n\n✅ Responda *PAGADO* cuando haya realizado el pago.` +
    `${invoice.notes ? `\n\nNota: ${invoice.notes}` : ''}`;

  // Send PDF document if URL provided
  if (invoice.document_url) {
    await sendDocument(
      invoice.client_phone,
      invoice.document_url,
      `${invoice.id}.pdf`,
      `Factura ${invoice.id} - ${formatCurrency(invoice.total)}`,
      {
        agent: 'invoice-sender',
        invoice_id: invoice.id,
        action: 'document_sent',
        total: invoice.total,
      }
    );
  }

  // Send summary text
  await sendText(invoice.client_phone, summary, {
    agent: 'invoice-sender',
    invoice_id: invoice.id,
    action: 'invoice_sent',
    total: invoice.total,
    due_date: invoice.due_date,
  });

  invoice.status = 'sent';
  invoice.sent_at = new Date().toISOString();

  // Update conversation metadata
  await syncMetadata(invoice.client_phone);

  console.log(`[${new Date().toISOString()}] Invoice ${invoice.id} sent to ${invoice.client_phone}`);
  return invoice;
}

// ── Payment reminders ─────────────────────────────────────────────────
async function sendPaymentReminder(invoiceId) {
  const invoice = invoices.get(invoiceId);
  if (!invoice || invoice.status === 'paid' || invoice.status === 'cancelled') return;

  const daysOverdue = getDaysOverdue(invoice.due_date);
  invoice.reminders_sent++;

  let message;
  if (daysOverdue === 0) {
    message =
      `⏰ *Recordatorio: Factura ${invoice.id}*\n\n` +
      `Hoy vence su factura por *${formatCurrency(invoice.total)}*.\n\n` +
      `Si ya realizó el pago, responda *PAGADO*.\n` +
      `Si tiene alguna consulta, responda *CONSULTA*.`;
  } else if (daysOverdue <= 7) {
    message =
      `⚠️ *Factura vencida: ${invoice.id}*\n\n` +
      `Su factura por *${formatCurrency(invoice.total)}* venció hace ${daysOverdue} día(s) (${invoice.due_date}).\n\n` +
      `Por favor regularice el pago a la brevedad.\n\n` +
      `Si ya pagó, responda *PAGADO*.`;
  } else if (daysOverdue <= 30) {
    message =
      `🚨 *Pago pendiente: ${invoice.id}*\n\n` +
      `Tiene una factura pendiente de *${formatCurrency(invoice.total)}* con ${daysOverdue} días de atraso.\n\n` +
      `Le solicitamos regularizar el pago. Si necesita un plan de pago, responda *PLAN*.\n` +
      `Si ya pagó, responda *PAGADO*.`;
  } else {
    message =
      `🔴 *URGENTE — Factura ${invoice.id}*\n\n` +
      `Su factura por *${formatCurrency(invoice.total)}* tiene ${daysOverdue} días de atraso.\n\n` +
      `Por favor contáctenos al +54 11 4444-1234 para resolver esta situación.\n` +
      `Responda *PAGADO* si ya abonó.`;
  }

  invoice.status = 'overdue';

  await sendText(invoice.client_phone, message, {
    agent: 'invoice-sender',
    invoice_id: invoice.id,
    action: 'payment_reminder',
    days_overdue: daysOverdue,
    reminder_number: invoice.reminders_sent,
    total: invoice.total,
  });

  await syncMetadata(invoice.client_phone);
  console.log(`[${new Date().toISOString()}] Payment reminder #${invoice.reminders_sent} for ${invoice.id} (${daysOverdue} days overdue)`);
}

// ── Check for overdue invoices (run daily) ────────────────────────────
async function checkOverdueInvoices() {
  const today = new Date().toISOString().split('T')[0];

  for (const [id, invoice] of invoices) {
    if (invoice.status === 'paid' || invoice.status === 'cancelled') continue;
    if (invoice.status === 'created') continue; // Not sent yet

    const daysOverdue = getDaysOverdue(invoice.due_date);

    // Send reminders at specific intervals
    const shouldRemind =
      (daysOverdue === 0 && invoice.reminders_sent === 0) ||   // On due date
      (daysOverdue === 3 && invoice.reminders_sent <= 1) ||     // 3 days after
      (daysOverdue === 7 && invoice.reminders_sent <= 2) ||     // 1 week after
      (daysOverdue === 14 && invoice.reminders_sent <= 3) ||    // 2 weeks after
      (daysOverdue === 30 && invoice.reminders_sent <= 4);      // 1 month after

    if (shouldRemind) {
      try {
        await sendPaymentReminder(id);
      } catch (err) {
        console.error(`Error sending reminder for ${id}:`, err.message);
      }
    }
  }
}

// Run daily check
setInterval(checkOverdueInvoices, 24 * 60 * 60 * 1000);

// ── Sync metadata ─────────────────────────────────────────────────────
async function syncMetadata(phone) {
  const invoiceIds = clientIndex.get(phone) || [];
  const clientInvoices = invoiceIds.map(id => invoices.get(id)).filter(Boolean);

  const summary = {
    total_invoices: clientInvoices.length,
    pending: clientInvoices.filter(i => ['sent', 'overdue'].includes(i.status)).length,
    paid: clientInvoices.filter(i => i.status === 'paid').length,
    total_outstanding: clientInvoices
      .filter(i => ['sent', 'overdue'].includes(i.status))
      .reduce((sum, i) => sum + i.total, 0),
    invoices: clientInvoices.map(i => ({
      id: i.id,
      total: i.total,
      due_date: i.due_date,
      status: i.status,
    })),
  };

  await updateConversation(phone, {
    agent: 'invoice-sender',
    billing: summary,
  });
}

// ── Webhook handler (client responses) ────────────────────────────────
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

  const invoiceIds = clientIndex.get(phone) || [];
  if (invoiceIds.length === 0) return;

  console.log(`[${new Date().toISOString()}] Invoice response from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const upper = messageText.toUpperCase();

    // Find the most recent unpaid invoice
    const pendingInvoices = invoiceIds
      .map(id => invoices.get(id))
      .filter(i => i && ['sent', 'overdue'].includes(i.status))
      .sort((a, b) => new Date(b.sent_at) - new Date(a.sent_at));

    if (upper === 'PAGADO') {
      if (pendingInvoices.length === 0) {
        await sendText(phone, 'No tiene facturas pendientes de pago. ¡Gracias!', {
          agent: 'invoice-sender', action: 'no_pending',
        });
        return;
      }

      const invoice = pendingInvoices[0];

      if (pendingInvoices.length > 1) {
        // Multiple pending — ask which one
        const list = pendingInvoices.map((inv, i) =>
          `*${i + 1}* - ${inv.id}: ${formatCurrency(inv.total)} (venc. ${inv.due_date})`
        ).join('\n');

        await sendText(phone, `Tiene ${pendingInvoices.length} facturas pendientes:\n\n${list}\n\n¿Cuál pagó? Responda con el número.`, {
          agent: 'invoice-sender', action: 'select_invoice',
        });
      } else {
        invoice.status = 'paid';
        invoice.paid_at = new Date().toISOString();

        await sendText(phone,
          `✅ *Pago registrado*\n\n` +
          `Factura: ${invoice.id}\n` +
          `Monto: ${formatCurrency(invoice.total)}\n` +
          `Fecha: ${new Date().toLocaleDateString('es-AR')}\n\n` +
          `¡Gracias por su pago!`,
          {
            agent: 'invoice-sender',
            invoice_id: invoice.id,
            action: 'payment_confirmed',
            total: invoice.total,
          }
        );

        // Notify billing team
        const BILLING_PHONE = process.env.BILLING_PHONE;
        if (BILLING_PHONE) {
          await sendText(BILLING_PHONE,
            `💰 Pago reportado: ${invoice.id} - ${formatCurrency(invoice.total)} - ${invoice.client_name} (${phone})`,
            { agent: 'invoice-sender', action: 'billing_notification' }
          );
        }

        await syncMetadata(phone);
      }
    } else if (upper === 'CONSULTA') {
      await sendText(phone,
        `Para consultas sobre facturación:\n\n` +
        `📧 facturacion@example.com\n` +
        `📞 +54 11 4444-1234\n` +
        `🕐 Lunes a Viernes 9:00-18:00\n\n` +
        `O escriba su consulta aquí y le responderemos.`,
        { agent: 'invoice-sender', action: 'billing_query' }
      );
    } else if (upper === 'PLAN') {
      await sendText(phone,
        `Para solicitar un plan de pago, llame al +54 11 4444-1234 o envíe un email a facturacion@example.com indicando el número de factura.`,
        { agent: 'invoice-sender', action: 'plan_requested' }
      );
    } else if (upper === 'FACTURAS' || upper === 'ESTADO') {
      const list = invoiceIds.map(id => {
        const inv = invoices.get(id);
        if (!inv) return null;
        const statusIcon = inv.status === 'paid' ? '✅' : inv.status === 'overdue' ? '🔴' : '🟡';
        return `${statusIcon} ${inv.id}: ${formatCurrency(inv.total)} - ${inv.status === 'paid' ? 'Pagada' : inv.status === 'overdue' ? `Vencida (${getDaysOverdue(inv.due_date)} días)` : `Vence ${inv.due_date}`}`;
      }).filter(Boolean).join('\n');

      await sendText(phone, `*Sus facturas:*\n\n${list}`, {
        agent: 'invoice-sender', action: 'invoices_listed',
      });
    } else {
      // Handle invoice selection by number
      const num = parseInt(messageText, 10);
      if (!isNaN(num) && num >= 1 && num <= pendingInvoices.length) {
        const invoice = pendingInvoices[num - 1];
        invoice.status = 'paid';
        invoice.paid_at = new Date().toISOString();

        await sendText(phone,
          `✅ Pago registrado para factura *${invoice.id}* (${formatCurrency(invoice.total)}). ¡Gracias!`,
          {
            agent: 'invoice-sender',
            invoice_id: invoice.id,
            action: 'payment_confirmed',
            total: invoice.total,
          }
        );

        await syncMetadata(phone);
      }
    }
  } catch (err) {
    console.error(`Error handling response from ${phone}:`, err.message);
  }
});

// ── API: Create and send invoice ──────────────────────────────────────
app.post('/api/invoices/send', async (req, res) => {
  try {
    const invoice = createInvoice(req.body);
    await sendInvoice(invoice.id);
    res.json(invoice);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Create invoice (without sending) ─────────────────────────────
app.post('/api/invoices', (req, res) => {
  try {
    const invoice = createInvoice(req.body);
    res.json(invoice);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Send existing invoice ────────────────────────────────────────
app.post('/api/invoices/:id/send', async (req, res) => {
  try {
    const invoice = await sendInvoice(req.params.id);
    res.json(invoice);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Mark invoice as paid ─────────────────────────────────────────
app.post('/api/invoices/:id/paid', async (req, res) => {
  const invoice = invoices.get(req.params.id);
  if (!invoice) return res.status(404).json({ error: 'Invoice not found' });

  invoice.status = 'paid';
  invoice.paid_at = new Date().toISOString();

  await syncMetadata(invoice.client_phone);
  res.json(invoice);
});

// ── API: Send reminder ────────────────────────────────────────────────
app.post('/api/invoices/:id/remind', async (req, res) => {
  try {
    await sendPaymentReminder(req.params.id);
    res.json({ status: 'sent' });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Trigger overdue check ────────────────────────────────────────
app.post('/api/invoices/check-overdue', async (req, res) => {
  await checkOverdueInvoices();
  res.json({ status: 'checked' });
});

// ── API: Aging report ─────────────────────────────────────────────────
app.get('/api/invoices/aging', (req, res) => {
  const unpaid = [...invoices.values()].filter(i => ['sent', 'overdue'].includes(i.status));

  const aging = {
    current: [],     // Not yet due
    '1-7': [],       // 1-7 days overdue
    '8-14': [],      // 8-14 days overdue
    '15-30': [],     // 15-30 days overdue
    '30+': [],       // 30+ days overdue
  };

  for (const inv of unpaid) {
    const days = getDaysOverdue(inv.due_date);
    const entry = { id: inv.id, client: inv.client_name, total: inv.total, days_overdue: days };

    if (days === 0) aging.current.push(entry);
    else if (days <= 7) aging['1-7'].push(entry);
    else if (days <= 14) aging['8-14'].push(entry);
    else if (days <= 30) aging['15-30'].push(entry);
    else aging['30+'].push(entry);
  }

  const totals = Object.fromEntries(
    Object.entries(aging).map(([k, v]) => [k, v.reduce((sum, i) => sum + i.total, 0)])
  );

  res.json({ aging, totals, total_outstanding: Object.values(totals).reduce((a, b) => a + b, 0) });
});

// ── API: List invoices ────────────────────────────────────────────────
app.get('/api/invoices', (req, res) => {
  const { status, phone } = req.query;
  let results = [...invoices.values()];
  if (status) results = results.filter(i => i.status === status);
  if (phone) results = results.filter(i => i.client_phone === phone);
  res.json({ invoices: results, total: results.length });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...invoices.values()];
  res.json({
    status: 'ok',
    total_invoices: all.length,
    sent: all.filter(i => i.status === 'sent').length,
    paid: all.filter(i => i.status === 'paid').length,
    overdue: all.filter(i => i.status === 'overdue').length,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Invoice sender bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "invoice-sender",
  "billing": {
    "total_invoices": 5,
    "pending": 2,
    "paid": 3,
    "total_outstanding": 45000,
    "invoices": [
      { "id": "INV-M1A2B3", "total": 25000, "due_date": "2026-04-15", "status": "sent" },
      { "id": "INV-M4D5E6", "total": 20000, "due_date": "2026-03-30", "status": "overdue" }
    ]
  }
}
```

Per-message metadata:

```json
{
  "agent": "invoice-sender",
  "invoice_id": "INV-M1A2B3",
  "action": "payment_reminder",
  "days_overdue": 5,
  "reminder_number": 2,
  "total": 25000
}
```

## How to Run

```bash
# 1. Create the project
mkdir invoice-sender && cd invoice-sender
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

# 6. Send an invoice
curl -X POST http://localhost:3000/api/invoices/send \
  -H "Content-Type: application/json" \
  -d '{
    "client_phone": "5491155559999",
    "client_name": "TechCorp SRL",
    "items": [
      { "description": "Desarrollo web - Marzo 2026", "amount": 120000 },
      { "description": "Hosting mensual", "amount": 5000 }
    ],
    "subtotal": 125000,
    "tax": 26250,
    "total": 151250,
    "due_date": "2026-04-15",
    "document_url": "https://your-server.com/invoices/INV-001.pdf",
    "payment_methods": ["Transferencia bancaria", "Mercado Pago"]
  }'

# 7. Check aging report
curl http://localhost:3000/api/invoices/aging
```
