# Claude Cowork Skills Portfolio

A collection of production Claude Cowork skills built for a digital marketing agency's sales and operations team. These skills connect Claude to real business tools CRM, Gmail, n8n automation, Apollo.io, and more replacing manual work with AI-driven workflows.

---

## What This Is

Claude Cowork lets you give Claude access to your business tools and then build "skills" structured instructions that tell Claude exactly how to handle a specific task from start to finish. Think of each skill as a custom automation that Claude reads before acting.

This repo contains four skills actively used in production at a agency handling 50+ inbound leads per month across Upwork, LinkedIn, and website channels.

---

## Skills Included

Note: Skill.md files contains examples of my previous clients, that can't be used as a pure prompt for any automation.

| Skill | What It Does | Tools Used |
|---|---|---|
| [Lead Capture & CRM](#1-lead-capture--crm-automation) | Screenshot → ClientJoy lead + AI reply in under 2 minutes | n8n, ClientJoy, Conversation Manager |
| [Cold Email Outreach](#2-cold-email-outreach) | Finds a quiet lead's email and drafts a personalised follow-up | Apollo.io, Gmail, Leads DB |
| [Sales Intelligence Agent](#3-sales-intelligence-agent) | Researches any prospect and builds a full sales battle plan | Web search, LinkedIn, Apollo.io |
| [n8n Workflow Automation](#4-n8n-workflow-automation) | Documents how Claude orchestrates multi-step n8n workflows | n8n MCP |

---

## How Claude Cowork Skills Work

Each skill lives in its own folder with a `SKILL.md` file. When you set up Claude Cowork, you point it at this folder. Claude reads the skill description to decide when to use it, then reads the full skill body to know what to do step by step.

```
skill-name/
├── SKILL.md          ← Claude reads this when the skill triggers
├── examples/         ← Real input/output examples
└── docs/             ← Supporting reference material
```

The skill description (in the YAML frontmatter) is the trigger it tells Claude when to activate the skill without being asked explicitly. The body contains the step-by-step process, hard rules, edge cases, and API call formats.

---

## 1. Lead Capture & CRM Automation

**Folder:** `skills/lead-capture-crm/`

Takes an inbound Upwork, LinkedIn, or website lead screenshot and runs the full intake pipeline automatically no manual data entry.

### What Happens

1. Claude reads the screenshot and extracts the client name, message, location, and channel
2. Auto-assigns the lead to the right BDE based on which Upwork profile received it (or time of day for website leads)
3. Creates the lead in ClientJoy via an n8n webhook
4. Logs background info + the verbatim client message to the Conversation Manager
5. Generates a personalised reply via the Supervisor_Sales n8n workflow
6. Presents both a full version and a shorter version of the reply for the user to pick and send

### Time Saved

Manual version: ~15 minutes (copy to CRM, write reply, log everything)
With this skill: ~90 seconds

### Tools Connected

- `n8n MCP` — triggers the ClientJoy creation workflow and the Supervisor_Sales AI workflow
- `Leads Conversation Manager` — logs all entries
- `Leads Database` — for lookups on returning leads

### Key Design Decisions

- **No routing questions asked** BDE assignment is fully automated using a profile-to-BDE lookup table baked into the skill
- **Verbatim logging** the client's exact message is always logged word for word, never summarised
- **Dual reply format** the AI generates both a long and a short version so the user can pick based on context

---

## 2. Cold Email Outreach

**Folder:** `skills/cold-email-outreach/`

When a lead goes quiet on Upwork or LinkedIn, this skill finds their email address, generates a personalised follow-up, and creates a Gmail draft ready to send with one click.

### What Happens

1. Looks up the lead in the Leads Database by ID or name
2. Pulls the full conversation history from the Conversation Manager
3. Searches for the lead's email in this order: conversation history → database → web search → Apollo.io search + enrich → pattern-guess on verified employer domain
4. Generates a short, specific follow-up body using the Supervisor_Sales n8n workflow (referencing real details from earlier conversations)
5. Prepends an intro line: `Hi [Name], [Persona First Name] this side.`
6. Creates a Gmail draft in Verdana 14px from `marketing@incrementors.com`
7. Logs everything back to the Conversation Manager

### Rules Enforced By the Skill

- Sender is always the Upwork/LinkedIn persona (e.g. "Kuldeep"), never the real name
- No em dashes anywhere caught before creating the draft
- No signature at the bottom
- Under 120 words total
- Verdana 14px via HTML body consistent formatting across all outreach
- Logs which email source was used (Apollo verified / web search / pattern-guess)

### Apollo.io Integration Pattern

A key part of this skill is the two-stage Apollo lookup that actually works:

1. `apollo_mixed_people_api_search` with name + location keywords → gets a person ID
2. `apollo_people_match` with that person ID + `reveal_personal_emails: true` → gets the email

Single-step name-only lookups return empty stubs. The two-stage pattern is documented in the skill with the exact parameters.

---

## 3. Sales Intelligence Agent

**Folder:** `skills/sales-intelligence-agent/`

Researches any prospect using web search, LinkedIn, company data, and news — then outputs a structured sales battle plan with specific opening hooks, objection counters, and a multi-touch outreach sequence.

### Output Structure

Every brief covers 8 intelligence categories:

1. **Pain point discovery** specific, evidence-backed challenges
2. **Buying signals** — trigger events that indicate purchase readiness
3. **Decision-making intelligence** who controls the budget and how they buy
4. **Tech stack** current tools, integration needs, vendor signals
5. **Personalization hooks** recent posts, wins, shared context
6. **Value alignment** — maps each pain point to a specific service
7. **Objection pre-emption** top 2 objections with specific counter-language
8. **Social proof** relevant case studies and credibility bridges

### Final Deliverable

A complete sales battle plan with:
- Deal potential score (Hot / Warm / Cold)
- Personalised opening hook (specific, ready to copy-paste)
- Multi-touch sequence (LinkedIn Day 1 → Email Day 3 → Call Day 7 → Video Day 10)
- Deal acceleration checklist
- Estimated contract size and win probability

### Design Principle

Every output section answers the question: "How does this help me close the deal?" Generic filler is explicitly banned in the skill instructions.

---

## 4. n8n Workflow Automation

**Folder:** `skills/n8n-workflow-automation/`

Documents the patterns used to orchestrate multi-step n8n workflows from Claude covering trigger types, execution polling, response extraction, and error handling.

### Workflows in Use

| Workflow | ID | Trigger Type | Purpose |
|---|---|---|---|
| Add Leads To ClientJoy | `hkSIbx72qd63HyYh` | form | Creates a new lead in the CRM |
| Supervisor_Sales | `DUKNqcvyjpLEV9b5` | chat | Generates AI replies and follow-ups |
| Lead Conversation History | `YdXyxvGcof2CNmlX` | webhook | Fetches full lead history |
| Sales Project Onboarding | `X5vmnKDaAkFdP9nt` | form | Hands a closed client to ops |

### Key Patterns Documented

- How to handle `type: "form"` vs `type: "chat"` triggers — they require different payload shapes
- How to poll for execution results using `get_execution` when workflows return `status: "waiting"`
- How to extract nested response data from the n8n result object
- How to handle timeouts (retry once, then fall back to manual)
- The Google Sheets safety rule: never start a logged field with `=`, `+`, `-`, `@`, or `|`

---

## Stack Overview

| Layer | Tool |
|---|---|
| AI Orchestration | Claude Cowork (Anthropic) |
| Workflow Automation | n8n (self-hosted) |
| CRM | ClientJoy |
| Email | Gmail via MCP connector |
| Lead Data | Leads Database MCP (custom Cloudflare Worker) |
| Conversation Logs | Leads Conversation Manager MCP (custom Cloudflare Worker) |
| Contact Discovery | Apollo.io MCP |
| Web Research | Firecrawl MCP + web search |

---

## Setup

These skills are designed for Claude Cowork. To use them:

1. Connect the required MCP tools (n8n, Gmail, Apollo.io, your Leads Database)
2. Add the skill files to your Claude Cowork project
3. Claude will read the skill descriptions and activate the right skill based on what you ask

Each skill folder contains a `docs/setup.md` with the specific tool requirements and configuration notes for that skill.

---

## License

MIT — use and adapt freely.
