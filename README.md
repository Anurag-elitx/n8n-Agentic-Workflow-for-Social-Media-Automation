# n8n Agentic Workflow for Social Media Automation

## 1. Project Overview
This project is a complete multi-agent automation system built with n8n. It autonomously researches trending topics, generates social media content using AI (OpenAI GPT-4o), runs it through a Human-in-the-Loop (HITL) approval process via Gmail and Google Sheets, and finally publishes it to platforms like LinkedIn and Twitter. It includes full error monitoring, fallback handling, and retry mechanisms.

## 2. Architecture Diagram

```text
+---------------------+     +--------------------------+     +--------------------------+
|                     |     |                          |     |                          |
| 1. Topic Research   +----->  Google Sheets (Topics)  +-----> 2. Content Generation    |
|    Agent (Mon 9 AM) |     |                          |     |    Agent                 |
|                     |     +--------------------------+     |                          |
+---------+-----------+                                      +------------+-------------+
          |                                                               |
          |                                                               v
+---------v-----------+     +--------------------------+     +------------+-------------+
|                     |     |                          |     |                          |
| 5. Monitoring &     |     | Google Sheets (Content)  <-----+ 3. HITL Approval         |
|    Alerting Agent   |     |                          |     |    Workflow (Email)      |
|    (Runs Hourly)    |     +--------------------------+     |                          |
+---------------------+             ^           |            +------------+-------------+
                                    |           |                         |
                                    |           v                         v
                            +-------+-----------v------+     +------------+-------------+
                            |                          |     |                          |
                            | 4. Publishing Agent      <-----+ Webhook (On Approval)    |
                            |    (LinkedIn & Twitter)  |     |                          |
                            |                          |     +--------------------------+
                            +--------------------------+
```

## 3. Workflow Details & Node Configuration

Here is the exact node-by-node configuration to build each workflow in n8n.

### WORKFLOW 1: TOPIC RESEARCH AGENT
**Trigger:** Schedule (runs every Monday 9 AM)

1. **Schedule Trigger Node**
   - **Type:** Schedule Trigger
   - **Settings:** Rule: Every Week, Day: Monday, Hour: 9, Minute: 0
   - **Connect to:** HTTP Request Node

2. **HTTP Request Node (Fetch Reddit)**
   - **Type:** HTTP Request
   - **Settings:** Method: GET, URL: `https://www.reddit.com/r/technology/top.json?limit=10`
   - **Connect to:** OpenAI Node

3. **OpenAI Node (Analyze Topics)**
   - **Type:** OpenAI
   - **Settings:** Resource: Chat, Operation: Create
   - **Fields:** 
     - Model: `gpt-4o`
     - Messages: 
       - Role: User
       - Content: `Analyze these Reddit posts and extract 5 trending topic ideas suitable for LinkedIn and Twitter content. Return as JSON array: [{"topic": "...", "angle": "...", "target_audience": "...", "estimated_engagement": "..."}]. Data: {{$json.data.children}}`
   - **Connect to:** Code Node

4. **Code Node (Parse JSON)**
   - **Type:** Code
   - **Settings:** Mode: Run Once for All Items
   - **Code:** 
     ```javascript
     const content = JSON.parse($input.first().json.message.content);
     return content.map(item => ({ json: item }));
     ```
   - **Connect to:** Google Sheets Node

5. **Google Sheets Node (Append to Queue)**
   - **Type:** Google Sheets
   - **Settings:** Resource: Row, Operation: Append
   - **Fields:** Document: [Select Your Sheet], Sheet: `Topic Queue`, Mapping Mode: Auto-Map (Map Date, Topic, Angle, Audience from previous node). Hardcode `Status` as `Pending`.
   - **Connect to:** Gmail Node

6. **Gmail Node (Notify Review)**
   - **Type:** Gmail
   - **Settings:** Resource: Message, Operation: Send
   - **Fields:** To: `your-email@example.com`, Subject: `5 new topics added to queue for review`, Message: `Please check the Google Sheet to review the new topics.`
   - **Connect to:** (End)

*Note: Add an Error Trigger node connected to a Gmail node to send error logs if any step fails.*

---

### WORKFLOW 2: CONTENT GENERATION AGENT
**Trigger:** Webhook (called when topic approved in Google Sheets)

1. **Webhook Trigger Node**
   - **Type:** Webhook
   - **Settings:** Method: POST, Path: `generate-content`, Respond: Immediately
   - **Connect to:** Google Sheets Node

2. **Google Sheets Node (Fetch Topic)**
   - **Type:** Google Sheets
   - **Settings:** Resource: Row, Operation: Read
   - **Fields:** Document: [Select Your Sheet], Sheet: `Topic Queue`, Range/Row: lookup by `topic_id` from Webhook payload.
   - **Connect to:** OpenAI Node

3. **OpenAI Node (Generate Content)**
   - **Type:** OpenAI
   - **Settings:** Resource: Chat, Operation: Create
   - **Fields:** Model: `gpt-4o`
   - **Prompt:** `Write a {{$json.platform}} post about {{$json.topic}}. Angle: {{$json.angle}}. Target: {{$json.audience}}. Requirements: 150-200 words, include 3 hashtags, professional tone. Return JSON: {"post_text": "...", "hashtags": "...", "character_count": 0}`
   - **Connect to:** Code Node

4. **Code Node (Format Output)**
   - **Type:** Code
   - **Settings:** Parse the OpenAI JSON response and structure it for Google Sheets.
   - **Connect to:** Google Sheets Node

5. **Google Sheets Node (Write to Content Queue)**
   - **Type:** Google Sheets
   - **Settings:** Resource: Row, Operation: Append
   - **Fields:** Sheet: `Content Queue`. Map topic_id, platform, generated content. Set `Status` to `Awaiting Approval`.
   - **Connect to:** (End)

---

### WORKFLOW 3: HUMAN-IN-THE-LOOP APPROVAL
**Trigger:** Google Sheets Trigger

1. **Google Sheets Trigger Node**
   - **Type:** Google Sheets Trigger
   - **Settings:** Trigger On: Row Added, Sheet: `Content Queue`
   - **Connect to:** IF Node (Check if Status == Awaiting Approval)

2. **IF Node**
   - **Type:** IF
   - **Settings:** Condition: String `{{$json.Status}}` Equal `Awaiting Approval`
   - **True Connect to:** Code Node (Format Email)

3. **Gmail Node (Send Approval Request)**
   - **Type:** Gmail
   - **Settings:** Resource: Message, Operation: Send
   - **Fields:** To: `approver@example.com`, Subject: `Content Ready for Review: {{$json.Topic}}`
   - **Message (HTML):** 
     ```html
     <p>Please review:</p>
     <p>{{$json.Content}}</p>
     <a href="http://YOUR_N8N_URL/webhook/approve?id={{$json.ID}}">APPROVE</a> | 
     <a href="http://YOUR_N8N_URL/webhook/reject?id={{$json.ID}}">REJECT</a>
     ```

*(Sub-Workflow for Webhooks)*
1. **Webhook Trigger (Approve/Reject)**
   - **Type:** Webhook (GET, Path: `approve`, another for `reject`)
   - **Connect to:** Google Sheets Node (Update Row status to `Approved` or `Rejected` based on Webhook path).
   - **If Approved Connect to:** HTTP Request Node (POST to Publishing Agent Webhook).

---

### WORKFLOW 4: PUBLISHING AGENT
**Trigger:** Webhook (Called after approval)

1. **Webhook Trigger Node**
   - **Type:** Webhook
   - **Settings:** Method: POST, Path: `publish-content`
   - **Connect to:** Google Sheets Node (Fetch Content details by ID)

2. **Switch Node (Route Platform)**
   - **Type:** Switch
   - **Settings:** Routing key: `{{$json.Platform}}`
   - **Rules:** 1. `LinkedIn`, 2. `Twitter`
   - **LinkedIn Route Connect to:** HTTP Request (LinkedIn API)
   - **Twitter Route Connect to:** HTTP Request (Twitter API)

3. **HTTP Request Node (LinkedIn API)**
   - **Type:** HTTP Request
   - **Settings:** Method: POST, URL: `https://api.linkedin.com/v2/ugcPosts`
   - **Authentication:** OAuth2 (LinkedIn)
   - **Retry Settings (Under Node Settings):** On Error: Retry, Max Tries: 3, Wait Between Tries: 5000ms.
   - **Connect to:** Google Sheets Node (Update Status to Published)

4. **HTTP Request Node (Twitter API)**
   - **Type:** HTTP Request
   - **Settings:** Method: POST, URL: `https://api.twitter.com/2/tweets`
   - **Connect to:** Wait Node (30s) -> HTTP Request (Tweet 2) -> Wait Node (30s) -> HTTP Request (Tweet 3)

---

### WORKFLOW 5: MONITORING & ALERTING
**Trigger:** Schedule (Runs hourly)

1. **Schedule Trigger Node**
   - **Type:** Schedule Trigger
   - **Settings:** Rule: Every Hour
   - **Connect to:** Google Sheets Node

2. **Google Sheets Node (Fetch Failed/Stuck)**
   - **Type:** Google Sheets
   - **Settings:** Resource: Row, Operation: Read
   - **Fields:** Sheet: `Content Queue`
   - **Connect to:** IF Node

3. **IF Node (Check Issues)**
   - **Type:** IF
   - **Settings:** Condition: `Status == Publish Failed` OR `Status == Awaiting Approval && Generated_At < (Now - 48h)`
   - **True Connect to:** Gmail Node

4. **Gmail Node (Alert)**
   - **Type:** Gmail
   - **Settings:** Send alert email with list of stuck items.
   - **Connect to:** Postgres Node (Log run)

## 4. Setup Instructions
1. Clone this repository.
2. The `.env` and `docker-compose.yml` files are already configured. Modify `.env` with your actual OAuth keys if running in a real environment (mock keys are present for demonstration).
3. Run `docker-compose up -d` to start n8n and PostgreSQL.
4. Access n8n at `http://localhost:5678`.
5. Create the Google Sheet based on `google-sheets-template/content_calendar_template.md`.
6. Import the workflow JSON files from the `workflows/` directory into your n8n instance.
7. Configure Google Sheets, Gmail, and OpenAI credentials inside n8n UI.

## 5. How to Use
1. The **Topic Research** workflow runs every Monday at 9 AM, automatically pulling Reddit trends.
2. Check your Google Sheet (`Topic Queue`).
3. Set a topic status to `Approved` to trigger the **Content Generation**.
4. You will receive an email with the generated post. Click `APPROVE` directly from the email.
5. The post will be automatically published to LinkedIn or Twitter.
6. The Google Sheet will update with the Post URL and `Published` status.

## 6. Error Handling
- **Retry Logic:** The Publishing Agent utilizes n8n's built-in node retry settings. If the LinkedIn or Twitter APIs rate-limit the request, it will wait 5 seconds and retry up to 3 times (Exponential Backoff concept).
- **Fallback:** If publishing fails after all retries, an Error Trigger catches the failure, updates the Google Sheet to `Publish Failed`, and writes the stack trace to the PostgreSQL DB for auditing.
- **Monitoring:** The hourly cron checks for items stuck in `Awaiting Approval` for over 24 hours and alerts the admin via email.

## 7. Screenshots
*(Add screenshots of your n8n workflow canvas here once imported)*
- `![Topic Research Workflow](docs/images/research-workflow.png)`
- `![HITL Approval Workflow](docs/images/hitl-workflow.png)`

## 8. Key Features
- **Multi-agent architecture:** 3 specialized agents handle distinct parts of the lifecycle.
- **Human-in-the-loop approval:** Prevents hallucinations or inappropriate content from being published.
- **Automatic retry with backoff:** Resilient against external API rate limits.
- **Full audit trail:** Every state change is tracked in Google Sheets and Postgres.
- **Hourly monitoring:** Ensures no content falls through the cracks.