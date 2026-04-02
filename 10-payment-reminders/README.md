# 10 - Payment Reminders

Send payment reminders with invoice details, escalating from friendly reminder to firm reminder to final notice based on metadata tracking which reminders were sent and when.

## Problem

An accounting firm in Tucuman manages billing for 150 small business clients. Every month, 40% of invoices aren't paid on time. The firm's admin assistant manually sends WhatsApp reminders, but there's no system to track who received what, when, and how many times. Some clients get reminded 5 times, others fall through the cracks. Late payments average $2M ARS/month in outstanding receivables, costing the firm in cash flow and admin hours.

## Solution

An automated payment reminder system that:
- Sends reminders on a configurable escalation schedule (Day 1, Day 7, Day 14)
- Escalates tone from friendly to firm to final notice
- Includes formatted invoice details (amount, due date, payment methods)
- Tracks every reminder sent per invoice in conversation metadata
- Recognizes when a client confirms payment and stops reminders
- Provides an overdue dashboard with aging analysis
- No AI needed -- template-based with smart state management

## Architecture

```
+-----------------+     POST /remind         +--------------------+
|  Your Billing   | -----------------------> |  Your Express App  |
|  System / Cron  |                          |   (this code)      |
+-----------------+                          +--------------------+
                                                      |
                                                      v
                                             +------------------+
                                             |   Opero WPP API  |
                                             |  wpp-api.opero.so|
                                             +------------------+
                                                      |
                                                webhook POST
                                                (client replies)
                                                      |
                                                      v
                                             +--------------------+
                                             | Payment Detector   |
                                             | (regex / keywords) |
                                             +--------------------+
                                                      |
                                               Update metadata:
                                               reminder_count,
                                               escalation_level,
                                               payment_status
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

// ── Escalation configuration ─────────────────────────────────────────
const ESCALATION_LEVELS = {
  friendly: {
    level: 1,
    label: 'Recordatorio amigable',
    days_after_due: 1,
    template: (invoice) =>
      `Hola ${invoice.client_name}! Te recordamos que la factura ${invoice.invoice_number} ` +
      `por ${formatCurrency(invoice.amount, invoice.currency)} vencio el ${formatDate(invoice.due_date)}.\n\n` +
      `Si ya realizaste el pago, responde "ya pague" y lo verificamos.\n\n` +
      `Datos para el pago:\n` +
      `${formatPaymentMethods(invoice)}\n\n` +
      `Cualquier consulta, estamos a disposicion.`,
  },
  firm: {
    level: 2,
    label: 'Segundo aviso',
    days_after_due: 7,
    template: (invoice) =>
      `Hola ${invoice.client_name}. Te contactamos nuevamente respecto a la factura ${invoice.invoice_number} ` +
      `por ${formatCurrency(invoice.amount, invoice.currency)}, vencida el ${formatDate(invoice.due_date)}.\n\n` +
      `Llevas ${daysSinceDue(invoice.due_date)} dias de atraso. ` +
      `Te pedimos regularizar la situacion a la brevedad para evitar intereses moratorios.\n\n` +
      `Datos para el pago:\n` +
      `${formatPaymentMethods(invoice)}\n\n` +
      `Si ya pagaste, responde "ya pague". Si necesitas un plan de pago, avisanos.`,
  },
  final: {
    level: 3,
    label: 'Aviso final',
    days_after_due: 14,
    template: (invoice) =>
      `${invoice.client_name}, este es el tercer y ultimo aviso respecto a la factura ${invoice.invoice_number} ` +
      `por ${formatCurrency(invoice.amount, invoice.currency)}.\n\n` +
      `Deuda vencida hace ${daysSinceDue(invoice.due_date)} dias. ` +
      `De no regularizar en las proximas 48 horas, procederemos con las acciones previstas en el contrato ` +
      `(suspension de servicio e inicio de gestion de cobranza).\n\n` +
      `Datos para el pago:\n` +
      `${formatPaymentMethods(invoice)}\n\n` +
      `Para coordinar un plan de pago, responde a este mensaje o llama al (0381) 555-1234.`,
  },
};

// ── Invoice storage (replace with your billing system) ────────────────
const invoices = new Map();

// ── Formatting helpers ────────────────────────────────────────────────
function formatCurrency(amount, currency = 'ARS') {
  if (currency === 'USD') {
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount);
  }
  return new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS', maximumFractionDigits: 0 }).format(amount);
}

function formatDate(dateStr) {
  const date = new Date(dateStr + 'T12:00:00');
  return date.toLocaleDateString('es-AR', {
    day: 'numeric',
    month: 'long',
    year: 'numeric',
  });
}

function daysSinceDue(dueDate) {
  const due = new Date(dueDate + 'T00:00:00');
  const now = new Date();
  return Math.floor((now - due) / 86400000);
}

function formatPaymentMethods(invoice) {
  const methods = [];

  if (invoice.bank_account) {
    methods.push(
      `Transferencia bancaria:\n` +
      `  CBU: ${invoice.bank_account.cbu}\n` +
      `  Alias: ${invoice.bank_account.alias}\n` +
      `  Titular: ${invoice.bank_account.holder}`
    );
  }

  if (invoice.mercadopago_link) {
    methods.push(`MercadoPago: ${invoice.mercadopago_link}`);
  }

  if (invoice.payment_reference) {
    methods.push(`Referencia de pago: ${invoice.payment_reference}`);
  }

  return methods.join('\n\n') || 'Consultanos por los medios de pago disponibles.';
}

// ── Payment confirmation detection ────────────────────────────────────
function detectPaymentConfirmation(text) {
  const patterns = [
    /ya pagu[eé]/i,
    /ya transfer[ií]/i,
    /pago realizado/i,
    /abonado/i,
    /ya deposit[eé]/i,
    /hice (el|la) transferencia/i,
    /te hice la transferencia/i,
    /acabo de pagar/i,
    /ya esta pago/i,
    /pague hoy/i,
  ];

  return patterns.some(p => p.test(text));
}

function detectPaymentPlanRequest(text) {
  const patterns = [
    /plan de pago/i,
    /cuotas/i,
    /pagar en partes/i,
    /puedo pagar.*(despues|mas adelante)/i,
    /no llego/i,
    /no puedo pagar/i,
    /dificultad/i,
  ];

  return patterns.some(p => p.test(text));
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
  if (!res.ok) throw new Error(`Opero API error ${res.status}: ${await res.text()}`);
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

async function listConversations() {
  const res = await operoFetch(`/api/numbers/${NUMBER_ID}/conversations?limit=200`);
  return res.data || [];
}

// ── Send a reminder ───────────────────────────────────────────────────
app.post('/remind', async (req, res) => {
  const { invoice_number, phone, client_name, amount, currency, due_date, bank_account, mercadopago_link, payment_reference, level } = req.body;

  if (!invoice_number || !phone || !amount || !due_date) {
    return res.status(400).json({ error: 'Required: invoice_number, phone, amount, due_date' });
  }

  const invoice = {
    invoice_number,
    phone,
    client_name: client_name || 'Cliente',
    amount,
    currency: currency || 'ARS',
    due_date,
    bank_account,
    mercadopago_link,
    payment_reference,
  };

  // Store invoice for reference
  invoices.set(invoice_number, invoice);

  // Get existing conversation state
  const conversation = await getConversation(phone);
  const meta = conversation?.metadata || {};
  const billing = meta.billing || {};
  const invoiceReminders = billing.invoices || {};
  const invoiceState = invoiceReminders[invoice_number] || {
    reminders_sent: [],
    status: 'pending', // pending | reminded | payment_confirmed | payment_plan | escalated
  };

  // Determine escalation level
  let escalation;
  if (level) {
    escalation = ESCALATION_LEVELS[level];
  } else {
    const daysPastDue = daysSinceDue(due_date);
    if (daysPastDue >= 14) escalation = ESCALATION_LEVELS.final;
    else if (daysPastDue >= 7) escalation = ESCALATION_LEVELS.firm;
    else escalation = ESCALATION_LEVELS.friendly;
  }

  // Don't send if payment already confirmed
  if (invoiceState.status === 'payment_confirmed') {
    return res.json({ message: 'Payment already confirmed for this invoice', skipped: true });
  }

  // Don't exceed 3 reminders
  if (invoiceState.reminders_sent.length >= 3) {
    return res.json({ message: 'Maximum reminders reached for this invoice', skipped: true });
  }

  try {
    // Generate message from template
    const message = escalation.template(invoice);

    // Send the reminder
    await sendText(phone, message, {
      agent: 'payment-reminders',
      type: 'reminder',
      escalation_level: escalation.level,
      escalation_label: escalation.label,
      invoice_number,
      amount,
      currency: currency || 'ARS',
      due_date,
      days_overdue: daysSinceDue(due_date),
    });

    // Update invoice state
    invoiceState.reminders_sent.push({
      level: escalation.level,
      label: escalation.label,
      sent_at: new Date().toISOString(),
      days_overdue: daysSinceDue(due_date),
    });
    invoiceState.status = 'reminded';
    invoiceState.last_reminder_at = new Date().toISOString();
    invoiceState.current_level = escalation.level;

    invoiceReminders[invoice_number] = invoiceState;

    // Update conversation metadata
    await updateConversation(phone, {
      ...meta,
      billing: {
        ...billing,
        invoices: invoiceReminders,
        total_outstanding: Object.values(invoiceReminders)
          .filter(i => i.status !== 'payment_confirmed')
          .length,
        last_reminder_sent: new Date().toISOString(),
        client_name,
      },
    });

    console.log(`[Reminder] Sent ${escalation.label} to ${phone} for ${invoice_number} (${formatCurrency(amount, currency)})`);

    res.json({
      success: true,
      escalation_level: escalation.level,
      escalation_label: escalation.label,
      reminders_total: invoiceState.reminders_sent.length,
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Webhook: handle client replies ────────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';

  if (!messageText.trim()) return;

  try {
    const conversation = await getConversation(phone);
    const meta = conversation?.metadata || {};
    const billing = meta.billing || {};
    const invoiceStates = billing.invoices || {};

    // Find active (non-paid) invoices for this client
    const activeInvoices = Object.entries(invoiceStates)
      .filter(([, state]) => state.status !== 'payment_confirmed');

    if (activeInvoices.length === 0) return; // No pending invoices

    if (detectPaymentConfirmation(messageText)) {
      // Mark all active invoices as payment_confirmed
      // In production, you'd ask which invoice if there are multiple
      for (const [invNum, state] of activeInvoices) {
        state.status = 'payment_confirmed';
        state.confirmed_at = new Date().toISOString();
        state.confirmed_by = 'client_message';
      }

      await updateConversation(phone, {
        ...meta,
        billing: {
          ...billing,
          invoices: invoiceStates,
          total_outstanding: 0,
          last_payment_confirmed: new Date().toISOString(),
        },
      });

      await sendText(phone,
        `Gracias por avisar! Vamos a verificar el pago. ` +
        `Si esta todo en orden, te enviaremos el recibo correspondiente.\n\n` +
        `Cualquier consulta, estamos a disposicion.`,
        {
          agent: 'payment-reminders',
          type: 'payment_acknowledged',
          invoices: activeInvoices.map(([num]) => num),
        }
      );

      console.log(`[Payment] ${phone} confirmed payment for ${activeInvoices.map(([n]) => n).join(', ')}`);
    } else if (detectPaymentPlanRequest(messageText)) {
      for (const [, state] of activeInvoices) {
        state.status = 'payment_plan';
        state.plan_requested_at = new Date().toISOString();
      }

      await updateConversation(phone, {
        ...meta,
        billing: {
          ...billing,
          invoices: invoiceStates,
          payment_plan_requested: true,
        },
      });

      await sendText(phone,
        `Entendemos la situacion. Nuestro equipo de cobranzas se va a comunicar con vos ` +
        `para coordinar un plan de pago. Generalmente podemos ofrecer hasta 3 cuotas.\n\n` +
        `Si preferis llamar directamente: (0381) 555-1234`,
        {
          agent: 'payment-reminders',
          type: 'payment_plan_requested',
          invoices: activeInvoices.map(([num]) => num),
        }
      );

      console.log(`[Payment Plan] ${phone} requested payment plan for ${activeInvoices.map(([n]) => n).join(', ')}`);
    }
  } catch (err) {
    console.error(`Error processing reply from ${phone}:`, err.message);
  }
});

// ── Batch remind: send reminders for all overdue invoices ─────────────
app.post('/remind/batch', async (req, res) => {
  const { invoices: invoiceList } = req.body;

  if (!invoiceList?.length) {
    return res.status(400).json({ error: 'Required: invoices[]' });
  }

  const results = [];
  for (const invoice of invoiceList) {
    try {
      const response = await fetch(`http://localhost:${PORT}/remind`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(invoice),
      });
      const result = await response.json();
      results.push({ invoice_number: invoice.invoice_number, ...result });
    } catch (err) {
      results.push({ invoice_number: invoice.invoice_number, error: err.message });
    }
    // Delay between sends
    await new Promise(r => setTimeout(r, 5000));
  }

  res.json({ results, total: results.length });
});

// ── Auto-reminder cron ────────────────────────────────────────────────
async function autoRemindOverdue() {
  console.log(`[Auto-Remind] Checking for overdue invoices...`);

  try {
    const conversations = await listConversations();

    for (const conv of conversations) {
      const billing = conv.metadata?.billing;
      if (!billing?.invoices) continue;

      for (const [invNum, state] of Object.entries(billing.invoices)) {
        if (state.status === 'payment_confirmed' || state.status === 'payment_plan') continue;
        if (state.reminders_sent?.length >= 3) continue;

        const invoice = invoices.get(invNum);
        if (!invoice) continue;

        const lastReminder = state.last_reminder_at ? new Date(state.last_reminder_at) : null;
        const hoursSinceLastReminder = lastReminder
          ? (Date.now() - lastReminder.getTime()) / 3600000
          : Infinity;

        // Don't send more than one reminder per week
        if (hoursSinceLastReminder < 168) continue;

        const daysPastDue = daysSinceDue(invoice.due_date);

        let level;
        if (daysPastDue >= 14 && state.current_level < 3) level = 'final';
        else if (daysPastDue >= 7 && state.current_level < 2) level = 'firm';
        else continue; // Not time for next escalation yet

        try {
          await fetch(`http://localhost:${PORT}/remind`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ ...invoice, level }),
          });
          console.log(`[Auto-Remind] Sent ${level} reminder for ${invNum}`);
        } catch (err) {
          console.error(`[Auto-Remind] Failed for ${invNum}:`, err.message);
        }

        await new Promise(r => setTimeout(r, 10000)); // 10s between auto-reminders
      }
    }
  } catch (err) {
    console.error(`[Auto-Remind] Error:`, err.message);
  }
}

// Run auto-remind daily at 10am
setInterval(autoRemindOverdue, 24 * 60 * 60 * 1000);

// ── Overdue dashboard ─────────────────────────────────────────────────
app.get('/overdue', async (req, res) => {
  try {
    const conversations = await listConversations();

    const overdue = [];
    let totalOutstanding = 0;

    for (const conv of conversations) {
      const billing = conv.metadata?.billing;
      if (!billing?.invoices) continue;

      for (const [invNum, state] of Object.entries(billing.invoices)) {
        if (state.status === 'payment_confirmed') continue;

        const invoice = invoices.get(invNum);
        const amount = invoice?.amount || 0;
        const dueDate = invoice?.due_date;
        const daysOverdue = dueDate ? daysSinceDue(dueDate) : 0;

        if (daysOverdue <= 0) continue;

        totalOutstanding += amount;

        overdue.push({
          invoice_number: invNum,
          phone: conv.contact_phone,
          client_name: billing.client_name || conv.contact_name,
          amount,
          currency: invoice?.currency || 'ARS',
          due_date: dueDate,
          days_overdue: daysOverdue,
          reminders_sent: state.reminders_sent?.length || 0,
          current_level: state.current_level || 0,
          status: state.status,
          last_reminder: state.last_reminder_at,
        });
      }
    }

    // Aging buckets
    const aging = {
      '1-7 days': overdue.filter(o => o.days_overdue >= 1 && o.days_overdue <= 7),
      '8-14 days': overdue.filter(o => o.days_overdue >= 8 && o.days_overdue <= 14),
      '15-30 days': overdue.filter(o => o.days_overdue >= 15 && o.days_overdue <= 30),
      '30+ days': overdue.filter(o => o.days_overdue > 30),
    };

    res.json({
      total_overdue_invoices: overdue.length,
      total_outstanding: totalOutstanding,
      aging_summary: Object.fromEntries(
        Object.entries(aging).map(([bucket, items]) => [
          bucket,
          { count: items.length, total: items.reduce((s, i) => s + i.amount, 0) },
        ])
      ),
      invoices: overdue.sort((a, b) => b.days_overdue - a.days_overdue),
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', tracked_invoices: invoices.size, uptime: process.uptime() });
});

const PORT = process.env.PORT || 3009;
app.listen(PORT, () => {
  console.log(`Payment reminders listening on port ${PORT}`);
});
```

## Metadata Schema

### Conversation-level metadata

```json
{
  "billing": {
    "client_name": "Distribuidora Norte SRL",
    "total_outstanding": 2,
    "last_reminder_sent": "2026-04-02T10:00:00.000Z",
    "last_payment_confirmed": "2026-03-15T14:00:00.000Z",
    "payment_plan_requested": false,
    "invoices": {
      "FAC-2026-0089": {
        "status": "reminded",
        "current_level": 2,
        "last_reminder_at": "2026-04-02T10:00:00.000Z",
        "reminders_sent": [
          {
            "level": 1,
            "label": "Recordatorio amigable",
            "sent_at": "2026-03-26T10:00:00.000Z",
            "days_overdue": 1
          },
          {
            "level": 2,
            "label": "Segundo aviso",
            "sent_at": "2026-04-02T10:00:00.000Z",
            "days_overdue": 8
          }
        ]
      },
      "FAC-2026-0072": {
        "status": "payment_confirmed",
        "current_level": 1,
        "confirmed_at": "2026-03-15T14:00:00.000Z",
        "confirmed_by": "client_message",
        "reminders_sent": [
          {
            "level": 1,
            "label": "Recordatorio amigable",
            "sent_at": "2026-03-12T10:00:00.000Z",
            "days_overdue": 2
          }
        ]
      }
    }
  }
}
```

### Message-level metadata

```json
{
  "agent": "payment-reminders",
  "type": "reminder",
  "escalation_level": 2,
  "escalation_label": "Segundo aviso",
  "invoice_number": "FAC-2026-0089",
  "amount": 125000,
  "currency": "ARS",
  "due_date": "2026-03-25",
  "days_overdue": 8
}
```

## How to Run

```bash
# 1. Create the project
mkdir payment-reminders && cd payment-reminders
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"

# 3. Start the server
node server.js

# 4. Expose with ngrok
ngrok http 3009

# 5. Register webhook
curl -X POST https://wpp-api.opero.so/api/webhooks \
  -H "Authorization: Bearer $OPERO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ngrok-url.ngrok.io/webhook",
    "events": ["message.received"]
  }'

# 6. Send a single reminder
curl -X POST http://localhost:3009/remind \
  -H "Content-Type: application/json" \
  -d '{
    "invoice_number": "FAC-2026-0089",
    "phone": "5493815551234",
    "client_name": "Distribuidora Norte SRL",
    "amount": 125000,
    "currency": "ARS",
    "due_date": "2026-03-25",
    "bank_account": {
      "cbu": "0000003100000000000001",
      "alias": "estudio.contable.tuc",
      "holder": "Estudio Contable Rodriguez"
    },
    "mercadopago_link": "https://mpago.la/2aB3cD4"
  }'

# 7. Send batch reminders
curl -X POST http://localhost:3009/remind/batch \
  -H "Content-Type: application/json" \
  -d '{
    "invoices": [
      { "invoice_number": "FAC-2026-0089", "phone": "5493815551234", "client_name": "Distribuidora Norte", "amount": 125000, "due_date": "2026-03-25" },
      { "invoice_number": "FAC-2026-0091", "phone": "5493815555678", "client_name": "Ferreteria Central", "amount": 85000, "due_date": "2026-03-28" }
    ]
  }'

# 8. View overdue dashboard
curl http://localhost:3009/overdue
```
