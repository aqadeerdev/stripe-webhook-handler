# Payment Webhook Handler (Stripe)

An n8n workflow that securely receives Stripe payment webhooks, verifies them, logs outcomes to a CRM, and automatically retries a customer recovery notification.

## The problem
Payment failures are inevitable, and without automation, a failed payment can get silently lost if some does not manually notice and follow up. This workflow handles the full lifecycle by verifying whether the event is genuinely from Stripe, log it, and attempt recovery automatically.

## What it does
1. A generic Webhook node listens for incoming Stripe events.
2. A custom Code node recomputes Stripe's HMAC-SHA256 signatures against the raw request body and compares it using a timing-safe comparison to block any requests not signed by Stripe.
3. Includes a check against replay attacks by rejecting signed requests older than 5 minutes.
4. Checks for duplicates before any processing using the event ID to safeguard against processing or logging the same event again.
5. A Code node extracts and normalizes the event data and the workflow splits based on the payment's success.
6. Successful payments are logged directly to Airtable.
7. A recovery email is sent to the customer for failed payments. If the sending process itself fails, the workflow retries up to 3 times with increasing backoff times. An alert is sent to Slack in case max retries are exhausted.
8. Every outcome is logged to Airtable.

## Tools used
- **n8n** (self-hosted via **Docker**) for workflow orchestration
- **Stripe** (test mode) for payment event source and webhook signing
- **JavaScript** (n8n Code node) for signature verification, event parsing, and retry state management
- **Airtable** for event logging
- **Slack API** for failure escalation alerts
- **SMTP** (Gmail) for customer recovery emails

## Setup
1. Import the workflow JSON into your own n8n instance.
2. Create a Stripe account (test mode), register a webhook endpoint pointing at your n8n instance, and note the signing secret.
3. Set the following environment variables on your n8n instance: `STRIPE_WEBHOOK_SECRET`, `NODE_FUNCTION_ALLOW_BUILTIN=crypto`, `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`.
4. Create your own credentials for Airtable and Slack, and update the relevant nodes to point at your own base/channel.
5. Use the Stripe CLI to send test events: `stripe trigger payment_intent.succeeded` and `stripe trigger payment_intent.payment_failed`.

## Possible improvements
- The recovery email includes a placeholder where the real "retry your payment" link would go. This was a deliberate choice as generating the new link would require a fresh Checkout Session with line-item/product data, something that is not provided in the PaymentIntent object received from the Webhook. A real link will look up the original Checkout Session or rely on an external order database.

## Possible extensions
- Support additional Stripe event types (refunds, disputes).
- Add a dead-letter queue pattern for events that fail signature verification completely to distinguish misconfiguration from active abuse.