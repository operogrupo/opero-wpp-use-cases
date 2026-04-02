# 53 - Document Collection Bot

Document collection bot for applications (loans, insurance, onboarding). Lists required documents, receives photos/PDFs, validates completeness, requests missing items, and tracks submission status per document in conversation metadata.

## Problem

A lending company processes 100 loan applications per month. Each application requires 5-8 documents (ID, proof of income, bank statements, etc.). Applicants submit documents piecemeal via email, often with wrong files, poor quality photos, or missing pages. Staff spends 4 hours daily chasing missing documents. Average time to complete a document set: 12 days. 20% of applicants abandon the process because it's too cumbersome.

## Solution

Deploy a document collection bot that:
- Lists all required documents for each application type
- Receives photos and PDFs directly via WhatsApp
- Validates that a received file matches the expected document type
- Tracks submission status per document (pending, received, approved, rejected)
- Sends reminders for missing documents
- Notifies the reviewer when a complete set is ready
- Provides real-time progress updates to the applicant

## Architecture

```
Applicant sends photo/PDF via WhatsApp
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
| wpp-api.opero.so |                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |  Claude Haiku    |
         |                                 | (validate docs)  |
         |                                 +------------------+
         |                                          |
         |    GET/PUT metadata, POST /messages/text |
         +------------------------------------------+
         |
         v
+------------------+       notification     +------------------+
| Application      | <--------------------- | Reviewer Alert   |
| Reviewer         |                        |                  |
+------------------+                        +------------------+
```

## Code

```javascript
// server.js
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

// -- Config --
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const REVIEWER_PHONE = process.env.REVIEWER_PHONE;

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// -- Application templates --
const APPLICATION_TYPES = {
  personal_loan: {
    name: 'Personal Loan',
    documents: [
      { id: 'gov_id', name: 'Government ID (DNI/Passport)', description: 'Front and back of your ID', required: true, accepts: ['image', 'document'] },
      { id: 'proof_income', name: 'Proof of Income', description: 'Last 3 payslips or tax return', required: true, accepts: ['image', 'document'] },
      { id: 'bank_statements', name: 'Bank Statements', description: 'Last 3 months of bank statements', required: true, accepts: ['document'] },
      { id: 'proof_address', name: 'Proof of Address', description: 'Utility bill or bank letter (less than 3 months old)', required: true, accepts: ['image', 'document'] },
      { id: 'employment_letter', name: 'Employment Letter', description: 'Letter from your employer confirming position and salary', required: true, accepts: ['image', 'document'] },
    ],
  },
  insurance: {
    name: 'Insurance Application',
    documents: [
      { id: 'gov_id', name: 'Government ID', description: 'Front and back of your ID', required: true, accepts: ['image', 'document'] },
      { id: 'medical_history', name: 'Medical History', description: 'Recent medical checkup report', required: true, accepts: ['document'] },
      { id: 'proof_address', name: 'Proof of Address', description: 'Utility bill (less than 3 months old)', required: true, accepts: ['image', 'document'] },
      { id: 'beneficiary_id', name: 'Beneficiary ID', description: 'ID of your designated beneficiary', required: false, accepts: ['image', 'document'] },
    ],
  },
  account_opening: {
    name: 'Account Opening',
    documents: [
      { id: 'gov_id', name: 'Government ID', description: 'Front and back of your ID', required: true, accepts: ['image', 'document'] },
      { id: 'selfie', name: 'Selfie with ID', description: 'Photo of yourself holding your ID', required: true, accepts: ['image'] },
      { id: 'proof_address', name: 'Proof of Address', description: 'Utility bill or bank letter', required: true, accepts: ['image', 'document'] },
      { id: 'tax_id', name: 'Tax ID (CUIT/CUIL)', description: 'Constancia de CUIT/CUIL', required: true, accepts: ['image', 'document'] },
    ],
  },
};

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

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
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

// -- Document status helpers --
function getApplicationProgress(metadata) {
  if (!metadata.documents) return { received: 0, total: 0, pct: 0 };

  const docs = Object.values(metadata.documents);
  const required = docs.filter(d => d.required);
  const received = required.filter(d => d.status === 'received' || d.status === 'approved');

  return {
    received: received.length,
    total: required.length,
    pct: required.length > 0 ? Math.round((received.length / required.length) * 100) : 0,
    all_optional_count: docs.filter(d => !d.required).length,
    complete: received.length === required.length,
  };
}

function getNextMissingDocument(metadata) {
  if (!metadata.documents) return null;
  return Object.values(metadata.documents).find(
    d => d.required && d.status === 'pending'
  );
}

function formatDocumentList(metadata) {
  if (!metadata.documents) return 'No documents configured.';

  return Object.values(metadata.documents)
    .map(doc => {
      const statusIcon = {
        pending: '[ ]',
        received: '[~]',
        approved: '[x]',
        rejected: '[!]',
      }[doc.status] || '[ ]';
      const reqTag = doc.required ? '' : ' (optional)';
      return `${statusIcon} ${doc.name}${reqTag}`;
    })
    .join('\n');
}

// -- AI document validation --
async function validateDocument(messageType, caption, expectedDoc) {
  try {
    const response = await anthropic.messages.create({
      model: 'claude-haiku-4-20250414',
      max_tokens: 200,
      system: 'You validate whether a received WhatsApp message likely contains the expected document type. Respond with JSON only: {"valid": true/false, "confidence": "high/medium/low", "reason": "brief explanation"}',
      messages: [{
        role: 'user',
        content: `Expected document: ${expectedDoc.name} (${expectedDoc.description})\nMessage type: ${messageType}\nCaption/filename: ${caption || 'none'}\nAccepted formats: ${expectedDoc.accepts.join(', ')}\n\nDoes this message type and caption match the expected document?`,
      }],
    });

    const text = response.content[0].text;
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    return jsonMatch ? JSON.parse(jsonMatch[0]) : { valid: true, confidence: 'low', reason: 'Could not validate' };
  } catch {
    return { valid: true, confidence: 'low', reason: 'Validation unavailable' };
  }
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received') return;

  const phone = data.from;
  const messageType = data.type; // text, image, document
  const messageText = messageType === 'text'
    ? (typeof data.content === 'string' ? data.content : data.content?.text || '')
    : '';
  const caption = data.content?.caption || data.content?.filename || '';

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    // No active application
    if (!metadata.application_type) {
      if (messageType === 'text') {
        await sendText(phone, [
          'Welcome! I help collect documents for applications.',
          '',
          'Available application types:',
          ...Object.entries(APPLICATION_TYPES).map(([key, app]) =>
            `- Reply *${key.toUpperCase().replace('_', ' ')}* for ${app.name}`
          ),
        ].join('\n'), {
          agent: 'document-collection', type: 'welcome',
        });
      }
      return;
    }

    // Handle text commands
    if (messageType === 'text' && messageText.trim()) {
      const upper = messageText.trim().toUpperCase();

      // STATUS command
      if (upper === 'STATUS' || upper === 'PROGRESS') {
        const progress = getApplicationProgress(metadata);
        const docList = formatDocumentList(metadata);

        await sendText(phone, [
          `*${metadata.application_name} - Document Status*`,
          `Progress: ${progress.received}/${progress.total} required documents (${progress.pct}%)`,
          ``,
          docList,
          ``,
          `Legend: [x] Approved  [~] Received  [ ] Pending  [!] Rejected`,
          progress.complete
            ? '\nAll required documents received! Under review.'
            : `\nNext: Send your *${getNextMissingDocument(metadata)?.name || 'remaining document'}*`,
        ].join('\n'), {
          agent: 'document-collection', type: 'status',
          progress_pct: progress.pct,
        });
        return;
      }

      // HELP command
      if (upper === 'HELP') {
        const nextDoc = getNextMissingDocument(metadata);
        await sendText(phone, [
          'Send documents as photos or PDF files via WhatsApp.',
          '',
          'Commands:',
          '- *STATUS* - View your document checklist',
          '- *HELP* - Show this help message',
          nextDoc ? `\nPlease send: *${nextDoc.name}*\n${nextDoc.description}` : '',
        ].filter(Boolean).join('\n'), {
          agent: 'document-collection', type: 'help',
        });
        return;
      }

      // Check if starting a new application
      const appKey = messageText.trim().toLowerCase().replace(/\s+/g, '_');
      if (APPLICATION_TYPES[appKey] && !metadata.documents) {
        // This shouldn't happen as application_type is already set
        return;
      }

      // Text message while expecting document
      const nextDoc = getNextMissingDocument(metadata);
      if (nextDoc) {
        await sendText(phone, `I'm expecting a document: *${nextDoc.name}*.\n\n${nextDoc.description}\n\nPlease send it as a photo or PDF file.`, {
          agent: 'document-collection', type: 'expecting-document',
        });
      }
      return;
    }

    // Handle document/image upload
    if (messageType === 'image' || messageType === 'document') {
      await sendTyping(phone);

      const nextDoc = getNextMissingDocument(metadata);
      if (!nextDoc) {
        const progress = getApplicationProgress(metadata);
        if (progress.complete) {
          await sendText(phone, 'All required documents have been received! Your application is under review.', {
            agent: 'document-collection', type: 'all-received',
          });
        } else {
          await sendText(phone, 'Thanks, but all required documents are already submitted. Reply *STATUS* to check.', {
            agent: 'document-collection', type: 'no-pending-docs',
          });
        }
        return;
      }

      // Validate the document
      const validation = await validateDocument(messageType, caption, nextDoc);

      if (!validation.valid && validation.confidence === 'high') {
        await sendText(phone, [
          `This doesn't look like the right document.`,
          ``,
          `Expected: *${nextDoc.name}*`,
          `${nextDoc.description}`,
          ``,
          `Please try again with the correct document.`,
        ].join('\n'), {
          agent: 'document-collection',
          type: 'validation-failed',
          expected_doc: nextDoc.id,
          reason: validation.reason,
        });
        return;
      }

      // Accept the document
      metadata.documents[nextDoc.id].status = 'received';
      metadata.documents[nextDoc.id].received_at = new Date().toISOString();
      metadata.documents[nextDoc.id].message_id = data.wa_message_id || data.id;
      metadata.documents[nextDoc.id].file_type = messageType;
      metadata.documents[nextDoc.id].validation = validation;

      const progress = getApplicationProgress(metadata);
      await updateConversationMetadata(phone, metadata);

      if (progress.complete) {
        // All documents received!
        metadata.application_status = 'documents_complete';
        metadata.completed_at = new Date().toISOString();
        await updateConversationMetadata(phone, metadata);

        await sendText(phone, [
          `*${nextDoc.name}* received!`,
          ``,
          `All required documents are now submitted! (${progress.received}/${progress.total})`,
          ``,
          `Your application is being forwarded for review. We'll notify you of any updates.`,
        ].join('\n'), {
          agent: 'document-collection',
          type: 'application-complete',
          doc_received: nextDoc.id,
        });

        // Notify reviewer
        await sendText(REVIEWER_PHONE, [
          `*COMPLETE APPLICATION*`,
          ``,
          `Applicant: ${metadata.applicant_name} (${phone})`,
          `Type: ${metadata.application_name}`,
          `Documents: ${progress.received}/${progress.total} received`,
          `Submitted: ${new Date().toISOString()}`,
          ``,
          `Ready for review.`,
        ].join('\n'), {
          agent: 'document-collection',
          type: 'reviewer-notification',
          applicant_phone: phone,
        });
      } else {
        // Ask for next document
        const next = getNextMissingDocument(metadata);
        await sendText(phone, [
          `*${nextDoc.name}* received! (${progress.received}/${progress.total})`,
          ``,
          `Next, please send: *${next.name}*`,
          `${next.description}`,
        ].join('\n'), {
          agent: 'document-collection',
          type: 'doc-received',
          doc_received: nextDoc.id,
          progress_pct: progress.pct,
        });
      }
    }
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Sorry, something went wrong. Please try sending the document again.', {
        agent: 'document-collection', error: true, error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback failed:', fallbackErr.message);
    }
  }
});

// -- Admin: start a new application --
app.post('/admin/applications', async (req, res) => {
  try {
    const { phone, applicant_name, application_type } = req.body;
    const template = APPLICATION_TYPES[application_type];

    if (!template) {
      return res.status(400).json({
        error: 'Invalid application type',
        available: Object.keys(APPLICATION_TYPES),
      });
    }

    // Build documents object from template
    const documents = {};
    for (const doc of template.documents) {
      documents[doc.id] = {
        ...doc,
        status: 'pending',
        received_at: null,
        message_id: null,
      };
    }

    const metadata = {
      role: 'applicant',
      applicant_name,
      application_type,
      application_name: template.name,
      application_status: 'collecting_documents',
      documents,
      created_at: new Date().toISOString(),
    };

    await updateConversationMetadata(phone, metadata);

    // Send welcome message with document list
    const docList = template.documents
      .map(d => `${d.required ? '*' : '-'} ${d.name}${d.required ? '' : ' (optional)'}`)
      .join('\n');

    const firstDoc = template.documents.find(d => d.required);

    await sendText(phone, [
      `Hi ${applicant_name}! Welcome to the ${template.name} application process.`,
      ``,
      `We need the following documents:`,
      docList,
      ``,
      `Let's start with: *${firstDoc.name}*`,
      `${firstDoc.description}`,
      ``,
      `Send it as a photo or PDF. Reply *STATUS* anytime to check progress.`,
    ].join('\n'), {
      agent: 'document-collection', type: 'application-started', application_type,
    });

    res.json({ success: true, metadata });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: approve/reject a document --
app.post('/admin/applications/:phone/documents/:docId', async (req, res) => {
  try {
    const { phone, docId } = req.params;
    const { status, reason } = req.body; // status: approved or rejected

    const metadata = await getConversationMetadata(phone);
    if (!metadata.documents || !metadata.documents[docId]) {
      return res.status(404).json({ error: 'Document not found' });
    }

    metadata.documents[docId].status = status;
    metadata.documents[docId].review_date = new Date().toISOString();
    if (reason) metadata.documents[docId].review_reason = reason;

    await updateConversationMetadata(phone, metadata);

    if (status === 'rejected') {
      const doc = metadata.documents[docId];
      await sendText(phone, [
        `Your *${doc.name}* needs to be resubmitted.`,
        reason ? `Reason: ${reason}` : '',
        ``,
        `Please send a new version of this document.`,
      ].filter(Boolean).join('\n'), {
        agent: 'document-collection', type: 'doc-rejected', doc_id: docId,
      });

      // Reset status to pending so it shows as next required
      metadata.documents[docId].status = 'pending';
      await updateConversationMetadata(phone, metadata);
    } else {
      await sendText(phone, `Your *${metadata.documents[docId].name}* has been approved!`, {
        agent: 'document-collection', type: 'doc-approved', doc_id: docId,
      });
    }

    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Reminder cron: remind applicants with pending documents --
async function sendReminders() {
  console.log(`[${new Date().toISOString()}] Sending document reminders...`);
  const conversations = await listConversations();

  for (const conv of conversations) {
    const metadata = conv.metadata;
    if (!metadata?.application_status || metadata.application_status !== 'collecting_documents') continue;

    const phone = conv.phone || conv.jid;
    const progress = getApplicationProgress(metadata);

    if (progress.complete) continue;

    // Only remind if last interaction was >24h ago
    const lastInteraction = metadata.last_reminder || metadata.created_at;
    const hoursSince = (Date.now() - new Date(lastInteraction).getTime()) / (1000 * 60 * 60);

    if (hoursSince < 24) continue;

    const nextDoc = getNextMissingDocument(metadata);
    if (!nextDoc) continue;

    await sendText(phone, [
      `Hi ${metadata.applicant_name}! Just a reminder -- we're still waiting for your *${nextDoc.name}*.`,
      ``,
      `${nextDoc.description}`,
      ``,
      `Progress: ${progress.received}/${progress.total} documents received.`,
      `Reply *STATUS* to see your full checklist.`,
    ].join('\n'), {
      agent: 'document-collection', type: 'reminder', doc: nextDoc.id,
    });

    metadata.last_reminder = new Date().toISOString();
    await updateConversationMetadata(phone, metadata);
  }
}

app.post('/admin/send-reminders', async (req, res) => {
  try {
    await sendReminders();
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
  console.log(`Document collection bot listening on port ${PORT}`);
});
```

## Metadata Schema

Each applicant conversation stores document submission state:

```json
{
  "role": "applicant",
  "applicant_name": "Ana Torres",
  "application_type": "personal_loan",
  "application_name": "Personal Loan",
  "application_status": "collecting_documents",
  "documents": {
    "gov_id": {
      "id": "gov_id",
      "name": "Government ID (DNI/Passport)",
      "description": "Front and back of your ID",
      "required": true,
      "accepts": ["image", "document"],
      "status": "approved",
      "received_at": "2026-04-01T10:00:00.000Z",
      "message_id": "msg_abc123",
      "file_type": "image",
      "review_date": "2026-04-01T11:00:00.000Z"
    },
    "proof_income": {
      "id": "proof_income",
      "name": "Proof of Income",
      "description": "Last 3 payslips or tax return",
      "required": true,
      "accepts": ["image", "document"],
      "status": "received",
      "received_at": "2026-04-02T09:00:00.000Z",
      "message_id": "msg_def456",
      "file_type": "document"
    },
    "bank_statements": {
      "id": "bank_statements",
      "name": "Bank Statements",
      "required": true,
      "status": "pending",
      "received_at": null
    }
  },
  "created_at": "2026-04-01T09:00:00.000Z",
  "last_reminder": "2026-04-02T09:00:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "document-collection",
  "type": "doc-received",
  "doc_received": "proof_income",
  "progress_pct": 40
}
```

## How to Run

```bash
# 1. Create the project
mkdir document-collection && cd document-collection
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export REVIEWER_PHONE="5491155551234"

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

# 6. Start an application for a customer
curl -X POST http://localhost:3000/admin/applications \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155559999",
    "applicant_name": "Ana Torres",
    "application_type": "personal_loan"
  }'

# 7. Approve a submitted document
curl -X POST http://localhost:3000/admin/applications/5491155559999/documents/gov_id \
  -H "Content-Type: application/json" \
  -d '{ "status": "approved" }'

# 8. Send reminders for pending documents
curl -X POST http://localhost:3000/admin/send-reminders
```
