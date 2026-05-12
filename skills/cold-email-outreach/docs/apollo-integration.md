# Apollo.io Integration Reference

How to reliably find email addresses using Apollo.io from Claude Cowork.

---

## The Core Problem

A single `apollo_people_match` call with just a name returns an empty stub most of the
time — no email, no LinkedIn, no organisation. This is a known limitation.

The reliable pattern is always two stages: Search first, then Enrich with the ID.

---

## Two-Stage Lookup Pattern

### Stage 1: Search for the Person

Use `apollo_mixed_people_api_search`:

```json
{
  "q_keywords": "Gabriel Taylor Roldan marketing",
  "person_locations": ["Costa Rica"],
  "per_page": 10
}
```

Tips:
- Add a distinguishing keyword beyond just the name (their industry, role, or a company name from the conversation)
- Include their city or country from the lead record to reduce false positives
- For very common names (e.g. "John Smith"), add a company name or industry term — without it you get hundreds of irrelevant results

This returns candidates with Apollo person IDs and high-level info (title, company, LinkedIn URL) but no email details yet.

### Stage 2: Enrich with the Person ID

Use `apollo_people_match` with the ID from Stage 1:

```json
{
  "id": "6423abc19f...",
  "reveal_personal_emails": true
}
```

Or if you found their LinkedIn URL from web search, use that instead of the ID:

```json
{
  "linkedin_url": "https://linkedin.com/in/gabriel-taylor-roldan",
  "reveal_personal_emails": true
}
```

---

## Reading the Apollo Response

After enrichment, check these fields in order:

| Field | Meaning |
|---|---|
| `email` | Primary work email — use this if populated |
| `email_status` | `"verified"` = high confidence; `"likely to engage"` = use it; `"unavailable"` = no email found |
| `personal_emails` | Personal email array — use if work email is empty |

**If `email_status` is `"unavailable"` and `personal_emails` is empty:** Apollo has no email for this person. Move to pattern-guessing on the employer domain.

---

## Handling Ambiguous Search Results

When Stage 1 returns multiple candidates with the same name, compare each one against what you know from the lead's conversation history:

- Job title (matches their role?)
- Company name (matches what they mentioned?)
- Location (matches the country on the lead record?)
- LinkedIn URL (if you found one via web search, does it match?)

If you can't confidently identify the right person, move on to other email discovery methods. A wrong email is worse than no email.

---

## Logging the Email Source

Always record where the email came from in the Conversation Manager entry. Use these labels:

- `Apollo verified` — `email_status` was `"verified"`
- `Apollo likely` — `email_status` was `"likely to engage"`
- `Apollo personal` — came from the `personal_emails` array
- `Web search` — found via direct web search
- `Conversation history` — the client mentioned it in the chat
- `Pattern-guess on [domain.com]` — constructed from name pattern on the employer domain

---

## Domain Pattern-Guessing (Last Resort Only)

Only use this if all other methods fail AND you can verify the employer domain is a real website.

Common patterns to try, in order:
1. `firstname@company.com`
2. `firstname.lastname@company.com`
3. `f.lastname@company.com`
4. `flastname@company.com`

Do not pattern-guess on a domain that doesn't exist as a real website. If the company name in the lead record doesn't have a verifiable domain, skip this step.

---

## When Apollo Has No Email

This happens for roughly 60% of leads, especially individual freelancers, consultants, or people at small companies. Do not treat it as an error — just move to pattern-guessing.

If pattern-guessing also fails, report clearly: "No email found for [Name] after checking conversation history, Lead Database, web search, Apollo, and domain patterns. Do you have a contact email?"
