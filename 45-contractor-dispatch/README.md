# 45 - Contractor Dispatch System

Automated contractor dispatch via WhatsApp. Receives job requests, matches with available contractors by skill and location, sends job details, tracks acceptance/rejection, and manages scheduling -- all through conversation metadata.

## Problem

A home services company manages 40 contractors (plumbers, electricians, painters, HVAC techs) across a metro area. When a customer calls, the dispatcher manually calls contractors one by one to find someone available. Average dispatch time is 25 minutes. During peak hours, 30% of jobs go unassigned because the dispatcher can't reach contractors fast enough. The company loses $2,000/week in missed jobs.

## Solution

Deploy a contractor dispatch system that:
- Receives job requests via API or WhatsApp from customers
- Automatically matches contractors by skill set, location proximity, and availability
- Sends job details to matched contractors simultaneously
- Tracks accept/reject/timeout responses from each contractor
- Assigns the job to the first contractor who accepts
- Manages contractor profiles, availability, and job history in metadata
- Sends schedule summaries to contractors daily

## Architecture

```
Customer/Dispatcher submits job
         |
         v
+--------------------+     match & dispatch     +-----------------+
|  Your Express App  | -----------------------> | Opero WPP API   |
|   (this code)      |                          | wpp-api.opero.so|
+--------------------+                          +-----------------+
         ^                                              |
         |           webhook POST (contractor replies)  |
         +----------------------------------------------+
         |
         v
+--------------------+
| Contractor Metadata|
| (skill, location,  |
|  availability, etc)|
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
const DISPATCH_TIMEOUT_MS = 10 * 60 * 1000; // 10 minutes to respond

// -- In-memory job queue (use Redis in production) --
const activeJobs = new Map();

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

// -- Contractor matching --
function calculateDistance(lat1, lon1, lat2, lon2) {
  // Haversine formula (returns km)
  const R = 6371;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;
  const a = Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
    Math.sin(dLon / 2) ** 2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

async function findMatchingContractors(job) {
  const conversations = await listConversations();
  const matches = [];

  for (const conv of conversations) {
    const meta = conv.metadata;
    if (!meta?.role || meta.role !== 'contractor') continue;
    if (!meta.available) continue;
    if (!meta.skills?.includes(job.skill_required)) continue;

    // Check if contractor is already on a job
    if (meta.current_job && meta.current_job.status === 'in_progress') continue;

    // Calculate distance if coordinates are available
    let distance = null;
    if (meta.location?.lat && job.location?.lat) {
      distance = calculateDistance(
        meta.location.lat, meta.location.lng,
        job.location.lat, job.location.lng
      );
      // Skip if more than max_radius
      if (meta.max_radius_km && distance > meta.max_radius_km) continue;
    }

    matches.push({
      phone: conv.phone || conv.jid,
      name: meta.name,
      skills: meta.skills,
      rating: meta.rating || 0,
      completed_jobs: meta.completed_jobs || 0,
      distance_km: distance ? Math.round(distance * 10) / 10 : null,
    });
  }

  // Sort by distance first, then by rating
  matches.sort((a, b) => {
    if (a.distance_km !== null && b.distance_km !== null) {
      return a.distance_km - b.distance_km;
    }
    return (b.rating || 0) - (a.rating || 0);
  });

  return matches;
}

// -- Job dispatch --
async function dispatchJob(job) {
  const jobId = `JOB-${Date.now().toString(36).toUpperCase()}`;
  job.id = jobId;
  job.status = 'dispatching';
  job.created_at = new Date().toISOString();
  job.dispatched_to = [];
  job.responses = {};

  const matches = await findMatchingContractors(job);

  if (matches.length === 0) {
    job.status = 'no_match';
    return { jobId, status: 'no_match', message: 'No available contractors found for this job.' };
  }

  // Dispatch to top 5 contractors
  const dispatched = matches.slice(0, 5);
  job.dispatched_to = dispatched.map(c => c.phone);

  const jobMessage = [
    `*NEW JOB REQUEST* (${jobId})`,
    ``,
    `Service: ${job.skill_required}`,
    `Description: ${job.description}`,
    `Address: ${job.address}`,
    `Customer: ${job.customer_name}`,
    `Scheduled: ${job.scheduled_time || 'ASAP'}`,
    `Estimated pay: $${job.estimated_pay}`,
    ``,
    `Reply *ACCEPT* to take this job or *DECLINE* to pass.`,
    `You have 10 minutes to respond.`,
  ].join('\n');

  for (const contractor of dispatched) {
    await sendText(contractor.phone, jobMessage, {
      agent: 'contractor-dispatch',
      type: 'job-offer',
      job_id: jobId,
    });

    // Update contractor metadata with pending job
    const meta = await getConversationMetadata(contractor.phone);
    meta.pending_job = { id: jobId, offered_at: new Date().toISOString() };
    await updateConversationMetadata(contractor.phone, meta);
  }

  // Store job in active jobs
  activeJobs.set(jobId, job);

  // Set timeout for job
  setTimeout(async () => {
    const currentJob = activeJobs.get(jobId);
    if (currentJob && currentJob.status === 'dispatching') {
      currentJob.status = 'timed_out';
      // Notify unresponded contractors
      for (const phone of currentJob.dispatched_to) {
        if (!currentJob.responses[phone]) {
          const meta = await getConversationMetadata(phone);
          if (meta.pending_job?.id === jobId) {
            delete meta.pending_job;
            await updateConversationMetadata(phone, meta);
          }
        }
      }
      console.log(`Job ${jobId} timed out with no acceptance`);
    }
  }, DISPATCH_TIMEOUT_MS);

  return {
    jobId,
    status: 'dispatching',
    dispatched_to: dispatched.length,
    contractors: dispatched.map(c => ({ name: c.name, distance: c.distance_km })),
  };
}

// -- Handle contractor response --
async function handleContractorResponse(phone, response) {
  const metadata = await getConversationMetadata(phone);
  if (!metadata.pending_job) {
    await sendText(phone, 'You have no pending job offers right now.', {
      agent: 'contractor-dispatch', type: 'no-pending-job',
    });
    return;
  }

  const jobId = metadata.pending_job.id;
  const job = activeJobs.get(jobId);

  if (!job) {
    delete metadata.pending_job;
    await updateConversationMetadata(phone, metadata);
    await sendText(phone, 'That job offer has expired.', {
      agent: 'contractor-dispatch', type: 'job-expired',
    });
    return;
  }

  const normalizedResponse = response.toUpperCase().trim();

  if (normalizedResponse === 'ACCEPT' || normalizedResponse === 'YES') {
    if (job.status === 'assigned') {
      // Already assigned to someone else
      delete metadata.pending_job;
      await updateConversationMetadata(phone, metadata);
      await sendText(phone, `Sorry, job ${jobId} was already taken by another contractor.`, {
        agent: 'contractor-dispatch', type: 'job-taken',
      });
      return;
    }

    // Assign job to this contractor
    job.status = 'assigned';
    job.assigned_to = phone;
    job.assigned_at = new Date().toISOString();
    job.responses[phone] = 'accepted';

    // Update contractor metadata
    metadata.current_job = {
      id: jobId,
      status: 'in_progress',
      skill: job.skill_required,
      address: job.address,
      customer: job.customer_name,
      customer_phone: job.customer_phone,
      accepted_at: new Date().toISOString(),
    };
    delete metadata.pending_job;
    if (!metadata.job_history) metadata.job_history = [];
    await updateConversationMetadata(phone, metadata);

    // Confirm to contractor
    await sendText(phone, [
      `Job ${jobId} confirmed! Here are the details:`,
      ``,
      `Customer: ${job.customer_name}`,
      `Phone: ${job.customer_phone}`,
      `Address: ${job.address}`,
      `Service: ${job.skill_required}`,
      `Description: ${job.description}`,
      `Scheduled: ${job.scheduled_time || 'ASAP'}`,
      ``,
      `Reply *ARRIVED* when you get there, *DONE* when finished, or *ISSUE* if there's a problem.`,
    ].join('\n'), {
      agent: 'contractor-dispatch', type: 'job-confirmed', job_id: jobId,
    });

    // Notify other contractors that job is taken
    for (const otherPhone of job.dispatched_to) {
      if (otherPhone !== phone && !job.responses[otherPhone]) {
        const otherMeta = await getConversationMetadata(otherPhone);
        if (otherMeta.pending_job?.id === jobId) {
          delete otherMeta.pending_job;
          await updateConversationMetadata(otherPhone, otherMeta);
        }
        await sendText(otherPhone, `Job ${jobId} has been assigned to another contractor.`, {
          agent: 'contractor-dispatch', type: 'job-taken',
        });
      }
    }

    // Notify customer
    if (job.customer_phone) {
      await sendText(job.customer_phone,
        `Great news! ${metadata.name} has been assigned to your ${job.skill_required} request. They'll contact you shortly.`,
        { agent: 'contractor-dispatch', type: 'customer-assigned' }
      );
    }

    console.log(`Job ${jobId} assigned to ${metadata.name} (${phone})`);
  } else if (normalizedResponse === 'DECLINE' || normalizedResponse === 'NO') {
    job.responses[phone] = 'declined';
    delete metadata.pending_job;
    await updateConversationMetadata(phone, metadata);

    await sendText(phone, `No problem. You've declined job ${jobId}.`, {
      agent: 'contractor-dispatch', type: 'job-declined', job_id: jobId,
    });

    console.log(`${metadata.name} declined job ${jobId}`);
  }
}

// -- Handle job status updates from contractor --
async function handleJobStatusUpdate(phone, status) {
  const metadata = await getConversationMetadata(phone);
  if (!metadata.current_job) {
    await sendText(phone, 'You don\'t have an active job right now.', {
      agent: 'contractor-dispatch', type: 'no-active-job',
    });
    return;
  }

  const jobId = metadata.current_job.id;
  const normalizedStatus = status.toUpperCase().trim();

  if (normalizedStatus === 'ARRIVED') {
    metadata.current_job.arrived_at = new Date().toISOString();
    metadata.current_job.status = 'on_site';
    await updateConversationMetadata(phone, metadata);

    await sendText(phone, `Noted -- you're on site for job ${jobId}. Reply *DONE* when finished.`, {
      agent: 'contractor-dispatch', type: 'status-arrived', job_id: jobId,
    });
  } else if (normalizedStatus === 'DONE' || normalizedStatus === 'FINISHED') {
    metadata.current_job.completed_at = new Date().toISOString();
    metadata.current_job.status = 'completed';

    // Move to history
    if (!metadata.job_history) metadata.job_history = [];
    metadata.job_history.push({ ...metadata.current_job });
    metadata.completed_jobs = (metadata.completed_jobs || 0) + 1;
    delete metadata.current_job;
    await updateConversationMetadata(phone, metadata);

    await sendText(phone, `Job ${jobId} marked as complete. Great work, ${metadata.name}!`, {
      agent: 'contractor-dispatch', type: 'job-completed', job_id: jobId,
    });

    // Update active job
    const job = activeJobs.get(jobId);
    if (job) {
      job.status = 'completed';
      job.completed_at = new Date().toISOString();
    }
  } else if (normalizedStatus === 'ISSUE') {
    metadata.current_job.status = 'issue_reported';
    await updateConversationMetadata(phone, metadata);

    await sendText(phone, 'An issue has been flagged. A dispatcher will follow up shortly. Please describe the issue in your next message.', {
      agent: 'contractor-dispatch', type: 'issue-reported', job_id: jobId,
    });
  }
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = (typeof data.content === 'string'
    ? data.content
    : data.content?.text || '').trim();
  if (!messageText) return;

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    const metadata = await getConversationMetadata(phone);

    if (metadata.role !== 'contractor') {
      // Customer message -- could be a job request
      await sendText(phone, 'To submit a job request, please contact our dispatch team or use our website.', {
        agent: 'contractor-dispatch', type: 'customer-redirect',
      });
      return;
    }

    const upper = messageText.toUpperCase();

    // Check for job response
    if (upper === 'ACCEPT' || upper === 'YES' || upper === 'DECLINE' || upper === 'NO') {
      await handleContractorResponse(phone, messageText);
    }
    // Check for job status updates
    else if (['ARRIVED', 'DONE', 'FINISHED', 'ISSUE'].includes(upper)) {
      await handleJobStatusUpdate(phone, messageText);
    }
    // Show status
    else if (upper === 'STATUS') {
      const status = metadata.current_job
        ? `Active job: ${metadata.current_job.id}\nStatus: ${metadata.current_job.status}\nAddress: ${metadata.current_job.address}`
        : 'No active jobs. You\'re available for new assignments.';

      await sendText(phone, `${metadata.name}'s status:\n\n${status}\n\nJobs completed: ${metadata.completed_jobs || 0}\nRating: ${metadata.rating || 'N/A'}`, {
        agent: 'contractor-dispatch', type: 'status-check',
      });
    }
    // Toggle availability
    else if (upper === 'AVAILABLE' || upper === 'ON') {
      metadata.available = true;
      await updateConversationMetadata(phone, metadata);
      await sendText(phone, 'You\'re now marked as available. You\'ll receive job offers.', {
        agent: 'contractor-dispatch', type: 'availability-on',
      });
    } else if (upper === 'UNAVAILABLE' || upper === 'OFF') {
      metadata.available = false;
      await updateConversationMetadata(phone, metadata);
      await sendText(phone, 'You\'re now marked as unavailable. No new jobs will be sent.', {
        agent: 'contractor-dispatch', type: 'availability-off',
      });
    } else {
      await sendText(phone, `Hi ${metadata.name}! Commands:\n- *ACCEPT/DECLINE* - Respond to a job offer\n- *ARRIVED/DONE/ISSUE* - Update job status\n- *STATUS* - View your current status\n- *AVAILABLE/UNAVAILABLE* - Toggle availability`, {
        agent: 'contractor-dispatch', type: 'help',
      });
    }
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// -- Admin: register a contractor --
app.post('/admin/contractors', async (req, res) => {
  try {
    const { phone, name, skills, location, max_radius_km } = req.body;
    const metadata = {
      role: 'contractor',
      name,
      skills,
      location,
      max_radius_km: max_radius_km || 30,
      available: true,
      rating: 5.0,
      completed_jobs: 0,
      job_history: [],
      registered_at: new Date().toISOString(),
    };

    await updateConversationMetadata(phone, metadata);
    await sendText(phone,
      `Welcome ${name}! You've been registered as a contractor.\n\nSkills: ${skills.join(', ')}\nMax radius: ${max_radius_km || 30}km\n\nYou'll receive job offers via WhatsApp. Reply *AVAILABLE* or *UNAVAILABLE* to toggle.`,
      { agent: 'contractor-dispatch', type: 'registration' }
    );

    res.json({ success: true, metadata });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: dispatch a new job --
app.post('/admin/jobs', async (req, res) => {
  try {
    const result = await dispatchJob(req.body);
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -- Admin: view active jobs --
app.get('/admin/jobs', (req, res) => {
  const jobs = Array.from(activeJobs.entries()).map(([id, job]) => ({
    id,
    status: job.status,
    skill: job.skill_required,
    dispatched_to: job.dispatched_to.length,
    assigned_to: job.assigned_to || null,
    created_at: job.created_at,
  }));
  res.json({ jobs });
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    active_jobs: activeJobs.size,
    uptime: process.uptime(),
  });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Contractor dispatch system listening on port ${PORT}`);
});
```

## Metadata Schema

Contractor conversation metadata:

```json
{
  "role": "contractor",
  "name": "Juan Perez",
  "skills": ["plumbing", "general_repair"],
  "location": { "lat": -34.6037, "lng": -58.3816 },
  "max_radius_km": 25,
  "available": true,
  "rating": 4.8,
  "completed_jobs": 23,
  "current_job": {
    "id": "JOB-M1K2A3",
    "status": "on_site",
    "skill": "plumbing",
    "address": "Av. Corrientes 1234, CABA",
    "customer": "Maria Lopez",
    "customer_phone": "5491155551234",
    "accepted_at": "2026-04-02T10:15:00.000Z",
    "arrived_at": "2026-04-02T10:45:00.000Z"
  },
  "pending_job": null,
  "job_history": [
    {
      "id": "JOB-X9Y8Z7",
      "status": "completed",
      "skill": "plumbing",
      "accepted_at": "2026-04-01T09:00:00.000Z",
      "completed_at": "2026-04-01T12:00:00.000Z"
    }
  ],
  "registered_at": "2026-01-15T09:00:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "contractor-dispatch",
  "type": "job-offer",
  "job_id": "JOB-M1K2A3"
}
```

## How to Run

```bash
# 1. Create the project
mkdir contractor-dispatch && cd contractor-dispatch
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"

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

# 6. Register a contractor
curl -X POST http://localhost:3000/admin/contractors \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "5491155550001",
    "name": "Juan Perez",
    "skills": ["plumbing", "general_repair"],
    "location": { "lat": -34.6037, "lng": -58.3816 },
    "max_radius_km": 25
  }'

# 7. Dispatch a job
curl -X POST http://localhost:3000/admin/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "skill_required": "plumbing",
    "description": "Leaking kitchen faucet, needs replacement",
    "address": "Av. Corrientes 1234, CABA",
    "customer_name": "Maria Lopez",
    "customer_phone": "5491155559999",
    "location": { "lat": -34.6040, "lng": -58.3820 },
    "estimated_pay": 150,
    "scheduled_time": "Today 2-4 PM"
  }'
```
