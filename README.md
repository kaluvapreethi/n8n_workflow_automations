# Resume Workflow (Google) : n8n Workflow

## Overview

This workflow automates resume tailoring for job applications. It reads your master resume from Google Docs, analyzes it against a job description using a local LLM, rewrites relevant bullet points to improve alignment, saves the tailored version as a new Google Doc, and logs everything to a Postgres database.

---

## Workflow Diagram

```
Manual Trigger
    └── Read_Resume (Google Docs)
            └── Job_Description (Set node)
                    └── LLM_Analysis (Ollama / qwen2.5:7b)
                            └── Parse_LLM_Output (Code)
                                    └── Replicate_Resume (Google Drive)
                                            └── Update_Replicated_Resume (Google Docs API)
                                                    └── New_Match_Percentage (Ollama / qwen2.5:7b)
                                                            └── Insert_Record (Postgres)
                                                                    └── [Create a document — disabled]
                                                                            └── [Update a document — disabled]
```

---

## Nodes

| Node | Type | Purpose |
|---|---|---|
| **When clicking 'Execute workflow'** | Manual Trigger | Starts the workflow on demand |
| **Read_Resume** | Google Docs | Reads the master resume (doc ID hardcoded) |
| **Job_Description** | Set | Stores the target job description as a variable |
| **LLM_Analysis** | HTTP Request → Ollama | Sends resume + JD to local LLM; returns JSON with match %, missing skills, strong matches, and rewritten bullets |
| **Parse_LLM_Output** | Code (JS) | Parses LLM JSON, builds the updated resume text, and prepares Google Docs `batchUpdate` requests |
| **Replicate_Resume** | Google Drive | Copies the master resume and names the copy after the target company |
| **Update_Replicated_Resume** | HTTP Request → Google Docs API | Applies find-and-replace edits to the copied doc using the LLM's rewrites |
| **New_Match_Percentage** | HTTP Request → Ollama | Re-evaluates the updated resume against the JD and returns a new match score |
| **Insert_Record** | Postgres | Saves all results (company, JD, match scores, skills, rewrites, generated resume) to the `job_applications` table |
| **Execute a SQL query** | Postgres | Creates the `job_applications` table if it doesn't exist (run once for setup) |
| **Create a document** *(disabled)* | Google Docs | Alternative: creates a fresh Google Doc for the output |
| **Update a document** *(disabled)* | Google Docs | Alternative: inserts updated resume text into the created doc |

---

## Prerequisites

- **n8n** self-hosted instance with Docker
- **Ollama** running locally with `qwen2.5:7b` pulled (`ollama pull qwen2.5:7b`)
- **Google Docs OAuth2** credential configured in n8n
- **Google Drive OAuth2** credential configured in n8n
- **Postgres** database accessible from n8n with credentials configured

---

## Setup Instructions

### 1. Database Setup
Run the **Execute a SQL query** node once manually to create the `job_applications` table. It is wired separately and won't auto-run with the main flow.

### 2. Configure Your Resume
In the **Read_Resume** node, replace the hardcoded document ID (`13tLnyZpWXAvc1NG5vqh45yJnbl-f22bdZ3DRplkj9dI`) with your own Google Doc's document ID. The ID is in the URL:
```
https://docs.google.com/document/d/YOUR_DOCUMENT_ID/edit
```

### 3. Set the Job Description
Paste the target job description into the **Job_Description** Set node's `Job Description` field.

### 4. Set the Output Folder (optional)
In **Create a document** (currently disabled), update the `folderId` to your preferred Google Drive folder.

### 5. Run the Workflow
Click **Execute workflow** in n8n to trigger the full pipeline.

---

## LLM Prompt Logic

**LLM_Analysis** instructs the model to:
- Extract the hiring company name
- Calculate a semantic match % between resume and JD
- Identify strong matching skills and critical missing skills
- Rewrite only the experience bullets that need improvement
- Target ≥ 90% match after rewrites
- Never invent skills or experiences — factual accuracy is enforced

**New_Match_Percentage** is a second LLM call that independently re-scores the updated resume against the JD to validate the improvement.

---

## Output

- A tailored Google Doc copy of your resume, named `Preethi_Kaluva_<CompanyName>`
- A Postgres record containing:
  - Company name
  - Original job description
  - Before/after match percentages
  - Missing and matching skills
  - All rewritten bullet points
  - Full generated resume text

---

## Notes & Known Limitations

- The `Parse_LLM_Output` code references both `r.original_bullet` / `r.rewritten_bullet` and `r.original` / `r.rewritten` — ensure the LLM consistently returns `original` and `rewritten` keys as specified in the prompt schema.
- `qwen2.5:7b` is a mid-sized local model; for better rewrite quality, consider upgrading to a larger model like `qwen2.5:14b` or routing to a cloud API.
- The **Create a document** and **Update a document** nodes are disabled — enable them if you prefer creating a fresh Google Doc rather than copying the original.
- The workflow is triggered manually. Add a webhook or form trigger to accept job descriptions dynamically.


--------------------------------------------------

# Newsletter Summarizer — n8n Workflow

## Overview

This workflow runs every night at 10 PM, fetches the latest newsletter emails from a specific Gmail label, summarizes them using a local LLM (Feynman technique), converts the output to styled HTML, and sends a digest email back to you.

---

## Workflow Diagram

```
Schedule Trigger (10 PM daily)
    └── Get many messages (Gmail — by label)
            └── Get a message (Gmail — full content)
                    └── Aggregate
                            └── Code in JavaScript (format transcript)
                                    └── HTTP Request (Ollama / qwen2.5:7b)
                                            └── Markdown (→ HTML)
                                                    └── Send a message (Gmail)
```

---

## Nodes

| Node | Type | Purpose |
|---|---|---|
| **Schedule Trigger** | Schedule | Fires every day at 10:00 PM |
| **Get many messages** | Gmail | Fetches up to 2 emails matching a specific label ID |
| **Get a message** | Gmail | Retrieves full content (body, headers) for each email |
| **Aggregate** | Aggregate | Combines all email fields (text, sender, subject, threadId) into arrays |
| **Code in JavaScript** | Code (JS) | Formats the aggregated emails into a structured transcript string, including direct Gmail links |
| **HTTP Request** | HTTP Request → Ollama | Sends the transcript to the local LLM for summarization with Feynman-style explanations |
| **Markdown** | Markdown | Converts the LLM's Markdown response to HTML |
| **Send a message** | Gmail | Emails the HTML digest to the configured recipient |

---

## Prerequisites

- **n8n** self-hosted instance with Docker
- **Ollama** running locally with `qwen2.5:7b` pulled (`ollama pull qwen2.5:7b`)
- **Gmail OAuth2** credential configured in n8n
- A Gmail label applied to the newsletters you want summarized

---

## Setup Instructions

### 1. Create a Gmail Label
In Gmail, create a label (e.g., `Newsletters`) and apply it to any newsletter senders you want included.

### 2. Get the Label ID
The **Get many messages** node uses a hardcoded label ID (`Label_6245744663598007251`). To find yours:
- In Gmail, right-click the label → **Label settings**, or use the [Gmail API labels.list](https://developers.google.com/gmail/api/reference/rest/v1/users.labels/list) to retrieve it.
- Replace the label ID in the node's `labelIds` filter.

### 3. Set the Recipient Email
In the **Send a message** node, update the `sendTo` field to your email address.

### 4. Verify Ollama is Running
Ensure Ollama is accessible at `http://host.docker.internal:11434` from within your Docker container. Test with:
```bash
curl http://host.docker.internal:11434/api/tags
```

### 5. Activate the Workflow
Toggle the workflow to **Active** in n8n so the schedule trigger fires automatically each night.

---

## LLM Prompt Logic

The **HTTP Request** node sends the full formatted email transcript to `qwen2.5:7b` with these instructions:

- Summarize each email into a daily digest
- For new features: explain what was missing before and what changed
- Use **Feynman's technique** — explain as if to someone unfamiliar with the topic
- Write in an engaging way that makes the reader want to click through
- Format output with consistent Markdown structure per email:
  - Sender, Subject, Link to Email
  - 2–3 sentence summary
  - Key bullet points
  - Horizontal rule separator between emails

---

## Output Format (per email)

```markdown
### Email #1 Summary
**Sender:** example@newsletter.com
**Subject:** This Week in AI
**Link to Email:** [Click Here to View Email](https://mail.google.com/mail/u/0/#inbox/THREAD_ID)

**Summary:**
Feynman-style explanation of the email's core idea...

**Key Points:**
- Point 1
- Point 2
---
```

The Markdown is converted to HTML before being emailed.

---

## Notes & Known Limitations

- The workflow fetches a maximum of **2 emails** per run (configurable in **Get many messages** → `limit`).
- The `dates` field is referenced in the JavaScript code but not aggregated in the **Aggregate** node — dates will show as `Unknown Date`. Add `headers.date` to the Aggregate node's fields to fix this.
- If newsletters are long, the LLM context window may truncate content. Consider chunking or filtering the body before sending.
- The schedule is fixed at 10 PM server time. Adjust the `triggerAtHour` value in the Schedule Trigger node for your timezone.
- The email body cleaner in the JS code strips image captions, view-image lines, and all URLs from the body before sending to the LLM — this keeps the prompt focused on text content only.
