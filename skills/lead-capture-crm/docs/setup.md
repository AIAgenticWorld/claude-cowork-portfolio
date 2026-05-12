# Setup — Lead Capture & CRM Automation

## Required MCP Tools

| Tool | Purpose |
|---|---|
| `n8n MCP` | Triggers CRM creation workflow and Supervisor_Sales AI workflow |
| `Leads Conversation Manager` | Logs all conversation entries per lead |
| `Leads Database` | Lead lookup for returning leads |

## n8n Workflows Required

| Workflow | Trigger Type | Purpose |
|---|---|---|
| Add Leads To ClientJoy | `form` | Creates the lead record in ClientJoy CRM |
| Supervisor_Sales | `chat` | Generates personalised AI replies |

## CRM Field Notes

The ClientJoy creation workflow expects these exact field names (case-sensitive):

```
Client Name
Source           → must be one of: Upwork, LinkedIn, Website, Referral
Asignee          → intentionally misspelled — do not correct
Email
Phone Number
Company
Website
Country
```

All 8 fields must be sent even if some are empty strings.

## Payload Shapes

**CRM Workflow (form trigger):**

```json
{
  "workflowId": "<your-workflow-id>",
  "executionMode": "production",
  "inputs": {
    "type": "form",
    "formData": {
      "Client Name": "...",
      "Source": "Upwork",
      "Asignee": "bde@youragency.com",
      ...
    }
  }
}
```

**Supervisor_Sales (chat trigger):**

```json
{
  "workflowId": "<your-workflow-id>",
  "executionMode": "production",
  "inputs": {
    "type": "chat",
    "chatInput": "{\"action\": \"Generate Message Reply\", \"lead_id\": \"123\", \"client_message\": \"...\"}"
  }
}
```

Note: `chatInput` must be a JSON string (not a nested object).

## Extracting the Lead ID After CRM Creation

The lead ID is nested in the n8n execution result. After calling `get_execution`, find it at:

```
data.resultData.runData["HTTP Request"][0].data.main[0][0].json.data.id
```

## Extracting the AI Reply After Supervisor_Sales

Find the generated reply at:

```
data.resultData.runData["AI Agent"][0].data.main[0][0].json.output
```

The output contains both a Full Message and a Smaller Version as plain text.

## BDE Routing Table

Adapt this to your team structure. Map each Upwork profile to a BDE email address. LinkedIn and Website leads use fixed routing rules (see SKILL.md Step 1).

## Conversation Manager Entry Types

| Type | When Used |
|---|---|
| `Client Background Info` | First entry after CRM creation — profile signals, source, BDE |
| `Client Message` | Verbatim text of the client's inbound message |
| `Our Reply` | The exact reply that was sent (logged after user confirms) |
