# 21 — Recruitment Screening

## Problem

HR teams are overwhelmed with initial candidate screening. For each open position, they receive dozens of WhatsApp inquiries and need to collect resume info, ask screening questions, score candidates, and schedule interviews — all manually. Good candidates drop off when response time is slow.

## Solution

An automated screening bot powered by Claude Sonnet that conducts initial candidate screenings via WhatsApp. It collects resume information conversationally, asks role-specific screening questions, scores candidates on key criteria, and schedules interviews for those who pass. The entire screening pipeline is tracked in conversation metadata.

## Architecture

```
Candidate: "Hi, I saw the Software Engineer posting"
    |
    v
┌──────────────────────────────────────┐
│   Webhook Server                     │
│                                      │
│   1. Identify which role             │
│   2. Screening stages:              │
│      intro → collect_info →          │
│      screening_questions →           │
│      scoring → schedule_or_reject    │
│   3. AI scores each answer (1-5)     │
│   4. If total score >= threshold     │
│      → schedule interview            │
│   5. Notify HR with candidate card   │
└──────────────────────────────────────┘
    |
    ├── Intro: Identify role, set expectations
    ├── Collect Info: Name, experience, skills
    ├── Screening: 3-5 role-specific questions
    ├── Scoring: AI evaluates answers → total score
    └── Result: Schedule interview or polite decline
```

## Metadata Schema

```json
{
  "candidate": {
    "name": "Alex Rivera",
    "email": "alex@email.com",
    "location": "Buenos Aires",
    "years_experience": 5,
    "current_role": "Senior Developer at TechCo",
    "education": "CS degree, University of Buenos Aires",
    "skills": ["JavaScript", "Python", "React", "Node.js", "AWS"],
    "linkedin": "linkedin.com/in/alexrivera",
    "salary_expectation": "$80,000-$100,000",
    "start_availability": "2 weeks notice"
  },
  "screening": {
    "role_id": "SE-2026-001",
    "role_title": "Senior Software Engineer",
    "stage": "completed",
    "questions_asked": 4,
    "questions_total": 4,
    "answers": [
      {
        "question": "Describe a complex system you designed",
        "answer": "I designed a real-time event processing pipeline...",
        "score": 4,
        "notes": "Strong system design thinking, good scale awareness"
      }
    ],
    "total_score": 16,
    "max_score": 20,
    "percentage": 80,
    "passed": true
  },
  "interview": {
    "scheduled": true,
    "date": "2026-04-08",
    "time": "14:00",
    "type": "video",
    "interviewer": "Sarah (Engineering Manager)",
    "link": "https://meet.google.com/abc-def-ghi"
  },
  "status": "interview_scheduled"
}
```

## Code

```javascript
// recruitment-screening.js
const express = require("express");
const Anthropic = require("@anthropic-ai/sdk");

const app = express();
app.use(express.json());

const API_BASE = "https://wpp-api.opero.so";
const API_KEY = process.env.OPERO_API_KEY;
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const HR_PHONE = process.env.HR_PHONE;

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// ── Open Positions ─────────────────────────────────────────────────────

const POSITIONS = [
  {
    id: "SE-2026-001",
    title: "Senior Software Engineer",
    department: "Engineering",
    location: "Remote",
    salary_range: "$90,000 - $130,000",
    requirements: ["5+ years experience", "JavaScript/TypeScript", "System design", "Cloud services (AWS/GCP)"],
    screening_questions: [
      "Tell me about a complex technical system you designed. What were the trade-offs?",
      "How do you approach debugging a production issue that affects multiple services?",
      "Describe your experience with CI/CD pipelines and deployment strategies.",
      "How do you mentor junior developers and ensure code quality in a team?",
    ],
    pass_threshold: 14, // out of 20 (4 questions * 5 max each)
    interviewer: "Sarah (Engineering Manager)",
  },
  {
    id: "PM-2026-002",
    title: "Product Manager",
    department: "Product",
    location: "Hybrid (Buenos Aires)",
    salary_range: "$80,000 - $110,000",
    requirements: ["3+ years PM experience", "Data-driven decision making", "Agile methodology", "Stakeholder management"],
    screening_questions: [
      "How do you prioritize features when you have more requests than capacity?",
      "Describe a product decision you made that was data-driven. What metrics did you use?",
      "How do you handle disagreements between engineering and business stakeholders?",
      "Tell me about a product launch that didn't go as planned. What did you learn?",
    ],
    pass_threshold: 14,
    interviewer: "Mike (VP Product)",
  },
  {
    id: "DS-2026-003",
    title: "Customer Success Manager",
    department: "Customer Success",
    location: "Remote",
    salary_range: "$55,000 - $75,000",
    requirements: ["2+ years in customer-facing role", "SaaS experience", "Problem-solving", "Communication skills"],
    screening_questions: [
      "How would you handle a customer threatening to cancel due to a product issue?",
      "Describe how you've proactively identified and prevented churn.",
      "How do you balance the needs of individual customers with product team priorities?",
    ],
    pass_threshold: 10, // out of 15
    interviewer: "Laura (CS Director)",
  },
];

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

async function getMessages(phone, limit = 30) {
  return apiRequest("GET", `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`);
}

async function sendText(phone, text, metadata = {}) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/messages/text`, { phone, text, metadata });
}

async function sendTyping(phone) {
  return apiRequest("POST", `/api/numbers/${NUMBER_ID}/presence`, { phone, type: "composing" });
}

// ── AI Functions ───────────────────────────────────────────────────────

async function processScreening(messages, metadata) {
  const screening = metadata.screening || {};
  const candidate = metadata.candidate || {};
  const position = POSITIONS.find((p) => p.id === screening.role_id);

  const positionList = POSITIONS.map((p) => `${p.id}: ${p.title} (${p.location}) — ${p.salary_range}`).join("\n");

  let screeningContext = "";
  if (position) {
    screeningContext = `
Role: ${position.title} (${position.id})
Requirements: ${position.requirements.join(", ")}
Screening stage: ${screening.stage || "intro"}
Questions asked: ${screening.questions_asked || 0}/${position.screening_questions.length}
Answers so far: ${JSON.stringify(screening.answers || [])}`;
  }

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: `You are a professional but friendly HR screening assistant. Your tone should be warm and encouraging while still being evaluative.

Open positions:
${positionList}

Candidate info collected: ${JSON.stringify(candidate)}
${screeningContext}

SCREENING FLOW:
1. INTRO: Greet, identify which role they're interested in
2. COLLECT INFO: Get name, experience, current role, relevant skills, email
3. SCREENING QUESTIONS: Ask one question at a time from the role's screening questions
4. For each answer, internally score 1-5 (do NOT tell the candidate their score)

After the candidate answers a screening question, score it:
- 5: Exceptional, detailed, shows deep expertise
- 4: Strong, good examples, relevant experience
- 3: Adequate, meets basic expectations
- 2: Below average, lacks depth or relevance
- 1: Poor, off-topic, or concerning answer

End your response with JSON:
{
  "stage": "intro | collecting_info | screening | completed",
  "role_id": "position ID or null",
  "candidate_update": { "name": null, "email": null, "location": null, "years_experience": null, "current_role": null, "education": null, "skills": [], "salary_expectation": null, "start_availability": null },
  "question_index": null,
  "answer_score": null,
  "answer_notes": "brief evaluation notes or null"
}`,
    messages,
  });

  const text = response.content[0].text;
  const jsonMatch = text.match(/\{[\s\S]*\}$/);

  let parsed = {};
  let cleanText = text;

  if (jsonMatch) {
    try {
      parsed = JSON.parse(jsonMatch[0]);
      cleanText = text.replace(jsonMatch[0], "").trim();
    } catch { /* use text as-is */ }
  }

  return { text: cleanText, ...parsed };
}

// ── Webhook Handler ────────────────────────────────────────────────────

app.post("/webhook", async (req, res) => {
  res.sendStatus(200);

  const { event, data } = req.body;
  if (event !== "message.received" || data.type !== "text") return;

  const phone = data.from;
  const messageText = typeof data.content === "string" ? data.content : data.content?.text || "";

  try {
    await sendTyping(phone);

    const convoRes = await getConversation(phone);
    const metadata = convoRes.data?.metadata || {};
    metadata.candidate = metadata.candidate || {};
    metadata.screening = metadata.screening || { stage: "intro", answers: [], questions_asked: 0 };

    // If already completed, handle accordingly
    if (metadata.status === "interview_scheduled" || metadata.status === "rejected") {
      await sendText(phone, metadata.status === "interview_scheduled"
        ? `Your interview is scheduled for ${metadata.interview?.date} at ${metadata.interview?.time}. Is there anything else you'd like to know about the role?`
        : "Thank you for your interest. We'll keep your profile on file for future opportunities. Feel free to check back for new openings!"
      );
      return;
    }

    // Build conversation history
    const messagesRes = await getMessages(phone, 30);
    const history = (messagesRes.data || [])
      .sort((a, b) => new Date(a.created_at) - new Date(b.created_at))
      .map((msg) => ({
        role: msg.direction === "inbound" ? "user" : "assistant",
        content: typeof msg.content === "string" ? msg.content : msg.content?.text || JSON.stringify(msg.content),
      }));

    const result = await processScreening(history, metadata);

    // Update role selection
    if (result.role_id) {
      metadata.screening.role_id = result.role_id;
    }

    // Update candidate info
    if (result.candidate_update) {
      for (const [key, value] of Object.entries(result.candidate_update)) {
        if (value === null || value === undefined) continue;
        if (Array.isArray(value) && value.length === 0) continue;
        metadata.candidate[key] = value;
      }
    }

    // Update stage
    if (result.stage) {
      metadata.screening.stage = result.stage;
    }

    // Record answer score
    if (result.answer_score && result.answer_score > 0) {
      const position = POSITIONS.find((p) => p.id === metadata.screening.role_id);
      if (position) {
        const qIndex = metadata.screening.questions_asked;
        metadata.screening.answers.push({
          question: position.screening_questions[qIndex - 1] || `Question ${qIndex}`,
          answer: messageText.substring(0, 500),
          score: result.answer_score,
          notes: result.answer_notes || "",
        });
      }
    }

    // Track question progress
    if (result.question_index !== null && result.question_index !== undefined) {
      metadata.screening.questions_asked = result.question_index + 1;
    }

    // Check if screening is complete
    if (result.stage === "completed" && metadata.screening.role_id) {
      const position = POSITIONS.find((p) => p.id === metadata.screening.role_id);
      const totalScore = metadata.screening.answers.reduce((sum, a) => sum + a.score, 0);
      const maxScore = position.screening_questions.length * 5;
      const percentage = Math.round((totalScore / maxScore) * 100);
      const passed = totalScore >= position.pass_threshold;

      metadata.screening.total_score = totalScore;
      metadata.screening.max_score = maxScore;
      metadata.screening.percentage = percentage;
      metadata.screening.passed = passed;

      if (passed) {
        // Schedule interview (next available slot)
        const interviewDate = new Date();
        interviewDate.setDate(interviewDate.getDate() + 3); // 3 days from now
        // Skip weekends
        while (interviewDate.getDay() === 0 || interviewDate.getDay() === 6) {
          interviewDate.setDate(interviewDate.getDate() + 1);
        }

        metadata.interview = {
          scheduled: true,
          date: interviewDate.toISOString().split("T")[0],
          time: "14:00",
          type: "video",
          interviewer: position.interviewer,
          link: "https://meet.google.com/abc-def-ghi",
        };
        metadata.status = "interview_scheduled";

        await updateConversation(phone, metadata);

        await sendText(phone, [
          `Thank you for completing the screening! I'm impressed with your responses.`,
          ``,
          `I'd like to schedule a video interview with ${position.interviewer}:`,
          ``,
          `Date: ${metadata.interview.date}`,
          `Time: ${metadata.interview.time}`,
          `Format: Video call`,
          `Link: ${metadata.interview.link}`,
          ``,
          `Does this time work for you? If not, let me know and we'll find an alternative.`,
        ].join("\n"), { action: "interview_scheduled" });

        // Notify HR
        if (HR_PHONE) {
          await sendText(HR_PHONE, [
            `CANDIDATE PASSED SCREENING`,
            ``,
            `Name: ${metadata.candidate.name || "Unknown"}`,
            `Role: ${position.title}`,
            `Score: ${totalScore}/${maxScore} (${percentage}%)`,
            `Experience: ${metadata.candidate.years_experience || "?"} years`,
            `Current: ${metadata.candidate.current_role || "?"}`,
            `Phone: ${phone}`,
            `Interview: ${metadata.interview.date} at ${metadata.interview.time}`,
          ].join("\n"), { alert_type: "candidate_passed", customer_phone: phone });
        }
        return;
      } else {
        // Polite rejection
        metadata.status = "rejected";
        await updateConversation(phone, metadata);

        await sendText(phone, [
          `Thank you for taking the time to go through our screening process, ${metadata.candidate.name || ""}!`,
          ``,
          `After reviewing your responses, we've decided to move forward with other candidates for this particular role. This doesn't reflect on your abilities — it's about finding the right fit for this specific position.`,
          ``,
          `We'll keep your profile on file and reach out if a more suitable opportunity comes up. We encourage you to check back for new openings!`,
          ``,
          `Wishing you all the best in your job search.`,
        ].join("\n"), { action: "screening_rejected" });

        // Notify HR
        if (HR_PHONE) {
          await sendText(HR_PHONE, [
            `Candidate screened (did not pass):`,
            `Name: ${metadata.candidate.name || "Unknown"} | Role: ${position.title}`,
            `Score: ${totalScore}/${maxScore} (${percentage}%) | Threshold: ${position.pass_threshold}`,
          ].join("\n"), { alert_type: "candidate_rejected", customer_phone: phone });
        }
        return;
      }
    }

    // Save and respond normally
    await updateConversation(phone, metadata);
    await sendText(phone, result.text, { stage: metadata.screening.stage });
  } catch (err) {
    console.error("Recruitment bot error:", err);
    await sendText(phone, "Sorry, I'm having a technical issue. Please email hr@company.com to continue your application.");
  }
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(3000, () => console.log("Recruitment screening bot running on :3000"));
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
export HR_PHONE="5491155551234"  # HR team phone

# 3. Start the server
node recruitment-screening.js

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

- **Multi-position support**: Different roles have different screening questions and pass thresholds
- **AI scoring**: Each answer is scored 1-5 with notes — candidates never see their scores
- **Staged pipeline**: intro -> info collection -> screening questions -> scoring -> result
- **Automatic interview scheduling**: Qualified candidates get interview slots immediately
- **HR notifications**: HR team gets alerts for both passed and rejected candidates with full context
- **Candidate experience**: Warm, encouraging tone throughout — even rejections are handled gracefully
