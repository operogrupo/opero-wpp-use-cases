# 41 - Course Enrollment Bot

Online course enrollment assistant via WhatsApp. Browse courses, check prerequisites, enroll students, deliver materials, and track progress -- all through conversational AI.

## Problem

An online education platform offers 50+ courses across multiple categories. Students browse the website, get overwhelmed by options, and abandon before enrolling. The support team spends 60% of their time answering "which course should I take?" and "am I eligible for this course?" questions. Enrollment completion rate sits at 35% because the process requires navigating multiple web pages.

## Solution

Deploy a WhatsApp enrollment bot that:
- Lets students browse courses by category, difficulty, or topic
- Checks prerequisites against the student's completed courses
- Handles the full enrollment flow conversationally
- Sends course materials (PDFs, links) after enrollment
- Tracks progress through modules and sends reminders for incomplete courses
- Uses conversation metadata to store enrollment state and progress per student

## Architecture

```
Student sends WhatsApp message
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
| wpp-api.opero.so |                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 |  Claude Sonnet   |
         |                                 | (Anthropic API)  |
         |                                 +------------------+
         |                                          |
         |          GET conversation metadata       |
         |          PUT conversation metadata       |
         |          POST /messages/text              |
         +------------------------------------------+
                            |
                            v
                   +------------------+
                   |  Course Database |
                   |  (in-memory/DB)  |
                   +------------------+
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

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });

// -- Course catalog (replace with real DB) --
const COURSES = [
  {
    id: 'WEB101',
    name: 'Introduction to Web Development',
    category: 'Programming',
    difficulty: 'Beginner',
    price: 49.99,
    duration: '6 weeks',
    prerequisites: [],
    modules: ['HTML Basics', 'CSS Fundamentals', 'JavaScript Intro', 'DOM Manipulation', 'Responsive Design', 'Final Project'],
    description: 'Build your first website from scratch. No prior experience needed.',
  },
  {
    id: 'WEB201',
    name: 'Advanced JavaScript & React',
    category: 'Programming',
    difficulty: 'Intermediate',
    price: 79.99,
    duration: '8 weeks',
    prerequisites: ['WEB101'],
    modules: ['ES6+', 'Async Programming', 'React Fundamentals', 'State Management', 'Hooks', 'API Integration', 'Testing', 'Deployment'],
    description: 'Master modern JavaScript and build production React apps.',
  },
  {
    id: 'DATA101',
    name: 'Data Analysis with Python',
    category: 'Data Science',
    difficulty: 'Beginner',
    price: 59.99,
    duration: '6 weeks',
    prerequisites: [],
    modules: ['Python Basics', 'NumPy', 'Pandas', 'Data Visualization', 'Statistics', 'Capstone'],
    description: 'Learn to analyze and visualize data using Python.',
  },
  {
    id: 'DATA201',
    name: 'Machine Learning Fundamentals',
    category: 'Data Science',
    difficulty: 'Intermediate',
    price: 99.99,
    duration: '10 weeks',
    prerequisites: ['DATA101'],
    modules: ['Supervised Learning', 'Unsupervised Learning', 'Feature Engineering', 'Model Evaluation', 'Neural Networks Intro', 'NLP Basics', 'Computer Vision Basics', 'ML Pipeline', 'Ethics in AI', 'Final Project'],
    description: 'Build and deploy machine learning models from scratch.',
  },
  {
    id: 'MKT101',
    name: 'Digital Marketing Essentials',
    category: 'Marketing',
    difficulty: 'Beginner',
    price: 39.99,
    duration: '4 weeks',
    prerequisites: [],
    modules: ['Marketing Fundamentals', 'Social Media Strategy', 'SEO Basics', 'Email Marketing'],
    description: 'Master the fundamentals of digital marketing.',
  },
];

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

async function sendTyping(phone) {
  return operoFetch(`/api/numbers/${NUMBER_ID}/presence`, {
    method: 'POST',
    body: JSON.stringify({ phone, type: 'composing' }),
  });
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

async function getMessages(phone, limit = 20) {
  try {
    const res = await operoFetch(
      `/api/numbers/${NUMBER_ID}/conversations/${phone}/messages?limit=${limit}`
    );
    return res.data || [];
  } catch {
    return [];
  }
}

// -- Enrollment logic --
function checkPrerequisites(courseId, completedCourses = []) {
  const course = COURSES.find(c => c.id === courseId);
  if (!course) return { eligible: false, missing: [], reason: 'Course not found' };

  const missing = course.prerequisites.filter(p => !completedCourses.includes(p));
  return {
    eligible: missing.length === 0,
    missing,
    missingNames: missing.map(id => COURSES.find(c => c.id === id)?.name || id),
  };
}

function formatCourseList(courses) {
  return courses.map(c =>
    `- *${c.id}*: ${c.name} (${c.difficulty}, ${c.duration}, $${c.price})`
  ).join('\n');
}

function formatCourseDetail(course) {
  return [
    `*${course.name}* (${course.id})`,
    `Category: ${course.category}`,
    `Difficulty: ${course.difficulty}`,
    `Duration: ${course.duration}`,
    `Price: $${course.price}`,
    ``,
    course.description,
    ``,
    `Modules:`,
    ...course.modules.map((m, i) => `  ${i + 1}. ${m}`),
    course.prerequisites.length > 0
      ? `\nPrerequisites: ${course.prerequisites.join(', ')}`
      : '',
  ].filter(Boolean).join('\n');
}

// -- AI integration --
const SYSTEM_PROMPT = `You are the enrollment assistant for TechLearn Academy, an online education platform.

Available courses:
${COURSES.map(c => `- ${c.id}: ${c.name} (${c.category}, ${c.difficulty}, $${c.price}, ${c.duration})`).join('\n')}

You help students:
1. Browse and discover courses by category or interest
2. Get detailed course information
3. Check prerequisites
4. Enroll in courses
5. Track their progress

When a student wants to enroll, respond with exactly: [ENROLL:COURSE_ID]
When they ask to see a course list by category, respond with: [LIST:CATEGORY] (use "all" for all courses)
When they ask about course details, respond with: [DETAIL:COURSE_ID]
When they ask about their progress, respond with: [PROGRESS]
When they complete a module, respond with: [COMPLETE_MODULE:COURSE_ID:MODULE_INDEX]

Include these tags naturally within your conversational response. Keep responses friendly and concise.

Current student metadata will be provided as context. Use it to personalize recommendations.`;

async function generateReply(phone, message, metadata) {
  const history = await getMessages(phone, 10);
  const messages = history
    .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    .map(m => ({
      role: m.from_me ? 'assistant' : 'user',
      content: m.text || m.content?.text || '',
    }))
    .filter(m => m.content);

  messages.push({ role: 'user', content: message });

  const metadataContext = Object.keys(metadata).length > 0
    ? `\n\nStudent profile:\n${JSON.stringify(metadata, null, 2)}`
    : '\n\nNew student (no enrollment history).';

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: SYSTEM_PROMPT + metadataContext,
    messages: messages.slice(-20),
  });

  return response.content[0].text;
}

// -- Action processor --
async function processActions(phone, aiResponse, metadata) {
  let finalResponse = aiResponse;
  const updatedMetadata = { ...metadata };

  // Initialize student data if needed
  if (!updatedMetadata.enrolled_courses) updatedMetadata.enrolled_courses = [];
  if (!updatedMetadata.completed_courses) updatedMetadata.completed_courses = [];
  if (!updatedMetadata.progress) updatedMetadata.progress = {};

  // Handle enrollment
  const enrollMatch = aiResponse.match(/\[ENROLL:(\w+)\]/);
  if (enrollMatch) {
    const courseId = enrollMatch[1];
    const course = COURSES.find(c => c.id === courseId);

    if (!course) {
      finalResponse = finalResponse.replace(enrollMatch[0], '');
      finalResponse += '\n\nSorry, that course was not found.';
    } else if (updatedMetadata.enrolled_courses.includes(courseId)) {
      finalResponse = finalResponse.replace(enrollMatch[0], '');
      finalResponse += `\n\nYou're already enrolled in ${course.name}!`;
    } else {
      const prereqCheck = checkPrerequisites(courseId, updatedMetadata.completed_courses);
      if (!prereqCheck.eligible) {
        finalResponse = finalResponse.replace(enrollMatch[0], '');
        finalResponse += `\n\nYou need to complete these courses first: ${prereqCheck.missingNames.join(', ')}`;
      } else {
        updatedMetadata.enrolled_courses.push(courseId);
        updatedMetadata.progress[courseId] = {
          current_module: 0,
          total_modules: course.modules.length,
          started_at: new Date().toISOString(),
          completed_modules: [],
        };
        finalResponse = finalResponse.replace(enrollMatch[0], '');
        finalResponse += `\n\nYou're now enrolled in *${course.name}*! Your first module is: *${course.modules[0]}*`;
      }
    }
  }

  // Handle course listing
  const listMatch = aiResponse.match(/\[LIST:(\w+)\]/);
  if (listMatch) {
    const category = listMatch[1].toLowerCase();
    const filtered = category === 'all'
      ? COURSES
      : COURSES.filter(c => c.category.toLowerCase() === category);
    finalResponse = finalResponse.replace(listMatch[0], '');
    finalResponse += '\n\n' + (filtered.length > 0
      ? formatCourseList(filtered)
      : 'No courses found in that category.');
  }

  // Handle course detail
  const detailMatch = aiResponse.match(/\[DETAIL:(\w+)\]/);
  if (detailMatch) {
    const course = COURSES.find(c => c.id === detailMatch[1]);
    finalResponse = finalResponse.replace(detailMatch[0], '');
    if (course) {
      finalResponse += '\n\n' + formatCourseDetail(course);
    }
  }

  // Handle progress check
  if (aiResponse.includes('[PROGRESS]')) {
    finalResponse = finalResponse.replace('[PROGRESS]', '');
    if (updatedMetadata.enrolled_courses.length === 0) {
      finalResponse += "\n\nYou're not enrolled in any courses yet.";
    } else {
      const progressLines = updatedMetadata.enrolled_courses.map(cid => {
        const course = COURSES.find(c => c.id === cid);
        const prog = updatedMetadata.progress[cid];
        if (!course || !prog) return null;
        const pct = Math.round((prog.completed_modules.length / prog.total_modules) * 100);
        return `- *${course.name}*: ${prog.completed_modules.length}/${prog.total_modules} modules (${pct}%)`;
      }).filter(Boolean);
      finalResponse += '\n\n' + progressLines.join('\n');
    }
  }

  // Handle module completion
  const completeMatch = aiResponse.match(/\[COMPLETE_MODULE:(\w+):(\d+)\]/);
  if (completeMatch) {
    const courseId = completeMatch[1];
    const moduleIdx = parseInt(completeMatch[2]);
    const course = COURSES.find(c => c.id === courseId);
    const prog = updatedMetadata.progress[courseId];
    finalResponse = finalResponse.replace(completeMatch[0], '');

    if (course && prog && moduleIdx < course.modules.length) {
      if (!prog.completed_modules.includes(moduleIdx)) {
        prog.completed_modules.push(moduleIdx);
        prog.current_module = Math.max(prog.current_module, moduleIdx + 1);
      }

      const pct = Math.round((prog.completed_modules.length / prog.total_modules) * 100);
      finalResponse += `\n\nModule "${course.modules[moduleIdx]}" marked as complete! Progress: ${pct}%`;

      // Check if course is fully completed
      if (prog.completed_modules.length === prog.total_modules) {
        prog.completed_at = new Date().toISOString();
        updatedMetadata.completed_courses.push(courseId);
        finalResponse += `\n\nCongratulations! You've completed *${course.name}*!`;
      } else {
        const nextModule = course.modules[prog.current_module];
        if (nextModule) {
          finalResponse += `\nNext up: *${nextModule}*`;
        }
      }
    }
  }

  // Save updated metadata
  if (JSON.stringify(updatedMetadata) !== JSON.stringify(metadata)) {
    updatedMetadata.last_interaction = new Date().toISOString();
    await updateConversationMetadata(phone, updatedMetadata);
  }

  // Clean any remaining tags
  finalResponse = finalResponse.replace(/\[\w+:?\w*:?\w*\]/g, '').trim();

  return finalResponse;
}

// -- Webhook handler --
app.post('/webhook', async (req, res) => {
  res.status(200).json({ received: true });

  const { event, data } = req.body;
  if (event !== 'message.received' || data.type !== 'text') return;

  const phone = data.from;
  const messageText = typeof data.content === 'string'
    ? data.content
    : data.content?.text || '';
  if (!messageText.trim()) return;

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);
    await sendTyping(phone);

    const metadata = await getConversationMetadata(phone);
    const aiResponse = await generateReply(phone, messageText, metadata);
    const finalResponse = await processActions(phone, aiResponse, metadata);

    const typingDelay = Math.min(finalResponse.length * 30, 3000);
    await new Promise(resolve => setTimeout(resolve, typingDelay));

    await sendText(phone, finalResponse, {
      agent: 'course-enrollment-bot',
      model: 'claude-sonnet-4-20250514',
      enrolled_courses: metadata.enrolled_courses?.length || 0,
      generated_at: new Date().toISOString(),
    });

    console.log(`[${new Date().toISOString()}] Reply sent to ${phone}`);
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
    try {
      await sendText(phone, 'Sorry, something went wrong. Please try again in a moment.', {
        agent: 'course-enrollment-bot',
        error: true,
        error_message: err.message,
      });
    } catch (fallbackErr) {
      console.error('Fallback message also failed:', fallbackErr.message);
    }
  }
});

// -- Health check --
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    courses_available: COURSES.length,
    uptime: process.uptime(),
  });
});

// -- Start server --
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Course enrollment bot listening on port ${PORT}`);
  console.log(`Webhook URL: POST http://localhost:${PORT}/webhook`);
});
```

## Metadata Schema

Each conversation stores student enrollment state:

```json
{
  "enrolled_courses": ["WEB101", "DATA101"],
  "completed_courses": ["WEB101"],
  "progress": {
    "WEB101": {
      "current_module": 6,
      "total_modules": 6,
      "started_at": "2026-03-15T10:00:00.000Z",
      "completed_at": "2026-04-01T14:30:00.000Z",
      "completed_modules": [0, 1, 2, 3, 4, 5]
    },
    "DATA101": {
      "current_module": 2,
      "total_modules": 6,
      "started_at": "2026-04-01T09:00:00.000Z",
      "completed_modules": [0, 1]
    }
  },
  "last_interaction": "2026-04-02T14:30:00.000Z"
}
```

Outbound message metadata:

```json
{
  "agent": "course-enrollment-bot",
  "model": "claude-sonnet-4-20250514",
  "enrolled_courses": 2,
  "generated_at": "2026-04-02T14:30:00.000Z"
}
```

## How to Run

```bash
# 1. Create the project
mkdir course-enrollment-bot && cd course-enrollment-bot
npm init -y
npm install express @anthropic-ai/sdk

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export ANTHROPIC_API_KEY="your-anthropic-api-key"

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
```
