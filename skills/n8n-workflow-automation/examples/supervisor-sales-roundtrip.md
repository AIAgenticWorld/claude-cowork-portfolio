# Example: n8n Workflow Round-Trip Walkthrough

A complete annotated example of Claude triggering a workflow, polling for the result,
and extracting the output.

---

## Scenario

Claude needs to generate a sales reply for a new Upwork lead. The Supervisor_Sales
workflow will handle the AI generation. Claude has the Lead ID (1263847) and the
client's message.

---

## Step 1: Trigger the Workflow

**Tool called:** `n8n MCP:execute_workflow`

**Payload:**

```json
{
  "workflowId": "DUKNqcvyjpLEV9b5",
  "executionMode": "production",
  "inputs": {
    "type": "chat",
    "chatInput": "{\"action\": \"Generate Message Reply\", \"lead_id\": \"1263847\", \"client_message\": \"Hi Ankur, I'm looking for help with SEO for my SaaS product. We're a B2B tool for HR teams and we've been struggling to rank for any of our target keywords. I've tried a few agencies before but the reporting was always vague.\"}"
  }
}
```

**Response from n8n:**

```json
{
  "executionId": "exe_7f3a91bc",
  "status": "running"
}
```

---

## Step 2: Poll for the Result

**Tool called:** `n8n MCP:get_execution`

**Payload:**

```json
{
  "executionId": "exe_7f3a91bc",
  "workflowId": "DUKNqcvyjpLEV9b5"
}
```

**Response status:** `"waiting"` — this is expected for AI Agent workflows. The agent
finished and the output is in the execution data.

---

## Step 3: Extract the Output

The full response structure (abbreviated):

```json
{
  "data": {
    "resultData": {
      "runData": {
        "Chat Trigger": [...],
        "AI Agent": [
          {
            "data": {
              "main": [
                [
                  {
                    "json": {
                      "output": "Full Message:\n\nHi David,\n\nThanks for reaching out. B2B SaaS SEO for HR tools is something we do a lot of work in...\n\n---\n\nSmaller Version:\n\nHi David,\n\nGood timing - B2B SaaS SEO for HR tools is a strong area for us..."
                    }
                  }
                ]
              ]
            }
          }
        ]
      }
    }
  }
}
```

**Extraction path used:**

```
data.resultData.runData["AI Agent"][0].data.main[0][0].json.output
```

**Extracted output:**

```
Full Message:

Hi David,

Thanks for reaching out. B2B SaaS SEO for HR tools is something we do a lot of work in,
and the frustration you're describing with vague reporting is something we hear often.

We approach it differently - every week you get a plain-English breakdown of exactly what
was done, what moved, and what's next. No fluff.

For HR-tech specifically, we've seen good traction with a mix of comparison-page SEO
(targeting buyers evaluating tools like yours) and topical authority building around the
core HR workflows your tool addresses.

Would a 20-minute call this week work to walk through where things currently stand?

---

Smaller Version:

Hi David,

Good timing - B2B SaaS SEO for HR tools is a strong area for us.

The vague reporting issue is exactly what we fixed for several past clients. You'd get a
plain-English weekly update showing exactly what moved and why.

Happy to do a quick 20-minute call to look at your current keyword situation. Does
Thursday or Friday work?
```

---

## What Would Have Gone Wrong

**If `chatInput` was passed as an object (not a string):**

```json
"chatInput": {
  "action": "Generate Message Reply",
  "lead_id": "1263847"
}
```

n8n's Chat Trigger would not parse this correctly. The AI Agent might receive no input
context, producing a generic or empty response. The fix: always stringify.

**If `executionMode` was `"test"` instead of `"production"`:**

n8n would use pinned test data from a previous run instead of processing the actual input.
The output would not reflect the real client message.

**If the node name in the extraction path was wrong:**

If the AI Agent node in the n8n editor was actually named `"Sales Agent"` (not `"AI Agent"`),
the path `runData["AI Agent"]` would return `undefined` and the extraction would fail.
Always verify node names from a successful execution before hardcoding the path.
