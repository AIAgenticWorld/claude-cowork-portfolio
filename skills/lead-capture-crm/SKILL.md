---
name: upwork-lead-capture
description: >
  End-to-end capture of an inbound lead from a screenshot (Upwork, LinkedIn, or Website).
  Use this skill whenever the user uploads or pastes a screenshot of a client inquiry and
  wants it captured in the CRM with an AI-generated reply. Trigger on phrases like
  "capture this lead", "add this to CRM", "log this lead and generate response",
  "new Upwork message", "new LinkedIn lead", or simply uploading a screenshot of an
  inbound message from a prospect. Handles channel-based BDE assignment automatically —
  no routing questions asked.
---

# Lead Capture & CRM Automation

Captures an inbound lead from a screenshot and runs the full intake pipeline:

1. Extract lead data from the screenshot
2. Auto-assign to the right BDE based on channel and profile
3. Create lead in the CRM via n8n
4. Log background info + verbatim client message to Conversation Manager
5. Generate a reply via the Supervisor_Sales n8n workflow
6. Present full and short versions of the reply for the user to pick

No routing questions are asked. Assignment is automatic based on the source channel, profile name, and time of day.

---

## Step 0 - Extract Lead Info From the Screenshot

Look at the screenshot and extract:

- **Client name** (required — never guess; ask if unclear)
- **Source** — infer from the UI:
  - Upwork interface → `Upwork`
  - LinkedIn interface → `LinkedIn`
  - Website contact form / inbound email → `Website`
  - Word-of-mouth / referral mentioned → `Referral`
- **Profile name** — for Upwork only. Usually shown in the URL, sidebar, or as "Hi [Profile Name]" in the message. If ambiguous, ask.
- **Client message** — full verbatim text
- **Client location** — if visible; default to `USA` if not shown
- **Email, phone, company, website** — usually not visible on Upwork; leave empty if not shown
- **Upwork profile signals** (if Upwork) — jobs posted, hire rate, total spent, avg rate

---

## Step 1 - Auto-Assign the BDE

### Upwork

Look up the profile name to find the assigned BDE. Each Upwork profile maps to a fixed BDE. If the profile is not in your mapping table, ask the user which BDE to assign.

### LinkedIn

Always assign to the LinkedIn BDE (fixed assignment, no variation).

### Website

Check the time the message was received:
- USA business hours (roughly 18:30 - 08:30 IST) → USA-facing BDE
- India business hours (roughly 08:30 - 18:30 IST) → India-facing BDE

If ambiguous, use the client's stated location to decide.

### Referral

Ask the user who to assign. No default rule.

---

## Step 2 - Create the Lead in the CRM

Trigger the CRM creation workflow via `n8n MCP:execute_workflow`:

- Use `type: "form"` as the trigger type (not webhook)
- Pass all required fields: name, source, assignee, email, phone, company, website, country
- Send all fields even if some are empty strings

After execution, retrieve the execution result using `n8n MCP:get_execution` with the returned execution ID. Extract the Lead ID from the response — you will need it for all subsequent steps.

If execution fails, report the error and stop.

---

## Step 3 - Log to Conversation Manager

Create two entries in order, using the Lead ID from Step 2.

**Entry 1 — Client Background Info**

Include:
- Name, email, phone, company, website, country, source
- Assigned BDE name and email
- Upwork profile name (Upwork leads only)
- Any Upwork profile signals visible in the screenshot

**Entry 2 — Client Message**

Log the full verbatim text of the client's message exactly as it appears in the screenshot. Do not summarise or rephrase.

---

## Step 4 - Generate the Reply

Trigger the Supervisor_Sales workflow via `n8n MCP:execute_workflow`:

- Use `type: "chat"` as the trigger type
- Pass `chatInput` as a JSON string (not an object) containing:
  - `action`: `"Generate Message Reply"`
  - `lead_id`: the Lead ID from Step 2
  - `client_message`: the full client message text

The workflow returns `status: "waiting"` when the AI Agent finishes — this is normal. Call `n8n MCP:get_execution` to retrieve the output.

The response will contain both a Full Message and a Smaller Version. Present both to the user.

---

## Step 5 - Present Output and Wait for User Decision

Summarise what was done:
- Lead created in CRM with the assigned BDE
- Conversation Manager: Background Info and Client Message logged
- Supervisor_Sales reply generated

Present both reply versions clearly. Flag anything that needs manual correction.

Ask: "Which version do you want to send, or do you want to edit before sending?"

---

## Step 6 - Log the Final Sent Reply

Once the user confirms which version they sent (or pastes their edited version), log it to the Conversation Manager:

- Message type: `Our Reply`
- Response type: `Manual` if the user edited; `AI` if they sent the Supervisor_Sales output verbatim
- Details: the exact final text that was sent

---

## Hard Rules

1. Never ask routing questions for Upwork, LinkedIn, or Website leads — use the assignment logic in Step 1
2. Always log the client's message verbatim — never summarise
3. The `type` parameter for the CRM workflow is `"form"`, not `"webhook"` — do not mix these up
4. The `type` parameter for the Supervisor_Sales workflow is `"chat"` with a JSON-stringified `chatInput`
5. Always send all CRM fields even if some are empty strings
6. Do not send the reply on behalf of the user — only log it after they confirm
7. No em dashes anywhere in output, logs, or summaries — use plain hyphens
8. No AI-sounding language in anything visible to the user

---

## See Also

- `examples/upwork-lead-example.md` — annotated walkthrough of a real Upwork lead capture
- `examples/linkedin-lead-example.md` — LinkedIn source example with fixed BDE routing
- `docs/setup.md` — required MCP tools and n8n workflow IDs
