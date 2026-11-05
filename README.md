# Human-in-the-loop lead reply (n8n workflow)

An n8n workflow that drafts a first reply to every inbound lead, posts it to Slack, and **waits for a person to approve, edit, or reject it** before anything is emailed. Every outcome is logged to Google Sheets.

No AI key is required. Drafts come from editable templates in a Code node. An optional OpenAI step is included but disabled.

```
Webhook ─► Config ─► Build Draft (template) ─► [Optional: AI Draft] ─► Pick Draft
   ─► Post Draft to Slack ─► Wait for Approval (n8n form, 48h limit) ─► Approved?
        ├─ yes ─► Final Reply ─► Send Reply (Gmail) ─► Log: Sent ───────────┐
        └─ no / timed out ─► Log: Rejected / Expired ─────────────────────┴─► Append to Google Sheet
```

## What it does

1. **Lead Webhook** receives `POST /webhook/lead-reply` with `{ name, email, message, company? }` and responds `200` right away.
2. **Config** holds the values you will want to change: Slack webhook URL, email signature, subject line.
3. **Build Draft (template)** validates the payload (it fails loudly on a missing or invalid email or an empty message), guesses intent (`pricing`, `demo`, `support`, `partnership`, `general`) from keywords, and fills a short reply template.
4. **Optional: AI Draft (OpenAI)** is *disabled by default*. Enable it to have a model rewrite the template draft. If it's off, the template draft is used.
5. **Pick Draft** settles on one clean record: lead fields, `draft`, and `draftSource` (`template` or `ai`).
6. **Post Draft to Slack** sends a Block Kit message through a Slack **Incoming Webhook**, so no Slack app credential is needed. It shows the lead, their message, the draft, and a **Review** button.
7. **Wait for Approval** pauses the execution. The button opens an n8n form with a **Decision** dropdown (Approve & send / Reject) and a **Reply** textarea that is pre-filled with the draft and can be edited. If nobody responds within 48 hours, the execution resumes as `expired`.
8. **Approved?** routes the result:
   - **Approve:** **Final Reply** uses the edited text, or the draft if the box was left empty. **Send Reply (Gmail)** then emails the lead, and the row is logged as `sent` or `sent_edited`.
   - **Reject or timeout:** the row is logged as `rejected` or `expired`, and no email is sent.
9. **Append to Google Sheet** writes `timestamp, receivedAt, name, email, company, intent, status, draftSource, reply`.

## Import

1. Download [`workflows/human-in-the-loop-reply.json`](workflows/human-in-the-loop-reply.json).
2. In n8n, go to **Workflows → Add workflow → ⋯ → Import from File** (or paste the JSON onto the canvas).
3. Follow [SETUP.md](SETUP.md) to add credentials and replace the placeholders.
4. Send the sample payload in [`examples/sample-payload.json`](examples/sample-payload.json) (see SETUP.md for the curl command).

## Required credentials

| Node | Credential | Required? |
|---|---|---|
| Post Draft to Slack | None. Paste a Slack Incoming Webhook URL into **Config** | Yes |
| Send Reply (Gmail) | Gmail OAuth2 | Yes (or swap for **Send Email**/SMTP) |
| Append to Google Sheet | Google Sheets OAuth2 | Yes (or delete the node if you don't need a log) |
| Optional: AI Draft (OpenAI) | Header Auth (`Authorization: Bearer sk-...`) | No, disabled by default |

The JSON contains no credentials, IDs, or personal data. Every value you need to replace starts with `REPLACE` or is a placeholder in **Config**.

## Limitations

- **The Review link has to be reachable by whoever clicks it.** It points to your n8n instance's `/form-waiting/...` URL. On self-hosted n8n, set `WEBHOOK_URL` to your public URL, or links will say `localhost`.
- **Anyone with the link can approve.** The link is signed, but it isn't tied to a Slack user. Keep the channel private, or set *Form Authentication* on the Wait node.
- **One lead per execution, waiting up to 48 hours.** Waiting executions are stored in n8n's database. Change `resumeAmount` on the Wait node if you need a different window.
- **Intent detection is just keyword regex**, and the templates are generic. Edit `RULES` and `TEMPLATES` in the Code node to match your business.
- **No spam filtering or deduplication.** Put a check in front of the webhook if your form gets bot traffic.
- **Plain-text email.** The reply is sent as a new message, not threaded onto an earlier one.
- The Slack button is a URL button, so you don't need a Slack app or interactivity endpoint. The approval itself happens in the n8n form.

## Tested

The webhook → template draft → Slack message → form approval → logging path was run end to end on n8n 2.46 (self-hosted), with Slack pointed at a local mock and the Gmail/Sheets nodes disabled. Four cases were run: approve with an edit, approve as-is, reject, and an invalid payload. The 48-hour expiry path and the OpenAI node were not run. Node versions were chosen to import on recent n8n 1.x as well.

## License

MIT. See [LICENSE](LICENSE).
