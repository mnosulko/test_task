# PulseFit AI Support Automation (n8n)

An n8n workflow that processes a batch of support tickets for a fictional fitness subscription app. It classifies each ticket with AI, runs deterministic checks, simulates the action, generates a customer reply with AI, validates it, and logs every ticket.

All billing/subscription actions are **simulations**. No real Zendesk, Stripe or PayPal APIs are called.

## Files
- `PulseFitAISupportAutomation.json`: the n8n workflow
- `sample-input.json`: the 6 synthetic tickets

## Setup and run
1. Import `PulseFitAISupportAutomation.json` into n8n.
2. In `Classifier Model` and `Response Model`, select your own Google Gemini credential (any chat model can be used instead).
3. Create an n8n Data Table `pulsefit_support_log` with columns:
   `timestamp` (date), `ticket_id`, `customer_email`, `detected_intent`, `confidence` (number), `matching_subscription_count` (number), `action_type`, `action_status`, `execution_status`, `error_message`, `response_subject`, `response_message`, `classification_note`, `response_source`, `result_json` (all strings unless noted). Then select this table in `Log Ticket Result`.
4. Click **Execute workflow** from `Run Batch`. It uses the 6 tickets from `sample-input.json` (copied into `Load Sample Tickets` so the workflow runs on any n8n instance).
5. Production path: `Ticket Webhook` (POST `/pulsefit-tickets`, body = JSON array of tickets) feeds the same pipeline.

## Flow
Input → Split Into Tickets → Validate & Normalize → Is Valid? → AI Classifier → Parse Classification → Switch (5 intents + classifier-error fallback) → branch check → Merge → Branch Succeeded? → AI Response Generator → Validate Response & Assemble → Log Ticket Result

## Where AI is used and where it is not
**AI (only two places):**
- Classifier: returns `{intent, confidence}` via a structured output parser (temperature 0). It only classifies, it never decides whether an action is possible.
- Response generator: writes the customer reply from the real workflow result (`{subject, message, claims_action_completed}`).

**Deterministic (Code nodes):**
- Validation of required fields (`ticket_id`, `message`, `product`) and email normalization (trim, lowercase, `+alias` removed for gmail.com/googlemail.com)
- `matching_subscription_count` (all subscriptions with the same product, any status)
- Action decisions: cancellation (1 active → `simulated_success`, 0 → `not_executed`, 2+ → `requires_review`), refund (eligible if ≥1 matching), technical KB keyword match (no match → `requires_review`), other (`queued`, no action)
- Validation of the AI reply and the fallback template

## Design decisions
- **AI decides intent, code decides action.** The model cannot trigger or promise an action by itself.
- **One standard result shape** from every branch (`action_type`, `action_status`, `details.reason`, `execution_status`), so downstream nodes are identical for all intents.
- **Untrusted input:** customer text is wrapped in `<customer_message>` tags and both prompts tell the model to ignore instructions inside it.
- **Reply validation before sending:** the reply is replaced with a safe template if the AI output is missing/empty/too short, claims a success when `action_status` is not `simulated_success` (phrase list + `claims_action_completed` flag), leaks internal fields, or skips KB troubleshooting steps.

## Error handling and logging
- Invalid tickets are flagged, not dropped. They go straight to the log with `Failed` and a clear message (e.g. `Validation failed: message is required`). One bad ticket never stops the batch.
- Classifier failures (node error, empty output, unsupported intent) are retried 3 times, then routed through the Switch fallback to `Format Classifier Error` and logged as `Failed`.
- Branch code errors are caught per item. The AI reply is skipped and a safe template is used.
- `execution_status`: `Success` (valid AI reply), `Partial` (fallback reply used), `Failed` (validation, classifier or branch error). An empty AI reply is never `Success`.
- Every ticket gets exactly one row in the Data Table. If a write fails, the failure is collected and the run ends with a visible error.

## Test result (sample-input.json)
| ticket | intent | matching subs | action_status | execution |
|---|---|---|---|---|
| 10001 | cancellation | 1 | simulated_success | Success |
| 10002 | cancellation_with_billing_clarification | 1 | simulated_success | Success |
| 10003 | refund | 2 | simulated_success | Success |
| 10004 | technical_issue | 0 | simulated_success | Success |
| 10005 | invalid | 0 | not_executed | Failed (message required) |
| 10006 | other | 0 | queued | Success |

## Known limitations and next steps
- Sample tickets are embedded in a Code node for the manual run (the Webhook is the real input).
- Webhook has no authentication yet. Add Header Auth.
- No idempotency: the same ticket sent twice would be processed twice.
- The success-phrase check is a keyword list; a stricter check would use a second validation step.
- No separate error-handling workflow and no sub-workflows.
- Subscription data comes from the ticket payload. In production it must be fetched from the billing backend.

## Low-confidence classifier results (production)
Add a threshold (e.g. 0.75) after `Parse Classification`. Below it, skip any action branch, set `action_status = requires_review`, and send the ticket to a human queue with the model's guess as a hint. Use a stricter threshold for money-moving intents (cancellation, refund). Log confidence and review the low-confidence tickets weekly to improve the prompt.

## Integration: Zendesk and credentials
- **Input:** a Zendesk trigger (or a Zendesk webhook on ticket creation) calls the n8n Webhook. Verify the Zendesk signature or a shared secret header. Tickets are processed asynchronously.
- **Output:** a Zendesk node/API call adds the generated reply as a draft or internal note first, and later as a public reply or a tag for the human queue.
- **Billing APIs:** called through an HTTP Request node with a scoped service account and idempotency keys.
- **Credentials:** stored in n8n Credentials (encrypted with `N8N_ENCRYPTION_KEY`), never in Code nodes or workflow JSON. Use least-privilege tokens per system, separate dev/prod credentials, and rotation.

## Three improvements before production (10,000 tickets/month, real refunds/cancellations)
1. **Human approval and guardrails for money actions.** Start in "draft/approve" mode, add refund limits and eligibility checks against real billing data. An AI error must never reach a real refund unreviewed.
2. **Idempotency, security and audit.** Deduplicate by ticket ID, authenticate the webhook, add an immutable audit log of who/what triggered each action, and mask personal data in logs and prompts.
3. **Monitoring and evaluation.** Alert on failures and `Partial` rates, queue with retries for API outages and rate limits, and a regression test set of labeled tickets to measure classifier accuracy after each prompt or model change.
