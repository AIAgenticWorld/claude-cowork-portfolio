---
name: cold-email-outreach
description: >
  Reach out to an inbound lead over email when they've gone quiet on the original channel
  (Upwork, LinkedIn, or website). Use this skill whenever the user says "email this lead",
  "reach out to [client name] over email", "cold email this client", "find their email and
  message them", "client is not active on Upwork, email them", or provides a Lead ID and
  asks to contact them off-platform. This skill finds the lead, discovers their email
  address through a multi-stage process (conversation history, database, web search,
  Apollo.io, domain pattern), generates a short personalised follow-up, and creates a
  Gmail draft ready to send. Always use this skill for any off-platform email outreach
  to an existing lead.
---

# Cold Email Outreach

Moves a quiet inbound lead onto email. The skill finds the lead, locates the best email
address available, writes a short and specific follow-up that references the earlier
conversation, and creates a Gmail draft from the agency's outreach account.

Tool chain: Leads Database → Conversation Manager → email discovery → n8n Supervisor_Sales → Gmail create_draft → Conversation Manager log.

---

## Critical Rules (Read These First)

**Rule 1: Sender persona comes from the lead record, not the user's real name.**

The lead was originally contacted under an Upwork or LinkedIn persona. The `Secondary Source` field on the lead record holds this name (e.g. "Kuldeep Singh Tomar"). Use the FIRST NAME only.

- Correct: `Hi David, Kuldeep this side.`
- Wrong: `Hi David, Kuldeep Singh Tomar this side.`
- Wrong: `Hi David, Rahul this side.` (real name, not the persona)

If Secondary Source is empty, ask the user which persona to send from before proceeding.

**Rule 2: No em dashes anywhere.**

Scan the full assembled email after generating it. Replace every em dash (—) with a plain hyphen (-) or rephrase. This is the most common issue to catch.

**Rule 3: No signature.**

No "Best regards", no name block, no company footer. The email ends on the last content line (usually the CTA question). If the AI generates a sign-off, strip it.

**Rule 4: Under 120 words total.**

Intro line + body. Trim anything above 150 words before creating the draft.

**Rule 5: Intro line is mandatory.**

The recipient won't recognise the sender's domain. Always open with:

```
Hi [First Name], [Persona First Name] this side.
```

Then a one-sentence bridge explaining the channel context (e.g. "We were chatting on Upwork about your hotel project...").

**Rule 6: Email discovery order is strict.**

Conversation history → Lead Database → web search → Apollo.io (two-stage) → domain pattern-guess. Do not skip steps. Log which source the email came from.

**Rule 7: Gmail drafts only — no direct send.**

The Gmail connector creates drafts. Never claim the email was "sent". Always say "draft created". The user clicks Send in Gmail.

**Rule 8: Verdana 14px via HTML body.**

Every draft uses `htmlBody` with inline CSS: `font-family: Verdana, Geneva, sans-serif; font-size: 14px`.

---

## Step 1 — Resolve the Lead

If the user provides a Lead ID: look it up directly in the Leads Database.

If the user provides a name: search the database. If multiple leads match, show the candidates (name, company, source, date) and ask the user to pick.

Capture from the lead record:
- Lead ID
- Full name and first name
- Company name
- Source (Upwork / LinkedIn / Website)
- Secondary Source — this is the sender persona; take the first name only
- Any email already on file
- BDE name (context only)

---

## Step 2 — Pull Conversation History

Retrieve all entries from the Conversation Manager for this Lead ID.

Read everything — original inquiry, every client message, call transcripts, replies already sent, follow-ups already sent. Capture:
- What the client originally asked for
- Their business details and pain points
- Any email addresses mentioned anywhere in the history
- Any personal or company domain mentioned
- How long since the last client message
- Any pricing or timeline discussed

This context feeds the reply generation in Step 5.

---

## Step 3 — Find the Email Address

Work through this list in order. Stop at the first verified or high-confidence result.

**3a. Conversation history** — scan everything from Step 2. If an email is there, use it.

**3b. Lead Database email field** — if populated, use it.

**3c. Web search** — run 2-3 targeted queries:
- `"First Last" "Company Name" email`
- `"First Last" site:linkedin.com/in/` (to find their LinkedIn URL for Apollo)
- `"Company Name" contact OR team OR about`

Include a disambiguator (city, company, industry) for common names to avoid false positives.

**3d. Apollo.io — two-stage lookup**

Stage 1 — Search:
- Use `apollo_mixed_people_api_search` with name + a distinguishing keyword (industry, role)
- Filter by country or city from the lead record
- Get up to 10 candidates, identify the right one by title and company

Stage 2 — Enrich:
- Use `apollo_people_match` with the Apollo person ID from Stage 1 (or a LinkedIn URL if you found one)
- Set `reveal_personal_emails: true`
- Check the `email` field, `email_status`, and `personal_emails` array

Do NOT use name-only lookups for Stage 1 — they return empty stubs. Always search first, then enrich with the ID.

**3e. Domain pattern-guess (last resort)**

Only if you can verify the employer domain actually exists as a real website. Try common patterns: `firstname@domain.com`, `firstname.lastname@domain.com`, `f.lastname@domain.com`.

Log whichever email source was used.

---

## Step 4 — Build the Context Summary

Prepare a brief for the n8n reply generation. Include:
- Original inquiry summary
- What was discussed or offered
- How long it's been quiet
- Any specific details worth referencing (project name, pain point, company context)
- The channel the lead came from

---

## Step 5 — Generate the Reply Body

Trigger the Supervisor_Sales n8n workflow with:
- Action: `"Generate Followup"`
- Lead ID
- Client message: the context summary from Step 4

After receiving the output:
- Strip any sign-off lines
- Replace all em dashes with hyphens
- Check the word count — trim if above 150

---

## Step 6 — Assemble the Full Email

**Subject line:**

```
Following up on our [Upwork / LinkedIn] chat, [FirstName]
```

Adjust if the context calls for something more specific (e.g. referencing the project name).

**Body structure:**

```
Hi [First Name], [Persona First Name] this side.

[One sentence bridging to the previous channel — e.g. "We were chatting on Upwork
about your SaaS SEO project and I wanted to follow up."]

[Body from n8n — already trimmed and cleaned]

[CTA question — e.g. "Does Thursday or Friday work for a quick call?"]
```

No signature. No closing line after the CTA.

**HTML wrapper:**

```html
<div style="font-family: Verdana, Geneva, sans-serif; font-size: 14px; line-height: 1.5; color: #000000;">
  <p>Hi [First Name], [Persona First Name] this side.</p>
  <p>[Bridge sentence]</p>
  <p>[Body paragraphs]</p>
  <p>[CTA]</p>
</div>
```

---

## Step 7 — Create the Gmail Draft

Use `Gmail:create_draft` with:
- `to`: the discovered email address
- `subject`: the subject line from Step 6
- `body`: plain text version
- `htmlBody`: the HTML version from Step 6

Send from `marketing@incrementors.com` (confirm the Gmail connector is authenticated to this account before proceeding).

---

## Step 8 — Log and Report

Log to the Conversation Manager:
- Entry type: `Followup Message I sent`
- Include: email address used, email source, subject line, full body text

Report to the user:
- Draft created in Gmail (not "sent")
- Lead name, Lead ID
- Email address and source
- Sender persona
- Subject line
- Full body text for reference
- Prompt to open Gmail and click Send

---

## Tone Reference

**Good — specific, human, direct:**

> Hi Gabriel, Kuldeep this side.
>
> We were chatting on Upwork about El Establo, and since things got quiet there I wanted to drop a quick note.
>
> I checked the site again — the spam issue has spread to 5 more pages since we last spoke. That is actively blocking any AI search recommendations for the property.
>
> I also have a competitive visual ready showing where El Establo currently ranks vs the top 3 hotels ChatGPT recommends for Monteverde.
>
> Would Wednesday 3 PM or Thursday 10 AM work for a quick call?

**Bad — generic, template-sounding:**

> Hi Gabriel,
>
> I hope this email finds you well. I wanted to reach out to follow up on our previous conversation regarding your innovative project. In today's digital landscape, AI visibility has become mission-critical...

---

## Edge Cases

**Secondary Source is empty** — ask the user which persona to send from before doing anything else.

**Apollo name-only lookup returns an empty stub** — expected. Always run the two-stage pattern: Search first, then Enrich with the returned ID.

**No email found after all 5 steps** — report this clearly. Do not guess. Ask the user if they have a contact email.

**n8n workflow times out** — retry once with the identical payload. If it fails twice, write the body manually following the tone reference above, flag this in the summary.

**Gmail is authenticated to the wrong account** — stop immediately. Do not create the draft from the wrong account. Ask the user to reconnect Gmail to the correct account.

**Email bounces** — if the user reports a bounce, re-run Step 3 starting from 3c and avoid the pattern that failed.
