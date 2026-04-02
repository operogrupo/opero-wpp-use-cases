# 36 - Referral Program

Referral program manager via WhatsApp. Customers share referral links, track when referred contacts sign up, get notified of rewards, and receive reward codes. All tracked in metadata.

## Problem

A subscription business runs a referral program through email, but tracking is poor and rewards are delayed. Customers forget they referred someone, don't know when to expect rewards, and support handles 20+ "where's my referral bonus?" tickets weekly. The current system has a 3% referral conversion rate because the process is friction-heavy.

## Solution

Deploy a WhatsApp referral bot that:
- Generates unique referral codes/links per customer
- Notifies referrers instantly when a referred contact signs up
- Tracks referral status (pending, signed up, qualified, rewarded)
- Automatically sends reward codes when referrals qualify
- Provides a leaderboard and referral stats
- Stores all referral data in conversation metadata

## Architecture

```
Customer asks for referral link
         |
         v
+------------------+     webhook POST      +--------------------+
|   Opero WPP API  | --------------------> |  Your Express App  |
|  wpp-api.opero.so|                       |   (this code)      |
+------------------+                       +--------------------+
         ^                                          |
         |                                          v
         |                                 +------------------+
         |                                 | Referral Engine  |
         |                                 | (track & reward) |
         |                                 +------------------+
         |                                          |
         |      POST /messages/text                 |
         |      PUT  /conversations/:phone          |
         +------------------------------------------+
                    |
                    v
           External signup webhook
           (new user registers with referral code)
```

## Code

```javascript
// server.js
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.json());

// ── Config ────────────────────────────────────────────────────────────
const OPERO_API_KEY = process.env.OPERO_API_KEY;
const OPERO_BASE_URL = 'https://wpp-api.opero.so';
const NUMBER_ID = process.env.OPERO_NUMBER_ID;
const REFERRAL_BASE_URL = process.env.REFERRAL_BASE_URL || 'https://app.example.com/ref';

// ── Reward configuration ──────────────────────────────────────────────
const REWARD_CONFIG = {
  referrer_reward: 'DESCUENTO-20',    // What the referrer gets
  referrer_reward_label: '20% de descuento por 3 meses',
  referee_reward: 'BIENVENIDA-10',     // What the new user gets
  referee_reward_label: '10% de descuento en el primer mes',
  qualification_days: 7,               // Days the referred user must stay active
  max_referrals_per_user: 20,
  tier_bonuses: {
    5: { code: 'SUPER-REF-5', label: 'Mes gratis por 5 referidos' },
    10: { code: 'SUPER-REF-10', label: '2 meses gratis por 10 referidos' },
    20: { code: 'VIP-LIFETIME', label: 'Descuento permanente del 30%' },
  },
};

// ── Data stores ───────────────────────────────────────────────────────
const referrers = new Map();    // phone -> referrer profile
const referralCodes = new Map(); // code -> phone
const referrals = new Map();     // referralId -> referral record

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

// ── Referral engine ───────────────────────────────────────────────────
function getOrCreateReferrer(phone, name) {
  if (!referrers.has(phone)) {
    const code = generateCode();
    const referrer = {
      phone,
      name: name || '',
      referral_code: code,
      referral_link: `${REFERRAL_BASE_URL}/${code}`,
      total_referrals: 0,
      qualified_referrals: 0,
      rewards_earned: [],
      created_at: new Date().toISOString(),
    };
    referrers.set(phone, referrer);
    referralCodes.set(code, phone);
  }
  return referrers.get(phone);
}

function generateCode() {
  return crypto.randomBytes(4).toString('hex').toUpperCase();
}

function generateRewardCode() {
  return `RWD-${crypto.randomBytes(3).toString('hex').toUpperCase()}`;
}

async function recordReferral(referrerCode, referredPhone, referredName) {
  const referrerPhone = referralCodes.get(referrerCode);
  if (!referrerPhone) throw new Error('Invalid referral code');

  const referrer = referrers.get(referrerPhone);
  if (!referrer) throw new Error('Referrer not found');

  if (referrer.total_referrals >= REWARD_CONFIG.max_referrals_per_user) {
    return { status: 'max_reached' };
  }

  const refId = `REF-${Date.now().toString(36).toUpperCase()}`;
  const referral = {
    id: refId,
    referrer_phone: referrerPhone,
    referred_phone: referredPhone,
    referred_name: referredName || '',
    referral_code: referrerCode,
    status: 'signed_up', // signed_up, qualified, rewarded, expired
    signed_up_at: new Date().toISOString(),
    qualified_at: null,
    rewarded_at: null,
  };

  referrals.set(refId, referral);
  referrer.total_referrals++;

  // Notify referrer
  try {
    await sendText(referrerPhone,
      `🎉 *¡Nuevo referido!*\n\n` +
      `${referredName || 'Alguien'} se registró con tu código.\n\n` +
      `Estado: Pendiente de calificación\n` +
      `Se calificará en ${REWARD_CONFIG.qualification_days} días si permanece activo.\n\n` +
      `Referidos totales: ${referrer.total_referrals}`,
      {
        agent: 'referral-program',
        action: 'referral_signup',
        referral_id: refId,
        total_referrals: referrer.total_referrals,
      }
    );

    await syncMetadata(referrerPhone);
  } catch (err) {
    console.error('Failed to notify referrer:', err.message);
  }

  return { status: 'recorded', referral_id: refId };
}

async function qualifyReferral(referralId) {
  const referral = referrals.get(referralId);
  if (!referral || referral.status !== 'signed_up') return;

  referral.status = 'qualified';
  referral.qualified_at = new Date().toISOString();

  const referrer = referrers.get(referral.referrer_phone);
  referrer.qualified_referrals++;

  // Generate reward
  const rewardCode = generateRewardCode();
  referral.status = 'rewarded';
  referral.rewarded_at = new Date().toISOString();

  referrer.rewards_earned.push({
    code: rewardCode,
    type: REWARD_CONFIG.referrer_reward,
    label: REWARD_CONFIG.referrer_reward_label,
    referral_id: referralId,
    earned_at: referral.rewarded_at,
  });

  // Notify referrer with reward
  try {
    await sendText(referral.referrer_phone,
      `🏆 *¡Recompensa desbloqueada!*\n\n` +
      `Su referido ${referral.referred_name || ''} calificó.\n\n` +
      `🎁 Su recompensa: *${REWARD_CONFIG.referrer_reward_label}*\n` +
      `Código: *${rewardCode}*\n\n` +
      `Referidos calificados: ${referrer.qualified_referrals}/${referrer.total_referrals}`,
      {
        agent: 'referral-program',
        action: 'reward_earned',
        referral_id: referralId,
        reward_code: rewardCode,
        qualified_referrals: referrer.qualified_referrals,
      }
    );

    // Check for tier bonuses
    const tierBonus = REWARD_CONFIG.tier_bonuses[referrer.qualified_referrals];
    if (tierBonus) {
      referrer.rewards_earned.push({
        code: tierBonus.code,
        type: 'tier_bonus',
        label: tierBonus.label,
        earned_at: new Date().toISOString(),
      });

      await sendText(referral.referrer_phone,
        `🌟 *¡Bonus por hito alcanzado!*\n\n` +
        `¡${referrer.qualified_referrals} referidos calificados!\n\n` +
        `🎁 Bonus adicional: *${tierBonus.label}*\n` +
        `Código: *${tierBonus.code}*`,
        {
          agent: 'referral-program',
          action: 'tier_bonus',
          tier: referrer.qualified_referrals,
          bonus_code: tierBonus.code,
        }
      );
    }

    await syncMetadata(referral.referrer_phone);
  } catch (err) {
    console.error('Failed to send reward:', err.message);
  }
}

async function syncMetadata(phone) {
  const referrer = referrers.get(phone);
  if (!referrer) return;

  const myReferrals = [...referrals.values()]
    .filter(r => r.referrer_phone === phone)
    .map(r => ({
      id: r.id,
      referred_name: r.referred_name,
      status: r.status,
      signed_up_at: r.signed_up_at,
    }));

  await updateConversation(phone, {
    agent: 'referral-program',
    referral_code: referrer.referral_code,
    referral_link: referrer.referral_link,
    total_referrals: referrer.total_referrals,
    qualified_referrals: referrer.qualified_referrals,
    rewards_earned: referrer.rewards_earned.length,
    referrals: myReferrals,
  });
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

  console.log(`[${new Date().toISOString()}] Referral message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const upper = messageText.toUpperCase();
    const referrer = getOrCreateReferrer(phone);

    if (upper === 'MENU' || upper === 'HOLA' || upper === 'REFERIR') {
      await sendText(phone,
        `🎁 *Programa de Referidos*\n\n` +
        `Su código: *${referrer.referral_code}*\n` +
        `Su link: ${referrer.referral_link}\n\n` +
        `Referidos: ${referrer.total_referrals} (${referrer.qualified_referrals} calificados)\n` +
        `Recompensas ganadas: ${referrer.rewards_earned.length}\n\n` +
        `Opciones:\n` +
        `*1* - Compartir mi link\n` +
        `*2* - Ver mis referidos\n` +
        `*3* - Ver mis recompensas\n` +
        `*4* - Ver ranking\n` +
        `*5* - ¿Cómo funciona?`,
        { agent: 'referral-program', action: 'menu_shown' }
      );
      return;
    }

    switch (messageText.trim()) {
      case '1': { // Share link
        const shareMessage =
          `Te invito a probar esta app increíble. ` +
          `Registrate con mi link y obtenés ${REWARD_CONFIG.referee_reward_label}:\n\n` +
          `${referrer.referral_link}`;

        await sendText(phone,
          `📤 *Compartí este mensaje:*\n\n${shareMessage}\n\n` +
          `_Copie y pegue este mensaje a sus contactos._`,
          { agent: 'referral-program', action: 'share_link' }
        );
        break;
      }

      case '2': { // My referrals
        const myRefs = [...referrals.values()].filter(r => r.referrer_phone === phone);
        if (myRefs.length === 0) {
          await sendText(phone, 'Aún no tiene referidos. ¡Comparta su link para empezar!', {
            agent: 'referral-program', action: 'no_referrals',
          });
        } else {
          const statusIcons = { signed_up: '⏳', qualified: '✅', rewarded: '🏆', expired: '❌' };
          const list = myRefs.map(r =>
            `${statusIcons[r.status] || '?'} ${r.referred_name || 'Anónimo'} — ${r.status === 'rewarded' ? 'Recompensado' : r.status === 'qualified' ? 'Calificado' : r.status === 'signed_up' ? 'Pendiente' : 'Expirado'}`
          ).join('\n');

          await sendText(phone, `*Mis referidos (${myRefs.length}):*\n\n${list}`, {
            agent: 'referral-program', action: 'referrals_listed',
          });
        }
        break;
      }

      case '3': { // My rewards
        if (referrer.rewards_earned.length === 0) {
          await sendText(phone, 'Aún no tiene recompensas. Necesita que sus referidos se registren y permanezcan activos.', {
            agent: 'referral-program', action: 'no_rewards',
          });
        } else {
          const list = referrer.rewards_earned.map(r =>
            `🎁 *${r.code}* — ${r.label}\n   Ganada el ${r.earned_at.split('T')[0]}`
          ).join('\n\n');

          await sendText(phone, `*Mis recompensas:*\n\n${list}`, {
            agent: 'referral-program', action: 'rewards_listed',
          });
        }
        break;
      }

      case '4': { // Leaderboard
        const leaderboard = [...referrers.values()]
          .sort((a, b) => b.qualified_referrals - a.qualified_referrals)
          .slice(0, 10)
          .map((r, i) => {
            const medal = i === 0 ? '🥇' : i === 1 ? '🥈' : i === 2 ? '🥉' : `${i + 1}.`;
            const isMe = r.phone === phone;
            return `${medal} ${isMe ? '*' : ''}${r.name || r.phone.slice(-4)}${isMe ? ' (Tú)*' : ''} — ${r.qualified_referrals} referidos`;
          });

        await sendText(phone, `🏆 *Top 10 Referidores*\n\n${leaderboard.join('\n')}`, {
          agent: 'referral-program', action: 'leaderboard_shown',
        });
        break;
      }

      case '5': { // How it works
        const tiers = Object.entries(REWARD_CONFIG.tier_bonuses)
          .map(([count, bonus]) => `  ${count} referidos → ${bonus.label}`)
          .join('\n');

        await sendText(phone,
          `📖 *¿Cómo funciona?*\n\n` +
          `1. Comparta su link personal con amigos\n` +
          `2. Cuando se registran, usted recibe una notificación\n` +
          `3. Si permanecen activos ${REWARD_CONFIG.qualification_days} días, califica\n` +
          `4. ¡Recibe su recompensa automáticamente!\n\n` +
          `*Recompensa por cada referido:*\n` +
          `- Usted: ${REWARD_CONFIG.referrer_reward_label}\n` +
          `- Su amigo/a: ${REWARD_CONFIG.referee_reward_label}\n\n` +
          `*Bonos por hitos:*\n${tiers}\n\n` +
          `Máximo ${REWARD_CONFIG.max_referrals_per_user} referidos por persona.`,
          { agent: 'referral-program', action: 'how_it_works' }
        );
        break;
      }

      default:
        await sendText(phone, 'Escriba *MENU* para ver las opciones del programa de referidos.', {
          agent: 'referral-program', action: 'unrecognized',
        });
        break;
    }
  } catch (err) {
    console.error(`Error from ${phone}:`, err.message);
  }
});

// ── Webhook: New user signup with referral code ───────────────────────
app.post('/api/referrals/signup', async (req, res) => {
  const { referral_code, phone, name } = req.body;
  if (!referral_code || !phone) {
    return res.status(400).json({ error: 'referral_code and phone are required' });
  }

  try {
    const result = await recordReferral(referral_code, phone, name);
    res.json(result);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── Webhook: Referral qualification (called after N days active) ──────
app.post('/api/referrals/:referralId/qualify', async (req, res) => {
  try {
    await qualifyReferral(req.params.referralId);
    res.json({ status: 'qualified' });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Stats ────────────────────────────────────────────────────────
app.get('/api/stats', (req, res) => {
  res.json({
    total_referrers: referrers.size,
    total_referrals: referrals.size,
    qualified: [...referrals.values()].filter(r => r.status === 'qualified' || r.status === 'rewarded').length,
    rewards_issued: [...referrers.values()].reduce((sum, r) => sum + r.rewards_earned.length, 0),
  });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    referrers: referrers.size,
    referrals: referrals.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Referral program bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "referral-program",
  "referral_code": "A1B2C3D4",
  "referral_link": "https://app.example.com/ref/A1B2C3D4",
  "total_referrals": 7,
  "qualified_referrals": 5,
  "rewards_earned": 5,
  "referrals": [
    { "id": "REF-M1A2B3", "referred_name": "Carlos López", "status": "rewarded", "signed_up_at": "2026-03-15T10:00:00.000Z" },
    { "id": "REF-M4D5E6", "referred_name": "Ana García", "status": "signed_up", "signed_up_at": "2026-04-01T14:00:00.000Z" }
  ]
}
```

Per-message metadata:

```json
{
  "agent": "referral-program",
  "action": "reward_earned",
  "referral_id": "REF-M1A2B3",
  "reward_code": "RWD-F1E2D3",
  "qualified_referrals": 5
}
```

## How to Run

```bash
# 1. Create the project
mkdir referral-program && cd referral-program
npm init -y
npm install express

# 2. Set environment variables
export OPERO_API_KEY="your-opero-api-key"
export OPERO_NUMBER_ID="your-number-id"
export REFERRAL_BASE_URL="https://app.example.com/ref"

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

# 6. Simulate a referral signup (from your app's signup flow)
curl -X POST http://localhost:3000/api/referrals/signup \
  -H "Content-Type: application/json" \
  -d '{
    "referral_code": "A1B2C3D4",
    "phone": "5491155551002",
    "name": "Carlos López"
  }'

# 7. Simulate qualification (after 7 days active)
curl -X POST http://localhost:3000/api/referrals/REF-xxx/qualify
```
