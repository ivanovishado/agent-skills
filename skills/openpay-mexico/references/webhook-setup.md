# OpenPay Webhook Setup and Testing

## Overview

OpenPay webhooks are critical for SPEI and OXXO payments, which confirm asynchronously after the initial charge creation. Without webhooks, you'll never know when these payments complete.

**Webhook events:**
- `charge.succeeded` - Payment completed successfully
- `charge.failed` - Payment attempt failed
- `charge.cancelled` - Payment cancelled by user
- `charge.pending` - OXXO payment created but not yet paid

## Production Setup

### 1. Configure Webhook URL in OpenPay Dashboard

Navigate to: OpenPay Dashboard → Settings → Webhooks

Add webhook URL:
```
https://yourdomain.com/api/webhooks/openpay
```

**Requirements:**
- ✅ Must use HTTPS (not HTTP)
- ✅ Must return 200 status code within 5 seconds
- ✅ Must be publicly accessible (not localhost)

### 2. Get Webhook Secret

OpenPay provides a webhook secret for signature verification. Store in `.env.local`:

```bash
OPENPAY_WEBHOOK_SECRET=your_secret_from_openpay_dashboard
```

**Critical:** Never commit this secret to version control.

### 3. Verify Signature Implementation

Your webhook handler MUST verify the signature:

```typescript
const verifyWebhookSignature = (payload: string, signature: string | null): boolean => {
  if (!signature) return false;

  const webhookSecret = process.env.OPENPAY_WEBHOOK_SECRET;
  if (!webhookSecret) {
    console.error("OPENPAY_WEBHOOK_SECRET not configured");
    return false;
  }

  const hmac = crypto.createHmac("sha256", webhookSecret);
  hmac.update(payload);
  const expectedSignature = hmac.digest("hex");

  // Use timing-safe comparison to prevent timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
};
```

**Why this matters:**
- Prevents attackers from sending fake payment confirmations
- Ensures webhooks actually come from OpenPay
- Required for PCI compliance

## Local Development Testing

OpenPay webhooks require publicly accessible URLs. Use tunneling tools to expose localhost.

### Option 1: ngrok (Recommended)

**Install:**
```bash
npm install -g ngrok
# or download from ngrok.com
```

**Start tunnel:**
```bash
# Start your Next.js dev server
npm run dev

# In another terminal, expose port 3000
ngrok http 3000
```

**Output:**
```
Forwarding   https://abc123.ngrok.io -> http://localhost:3000
```

**Configure in OpenPay:**
```
Webhook URL: https://abc123.ngrok.io/api/webhooks/openpay
```

**Pros:**
- Free tier available
- Stable URLs (with paid plan)
- Inspection UI at http://localhost:4040
- Replay webhooks from UI

**Cons:**
- URL changes on each restart (free tier)
- Requires external service

### Option 2: localtunnel

**Install:**
```bash
npm install -g localtunnel
```

**Start tunnel:**
```bash
lt --port 3000 --subdomain myapp-openpay
```

**Output:**
```
your url is: https://myapp-openpay.loca.lt
```

**Pros:**
- Free and open source
- Can request specific subdomain
- No account required

**Cons:**
- Less stable than ngrok
- No inspection UI
- Password prompt on first visit

### Option 3: Vercel Preview Deployments

Push to feature branch, get preview URL:

```bash
git push origin feature/openpay-integration
```

Vercel provides URL like:
```
https://rentatuterraza-abc123.vercel.app
```

**Configure in OpenPay:**
```
Webhook URL: https://rentatuterraza-abc123.vercel.app/api/webhooks/openpay
```

**Pros:**
- Real production environment
- HTTPS included
- Stable URL per deployment

**Cons:**
- Slower iteration (deploy on every change)
- Shares webhook URL with OpenPay sandbox for all devs

## Testing Webhook Events

### Test with OpenPay Sandbox

OpenPay sandbox automatically triggers webhooks for test transactions.

**Card payment (immediate):**
```bash
# Create charge with test card 4111 1111 1111 1111
# Webhook fires immediately: charge.succeeded
```

**SPEI payment:**
```bash
# Create SPEI charge
# In OpenPay sandbox dashboard, manually "complete" the transfer
# Webhook fires: charge.succeeded
```

**OXXO payment:**
```bash
# Create OXXO charge
# In OpenPay sandbox dashboard, manually "mark as paid"
# Webhook fires: charge.succeeded
```

### Manual Webhook Testing

Use `curl` to send test webhook events:

```bash
# Generate test signature
WEBHOOK_SECRET="your_webhook_secret"
PAYLOAD='{"type":"charge.succeeded","event_date":"2024-01-15T10:30:00Z","transaction":{"id":"test_123","amount":5500,"status":"completed","order_id":"booking_abc","method":"bank_account"}}'

SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" -hex | sed 's/^.* //')

# Send webhook
curl -X POST https://your-domain.com/api/webhooks/openpay \
  -H "Content-Type: application/json" \
  -H "x-openpay-signature: $SIGNATURE" \
  -d "$PAYLOAD"
```

**Expected response:**
```json
{"received": true}
```

### Webhook Debugging Checklist

If webhooks aren't working:

1. **Check URL accessibility:**
   ```bash
   curl https://your-domain.com/api/webhooks/openpay
   # Should return 405 Method Not Allowed (expecting POST)
   ```

2. **Verify signature calculation:**
   - Log the raw request body
   - Log the received signature header
   - Log the calculated expected signature
   - Ensure they match exactly

3. **Check environment variable:**
   ```typescript
   console.log("Webhook secret configured:", !!process.env.OPENPAY_WEBHOOK_SECRET);
   ```

4. **Monitor OpenPay dashboard:**
   - Check "Webhooks" section for delivery attempts
   - Look for failed delivery logs
   - Check response codes/errors

5. **Test signature separately:**
   ```typescript
   const testPayload = '{"type":"charge.succeeded"}';
   const testSignature = "expected_signature_from_openpay_logs";
   const isValid = verifyWebhookSignature(testPayload, testSignature);
   console.log("Signature valid:", isValid);
   ```

## Webhook Idempotency

OpenPay may send duplicate webhooks if it doesn't receive a 200 response. Handle idempotency:

```typescript
async function handleChargeSucceeded(transaction: any) {
  const bookingId = transaction.order_id;

  // Query current booking status first
  const { data: booking } = await supabase
    .from("bookings")
    .select("status, payment_id")
    .eq("id", bookingId)
    .single();

  // Idempotency check
  if (booking?.status === "confirmed" && booking?.payment_id === transaction.id) {
    console.log(`Booking ${bookingId} already confirmed, skipping duplicate webhook`);
    return NextResponse.json({ received: true }, { status: 200 });
  }

  // Only update if status is "approved" (waiting for payment)
  await supabase
    .from("bookings")
    .update({ status: "confirmed" })
    .eq("id", bookingId)
    .eq("payment_id", transaction.id)
    .eq("status", "approved");

  return NextResponse.json({ received: true }, { status: 200 });
}
```

## Webhook Retry Logic

OpenPay retry schedule if your endpoint fails:
1. Immediate retry
2. 5 minutes later
3. 30 minutes later
4. 1 hour later
5. 6 hours later
6. 24 hours later

**Best practices:**
- Always return 200, even on error (log for manual review)
- Make webhook handlers idempotent
- Keep processing under 5 seconds (or return 200, process async)

## Production Monitoring

### Log All Webhooks

```typescript
console.log({
  timestamp: new Date().toISOString(),
  event: event.type,
  bookingId: transaction.order_id,
  paymentId: transaction.id,
  amount: transaction.amount,
  method: transaction.method,
  status: transaction.status,
});
```

### Alert on Failures

Set up alerts for:
- Invalid signature attempts (potential attack)
- Failed database updates
- Missing order_id in transaction
- Unexpected event types

### Webhook Dashboard

OpenPay dashboard shows:
- All webhook delivery attempts
- Response codes
- Response times
- Failure reasons

Check regularly to catch issues early.

## Security Considerations

### Always Verify Signature

```typescript
// ❌ WRONG - Accepting all webhooks
export async function POST(request: NextRequest) {
  const event = await request.json();
  await handleChargeSucceeded(event.transaction);
}

// ✅ RIGHT - Verifying signature first
export async function POST(request: NextRequest) {
  const rawBody = await request.text();
  const signature = request.headers.get("x-openpay-signature");

  const isValid = verifyWebhookSignature(rawBody, signature);
  if (!isValid) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const event = JSON.parse(rawBody);
  await handleChargeSucceeded(event.transaction);
}
```

### Use HTTPS Only

```typescript
// In production, reject HTTP requests
if (process.env.NODE_ENV === "production" && !request.url.startsWith("https")) {
  return NextResponse.json({ error: "HTTPS required" }, { status: 400 });
}
```

### Rate Limiting

Implement rate limiting to prevent webhook flooding:

```typescript
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
});
```

## Common Issues

### Issue: Webhooks not received at all

**Diagnosis:**
- Check OpenPay dashboard webhook delivery logs
- Verify URL is publicly accessible
- Test with curl from external IP

**Solution:**
- Ensure HTTPS is configured
- Check firewall/security group settings
- Verify route handler is exported correctly

### Issue: Signature verification fails

**Diagnosis:**
- Log raw body and signature
- Check webhook secret matches dashboard
- Verify no body parsing middleware interferes

**Solution:**
```typescript
// Next.js App Router: Use request.text() not request.json()
const rawBody = await request.text(); // ✅
const body = await request.json();   // ❌ Changes body, breaks signature
```

### Issue: Duplicate confirmations

**Diagnosis:**
- OpenPay retrying due to timeout
- No idempotency checks

**Solution:**
- Add idempotency logic (check booking status before update)
- Optimize webhook handler to respond in <5s
- Use database constraints to prevent duplicate updates

### Issue: 500 errors in webhook handler

**Diagnosis:**
- Database connection issues
- Missing environment variables
- Unhandled edge cases

**Solution:**
- Wrap in try-catch, always return 200
- Log errors for manual investigation
- Test with various event types in development
