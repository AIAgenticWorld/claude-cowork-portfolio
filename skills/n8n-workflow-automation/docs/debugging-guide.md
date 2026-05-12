# n8n Debugging Guide

Common failure patterns and how to fix them when Claude Cowork calls n8n workflows.

---

## Failure: "Cannot read properties of undefined (reading '0')"

**What it means:** The workflow ran but the node Claude is trying to read from either
did not execute, returned nothing, or the data path is wrong.

**Checklist:**
- [ ] Is the trigger type correct? (`"form"` vs `"chat"` vs `"webhook"`)
- [ ] For chat triggers: is `chatInput` a JSON string (not a nested object)?
- [ ] Are `formData` field names exactly matching the Form node labels in n8n?
- [ ] Did the workflow actually reach the node you're reading from? Check other nodes in `runData` to see which ones ran.

**Fix:** Inspect the full `data.resultData.runData` object to see which nodes ran and what they returned. The node that didn't produce data is where the problem is.

---

## Failure: Workflow Returns `status: "waiting"` With No Output

**What it means:** This is NOT an error. For AI Agent workflows, `"waiting"` means the
agent has finished and is waiting for further chat input. The output is already in the
execution data.

**Fix:** Call `get_execution` immediately after `execute_workflow`. The output will be
in the AI Agent node's result data.

---

## Failure: Workflow Times Out Before Returning

**What it means:** Long-running workflows (AI agents with sub-workflows, or workflows
doing multiple API calls) can take 30-60 seconds. Claude's tool call may time out
before the workflow finishes.

**Fix:**
1. Wait 10-15 seconds and call `get_execution` with the execution ID
2. If the execution is still running (status: `"running"`), wait another 15 seconds and try again
3. If it never completes, retry the `execute_workflow` call once with the identical payload
4. If it fails twice, handle the step manually and flag it in the summary

---

## Failure: n8n Returns an Error Node Result

**What it means:** The workflow hit an error node, usually because:
- An upstream API call failed (ClientJoy, Gmail, Apollo, etc.)
- A required field was missing or the wrong type
- The AI Agent ran out of context or hit a rate limit

**Fix:** Check the `error` field in the result data. The error message from n8n usually
tells you which external service failed and why.

---

## Failure: Form Workflow Creates the Lead But Returns Wrong ID

**What it means:** The extraction path for the Lead ID is wrong, or the HTTP Request
node name in n8n has changed.

**Debug step:** Log the full `data.resultData.runData` and look at the keys at the top
level. The HTTP Request node may have a different name (e.g. `"HTTP Request1"` or
`"Create Lead in ClientJoy"`).

**Fix:** Update the extraction path to use the actual node name as it appears in the
`runData` keys.

---

## Failure: Supervisor_Sales Returns a Body But It Includes a Sign-Off

**What it means:** The AI Agent in n8n added a closing line (e.g. "Warm regards, Kuldeep").
This is a known pattern — the AI sometimes adds a sign-off even though the skill instructions
say not to.

**Fix:** After extracting the output, scan for and strip any closing line that starts with:
"Warm regards", "Best", "Thanks", "Cheers", or a standalone name on the last line.
This is handled in the cold-email-outreach skill's Rule 3.

---

## Failure: ChatInput Was Sent as an Object, Not a String

**What it means:** The `chatInput` field was passed as a nested JSON object instead of a
JSON-stringified string. n8n's Chat Trigger node expects a string.

**Wrong:**
```json
"chatInput": {
  "action": "Generate Message Reply",
  "lead_id": "123"
}
```

**Correct:**
```json
"chatInput": "{\"action\": \"Generate Message Reply\", \"lead_id\": \"123\"}"
```

**Fix:** Always stringify the inner object before passing it as `chatInput`.

---

## Failure: Conversation Manager Returns "Unauthorized"

**What it means:** The `responseType` field value caused an authentication issue with
the Conversation Manager MCP.

**Fix:** Retry the same `add_lead_entry` call but omit the `responseType` parameter
entirely. This has consistently worked where the `responseType: "AI"` variant failed.

---

## General Debugging Approach

When a workflow call fails or returns unexpected data:

1. Log the full `data.resultData.runData` object
2. Check which node names are present as keys
3. Find the last node that ran successfully
4. Read the data that node returned
5. Identify where the chain broke

This is always faster than guessing at the payload format.
