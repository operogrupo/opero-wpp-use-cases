# 34 - Warranty Claims

Warranty claims handler via WhatsApp. Collect product info, purchase date, problem description, and photos. AI assesses claim validity and routes approved claims to the service team.

## Problem

An electronics retailer processes 200+ warranty claims monthly. Customers call, wait on hold, and explain their issue to multiple people. Claims get lost in email threads, photos arrive in wrong formats, and staff spend hours manually checking if products are still under warranty. 30% of claims are for out-of-warranty products, wasting everyone's time.

## Solution

Deploy a WhatsApp warranty claims bot that:
- Collects product info, serial number, purchase date, and problem description
- Accepts photos of the defect and receipt
- Uses AI to assess if the claim is likely valid (within warranty period, covered defect)
- Automatically rejects clearly invalid claims with explanation
- Routes approved claims to the appropriate service team
- Tracks full claim lifecycle in conversation metadata

## Architecture

```
Customer sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |  Claude Haiku    |
         |                                 |  (assess claim)  |
         |                                 +------------------+
         |                                          |
         |      POST /messages/text                 |
         |      PUT  /conversations/:phone          |
         +------------------------------------------+
                                                    |
                                                    v
                                           +------------------+
                                           | Service Team     |
                                           | (notified via WPP)|
                                           +------------------+
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
const SERVICE_TEAM_PHONE = process.env.SERVICE_TEAM_PHONE || '5491155550000';

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// ── Product warranty database (use a real DB in production) ───────────
const WARRANTY_PERIODS = {
  'smartphone': { months: 12, category: 'electronics' },
  'laptop': { months: 24, category: 'electronics' },
  'tablet': { months: 12, category: 'electronics' },
  'headphones': { months: 6, category: 'accessories' },
  'charger': { months: 6, category: 'accessories' },
  'tv': { months: 24, category: 'appliances' },
  'refrigerator': { months: 36, category: 'appliances' },
  'washing_machine': { months: 24, category: 'appliances' },
  'microwave': { months: 12, category: 'appliances' },
  'other': { months: 12, category: 'general' },
};

// ── Claim state machine ──────────────────────────────────────────────
const CLAIM_STEPS = [
  {
    key: 'product_type',
    question: '¿Qué tipo de producto es?\n\n1. Smartphone\n2. Laptop\n3. Tablet\n4. Auriculares\n5. Cargador\n6. TV\n7. Heladera\n8. Lavarropas\n9. Microondas\n10. Otro',
  },
  {
    key: 'product_brand_model',
    question: '¿Cuál es la marca y modelo del producto? (ej: Samsung Galaxy S24)',
  },
  {
    key: 'serial_number',
    question: '¿Cuál es el número de serie? Lo encontrará en la caja original, en la factura, o en el menú de configuración del dispositivo.',
  },
  {
    key: 'purchase_date',
    question: '¿Cuándo lo compró? (formato: DD/MM/AAAA, ej: 15/03/2025)',
  },
  {
    key: 'problem_description',
    question: 'Describa el problema en detalle. ¿Cuándo empezó? ¿El producto sufrió golpes o contacto con líquidos?',
  },
  {
    key: 'photo',
    question: '📷 Por favor envíe una foto del defecto o problema. Si también tiene la factura de compra, envíela también.\n\n_Si no puede enviar foto ahora, escriba "SIN FOTO"._',
  },
];

const PRODUCT_TYPE_MAP = {
  '1': 'smartphone', '2': 'laptop', '3': 'tablet',
  '4': 'headphones', '5': 'charger', '6': 'tv',
  '7': 'refrigerator', '8': 'washing_machine', '9': 'microwave', '10': 'other',
};

// ── Data stores ───────────────────────────────────────────────────────
const claims = new Map();
const sessions = new Map();

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

// ── AI claim assessment ───────────────────────────────────────────────
async function assessClaim(claimData) {
  const warrantyInfo = WARRANTY_PERIODS[claimData.product_type] || WARRANTY_PERIODS.other;
  const purchaseDate = parseDate(claimData.purchase_date);
  const now = new Date();
  const monthsSincePurchase = purchaseDate
    ? Math.floor((now - purchaseDate) / (1000 * 60 * 60 * 24 * 30))
    : null;
  const withinWarranty = monthsSincePurchase !== null && monthsSincePurchase <= warrantyInfo.months;

  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250414',
    max_tokens: 512,
    system: `You are a warranty claims assessor for an electronics retailer. Analyze the claim and respond in JSON only.

Product warranty period: ${warrantyInfo.months} months
Months since purchase: ${monthsSincePurchase ?? 'unknown'}
Within warranty period: ${withinWarranty}

Not covered:
- Physical damage (drops, cracks caused by user)
- Water/liquid damage (unless product is water-resistant)
- Unauthorized modifications or repairs
- Normal wear and tear
- Cosmetic damage that doesn't affect function

Respond with exactly this JSON:
{
  "decision": "approved" | "rejected" | "needs_review",
  "reason_es": "explanation in Spanish for the customer",
  "internal_notes": "notes for the service team in English",
  "warranty_valid": true/false,
  "estimated_category": "repair" | "replacement" | "refund",
  "priority": "high" | "medium" | "low"
}`,
    messages: [{
      role: 'user',
      content: `Claim details:
Product: ${claimData.product_brand_model} (${claimData.product_type})
Serial: ${claimData.serial_number}
Purchase date: ${claimData.purchase_date} (${monthsSincePurchase} months ago)
Problem: ${claimData.problem_description}
Has photo: ${claimData.has_photo ? 'yes' : 'no'}`,
    }],
  });

  try {
    return JSON.parse(response.content[0].text);
  } catch {
    return {
      decision: 'needs_review',
      reason_es: 'Su reclamo requiere revisión manual por nuestro equipo técnico.',
      internal_notes: 'AI assessment failed to parse',
      warranty_valid: withinWarranty,
      estimated_category: 'repair',
      priority: 'medium',
    };
  }
}

function parseDate(dateStr) {
  if (!dateStr) return null;
  const parts = dateStr.split('/');
  if (parts.length === 3) {
    const [day, month, year] = parts;
    return new Date(`${year}-${month.padStart(2, '0')}-${day.padStart(2, '0')}`);
  }
  return new Date(dateStr);
}

// ── Session management ────────────────────────────────────────────────
function getSession(phone) {
  if (!sessions.has(phone)) {
    sessions.set(phone, {
      step: -1,
      data: {},
      started_at: new Date().toISOString(),
    });
  }
  return sessions.get(phone);
}

// ── Webhook handler ───────────────────────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received') return;

  const phone = data.from;
  const isText = data.type === 'text';
  const isImage = data.type === 'image';

  const messageText = isText
    ? (typeof data.content === 'string' ? data.content : data.content?.text || '').trim()
    : '';

  console.log(`[${new Date().toISOString()}] Claim message from ${phone}: ${isImage ? '[IMAGE]' : messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const session = getSession(phone);

    // Restart command
    if (messageText.toUpperCase() === 'NUEVO') {
      sessions.delete(phone);
      const fresh = getSession(phone);
      fresh.step = 0;
      await sendText(phone,
        '🔧 *Centro de Garantías - Nuevo Reclamo*\n\n' +
        'Vamos a registrar su reclamo de garantía. Le haré algunas preguntas.\n\n' +
        CLAIM_STEPS[0].question,
        { agent: 'warranty-claims', step: 'product_type', action: 'start' }
      );
      return;
    }

    // Check status of existing claim
    if (messageText.toUpperCase() === 'ESTADO') {
      const existingClaims = [...claims.values()].filter(c => c.phone === phone);
      if (existingClaims.length === 0) {
        await sendText(phone, 'No tiene reclamos registrados. Escriba *NUEVO* para iniciar uno.', {
          agent: 'warranty-claims', action: 'no_claims',
        });
      } else {
        const list = existingClaims.map(c =>
          `🔖 *${c.id}*\n  Producto: ${c.data.product_brand_model}\n  Estado: ${c.status}\n  Fecha: ${c.created_at.split('T')[0]}`
        ).join('\n\n');
        await sendText(phone, `*Sus reclamos:*\n\n${list}`, {
          agent: 'warranty-claims', action: 'claims_listed',
        });
      }
      return;
    }

    // First message — start intake
    if (session.step === -1) {
      session.step = 0;
      await sendText(phone,
        '🔧 *Centro de Garantías*\n\n' +
        'Bienvenido/a. Vamos a registrar su reclamo de garantía.\n\n' +
        CLAIM_STEPS[0].question,
        { agent: 'warranty-claims', step: 'product_type', action: 'start' }
      );
      return;
    }

    const currentStep = CLAIM_STEPS[session.step];

    // Handle photo step
    if (currentStep.key === 'photo') {
      if (isImage) {
        session.data.has_photo = true;
        session.data.photo_message_id = data.wa_message_id || data.id;
      } else if (messageText.toUpperCase() === 'SIN FOTO') {
        session.data.has_photo = false;
      } else {
        await sendText(phone, 'Por favor envíe una foto del defecto o escriba *SIN FOTO* para continuar sin foto.', {
          agent: 'warranty-claims', action: 'waiting_photo',
        });
        return;
      }
    } else if (currentStep.key === 'product_type') {
      const productType = PRODUCT_TYPE_MAP[messageText.trim()];
      if (!productType) {
        await sendText(phone, 'Por favor elija un número del 1 al 10.', {
          agent: 'warranty-claims', action: 'invalid_product_type',
        });
        return;
      }
      session.data.product_type = productType;
    } else if (currentStep.key === 'purchase_date') {
      const parsed = parseDate(messageText);
      if (!parsed || isNaN(parsed.getTime())) {
        await sendText(phone, 'Formato de fecha inválido. Use DD/MM/AAAA (ej: 15/03/2025).', {
          agent: 'warranty-claims', action: 'invalid_date',
        });
        return;
      }
      session.data.purchase_date = messageText;
    } else {
      session.data[currentStep.key] = messageText;
    }

    // Advance
    session.step++;

    if (session.step < CLAIM_STEPS.length) {
      const nextStep = CLAIM_STEPS[session.step];
      await sendText(phone, nextStep.question, {
        agent: 'warranty-claims',
        step: nextStep.key,
        progress: `${session.step + 1}/${CLAIM_STEPS.length}`,
      });
    } else {
      // All info collected — assess claim
      await sendText(phone, '⏳ Analizando su reclamo...', {
        agent: 'warranty-claims', action: 'assessing',
      });

      const assessment = await assessClaim(session.data);

      const claimId = `CLM-${Date.now().toString(36).toUpperCase()}`;
      const claim = {
        id: claimId,
        phone,
        data: session.data,
        assessment,
        status: assessment.decision === 'approved' ? 'approved'
          : assessment.decision === 'rejected' ? 'rejected'
          : 'under_review',
        created_at: new Date().toISOString(),
      };
      claims.set(claimId, claim);

      // Notify customer
      if (assessment.decision === 'approved') {
        await sendText(phone,
          `✅ *Reclamo aprobado* (${claimId})\n\n` +
          `${assessment.reason_es}\n\n` +
          `Resolución estimada: *${assessment.estimated_category === 'repair' ? 'Reparación' : assessment.estimated_category === 'replacement' ? 'Reemplazo' : 'Reembolso'}*\n\n` +
          `Nuestro equipo técnico se comunicará con usted en las próximas 48 horas para coordinar.`,
          {
            agent: 'warranty-claims',
            claim_id: claimId,
            action: 'claim_approved',
            resolution: assessment.estimated_category,
          }
        );
      } else if (assessment.decision === 'rejected') {
        await sendText(phone,
          `❌ *Reclamo no aprobado* (${claimId})\n\n` +
          `${assessment.reason_es}\n\n` +
          `Si considera que esta decisión es incorrecta, puede:\n` +
          `- Escribir *APELAR* para solicitar una revisión manual\n` +
          `- Llamar a atención al cliente: +54 11 4444-9999`,
          {
            agent: 'warranty-claims',
            claim_id: claimId,
            action: 'claim_rejected',
          }
        );
      } else {
        await sendText(phone,
          `🔍 *Reclamo en revisión* (${claimId})\n\n` +
          `${assessment.reason_es}\n\n` +
          `Recibirá una respuesta dentro de las próximas 72 horas. Puede consultar el estado escribiendo *ESTADO*.`,
          {
            agent: 'warranty-claims',
            claim_id: claimId,
            action: 'claim_under_review',
          }
        );
      }

      // Notify service team for approved/review claims
      if (assessment.decision !== 'rejected') {
        await sendText(SERVICE_TEAM_PHONE,
          `🔧 *Nuevo reclamo: ${claimId}*\n\n` +
          `Cliente: ${phone}\n` +
          `Producto: ${session.data.product_brand_model} (${session.data.product_type})\n` +
          `Serial: ${session.data.serial_number}\n` +
          `Compra: ${session.data.purchase_date}\n` +
          `Problema: ${session.data.problem_description}\n` +
          `Foto: ${session.data.has_photo ? 'Sí' : 'No'}\n\n` +
          `Decisión AI: ${assessment.decision}\n` +
          `Prioridad: ${assessment.priority}\n` +
          `Notas: ${assessment.internal_notes}`,
          {
            agent: 'warranty-claims',
            claim_id: claimId,
            action: 'service_notification',
            priority: assessment.priority,
          }
        );
      }

      // Store in conversation metadata
      await updateConversation(phone, {
        agent: 'warranty-claims',
        latest_claim: claim,
        total_claims: [...claims.values()].filter(c => c.phone === phone).length,
      });

      sessions.delete(phone);
      console.log(`[${new Date().toISOString()}] Claim ${claimId}: ${assessment.decision}`);
    }
  } catch (err) {
    console.error(`Error in warranty claim from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Disculpe, ocurrió un error. Escriba *NUEVO* para reiniciar o llame al +54 11 4444-9999.', {
        agent: 'warranty-claims', error: true, error_message: err.message,
      });
    } catch (e) {
      console.error('Fallback failed:', e.message);
    }
  }
});

// ── API: Update claim status ──────────────────────────────────────────
app.put('/api/claims/:claimId', async (req, res) => {
  const claim = claims.get(req.params.claimId);
  if (!claim) return res.status(404).json({ error: 'Claim not found' });

  const oldStatus = claim.status;
  claim.status = req.body.status;
  if (req.body.notes) claim.service_notes = req.body.notes;

  // Notify customer of status change
  const statusLabels = {
    approved: 'Aprobado',
    under_review: 'En revisión',
    in_repair: 'En reparación',
    ready: 'Listo para retiro',
    completed: 'Completado',
    rejected: 'No aprobado',
  };

  try {
    await sendText(claim.phone,
      `🔖 *Actualización de reclamo ${claim.id}*\n\n` +
      `Estado: ${statusLabels[oldStatus] || oldStatus} → *${statusLabels[claim.status] || claim.status}*` +
      `${req.body.message ? `\n\n${req.body.message}` : ''}`,
      {
        agent: 'warranty-claims',
        claim_id: claim.id,
        action: 'status_updated',
        old_status: oldStatus,
        new_status: claim.status,
      }
    );
  } catch (err) {
    console.error('Error notifying customer:', err.message);
  }

  res.json(claim);
});

// ── API: List claims ──────────────────────────────────────────────────
app.get('/api/claims', (req, res) => {
  const { status, phone } = req.query;
  let results = [...claims.values()];
  if (status) results = results.filter(c => c.status === status);
  if (phone) results = results.filter(c => c.phone === phone);
  res.json({ claims: results, total: results.length });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  const all = [...claims.values()];
  res.json({
    status: 'ok',
    total_claims: all.length,
    approved: all.filter(c => c.status === 'approved').length,
    rejected: all.filter(c => c.status === 'rejected').length,
    under_review: all.filter(c => c.status === 'under_review').length,
    active_sessions: sessions.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Warranty claims bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "warranty-claims",
  "latest_claim": {
    "id": "CLM-M1A2B3C4",
    "phone": "5491155559999",
    "data": {
      "product_type": "smartphone",
      "product_brand_model": "Samsung Galaxy S24",
      "serial_number": "RF8N30ABCDE",
      "purchase_date": "15/08/2025",
      "problem_description": "La pantalla tiene líneas verdes verticales...",
      "has_photo": true
    },
    "assessment": {
      "decision": "approved",
      "reason_es": "El producto está dentro del período de garantía...",
      "warranty_valid": true,
      "estimated_category": "repair",
      "priority": "high"
    },
    "status": "approved",
    "created_at": "2026-04-02T10:00:00.000Z"
  },
  "total_claims": 1
}
```

Per-message metadata:

```json
{
  "agent": "warranty-claims",
  "claim_id": "CLM-M1A2B3C4",
  "action": "claim_approved",
  "resolution": "repair"
}
```

## How to Run

```bash
# 1. Create the project
mkdir warranty-claims && cd warranty-claims
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export SERVICE_TEAM_PHONE="5491155550000"

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
```
