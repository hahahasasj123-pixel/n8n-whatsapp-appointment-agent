# n8n WhatsApp Appointment Agent

An n8n AI agent that handles appointment booking end-to-end over WhatsApp: books, reschedules, cancels and looks up appointments, checks real-time availability against Google Calendar, stores clients/appointments in Google Sheets, and escalates to a human when needed.

Built for a barbershop originally and generalized here so it fits any appointment-based business (salons, clinics, personal trainers, consultants, etc.).

## What it does

- Receives WhatsApp messages via [Evolution API](https://github.com/EvolutionAPI/evolution-api) webhook
- Handles both text and audio messages (audio requires a transcription step — not included, see below)
- Normalizes phone numbers to a consistent format
- Runs an LLM agent (via OpenRouter, swap for OpenAI/Anthropic directly if preferred) with a detailed system prompt covering the full booking lifecycle
- Prevents double-booking with a real-time re-check at the moment of creating the event
- Uses NSFW/PII guardrails on both inbound and outbound messages
- Escalates to a human via a separate sub-workflow when the agent can't resolve something or the customer asks for a person

## Architecture

```
Webhook (Evolution API) → filter private chats → route by message type
  → normalize data (phone, message text, reference calendar)
  → guardrails (NSFW check)
  → AI Agent (LLM + memory + tools)
      tools: check availability, create/update/delete calendar event,
             find/register client, save/update appointment, escalate to human
  → guardrails (PII/secrets sanitize on output)
  → send WhatsApp reply
```

## Requirements

- Self-hosted or cloud n8n instance with the [LangChain nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/) enabled
- [Evolution API](https://github.com/EvolutionAPI/evolution-api) instance connected to a WhatsApp number (or swap this node for the official WhatsApp Business API / another provider)
- Google Calendar + Google Sheets credentials
- An LLM provider credential (OpenRouter, OpenAI, etc.)

## Setup

1. Import `workflow.json` into your n8n instance.
2. Create the credentials referenced by the nodes (Evolution API, Google Calendar OAuth2, Google Sheets OAuth2, your LLM provider) and reassign them on each node.
3. Replace every `{{PLACEHOLDER}}` in the system prompt and node parameters:

   | Placeholder | Description |
   |---|---|
   | `{{BUSINESS_NAME}}` | Business name |
   | `{{BUSINESS_ADDRESS}}` | Address shown to customers |
   | `{{BUSINESS_WHATSAPP}}` | Contact number |
   | `{{HORARIO_ATENCION}}` | Opening hours |
   | `{{TIMEZONE}}` | IANA timezone, e.g. `America/Argentina/Cordoba` |
   | `{{STAFF_NAME}}` / `{{STAFF_DESCRIPTION}}` | Staff member(s) — extend the prompt if there's more than one |
   | `{{TABLA_SERVICIOS}}` / `{{SERVICIOS_DISPONIBLES}}` | Services offered, price and duration |
   | `{{ANTELACION_MINIMA}}` / `{{ANTELACION_MAXIMA}}` / `{{DIAS_VALIDOS}}` | Booking rules |
   | `{{COUNTRY_CODE}}` / `{{UTC_OFFSET_HOURS}}` | Used by the phone-normalization and date-reference code |
   | `{{GOOGLE_CALENDAR_ID}}` / `{{GOOGLE_SHEET_ID}}` | Your Calendar and Sheet IDs |
   | `{{SUBWORKFLOW_ID_DISPONIBILIDAD}}` / `{{SUBWORKFLOW_ID_CREAR_TURNO}}` / `{{SUBWORKFLOW_ID_ESCALADO}}` | See below |

4. Build the three sub-workflows this agent calls (kept out of this repo since they're business-specific plumbing, not part of the agent's reasoning):
   - **Availability checker** — given `fecha_consulta` + `servicio`, returns free time slots (opening hours minus existing Calendar events).
   - **Verified appointment creator** — re-checks the exact slot is still free at creation time (guards against a race between two customers booking the same slot) and creates the Calendar event.
   - **Human escalation** — sends a notification (email/WhatsApp) to staff with the conversation context.
5. Set up a Google Sheet with two tabs: `Clientes` (Nombre, Telefono, Email, Fecha_nacimiento, Notas, Fecha_registro) and `Turnos` (ID, Event_ID_Google, Cliente_Nombre, Cliente_Telefono, Fecha_Cita, Hora_Cita, Servicio, Profesional, Estado, Notas, Recordatorio_24h, Recordatorio_1h, Fecha_creacion).
6. Activate the workflow and point your Evolution API instance's webhook at the URL n8n generates.

## Notes

- The system prompt is intentionally strict about tool-call ordering to avoid double-booking — read the rules before changing them if you fork this.
- Audio transcription (e.g. via Whisper) is referenced in the code node but not included as a node here; add one before "Preparar Datos" if you need it.
- No credentials, spreadsheet IDs, calendar IDs or business data are included in this repo — everything sensitive is a placeholder.

## License

MIT
