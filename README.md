# AI-Powered Gmail Unread Email Summarizer with n8n

An on-demand **n8n workflow** that retrieves up to five unread Gmail messages, summarizes each with an **AI Agent powered by an OpenAI Chat Model**, aggregates the summaries into one message, and emails the digest to your selected Gmail address.

## Workflow Architecture

![n8n workflow screenshot](workflow.png)

```text
Manual Trigger
    ↓
Gmail — Unread message (Get Many, unread, limit 5)
    ↓ (up to 5 items)
AI Agent ← OpenAI Chat Model
    ↓ (one summary per email)
Aggregate (5 items → 1 item)
    ↓
Gmail — Send a message (one digest email)
```

## Key Features

- **Run on demand:** Start only when you click **Execute workflow**.
- **Unread email retrieval:** Fetch up to five unread Gmail messages.
- **AI summarization:** Generate a concise summary of each message, limited to **20 words**.
- **Single email digest:** Combine all summaries into one email body, with **one separate line per email**.
- **No header or greeting in the digest body:** The Gmail message's subject line carries the title.

## Prerequisites

- An n8n instance (Cloud or self-hosted).
- Gmail OAuth2 credentials authorized to read messages and send emails.
- An OpenAI API credential and a supported chat model in n8n.
- An email address to receive the final digest.

> **Privacy:** Email content is sent to the configured AI provider for summarization. Review your organization's data handling policies before enabling the workflow.

## Node-by-Node Configuration

### 1. Manual Trigger

- **Node:** Manual Trigger (When clicking 'Execute workflow')
- **Purpose:** Run the workflow only when requested.

### 2. Gmail — Unread message

- **Resource:** Message
- **Operation:** Get Many (Get All)
- **Read Status:** Unread
- **Return All:** Off
- **Limit:** 5
- **Credentials:** Your Gmail OAuth2 connection

Make sure the returned message data includes enough email content for summarization; if the node returns only metadata or a snippet, adjust its output settings or fetch the full message before sending it to the agent.

### 3. AI Agent

- **Node:** AI Agent
- **Chat model connection:** OpenAI Chat Model (connected to the **Chat Model** port, as shown in the screenshot).
- **Prompt source:** Define a custom prompt from the current input item's email content.
- **Expected output:** One plain-text summary for each email.

**Suggested prompt:**

```text
Summarize the following email in one concise line of 20 words maximum.
Output only the summary: no greeting, heading, numbering, bullets, or line breaks.
Do not invent information.

Subject: {{ $json.subject || '' }}
From: {{ $json.from || '' }}
Email: {{ $json.textPlain || $json.snippet || $json.body || '' }}
```

> Field names differ among Gmail node output versions. Inspect the node's **Output** and replace the expressions with the actual email subject/body fields. If the Gmail body arrives in encoded or HTML form, decode/strip it before summarization.

### 4. Aggregate

- **Node:** Aggregate
- **Purpose:** Combine the separate AI Agent outputs into **one item**.
- **Approach:** Aggregate individual AI output values (`output`) into a list/array.

For example, name the aggregated list `summaries`. The screenshot shows the item count dropping from **5 items** to **1 item** at this stage.

**Format the final message as separate lines** in the Gmail Send node with an expression similar to:

```javascript
{{ $json.summaries.map(s => typeof s === 'string' ? s : s.output).filter(Boolean).join('\n') }}
```

This assumes `summaries` contains strings or objects with an `output` property. Check the Aggregate node's actual output schema and adapt the expression if needed. For blank lines between summaries, use `join('\n\n')` instead.

### 5. Gmail — Send a message

- **Resource:** Message
- **Operation:** Send
- **To:** Your own destination email address
- **Subject:** `AI Digest | 5 Latest Unread Emails`
- **Email Type:** Text / plain text
- **Message:** The aggregated, newline-separated summaries expression above.

Use a dynamic subject if fewer than five emails may be returned, for example `AI Digest | Unread Email Summaries`.

## Example Digest

**Subject:** AI Digest | Unread Email Summaries

```text
SerpApi sent instructions for using its search-result scraping API.
Google notified you about account information shared with SerpApi.
AWS announced three free agentic AI microcredentials with hands-on labs.
Buildathon organizers welcomed participants and encouraged project ideas.
A guide introduced Claude Certified Architect certification and its benefits.
```

The email **body** consists only of summaries, one per line. The subject is set separately in Gmail.

## How to Test

1. Ensure the inbox contains between one and five unread emails.
2. In n8n, click **Execute workflow**.
3. Confirm the Gmail Read node outputs up to **5 items**.
4. Confirm the AI Agent returns one `output` summary per item.
5. Confirm Aggregate outputs **1 item** containing the summaries.
6. Check your Gmail inbox for a **single digest message** with properly separated lines.

## Troubleshooting

| Issue | Resolution |
|---|---|
| All summaries appear on one line | Join summaries using `\n` and set Gmail **Email Type** to **Text**. |
| Only one email is summarized | Inspect the incoming item count and ensure the Agent executes once per incoming email. |
| No emails are retrieved | Verify there are unread emails, Gmail OAuth scopes, and the Read Status filter. |
| The agent summarizes only subject/snippet | Map the complete message body or add a Gmail Get Message step if needed. |
| Digest sends several separate emails | Ensure Aggregate reduces input items to **one item** before Gmail Send. |
| AI Agent asks for input | Use a defined prompt that reads the incoming Gmail item's fields. |

## Considerations

- **Not automatically marked as read:** If your Gmail read operation does not change message status, reruns can summarize the same emails. Add a separate mark-as-read or label step only if desired.
- **Zero unread messages:** Consider a conditional branch so the Send node does not send an empty digest.
- **20-word constraint:** The prompt requests a maximum of 20 words; for strict enforcement, add a validation/code step after the Agent.
- **PII/data protection:** Avoid logging or sharing sensitive email contents unnecessarily.

## Project Structure

```text
n8n-gmail-ai-summarizer/
├── README.md
├── workflow.png
└── workflow.json       # Export separately from your own n8n instance
```

## Technology Stack

**n8n** · **Gmail** · **AI Agent** · **OpenAI Chat Model** · **Aggregate**

## Status

The supplied screenshot shows successful execution (green checkmarks) for Manual Trigger, Gmail Read, AI Agent, OpenAI Chat Model, Aggregate, and Gmail Send. Actual configuration details and output formatting must be verified in your n8n instance.
