# n8n Vikunja AI Workflows

Two complete n8n workflow JSON files for integrating AI-powered task management with Vikunja.

## Files

1. **workflow-1-chat-ai-vikunja.json** - Chat → AI → Vikunja Task Creator
2. **workflow-2-daily-task-summary.json** - Daily Task Summary with AI

## Workflow 1: Chat → AI → Vikunja Task Creator

### Overview
This workflow allows users to create Vikunja tasks using natural language input in German. The AI agent parses the input, extracts task details, and creates the task via the Vikunja API.

### Nodes
1. **Chat Trigger** - Web-based chat interface for user input
2. **AI Agent** - Orchestrates the AI processing with Ollama
3. **Ollama Chat Model** - Local LLM (llama3.1:8b-instruct-q4_0)
4. **Structured Output Parser** - Ensures JSON output with required fields
5. **Create Vikunja Task** - HTTP Request to Vikunja API
6. **Respond to Chat** - Sends confirmation back to user

### How It Works
```
User Input: "Erinnere mich morgen an Einkaufen"
    ↓
Chat Trigger captures input
    ↓
AI Agent with Ollama processes and extracts:
  - task_title: "Einkaufen"
  - due_date: "2026-02-17T12:00:00Z"
  - project_id: 1
  - priority: 3
    ↓
HTTP Request creates task in Vikunja
    ↓
Respond to Chat: "Aufgabe erstellt: Einkaufen (Fällig: 2026-02-17T12:00:00Z)"
```

### Configuration Required

#### 1. Ollama Credentials
Before importing, you need to create Ollama API credentials in n8n:
- Go to **Settings → Credentials → Add Credential**
- Select **Ollama API**
- Set Base URL: `http://ollama.ollama.svc.cluster.local:11434`
- Save with name: "Ollama API"

The workflow references this credential by name.

#### 2. Vikunja API Token
The workflow uses a hardcoded Bearer token in the HTTP Request node:
```
Bearer tk_ba2a509cbe18e5368df76e9de325f55e12506358
```

**IMPORTANT**: Replace this token with your actual Vikunja API token before using in production.

#### 3. Chat Access
After activation, access the chat interface at:
```
https://your-n8n-instance.com/webhook/vikunja-task-chat-trigger
```

### Example Inputs
- "Erinnere mich morgen an Einkaufen"
- "Wichtig: Projekt-Präsentation in 3 Tagen vorbereiten"
- "Nächste Woche Arzttermin vereinbaren, hohe Priorität"

### AI System Prompt
The AI is instructed to:
- Extract task title, due date, project ID, and priority
- Convert relative dates ("morgen", "in 3 Tagen") to absolute ISO 8601 format
- Default to project_id=1 and priority=3 if not specified
- Always return valid JSON

---

## Workflow 2: Daily Task Summary

### Overview
Runs every day at 8:00 AM, fetches all open tasks from Vikunja, and generates an AI-powered summary in German.

### Nodes
1. **Schedule Trigger** - Cron: `0 8 * * *` (8:00 AM daily)
2. **Get Open Tasks** - HTTP Request to Vikunja API (filter: done=false)
3. **Prepare Task Data** - Formats task data for AI processing
4. **Basic LLM Chain** - Generates summary with Ollama
5. **Ollama Chat Model** - Local LLM (llama3.1:8b-instruct-q4_0)
6. **Format Summary Output** - Structures final output
7. **Send Summary (Optional)** - HTTP webhook for external delivery (disabled by default)

### How It Works
```
Every day at 8:00 AM
    ↓
Fetch all open tasks from Vikunja API
    ↓
Prepare task data (count tasks, format JSON)
    ↓
AI processes and creates summary:
  - Total open tasks
  - Tasks grouped by priority
  - Tasks due today
  - Tasks due in next 3 days
  - Overdue tasks
    ↓
Format output with timestamp
    ↓
[Optional] Send to webhook
```

### Configuration Required

#### 1. Ollama Credentials
Same as Workflow 1 - reuses the "Ollama API" credential.

#### 2. Vikunja API Token
Same token as Workflow 1 is used in the "Get Open Tasks" node.

#### 3. Schedule Configuration
Default: Daily at 8:00 AM
To change the schedule, modify the cron expression in the Schedule Trigger node:
- Every 6 hours: `0 */6 * * *`
- Weekdays only at 8 AM: `0 8 * * 1-5`
- Twice daily (8 AM and 6 PM): Add a second trigger rule

#### 4. Output Destination (Optional)
The "Send Summary (Optional)" node is **disabled by default**. Enable it to send summaries to:
- A webhook endpoint
- Slack/Discord via webhook
- Email service
- Any HTTP endpoint

Replace the URL in the node:
```json
"url": "http://your-webhook-endpoint.com/task-summary"
```

### Summary Format
The AI generates summaries in German with:
```markdown
## Aufgabenzusammenfassung - [Datum]

**Gesamtanzahl offene Aufgaben:** 15

### Nach Priorität:
- **Sehr hoch (5):** 2 Aufgaben
- **Hoch (4):** 5 Aufgaben
- **Mittel (3):** 6 Aufgaben
- **Niedrig (2):** 2 Aufgaben

### Heute fällig:
- Aufgabe 1
- Aufgabe 2

### Nächste 3 Tage:
- Aufgabe 3 (morgen)
- Aufgabe 4 (übermorgen)

### Überfällig:
- Aufgabe 5 (seit 2 Tagen)
```

---

## Import Instructions

### Step 1: Import Workflows
1. Open your n8n instance
2. Click **Workflows → Import from File** (or **Import from URL**)
3. Select `workflow-1-chat-ai-vikunja.json`
4. Repeat for `workflow-2-daily-task-summary.json`

### Step 2: Configure Credentials

#### Create Ollama API Credential
1. Go to **Settings → Credentials**
2. Click **Add Credential**
3. Search for "Ollama"
4. Configure:
   - **Name:** Ollama API
   - **Base URL:** `http://ollama.ollama.svc.cluster.local:11434`
5. Click **Save**

#### Update Vikunja API Token
1. Open each workflow
2. Find the HTTP Request nodes that call Vikunja API
3. Update the Authorization header with your actual token:
   ```
   Bearer YOUR_VIKUNJA_API_TOKEN_HERE
   ```

### Step 3: Verify Model Availability
Ensure the Ollama model `llama3.1:8b-instruct-q4_0` is available:
```bash
kubectl exec -n ollama deployment/ollama -- ollama list
```

If not available, pull it:
```bash
kubectl exec -n ollama deployment/ollama -- ollama pull llama3.1:8b-instruct-q4_0
```

### Step 4: Test Workflows

#### Test Workflow 1 (Chat)
1. Open the workflow
2. Click **Test workflow** button
3. Click the chat icon to open the chat interface
4. Type: "Erinnere mich morgen an Einkaufen"
5. Check that a task is created in Vikunja

#### Test Workflow 2 (Daily Summary)
1. Open the workflow
2. Click **Execute Workflow** button (manually trigger)
3. Check the execution log for the AI-generated summary
4. Verify the summary includes all open tasks

### Step 5: Activate Workflows
1. Toggle the **Active** switch on each workflow
2. Workflow 1: Chat interface becomes accessible
3. Workflow 2: Scheduled execution begins

---

## Troubleshooting

### Connection Issues

#### Ollama Not Reachable
- Verify Ollama is running: `kubectl get pods -n ollama`
- Check service DNS: `http://ollama.ollama.svc.cluster.local:11434`
- Test connection from n8n pod:
  ```bash
  kubectl exec -n n8n deployment/n8n -- curl http://ollama.ollama.svc.cluster.local:11434
  ```

#### Vikunja API Errors
- Verify Vikunja is running: `kubectl get pods -n vikunja`
- Check API token validity
- Test API manually:
  ```bash
  curl -H "Authorization: Bearer YOUR_TOKEN" \
    http://vikunja.vikunja.svc.cluster.local:3456/api/v1/tasks/all
  ```

### AI Agent Issues

#### Model Not Found
Error: `Model llama3.1:8b-instruct-q4_0 not found`
- Pull the model: `kubectl exec -n ollama deployment/ollama -- ollama pull llama3.1:8b-instruct-q4_0`

#### Invalid JSON Output
If AI returns invalid JSON:
- Check the Structured Output Parser configuration
- Verify the system prompt in AI Agent node
- Try with a different model or adjust temperature

#### Date Parsing Errors
If dates are incorrectly calculated:
- Update today's date in the system prompt
- Provide more explicit examples
- Use more specific relative date terms

### Workflow Execution Errors

#### Credential Errors
Error: `Credentials with name 'Ollama API' not found`
- Create the credential as described above
- Ensure the name matches exactly: "Ollama API"

#### HTTP Request Timeouts
- Increase timeout in HTTP Request node options
- Check network connectivity between services

---

## Architecture Notes

### n8n Workflow JSON Structure

Both workflows follow the standard n8n v2.7.5 JSON format:

```json
{
  "name": "Workflow Name",
  "nodes": [ /* Array of node objects */ ],
  "connections": { /* Connection mapping */ },
  "active": false,
  "settings": { "executionOrder": "v1" },
  "versionId": "unique-version-id",
  "meta": {},
  "id": "unique-workflow-id",
  "tags": []
}
```

### Node Types Used

| Type | Package | Purpose |
|------|---------|---------|
| `chatTrigger` | `@n8n/n8n-nodes-langchain` | Chat interface trigger |
| `agent` | `@n8n/n8n-nodes-langchain` | AI Agent orchestrator |
| `lmChatOllama` | `@n8n/n8n-nodes-langchain` | Ollama LLM integration |
| `outputParserStructured` | `@n8n/n8n-nodes-langchain` | JSON schema enforcement |
| `chainLlm` | `@n8n/n8n-nodes-langchain` | Basic LLM chain |
| `respondToChat` | `@n8n/n8n-nodes-langchain` | Chat response |
| `httpRequest` | `n8n-nodes-base` | HTTP API calls |
| `scheduleTrigger` | `n8n-nodes-base` | Cron scheduling |
| `set` | `n8n-nodes-base` | Data transformation |

### Connection Types

| Type | Description |
|------|-------------|
| `main` | Standard data flow between nodes |
| `ai_languageModel` | Connects LLM to agent/chain |
| `ai_outputParser` | Connects output parser to agent |

### Positioning

Nodes are positioned on the canvas using `[x, y]` coordinates:
- Horizontal spacing: ~220 pixels between nodes
- Vertical spacing: ~220 pixels for sub-nodes (LLM, parsers)
- Workflow flows left-to-right

---

## Customization Ideas

### Workflow 1 Enhancements
- Add memory node for conversation context
- Support multiple languages
- Add task templates (recurring tasks, subtasks)
- Integrate with calendar for due date suggestions
- Add task assignment to team members

### Workflow 2 Enhancements
- Send summary via email (SMTP node)
- Post to Slack/Discord
- Generate charts/graphs of task distribution
- Compare with previous summaries (trend analysis)
- Filter by specific projects or labels
- Add "focus tasks" recommendation based on priority and due date

---

## References

### n8n Documentation
- [Chat Trigger Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chattrigger/)
- [AI Agent Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)
- [Ollama Chat Model](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatollama/)
- [HTTP Request Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)
- [Schedule Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/)
- [Structured Output Parser](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.outputparserstructured/)

### Additional Resources
- [n8n Workflow Export/Import Guide](https://docs.n8n.io/workflows/export-import/)
- [LangChain in n8n](https://docs.n8n.io/advanced-ai/langchain/overview/)
- [Ollama Integration](https://docs.ollama.com/integrations/n8n)
- [Vikunja API Documentation](https://vikunja.io/docs/api-documentation/)

---

## License & Support

These workflows are provided as-is for use with n8n v2.7.5+, Ollama, and Vikunja.

Built on 2026-02-16 using official n8n workflow JSON structure and LangChain nodes.
