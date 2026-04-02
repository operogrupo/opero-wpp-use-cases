# 31 - Pet Care Reminders

Veterinary clinic reminders for vaccinations, checkups, and medications. Multi-pet support per owner with pet profiles stored in conversation metadata.

## Problem

A veterinary clinic manages 800+ active patients (pets). Vaccination schedules are tracked in spreadsheets, and staff spend hours calling pet owners for reminders. Missed vaccinations put pets at risk and reduce clinic revenue. Owners with multiple pets lose track of which pet needs what, and medication adherence for chronic conditions is poor without follow-up.

## Solution

Deploy a WhatsApp reminder bot that:
- Stores pet profiles (species, breed, age, medical history) in conversation metadata
- Sends vaccination reminders based on each pet's schedule
- Sends medication reminders at prescribed intervals
- Supports multiple pets per owner
- Allows owners to confirm appointments, request refills, or ask questions
- Tracks compliance history for each pet

## Architecture

```
+-------------------+     cron / API call     +--------------------+
|  Reminder Engine  | ----------------------> |  Your Express App  |
|  (daily check)    |                         |   (this code)      |
+-------------------+                         +--------------------+
                                                       |
         +---------------------------------------------+
         |                                             |
         v                                             v
+------------------+                          +------------------+
|   Opero WPP API  |                          | Pet Owner        |
|  wpp-api.opero.so|  <---  webhook  -------- | responds via WPP |
+------------------+                          +------------------+
         |
         v
  PUT /conversations/:phone
  (pet profiles + reminder history in metadata)
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

// ── Data stores (use a real DB in production) ─────────────────────────
const owners = new Map(); // phone -> { name, pets: Map<petId, pet> }
const reminders = new Map(); // reminderId -> reminder
const sessions = new Map(); // phone -> session state

// ── Vaccination schedules by species ──────────────────────────────────
const VAX_SCHEDULES = {
  dog: [
    { name: 'Antirrábica', interval_months: 12 },
    { name: 'Quíntuple', interval_months: 12 },
    { name: 'Tos de las perreras', interval_months: 6 },
    { name: 'Desparasitación', interval_months: 3 },
  ],
  cat: [
    { name: 'Antirrábica', interval_months: 12 },
    { name: 'Triple felina', interval_months: 12 },
    { name: 'Leucemia felina', interval_months: 12 },
    { name: 'Desparasitación', interval_months: 3 },
  ],
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

// ── Owner/pet management ──────────────────────────────────────────────
function getOwner(phone) {
  if (!owners.has(phone)) {
    owners.set(phone, { name: '', pets: new Map() });
  }
  return owners.get(phone);
}

function addPet(phone, petData) {
  const owner = getOwner(phone);
  const petId = `PET-${Date.now().toString(36).toUpperCase()}`;
  const pet = {
    id: petId,
    name: petData.name,
    species: petData.species, // dog, cat
    breed: petData.breed || '',
    age_years: petData.age_years || null,
    weight_kg: petData.weight_kg || null,
    medical_notes: petData.medical_notes || [],
    vaccinations: [], // { name, date, next_due }
    medications: [],  // { name, dosage, frequency, start_date, end_date }
    created_at: new Date().toISOString(),
  };
  owner.pets.set(petId, pet);
  return pet;
}

function addVaccination(phone, petId, vaxName, date) {
  const owner = getOwner(phone);
  const pet = owner.pets.get(petId);
  if (!pet) throw new Error('Pet not found');

  const schedule = VAX_SCHEDULES[pet.species]?.find(v => v.name === vaxName);
  const nextDue = new Date(date);
  if (schedule) {
    nextDue.setMonth(nextDue.getMonth() + schedule.interval_months);
  } else {
    nextDue.setFullYear(nextDue.getFullYear() + 1);
  }

  const record = {
    name: vaxName,
    date,
    next_due: nextDue.toISOString().split('T')[0],
  };

  // Replace existing or add new
  const existingIdx = pet.vaccinations.findIndex(v => v.name === vaxName);
  if (existingIdx >= 0) {
    pet.vaccinations[existingIdx] = record;
  } else {
    pet.vaccinations.push(record);
  }

  return record;
}

function addMedication(phone, petId, medData) {
  const owner = getOwner(phone);
  const pet = owner.pets.get(petId);
  if (!pet) throw new Error('Pet not found');

  const medication = {
    id: `MED-${Date.now().toString(36).toUpperCase()}`,
    name: medData.name,
    dosage: medData.dosage,
    frequency: medData.frequency, // daily, twice_daily, weekly
    start_date: medData.start_date,
    end_date: medData.end_date || null,
    active: true,
    reminders_sent: 0,
    confirmations: 0,
  };

  pet.medications.push(medication);
  return medication;
}

// ── Sync metadata to conversation ─────────────────────────────────────
async function syncMetadata(phone) {
  const owner = getOwner(phone);
  const petsArray = [...owner.pets.values()].map(pet => ({
    id: pet.id,
    name: pet.name,
    species: pet.species,
    breed: pet.breed,
    vaccinations: pet.vaccinations,
    medications: pet.medications.filter(m => m.active),
  }));

  await updateConversation(phone, {
    agent: 'pet-care-reminders',
    owner_name: owner.name,
    pets: petsArray,
    updated_at: new Date().toISOString(),
  });
}

// ── Reminder engine ───────────────────────────────────────────────────
async function checkVaccinationReminders() {
  const today = new Date().toISOString().split('T')[0];
  const reminderWindow = new Date();
  reminderWindow.setDate(reminderWindow.getDate() + 14);
  const windowStr = reminderWindow.toISOString().split('T')[0];

  for (const [phone, owner] of owners) {
    for (const [petId, pet] of owner.pets) {
      for (const vax of pet.vaccinations) {
        if (vax.next_due >= today && vax.next_due <= windowStr) {
          const reminderId = `${phone}-${petId}-${vax.name}`;
          if (reminders.has(reminderId)) continue;

          reminders.set(reminderId, {
            id: reminderId,
            type: 'vaccination',
            phone,
            pet_id: petId,
            sent_at: new Date().toISOString(),
          });

          const daysUntil = Math.ceil((new Date(vax.next_due) - new Date(today)) / (1000 * 60 * 60 * 24));

          try {
            await sendText(phone,
              `🐾 *Recordatorio de vacunación*\n\n` +
              `Mascota: *${pet.name}* (${pet.breed || pet.species})\n` +
              `Vacuna: *${vax.name}*\n` +
              `Vencimiento: *${vax.next_due}* (en ${daysUntil} días)\n\n` +
              `Responda:\n` +
              `*1* - Agendar turno\n` +
              `*2* - Ya fue vacunado/a\n` +
              `*3* - Recordar la semana próxima`,
              {
                agent: 'pet-care-reminders',
                reminder_type: 'vaccination',
                pet_id: petId,
                pet_name: pet.name,
                vaccine: vax.name,
                due_date: vax.next_due,
              }
            );
          } catch (err) {
            console.error(`Failed to send vax reminder for ${pet.name}:`, err.message);
          }
        }
      }
    }
  }
}

async function checkMedicationReminders() {
  const today = new Date().toISOString().split('T')[0];

  for (const [phone, owner] of owners) {
    for (const [petId, pet] of owner.pets) {
      for (const med of pet.medications) {
        if (!med.active) continue;
        if (med.end_date && med.end_date < today) {
          med.active = false;
          continue;
        }

        try {
          await sendText(phone,
            `💊 *Recordatorio de medicación*\n\n` +
            `Mascota: *${pet.name}*\n` +
            `Medicamento: *${med.name}*\n` +
            `Dosis: ${med.dosage}\n\n` +
            `¿Ya le dió la medicación? Responda *SI* para confirmar.`,
            {
              agent: 'pet-care-reminders',
              reminder_type: 'medication',
              pet_id: petId,
              pet_name: pet.name,
              medication: med.name,
              medication_id: med.id,
            }
          );
          med.reminders_sent++;
        } catch (err) {
          console.error(`Failed to send medication reminder for ${pet.name}:`, err.message);
        }
      }
    }
  }
}

// Run checks periodically
setInterval(checkVaccinationReminders, 24 * 60 * 60 * 1000); // Daily
setInterval(checkMedicationReminders, 12 * 60 * 60 * 1000);  // Twice daily

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

  console.log(`[${new Date().toISOString()}] Message from ${phone}: ${messageText}`);

  try {
    await markAsRead(phone, [data.wa_message_id || data.id]);

    const upper = messageText.toUpperCase();
    const owner = getOwner(phone);
    const session = sessions.get(phone) || { step: 'idle' };

    // Global commands
    if (upper === 'MENU' || upper === 'HOLA') {
      sessions.delete(phone);
      const petCount = owner.pets.size;
      await sendText(phone,
        `🐾 *Veterinaria San Roque*\n\n` +
        `${petCount > 0 ? `Mascotas registradas: ${petCount}` : 'No tiene mascotas registradas'}\n\n` +
        `¿Qué desea hacer?\n\n` +
        `*1* - Registrar una mascota\n` +
        `*2* - Ver mis mascotas\n` +
        `*3* - Ver calendario de vacunas\n` +
        `*4* - Solicitar turno\n` +
        `*5* - Consulta general`,
        { agent: 'pet-care-reminders', action: 'menu_shown' }
      );
      return;
    }

    if (upper === 'SI') {
      // Medication confirmation
      await sendText(phone, '✅ Medicación confirmada. ¡Buen trabajo cuidando a su mascota!', {
        agent: 'pet-care-reminders', action: 'medication_confirmed',
      });
      return;
    }

    switch (session.step) {
      case 'idle': {
        switch (messageText.trim()) {
          case '1': // Register pet
            sessions.set(phone, { step: 'register_name', data: {} });
            await sendText(phone, '¿Cuál es el nombre de su mascota?', {
              agent: 'pet-care-reminders', step: 'register_name',
            });
            break;

          case '2': // View pets
            if (owner.pets.size === 0) {
              await sendText(phone, 'No tiene mascotas registradas. Escriba *1* para registrar una.', {
                agent: 'pet-care-reminders', action: 'no_pets',
              });
            } else {
              const petList = [...owner.pets.values()].map(pet => {
                const nextVax = pet.vaccinations
                  .filter(v => v.next_due >= new Date().toISOString().split('T')[0])
                  .sort((a, b) => a.next_due.localeCompare(b.next_due))[0];
                const activeMeds = pet.medications.filter(m => m.active).length;
                return `🐾 *${pet.name}* (${pet.breed || pet.species})\n` +
                  `   ID: ${pet.id}\n` +
                  `   ${nextVax ? `Próxima vacuna: ${nextVax.name} - ${nextVax.next_due}` : 'Vacunas al día'}\n` +
                  `   Medicamentos activos: ${activeMeds}`;
              }).join('\n\n');
              await sendText(phone, `*Sus mascotas:*\n\n${petList}`, {
                agent: 'pet-care-reminders', action: 'pets_listed',
              });
            }
            break;

          case '3': // Vaccine calendar
            if (owner.pets.size === 0) {
              await sendText(phone, 'Registre una mascota primero con la opción *1*.', {
                agent: 'pet-care-reminders', action: 'no_pets',
              });
            } else {
              const calendar = [...owner.pets.values()].map(pet => {
                const vaxList = pet.vaccinations.length > 0
                  ? pet.vaccinations.map(v => `  - ${v.name}: última ${v.date} → próxima *${v.next_due}*`).join('\n')
                  : '  Sin vacunas registradas';
                return `🐾 *${pet.name}*\n${vaxList}`;
              }).join('\n\n');
              await sendText(phone, `*Calendario de vacunas:*\n\n${calendar}`, {
                agent: 'pet-care-reminders', action: 'calendar_shown',
              });
            }
            break;

          case '4': // Request appointment
            await sendText(phone, 'Para agendar un turno, llame al +54 11 4444-7777 o escriba el motivo de la consulta y lo contactaremos.', {
              agent: 'pet-care-reminders', action: 'appointment_request',
            });
            break;

          case '5': // General question
            sessions.set(phone, { step: 'question' });
            await sendText(phone, 'Escriba su consulta y un veterinario le responderá dentro de las 24 horas.', {
              agent: 'pet-care-reminders', step: 'question',
            });
            break;

          default:
            await sendText(phone, 'Bienvenido/a a la Veterinaria San Roque. Escriba *MENU* para ver las opciones.', {
              agent: 'pet-care-reminders', action: 'welcome',
            });
            break;
        }
        break;
      }

      case 'register_name':
        session.data.name = messageText;
        session.step = 'register_species';
        sessions.set(phone, session);
        await sendText(phone, `¿Qué tipo de mascota es ${messageText}?\n\n*1* - Perro\n*2* - Gato`, {
          agent: 'pet-care-reminders', step: 'register_species',
        });
        break;

      case 'register_species': {
        const speciesMap = { '1': 'dog', '2': 'cat' };
        const species = speciesMap[messageText.trim()];
        if (!species) {
          await sendText(phone, 'Por favor elija *1* (Perro) o *2* (Gato).', {
            agent: 'pet-care-reminders', action: 'invalid_species',
          });
          return;
        }
        session.data.species = species;
        session.step = 'register_breed';
        sessions.set(phone, session);
        await sendText(phone, `¿Cuál es la raza de ${session.data.name}? (escriba "mestizo" si no sabe)`, {
          agent: 'pet-care-reminders', step: 'register_breed',
        });
        break;
      }

      case 'register_breed':
        session.data.breed = messageText;
        session.step = 'register_age';
        sessions.set(phone, session);
        await sendText(phone, `¿Cuántos años tiene ${session.data.name}? (número aproximado)`, {
          agent: 'pet-care-reminders', step: 'register_age',
        });
        break;

      case 'register_age': {
        const age = parseFloat(messageText);
        session.data.age_years = isNaN(age) ? null : age;

        const pet = addPet(phone, session.data);
        sessions.delete(phone);

        const speciesLabel = session.data.species === 'dog' ? 'perro' : 'gato';
        const vaxNeeded = VAX_SCHEDULES[session.data.species]?.map(v => `- ${v.name} (cada ${v.interval_months} meses)`).join('\n');

        await sendText(phone,
          `✅ *Mascota registrada*\n\n` +
          `Nombre: ${pet.name}\n` +
          `Tipo: ${speciesLabel}\n` +
          `Raza: ${pet.breed}\n` +
          `Edad: ${pet.age_years || 'no especificada'} años\n` +
          `ID: ${pet.id}\n\n` +
          `Vacunas recomendadas:\n${vaxNeeded}\n\n` +
          `Cuando vacune a ${pet.name}, infórmenos para programar los recordatorios.`,
          {
            agent: 'pet-care-reminders',
            action: 'pet_registered',
            pet_id: pet.id,
            pet_name: pet.name,
            species: pet.species,
          }
        );

        await syncMetadata(phone);
        break;
      }

      case 'question':
        sessions.delete(phone);
        await sendText(phone, 'Gracias por su consulta. Un veterinario le responderá pronto. Escriba *MENU* para volver al inicio.', {
          agent: 'pet-care-reminders', action: 'question_received', question: messageText,
        });

        // Notify vet staff
        const VET_PHONE = process.env.VET_PHONE || '5491155550000';
        await sendText(VET_PHONE, `📩 Consulta de ${owner.name || phone}:\n\n${messageText}`, {
          agent: 'pet-care-reminders', action: 'vet_notification', client_phone: phone,
        });
        break;

      default:
        sessions.delete(phone);
        await sendText(phone, 'Escriba *MENU* para ver las opciones.', {
          agent: 'pet-care-reminders', action: 'reset',
        });
        break;
    }
  } catch (err) {
    console.error(`Error handling message from ${phone}:`, err.message);
  }
});

// ── API: Register pet ─────────────────────────────────────────────────
app.post('/api/owners/:phone/pets', (req, res) => {
  try {
    const owner = getOwner(req.params.phone);
    if (req.body.owner_name) owner.name = req.body.owner_name;
    const pet = addPet(req.params.phone, req.body);
    res.json(pet);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Record vaccination ───────────────────────────────────────────
app.post('/api/owners/:phone/pets/:petId/vaccinations', async (req, res) => {
  try {
    const { name, date } = req.body;
    const record = addVaccination(req.params.phone, req.params.petId, name, date);
    await syncMetadata(req.params.phone);
    res.json(record);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: Add medication ───────────────────────────────────────────────
app.post('/api/owners/:phone/pets/:petId/medications', async (req, res) => {
  try {
    const medication = addMedication(req.params.phone, req.params.petId, req.body);
    await syncMetadata(req.params.phone);
    res.json(medication);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ── API: View owner profile ───────────────────────────────────────────
app.get('/api/owners/:phone', (req, res) => {
  const owner = owners.get(req.params.phone);
  if (!owner) return res.status(404).json({ error: 'Owner not found' });
  res.json({
    name: owner.name,
    pets: [...owner.pets.values()],
  });
});

// ── API: Trigger reminder checks ──────────────────────────────────────
app.post('/api/reminders/check', async (req, res) => {
  await checkVaccinationReminders();
  res.json({ status: 'checked' });
});

// ── Health check ──────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    owners: owners.size,
    total_pets: [...owners.values()].reduce((sum, o) => sum + o.pets.size, 0),
    active_reminders: reminders.size,
    uptime: process.uptime(),
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Pet care reminders bot listening on port ${PORT}`);
});
```

## Metadata Schema

Conversation metadata:

```json
{
  "agent": "pet-care-reminders",
  "owner_name": "Carolina Fernández",
  "pets": [
    {
      "id": "PET-M1A2B3",
      "name": "Luna",
      "species": "dog",
      "breed": "Golden Retriever",
      "vaccinations": [
        { "name": "Antirrábica", "date": "2025-10-15", "next_due": "2026-10-15" },
        { "name": "Quíntuple", "date": "2025-10-15", "next_due": "2026-10-15" }
      ],
      "medications": [
        { "name": "Heartgard", "dosage": "1 comprimido", "frequency": "monthly", "active": true }
      ]
    }
  ],
  "updated_at": "2026-04-02T14:00:00.000Z"
}
```

Per-message metadata:

```json
{
  "agent": "pet-care-reminders",
  "reminder_type": "vaccination",
  "pet_id": "PET-M1A2B3",
  "pet_name": "Luna",
  "vaccine": "Antirrábica",
  "due_date": "2026-10-15"
}
```

## How to Run

```bash
# 1. Create the project
mkdir pet-care-reminders && cd pet-care-reminders
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

# 6. Register a pet via the API
curl -X POST http://localhost:3000/api/owners/5491155559999/pets \
  -H "Content-Type: application/json" \
  -d '{
    "owner_name": "Carolina Fernández",
    "name": "Luna",
    "species": "dog",
    "breed": "Golden Retriever",
    "age_years": 3
  }'

# 7. Record a vaccination
curl -X POST http://localhost:3000/api/owners/5491155559999/pets/PET-xxx/vaccinations \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Antirrábica",
    "date": "2026-04-01"
  }'
```
