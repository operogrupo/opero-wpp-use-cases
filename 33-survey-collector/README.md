# 33 - Survey Collector

Multi-question survey via WhatsApp with sequential questions, skip logic, answer validation, and summary generation. No AI needed.

## Problem

A market research firm needs to conduct surveys with 500+ respondents. Phone surveys cost $15+ per completion and have declining response rates. Online survey links sent via email get 8% open rates. The firm needs a channel where people actually respond, with structured data collection and real-time analytics.

## Solution

Deploy a WhatsApp survey bot that:
- Delivers sequential questions with typed responses
- Supports skip logic (conditional questions based on previous answers)
- Validates answers (numeric ranges, required fields, single/multi-select)
- Stores all responses in conversation metadata
- Generates a summary when the survey is complete
- No AI required -- pure deterministic logic

## Architecture

```
+--------------------+    POST /api/surveys     +--------------------+
|  Survey Designer   | -----------------------> |  Your Express App  |
|  (create surveys)  |                          |   (this code)      |
+--------------------+                          +--------------------+
                                                         |
        +------------------------------------------------+
        |                     |                          |
        v                     v                          v
+------------------+   Send first question       Store responses
|   Opero WPP API  |   to respondent list       in metadata
|  wpp-api.opero.so|                                    |
+------------------+                                    v
        ^                                      +-----------------+
        |           webhook                    | Survey Results  |
        +---- respondent answers  -----------> | GET /api/results|
                                               +-----------------+
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
const surveys = new Map();
const respondents = new Map(); // phone -> { survey_id, current_question, answers }

// ── Question types ────────────────────────────────────────────────────
const QUESTION_TYPES = {
  TEXT: 'text',           // Free text
  NUMBER: 'number',       // Numeric with optional min/max
  SINGLE: 'single',       // Single choice from options
  MULTI: 'multi',         // Multiple choice (comma-separated numbers)
  SCALE: 'scale',         // 1-N scale
  YES_NO: 'yes_no',       // Yes/No
};

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

// ── Survey management ─────────────────────────────────────────────────
function createSurvey(data) {
  const id = `SURV-${Date.now().toString(36).toUpperCase()}`;
  const survey = {
    id,
    title: data.title,
    description: data.description || '',
    welcome_message: data.welcome_message || `Gracias por participar en nuestra encuesta: *${data.title}*`,
    completion_message: data.completion_message || '¡Gracias por completar la encuesta!',
    questions: data.questions.map((q, i) => ({
      id: `Q${i + 1}`,
      index: i,
      text: q.text,
      type: q.type || QUESTION_TYPES.TEXT,
      options: q.options || [],
      required: q.required !== false,
      min: q.min,
      max: q.max,
      scale_max: q.scale_max || 5,
      scale_labels: q.scale_labels || {},
      skip_if: q.skip_if || null, // { question_id, answer } — skip this if condition met
    })),
    responses: [],
    created_at: new Date().toISOString(),
    status: 'active',
  };
  surveys.set(id, survey);
  return survey;
}

// ── Format question for WhatsApp ──────────────────────────────────────
function formatQuestion(question, questionNumber, totalQuestions) {
  let text = `*Pregunta ${questionNumber}/${totalQuestions}*\n\n${question.text}\n`;

  switch (question.type) {
    case QUESTION_TYPES.SINGLE:
      text += '\n' + question.options.map((opt, i) => `*${i + 1}* - ${opt}`).join('\n');
      text += '\n\n_Responda con el número de su elección._';
      break;

    case QUESTION_TYPES.MULTI:
      text += '\n' + question.options.map((opt, i) => `*${i + 1}* - ${opt}`).join('\n');
      text += '\n\n_Responda con los números separados por coma (ej: 1,3,5)._';
      break;

    case QUESTION_TYPES.SCALE:
      const labels = question.scale_labels;
      text += `\n_Responda del 1 al ${question.scale_max}_`;
      if (labels['1'] || labels[String(question.scale_max)]) {
        text += `\n1 = ${labels['1'] || 'Mínimo'} | ${question.scale_max} = ${labels[String(question.scale_max)] || 'Máximo'}`;
      }
      break;

    case QUESTION_TYPES.YES_NO:
      text += '\n*1* - Sí\n*2* - No';
      break;

    case QUESTION_TYPES.NUMBER:
      if (question.min !== undefined || question.max !== undefined) {
        text += `\n_Ingrese un número${question.min !== undefined ? ` (mín: ${question.min}` : ''}${question.max !== undefined ? `, máx: ${question.max})` : ')'}._`;
      }
      break;

    case QUESTION_TYPES.TEXT:
    default:
      if (!question.required) {
        text += '\n_Escriba su respuesta o "SALTAR" para omitir._';
      }
      break;
  }

  return text;
}

// ── Validate answer ───────────────────────────────────────────────────
function validateAnswer(question, answer) {
  const trimmed = answer.trim();

  if (!question.required && (trimmed.toUpperCase() === 'SALTAR' || trimmed === '')) {
    return { valid: true, value: null, skipped: true };
  }

  switch (question.type) {
    case QUESTION_TYPES.SINGLE: {
      const idx = parseInt(trimmed, 10) - 1;
      if (isNaN(idx) || idx < 0 || idx >= question.options.length) {
        return { valid: false, error: `Elija un número entre 1 y ${question.options.length}.` };
      }
      return { valid: true, value: question.options[idx], index: idx };
    }

    case QUESTION_TYPES.MULTI: {
      const indices = trimmed.split(',').map(s => parseInt(s.trim(), 10) - 1);
      const invalid = indices.some(i => isNaN(i) || i < 0 || i >= question.options.length);
      if (invalid) {
        return { valid: false, error: `Ingrese números válidos separados por coma (1-${question.options.length}).` };
      }
      return { valid: true, value: indices.map(i => question.options[i]), indices };
    }

    case QUESTION_TYPES.SCALE: {
      const num = parseInt(trimmed, 10);
      if (isNaN(num) || num < 1 || num > question.scale_max) {
        return { valid: false, error: `Ingrese un número del 1 al ${question.scale_max}.` };
      }
      return { valid: true, value: num };
    }

    case QUESTION_TYPES.YES_NO: {
      const map = { '1': true, '2': false, 'si': true, 'sí': true, 'no': false };
      const val = map[trimmed.toLowerCase()];
      if (val === undefined) {
        return { valid: false, error: 'Responda *1* (Sí) o *2* (No).' };
      }
      return { valid: true, value: val };
    }

    case QUESTION_TYPES.NUMBER: {
      const num = parseFloat(trimmed);
      if (isNaN(num)) {
        return { valid: false, error: 'Ingrese un número válido.' };
      }
      if (question.min !== undefined && num < question.min) {
        return { valid: false, error: `El valor mínimo es ${question.min}.` };
      }
      if (question.max !== undefined && num > question.max) {
        return { valid: false, error: `El valor máximo es ${question.max}.` };
      }
      return { valid: true, value: num };
    }

    case QUESTION_TYPES.TEXT:
    default:
      if (question.required && !trimmed) {
        return { valid: false, error: 'Esta pregunta es obligatoria.' };
      }
      return { valid: true, value: trimmed };
  }
}

// ── Determine next question (skip logic) ──────────────────────────────
function getNextQuestionIndex(survey, currentIndex, answers) {
  for (let i = currentIndex + 1; i < survey.questions.length; i++) {
    const q = survey.questions[i];
    if (q.skip_if) {
      const prevAnswer = answers[q.skip_if.question_id];
      if (prevAnswer !== undefined && prevAnswer === q.skip_if.answer) {
        continue; // Skip this question
      }
    }
    return i;
  }
  return -1; // Survey complete
}

// ── Generate summary ──────────────────────────────────────────────────
function generateSummary(survey, answers) {
  const lines = survey.questions.map(q => {
    const answer = answers[q.id];
    if (answer === null || answer === undefined) return `*${q.text}*\n  _Omitida_`;

    let displayAnswer;
    if (Array.isArray(answer)) {
      displayAnswer = answer.join(', ');
    } else if (typeof answer === 'boolean') {
      displayAnswer = answer ? 'Sí' : 'No';
    } else {
      displayAnswer = String(answer);
    }
    return `*${q.text}*\n  ${displayAnswer}`;
  });

  return `📊 *Resumen de sus respuestas*\n\n${lines.join('\n\n')}`;
}

// ── Webhook handler ───────────────────────────────────────────────────
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

  console.log(`[${new Date().toISOString()}] Survey response from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const respondent = respondents.get(phone);
    if (!respondent) return; // Not part of any active survey

    const survey = surveys.get(respondent.survey_id);
    if (!survey) return;

    // Not started yet — should not happen, but handle gracefully
    if (respondent.current_question === -1) return;

    const currentQuestion = survey.questions[respondent.current_question];
    const validation = validateAnswer(currentQuestion, messageText);

    if (!validation.valid) {
      await sendText(phone, `❌ ${validation.error}`, {
        agent: 'survey-collector',
        survey_id: survey.id,
        question_id: currentQuestion.id,
        action: 'validation_error',
      });
      return;
    }

    // Store answer
    respondent.answers[currentQuestion.id] = validation.skipped ? null : validation.value;

    // Find next question
    const nextIndex = getNextQuestionIndex(survey, respondent.current_question, respondent.answers);

    if (nextIndex === -1) {
      // Survey complete
      respondent.completed = true;
      respondent.completed_at = new Date().toISOString();

      // Store in survey responses
      survey.responses.push({
        phone,
        answers: { ...respondent.answers },
        started_at: respondent.started_at,
        completed_at: respondent.completed_at,
      });

      // Send summary
      const summary = generateSummary(survey, respondent.answers);
      await sendText(phone, `${summary}\n\n${survey.completion_message}`, {
        agent: 'survey-collector',
        survey_id: survey.id,
        action: 'survey_completed',
        response_count: survey.responses.length,
      });

      // Update conversation metadata
      await updateConversation(phone, {
        agent: 'survey-collector',
        survey_id: survey.id,
        survey_title: survey.title,
        status: 'completed',
        answers: respondent.answers,
        started_at: respondent.started_at,
        completed_at: respondent.completed_at,
      });

      console.log(`[${new Date().toISOString()}] Survey completed by ${phone} (response #${survey.responses.length})`);
    } else {
      // Send next question
      respondent.current_question = nextIndex;
      const nextQuestion = survey.questions[nextIndex];
      const questionText = formatQuestion(nextQuestion, nextIndex + 1, survey.questions.length);

      await sendText(phone, questionText, {
        agent: 'survey-collector',
        survey_id: survey.id,
        question_id: nextQuestion.id,
        question_number: nextIndex + 1,
        total_questions: survey.questions.length,
      });
    }
  } catch (err) {
    console.error(`Error handling survey response from ${phone}:`, err.message);
  }
});

// ── API: Create survey ────────────────────────────────────────────────
app.post('/api/surveys', (req, res) => {
  try {
    const survey = createSurvey(req.body);
    res.json(survey);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Send survey to respondents ───────────────────────────────────
app.post('/api/surveys/:surveyId/send', async (req, res) => {
  const survey = surveys.get(req.params.surveyId);
  if (!survey) return res.status(404).json({ error: 'Survey not found' });

  const { phones } = req.body; // Array of phone numbers
  if (!Array.isArray(phones) || phones.length === 0) {
    return res.status(400).json({ error: 'phones array is required' });
  }

  const results = { sent: 0, failed: 0 };

  for (const phone of phones) {
    try {
      // Register respondent
      respondents.set(phone, {
        survey_id: survey.id,
        current_question: 0,
        answers: {},
        started_at: new Date().toISOString(),
        completed: false,
      });

      // Send welcome + first question
      const firstQuestion = survey.questions[0];
      const questionText = formatQuestion(firstQuestion, 1, survey.questions.length);

      await sendText(phone,
        `${survey.welcome_message}\n\n` +
        `📝 Son ${survey.questions.length} preguntas. Empecemos:\n\n` +
        questionText,
        {
          agent: 'survey-collector',
          survey_id: survey.id,
          action: 'survey_started',
          question_id: firstQuestion.id,
          question_number: 1,
          total_questions: survey.questions.length,
        }
      );

      results.sent++;
      await new Promise(resolve => setTimeout(resolve, 300));
    } catch (err) {
      results.failed++;
      console.error(`Failed to send survey to ${phone}:`, err.message);
    }
  }

  res.json(results);
});

// ── API: Get survey results ───────────────────────────────────────────
app.get('/api/surveys/:surveyId/results', (req, res) => {
  const survey = surveys.get(req.params.surveyId);
  if (!survey) return res.status(404).json({ error: 'Survey not found' });

  // Aggregate results
  const aggregated = {};
  for (const question of survey.questions) {
    const answers = survey.responses.map(r => r.answers[question.id]).filter(a => a !== null && a !== undefined);

    switch (question.type) {
      case QUESTION_TYPES.SINGLE:
      case QUESTION_TYPES.YES_NO: {
        const counts = {};
        for (const a of answers) {
          const key = typeof a === 'boolean' ? (a ? 'Sí' : 'No') : a;
          counts[key] = (counts[key] || 0) + 1;
        }
        aggregated[question.id] = { question: question.text, type: question.type, counts, total: answers.length };
        break;
      }
      case QUESTION_TYPES.SCALE:
      case QUESTION_TYPES.NUMBER: {
        const nums = answers.filter(a => typeof a === 'number');
        aggregated[question.id] = {
          question: question.text,
          type: question.type,
          average: nums.length > 0 ? (nums.reduce((a, b) => a + b, 0) / nums.length).toFixed(2) : null,
          min: nums.length > 0 ? Math.min(...nums) : null,
          max: nums.length > 0 ? Math.max(...nums) : null,
          total: nums.length,
        };
        break;
      }
      case QUESTION_TYPES.MULTI: {
        const counts = {};
        for (const a of answers) {
          if (Array.isArray(a)) {
            for (const item of a) {
              counts[item] = (counts[item] || 0) + 1;
            }
          }
        }
        aggregated[question.id] = { question: question.text, type: question.type, counts, total: answers.length };
        break;
      }
      default:
        aggregated[question.id] = { question: question.text, type: question.type, answers, total: answers.length };
        break;
    }
  }

  res.json({
    survey_id: survey.id,
    title: survey.title,
    total_responses: survey.responses.length,
    results: aggregated,
    raw_responses: survey.responses,
  });
});

// ── API: Get survey status ────────────────────────────────────────────
app.get('/api/surveys/:surveyId', (req, res) => {
  const survey = surveys.get(req.params.surveyId);
  if (!survey) return res.status(404).json({ error: 'Survey not found' });

  const activeRespondents = [...respondents.values()]
    .filter(r => r.survey_id === survey.id && !r.completed);

  res.json({
    ...survey,
    stats: {
      total_sent: [...respondents.values()].filter(r => r.survey_id === survey.id).length,
      completed: survey.responses.length,
      in_progress: activeRespondents.length,
    },
  });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    surveys: surveys.size,
    active_respondents: [...respondents.values()].filter(r => !r.completed).length,
    total_responses: [...surveys.values()].reduce((sum, s) => sum + s.responses.length, 0),
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Survey collector bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata after survey completion:

```json
{
  "agent": "survey-collector",
  "survey_id": "SURV-M1A2B3",
  "survey_title": "Satisfacción del cliente Q1 2026",
  "status": "completed",
  "answers": {
    "Q1": "Muy satisfecho",
    "Q2": 9,
    "Q3": true,
    "Q4": ["Precio", "Calidad"],
    "Q5": "Mejorar los tiempos de entrega"
  },
  "started_at": "2026-04-02T10:00:00.000Z",
  "completed_at": "2026-04-02T10:08:00.000Z"
}
```

Per-message metadata:

```json
{
  "agent": "survey-collector",
  "survey_id": "SURV-M1A2B3",
  "question_id": "Q3",
  "question_number": 3,
  "total_questions": 5
}
```

## How to Run

```bash
# 1. Create the project
mkdir survey-collector && cd survey-collector
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

# 6. Create a survey
curl -X POST http://localhost:3000/api/surveys \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Satisfacción del cliente Q1 2026",
    "questions": [
      {
        "text": "¿Cómo calificaría nuestro servicio en general?",
        "type": "single",
        "options": ["Muy satisfecho", "Satisfecho", "Neutral", "Insatisfecho", "Muy insatisfecho"]
      },
      {
        "text": "Del 1 al 10, ¿qué tan probable es que nos recomiende?",
        "type": "scale",
        "scale_max": 10,
        "scale_labels": { "1": "Nada probable", "10": "Muy probable" }
      },
      {
        "text": "¿Volvería a utilizar nuestros servicios?",
        "type": "yes_no"
      },
      {
        "text": "¿Qué aspectos valora más? (puede elegir varios)",
        "type": "multi",
        "options": ["Precio", "Calidad", "Atención", "Rapidez", "Variedad"]
      },
      {
        "text": "¿Tiene alguna sugerencia de mejora?",
        "type": "text",
        "required": false
      }
    ]
  }'

# 7. Send the survey
curl -X POST http://localhost:3000/api/surveys/SURV-xxx/send \
  -H "Content-Type: application/json" \
  -d '{
    "phones": ["5491155551001", "5491155551002", "5491155551003"]
  }'

# 8. Check results
curl http://localhost:3000/api/surveys/SURV-xxx/results
```
