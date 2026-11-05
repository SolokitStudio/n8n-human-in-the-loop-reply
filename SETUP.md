# Setup

Allow about 10 minutes. You need an n8n instance that is reachable from the browser you'll approve in (n8n Cloud works as-is; for self-hosted, see step 6).

## 1. Import

**Workflows → Add workflow → ⋯ → Import from File** → `workflows/human-in-the-loop-reply.json`.

## 2. Slack (Incoming Webhook)

1. Go to https://api.slack.com/apps → **Create New App → From scratch**, then pick your workspace.
2. Open **Incoming Webhooks**, turn it **On**, choose **Add New Webhook to Workspace**, and pick a private channel such as `#lead-replies`.
3. Copy the URL (`https://hooks.slack.com/services/...`).
4. In n8n, open **Config** and paste it into `slackWebhookUrl`.

While you're in **Config**, also set `signature` (the sign-off that goes at the end of every reply) and `subjectPrefix` (the subject becomes `Re: <subjectPrefix> (<company>)`).

## 3. Gmail

Open **Send Reply (Gmail)** → **Credential** → **Create new** → Gmail OAuth2, then follow n8n's prompts.

To use SMTP instead, delete the Gmail node and add **Send Email** in its place, between **Final Reply** and **Log: Sent**:
- To: `{{ $('Pick Draft').item.json.email }}`
- Subject: `{{ $('Pick Draft').item.json.subject }}`
- Text: `{{ $json.finalReply }}`

## 4. Google Sheets log

1. Create a spreadsheet with a tab named **`Log`** whose row 1 is exactly:
   `timestamp | receivedAt | name | email | company | intent | status | draftSource | reply`
2. Copy the spreadsheet ID from its URL (`https://docs.google.com/spreadsheets/d/<THIS_PART>/edit`).
3. Open **Append to Google Sheet**: add a Google Sheets OAuth2 credential and replace `REPLACE_WITH_SPREADSHEET_ID`.

If you don't want a log, delete this node. Nothing else depends on it.

## 5. Optional: AI drafts

1. Open **Optional: AI Draft (OpenAI)** → Credential → **Header Auth**, with Name `Authorization` and Value `Bearer sk-...`.
2. Change the model in the JSON body if you like (the default is `gpt-4o-mini`).
3. Right-click the node → **Activate** (it ships deactivated).

The model gets the lead plus the template draft and returns a rewritten draft. **Pick Draft** falls back to the template if the response is empty. Someone still approves every reply.

## 6. Self-hosted only: public URLs

The Slack button links to `$execution.resumeFormUrl`, which is built from your instance URL. Set:

```
WEBHOOK_URL=https://n8n.yourdomain.com/
```

and restart n8n. Otherwise the links will point at `localhost`.

## 7. Test

Activate (publish) the workflow, then run:

```bash
curl -X POST https://YOUR-N8N/webhook/lead-reply \
  -H 'Content-Type: application/json' \
  -d @examples/sample-payload.json
```

(Before activating, use `/webhook-test/lead-reply` with **Listen for test event**.)

You should see a Slack message. Click **Review**, edit if you want, choose **Approve & send**, and check your inbox and the sheet.

## Connecting a real form

Point any form at the webhook: Webflow, Framer, Typeform, Tally, or a plain HTML `<form>` with a little JS. It needs to send JSON (or form fields) named `name`, `email`, `message`, and optionally `company`. If your form uses other field names, either rename them at the top of the **Build Draft (template)** Code node or put a Set node in front of it.
