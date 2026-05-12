# Example: Upwork Lead Capture Walkthrough

This is an annotated example showing how the lead capture skill processes a real Upwork message.

---

## Input

User uploads a screenshot of an Upwork message. The screenshot shows:

- A message starting with "Hi Ankur, I'm looking for help with SEO for my SaaS product..."
- Upwork sidebar showing: 5 jobs posted, 80% hire rate, $12k total spent
- Client name: David Halberstam
- No email or phone visible

---

## Step 0 Output — Extracted Data

```
Client name:    David Halberstam
Source:         Upwork
Profile:        Ankur Gupta  (inferred from "Hi Ankur" + sidebar)
Country:        USA  (not stated — default applied)
Email:          (not visible)
Phone:          (not visible)
Company:        (not visible)
Website:        (not visible)

Client message (verbatim):
"Hi Ankur, I'm looking for help with SEO for my SaaS product. We're a B2B tool for
HR teams and we've been struggling to rank for any of our target keywords. I've tried
a few agencies before but the reporting was always vague and we never really understood
what was being done. Looking for someone transparent and results-focused."

Upwork signals:
Jobs posted: 5 | Hire rate: 80% | Total spent: $12k | Hires: 4 | Avg rate: not visible
```

---

## Step 1 Output — BDE Assignment

Profile "Ankur Gupta" maps to: **Amrendra** (amrendra345@iwantonlinemarketing.com)

No question asked — assignment is automatic.

---

## Step 2 Output — CRM Creation

Payload sent to n8n (ClientJoy creation workflow):

```json
{
  "Client Name": "David Halberstam",
  "Source": "Upwork",
  "Asignee": "amrendra345@iwantonlinemarketing.com",
  "Email": "",
  "Phone Number": "",
  "Company": "",
  "Website": "",
  "Country": "USA"
}
```

Note: `"Asignee"` is intentionally misspelled — the n8n workflow requires this exact key.

Result: Lead created. Lead ID: **1263847**

---

## Step 3 Output — Conversation Manager Entries

**Entry 1 — Client Background Info**

```
Lead created in CRM.
Name: David Halberstam
Email: (not provided)
Phone: (not provided)
Company: (not provided)
Website: (not provided)
Country: USA
Source: Upwork
Assigned to: Amrendra (amrendra345@iwantonlinemarketing.com)
Incrementors profile: Ankur Gupta

Upwork profile signals: 5 jobs posted, 80% hire rate, $12k total spent, 4 hires.
```

**Entry 2 — Client Message**

```
Hi Ankur, I'm looking for help with SEO for my SaaS product. We're a B2B tool for
HR teams and we've been struggling to rank for any of our target keywords. I've tried
a few agencies before but the reporting was always vague and we never really understood
what was being done. Looking for someone transparent and results-focused.
```

---

## Step 4 Output — Generated Reply

The Supervisor_Sales workflow returned two versions:

**Full Message:**

```
Hi David,

Thanks for reaching out. SaaS SEO for B2B tools is something we do a lot of work in,
and the frustration you're describing with vague reporting is something we hear often.

We approach it differently - every week you get a plain-English breakdown of exactly
what was done, what moved, and what's next. No fluff.

For HR-tech specifically, we've seen good traction with a mix of comparison-page SEO
(targeting buyers evaluating tools like yours) and topical authority building around
the core HR workflows your tool addresses.

Would a 20-minute call this week work to walk through where things currently stand
with your keywords and put together a quick action plan?

Ankur
```

**Smaller Version:**

```
Hi David,

Good timing - B2B SaaS SEO, especially for HR tools, is a strong area for us.

The vague reporting issue is exactly what we fixed for several past clients. You'd
get a plain-English weekly update showing exactly what moved and why.

Happy to do a quick 20-minute call to look at your current keyword situation.
Does Thursday or Friday work?

Ankur
```

---

## Step 5 — User Decision

Claude presents both versions and asks which one to send.

User responds: "Send the shorter one, looks good."

---

## Step 6 Output — Logged Reply

```
Entry type: Our Reply
Response type: AI
Details: [shorter version text as confirmed by user]
```

---

## Total Time

From screenshot upload to logged reply: approximately 90 seconds.

Manual equivalent: 12-18 minutes (copy data to CRM, write reply, log everything).
